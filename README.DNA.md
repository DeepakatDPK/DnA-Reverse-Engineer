# DnA Reverse Engineer — REA Framework

This repository contains the complete **Reverse Engineer Anything (REA)** framework, a production-grade reverse engineering toolchain that exposes analysis capabilities through MCP and CLI.

## What is REA?

REA bridges AI agents to reverse engineering. It enables agents to inspect native binaries, JavaScript/Electron apps, firmware, Android APKs, .NET assemblies, and more — **without requiring source code**.

### Core Capabilities
- **Native Binaries**: Pseudocode, assembly, symbols, call traces, decompilation (Hopper/Ghidra/IDA)
- **JavaScript/Electron**: Module graphs, IPC channels, route discovery, source map recovery
- **Firmware/Android**: Firmware regions, APK structure, manifest extraction
- **Browser/Web**: DOM analysis, runtime capture, network observation
- **.NET Assemblies**: CIL inspection, metadata analysis, native dependencies
- **Cross-Platform**: ELF, PE, Mach-O, EVM bytecode, managed code

## Quick Start

### Prerequisites
- Node.js 22.x+ (22.19+, 24.x+, or 26+)
- npm
- Optional: Hopper, Ghidra, IDA (BYO providers)

### Installation
```bash
npm install -g rea-agents
npx rea-agents setup
```

### Agent Integration
```bash
# Register MCP server (auto-detects agent type)
npx rea-agents setup

# Your agent now sees 40+ tools via MCP:
# - binary_overview, trace_references, decompile_function
# - analyze_javascript_app, extract_routes
# - capture_browser_scenario, compare_binaries
# - plus 35+ more tools
```

### Example Usage
```bash
# Analyze a JavaScript application
npx rea-agents analyze-javascript-application /path/to/app

# Inspect a native binary
npx rea-agents native-investigate /path/to/target --provider hopper

# Capture browser behavior
npx rea-agents capture-browser-scenario https://example.com
```

## Repository Structure

```
DnA-Reverse-Engineer/
├── src/                          # TypeScript application core
│   ├── domain/                   # Pure, provider-neutral semantics
│   ├── contracts/                # Caller-visible schemas
│   ├── application/              # Workflows, Evidence, Unknowns
│   ├── server/                  # MCP request translation
│   ├── cli/                     # CLI entry points
│   ├── hopper/                  # Hopper bridge adapter
│   ├── ghidra/                  # Ghidra adapter
│   ├── browser/                 # CDP/Playwright capture
│   ├── native/                  # pwntools, pwndbg adapters
│   ├── artifacts/               # Resource extraction
│   └── evaluation/              # Agent release evaluation
├── bridge/                       # Provider-side code (Python/Java)
│   ├── hopper_bridge.py         # Hopper Python adapter
│   └── ghidra/                  # Ghidra Java bridge
├── docs/                        # Comprehensive guides
│   ├── architecture.mermaid     # Component diagram
│   ├── mcp-contracts.md         # Full tool catalog (40+ tools)
│   ├── native-investigation.md  # Binary analysis guide
│   └── javascript-application-workflows.md
├── tests/                       # Vitest suite (real E2E)
├── scripts/verify-*             # Provider chain verification
└── package.json                 # Dependencies + scripts
```

## Tool Catalog (40+ MCP Tools)

### Binary Analysis
- `binary_overview` — High-level target summary
- `trace_references` — Follow function/data calls
- `disassemble_function` — Raw assembly dump
- `decompile_function` — Pseudocode generation
- `extract_strings` — String extraction and categorization

### JavaScript/Electron
- `analyze_javascript_app` — Full app structure extraction
- `trace_electron_ipc` — IPC channel mapping
- `extract_routes` — URL/router discovery
- `analyze_source_maps` — Source location recovery

### Native Investigation
- `native_investigate` — End-to-end native analysis
- `capture_runtime_behavior` — Terminal/FS observations
- `compare_binaries` — Side-by-side diff

### Browser & Web
- `inspect_web_page` — DOM/JS/structure analysis
- `capture_browser_scenario` — Full interaction capture
- `compare_web_captures` — Before/after comparison

### Artifact Extraction
- `extract_artifacts` — Resources from bundles
- `analyze_apple_bundle` — macOS app anatomy
- `reconstruct_javascript_artifact` — Directory/ASAR reconstruction

### Session & Meta
- `binary_session` — Session info, tool availability
- `discover_webmcp_tools` — Find web MCP capabilities
- `capture_network` — HAR/network capture

## Documentation

- **Official Website**: https://rea.tools/
- **Guides**: https://rea.tools/guides/
- **Showcases**: https://rea.tools/showcase/
- **MCP Contracts**: `docs/mcp-contracts.md`
- **CLI Guide**: `docs/cli.md`
- **Testing**: `docs/testing.md`

## Architecture Highlights

### Layered Design
```
Entry Points (CLI/MCP) → Composition → Application → Provider Abstractions → Host/Process
```

### Immutable Sessions
- One immutable binding per target analysis
- Snapshot isolation for thread-safe concurrent operations
- Evidence provenance with explicit unknown tracking

### Provider Agnostic
- Hopper, Ghidra, IDA are adapters, not dependencies
- BYO (Bring Your Own) providers plug in via standard interface
- Installation is caller-managed; REA never installs third-party tools

## License

MIT — See LICENSE file for details.

## Contributing

See CONTRIBUTING.md for development guidelines, testing strategy, and commit conventions.

---
*Repository: https://github.com/DeepakatDPK/DnA-Reverse-Engineer*
*Original Author: Nabil RFG (@NabilRFG12)*
