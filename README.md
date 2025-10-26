# arrow

**Purpose:**  
`arrow` is a bash-first toolkit for rapid reconnaissance and proof-of-concept (PoC) testing against web applications during vendor POCs. It is designed to produce repeatable, documented findings quickly. `arrow` is not a replacement for a full penetration test performed by qualified professionals; it is intended for focused discovery and validation in authorized environments.

---

## Important Legal Notice (Read before use)
You **must** have explicit, written authorization to test the target(s) (a signed POC/Terms of Reference or equivalent). Use of this tool against systems without authorization is illegal. The authors and contributors accept no liability for misuse. By using this tool you acknowledge you are authorized to perform the listed tests.

---

## Features
- Reconnaissance: DNS, port scans, basic service detection, WAF detection.
- TLS checks: SNI behavior, certificate collection, passthrough vs termination checks.
- Web scanning: directory discovery, lightweight vulnerability probes (XSS, SQLi, SSRF).
- Targeted PoCs: JWT tampering, file upload checks, SSRF callbacks.
- Evidence collection: captures requests/responses, certificate chains, and optional pcaps.
- Reporting: generates JSON/CSV/Markdown summaries with severity and remediation guidance.

---

## Requirements
Tested on Debian/Ubuntu and macOS. Required tools (install via package manager):
- `bash` (4+), `curl`, `jq`, `openssl`, `nc/netcat`, `tcpdump`, `wget`, `sed`, `awk`, `python3`
- `nmap`, `ffuf` (or `gobuster`), `sqlmap`, `nikto`, `wafw00f`
Optional but recommended:
- `mitmproxy` or `Burp Suite` for interactive testing
- `masscan`, `tshark`/`wireshark` for deeper analysis

Example install on Debian/Ubuntu:
```bash
sudo apt update
sudo apt install -y nmap ffuf sqlmap nikto wafw00f jq curl openssl netcat tcpdump python3
```

---

## Installation
Clone and install a simple wrapper:
```bash
git clone https://github.com/dpereowei/arrow.git
cd arrow
chmod +x arrow.sh
sudo ln -s "$(pwd)/arrow.sh" /usr/local/bin/arrow
```
`arrow` will then be available on your PATH.

---

## Quick Start
1. Place a signed authorization file (e.g., `poc_auth.txt`) in your working directory.
2. Create an evidence directory:
```bash
mkdir -p ./evidence/target
```
3. Run reconnaissance:
```bash
arrow recon --host target.example.com --out ./evidence/target
```
4. Run TLS checks:
```bash
arrow tls-check --host target.example.com --out ./evidence/target
```
5. Run a quick web scan (non-destructive):
```bash
arrow web-scan --url https://target.example.com --out ./evidence/target
```
6. Generate a report:
```bash
arrow report --evidence ./evidence/target --format md,json --out ./reports/target
```

---

## Command Overview
Run `arrow help` for full usage. Key subcommands:

- `recon` — DNS, ports, service detection, WAF detection.  
  Example:
  ```bash
  arrow recon --host target.example.com --out ./evidence/target
  ```

- `tls-check` — Collect certificate chains, test SNI behavior, and verify TLS configuration.
  ```bash
  arrow tls-check --host target.example.com --port 443 --out ./evidence/target
  ```

- `web-scan` — Directory discovery, Nikto scan, and light injection probes (safe defaults).
  ```bash
  arrow web-scan --url https://target.example.com --out ./evidence/target
  ```

- `xss-test` — Targeted XSS checks on specific parameters.
  ```bash
  arrow xss-test --url "https://target.example.com/search?q=FUZZ" --payload "<svg/onload=alert(1)>"
  ```

- `sqli-test` — Non-destructive SQLi detection with optional deeper `sqlmap` runs.
  ```bash
  arrow sqli-test --url "https://target.example.com/item?id=1"
  ```

- `ssrf-test` — SSRF PoC using a collaborator domain (e.g., Burp Collaborator).
  ```bash
  arrow ssrf-test --url "https://target.example.com/fetch?url=" --collab "xyz.burpcollab.net"
  ```

