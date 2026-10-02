# Part 78: Advanced Evasion Pattern Analysis (Defensive)

> **ระดับ:** ผู้เชี่ยวชาญ-ระดับโลก | **เวลาเรียน:** ~100 นาที | **ข้อกำหนด:** Part 01-77
>
> **ข้อกำหนดด้านจริยธรรม:** เนื้อหาใช้เพื่อพัฒนา IDS/WAF และทำ Security Research เท่านั้น

---

## 78.1 Protocol-Level Evasion Detection

```python
import re
from typing import Dict, List

print("Advanced Evasion Pattern Analysis (Defensive):")
print("=" * 60)

print("\n1. HTTP protocol-level evasion detection:")

# HTTP method case variation
HTTP_METHOD_CASE = re.compile(r'^(?:GET|POST|PUT|DELETE|PATCH|HEAD|OPTIONS|TRACE|CONNECT)\b',
                               re.IGNORECASE)

# HTTP/0.9 request (no headers, often bypasses WAF)
HTTP09 = re.compile(r'^(?:GET|POST)\s+\S+\s*$', re.MULTILINE)

# Chunked encoding abuse: chunk size 0 followed by injected headers
CHUNKED_INJECT = re.compile(
    r'^0\r?\n(?P<trailer>[A-Za-z-]+:\s*.+)\r?\n',
    re.MULTILINE
)

# Parameter pollution (same param multiple times)
PARAM_POLLUTION = re.compile(
    r'(?P<param>[^=&]+)=[^&]*(?:&[^=&]*=[^&]*)*&(?P=param)=',
    re.IGNORECASE
)

# HPP (HTTP Parameter Pollution) in query string
def detect_hpp(query_string: str) -> List[str]:
    params = {}
    duplicates = []
    for part in query_string.split('&'):
        if '=' in part:
            key, _, _ = part.partition('=')
            key = key.lower()
            if key in params:
                duplicates.append(key)
            params[key] = True
    return duplicates


# Request smuggling indicators (from Part 71, extended)
CONTENT_LENGTH = re.compile(r'^Content-Length:\s*(\d+)', re.IGNORECASE | re.MULTILINE)
TRANSFER_ENC   = re.compile(r'^Transfer-Encoding:\s*(\S+)', re.IGNORECASE | re.MULTILINE)
TRANSFER_OBFUSC = re.compile(
    r'^Transfer-Encoding:\s*(?:chunked\s*,\s*identity|identity\s*,\s*chunked|xchunked|'
    r'chunked\s+|x-chunked|\bchunked\b)',
    re.IGNORECASE | re.MULTILINE
)


def detect_smuggling(raw_request: str) -> List[str]:
    issues = []
    cl_matches = CONTENT_LENGTH.findall(raw_request)
    te_matches = TRANSFER_ENC.findall(raw_request)

    if len(cl_matches) > 1:
        issues.append(f'duplicate Content-Length: {cl_matches}')
    if len(te_matches) > 1:
        issues.append(f'duplicate Transfer-Encoding: {te_matches}')
    if cl_matches and te_matches:
        issues.append(f'CL+TE present (smuggling risk): CL={cl_matches}, TE={te_matches}')
    if TRANSFER_OBFUSC.search(raw_request):
        issues.append('obfuscated Transfer-Encoding')

    return issues


test_queries = [
    'user=alice&role=user&user=admin',
    'id=1&callback=func&id=2',
    'search=hello&page=1',
    'lang=en&debug=false&lang=..%2F..%2Fetc',
]

print(f"\n   HTTP Parameter Pollution (HPP) detection:")
for qs in test_queries:
    dups = detect_hpp(qs)
    if dups:
        print(f"   ⚠ {qs!r}")
        print(f"     Duplicate params: {dups}")
    else:
        print(f"   ✓ {qs!r}")


test_requests = [
    "POST /api HTTP/1.1\r\nHost: example.com\r\nContent-Length: 10\r\nContent-Length: 0\r\n\r\nbody",
    "POST /api HTTP/1.1\r\nHost: example.com\r\nTransfer-Encoding: chunked\r\nContent-Length: 5\r\n\r\n5\r\nhello\r\n0\r\n\r\n",
    "GET /page HTTP/1.1\r\nHost: example.com\r\nTransfer-Encoding: chunked, identity\r\n\r\n",
    "POST /api HTTP/1.1\r\nHost: example.com\r\nContent-Length: 10\r\n\r\nhello",
]

print(f"\n   Request smuggling detection:")
for req in test_requests:
    issues = detect_smuggling(req)
    first_line = req.split('\r\n')[0]
    if issues:
        print(f"   ⚠ {first_line!r}")
        for issue in issues:
            print(f"     → {issue}")
    else:
        print(f"   ✓ {first_line!r}")
```

