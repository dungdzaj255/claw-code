# Copilot Instructions for Claw Code

## Repository Overview

Claw Code is a locally-runnable coding agent CLI implemented in **Rust** with a **Python analysis/porting layer**:

- **`rust/`** — Production implementation (multi-crate workspace with CLI, runtime, tools, API client, plugins)
- **`src/`** — Python porting workspace (mirrors TypeScript architecture for analysis and parity tracking)
- **`tests/`** — Python validation suite for the porting layer

This is a **dual-implementation** repository: primary work happens in Rust; the Python layer exists for reference and architectural analysis.

---

## Build, Test, and Lint

### Rust Workspace (Primary)

**Location**: Work from `rust/` directory

```bash
# Navigate to Rust workspace
cd rust/

# Format code (required before commit)
cargo fmt --all

# Lint/clippy (must pass with no warnings)
cargo clippy --workspace --all-targets -- -D warnings

# Type check
cargo check --workspace

# Run all tests
cargo test --workspace

# Run tests from a specific crate
cargo test -p runtime  # Example: test just the runtime crate

# Build for release
cargo build --release

# Build just the CLI binary
cargo build --release -p claw-cli
```

**Pre-PR Verification Checklist** (from `CONTRIBUTING.md`):
```bash
cargo fmt --all --check
cargo clippy --workspace --all-targets -- -D warnings
cargo check --workspace
cargo test --workspace
```

All four must pass locally before opening a pull request.

### Python Workspace (Analysis/Porting)

**Location**: Run from repository root

```bash
# Run all Python tests
python3 -m unittest discover -s tests -v

# Run specific test file
python3 -m unittest tests.test_porting_workspace -v

# Run specific test method
python3 -m unittest tests.test_porting_workspace.PortingWorkspaceTests.test_manifest_counts_python_files -v

# Generate Python workspace summary
python3 -m src.main summary

# List Python modules
python3 -m src.main subsystems --limit 20

# Audit parity against original TypeScript (if archive available)
python3 -m src.main parity-audit
```

---

## High-Level Architecture

### Core Design Pattern: Generic Trait-Based Runtime

The conversation engine uses a generic trait pattern that decouples concerns:

```rust
pub struct ConversationRuntime<C: ApiClient, T: ToolExecutor> {
    session: Session,
    api_client: C,
    tool_executor: T,
    permission_policy: PermissionPolicy,
    system_prompt: Vec<String>,
    hook_runner: HookRunner,
}
```

This allows:
- **API Client** (`C`) to be swapped (Anthropic, Grok, OpenAI-compatible)
- **Tool Executor** (`T`) to be mocked for testing
- **Unified turn loop** that works with any provider implementation

**Key Files**:
- `crates/runtime/src/conversation.rs` — Turn execution loop with tool handling
- `crates/api/src/client.rs` — Provider abstraction (Anthropic, XAI, OpenAI-compat)
- `crates/tools/src/lib.rs` — Tool registry and specification

### Tool Execution Flow

Every tool execution follows this pipeline:

1. **User Input** → Conversation message
2. **API Call** → Stream events (TextDelta, ToolUse, Usage, MessageStop)
3. **Permission Check** → Evaluate against mode (ReadOnly / WorkspaceWrite / DangerFullAccess)
4. **PreToolUse Hook** → Optional middleware (can deny or mutate)
5. **Tool Execute** → Call ToolExecutor implementation
6. **PostToolUse Hook** → Optional result processing (can deny or append feedback)
7. **Tool Result Message** → Append to session
8. **Loop** → If assistant produced more tool uses, iterate; else break

**Key Files**:
- `crates/runtime/src/permissions.rs` — Permission policy evaluation
- `crates/runtime/src/hooks.rs` — Hook infrastructure
- Tool implementations in `crates/tools/src/`, `crates/claw-cli/src/` (bash, files, search, web, etc.)

### Provider Abstraction

The API layer supports multiple providers through a unified interface:

- **Anthropic** — Primary (env: `ANTHROPIC_API_KEY`, optional `ANTHROPIC_BASE_URL`)
- **Grok/XAI** — Secondary (env: `XAI_API_KEY`, optional `XAI_BASE_URL`)
- **OpenAI-compatible** — Generic endpoint support
- **OAuth** — Interactive login with callback flow (port 4545)

**Key Files**:
- `crates/api/src/providers/claw_provider.rs` — Main routing
- `crates/api/src/providers/openai_compat.rs` — Generic endpoint support
- `crates/api/src/client.rs` — OAuth and token management

### Configuration and Discovery

Configuration is merged across three sources (ordered by priority):

1. **Local** (`.claw/settings.local.json`) — Machine-specific overrides
2. **Project** (`.claw.json`) — Repository-wide settings
3. **User** (`~/.claw/config.json`) — User defaults

**Key Files**:
- `crates/runtime/src/config.rs` — ConfigLoader and merge logic
- `crates/runtime/src/prompt.rs` — CLAW.md discovery for workspace context

### Session Persistence

Sessions are stored as JSON in `~/.claw/sessions/<session-id>.json` with full message history. Resumption reconstructs the session and continues the conversation naturally.

**Key Files**:
- `crates/runtime/src/session.rs` — Session serialization/deserialization
- `crates/claw-cli/src/main.rs` — Resume flow in main loop

---

## Key Conventions

### Rust Crate Organization

