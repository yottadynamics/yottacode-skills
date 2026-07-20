---
name: security-scanner
description: Run defensive security scans for vulnerabilities, misconfigurations, secrets, licenses, and SBOMs. Uses Trivy as the default scanner for container images, filesystems, repositories, Kubernetes clusters, IaC, root filesystems, VM images, and SBOM analysis.
license: MIT
metadata:
  author: yottacode
---

# Security scanner

Use Trivy as the default engine for read-only security scanning. Prefer JSON output for agent parsing, summarize results before deep-diving individual CVEs, and never expose raw secret values found by a scan.

## 1. Confirm prerequisites and scope

Before scanning, identify the target and confirm Trivy is available:

```bash
trivy --version
trivy <target-type> --help
```

Use the installed Trivy help output as the authority for supported subcommands and flags. Trivy changes over time; do not assume every flag in this playbook is available in the user's installed version.

Ask for clarification when the target is ambiguous. Common target types:

| Target | Command | Use case |
|---|---|---|
| Image | `trivy image` | Local or remote container images. |
| Filesystem | `trivy fs` | Local projects, source trees, and lockfiles. |
| Repository | `trivy repo` | Local or remote Git repositories. |
| Config | `trivy config` | IaC such as Dockerfiles, Kubernetes YAML, Terraform, and Helm. |
| Kubernetes | `trivy k8s` | Live cluster resources and workloads. |
| SBOM | `trivy sbom` | Existing CycloneDX or SPDX SBOMs. |
| Rootfs | `trivy rootfs` | Extracted root filesystems. |
| VM | `trivy vm` | VM images, when supported by the installed Trivy version. |

## 2. Safety rules

Default to read-only scans. Do not modify images, repositories, clusters, or ignore files unless the user explicitly asks.

Require explicit confirmation before:

- Scanning a live Kubernetes cluster, especially all namespaces.
- Running long, broad scans over large directories or remote repositories.
- Writing or updating `.trivyignore`.
- Using server mode or sending scan data to a remote service.
- Running commands against production infrastructure.

For secret findings:

- Never paste raw secret values into chat.
- Report only rule ID, severity, file path, line number if available, and remediation guidance.
- If a finding appears to be a real credential, recommend rotation and removal from history as appropriate.

Use temporary output files by default. Keep reports only when the user asks for artifacts or when they are needed for CI/security review, and state the saved path.

## 3. Scanner selection

Use `--scanners` when supported to choose the checks that match the request:

- `vuln` — known CVEs in OS and language packages.
- `misconfig` — IaC and configuration issues.
- `secret` — hard-coded secrets and credentials.
- `license` — license classification and policy risk.

Useful combinations:

```bash
--scanners vuln,misconfig,secret
--scanners vuln,misconfig,secret,license
--scanners vuln,license
```

## 4. Recommended default flags

For most agent-driven scans, start with actionable high-signal output:

```bash
--severity HIGH,CRITICAL
--ignore-unfixed
--format json
--quiet
```

Add `--output <path>.json` when persisting results. Add `--exit-code 1` only when the user wants a CI gate or a failing command on findings.

Other useful flags when supported:

- `--list-all-pkgs` — include all packages in JSON, not only vulnerable packages.
- `--show-suppressed` — include findings suppressed by ignore rules.
- `--offline-scan` — avoid external calls during analysis.
- `--skip-db-update` — use the local vulnerability database.
- `--cache-dir` or `--cache-backend` — control caching for repeated scans.
- `--timeout 10m` — avoid hangs on large targets.
- `--exit-on-eol 1` — fail on end-of-life OS findings when supported.
- `--compliance <framework>` — run compliance checks when supported, such as Kubernetes or Docker benchmarks.

If the user asks for all findings, remove `--severity HIGH,CRITICAL` and consider dropping `--ignore-unfixed`.

## 5. Core workflows

### Container image scan

```bash
trivy image \
  --severity HIGH,CRITICAL \
  --ignore-unfixed \
  --scanners vuln,secret,misconfig \
  --format json \
  --output /tmp/trivy-image.json \
  <image:tag>
```

### Filesystem or project scan

```bash
trivy fs \
  --scanners vuln,misconfig,secret,license \
  --severity HIGH,CRITICAL \
  --ignore-unfixed \
  --format json \
  --output /tmp/trivy-fs.json \
  .
```

### Git repository scan

```bash
trivy repo \
  --scanners vuln,secret,misconfig \
  --severity HIGH,CRITICAL \
  --format json \
  --output /tmp/trivy-repo.json \
  <path-or-url>
```

### Config / IaC scan

```bash
trivy config \
  --severity HIGH,CRITICAL \
  --format json \
  --output /tmp/trivy-config.json \
  ./k8s/ ./Dockerfile ./terraform/
```

### Kubernetes cluster scan

Only scan live clusters after explicit confirmation and with the narrowest reasonable scope.

```bash
trivy k8s --report summary --severity HIGH,CRITICAL
```

For namespace-scoped detail when supported:

```bash
trivy k8s \
  --include-namespaces production \
  --severity HIGH,CRITICAL \
  --format json \
  --output /tmp/trivy-k8s.json \
  --report all
```

### SBOM generation and scan

Generate CycloneDX or SPDX when the user needs an artifact:

```bash
trivy image --format cyclonedx --output sbom.cdx.json <image:tag>
trivy image --format spdx-json --output sbom.spdx.json <image:tag>
```

Scan an existing SBOM:

```bash
trivy sbom \
  --scanners vuln,license \
  --severity HIGH,CRITICAL \
  --format json \
  --output /tmp/trivy-sbom.json \
  sbom.cdx.json
```

### Offline or air-gapped scan

On a connected machine, download the databases:

```bash
trivy image --download-db-only
trivy image --download-java-db-only
```

Then scan offline when supported:

```bash
trivy image --offline-scan --skip-db-update --format json <image:tag>
```

## 6. Result handling

Parse JSON before reporting whenever possible. Summarize:

- Total findings by severity.
- Top critical/high findings with package, installed version, fixed version, and CVE ID.
- Fixed vs unfixed counts.
- Affected target names, images, packages, or files.
- Secret and misconfiguration counts without exposing secret values.

Then offer focused next actions:

- Explain the top CVEs using `cve-explainer`.
- Build a remediation plan by package or image layer.
- Generate or update `.trivyignore` only for explicitly accepted risks.
- Re-scan after proposed fixes.
- Emit SARIF, CycloneDX, SPDX, or table output if the user needs a report artifact.

## 7. `.trivyignore` handling

Do not create or edit `.trivyignore` without explicit user request. An ignore entry is an accepted risk and should have a reason and expiration when possible.

Example:

```text
# Accepted risk: only reachable by admins; revisit after upstream patch.
CVE-2024-12345 exp:2026-12-31
```

Before recommending an ignore, verify that the finding is unfixed, not reachable, mitigated, or otherwise intentionally accepted.

## 8. Reporting format

For a concise terminal-friendly summary, use a table like:

| Sev | ID | Package | Installed | Fixed | Target |
|---|---|---|---|---|---|
| CRITICAL | CVE-YYYY-NNNN | openssl | 1.2.3 | 1.2.4 | image:tag |

End with:

- What command was run, with sensitive paths or tokens redacted.
- Where the JSON/report artifact was saved, if retained.
- The highest-priority remediation.
- Any limitations, such as skipped DB updates, offline mode, or unsupported flags.
