# Part 83: Binary, Hex & Data Encoding Patterns

> **ระดับ:** ผู้เชี่ยวชาญ-ระดับโลก | **เวลาเรียน:** ~90 นาที | **ข้อกำหนด:** Part 01-82

---

## 83.1 Binary & Hex Format Detection

```python
import re
from typing import Dict, List

print("Binary, Hex & Data Encoding Patterns:")
print("=" * 60)

print("\n1. Binary and hex format patterns:")

# Hexadecimal strings (even length for byte pairs)
HEX_STR_EVEN = re.compile(r'\b[0-9a-fA-F]{2}(?:[0-9a-fA-F]{2})*\b')
HEX_STR_ANY  = re.compile(r'\b[0-9a-fA-F]{4,}\b')

# Hex dump format (address: bytes  ASCII)
HEX_DUMP_LINE = re.compile(
    r'^(?P<addr>[0-9a-fA-F]{4,8}):\s+'
    r'(?P<bytes>(?:[0-9a-fA-F]{2}\s*)+)'
    r'(?:\s{2,}(?P<ascii>.+))?$',
    re.MULTILINE
)

# Binary string
BINARY_STR = re.compile(r'\b[01]{8}(?:[01]{8})*\b')

# Octal representation
OCTAL_STR = re.compile(r'\\0(?:[0-7]{1,3})|0(?:[0-7]{7,})\b')

# C-style byte array
C_BYTE_ARRAY = re.compile(
    r'(?:unsigned\s+char|uint8_t|byte)\s+\w+\[\]\s*=\s*\{(?P<bytes>[^}]+)\}',
    re.IGNORECASE
)

# Python byte string
PY_BYTES = re.compile(r"b'(?:[^'\\]|\\x[0-9a-fA-F]{2}|\\[nrtb\\'])*'")
PY_BYTES_HEX = re.compile(r'\\x[0-9a-fA-F]{2}')

# Base16/Hex in various formats
BASE16 = re.compile(r'0[xX][0-9a-fA-F]+')
PERCENT_HEX = re.compile(r'%[0-9a-fA-F]{2}')


def classify_hex_string(s: str) -> str:
    if len(s) == 32:  return 'likely MD5 hash'
    if len(s) == 40:  return 'likely SHA1 hash'
    if len(s) == 64:  return 'likely SHA256 hash'
    if len(s) == 128: return 'likely SHA512 hash'
    if len(s) % 2 == 0:
        try:
            decoded = bytes.fromhex(s).decode('ascii', errors='replace')
            printable = sum(1 for c in decoded if 32 <= ord(c) < 127)
            if printable / len(decoded) > 0.8:
                return f'hex-encoded ASCII: {decoded[:30]!r}'
        except Exception:
            pass
        return f'hex bytes ({len(s)//2} bytes)'
    return f'odd-length hex ({len(s)} chars)'


sample_data = [
    '0x4142434445',
    '48656c6c6f2c20576f726c64',
    'a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6',
    '01001000 01100101 01101100',
    "b'\\x41\\x42\\x43'",
]

print(f"\n   Hex string classification:")
for data in sample_data:
    hex_matches = HEX_STR_ANY.findall(data)
    bin_matches = BINARY_STR.findall(data)
    b16_matches = BASE16.findall(data)

    if hex_matches:
        for m in hex_matches[:2]:
            classification = classify_hex_string(m)
            print(f"   Hex: {m[:30]!r} → {classification}")
    if bin_matches:
        for m in bin_matches[:2]:
            decoded = ''.join(chr(int(m[i:i+8], 2)) for i in range(0, len(m), 8))
            print(f"   Bin: {m[:24]!r} → {decoded!r}")
    if b16_matches:
        for m in b16_matches[:2]:
            val = int(m, 16)
            print(f"   0x:  {m!r} → decimal={val} ascii={chr(val & 0x7f)!r}")
```

---

## 83.2 Shellcode & Exploit Pattern Detection

