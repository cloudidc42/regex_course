# Part 88: Web Application Firewall Evasion Techniques

> **ระดับ:** ผู้เชี่ยวชาญ-ระดับโลก | **เวลาเรียน:** ~110 นาที | **ข้อกำหนด:** Part 01-87

---

## 88.1 SQL Injection WAF Bypass Techniques

```python
import re
import urllib.parse
import html
from typing import List

print("WAF Evasion Techniques (Educational/Defensive):")
print("=" * 60)

print("\n1. SQL injection WAF bypass patterns:")

# WAF rule that might be bypassed
BASIC_SQLI_WAF = re.compile(
    r"(?:UNION\s+SELECT|SELECT\s+\*|DROP\s+TABLE|INSERT\s+INTO)",
    re.IGNORECASE
)

# Comment insertion bypass detection
SQLI_COMMENT_BYPASS = re.compile(
    r"UNION\s*/\*.*?\*/\s*(?:ALL\s*)?SELECT|"
    r"UN(?:/\*\*/|--\s*\n|#\s*\n)ION",
    re.IGNORECASE | re.DOTALL
)


def multi_decode(s: str, passes: int = 3) -> str:
    prev = s
    for _ in range(passes):
        decoded = urllib.parse.unquote(prev)
        decoded = html.unescape(decoded)
        if decoded == prev:
            break
        prev = decoded
    return prev


def normalize_sql(text: str) -> str:
    t = multi_decode(text)
    t = re.sub(r'/\*.*?\*/', ' ', t, flags=re.DOTALL)
    t = re.sub(r'--[^\n]*', ' ', t)
    t = re.sub(r'#[^\n]*', ' ', t)
    t = re.sub(r'[\t\n\r\x0b\x0c\xa0+]+', ' ', t)
    t = t.replace('\x00', '')
    return t.upper()


SQLI_ROBUST = re.compile(
    r"(?:UNION\s+(?:ALL\s+)?SELECT|"
    r"SELECT\s+.*\s+FROM\s+|"
    r"(?:AND|OR)\s+\d+\s*=\s*\d+|"
    r"SLEEP\s*\(|BENCHMARK\s*\(|"
    r"DROP\s+TABLE|INSERT\s+INTO|"
    r"INFORMATION_SCHEMA\.|"
    r"LOAD_FILE\s*\(|INTO\s+OUTFILE|INTO\s+DUMPFILE)"
)


bypass_attempts = [
    "' UNION/**/SELECT/**/NULL,NULL--",
    "' UNiOn SeLeCt 1,2,3--",
    "' %55%4e%49%4f%4e %53%45%4c%45%43%54 1--",
    "' UNION%09SELECT%0a1,2--",
    "' /*!UNION*/ /*!SELECT*/ 1,2,3--",
    "1 AND 1=1",
    "normal search term",
]

print(f"\n   Robust vs basic WAF detection:")
print(f"   {'Payload':<50} {'BasicWAF':<10} {'RobustWAF':<10}")
print(f"   {'-'*50} {'-'*10} {'-'*10}")
for payload in bypass_attempts:
    basic_hit  = bool(BASIC_SQLI_WAF.search(payload))
    normalized = normalize_sql(payload)
    robust_hit = bool(SQLI_ROBUST.search(normalized))
    basic_str  = 'BLOCK' if basic_hit else 'PASS'
    robust_str = 'BLOCK' if robust_hit else 'PASS'
    flag = 'BYPASS' if not basic_hit and robust_hit else ''
    print(f"   {payload[:50]!r:<52} {basic_str:<10} {robust_str:<10} {flag}")
```

---

## 88.2 XSS WAF Bypass Patterns

```python
import re
import html
import urllib.parse
from typing import List

print("\nXSS WAF Bypass Patterns:")
print("=" * 60)

# Weak WAF that only blocks <script>
WEAK_XSS_WAF = re.compile(r'<\s*script\s*>', re.IGNORECASE)

# Robust XSS detection (multiple bypass-aware)
XSS_BYPASS_PATTERNS = [
    re.compile(r'\bon\w+\s*=\s*["\']?[^"\'>\s]+', re.IGNORECASE),
    re.compile(r'javascript\s*:', re.IGNORECASE),
    re.compile(r'data:\s*text/html', re.IGNORECASE),
    re.compile(r'\{\{.*\}\}|\$\{.*\}'),
    re.compile(r'<\s*svg[^>]*\bon\w+\s*=', re.IGNORECASE),
    re.compile(r'<\s*img[^>]+onerror\s*=', re.IGNORECASE),
    re.compile(r'expression\s*\(', re.IGNORECASE),
    re.compile(r'<\s*[Ss][Cc][Rr][Ii][Pp][Tt]', re.IGNORECASE),
]


def detect_xss_robust(input_text: str) -> List[str]:
    layers = [
        input_text,
        urllib.parse.unquote(input_text),
        html.unescape(input_text),
        urllib.parse.unquote(urllib.parse.unquote(input_text)),
    ]
    found = []
    for layer in layers:
        for pattern in XSS_BYPASS_PATTERNS:
            m = pattern.search(layer)
            if m and m.group(0) not in found:
                found.append(m.group(0))
    return found


xss_payloads = [
    '<script>alert(1)</script>',
    '<img src=x onerror=alert(1)>',
    '<ScRiPt>alert(1)</ScRiPt>',
    'javascript:alert(1)',
    '<svg onload=alert(1)>',
    '%3Cscript%3Ealert%281%29%3C%2Fscript%3E',
    '{{7*7}}',
    '<img src="data:text/html,<script>alert(1)</script>">',
    'normal text content',
]

print(f"\n   XSS detection with bypass awareness:")
for payload in xss_payloads:
    weak_caught  = bool(WEAK_XSS_WAF.search(payload))
    robust_found = detect_xss_robust(payload)
    bypass   = 'BYPASS' if not weak_caught and robust_found else ''
    weak_str = 'BLOCK' if weak_caught else 'PASS'
    robust_str = 'BLOCK' if robust_found else 'PASS'
    print(f"\n   {payload[:60]!r}")
    print(f"     Weak WAF: {weak_str} | Robust: {robust_str} {bypass}")
    if robust_found:
        for r in robust_found[:2]:
            print(f"     → matched: {r!r}")
```

