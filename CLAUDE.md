# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Install dependencies
uv sync --extra dev --frozen

# Run all tests (excluding manual integration tests)
uv run pytest

# Run a single test file
uv run pytest tests/path/to/test_file.py

# Run tests excluding integration tests (requires real Google APIs)
uv run pytest -m "not integration"

# Lint and format
uv run ruff check
uv run ruff format

# Auto-fix lint and formatting
uv run ruff check --fix && uv run ruff format

# Run the server (stdio mode, all tools)
uv run main.py

# Run with specific services only
uv run main.py --tools gmail drive calendar

# Run with a tool tier
uv run main.py --tool-tier core   # core | extended | complete

# Run in HTTP transport mode
uv run main.py --transport streamable-http

# Run in read-only mode
uv run main.py --read-only

# Granular per-service permissions
uv run main.py --permissions gmail:organize drive:readonly
```

## Dependency Management

`uv.lock` is the single source of truth for what gets installed. It pins every package to an exact version and includes SHA256 hashes for all sdists and wheels. `uv sync --frozen` verifies those hashes on install — a tampered package would be rejected even if the version number matched.

**Always install with `--frozen`:**
```bash
uv sync --extra dev --frozen
```
Never run `uv sync` without `--frozen` during normal development. That flag prevents uv from silently updating the lockfile.

**Treat `uv.lock` diffs as security-sensitive.** When a PR modifies `uv.lock`, review it carefully:
- Confirm the bump was intentional, not a side effect of running `uv sync` without `--frozen`
- Check the changelog of any upgraded package for breaking changes or suspicious activity
- Be especially careful with packages in the auth path: `google-auth-oauthlib`, `cryptography`, `pyjwt`, `fastmcp`

**To intentionally upgrade a dependency:**
```bash
uv lock --upgrade-package <package-name>   # upgrade one package
uv sync --extra dev --frozen               # install the updated lockfile
uv run pytest                              # verify nothing broke
```
Upgrade packages one at a time so regressions are easy to bisect. Commit the lockfile change in its own commit with a clear message explaining why the bump was made.

**Do not upgrade dependencies speculatively** (e.g. "bumping to latest" without a specific reason). Prefer stable, well-established versions over bleeding-edge releases, particularly for security-critical packages.

## Architecture

### Entry Point & Server

`main.py` is the entry point. It imports all service modules (gmail, drive, etc.) to register their tools, then launches the FastMCP server. `core/server.py` defines `SecureFastMCP` (extends `FastMCP`) with OAuth middleware, custom routes (`/`, `/health`, `/oauth2callback`), and tool schema patching for single-user mode.

### Service Modules

Each Google service lives in its own package (`gmail/`, `gdrive/`, `gcalendar/`, `gdocs/`, `gsheets/`, `gslides/`, `gchat/`, `gforms/`, `gtasks/`, `gcontacts/`, `gsearch/`, `gappsscript/`). The `*_tools.py` file in each contains `@server.tool()` decorated async functions.

### Authentication Decorator (`auth/service_decorator.py`)

The central pattern for all tools is `@require_google_service(service_type, scopes)`:

```python
@require_google_service("gmail", "gmail_read")
async def search_gmail_messages(service, user_google_email: str, query: str) -> str:
    # `service` is injected automatically; `user_google_email` identifies the user
```

This decorator:
- Removes `service` (and `user_google_email` in OAuth 2.1 mode) from the exposed tool signature
- Handles both OAuth 2.0 (file-based credentials in `~/.workspace-mcp/credentials/`) and OAuth 2.1 (multi-user session store)
- Attaches `_required_google_scopes` to the wrapped function for tool filtering

For tools needing multiple Google services, use `@require_multiple_services([...])`.

### Tool Tiers & Filtering (`core/tool_tiers.yaml`, `core/tool_registry.py`)

Tools are categorized as `core`, `extended`, or `complete` per service in `tool_tiers.yaml`. At startup, `filter_server_tools()` removes tools that don't match the active tier, mode (read-only, OAuth 2.1), or granular permission level.

### OAuth Modes

- **OAuth 2.0 (legacy/stdio):** Credentials stored per-user as JSON files. `USER_GOOGLE_EMAIL` env var enables single-user mode where the email is injected automatically.
- **OAuth 2.1 (multi-user HTTP):** Enabled with `MCP_ENABLE_OAUTH21=true`. Uses FastMCP's `GoogleProvider` and a session store (`auth/oauth21_session_store.py`). The `MCPSessionMiddleware` and `AuthInfoMiddleware` extract session context.
- **Service Account:** Enabled with `GOOGLE_SERVICE_ACCOUNT_KEY_FILE` or `GOOGLE_SERVICE_ACCOUNT_KEY_JSON` + `USER_GOOGLE_EMAIL` for domain-wide delegation.

### Key Environment Variables

| Variable | Purpose |
|---|---|
| `USER_GOOGLE_EMAIL` | Default user email for single-user/stdio mode |
| `GOOGLE_OAUTH_CLIENT_ID` / `GOOGLE_OAUTH_CLIENT_SECRET` | OAuth app credentials |
| `MCP_ENABLE_OAUTH21=true` | Enable OAuth 2.1 multi-user mode |
| `WORKSPACE_MCP_TRANSPORT` | `stdio` (default) or `streamable-http` |
| `WORKSPACE_MCP_TOOLS` | Comma-separated services to enable |
| `WORKSPACE_MCP_TOOL_TIER` | `core`, `extended`, or `complete` |

### Adding a New Tool

1. Add the async function to the relevant `*_tools.py` file with `@server.tool()` and `@require_google_service(...)`.
2. The function signature must start with `service` as the first parameter.
3. Add the tool name to the appropriate tier in `core/tool_tiers.yaml`.
4. Add required scopes using scope group names from `SCOPE_GROUPS` in `auth/service_decorator.py`.
