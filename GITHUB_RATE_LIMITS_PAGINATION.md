# GitHub API Rate Limiting & Pagination Strategy

## Rate Limits (REST API)

| Type | Auth | Limit | Notes |
|------|------|-------|-------|
| Unauthenticated | None | 60 req/hour | IP-bound |
| PAT / OAuth | Bearer token | 5,000 req/hour | Per user |
| GitHub Apps | App token | ~15,000 req/hour | Enterprise |
| Secondary (CPU) | Any | 90s CPU/60s real-time | Shared REST+GraphQL |

### Response Headers to Monitor
```
X-RateLimit-Limit:        5000              (per hour)
X-RateLimit-Remaining:    4892              (current count)
X-RateLimit-Reset:        1731134400        (UTC epoch seconds)
X-RateLimit-Remaining-2:  5999              (secondary limit)
Retry-After:              60                (if rate-limited)
```

## Best Practices

### 1. Pagination (Link Headers)
```python
# NEVER parse page=1&per_page=100 manually
# Use Link header for efficient pagination

link_header = "https://api.github.com/repos/owner/repos?page=2&per_page=100; rel=\"next\""

# Parse with:
from urllib.parse import urlparse, parse_qs
url = urlparse(link_header.split("; rel=\\\"next\\\"")[0])
params = parse_qs(url.query)
next_page = params.get("page", [None])[0]
```

### 2. Conditional Requests (304 Not Modified)
```python
# Make auth requests conditional to reduce rate consumption
headers = {
    "Authorization": "token ghp_...",
    "If-None-Match": 'W/"abc123"'  # From ETag header
}
# Returns 304 with no body — doesn\'t count against rate limit
```

### 3. Concurrency Control
```python
# Secondary limit: ~100 concurrent requests max
# For bulk operations, use serial processing or bounded queue

import asyncio
import aiohttp
from aiohttp import ClientSession

async def fetch_with_backoff(session: ClientSession, url: str, token: str):
    retry_after = None
    while True:
        async with session.get(url, headers={
            "Authorization": f"token {token}",
            "Accept": "application/vnd.github+json"
        }) as resp:
            if resp.status == 429:
                retry_after = int(resp.headers.get("Retry-After", 60))
            elif resp.status == 403:
                reset = int(resp.headers.get("x-ratelimit-reset", 60))
                await asyncio.sleep(min(retry_after, reset))
            else:
                return await resp.json()
        if retry_after:
            await asyncio.sleep(retry_after)
            retry_after = None  # Reset for next batch

# Process in batches of ~50-100 concurrent requests
async def batch_fetch(urls: list[str], token: str, batch_size: int = 80):
    async with ClientSession() as session:
        for i in range(0, len(urls), batch_size):
            batch = urls[i:i + batch_size]
            tasks = [fetch_with_backoff(session, url, token) for url in batch]
            await asyncio.gather(*tasks)
```

### 4. Polling with x-poll-interval
```python
# If response includes x-poll-interval header, wait at least that many seconds
poll_interval = int(resp.headers.get("x-poll-interval", 30))
await asyncio.sleep(poll_interval)
```

### 5. Request Batching Strategies

#### A. GraphQL Batch (for complex queries)
```python
# Single GraphQL request can fetch multiple resources
query = """
{
  repository(owner: "DeepakatDPK", name: "DnA-Reverse-Engineer") {
    pullRequests(first: 100, states: OPEN) {
      nodes { number title createdAt }
      pageInfo { hasNextPage endCursor }
    }
  }
}
"""
# Returns all in one request — no pagination loop needed
```

#### B. Webhooks (ideal for event-driven)
```python
# Subscribe to events instead of polling
POST https://api.github.com/repos/owner/repo/subscribers
{
  "events": ["pull_request", "push"],
  "active": true
}

# GitHub calls your endpoint automatically — zero rate limit cost
```

## Session Continuation & Task Chunking

