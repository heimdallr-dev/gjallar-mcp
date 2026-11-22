# gjallar-mcp

Connect AI assistants to [Gjallar](https://github.com/heimdallr-dev/gjallar)
through the [Model Context Protocol](https://modelcontextprotocol.io). The MCP
server itself ships in the core repo (`gjallar-mcp` image); this repo holds
client setup, the tool reference and the safety model.

## Safety model

- **An assistant can never do more than its user.** The server holds no
  credentials; it forwards the caller's own Keycloak/OIDC token to the API,
  so roles, namespace scoping (multi-tenancy) and the audit log apply exactly
  as they do for that person.
- Read tools only read. `request_patch` needs the `gjallar-admin` role and
  only *builds* a patched image; it never changes running workloads.
- SQL goes to a sandboxed, read-only DuckDB session (locked to the lake).
- Nothing leaves your network unless you point a cloud assistant at it;
  a local model plus this server stays fully on-premises.

## Tools

| Tool | What it answers |
|---|---|
| `security_summary` | Headline numbers: images, workloads, findings by band and severity, KEV, fixable, posture, licences, time to fix |
| `top_risks` | Open findings ranked by risk, with where each image runs and *why* it scored that way |
| `find_package` | Where is a package running (log4j, openssl...)? |
| `image_report` | Everything about one image digest |
| `posture_violations` | Configuration problems on running workloads (GJ-xxx and regolibrary C-xxxx) |
| `license_denials` | Packages whose licence is denied by policy |
| `analytics_report` | Curated lakehouse reports (trend, time to fix, KEV exposure, runtime effect, package inventory...) |
| `analytics_query` | One read-only SQL query over the lakehouse |
| `request_patch` | Queue a Copa patch for an image (admin only) |

## Connect a client

Get a token: `gjallar login https://keycloak.example.com/realms/gjallar`
(device flow), then `export GJALLAR_TOKEN=$(jq -r .access_token ~/.config/gjallar/token.json)`.

| Client | Config |
|---|---|
| Claude Code | `claude mcp add --transport http gjallar https://gjallar-mcp.example.com/mcp --header "Authorization: Bearer $GJALLAR_TOKEN"`, or project file [`clients/claude-code.mcp.json`](clients/claude-code.mcp.json) as `.mcp.json` |
| VS Code (Copilot agent) | [`clients/vscode.mcp.json`](clients/vscode.mcp.json) as `.vscode/mcp.json` (asks for the token, stores it as a secret) |
| Cursor | [`clients/cursor.mcp.json`](clients/cursor.mcp.json) as `~/.cursor/mcp.json` |
| Claude Desktop | [`clients/claude-desktop.json`](clients/claude-desktop.json) (stdio bridge via `mcp-remote`, pinned to a release older than two weeks) |
| Anything else | Streamable HTTP at `POST /mcp`; OAuth protected-resource metadata at `/.well-known/oauth-protected-resource` |

## Verified

2026-11-22, Claude Code 2.1 against a local Gjallar (real binaries, NATS,
Postgres), tools restricted to `mcp__gjallar__*`. Asked for the most urgent
vulnerability, it answered from `top_risks`:

> CVE-2023-4863 in libwebp7 1.2.4 (deb), image harbor.local/shop/web.
> Risk 70 (urgent): CVSS 8.8, EPSS 0.94, fix in 1.2.4-0.2+deb12u1.
> Why: on CISA's known-exploited list and a fix is available; not seen running
> in any workload, so no runtime or exposure boost.

Token forwarding and role checks are covered by the core repo's MCP and e2e tests.

Apache-2.0.
