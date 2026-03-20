# OpenFang — Agent Instructions

## Project Overview

OpenFang is an open-source **Agent Operating System** written in Rust — a full OS for autonomous AI agents, not a chatbot framework. It compiles to a single ~32MB binary.

- **Version**: 0.2.2
- **License**: Apache-2.0 OR MIT
- **Rust edition**: 2021 (MSRV 1.75)
- **Codebase**: ~144K lines of Rust across 238 source files
- **Config**: `~/.openfang/config.toml`
- **Default API**: `http://127.0.0.1:4200`
- **CLI binary**: `target/release/openfang` (or `target/debug/openfang`)
- **Repository**: https://github.com/RightNow-AI/openfang

## Workspace Structure (13 crates + xtask)

```
crates/
├── openfang-types/       # Shared types & traits — no business logic (agent, config, event, tool, message, memory types)
├── openfang-memory/      # Memory substrate: SQLite structured store, semantic search, knowledge graph
├── openfang-runtime/     # Agent execution: LLM drivers, agent loop, tool runner, WASM sandbox, MCP, A2A
├── openfang-wire/        # OFP wire protocol — agent-to-agent networking over TCP (JSON-RPC framed)
├── openfang-kernel/      # Core kernel: agent lifecycle, scheduling, metering, config, event bus, RBAC
├── openfang-api/         # HTTP/WebSocket API server (Axum) — routes, middleware, dashboard SPA
├── openfang-cli/         # CLI binary (clap) — user-facing commands, interactive TUI (ratatui)
├── openfang-channels/    # 40+ messaging integrations (Telegram, Discord, Slack, Email, Matrix, etc.)
├── openfang-skills/      # Skill system — pluggable tool bundles (TOML+Python, TOML+WASM, OpenClaw compat)
├── openfang-hands/       # Hands — pre-built autonomous agent packages (7 bundled)
├── openfang-extensions/  # Integration system — 25 MCP server templates, credential vault, OAuth2 PKCE
├── openfang-migrate/     # Migration engine — import from OpenClaw, LangChain, AutoGPT
├── openfang-desktop/     # Native desktop app (Tauri 2.0) — requires GTK/WebKit system deps
└── xtask/                # Build automation (placeholder)
```

### Dependency Flow

```
openfang-types  (foundation — no internal deps)
    ↓
openfang-memory, openfang-wire, openfang-skills, openfang-hands, openfang-extensions, openfang-migrate
    ↓
openfang-runtime  (depends on types, memory)
    ↓
openfang-kernel  (depends on types, memory, runtime, wire, skills, hands, extensions)
    ↓
openfang-channels  (depends on types, kernel)
    ↓
openfang-api  (depends on kernel, runtime, wire, channels, types)
    ↓
openfang-cli, openfang-desktop  (top-level binaries)
```

## Build & Verify Workflow

After every feature implementation, run ALL THREE checks:
```bash
cargo build --workspace --lib          # Must compile (use --lib if exe is locked)
cargo test --workspace                 # All tests must pass (1,767+)
cargo clippy --workspace --all-targets -- -D warnings  # Zero warnings
```

If the `openfang-desktop` crate fails due to missing GTK/WebKit system deps, exclude it:
```bash
cargo build --workspace --lib --exclude openfang-desktop
cargo test --workspace --exclude openfang-desktop
cargo clippy --workspace --all-targets --exclude openfang-desktop -- -D warnings
```

Additional CI checks (run by GitHub Actions on PRs to `main`):
```bash
cargo fmt --check       # Formatting
cargo audit             # Security audit
```

## Key Architecture Concepts

### KernelHandle Trait
Defined in `openfang-runtime/src/kernel_handle.rs`. Allows the runtime to call back into the kernel (spawn, send, list, kill agents, shared memory) without circular dependencies. The kernel implements this trait and passes it into the agent loop.

