# GestEase Agent

[![Latest release](https://img.shields.io/github/v/release/GestEase/gestease-agent-dist?label=latest)](https://github.com/GestEase/gestease-agent-dist/releases/latest)

**Official Windows binaries for the GestEase Local Agent** — the on-premise component that securely exposes your organization's documents to the GestFlow SaaS platform, without your files ever leaving your infrastructure.

> Published and maintained by **GestEase Technology Inc.** — https://gestease.ca

---

## Table of contents

- [What is this?](#what-is-this)
- [Is this the official agent?](#is-this-the-official-agent)
- [Latest release](#latest-release)
- [How to install](#how-to-install)
- [System requirements](#system-requirements)
- [What the agent does](#what-the-agent-does)
- [What the agent does NOT do](#what-the-agent-does-not-do)
- [Security](#security)
- [Verifying a binary](#verifying-a-binary)
- [Unsigned binary notice](#unsigned-binary-notice)
- [Support](#support)
- [License](#license)

---

## What is this?

GestEase Agent is a lightweight Windows service that runs on a server you own. It acts as a secure bridge between your local file storage and the GestFlow SaaS platform, so that:

- **Your documents stay on your infrastructure.** GestFlow never stores or caches your files.
- **Your users work as usual** through the GestFlow web interface; the agent serves file contents on demand, only when an authorized user of your organization requests them.
- **No inbound port is opened** on your firewall. The agent establishes a single outbound tunnel (Cloudflare Tunnel) to reach GestFlow.

This approach is known as **Zero-Tenant-Data**: GestEase operates your software, but never holds your data.

---

## Is this the official agent?

**Yes.** This repository is the authoritative distribution channel for the GestEase Agent, operated by GestEase Technology Inc.

- Binaries are built automatically by GitHub Actions from our private source repository. The build is reproducible, uses pinned toolchain versions, and runs our full test suite before publication.
- The agent download URL always follows this pattern:

  ```
  https://github.com/GestEase/gestease-agent-dist/releases/download/<version>/agent.exe
  ```

- If you were directed to a different URL that claims to distribute the "GestEase Agent", **do not install it**. Report it to `security@gestease.ca`.

---

## Latest release

See the [Releases page](https://github.com/GestEase/gestease-agent-dist/releases) for version history and download links. Each release includes:

- `agent.exe` — the Windows amd64 binary
- Release notes summarizing the changes
- A SHA-256 hash of the binary in the notes (see [Verifying a binary](#verifying-a-binary))

---

## How to install

**You do not install this manually.** GestEase administrators use a guided flow from the GestFlow web interface that generates a customized PowerShell installer for your organization.

### Admin steps

1. Log in to GestFlow as an administrator (permission `agent-management` required).
2. Navigate to **Documents → Fournisseur de stockage** in the left sidebar.
3. If your storage provider is set to "agent", the "Télécharger l'installeur" section is visible.
4. Enter the Windows folder path on your server that the agent should expose to GestFlow (for example `D:\Documents_Entreprise`). The folder will be created automatically if it does not exist yet.
5. Click **Télécharger**. A file named `GestEaseAgent-<your-org>-install.ps1` is downloaded to your machine.

### IT / server-side steps

1. Transfer the `.ps1` file to the Windows server where you want the agent to run (email, Teams, USB — any channel).
2. Open **PowerShell as Administrator** on that server.
3. Navigate to the folder containing the `.ps1` file (for example `cd C:\Users\<you>\Downloads`).
4. Run:

   ```powershell
   powershell -ExecutionPolicy Bypass -File .\GestEaseAgent-<your-org>-install.ps1
   ```

   If Windows SmartScreen has blocked the file (common for files downloaded from the internet), first run:

   ```powershell
   Unblock-File .\GestEaseAgent-<your-org>-install.ps1
   ```

5. The installer will:
   - Create `C:\Program Files\GestEase Agent\`
   - Download `agent.exe` and `cloudflared.exe`
   - Write the configuration file with the embedded binding code
   - Register a Windows service named `GestEaseAgent` (auto-start on boot)
   - Start the service immediately

6. Return to GestFlow. The agent status badge in the header will turn green within 30 seconds.

The `.ps1` file can be deleted once the installer finishes.

---

## System requirements

| Item | Minimum |
|------|---------|
| Operating system | Windows Server 2019 / 2022, or Windows 10 / 11 (Pro or Enterprise) |
| Architecture | x86_64 (amd64) |
| PowerShell | 5.1 or later |
| Disk space | ~100 MB for the install directory, plus whatever space your exposed folder uses |
| Memory | ~50 MB resident for the agent process |
| Network | Outbound HTTPS (443) to: |
| | - `api.gestease.ca` (GestFlow backend) |
| | - `github.com` + `objects.githubusercontent.com` (binary downloads during install) |
| | - Cloudflare edge network (tunnel connectivity, standard HTTPS) |
| Privileges | Administrator for installation (required to register the Windows service) |

---

## What the agent does

- Runs as a Windows service (`GestEaseAgent`) with automatic startup on boot.
- Listens on `127.0.0.1:5900` (loopback only).
- Exposes the folder you configured during install via a JSON-RPC protocol, scoped strictly to that folder (path-traversal protection enforced).
- Establishes an outbound Cloudflare Tunnel to GestFlow using an automatically provisioned connector token.
- Authenticates every incoming request using an mTLS client certificate and a JWT bound to your organization.
- Logs activity to the Windows Event Log and to `C:\Program Files\GestEase Agent\*.log` for troubleshooting.

---

## What the agent does NOT do

- **Does not read or transmit files outside of your configured folder.** Any request for a path outside of that folder is rejected.
- **Does not open any inbound port** on your public-facing network or firewall.
- **Does not share your files or metadata with any party other than GestFlow.** No telemetry, no analytics, no third-party integrations.
- **Does not run arbitrary commands** sent by the backend. The protocol is strictly limited to: list directory, get file, put file, delete file.
- **Does not auto-update.** Updates are released as new versions and you choose when to install them.

---

## Security

### Architecture principles

- **Zero-Tenant-Data** — files are stored on your infrastructure; GestFlow requests file bytes on demand when an authorized user of your organization takes an action that requires them (preview, download, upload).
- **Outbound-only connectivity** — no inbound port is opened on your firewall, so the agent does not expose any new attack surface.
- **mTLS and JWT** — every request from the backend is authenticated with a client certificate unique to your organization and a short-lived JWT scoped to your organization's identifier.
- **Least-privilege on disk** — the service runs under the Windows `LocalSystem` account but file access is constrained to the folder(s) you explicitly configured.
- **Revocation built-in** — administrators can revoke an agent at any time from the GestFlow UI; the corresponding certificate is immediately invalidated on the backend.

### Reporting a vulnerability

See [SECURITY.md](SECURITY.md).

---

## Verifying a binary

Each release notes page includes a SHA-256 hash of `agent.exe`. To verify a downloaded binary, run:

```powershell
Get-FileHash agent.exe -Algorithm SHA256
```

Compare the output against the hash published in the release notes. If they do not match, do not install the binary and contact `security@gestease.ca`.

---

## Unsigned binary notice

Current releases are not code-signed. On first run, Windows SmartScreen may display a warning such as "Windows protected your PC" or "Publisher unknown". This is expected for the current phase of the product. You can verify authenticity via the SHA-256 hash as described above.

A code-signing certificate is on our roadmap. Once in place, future releases will be signed and the warning will disappear.

---

## Support

| Topic | Contact |
|-------|---------|
| Installation help, feature requests, bug reports | In-app chat on https://gestflow.ca (fastest) or `support@gestease.ca` |
| Security vulnerabilities | `security@gestease.ca` — see [SECURITY.md](SECURITY.md) |
| Licensing and commercial questions | `contact@gestease.ca` |

This repository does not accept public issues or pull requests. The GestEase Agent source code is proprietary and maintained in a private repository.

---

## License

Copyright © 2026 GestEase Technology Inc. All rights reserved.

See [LICENSE](LICENSE) for the full license text. The binaries in this repository are licensed exclusively to customers of the GestFlow SaaS platform under an active subscription.

---

*Last updated: 2026-04-23*
