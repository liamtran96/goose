# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

goose is an open-source AI agent with a native desktop app (Electron), CLI, and REST/WebSocket API backend. Built in Rust with a React/TypeScript frontend. Supports 15+ LLM providers and 70+ extensions via Model Context Protocol (MCP).

## Build & Development

### Prerequisites

Activate Hermit to get Rust, Node, pnpm, just, and other tools:
```bash
source bin/activate-hermit
```

### Common Commands

```bash
# Build
cargo build                    # Debug build
cargo build --release          # Release build

# Test
cargo test                     # Run all Rust tests
cargo test --jobs 2            # Run tests (CI uses --jobs 2)
cargo test -p goose            # Test a specific crate
cargo test -p goose --test mcp_integration_test  # Run a specific test file
cargo test test_name           # Run a specific test by name

# Lint & Format
cargo fmt --all                # Format Rust code
cargo clippy --all-targets -- -D warnings  # Lint (treats warnings as errors)

# Full pre-commit check (Rust + UI + OpenAPI)
just check-everything

# Server
cargo run -p goose-server --bin goosed agent  # Run the goosed server
just run-server                               # Same via just

# CLI
cargo run -p goose-cli -- session             # Run CLI session
cargo run -p goose-cli -- configure           # Configure providers

# OpenAPI
just generate-openapi          # Regenerate schema + TypeScript client

# UI (Electron desktop app)
just run-ui                    # Build release + start Electron app
cd ui/desktop && pnpm install  # Install frontend deps
cd ui/desktop && pnpm run lint:check  # Lint frontend (ESLint + Prettier)
cd ui/desktop && pnpm run test:run    # Run frontend tests (Vitest)
```

### MCP Replay Tests

Record MCP integration test fixtures:
```bash
just build-test-tools
GOOSE_RECORD_MCP=1 cargo test --package goose --test mcp_integration_test
```

## Architecture

### Rust Workspace (9 crates under `crates/`)

- **goose** - Core library: agent loop, providers, session management, context management, extensions, tool execution. This is where most business logic lives.
- **goose-server** - Axum HTTP/WebSocket server (`goosed` binary). Routes live in `src/routes/`. Generates OpenAPI schema via utoipa.
- **goose-cli** - CLI binary (`goose`). Commands in `src/commands/`. Uses clap for parsing, cliclack for interactive UI.
- **goose-mcp** - Built-in MCP servers: autovisualiser, computercontroller, memory, tutorial, peekaboo (macOS only).
- **goose-acp** - Agent Client Protocol server implementation. Bridges ACP protocol to the goose agent.
- **goose-sdk** - Lightweight Rust SDK for talking to goose over ACP.
- **goose-acp-macros** - Proc macros for ACP protocol.
- **goose-test-support** - Test fixtures and MCP test servers.
- **goose-test** - Benchmarking/capture tools.

### Frontend (`ui/desktop/`)

Electron 41 + React 19 + TypeScript + TailwindCSS 4. Uses Vite for bundling. The TypeScript API client is auto-generated from the backend's OpenAPI schema (`ui/desktop/openapi.json`). After changing server routes, run `just generate-openapi` to regenerate.

### Key Architectural Patterns

- **Provider system** (`crates/goose/src/providers/`): Unified trait-based interface for 50+ LLM providers. `base.rs` defines the common provider interface. Each provider implements completion/streaming.
- **Agent loop** (`crates/goose/src/agents/`): Processes user input through sessions, invokes tools via MCP/ACP extensions, streams responses back.
- **Session persistence**: SQLite via sqlx with migrations. Chat history, session state, and extension data stored locally.
- **Dual protocol stack**: MCP for external extensions, ACP for internal agent communication.
- **Error handling**: Use `anyhow::Result` throughout. No `unwrap()` in production code.
- **Async runtime**: tokio. Avoid blocking operations in async contexts.
- **TLS**: rustls by default, native-tls available via features.

### Data Flow

```
Frontend (Electron/React) --HTTP/WebSocket--> goosed (Axum server)
  --> Agent Engine --> Session/Thread Manager
  --> LLM Providers (Anthropic, OpenAI, Google, etc.)
  --> Extensions (MCP/ACP servers) --> Tool execution
  --> Response streaming back to frontend
```

## Code Conventions

- Rust edition 2021, toolchain 1.92
- Clippy configured with `uninlined_format_args = "allow"` and `string_slice = "warn"`
- Clippy max function lines: 200 (`clippy.toml`)
- Always refer to the project as "goose" (lowercase), never "Goose"
- Commits follow [Conventional Commits](https://www.conventionalcommits.org/) format
- Commits require DCO sign-off (`git commit -s`)
- Documentation changes (`/documentation`) should not be bundled with code changes (different deploy cadence)

## CI Pipeline

Runs on GitHub Actions (`.github/workflows/ci.yml`):
1. `cargo fmt --check` - Formatting
2. `cargo test --jobs 2` - Tests
3. `cargo clippy --all-targets -- -D warnings` - Linting
4. `just check-openapi-schema` - OpenAPI schema validation
5. `pnpm run lint:check` + `pnpm run test:run` in `ui/desktop/` - Frontend checks

System deps installed by CI: libdbus, gnome-keyring, libxcb.

## Feature Flags

Key cargo features on the `goose` crate:
- `local-inference` - Whisper transcription + local model support (optional CUDA)
- `aws-providers` - AWS Bedrock/SageMaker providers
- `telemetry` - PostHog analytics
- `otel` - OpenTelemetry tracing
- `code-mode` - IDE integration mode
