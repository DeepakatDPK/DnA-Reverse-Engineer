# Task Chunking & Session Continuation Patterns

## When Limits Are Reached

### 1. GitHub API Rate Limiting (60s / 5000 req/hr)
```python
# Pattern: Bounded pagination with retry-after handling
async def fetch_with_backoff(session, url, token, max_retries=3):
    for attempt in range(max_retries):
        async with session.get(url, headers={
            "Authorization": f"token {token}",
            "Accept": "application/vnd.github+json"
        }) as resp:
            if resp.status == 429:
                retry_after = int(resp.headers.get("Retry-After", 60))
                await asyncio.sleep(retry_after)
                continue  # Back off and retry
            elif resp.status == 403:
                # Primary limit exhausted
                reset = int(resp.headers.get("x-ratelimit-reset", 60))
                await asyncio.sleep(min(retry_after, reset))
                continue
            else:
                return await resp.json()
    raise RateLimitExceeded("Max retries exceeded")
```

### 2. Hermes Session Compression (/)
```
/
→ Compresses current session context
→ Creates new continuation session: "#2"
→ Full conversation history remains in SQLite
→ New session inherits all knowledge, skills, tools
```

**When to use:**
- After completing a major subtask
- When context grows large (>2000 messages)
- When switching to a new but related topic
- Before long pauses in work

### 3. Auto-Continuation Goals (/goal)
```yaml
# config.yaml
goals:
  max_turns: 20  # Auto-pause after N continuation turns
```

**Pattern:**
- Define a standing goal: `/goal Write comprehensive API docs for endpoint X`
- Hermes keeps working until goal is satisfied OR budget exhausted
- If budget hits: auto-pauses, asks user `/goal resume` to continue
- No context loss — resumes with full conversation history

### 4. Bounded Subtasks (delegate_task)
```python
# Break large operations into fixed chunks
tasks = [
    {"goal": f"Extract page {i} data from {url}", "index": i}
    for i in range(0, total_pages, BATCH_SIZE)
]

# Parallel execution with concurrency limit (default: 5)
results = await delegate_task(tasks=tasks)
# Aggregate in parent session
```

### 5. File-Based Checkpointing
```python
# Save state after each milestone
class ReversibleTask:
    def __init__(self):
        self.checkpoints = []
    
    def checkpoint(self, step, data):
        self.checkpoints.append({
            "step": step,
            "timestamp": datetime.utcnow().isoformat(),
            "data": data
        })
    
    def resume_from(self, idx):
        checkpoint = self.checkpoints[idx]
        self.state = checkpoint["data"]  # Restore exact state
```

## Complete Workflow Example

### Scenario: Processing 100 GitHub Repositories

```python
# Step 1: Break into chunks (batch_size=20)
chunks = [range(i, min(i+20, 100)) for i in range(0, 100, 20)]

# Step 2: Create bounded subtasks
tasks = []
for chunk_idx, chunk in enumerate(chunks):
    tasks.append({
        "goal": f"Analyze repos {chunk[0]}-{chunk[-1]}: extract stars, forks, last commit",
        "chunk": chunk,
        "parent_session": "DnA-Reverse-Engineer-repo-analysis"
    })

# Step 3: Run with bounded concurrency
all_results = await delegate_task(tasks=tasks)

# Step 4: Aggregate in parent session
import json
with open("repo_analysis.json", "w") as f:
    json.dump(all_results, f, indent=2)

# Step 5: If rate-limited during Step 3, retry with backoff
# Step 6: Resume from checkpoint if interrupted
```

## Session Resume Commands

```bash
# Resume most recent session with full context
hermes --continue
hermes -c

# Resume by name (auto-follows lineage)
hermes -c "DnA-Reverse-Engineer"
# → Resumes "DnA-Reverse-Engineer #3" if exists

# Resume specific ID
hermes --resume 20261109_184700_a1b2c3

# Interactive resume (for long tasks)
hermes chat --continue
```

## Key Takeaways

1. **GitHub API**: Use Link headers for pagination, conditional requests for caching, batch ≤100 concurrent
2. **Hermes**: Use `/` for context compression, `/goal` for auto-continuation, `delegate_task` for parallel subtasks
3. **Checkpointing**: Save state after each milestone, resume from last known good state
4. **Bounded Operations**: Never run unbounded loops — always have clear termination conditions

---
*Compiled November 2026 — Based on GitHub API docs and Hermes Agent session model*