### AppState
Defined in `openfang-api/src/routes.rs`. Bridges the kernel to HTTP route handlers:
```rust
pub struct AppState {
    pub kernel: Arc<OpenFangKernel>,
    pub started_at: Instant,
    pub peer_registry: Option<Arc<PeerRegistry>>,
    pub bridge_manager: tokio::sync::Mutex<Option<BridgeManager>>,
    pub channels_config: tokio::sync::RwLock<ChannelsConfig>,
    pub shutdown_notify: Arc<tokio::sync::Notify>,
}
```

### LLM Drivers
Located in `openfang-runtime/src/drivers/`. Support 25+ providers through 4 driver implementations:
- `anthropic.rs` — Anthropic Claude (native API)
- `gemini.rs` — Google Gemini (native API)
- `openai.rs` — OpenAI + all compatible providers (Groq, OpenRouter, DeepSeek, Together, Mistral, Fireworks, Ollama, vLLM, etc.)
- `copilot.rs` — GitHub Copilot
- `claude_code.rs` — Claude Code integration
- `fallback.rs` — Fallback/retry logic

### Memory Substrate
Three storage backends behind a unified `Memory` trait:
- **Structured store** (SQLite): Key-value pairs, sessions, agent state
- **Semantic store**: Text search (Phase 1: LIKE matching, Phase 2: vector embeddings)
- **Knowledge graph** (SQLite): Entities and relations

### Channels
40+ messaging platform integrations in `openfang-channels/`. Each channel converts platform messages into unified `ChannelMessage` events. Includes: Discord, Telegram, Slack, Email (SMTP+IMAP), Matrix, IRC, WhatsApp, Teams, Mastodon, Bluesky, Line, Feishu, and many more.

### Hands
Pre-built autonomous agent packages in `openfang-hands/`. Unlike regular agents (you chat with them), Hands work *for* you on schedules. Each bundles a HAND.toml manifest, system prompt, SKILL.md expertise reference, and guardrails.

### Wire Protocol (OFP)
Agent-to-agent networking in `openfang-wire/`. Provides cross-machine agent discovery, authentication, and communication over TCP using JSON-RPC framing. Key types: `PeerNode`, `PeerRegistry`, `WireMessage`.

### Extensions
One-click integration system in `openfang-extensions/`. 25 bundled MCP server templates, AES-256-GCM encrypted credential vault, OAuth2 PKCE flows for Google/GitHub/Microsoft/Slack.

## CLI Commands

The CLI (`openfang-cli`) provides these commands:

| Command | Description |
|---------|-------------|
| `init` | Initialize OpenFang (`~/.openfang/` + default config) |
| `start` | Start the kernel daemon (API server + kernel) |
| `stop` | Stop the running daemon |
| `agent` | Manage agents (new, list, chat, kill, spawn) |
| `workflow` | Manage workflows (list, create, run) |
| `trigger` | Manage event triggers (list, create, delete) |
| `skill` | Manage skills (install, list, search, create, remove) |
| `channel` | Manage channel integrations (setup, test, enable, disable) |
| `config` | Show or edit configuration |
| `chat` | Quick chat with the default agent |
| `status` | Show kernel status |
| `doctor` | Run diagnostic health checks |
| `dashboard` | Open the web dashboard in browser |
| `mcp` | Start MCP server over stdio |
| `add` / `remove` | Add/remove integrations (one-click MCP setup) |
| `vault` | Manage the credential vault |
| `models` | Browse models, aliases, and providers |
| `cron` | Manage scheduled jobs |
| `sessions` | List conversation sessions |
| `logs` | Tail the OpenFang log file |
| `tui` | Launch interactive terminal dashboard |
| `approvals` | Manage execution approvals |

## Adding New Features

### New API Endpoint
1. Implement the handler function in `crates/openfang-api/src/routes.rs`
2. Register the route in `crates/openfang-api/src/server.rs` (in the `build_router` function)
3. Add request/response types in `crates/openfang-api/src/types.rs` if needed