**Workspace Layout** (in `rust/`):
- `crates/api/` — Provider clients and streaming (do not modify lightly; affects all providers)
- `crates/runtime/` — Core engine: sessions, config, permissions, MCP, prompt construction
- `crates/tools/` — Tool specs and registry (global registry at module level)
- `crates/commands/` — Slash command definitions and handlers
- `crates/claw-cli/` — Main binary; entry point for all CLI modes
- `crates/plugins/` — Plugin loader and lifecycle (infrastructure incomplete)
- `crates/lsp/` — Language server support (helper types, not LSP client itself)
- `crates/server/` — HTTP/SSE endpoints (Axum-based; optional)
- `crates/compat-harness/` — Compatibility layer for upstream editors

### Error Handling

- Use `Result<T, RuntimeError>` for fallible operations in runtime
- Use `Result<T, ToolError>` for tool execution failures
- Include context in error messages (not just "failed", but why)
- Errors are intentionally displayed to the user; don't suppress context

**Pattern**:
```rust
fn some_operation() -> Result<String, RuntimeError> {
    operation_that_might_fail()
        .map_err(|e| RuntimeError::new(format!("operation failed: {}", e)))
}
```

### Permission Modes

Three-tier system; each tool declares its `required_permission`:

1. **ReadOnly** — No file writes, no tool execution
2. **WorkspaceWrite** — File writes in project directory only
3. **DangerFullAccess** — Unrestricted (default for local development)

When a tool is used, the policy checks: is the current mode >= required permission? If not, the tool result is a denial message.

### Hook Patterns

Hooks are **not yet executing** in the Rust runtime (parsed from config but not invoked). When hook execution is implemented:

- **PreToolUse** hooks run before tool execution (can deny, mutate input, append messages)
- **PostToolUse** hooks run after execution (can deny result, append feedback)

Hook configuration comes from `.claw.json`:
```json
{
  "hooks": {
    "pre_tool_use": ["script_name_or_command"],
    "post_tool_use": ["another_script"]
  }
}
```

### Python Workspace Conventions

- **Snapshots** in `src/reference_data/` are JSON mirrors of command/tool/MCP inventories
- **Metadata is loaded statically** — no real API calls in Python (by design)
- **Parity audit** compares Python against archived TypeScript snapshot (if present)
- **Query engine** simulates turns with metadata for testing/analysis

When working on Python changes:
- Update corresponding test in `tests/test_porting_workspace.py`
- Run `python3 -m src.main summary` to validate manifest generation
- Don't rely on external state; snapshots should be self-contained

### Workspaces Are Separate

- **Rust changes** don't require Python changes (and vice versa)
- **CLAW.md** says: `src/` and `tests/` should stay consistent with behavior changes
- Most work is in `rust/`; Python layer is mostly read-only analysis

### Provider-Specific Details

- **Default model**: `claude-opus-4-6` (in `claw-cli/src/main.rs`)
- **Default OAuth port**: 4545 (configurable, but hardcoded constant)
- **Max tokens**: Varies by model (look in `api/src/providers/`)
- **System prompt**: Loaded from `runtime/src/prompt.rs` (includes CLAW.md context)

### Testing Patterns

**Rust**:
- Unit tests live in `src/lib.rs` of each crate (or separate `tests/` directory)
- Integration tests in `crates/*/tests/`
- Run `cargo test -p <crate>` to test a single crate
- Use `#[tokio::test]` for async tests

**Python**:
- `unittest` framework in `tests/test_porting_workspace.py`
- Subprocess calls for CLI integration tests
- No external dependencies required (snapshots are embedded)

### Code Style

- Format with `cargo fmt` (no configuration needed; follows Rust standard)
- Run `cargo clippy` and fix all warnings before commit
- Workspace lint config in `Cargo.toml` forbids unsafe code globally
- Prefer focused changes over drive-by refactors (per CONTRIBUTING.md)

### Git and PR Workflow

- Branch from `main`
- Keep PRs scoped to one clear change
- Run full verification locally before requesting review:
  ```bash
  cd rust/
  cargo fmt --all --check
  cargo clippy --workspace --all-targets -- -D warnings
  cargo check --workspace
  cargo test --workspace
  ```
- Include git commit trailer: `Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>`

---

## Quick Reference

### Common Tasks

**Start interactive session**:
```bash
cd rust/
cargo run --bin claw --
```

**Test a single Rust crate**:
```bash
cd rust/
cargo test -p runtime
```

**Check for issues without running full test suite**:
```bash
cd rust/
cargo check --workspace
```

**Validate code style**:
```bash
cd rust/
cargo fmt --all && cargo clippy --workspace --all-targets -- -D warnings
```

**Rebuild after changes**:
```bash
cd rust/
cargo build
```

**Run Python parity audit**:
```bash
python3 -m src.main parity-audit
```

### Finding Code

- **Conversation loop**: `rust/crates/runtime/src/conversation.rs:run_turn()`
- **Tool registry**: `rust/crates/tools/src/lib.rs`
- **Slash commands**: `rust/crates/commands/src/lib.rs`
- **API client**: `rust/crates/api/src/client.rs`
- **Permissions**: `rust/crates/runtime/src/permissions.rs`
- **Configuration**: `rust/crates/runtime/src/config.rs`
- **Main CLI entry**: `rust/crates/claw-cli/src/main.rs`

---

## Important Notes

- **macOS/Linux only** — Windows support not yet verified (CI runs Ubuntu + macOS)
- **Source-build distribution** — No prebuilt binaries published yet; users must `cargo build --release`
- **Plugin system incomplete** — Infrastructure exists but marketplace/lifecycle features not yet implemented
- **Hooks parsed but not executed** — Config is loaded; execution pipeline incomplete
- **Python is analysis-only** — Primary product is the Rust binary; Python layer is for reference and testing

