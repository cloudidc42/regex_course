# Part 71: HTTP Security Patterns & Header Analysis

> **ระดับ:** สูง-มืออาชีพ | **เวลาเรียน:** ~80 นาที | **ข้อกำหนด:** Part 01-70

---

## 71.1 HTTP Request Header Analysis

```python
import re
from typing import Dict, List, Optional

print("HTTP Security Patterns & Header Analysis:")
print("=" * 60)

print("\n1. HTTP security header detection:")

# Security headers that SHOULD be present
SECURITY_HEADERS = {
    'strict-transport-security': re.compile(
        r'^max-age=(?P<age>\d+)(?:;\s*includeSubDomains)?(?:;\s*preload)?$',
        re.IGNORECASE
    ),
    'content-security-policy': re.compile(
        r"(?:default-src|script-src|style-src|img-src|connect-src|frame-src|font-src|media-src|object-src)\s+'?[^;]+'?",
        re.IGNORECASE
    ),
    'x-frame-options': re.compile(
        r'^(?:DENY|SAMEORIGIN|ALLOW-FROM\s+https?://\S+)$',
        re.IGNORECASE
    ),
    'x-content-type-options': re.compile(r'^nosniff$', re.IGNORECASE),
    'referrer-policy': re.compile(
        r'^(?:no-referrer|no-referrer-when-downgrade|origin|'
        r'origin-when-cross-origin|same-origin|strict-origin|'
        r'strict-origin-when-cross-origin|unsafe-url)$',
        re.IGNORECASE
    ),
    'permissions-policy': re.compile(
        r'(?:\w+=\([^)]*\))',
        re.IGNORECASE
    ),
}

CSP_UNSAFE = re.compile(r"'unsafe-(?:inline|eval)'", re.IGNORECASE)
CSP_WILDCARD = re.compile(r"(?:^|\s)\*(?:\s|;|$)")
HSTS_MIN_AGE = 15768000  # 6 months in seconds


def audit_security_headers(headers: Dict[str, str]) -> Dict:
    issues = {'missing': [], 'weak': [], 'ok': []}
    lower_headers = {k.lower(): v for k, v in headers.items()}
    for name, validator in SECURITY_HEADERS.items():
        if name not in lower_headers:
            issues['missing'].append(name)
        else:
            val = lower_headers[name]
            if validator.search(val):
                if name == 'strict-transport-security':
                    m = re.search(r'max-age=(\d+)', val, re.IGNORECASE)
                    if m and int(m.group(1)) < HSTS_MIN_AGE:
                        issues['weak'].append(f"{name}: max-age too short ({m.group(1)}s)")
                    else:
                        issues['ok'].append(name)
                elif name == 'content-security-policy':
                    if CSP_UNSAFE.search(val):
                        issues['weak'].append(f"{name}: contains unsafe-inline or unsafe-eval")
                    elif CSP_WILDCARD.search(val):
                        issues['weak'].append(f"{name}: contains wildcard (*)")
                    else:
                        issues['ok'].append(name)
                else:
                    issues['ok'].append(name)
            else:
                issues['weak'].append(f"{name}: invalid value '{val[:30]}'")
    return issues


secure_headers = {
    'Strict-Transport-Security': 'max-age=31536000; includeSubDomains; preload',
    'Content-Security-Policy': "default-src 'self'; script-src 'self'; style-src 'self'",
    'X-Frame-Options': 'DENY',
    'X-Content-Type-Options': 'nosniff',
    'Referrer-Policy': 'strict-origin-when-cross-origin',
}

insecure_headers = {
    'Strict-Transport-Security': 'max-age=300',
    'Content-Security-Policy': "default-src *; script-src 'unsafe-inline' 'unsafe-eval'",
    'X-Frame-Options': 'ALLOWALL',
}

for label, headers in [('Secure Headers', secure_headers), ('Insecure Headers', insecure_headers)]:
    result = audit_security_headers(headers)
    print(f"\n   {label}:")
    print(f"   OK ({len(result['ok'])}):      {result['ok']}")
    print(f"   Missing ({len(result['missing'])}): {result['missing']}")
    print(f"   Weak ({len(result['weak'])}):")
    for w in result['weak']:
        print(f"     ⚠  {w}")
```

---

## 71.2 HTTP Request Anomaly Detection