---

## 78.2 Unicode & Character Set Evasion

```python
import re
import unicodedata
from typing import Dict, List

print("\nUnicode & Character Set Evasion Detection:")
print("=" * 60)

# Unicode homoglyph detection (lookalike characters)
# Characters that look like ASCII but have different code points

LATIN_LOOKALIKES = {
    'а': 'a',  # Cyrillic a
    'е': 'e',  # Cyrillic e
    'о': 'o',  # Cyrillic o
    'р': 'r',  # Cyrillic r
    'с': 'c',  # Cyrillic c
    'х': 'x',  # Cyrillic x
    'і': 'i',  # Cyrillic i
    'ӏ': 'l',  # Cyrillic l
    '’': "'",  # Right single quotation
    '“': '"',  # Left double quotation
    '”': '"',  # Right double quotation
    '‐': '-',  # Hyphen
    '−': '-',  # Minus sign
}

# Fullwidth ASCII detection (FF01-FF5E range)
FULLWIDTH = re.compile(r'[！-～]')

# Cyrillic homoglyphs that look like Latin
HOMOGLYPH_CHARS = re.compile(
    r'[аеорсхіӏ！-～]'
)

# Unicode normalization forms
def normalize_unicode(text: str) -> str:
    # NFKC: compatibility decomposition + canonical composition
    # Converts fullwidth characters to ASCII, etc.
    return unicodedata.normalize('NFKC', text)


def detect_unicode_evasion(text: str) -> Dict:
    normalized = normalize_unicode(text)
    findings = []

    if FULLWIDTH.search(text):
        findings.append('fullwidth characters')
    if HOMOGLYPH_CHARS.search(text):
        findings.append('homoglyph characters')

    # Check if normalization reveals something different
    if normalized.lower() != text.lower():
        findings.append(f'normalization transforms: {text[:20]!r} -> {normalized[:20]!r}')

    # Check for Unicode direction override characters
    if re.search(r'[‎‏‪-‮⁦-⁩]', text):
        findings.append('Unicode direction control characters')

    # Zero-width characters (invisible but affect parsing)
    if re.search(r'[​‌‍﻿]', text):
        findings.append('zero-width characters')

    return {
        'original':   text,
        'normalized': normalized,
        'findings':   findings,
    }


# ASCII-encoded unicode escapes for 'alert'
_alert_escaped = 'alert'

unicode_payloads = [
    'SELECT',                           # Normal ASCII
    'ＳＥＬＥＣＴ',  # Fullwidth SELECT
    'ѕеlеct',            # Cyrillic lookalikes
    'SELECT​FROM',                 # Zero-width space
    'SE‪LECT',                     # LTR override
    '<scr​ipt>alert(1)</script>',  # Zero-width in tag
    'alert(1)',                         # Normal
    _alert_escaped + '(1)',             # Unicode escapes for 'alert'
]

print(f"\n   Unicode evasion detection:")
for payload in unicode_payloads:
    result = detect_unicode_evasion(payload)
    if result['findings']:
        print(f"\n   ⚠ Original:   {payload[:40]!r}")
        print(f"     Normalized: {result['normalized'][:40]!r}")
        for f in result['findings']:
            print(f"     → {f}")
    else:
        print(f"   ✓ {payload[:40]!r}")
```

---

## 78.3 Header Injection & CRLF Evasion