### New Config Field
1. Add the field to `KernelConfig` struct in `crates/openfang-types/src/config.rs`
2. Add `#[serde(default)]` attribute (or `#[serde(default = "fn_name")]`)
3. Add the field to the `Default` impl for `KernelConfig` — **build will fail if you forget this**
4. Add `Serialize` + `Deserialize` derives if using new types

### New Dashboard Tab
The dashboard is an Alpine.js SPA in `crates/openfang-api/static/index_body.html`. New tabs require:
1. HTML template for the tab content
2. JS data properties and methods in the Alpine.js component

### New Channel Integration
1. Create a new module in `crates/openfang-channels/src/`
2. Implement the channel bridge trait to convert platform messages to `ChannelMessage`
3. Register in `crates/openfang-channels/src/lib.rs`

## Key API Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/health` | GET | Basic health check |
| `/api/health/detail` | GET | Detailed health with component status |
| `/api/status` | GET | Kernel status overview |
| `/api/version` | GET | Version info |
| `/api/agents` | GET/POST | List / spawn agents |
| `/api/agents/{id}` | GET | Get single agent details |
| `/api/agents/{id}/message` | POST | Send message (triggers LLM call) |
| `/api/agents/{id}/stream` | POST | Send message with SSE streaming |
| `/api/agents/{id}/session` | GET | Get agent conversation history |
| `/api/agents/{id}/ws` | GET | WebSocket connection to agent |
| `/api/agents/{id}/kill` | POST | Kill an agent |
| `/api/agents/{id}/kv` | GET | Agent key-value store |
| `/api/workflows` | GET/POST | List / create workflows |
| `/api/workflows/{id}/run` | POST | Execute a workflow |
| `/api/triggers` | GET/POST | List / create event triggers |
| `/api/channels` | GET | List channel integrations |
| `/api/skills` | GET | List installed skills |
| `/api/hands` | GET | List available hands |
| `/api/hands/{id}` | GET | Hand details |
| `/api/models` | GET | List available models |
| `/api/providers` | GET | List LLM providers |
| `/api/config` | GET | Get current configuration |
| `/api/config/set` | POST | Update config fields |
| `/api/templates` | GET | List agent templates |
| `/api/sessions` | GET | List conversation sessions |
| `/api/security` | GET | Security status |
| `/api/usage` | GET | Usage statistics |
| `/api/logs/stream` | GET | SSE log streaming |
| `/api/peers` | GET | Connected OFP peers |
| `/api/metrics` | GET | Prometheus metrics |
| `/api/shutdown` | POST | Graceful shutdown |
| `/hooks/wake` | POST | Webhook wake endpoint |
| `/hooks/agent` | POST | Webhook agent endpoint |

## MANDATORY: Live Integration Testing

**After implementing any new endpoint, feature, or wiring change, you MUST run live integration tests.** Unit tests alone are not enough — they can pass while the feature is actually dead code. Live tests catch:
- Missing route registrations in server.rs
- Config fields not being deserialized from TOML
- Type mismatches between kernel and API layers
- Endpoints that compile but return wrong/empty data

### How to Run Live Integration Tests

#### Step 1: Stop any running daemon
```bash
ps aux | grep -i openfang
kill <pid>
sleep 3
```

#### Step 2: Build fresh release binary
```bash
cargo build --release -p openfang-cli
```

#### Step 3: Start daemon with required API keys
```bash
GROQ_API_KEY=<key> target/release/openfang start &
sleep 6
curl -s http://127.0.0.1:4200/api/health  # Verify it's up
```
The daemon command is `start` (not `daemon`).