---

## 88.3 HTTP Header & Protocol Level Bypasses

```python
import re
from typing import Dict, List

print("\nHTTP Header & Protocol-Level WAF Bypasses:")
print("=" * 60)

# X-Forwarded-For IP spoofing
XFF_SPOOF = re.compile(
    r'X-Forwarded-For:\s*127\.0\.0\.1|'
    r'X-Real-IP:\s*127\.0\.0\.1|'
    r'X-Originating-IP:\s*127\.0\.0\.1',
    re.IGNORECASE
)

# HTTP method override (bypass method-based WAF rules)
METHOD_OVERRIDE = re.compile(
    r'(?:'
    r'X-HTTP-Method-Override:\s*(?:DELETE|PUT|PATCH)|'
    r'X-Method-Override:\s*(?:DELETE|PUT)|'
    r'_method=(?:DELETE|PUT|PATCH)'
    r')',
    re.IGNORECASE
)

# Content-Type confusion
CT_CONFUSION = re.compile(
    r'Content-Type:\s*(?:text/xml|application/xml|text/json)',
    re.IGNORECASE
)

# Parameter pollution bypass
PARAM_POLLUTION_SAME = re.compile(
    r'(?:[?&])(\w+)=[^&]*&(?:[^&]*&)*\1='
)


def detect_protocol_bypass(request: str) -> List[str]:
    findings = []
    if XFF_SPOOF.search(request):
        findings.append('IP_SPOOF: X-Forwarded-For to 127.0.0.1')
    if METHOD_OVERRIDE.search(request):
        m = METHOD_OVERRIDE.search(request)
        findings.append(f'METHOD_OVERRIDE: {m.group(0)!r}')
    if CT_CONFUSION.search(request):
        findings.append('CONTENT_TYPE_CONFUSION')
    if PARAM_POLLUTION_SAME.search(request):
        m = PARAM_POLLUTION_SAME.search(request)
        findings.append(f'HTTP_PARAM_POLLUTION: duplicate param {m.group(1)!r}')
    return findings


requests = [
    'POST /delete HTTP/1.1\nHost: example.com\nX-HTTP-Method-Override: DELETE\n',
    'GET /admin HTTP/1.1\nHost: example.com\nX-Forwarded-For: 127.0.0.1\n',
    'POST /api HTTP/1.1\nContent-Type: text/xml\n',
    'GET /api?id=1&id=2 HTTP/1.1\nHost: example.com\n',
    'GET /normal HTTP/1.1\nHost: example.com\nUser-Agent: Mozilla/5.0\n',
]

print(f"\n   HTTP protocol-level bypass detection:")
for req in requests:
    findings = detect_protocol_bypass(req)
    short = req.replace('\n', ' ')[:60]
    if findings:
        print(f"\n   [ALERT] {short!r}")
        for f in findings:
            print(f"     → {f}")
    else:
        print(f"   OK: {short!r}")
```

---

## 88.4 สรุป Part 88

```
WAF Evasion & Defense:

1. SQLi bypass techniques:
   Comment insertion: UNI/**/ON → strip comments before checking
   Case mixing: UnIoN SeLeCt → uppercase/normalize before checking
   URL encoding: %55%4E%49%4F%4E → multi-pass decode (3 passes)
   Whitespace: UNION\t\nSELECT → normalize all whitespace to space
   MySQL conditional: /*!UNION*/ → strip /*! ... */ comments too

2. XSS bypass techniques:
   Event handlers: onerror= works without <script>
   SVG/IMG vectors: <svg onload=>, <img onerror=>
   Case variation: <ScRiPt> → case-insensitive or lowercase first
   URL encoding: %3Cscript%3E → decode before detection
   Protocol: javascript:, data:text/html → detect in href/src attrs

3. HTTP-level bypasses:
   X-Forwarded-For: 127.0.0.1 → WAF trusts internal IP
   X-HTTP-Method-Override: DELETE → method override header
   Parameter pollution: ?id=1&id=2 → some parsers take first, others last
   Content-Type confusion: text/xml for JSON endpoint

4. Defense principles:
   Normalize FIRST: decode → strip comments → normalize whitespace → lowercase
   Check BOTH raw and normalized input
   Use SEMANTIC detection: what does input mean, not just what it looks like
   Defense in depth: WAF + input validation + parameterized queries
   Log EVERYTHING: WAF bypass attempts → tune rules

5. OWASP ModSecurity CRS strategy:
   Paranoia level 1-4: increase strictness vs false-positive tradeoff
   Anomaly scoring: sum rule hits, block at threshold
   Per-rule exception: add exclusion for specific params, not global bypass
```

---

*[← Part 87: Penetration Testing Patterns](part-87-pentest-patterns.md) | [→ Part 89: Regex Performance & ReDoS](part-89-redos.md)*