### Hermes Agent Patterns

#### 1. Session Resume
```bash
# Continue previous session with full context
hermes --continue
hermes -c
hermes chat --continue

# Resume by ID or name (auto-follows lineage: "my-project" → "my-project #2")
hermes --resume 20261109_145300_a1b2c3
hermes -c "DnA-Reverse-Engineer"
```

#### 2. Auto-Continuation Goals
```yaml
# In config.yaml
goals:
  max_turns: 20  # Auto-pause goal after N continuation turns
```

#### 3. Context Compression
```
/
→ Compresses session, creates continuation session (#2)
→ Full history retained in session DB
→ New session inherits context automatically
```

### Task Chunking Strategy

#### Pattern 1: Bounded Subtasks
```python
# Break large operations into fixed-size chunks
def process_large_dataset(items, chunk_size=100):
    for i in range(0, len(items), chunk_size):
        chunk = items[i:i + chunk_size]
        # Each chunk is a bounded, independent task
        results = process_chunk(chunk)
        # Save results to file, then next chunk continues
        save_results(i, results)
```

#### Pattern 2: Checkpointing
```python
# Save state after each meaningful milestone
class ReversibleTask:
    def __init__(self):
        self.checkpoints = []
    
    def checkpoint(self, step: int, data: dict):
        self.checkpoints.append({
            "step": step,
            "timestamp": datetime.utcnow().isoformat(),
            "data": data
        })
    
    def resume_from(self, checkpoint_idx: int):
        # Restore state, continue from checkpoint
        checkpoint = self.checkpoints[checkpoint_idx]
        self.state = checkpoint["data"]
```

#### Pattern 3: Parallel Batch Processing
```python
# Use delegate_task for parallel subtasks (bounded concurrency)
tasks = [
    {"goal": f"Extract page {i} data from {url}", "index": i}
    for i in range(0, total_pages, BATCH_SIZE)
]

# Subagents run in parallel with 5 concurrent limit
results = await delegate_task(tasks=tasks)
# Aggregate results in parent session
```

## Complete GitHub API Workflow

```python
import aiohttp
import asyncio

class GitHubAPI:
    def __init__(self, token: str):
        self.token = token
        self.base_url = "https://api.github.com"
    
    async def fetch_paginated(self, endpoint: str, params: dict = None) -> list:
        """Efficient pagination using Link headers"""
        all_items = []
        async with aiohttp.ClientSession() as session:
            url = self.base_url + endpoint
            if params:
                url += "?" + "&".join(f"{k}={v}" for k, v in params.items())
            
            while url:
                async with session.get(url, headers={
                    "Authorization": f"token {self.token}",
                    "Accept": "application/vnd.github+json"
                }) as resp:
                    if resp.status == 429:
                        retry_after = int(resp.headers.get("Retry-After", 60))
                        await asyncio.sleep(retry_after)
                        continue
                    
                    data = await resp.json()
                    all_items.extend(data)
                    
                    # Use Link header for next page
                    link = resp.headers.get("Link")
                    if not link or "rel=\"next\"" not in link:
                        break
                    
                    # Extract next URL
                    next_url = link.split("rel=\"next\"")[1].split(",")[0].strip()
                    url = next_url
        
        return all_items
    
    async def create_repo(self, name: str, description: str = "") -> dict:
        """Create a public repository"""
        payload = {
            "name": name,
            "description": description,
            "private": False
        }
        
        async with aiohttp.ClientSession() as session:
            async with session.post(
                f"{self.base_url}/user/repos",
                json=payload,
                headers={
                    "Authorization": f"token {self.token}",
                    "Accept": "application/vnd.github+json"
                }
            ) as resp:
                if resp.status == 201:
                    return await resp.json()
                else:
                    raise Exception(f"Failed to create repo: {await resp.text()}")
```

---
*Compiled November 2026 — GitHub REST API v3*