#### Step 4: Test every new endpoint
```bash
# GET endpoints — verify they return real data, not empty/null
curl -s http://127.0.0.1:4200/api/<new-endpoint>

# POST/PUT endpoints — send real payloads
curl -s -X POST http://127.0.0.1:4200/api/<endpoint> \
  -H "Content-Type: application/json" \
  -d '{"field": "value"}'

# Verify write endpoints persist — read back after writing
curl -s -X PUT http://127.0.0.1:4200/api/<endpoint> -d '...'
curl -s http://127.0.0.1:4200/api/<endpoint>  # Should reflect the update
```

#### Step 5: Test real LLM integration
```bash
curl -s http://127.0.0.1:4200/api/agents | python3 -c "import sys,json; print(json.load(sys.stdin)[0]['id'])"
curl -s -X POST "http://127.0.0.1:4200/api/agents/<id>/message" \
  -H "Content-Type: application/json" \
  -d '{"message": "Say hello in 5 words."}'
```

#### Step 6: Cleanup
```bash
ps aux | grep -i openfang
kill <pid>
```

## Architecture Notes

- **Don't touch `openfang-cli`** — user is actively building the interactive CLI
- `KernelHandle` trait avoids circular deps between runtime and kernel
- `AppState` in `server.rs` bridges kernel to API routes
- New routes must be registered in `server.rs` router AND implemented in `routes.rs`
- Dashboard is Alpine.js SPA in `static/index_body.html` — new tabs need both HTML and JS data/methods
- Config fields need: struct field + `#[serde(default)]` + Default impl entry + Serialize/Deserialize derives
- The `build_router()` function in `server.rs` is extracted so embedders (e.g. openfang-desktop) can create the router without the full daemon lifecycle

## Common Gotchas

- Binary may be locked if daemon is running — use `--lib` flag or kill daemon first
- `PeerRegistry` is `Option<PeerRegistry>` on kernel but `Option<Arc<PeerRegistry>>` on `AppState` — wrap with `.as_ref().map(|r| Arc::new(r.clone()))`
- Config fields added to `KernelConfig` struct MUST also be added to the `Default` impl or build fails
- `AgentLoopResult` field is `.response` not `.response_text`
- CLI command to start daemon is `start` not `daemon`
- `openfang-desktop` requires GTK/WebKit system deps (`libwebkit2gtk-4.1-dev`, `libgtk-3-dev`, etc.) — exclude with `--exclude openfang-desktop` if unavailable
- The `imap-proto` crate triggers future-incompat warnings — this is a known upstream issue
- `routes.rs` is ~8,800 lines — search carefully before adding duplicate handlers
- Use `truncate_str()` from `openfang-types` for safe string slicing (never raw byte indexing)

## CI/CD

GitHub Actions workflows in `.github/workflows/`:
- **ci.yml**: Runs on push/PR to `main` — check, test, clippy, fmt, audit, secrets scan, installer smoke test (all 3 platforms: Linux, macOS, Windows)
- **release.yml**: Release automation

CI installs Tauri system deps on Linux before running workspace-wide checks.

## Key Dependencies

| Category | Crate | Purpose |
|----------|-------|---------|
| Async | `tokio` | Runtime |
| HTTP server | `axum` + `tower` + `tower-http` | API server with CORS, compression, tracing |
| HTTP client | `reqwest` (rustls) | LLM API calls |
| Database | `rusqlite` (bundled) | Memory substrate |
| Serialization | `serde` + `serde_json` + `toml` | Config & API types |
| CLI | `clap` + `clap_complete` | Argument parsing & shell completions |
| TUI | `ratatui` + `colored` | Interactive terminal dashboard |
| Security | `ed25519-dalek`, `aes-gcm`, `argon2`, `sha2` | Signing, encryption, hashing |
| WASM | `wasmtime` | Sandboxed skill execution |
| WebSocket | `tokio-tungstenite` | Discord/Slack gateway + agent WS |
| Desktop | `tauri` | Native desktop app |
| Rate limiting | `governor` | API rate limiting |
| Email | `lettre` + `imap` | SMTP send + IMAP receive |