- `upload-test` — Tests for insecure file upload handling (MIME tricks, double extensions).
  ```bash
  arrow upload-test --url "https://target.example.com/upload" --file ./pocs/php-shell.jpg
  ```

- `jwt-check` — Quick JWT tampering checks (alg:none, RS256→HS256 tests).
  ```bash
  arrow jwt-check --token "<jwt>" --pubkey ./keys/pub.pem
  ```

- `collect` — Collect evidence artifacts (pcap, burp requests/responses).
  ```bash
  arrow collect --target target.example.com --pcap --burp --out ./evidence/target
  ```

- `report` — Consolidate findings to JSON/CSV/Markdown.
  ```bash
  arrow report --evidence ./evidence/target --format json,md --out ./reports/target
  ```

---

## Evidence and Output
`arrow` stores evidence in `./evidence/<target>/`:
- `nmap.full.xml` — Nmap results  
- `tls.cert.pem` — Collected certificate chains  
- `ffuf.dirs.json` — Directory discovery results  
- `poc_<test>.req` / `poc_<test>.res` — PoC requests and responses  
- `capture.pcap` — Optional tcpdump capture  
- `report.*` — Consolidated outputs (json/csv/md)

**Handling evidence:** keep all evidence in a secure location. Do not commit sensitive artifacts (pcaps, keys, tokens) to public repositories.

---

## Operational Safety & Rules of Engagement
Default behavior is **non-destructive**. Destructive or potentially intrusive actions require explicit flags:
- Use `--aggressive` and `--allow-exploit` to enable higher-impact tests (only with authorization).
- Use `--rate`/`--concurrency` to limit request rates and avoid DoS.
- If a critical vulnerability is discovered (RCE, metadata SSRF, unauthenticated admin), stop further automated testing and follow your escalation procedure.

Escalation helper:
```bash
arrow alert --target target.example.com --finding "SSRF to metadata"
```

---

## Scoring and Reporting
Each finding includes:
- `id`, `title`, `severity` (Critical/High/Medium/Low), `confidence`, `poct` (PoC/test), `remediation`.

Severity guidance:
- **Critical** — RCE, SSRF to cloud metadata, unauthenticated admin access. Immediate stop and remediation.
- **High** — Auth bypasses, file upload RCE, SQLi with data access.
- **Medium** — Stored XSS, IDOR exposing PII.
- **Low** — Missing security headers, info leakage without direct exploitation.

Report formats: JSON, CSV, Markdown. Include reproduction steps, evidence references, and remediation notes for each finding.

---

## Configuration
Default config file: `~/.arrowrc` (optional)
```ini
DEFAULT_OUT=./evidence
COLLAB_DOMAIN=xyz.burpcollab.net
RATE=50
CONCURRENCY=25
```
Environment variables:
- `ARROW_OUT` — default evidence directory
- `ARROW_COLLAB` — default collaborator domain
- `ARROW_RATE` — default request rate
- `ARROW_USER` / `ARROW_PASS` — optional credentials (used only when explicitly provided)

---

## Example Session
```bash
mkdir -p ./evidence/target

arrow recon --host target.example.com --out ./evidence/target
arrow tls-check --host target.example.com --out ./evidence/target
arrow web-scan --url https://target.example.com --out ./evidence/target
arrow ssrf-test --url "https://target.example.com/fetch?url=" --collab "abc.burpcollab.net" --out ./evidence/target
arrow report --evidence ./evidence/target --format md,json --out ./reports/target
```

---

## Extending the Toolkit
`arrow` is modular. Add new modules for GraphQL, SAML/OAuth introspection, advanced deserialization checks, or integrations with SIEMs. Keep additions script-based and include documentation and tests.

---

## Responsible Use and Disclosure
If you discover vulnerabilities, follow your organization's disclosure policy or the vendor’s coordinated disclosure process. Avoid public disclosure without coordination and remediation.

---

## License
MIT License. Use responsibly.