```python
import re
from typing import Dict, List

print("\nHeader Injection & CRLF Evasion:")
print("=" * 60)

# CRLF injection patterns
CRLF_LITERAL   = re.compile(r'\r\n|\r|\n')
CRLF_ENCODED   = re.compile(r'%0[dD]%0[aA]|%0[aA]|%0[dD]|\r\n', re.IGNORECASE)
CRLF_UNICODE   = re.compile(r'%u000[dD]|%u000[aA]')
CRLF_DBL       = re.compile(r'%250[dD]%250[aA]|%250[aA]|%250[dD]', re.IGNORECASE)

# HTTP response splitting (injecting fake response after double CRLF)
RESP_SPLIT = re.compile(
    r'(?:\r\n|\r|\n|%0[dDaA]|%0[dD]%0[aA]){2,}',
    re.IGNORECASE
)

# Header value injection attempts
HEADER_INJECT = re.compile(
    r'(?:\r\n|\r|\n|%0[dDaA])+(?:Set-Cookie|Location|Content-Type|Status|HTTP/)',
    re.IGNORECASE
)

# Log injection (injecting fake log entries)
LOG_INJECT = re.compile(
    r'(?:\r\n|\r|\n)(?:\d{1,3}\.){3}\d{1,3}\s+-\s+-\s+\[',
)


def detect_crlf_injection(value: str) -> List[str]:
    findings = []
    if CRLF_LITERAL.search(value):
        findings.append('literal CRLF')
    if CRLF_ENCODED.search(value):
        findings.append('URL-encoded CRLF')
    if CRLF_UNICODE.search(value):
        findings.append('Unicode CRLF')
    if CRLF_DBL.search(value):
        findings.append('double-encoded CRLF')
    if RESP_SPLIT.search(value):
        findings.append('response splitting (double CRLF)')
    if HEADER_INJECT.search(value):
        findings.append('header injection')
    if LOG_INJECT.search(value):
        findings.append('log injection')
    return findings


# Email header injection
EMAIL_HEADER_INJECT = re.compile(
    r'(?:\r\n|\r|\n|%0[dDaA])+(?:To:|From:|CC:|BCC:|Subject:|Content-Type:)',
    re.IGNORECASE
)

# Redirect header injection
REDIRECT_INJECT = re.compile(
    r'(?:\r\n|\r|\n)+Location:\s*https?://',
    re.IGNORECASE
)


crlf_payloads = [
    '/redirect?url=https://evil.com',
    '/redirect?url=https://good.com%0d%0aSet-Cookie:admin=true',
    '/page?lang=en%0d%0aLocation:%20https://phishing.example.com',
    '/log?msg=user+logged+in%0a192.168.1.1+-+-+[01/Jan/2024] GET /admin 200',
    '/mail?to=user@example.com%0d%0aBCC:attacker@evil.com',
    '/page?name=Alice',
    '/page?name=Bob\r\nSet-Cookie: admin=true',
]

print(f"\n   CRLF injection detection:")
for payload in crlf_payloads:
    findings = detect_crlf_injection(payload)
    if findings:
        print(f"   ⚠ {payload[:60]!r}")
        print(f"     → {findings}")
    else:
        print(f"   ✓ {payload[:60]!r}")
```

---

## 78.4 Pattern Polymorphism Detection