```python
import re
from typing import List, Dict

print("\nShellcode & Exploit Pattern Detection:")
print("=" * 60)

# NOP sled patterns (0x90 repeated)
NOP_SLED = re.compile(r'(?:\\x90){10,}|(?:0x90,?\s*){10,}')

# Common x86 shellcode patterns
X86_INT80   = re.compile(r'\\xcd\\x80')        # int 0x80 (Linux syscall)
X86_SYSCALL = re.compile(r'\\x0f\\x05')        # syscall (x64)
X86_RET     = re.compile(r'\\xc3')             # ret
X86_JMP_ESP = re.compile(r'\\xff\\xe4')        # jmp esp
X86_CALL_EBX = re.compile(r'\\xff\\xd3')       # call ebx

# Common shellcode bytes (hex encoded)
SHELLCODE_MARKERS = re.compile(
    r'(?:\\x[0-9a-f]{2}){20,}',
    re.IGNORECASE
)

# Format string attack patterns
FMT_STRING = re.compile(
    r'(?:%n|%[1-9]\d*\$n|%[hn]{1,2})',
)

FMT_STR_MULTI = re.compile(
    r'(?:%[1-9]\d*[xdn]){3,}|(?:%[xd]){5,}'
)

# Heap spray indicators
HEAP_SPRAY = re.compile(
    r'(?:\\x90\\x90\\x90\\x90){50,}|'
    r'(?:0x90909090){10,}'
)

# Stack smashing indicators (long repeating patterns)
STACK_SMASH = re.compile(
    r'(?:A{200,}|\\x41{200,})',
    re.IGNORECASE
)

# Return-oriented programming gadgets pattern
ROP_PATTERN = re.compile(
    r'(?:ret|pop\s+\w+;\s*ret|add\s+esp,\s*\d+;\s*ret|'
    r'xor\s+\w+,\s*\w+;\s*ret)',
    re.IGNORECASE
)


def detect_exploit_pattern(data: str) -> List[str]:
    findings = []

    if NOP_SLED.search(data):
        findings.append('NOP sled detected')
    if SHELLCODE_MARKERS.search(data):
        sc_m = SHELLCODE_MARKERS.search(data)
        byte_count = len(re.findall(r'\\x[0-9a-f]{2}', sc_m.group(0), re.IGNORECASE))
        findings.append(f'shellcode-like sequence ({byte_count} bytes)')
    if X86_INT80.search(data):
        findings.append('x86 int 0x80 (Linux syscall)')
    if X86_SYSCALL.search(data):
        findings.append('x64 syscall instruction')
    if X86_JMP_ESP.search(data):
        findings.append('jmp esp (stack pivot)')
    if FMT_STRING.search(data):
        findings.append('format string %n write primitive')
    if FMT_STR_MULTI.search(data):
        findings.append('format string multi-printf (info leak)')
    if HEAP_SPRAY.search(data):
        findings.append('heap spray pattern')
    if STACK_SMASH.search(data):
        findings.append('stack smash (buffer overflow input)')

    return findings


exploit_payloads = [
    'A' * 300 + '\xef\xbe\xad\xde',
    '\\x90' * 20 + '\\xcd\\x80',
    '%1$n%2$n%3$n%4$n',
    '%x%x%x%x%x%x%x',
    '\\x41\\x42\\x43\\x44\\x45\\x46\\x47\\x48\\x49\\x4a' * 5,
    'normal user input data here',
]

print(f"\n   Exploit pattern detection:")
for payload in exploit_payloads:
    findings = detect_exploit_pattern(payload)
    short = repr(payload[:50])
    if findings:
        print(f"\n   [ALERT] {short}")
        for f in findings:
            print(f"     → {f}")
    else:
        print(f"   OK: {short}")
```

---

## 83.3 File Magic Bytes & MIME Detection

