# Codex CLI Windows Schannel TLS Workaround (`CRYPT_E_REVOKED`)

A documented solution and workaround guide for **Codex CLI on Windows** when connecting to remote/enterprise Model Context Protocol (MCP) servers fails due to Windows Schannel TLS revocation errors (`CRYPT_E_REVOKED`).

---

## 1. Problem Overview (The Bug / Limitation)

When configuring a remote MCP server in Codex CLI (`~/.codex/config.toml`) using direct HTTP/SSE URLs:

```toml
[mcp_servers.my_remote_server]
enabled = true
url = "https://mcp.internal-company.com/v1/gateway/mcp-sap"

[mcp_servers.my_remote_server.http_headers]
Authorization = "Bearer <token>"
```

On Windows, Codex CLI's native TLS stack fails to establish a TLS handshake and terminates the connection with:

```text
Koneksi TLS masih ditolak Windows Schannel: CRYPT_E_REVOKED — sertifikat dicabut.
Tool MCP native belum muncul di sesi ini.
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
[Remote MCP Gateway] (e.g., SAP / RAG / Internal Enterprise Services)
```

---

## 3. Configuration Guide

Edit your Codex configuration file at `~/.codex/config.toml` (typically located at `C:\Users\<Username>\.codex\config.toml`).

Replace direct `url` configurations with the following bridge structure:

```toml
[mcp_servers.sap-trias]
command = "C:\\Program Files\\nodejs\\node.exe"
args = [
  "C:\\Users\\<Username>\\.codex\\mcp-runtimes\\cloudflare\\node_modules\\mcp-remote\\dist\\proxy.js",
  "https://mcp.internal-company.com/v1/gateway/mcp-sap",
  "--header",
  "Authorization: Bearer <YOUR_ACCESS_TOKEN>"
]
enabled = true
startup_timeout_sec = 60

[mcp_servers.sap-trias.env]
NODE_TLS_REJECT_UNAUTHORIZED = "0"
```

> **Note on `mcp-remote` path**: If you do not have `mcp-remote` locally installed via an existing runtime, you can install it globally or run it via npx:
> ```bash
> npm install -g mcp-remote
> ```
> And configure `command = "mcp-remote"`.

---

## 4. Supplementary Troubleshooting: Enterprise VPN & TAP Adapter

If you encounter:
```text
dial tcp: lookup <hostname>: no such host
curl: (6) Could not resolve host
```
followed by a VPN client error (e.g., **Sophos Connect**):
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

## 5. Verification

Once applied, launch Codex CLI and execute:
```text
cek koneksi mcp sap trias
```

**Expected output:**
```text
Koneksi MCP SAP Trias berhasil. Panggilan RFC_SYSTEM_INFO mendapat respons langsung dari SAP.
- Server aktif: Sandbox Build Competence
- SID: TRD
- Mode: LIVE
- SAP Release: 731
- Database / OS: Oracle / Linux
```

---

## License
MIT