```python
import re
from typing import Dict, List

print("\nPattern Polymorphism Detection:")
print("=" * 60)

# Polymorphic XSS: same payload, many syntactic forms
# Defense: semantic/behavioral matching instead of string matching

def xss_signature(payload: str) -> str:
    """Semantic XSS fingerprint, not string match."""
    sig = []
    p = payload.lower()

    # JS execution vectors
    if re.search(r'<script', p): sig.append('script')
    if re.search(r'on\w+=', p): sig.append('event')
    if re.search(r'javascript:', p): sig.append('proto')
    if re.search(r'<svg|<img|<body|<iframe|<input|<object', p): sig.append('tag')
    if re.search(r'eval\(|setTimeout\(|setInterval\(', p): sig.append('js_exec')

    # Target functions
    if re.search(r'alert|confirm|prompt|console\.log', p): sig.append('test_fn')
    if re.search(r'document\.cookie', p): sig.append('cookie')
    if re.search(r'window\.location', p): sig.append('redirect')
    if re.search(r'fetch\(|xmlhttprequest', p): sig.append('exfil')

    return '+'.join(sorted(sig)) or 'benign'


polymorphic_xss = [
    '<script>alert(1)</script>',
    '<script>alert(document.cookie)</script>',
    '<img src=x onerror=alert(1)>',
    '<img src=x onerror=alert(document.cookie)>',
    '<svg onload=alert(1)>',
    '<body onload=alert(1)>',
    '<iframe src="javascript:alert(1)">',
    '<input autofocus onfocus=alert(1)>',
    '<div onmouseover=alert(1)>hover me</div>',
    '"><script>alert(1)</script>',
    "';alert(1)//",
    '</title><script>alert(1)</script>',
    'normal text no xss here',
]

print(f"\n   XSS semantic signatures:")
seen_sigs = set()
for payload in polymorphic_xss:
    sig = xss_signature(payload)
    is_dup = sig in seen_sigs and sig != 'benign'
    seen_sigs.add(sig)
    is_dangerous = 'script' in sig or 'event' in sig or 'proto' in sig or 'tag' in sig
    marker = '\U0001f534' if is_dangerous else '✓'
    dup_note = ' [variant]' if is_dup else ''
    print(f"   {marker} {payload[:50]:<53} sig={sig}{dup_note}")

print(f"\n   SQLi polymorphism detection:")

sqli_variants = [
    "' OR '1'='1",
    "' OR 1=1--",
    "' OR 1=1#",
    "' OR 1=1/*",
    "admin'--",
    "admin' #",
    "1 OR 1=1",
    "1 OR 1=1--",
    "') OR ('1'='1",
    "') OR ('1'='1'--",
    "1 UNION SELECT NULL--",
    "1 UNION SELECT NULL,NULL--",
    "1 UNION SELECT NULL,NULL,NULL--",
]

def sqli_semantic(payload: str) -> str:
    p = payload.lower()
    tags = []
    if re.search(r'\bor\b', p): tags.append('or_tautology')
    if re.search(r'\band\b', p): tags.append('and_tautology')
    if re.search(r'\bunion\b', p): tags.append('union_inject')
    if re.search(r'\bselect\b', p): tags.append('select')
    if re.search(r'--|#|/\*', p): tags.append('comment')
    if re.search(r"'", p): tags.append('quote')
    if re.search(r'\bsleep\b|\bbenchmark\b', p): tags.append('time_based')
    return '+'.join(tags) or 'unknown'

sems = {}
for payload in sqli_variants:
    sem = sqli_semantic(payload)
    sems.setdefault(sem, []).append(payload)

for sem, payloads in sems.items():
    print(f"\n   Semantic group: [{sem}]")
    for p in payloads:
        print(f"   • {p!r}")
```

---

## 78.5 สรุป Part 78

```
Advanced Evasion & Detection Strategies:

1. Protocol-level evasion:
   HPP: same param twice -> ?id=1&id=admin (last wins in some frameworks)
   Smuggling: CL+TE both present = TE-CL or CL-TE attack
   Chunked obfuscation: Transfer-Encoding: chunked, identity

2. Unicode evasion:
   Fullwidth: ＳＥＬＥＣＴ (FF01-FF5E range)
   Cyrillic: с, е, о look identical to c, e, o
   Zero-width: U+200B breaks keyword matching
   Direction: U+202A-U+202E invisible visual tricks
   Defense: NFKC normalization before ALL checks

3. CRLF injection:
   %0d%0a or %0a = newline in URL
   Double-CRLF = response splitting (fake response)
   Header injection: insert Set-Cookie, Location
   Log injection: insert fake access log entries
   Defense: reject \r \n in redirect URLs and headers

4. Polymorphism:
   Same attack, many syntactic forms
   Defense: semantic/behavioral matching, not string matching
   Group by: OR-tautology, UNION-inject, event-handler, etc.
   Signature: payload_class + function_called + context

5. Defense strategy summary:
   Level 1: Normalize (NFKC + URL decode + HTML decode)
   Level 2: Semantic WAF rules (not keyword matching)
   Level 3: Anomaly scoring (sum of risk signals)
   Level 4: Behavioral analysis (rate, pattern, sequence)
   Level 5: Defense-in-depth (parameterized SQL, CSP, etc.)
```

---

*[← Part 77: Filter Bypass Techniques](part-77-filter-bypass.md) | [→ Part 79: Threat Intelligence Patterns](part-79-threat-intel.md)*
