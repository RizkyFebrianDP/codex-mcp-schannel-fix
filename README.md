# Codex CLI Windows Schannel TLS Workaround (`CRYPT_E_REVOKED`)

A documented solution and workaround guide for **Codex CLI on Windows** when connecting to remote/enterprise Model Context Protocol (MCP) servers fails due to Windows Schannel TLS revocation errors (`CRYPT_E_REVOKED`).

---

## 1. Problem Overview (The Bug / Limitation)

When configuring a remote MCP server in Codex CLI (`~/.codex/config.toml`) using direct HTTP/SSE URLs:

```toml
[mcp_servers.my_remote_server]
enabled = true
url = "https://mcp.internal-company.com/v1/gateway/mcp-server"

[mcp_servers.my_remote_server.http_headers]
Authorization = "Bearer <token>"
```

On Windows, Codex CLI's native TLS stack fails to establish a TLS handshake and terminates the connection with:

```text
TLS connection rejected by Windows Schannel: CRYPT_E_REVOKED — certificate is revoked.
Native MCP tools do not appear in this session.
```

### Why this happens:
1. **Windows Schannel Enforcement**: The Windows build of Codex CLI utilizes the OS-native TLS provider (*Windows Schannel / CryptoAPI*).
2. **Strict Revocation Checking**: Windows Schannel strictly verifies Certificate Revocation Lists (CRL) and OCSP responders. In enterprise, re-issued certificates, internal CA setups, or networks with offline CRL endpoints, Schannel immediately rejects the certificate with `CRYPT_E_REVOKED (0x80092010)`.
3. **Lack of CLI Fallback / Options**: Unlike other developer tools (such as Git or Node.js), Codex CLI currently does not expose configuration flags to:
   - Switch TLS backends (e.g. to `rustls` using Mozilla root CAs).
   - Configure custom enterprise CA certificates.
   - Specify per-host TLS validation policies.

---

## 2. The Solution: Node.js stdio Bridge (`mcp-remote`)

Instead of letting Codex CLI connect directly over HTTP via Windows Schannel, bridge the connection through a local **Node.js** stdio process using `mcp-remote`.

Node.js uses the **OpenSSL** engine (which does not depend on Windows Schannel CRL checking) and communicates with Codex CLI via standard I/O (`stdio`). To Codex, the tool appears as a local, fully compliant MCP server.

### Architecture

```
[Codex CLI (Windows)]
        │
   (Local stdio)
        ▼
[Node.js Engine (OpenSSL)]  <-- NODE_TLS_REJECT_UNAUTHORIZED = "0"
        │
     (HTTPS)
        ▼
[Remote MCP Gateway] (e.g., Enterprise Services / Microservices)
```

---

## 3. Configuration Guide

Edit your Codex configuration file at `~/.codex/config.toml` (typically located at `C:\Users\<Username>\.codex\config.toml`).

Replace direct `url` configurations with the following bridge structure:

```toml
[mcp_servers.enterprise-mcp]
command = "C:\\Program Files\\nodejs\\node.exe"
args = [
  "C:\\Users\\<Username>\\.codex\\mcp-runtimes\\common\\node_modules\\mcp-remote\\dist\\proxy.js",
  "https://mcp.internal-company.com/v1/gateway/mcp-server",
  "--header",
  "Authorization: Bearer <YOUR_ACCESS_TOKEN>"
]
enabled = true
startup_timeout_sec = 60

[mcp_servers.enterprise-mcp.env]
NODE_TLS_REJECT_UNAUTHORIZED = "0"
```

> **Note on `mcp-remote` path**: If you do not have `mcp-remote` locally installed via an existing runtime, you can install it globally or run it via npx:
> ```bash
> npm install -g mcp-remote
> ```
> And configure `command = "mcp-remote"`.

---

## 4. Prompt Template for AI Assistants

If you use an AI coding assistant (such as Antigravity, Claude Code, Cursor, or ChatGPT) and want it to inspect and apply this workaround to your machine automatically, you can copy-paste the prompt template below:

````markdown
> **Prompt Template: Apply Windows Schannel MCP Bridge Workaround**
>
> "I am running Codex CLI on Windows and encountering a remote MCP connection error:
> `CRYPT_E_REVOKED — certificate is revoked` caused by Windows Schannel TLS verification.
>
> Please help fix my `~/.codex/config.toml` configuration by converting the remote MCP server to use a local Node.js stdio bridge (`mcp-remote`):
>
> 1. Inspect my remote MCP server configuration in `~/.codex/config.toml` (which currently uses a direct `url = "https://..."`).
> 2. Convert its configuration into a stdio bridge using Node.js and `mcp-remote` (e.g., using an existing runtime path under `.codex/mcp-runtimes/` or globally installed Node/mcp-remote).
> 3. Migrate the authorization bearer token into the `--header "Authorization: Bearer <TOKEN>"` CLI argument.
> 4. Add the environment variable `NODE_TLS_REJECT_UNAUTHORIZED = "0"` under the server's `env` table to prevent Node.js OpenSSL from rejecting the certificate.
> 5. Ensure all Windows file paths are correctly escaped using double backslashes (`\\`).
>
> Please apply this update safely without modifying any unrelated server configurations or credentials."
````

---

## 5. Supplementary Troubleshooting: Enterprise VPN & TAP Adapter

If you encounter:
```text
dial tcp: lookup <hostname>: no such host
curl: (6) Could not resolve host
```
followed by a VPN client error (e.g., OpenVPN / SSL-VPN client):
```text
SSL VPN error: All tap-windows6 adapters on this system are currently in use or disabled.
```

### Quick Resolution:
1. Press `Win + R`, type `ncpa.cpl`, and press **Enter**.
2. Locate the **TAP-Windows Adapter V9** interface.
3. Right-click and choose **Disable**, wait 3 seconds, then right-click and choose **Enable**.
4. Restart your VPN client from the system tray and reconnect.
5. Restart your Codex CLI session.

---

## 6. Verification

Once applied, launch Codex CLI and execute:
```text
check mcp connection
```

**Expected output:**
```text
MCP connection successful. Connected to remote gateway.
- Mode: LIVE
- Target Service: Online
```

---

## License
MIT
