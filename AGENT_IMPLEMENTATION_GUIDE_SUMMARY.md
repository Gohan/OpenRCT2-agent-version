# AI Agent Implementation Guide - Summary

## Overview

This repository contains a comprehensive guide (**游戏AI代理化实施指南.md**) on how to implement AI agents in management/simulation games, based on the OpenRCT2-agent-version project.

The guide is written in Chinese and provides detailed technical guidance for game developers who want to integrate AI assistants (like Claude Code) into their games to enhance gameplay and interactivity.

## What This Project Does

OpenRCT2-agent-version is an experimental fork of OpenRCT2 that integrates AI capabilities:

1. **In-Game Terminal** - A fully functional VT100/xterm terminal embedded in the game window
2. **Auto-Launch Claude Code** - Automatically starts Claude Code AI assistant when terminal opens  
3. **Custom CLI Tool (rctctl)** - A kubectl-style command-line interface for game control
4. **JSON-RPC API Server** - Exposes game state and actions via JSON-RPC 2.0 protocol (port 9876)

## Key Innovations

- **AI as Co-Player**: AI plays the game with same rules as human players (no cheating)
- **CLI-Driven**: AI interacts through commands, not visual interface
- **Deep Integration**: Fork approach allows PTY, terminal emulation, and native UI
- **Extensible Architecture**: Clean separation between communication layer, CLI, and game logic

## Core Components (~15,000 lines of new code)

| Component | Files | Lines | Description |
|-----------|-------|-------|-------------|
| JSON-RPC Server | 2 | ~600 | TCP server for game communication |
| RPC Handlers | 15 | ~8,000 | Business logic for park, rides, staff, guests, etc. |
| Terminal System | 10 | ~4,200 | PTY, libvterm integration, UI window |
| rctctl CLI Tool | 30+ | ~5,000 | Standalone command-line interface |
| Tests | 5 | ~800 | Python test scripts |

## Guide Contents (游戏AI代理化实施指南.md)

The comprehensive guide covers:

1. **Project Background** - What was changed and why
2. **Architecture Analysis** - Detailed system design and data flow
3. **Core Implementation** - Deep dive into each component
4. **Universal Implementation Steps** - How to do this in YOUR game
5. **Best Practices** - API design, error handling, performance, security
6. **Platform Support** - Windows, macOS, Linux considerations
7. **FAQ** - Common questions and solutions
8. **Appendices** - Complete API reference, templates, resources

## Quick Start for Other Games

The guide provides a step-by-step process to implement similar functionality:

### Phase 1: MVP (Week 1-3)
- Implement JSON-RPC server (or similar communication layer)
- Create basic CLI tool
- Expose 5-10 core APIs
- Test with external terminal

### Phase 2: Enhancement (Week 4-6)
- Expand to 10-15 APIs  
- Add error handling and validation
- Write AI system prompt
- Performance optimization

### Phase 3: Polish (Week 7-8, Optional)
- Integrate in-game terminal
- Auto-launch AI agent
- Add UI controls
- Session logging

## Technology Stack

- **Language**: C++ (game), C++ (CLI tool)
- **Communication**: JSON-RPC 2.0 over TCP sockets
- **Terminal**: libvterm + PTY (forkpty on Unix, ConPTY on Windows)
- **JSON**: nlohmann/json
- **Build**: CMake
- **Testing**: Python + CTest

## Applicable Game Types

**Highly Suitable**:
- ⭐⭐⭐⭐⭐ Management/Tycoon games (SimCity, RCT2)
- ⭐⭐⭐⭐⭐ Strategy games (Civilization, Total War)
- ⭐⭐⭐⭐ Building/Production (Factorio, Satisfactory)
- ⭐⭐⭐⭐ Simulation (Farming Simulator, Game Dev Tycoon)

**Not Suitable**:
- ❌ Real-time action (requires millisecond reactions)
- ❌ Online competitive (could be considered cheating)

## Design Philosophy

**Core Principle**: AI should play like a player, not cheat

