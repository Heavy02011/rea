# Security Policy

## Supported versions

Security fixes are provided for the latest published version of REA. Upgrade to the latest release before reporting a problem that may already have been resolved.

## Reporting a vulnerability

Do not disclose vulnerabilities, proof-of-concept exploits, credentials,
private binaries, Hopper documents, or private Ghidra projects in a public
issue.

Use **Report a vulnerability** in the repository's **Security** tab to submit the report privately. Do not open a public issue to request a private contact channel.

Include the affected REA version, operating system/distribution, selected
provider and version, impact, reproduction conditions, and any suggested
mitigation in the private report. Remove credentials, proprietary binaries,
Hopper documents, Ghidra project contents, socket capability tokens, and
unrelated local paths from logs or examples.

The maintainer will acknowledge the report, assess its scope, and coordinate
remediation and disclosure timing with the reporter. Please allow time for
provider-dependent behavior to be reproduced safely.

## Security boundary

REA authenticates each local provider bridge session with a random capability
token and a current-user Unix socket. Tokens are transferred through private
session descriptors rather than process arguments or environment variables.
This is not a sandbox and does not protect against malicious processes already
running as the same operating-system user. Opening an untrusted binary
delegates parsing and analysis to the selected local provider with that user's
permissions.

Ghidra sessions use a packaged Java `HeadlessScript`, an isolated temporary
project, and private home, cache, and runtime directories. REA imports only the
requested target, bounds startup and protocol messages, requests bounded CPU
and heap settings, and deletes the temporary project when the session closes.
These controls limit accidental persistence and resource use; they do not make
Ghidra a sandbox. REA does not open or modify user-owned Ghidra projects.

`GHIDRA_INSTALL_DIR` and optional `JAVA_HOME` identify user-managed
installations. Setup can add those values to detected client configuration only
after showing its plan and receiving approval. REA does not download, install,
upgrade, or modify Ghidra or Java.

On Linux, REA accepts release metadata only from Hopper's HTTPS endpoint, restricts package URLs to Hopper's public download origin, bounds the package size, and compares the downloaded bytes with Hopper's published checksum before requesting installation. The published SHA-1 value is a corruption check, not a modern package signature; HTTPS origin validation remains part of the trust boundary. Dependency installation is delegated without shell evaluation to `apt-get`, `dnf`, or `pacman`, directly as root or through `pkexec`. REA never invokes `sudo`.

## Security review findings

This repository was reviewed for concrete security concerns in the package installation flow, local bridge execution model, and reverse-engineering boundary assumptions. Dependency review is clean at the current lockfile: `npm audit --json --omit=dev` returned zero known vulnerabilities, but the following design-level issues remain important for operators and maintainers.

### 1) High: Local analysis of untrusted binaries is not sandboxed

Evidence:
- `SECURITY.md` explicitly states: "This is not a sandbox ... Opening an untrusted binary delegates parsing and analysis to the selected local provider with that user's permissions."
- The same requirement is reflected in the project documentation for platform safety: REA intentionally executes provider-bound analysis using the current OS user and does not claim process isolation.

Impact:
- A malicious binary processed through REA can access the local filesystem, installed credentials, and network paths available to the current user.
- The risk is not theoretical: the tool is designed to load and inspect user-selected binaries locally, and the project documents that analysis runs under the caller's identity rather than in a protected sandbox.

Recommended mitigation:
- Add an explicit "unsafe binary" or "untrusted target" mode that requires a sandbox/VM/container or restricted account.
- Offer Linux namespaces, macOS Seatbelt, or a disposable VM workflow for untrusted inputs before enabling full local analysis.
- Present a strong warning before any analysis of arbitrary binaries and require an opt-in for high-risk targets.

### 2) Medium: Same-user process isolation is weak even though the local socket is authenticated

Evidence:
- `bridge/hopper_bridge.py` creates a Unix socket and sets it to mode `0o600`, and the bridge validates a random `REA_TOKEN` with `hmac.compare_digest(...)` before accepting requests.
- The project's own security policy states the protection is against unrelated users, not malicious code already running as the same OS user.

Impact:
- If another process is already running under the same account, it can potentially abuse the same-user trust boundary and interact with the bridge or its temporary state.
- This is a practical risk in multi-process developer environments and agent workloads where the same account runs both analysis tools and external automation.

Recommended mitigation:
- Bind sockets to dedicated per-session directories, enforce stricter ownership checks, and avoid reusing paths across sessions.
- Reduce the blast radius by limiting bridge capabilities to a single target and a timestamped session directory.
- Consider adding process-level checks for unexpected parent process ownership and session token rotation on every reconnect.

### 3) Medium: Bridge startup uses dynamic Python execution from local files

Evidence:
- `bridge/pwntools/layout.py` and `bridge/pwndbg/launch.py` both execute compiled Python source using `exec(compile(..., "exec"), ...)` from on-disk bridge files.
- The design is intentionally trusted code execution: the bridge loads a fixed sibling implementation and then executes it as the current process.

Impact:
- This is safe only if the bridge files and their directories are protected from tampering. A compromised or modified local file under the repo or installation path becomes direct code execution under the user account.
- Because the execution model is dynamic, local file integrity and installation provenance are critical trust boundaries.

Recommended mitigation:
- Verify bridge file hashes or signatures before executing them.
- Store generated bridge payloads in a dedicated, owner-only directory and avoid writable locations.
- Reduce the amount of code executed from arbitrary filesystem paths in the launch path and prefer immutable packaged artifacts.

## Conclusion

REA's current security posture is best described as a powerful local reverse-engineering tool with a deliberately narrow trust model: it protects against unrelated local users and transport tampering, but it does not isolate untrusted binaries from the caller's OS privileges. This is an acceptable tradeoff for a developer tool, but it should be documented and enforced as a high-risk workflow requiring explicit confirmation and, where possible, a sandboxed or isolated execution environment.