```python
import re
from typing import Dict, List

print("\nHTTP Request Anomaly Detection:")
print("=" * 60)

# Suspicious User-Agent strings
UA_SCANNER     = re.compile(
    r'\b(?:nikto|sqlmap|nmap|masscan|nessus|openvas|acunetix|'
    r'burpsuite|zap|dirbuster|gobuster|wfuzz|ffuf|nuclei|'
    r'metasploit|w3af|skipfish|whatweb|hydra|medusa)\b',
    re.IGNORECASE
)

UA_CRAWLER     = re.compile(
    r'\b(?:Googlebot|Bingbot|Slurp|DuckDuckBot|Baiduspider|'
    r'YandexBot|facebot|Twitterbot|LinkedInBot)\b'
)

UA_HEADLESS    = re.compile(
    r'\b(?:HeadlessChrome|PhantomJS|Selenium|WebDriver|Puppeteer|'
    r'playwright|cypress|testcafe)\b',
    re.IGNORECASE
)

UA_EMPTY       = re.compile(r'^$|^\s*$')

# Suspicious path patterns
PATH_SENSITIVE = re.compile(
    r'/(?:\.env|\.git|\.htaccess|\.htpasswd|wp-admin|wp-config|'
    r'phpMyAdmin|phpmyadmin|admin|administrator|manager|'
    r'config\.php|database\.yml|secrets\.yml|'
    r'backup|dump|export|database)(?:\.[a-z]+)?$',
    re.IGNORECASE
)

PATH_SHELL     = re.compile(
    r'(?:\.(php|asp|aspx|jsp|cgi|pl|py|rb|sh)$|'
    r'/(?:cmd|shell|exec|eval|system)\b)',
    re.IGNORECASE
)

PATH_TRAVERSAL = re.compile(r'\.\.[/\\]|%2e%2e[%2f%5c]', re.IGNORECASE)

# Header smuggling patterns
CONTENT_LENGTH_RE = re.compile(r'^Content-Length:\s*(\d+)', re.IGNORECASE | re.MULTILINE)
TRANSFER_ENCODING_RE = re.compile(r'^Transfer-Encoding:\s*chunked', re.IGNORECASE | re.MULTILINE)


def analyze_request(method: str, path: str, headers: Dict[str, str], body: str = '') -> Dict:
    threats = []
    ua = headers.get('User-Agent', '')

    if UA_EMPTY.match(ua):
        threats.append('empty_user_agent')
    elif UA_SCANNER.search(ua):
        threats.append('scanner_ua')
    elif UA_HEADLESS.search(ua):
        threats.append('headless_browser')

    if PATH_SENSITIVE.search(path):
        threats.append('sensitive_path')
    if PATH_SHELL.search(path):
        threats.append('shell_path')
    if PATH_TRAVERSAL.search(path):
        threats.append('path_traversal')

    raw_request = '\n'.join(f'{k}: {v}' for k, v in headers.items())
    has_cl = CONTENT_LENGTH_RE.search(raw_request)
    has_te = TRANSFER_ENCODING_RE.search(raw_request)
    if has_cl and has_te:
        threats.append('te_cl_smuggling_risk')

    return {
        'method':  method,
        'path':    path,
        'threats': threats,
        'score':   len(threats),
    }


test_requests = [
    ('GET', '/api/users', {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64)'}),
    ('GET', '/.env', {'User-Agent': 'sqlmap/1.7.8'}),
    ('POST', '/admin/config.php', {'User-Agent': 'Nikto/2.1.6'}),
    ('GET', '/../../../etc/passwd', {'User-Agent': 'curl/7.68.0'}),
    ('GET', '/wp-admin/', {'User-Agent': ''}),
    ('POST', '/api/login', {
        'User-Agent': 'Mozilla/5.0',
        'Content-Length': '42',
        'Transfer-Encoding': 'chunked',
    }),
]

print(f"\n   Request analysis:")
for method, path, headers in test_requests:
    result = analyze_request(method, path, headers)
    status = 'BLOCK' if result['score'] > 0 else 'ALLOW'
    print(f"\n   {method} {path[:40]}")
    print(f"   Status: {status}  Threats: {result['threats']}")
```

---

## 71.3 Content-Type & MIME Validation

