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
- [How to uninstall](#how-to-uninstall)
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
- **No inbound port is opened** on your firewall. The agent establishes a single outbound Cloudflare Tunnel to reach GestFlow.

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

- `agent.exe` — the Windows amd64 binary (~7.5 MB, statically linked)
- Release notes summarizing the changes
- A SHA-256 hash of the binary in the notes (see [Verifying a binary](#verifying-a-binary))

---

## How to install

**You do not install this manually.** GestEase administrators use a guided flow from the GestFlow web interface that generates a customized `.cmd` installer for your organization.

### Admin steps (in GestFlow web UI)

1. Log in to GestFlow as an administrator (permission `agent-management` required).
2. Navigate to **Documents → Fournisseur de stockage** in the left sidebar.
3. If your storage provider is set to "agent", the "Télécharger l'installeur" section is visible.
4. Enter the Windows folder path on your server that the agent should expose to GestFlow (for example `D:\Documents_Entreprise`). The folder will be created automatically if it does not exist yet.
5. Click **Télécharger**. A file named `GestEaseAgent-<your-org>-install.cmd` is downloaded to your machine.

The binding code embedded in the `.cmd` is valid for 24 hours, giving your IT team time to run it.

### IT / server-side steps

1. Transfer the `.cmd` file to the Windows server where you want the agent to run (email, Teams, USB — any channel).
2. On that server, locate the file (typically `Downloads`).
3. **Right-click the `.cmd` file → "Exécuter en tant qu'administrateur"** (Run as administrator).
4. Accept the UAC prompt. A console window opens and shows the installer progress.

That's it. The installer performs the following steps automatically:

- Downloads `agent.exe` from this public repository
- Creates `C:\Program Files\GestEase Agent\`
- Copies the agent binary and downloads `cloudflared.exe` next to it
- Creates your watch folder if it doesn't exist
- Writes the agent configuration with the embedded binding code
- Registers a Windows service named `GestEaseAgent` (auto-start on boot, `LocalSystem` account)
- Starts the service immediately
- The agent exchanges the binding code with GestFlow, receives an mTLS certificate, a JWT tunnel token, and (in production) a Cloudflare Tunnel token
- cloudflared starts as a child process and establishes an outbound tunnel to Cloudflare

Return to GestFlow. The agent status badge in the header turns green within 30 seconds.

The `.cmd` file can be deleted once the installer finishes.

### If SmartScreen blocks the download

Windows SmartScreen may display a warning ("Windows protected your PC") for files downloaded from the internet. In the warning dialog, click **"Plus d'info"** (More info), then **"Exécuter quand même"** (Run anyway). This is expected because the binary is not yet code-signed (see [Unsigned binary notice](#unsigned-binary-notice)).

### Troubleshooting

- **"ERROR: This installer requires Administrator privileges"** — The `.cmd` was launched without admin elevation. Right-click → Run as administrator.
- **"ERROR: start service: The service did not respond..."** — Ensure you are on agent `v0.5.2` or later. Older versions had a Windows service protocol bug. Re-download the `.cmd` from GestFlow to get the latest agent version.
- **Badge stays DOWN after install** — The agent pushes a heartbeat to GestFlow every 60 seconds (`v0.5.2`+). On first install, allow up to 60 seconds for the first beat. If still DOWN after that: check the Windows service `GestEaseAgent` is running (`Get-Service GestEaseAgent`), check the `cloudflared` subprocess is alive (`Get-Process cloudflared`). If both are running, generate a new binding code in GestFlow and re-install.

---

## How to uninstall

Open **PowerShell as Administrator** and run:

```powershell
& "C:\Program Files\GestEase Agent\agent.exe" --uninstall
Remove-Item -Recurse -Force "C:\Program Files\GestEase Agent"
```

The first command stops the service, deletes the Windows service registration, and removes most of the install directory. The second command cleans up the binary itself (Windows locks `agent.exe` while the uninstaller process is running, so the file must be removed separately after).

Your configured watch folder (the one holding your documents) is **never** touched by the uninstaller.

---

## System requirements

| Item | Minimum |
|------|---------|
| Operating system | Windows Server 2019 / 2022, or Windows 10 / 11 (Pro or Enterprise) |
| Architecture | x86_64 (amd64) |
| Disk space | ~100 MB for the install directory, plus whatever space your exposed folder uses |
| Memory | ~50 MB resident for the agent process, ~20 MB for cloudflared |
| Network | Outbound HTTPS (443) to: |
| | - `api.gestease.ca` (GestFlow backend) |
| | - `github.com` + `objects.githubusercontent.com` (binary downloads during install) |
| | - `*.cftunnel.com` + Cloudflare edge network (tunnel connectivity) |
| Privileges | Administrator for installation (required to register the Windows service) |

---

## What the agent does

- Runs as a Windows service (`GestEaseAgent`) with automatic startup on boot.
- Listens on `127.0.0.1:5900` (loopback only) — no inbound port is exposed publicly.
- Exposes the folder you configured during install via a JSON-RPC protocol, scoped strictly to that folder (path-traversal protection enforced).
- Establishes an outbound **Cloudflare Tunnel** to GestFlow using an automatically provisioned connector token. A per-org hostname (`agent-<your-org>.gestflow.ca`) is created on Cloudflare's side automatically during pairing.
- Supervises the `cloudflared` child process: automatic restart with exponential backoff, degradation mode after burst crashes, safety shutdown after 50 crashes in 24 hours.
- Authenticates every incoming request using an mTLS client certificate and a JWT bound to your organization.
- Logs activity to `C:\Program Files\GestEase Agent\*.log` for troubleshooting.

---

## What the agent does NOT do

- **Does not read or transmit files outside of your configured folder.** Any request for a path outside of that folder is rejected at the agent level (path traversal guard).
- **Does not open any inbound port** on your public-facing network or firewall. All communication is outbound-initiated by the agent.
- **Does not share your files or metadata with any party other than GestFlow.** No telemetry, no analytics, no third-party integrations.
- **Does not run arbitrary commands** sent by the backend. The protocol is strictly limited to: `hello`, `heartbeat`, `list`, `get_file`, `put_file`, `delete_file`.
- **Does not auto-update.** Updates are released as new versions and you choose when to install them.

---

## Security

### Architecture principles

- **Zero-Tenant-Data** — files are stored on your infrastructure; GestFlow requests file bytes on demand when an authorized user of your organization takes an action that requires them (preview, download, upload).
- **Outbound-only connectivity** — no inbound port is opened on your firewall, so the agent does not expose any new attack surface.
- **mTLS + JWT** — every request from the backend is authenticated with a client certificate unique to your organization and a short-lived JWT scoped to your organization's identifier.
- **Least-privilege on disk** — the service runs under the Windows `LocalSystem` account but file access is constrained to the folder(s) you explicitly configured.
- **Integrity on writes** — `put_file` requires a SHA-256 checksum match before the file is written atomically (temp file + fsync + rename).
- **Revocation built-in** — administrators can revoke an agent at any time from the GestFlow UI; the corresponding certificate is immediately invalidated on the backend, and the Cloudflare tunnel is torn down.

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

*Last updated: 2026-04-24 — agent `v0.5.2` (heartbeat push model) validated in production*