```python
import re
import struct
from typing import Dict, List

print("\nFile Magic Bytes & MIME Detection:")
print("=" * 60)

# Magic byte signatures for common file formats
FILE_MAGIC = {
    b'\x89PNG': 'PNG image',
    b'\xff\xd8\xff': 'JPEG image',
    b'GIF8': 'GIF image',
    b'PK\x03\x04': 'ZIP archive',
    b'Rar!': 'RAR archive',
    b'\x7fELF': 'ELF binary (Linux executable)',
    b'MZ': 'PE/EXE binary (Windows)',
    b'\xca\xfe\xba\xbe': 'Java .class file',
    b'\xfe\xed\xfa\xce': 'Mach-O 32-bit binary (macOS)',
    b'\xcf\xfa\xed\xfe': 'Mach-O 64-bit binary (macOS)',
    b'%PDF': 'PDF document',
    b'\xd0\xcf\x11\xe0': 'MS Office (legacy .doc/.xls)',
    b'\x1f\x8b': 'gzip compressed',
    b'BZh': 'bzip2 compressed',
}

# Regex versions for text-based detection (base64-encoded binaries)
MAGIC_PATTERNS = {
    'elf_base64':  re.compile(r'^f0VMR(?:gA|gB|gC)', re.MULTILINE),
    'pe_base64':   re.compile(r'^TVo|^/9j/4A', re.MULTILINE),
    'zip_base64':  re.compile(r'^UEsD', re.MULTILINE),
    'pdf_base64':  re.compile(r'^JVBERi0', re.MULTILINE),
    'shell_b64':   re.compile(r'^[A-Za-z0-9+/]{50,}={0,2}$', re.MULTILINE),
}


def detect_file_magic(data: bytes) -> str:
    for magic, description in FILE_MAGIC.items():
        if data.startswith(magic):
            return description
    return 'unknown'


def detect_base64_magic(b64_str: str) -> List[str]:
    findings = []
    for name, pattern in MAGIC_PATTERNS.items():
        if pattern.search(b64_str):
            findings.append(name)
    return findings


# MIME type vs content mismatch detection
MIME_TYPE = re.compile(r'Content-Type:\s*([^\r\n;]+)', re.IGNORECASE)

def detect_mime_mismatch(headers: str, body_bytes: bytes) -> List[str]:
    issues = []
    mime_m = MIME_TYPE.search(headers)
    if not mime_m:
        return ['Missing Content-Type']

    declared_type = mime_m.group(1).strip().lower()
    actual = detect_file_magic(body_bytes)

    if 'image/jpeg' in declared_type and not actual.startswith('JPEG'):
        issues.append(f'MIME mismatch: declared image/jpeg but got {actual!r}')
    elif 'image/png' in declared_type and not actual.startswith('PNG'):
        issues.append(f'MIME mismatch: declared image/png but got {actual!r}')
    elif 'application/pdf' in declared_type and 'PDF' not in actual:
        issues.append(f'MIME mismatch: declared PDF but got {actual!r}')
    elif 'text/' in declared_type and actual in ('ELF binary (Linux executable)', 'PE/EXE binary (Windows)'):
        issues.append(f'CRITICAL: declared text but got executable {actual!r}')

    return issues


test_files = [
    (b'\x89PNG\r\n\x1a\n' + b'\x00' * 20, 'PNG image'),
    (b'\xff\xd8\xff\xe0' + b'\x00' * 20, 'JPEG image'),
    (b'\x7fELF\x02\x01\x01' + b'\x00' * 20, 'ELF binary'),
    (b'MZ\x90\x00' + b'\x00' * 20, 'PE executable'),
    (b'%PDF-1.4\n' + b'\x00' * 20, 'PDF document'),
    (b'Hello, World!\n' + b'\x00' * 5, 'text data'),
]

print(f"\n   File magic detection:")
for data, expected in test_files:
    actual = detect_file_magic(data)
    match = 'OK' if expected.split()[0].lower() in actual.lower() else '?'
    print(f"   {match} {data[:8]!r} → {actual}")


print(f"\n   Base64 magic detection:")
b64_samples = [
    'f0VMRgIBAQAAAAAAAAAAAAIAPgABAAAA',
    'TVqQAAMAAAAEAAAA//8AALgAAAAA',
    'UEsDBAAAAAA',
    'JVBERi0xLjQK',
    'SGVsbG8sIFdvcmxkIQ==',
]

for b64 in b64_samples:
    findings = detect_base64_magic(b64)
    print(f"   {'ALERT' if findings else 'OK'}: {b64[:40]!r} → {findings or 'benign'}")
```

---

## 83.4 สรุป Part 83

```
Binary & Hex Pattern Design:

1. Hex string classification:
   32 chars = MD5 hash
   40 chars = SHA1 hash
   64 chars = SHA256 hash
   Even-length hex: try decode as ASCII to see if readable
   0x prefix: C/Python integer literals

2. Shellcode detection:
   NOP sled: \x90 * 10+ (classic shellcode pad)
   int 0x80: \xcd\x80 (32-bit Linux syscall)
   syscall: \x0f\x05 (64-bit Linux syscall)
   jmp esp: \xff\xe4 (stack pivot = ROP)
   Format string: %n (write primitive)

3. File magic bytes (first 4-8 bytes):
   ELF:  7f 45 4c 46 (7fELF)
   PE:   4d 5a (MZ)
   PNG:  89 50 4e 47 (0x89PNG)
   JPEG: ff d8 ff
   PDF:  25 50 44 46 (%PDF)
   ZIP:  50 4b 03 04 (PK..)

4. MIME mismatch detection:
   Upload claims image/jpeg but magic bytes = ELF → polyglot
   Upload claims text/plain but magic = ZIP → dangerous
   Defense: validate magic bytes server-side, not file extension
   Defense: re-encode images server-side (strip metadata)

5. Base64 magic:
   f0VMR = ELF (likely shellcode)
   TVo = PE/DLL
   UEsD = ZIP (may contain malware)
   JVBERi0 = PDF
   /9j/4A = JPEG
```

---

*[← Part 82: API Security Patterns](part-82-api-security.md) | [→ Part 84: Cryptographic Pattern Analysis](part-84-crypto-patterns.md)*