- ✅ AI uses same game rules and actions
- ✅ AI queries structured data via CLI (can't "see" the screen)
- ✅ AI operates with fair limitations
- ❌ No direct memory manipulation
- ❌ No access to hidden information
- ❌ No debug/cheat commands (unless player has access too)

## Architecture Diagram

```
┌─────────────────────────────────────────┐
│         OpenRCT2 Game Process           │
│                                         │
│  ┌───────────────────────────────────┐  │
│  │ AI Agent Terminal Window          │  │
│  │  • libvterm (VT100 emulation)     │  │
│  │  • PTY (pseudo-terminal)          │  │
│  └───────────┬───────────────────────┘  │
│              │ spawns                   │
│  ┌───────────▼───────────────────────┐  │
│  │ Claude Code Process               │  │
│  │  • cwd: ai-agent-workspace/       │  │
│  │  • PATH includes rctctl           │  │
│  └───────────┬───────────────────────┘  │
│              │ calls                    │
│  ┌───────────▼───────────────────────┐  │
│  │ JSON-RPC Server (localhost:9876)  │  │
│  └───────────┬───────────────────────┘  │
│              │ dispatches               │
│  ┌───────────▼───────────────────────┐  │
│  │ Handler Registry                  │  │
│  │  • park.*, ride.*, staff.*, ...   │  │
│  └───────────┬───────────────────────┘  │
│              │ accesses                 │
│  ┌───────────▼───────────────────────┐  │
│  │ Game State & Actions              │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘

External Tool:
┌──────────┐      ┌──────────────┐
│  rctctl  │─TCP─▶│  JSON-RPC    │
│  (CLI)   │      │  :9876       │
└──────────┘      └──────────────┘
```

## Example: Querying Park Status

```bash
# Human-readable output
$ rctctl park status
Park: Gohan's Park
Cash: $12,500.00
Rating: 678
Guests: 543
Date: Year 3, September 15

# Machine-readable output (for AI parsing)
$ rctctl park status -o json
{
  "name": "Gohan's Park",
  "cash": 12500.00,
  "rating": 678,
  "guests": 543,
  "date": {
    "year": 3,
    "month": 9,
    "day": 15
  }
}
```

## Key Files

### Documentation
- **游戏AI代理化实施指南.md** - Comprehensive implementation guide (Chinese, 1100+ lines)
- **CODING_AGENT.md** - Architecture overview
- **AGENTS.md** - Project background
- **RCTCTL.md** - CLI design patterns

### Implementation
- `src/openrct2/scripting/JsonRpcServer.{h,cpp}` - RPC server
- `src/openrct2/scripting/rpc/handlers/` - API handlers (13 files)
- `src/openrct2/terminal/` - Terminal subsystem (6 files)
- `src/openrct2-ui/windows/AIAgentTerminal.cpp` - Terminal UI (3220 lines!)
- `rctctl/` - CLI tool (separate project)

### AI Workspace
- `ai-agent-workspace/IN_GAME_AGENT.md` - AI system prompt (~500 lines)
- `ai-agent-workspace/auto_prompts.txt` - Auto-prompt rotation

## Building

```bash
# Install dependencies (macOS)
brew install libvterm nlohmann-json

# Configure
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Release

# Build everything (game + CLI + assets)
cmake --build build --target agent_bundle -j8

# Run
./build/OpenRCT2.app/Contents/MacOS/OpenRCT2
```

## Testing

```bash
# CLI validation tests (fast, no game required)
ctest -R rctctl_validation

# End-to-end tests (with headless game)
ctest -R agent_scenarios
```

## License

GNU General Public License v3.0 (same as OpenRCT2)

## Credits

- Based on [OpenRCT2](https://github.com/OpenRCT2/OpenRCT2)
- Inspired by "AI Plays Pokemon" experiments
- Built for use with [Claude Code](https://claude.ai)

## Contributing

This is an experimental fork. Contributions welcome, especially:
- Windows ConPTY support
- Additional RPC handlers
- Improved AI prompts
- Bug reports from AI sessions

---

**For the complete technical guide with implementation details, best practices, and step-by-step instructions for your own game, see: 游戏AI代理化实施指南.md**