```python
import re
from typing import Dict, Optional

print("\nContent-Type & MIME Validation:")
print("=" * 60)

CONTENT_TYPE = re.compile(
    r'^(?P<type>[a-zA-Z][a-zA-Z0-9!#$&\-^]*)'
    r'/(?P<subtype>[a-zA-Z0-9][a-zA-Z0-9!#$&\-.^_]*)'
    r'(?:\s*;\s*(?P<params>.+))?$'
)

CHARSET_PARAM = re.compile(r'\bcharset\s*=\s*(?P<charset>[^\s;]+)', re.IGNORECASE)
BOUNDARY_PARAM = re.compile(r'\bboundary\s*=\s*"?(?P<boundary>[^\s;"]+)"?', re.IGNORECASE)

ALLOWED_UPLOAD_TYPES = re.compile(
    r'^(?:image/(?:jpeg|png|gif|webp|svg\+xml)|'
    r'application/(?:pdf|json|zip)|'
    r'text/(?:plain|csv))$',
    re.IGNORECASE
)

DANGEROUS_TYPES = re.compile(
    r'^(?:application/(?:x-(?:sh|bash|csh|ksh|tcsh|php|perl|python|ruby)|'
    r'x-msdownload|octet-stream)|'
    r'text/(?:x-(?:php|python|perl|ruby|sh|bash)))$',
    re.IGNORECASE
)


def parse_content_type(ct_header: str) -> Dict:
    m = CONTENT_TYPE.match(ct_header.strip())
    if not m:
        return {'valid': False, 'raw': ct_header}
    mime    = f"{m.group('type')}/{m.group('subtype')}"
    params  = m.group('params') or ''
    charset = CHARSET_PARAM.search(params)
    boundary = BOUNDARY_PARAM.search(params)
    return {
        'valid':     True,
        'mime':      mime,
        'type':      m.group('type'),
        'subtype':   m.group('subtype'),
        'charset':   charset.group('charset') if charset else None,
        'boundary':  boundary.group('boundary') if boundary else None,
        'allowed':   bool(ALLOWED_UPLOAD_TYPES.match(mime)),
        'dangerous': bool(DANGEROUS_TYPES.match(mime)),
    }


content_types = [
    'application/json; charset=utf-8',
    'multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxkTrZu0gW',
    'image/png',
    'text/html; charset=UTF-8',
    'application/x-php',
    'text/x-python',
    'application/octet-stream',
    'image/jpeg',
    'application/pdf',
]

print(f"\n   {'Content-Type':<45} {'MIME':<25} {'Allowed':<8} {'Danger'}")
print(f"   {'-'*45} {'-'*25} {'-'*8} {'-'*6}")
for ct in content_types:
    p = parse_content_type(ct)
    if p.get('valid'):
        print(f"   {ct[:43]:<45} {p['mime']:<25} {str(p['allowed']):<8} {str(p['dangerous'])}")
```

---

## 71.4 สรุป Part 71

```
HTTP Security Pattern Recap:

1. Security headers (must-have):
   Strict-Transport-Security: max-age>=15768000
   Content-Security-Policy: no unsafe-inline/eval/*
   X-Frame-Options: DENY or SAMEORIGIN
   X-Content-Type-Options: nosniff
   Referrer-Policy: strict-origin-when-cross-origin

2. Suspicious User-Agent regex:
   Scanners: \b(nikto|sqlmap|nmap|nessus|acunetix)\b
   Headless: \b(HeadlessChrome|PhantomJS|Selenium)\b
   Empty:    ^$|\s*$

3. Sensitive path patterns:
   Files:   /\.env|\.git|\.htaccess|wp-config
   Admin:   /admin|administrator|phpMyAdmin
   Shell:   \.(php|asp|aspx|cgi)$

4. HTTP Request Smuggling indicator:
   Both Content-Length and Transfer-Encoding
   present in same request (TE-CL or CL-TE)

5. Content-Type validation:
   Allowed uploads: image/(jpeg|png|gif|webp)
                    application/(pdf|json|zip)
                    text/(plain|csv)
   Dangerous:       application/x-php
                    text/x-python
                    application/octet-stream
```

---

*[← Part 70: JWT & Token Parsing](part-70-jwt-tokens.md) | [→ Part 72: ReDoS & Performance](part-72-redos-performance.md)*
