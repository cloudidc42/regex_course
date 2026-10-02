# Part 12: URL Matching — การจับคู่ URL และ URI

> **ระดับ:** พื้นฐาน-กลาง | **เวลาเรียน:** ~75 นาที | **ข้อกำหนด:** Part 01-11

---

## 12.1 โครงสร้างของ URL

```
https://user:pass@www.example.co.th:8443/path/to/page?key=value&other=1#section
  │      │    │    │                  │    │              │                │
  │      │    │    │                  │    │              query            fragment
  │      │    │    │                  │    path
  │      │    │    hostname           port
  │      │    password
  │      username
  scheme
```

```
ส่วนประกอบของ URL:
- scheme:    http, https, ftp, ws, wss, mailto, tel, ...
- authority: userinfo@host:port
- path:      /path/to/resource
- query:     ?key=value&other=1
- fragment:  #section
```

---

## 12.2 URL Patterns พื้นฐาน

```python
import re

# ============================================
# Pattern ง่าย: จับ http/https URLs
# ============================================
SIMPLE_URL = re.compile(r'https?://\S+')

text = """
Visit https://www.example.com for more info.
Also check http://dev.example.com/page?q=1#top
Not a URL: just_text or example.com (no scheme)
"""

urls = SIMPLE_URL.findall(text)
print("Simple URLs found:")
for url in urls:
    print(f"  {url}")
# Issue: จะรวม trailing punctuation (, . ! ) เข้าไปด้วย
```

### แก้ปัญหา trailing punctuation

```python
import re

# ============================================
# Pattern ที่ดีขึ้น: ไม่รวม trailing punctuation
# ============================================
BETTER_URL = re.compile(
    r'https?://'                      # scheme
    r'[a-zA-Z0-9\-._~:/?#\[\]@!$&\'()*+,;=%]+'  # valid URL chars
    r'(?<![.,;:!?"\')])'              # lookbehind: ไม่จบด้วยเครื่องหมาย
)

text2 = """
See https://example.com/path.
Also: (https://google.com/search?q=test)
And \"https://quoted.com/url!\"
"""

urls2 = BETTER_URL.findall(text2)
print("Better URLs:")
for url in urls2:
    print(f"  {url}")
```

---

## 12.3 URL Validator Class

```python
import re
from dataclasses import dataclass, field
from typing import Optional, Dict, List
from urllib.parse import urlparse, parse_qs, urlencode

@dataclass
class URLParseResult:
    is_valid: bool
    original: str
    scheme: Optional[str] = None
    username: Optional[str] = None
    password: Optional[str] = None
    host: Optional[str] = None
    port: Optional[int] = None
    path: Optional[str] = None
    query: Optional[str] = None
    fragment: Optional[str] = None
    query_params: Dict[str, List[str]] = field(default_factory=dict)
    errors: List[str] = field(default_factory=list)
    
    @property
    def normalized(self) -> Optional[str]:
        if not self.is_valid:
            return None
        parts = [f"{self.scheme}://"]
        if self.username:
            parts.append(self.username)
            if self.password:
                parts.append(f":{self.password}")
            parts.append("@")
        parts.append(self.host)
        if self.port:
            parts.append(f":{self.port}")
        parts.append(self.path or "/")
        if self.query:
            parts.append(f"?{self.query}")
        if self.fragment:
            parts.append(f"#{self.fragment}")
        return "".join(parts)


class URLValidator:
    """
    URL Validator และ Parser ด้วย Regex
    """
    
    VALID_SCHEMES = {'http', 'https', 'ftp', 'ftps', 'ws', 'wss'}
    
    # Full URL pattern
    URL_PATTERN = re.compile(r"""
        ^
        (?P<scheme>[a-zA-Z][a-zA-Z0-9+\-.]*) # scheme
        ://
        (?:
            (?P<userinfo>
                (?P<username>[^:@\s]+)
                (?::(?P<password>[^@\s]*))?
            )
            @
        )?
        (?P<host>
            (?:
                (?:[a-zA-Z0-9](?:[a-zA-Z0-9\-]{0,61}[a-zA-Z0-9])?\.)+
                [a-zA-Z]{2,}
            )
            |
            (?:\d{1,3}\.){3}\d{1,3}   # IPv4
            |
            \[(?:[0-9a-fA-F:]+)\]      # IPv6
            |
            localhost
        )
        (?::(?P<port>\d{1,5}))?       # port
        (?P<path>/[^?#\s]*)?          # path
        (?:\?(?P<query>[^#\s]*))?     # query
        (?:\#(?P<fragment>[^\s]*))?   # fragment
        $
    """, re.VERBOSE)
    
    # IP address pattern
    IPV4_PATTERN = re.compile(
        r'^(?:(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\.){3}'
        r'(?:25[0-5]|2[0-4]\d|[01]?\d\d?)$'
    )
    
    @classmethod
    def parse(cls, url: str, 
              require_scheme: bool = True,
              allowed_schemes: set = None) -> URLParseResult:
        """Parse และ validate URL"""
        
        original = url
        url = url.strip()
        result = URLParseResult(is_valid=False, original=original)
        
        if not url:
            result.errors.append("URL cannot be empty")
            return result
        
        # Add scheme ถ้าไม่มี
        if not re.match(r'^[a-zA-Z][a-zA-Z0-9+\-.]*://', url):
            if require_scheme:
                result.errors.append("Missing URL scheme (http:// or https://)")
                return result
            url = f"https://{url}"
        
        # Match pattern
        m = cls.URL_PATTERN.match(url)
        if not m:
            result.errors.append("Invalid URL format")
            return result
        
        d = m.groupdict()
        
        # Validate scheme
        scheme = d['scheme'].lower()
        allowed = allowed_schemes or cls.VALID_SCHEMES
        if scheme not in allowed:
            result.errors.append(f"Unsupported scheme: '{scheme}' (allowed: {', '.join(sorted(allowed))})")
            return result
        
        # Validate port
        port = None
        if d['port']:
            port = int(d['port'])
            if port > 65535:
                result.errors.append(f"Invalid port: {port} (max 65535)")
                return result
        
        # Validate host
        host = d['host']
        if host and not cls._validate_host(host):
            result.errors.append(f"Invalid host: '{host}'")
            return result
        
        # Parse query params
        query_params = {}
        if d['query']:
            query_params = parse_qs(d['query'], keep_blank_values=True)
        
        result.is_valid = True
        result.scheme = scheme
        result.username = d.get('username')
        result.password = d.get('password')
        result.host = host
        result.port = port
        result.path = d.get('path') or '/'
        result.query = d.get('query')
        result.fragment = d.get('fragment')
        result.query_params = query_params
        
        return result
    
    @classmethod
    def _validate_host(cls, host: str) -> bool:
        # localhost
        if host == 'localhost':
            return True
        
        # IPv4
        if cls.IPV4_PATTERN.match(host):
            return True
        
        # IPv6
        if host.startswith('[') and host.endswith(']'):
            return True
        
        # Domain name
        if not re.match(r'^[a-zA-Z0-9\-._]+$', host):
            return False
        if len(host) > 255:
            return False
        if '..' in host:
            return False
        
        labels = host.split('.')
        for label in labels:
            if not label or len(label) > 63:
                return False
            if label.startswith('-') or label.endswith('-'):
                return False
        
        return True
    
    @classmethod
    def extract_urls(cls, text: str) -> List[str]:
        """Extract URLs จาก text"""
        pattern = re.compile(
            r'(?:https?|ftp)://'
            r'[a-zA-Z0-9\-._~:/?#\[\]@!$&\'()*+,;=%]+'
            r'(?<![.,;:!?"\'])'
        )
        return pattern.findall(text)
    
    @classmethod
    def is_valid(cls, url: str) -> bool:
        return cls.parse(url).is_valid


# ============================================
# ทดสอบ URLValidator
# ============================================

test_urls = [
    "https://www.example.com",
    "http://example.com/path/to/page",
    "https://example.com:8443/secure",
    "https://user:pass@example.com/",
    "https://sub.domain.co.th/path?a=1&b=2#section",
    "ftp://files.example.com/file.txt",
    "https://192.168.1.100/admin",
    "https://localhost:3000/",
    "not-a-url",
    "javascript:alert(1)",           # invalid scheme
    "https://",                       # missing host
    "https://invalid..domain.com",   # consecutive dots
]

print("URL Validation Results:")
print("=" * 70)
for url in test_urls:
    r = URLValidator.parse(url)
    icon = '✅' if r.is_valid else '❌'
    print(f"\n{icon} {url}")
    if r.is_valid:
        print(f"   scheme={r.scheme}, host={r.host}", end="")
        if r.port:
            print(f":{r.port}", end="")
        print(f", path={r.path}", end="")
        if r.query_params:
            print(f", params={r.query_params}", end="")
        print()
    else:
        print(f"   Error: {', '.join(r.errors)}")
```

---

## 12.4 URL Pattern ต่างๆ

```python
import re

# ============================================
# 1. Extract ทุก URL จาก HTML
# ============================================
HTML_URL = re.compile(
    r'(?:href|src|action|data-url)\s*=\s*["\']([^"\']+)["\']',
    re.IGNORECASE
)

html = """
<a href="https://example.com/page">Link</a>
<img src="/images/photo.jpg" alt="photo">
<form action="/api/submit">
<script src="https://cdn.example.com/script.js"></script>
<div data-url="/api/data">
"""

print("URLs from HTML attributes:")
for m in HTML_URL.finditer(html):
    print(f"  {m.group(1)}")

# ============================================
# 2. Extract domain จาก URL
# ============================================
DOMAIN_FROM_URL = re.compile(
    r'https?://(?:www\.)?([a-zA-Z0-9\-]+\.[a-zA-Z0-9\-._]+?)(?:/|$|\?|#)'
)

urls = [
    "https://www.google.com/search?q=test",
    "http://blog.example.co.th/post/1",
    "https://api.service.io/v1/data",
]

print("\nDomains from URLs:")
for url in urls:
    m = DOMAIN_FROM_URL.search(url)
    if m:
        print(f"  {url:50} → {m.group(1)}")

# ============================================
# 3. URL Parameter manipulation
# ============================================
def add_utm_params(url: str, source: str, medium: str, campaign: str) -> str:
    """เพิ่ม UTM parameters ใน URL"""
    utm = f"utm_source={source}&utm_medium={medium}&utm_campaign={campaign}"
    
    if '?' in url:
        # ถ้ามี query string อยู่แล้ว
        if '#' in url:
            path, fragment = url.split('#', 1)
            return f"{path}&{utm}#{fragment}"
        return f"{url}&{utm}"
    else:
        # ไม่มี query string
        if '#' in url:
            path, fragment = url.split('#', 1)
            return f"{path}?{utm}#{fragment}"
        return f"{url}?{utm}"

test_url = "https://example.com/page#section"
print(f"\nURL with UTM:")
print(f"  {add_utm_params(test_url, 'newsletter', 'email', 'summer2026')}")

# ============================================
# 4. Detect URL patterns
# ============================================
def classify_url(url: str) -> dict:
    """จำแนกประเภทของ URL"""
    result = {
        'url': url,
        'is_absolute': bool(re.match(r'^https?://', url)),
        'is_relative': url.startswith('/') and not url.startswith('//'),
        'is_protocol_relative': url.startswith('//'),
        'has_query': '?' in url,
        'has_fragment': '#' in url,
        'is_api': bool(re.search(r'/api/', url)),
        'is_static': bool(re.search(r'\.(css|js|png|jpg|gif|ico|woff|woff2|ttf|svg)(\?|$)', url, re.I)),
        'is_admin': bool(re.search(r'/(?:admin|dashboard|manage|wp-admin|cpanel)', url, re.I)),
    }
    return result

sample_urls = [
    "https://api.example.com/v1/users?page=1",
    "/static/images/logo.png",
    "//cdn.example.com/script.js",
    "/admin/settings",
    "https://example.com/blog/post#comments",
]

print("\nURL Classification:")
for url in sample_urls:
    info = classify_url(url)
    features = [k for k, v in info.items() if v and k != 'url']
    print(f"  {url:50} → {features}")
```

---

## 12.5 URL Security Patterns

```python
import re
from urllib.parse import unquote

# ============================================
# Detect potentially dangerous URLs
# (for defensive/security purposes)
# ============================================

class URLSecurityChecker:
    """
    ตรวจสอบ URL ที่อาจเป็นอันตราย
    (ใช้สำหรับ defensive security)
    """
    
    # JavaScript URLs
    JS_URL = re.compile(
        r'javascript\s*:',
        re.IGNORECASE
    )
    
    # Data URLs
    DATA_URL = re.compile(
        r'data\s*:(?!image/(?:png|jpeg|gif|webp|svg\+xml))',
        re.IGNORECASE
    )
    
    # VBScript
    VBSCRIPT_URL = re.compile(
        r'vbscript\s*:',
        re.IGNORECASE
    )
    
    # Double encoding
    DOUBLE_ENCODE = re.compile(r'%25[0-9a-fA-F]{2}')
    
    # Path traversal in URL
    PATH_TRAVERSAL = re.compile(r'\.{2}[/\\]|[/\\]\.{2}')
    
    # SSRF-prone patterns (internal hosts)
    INTERNAL_IP = re.compile(
        r'https?://(?:'
        r'(?:192\.168|10\.\d+|172\.(?:1[6-9]|2\d|3[01]))\.\d+\.\d+'
        r'|localhost'
        r'|127\.0\.0\.1'
        r'|0\.0\.0\.0'
        r'|::1'
        r')',
        re.IGNORECASE
    )
    
    @classmethod
    def check(cls, url: str) -> dict:
        """ตรวจสอบ URL security"""
        
        # Decode URL encoding
        decoded = unquote(url)
        decoded_twice = unquote(decoded)
        
        issues = []
        severity = 'safe'
        
        if cls.JS_URL.search(decoded) or cls.JS_URL.search(decoded_twice):
            issues.append("JavaScript URL injection")
            severity = 'critical'
        
        if cls.VBSCRIPT_URL.search(decoded):
            issues.append("VBScript URL")
            severity = 'critical'
        
        if cls.DATA_URL.search(decoded):
            issues.append("Potentially dangerous data: URL")
            severity = 'high'
        
        if cls.DOUBLE_ENCODE.search(url):
            issues.append("Double URL encoding detected")
            severity = 'medium' if severity == 'safe' else severity
        
        if cls.PATH_TRAVERSAL.search(decoded):
            issues.append("Path traversal attempt")
            severity = 'high' if severity in ('safe', 'medium') else severity
        
        if cls.INTERNAL_IP.search(decoded):
            issues.append("Internal/private IP address (possible SSRF)")
            severity = 'high' if severity in ('safe', 'medium') else severity
        
        return {
            'url': url,
            'decoded': decoded,
            'is_safe': len(issues) == 0,
            'severity': severity,
            'issues': issues,
        }
    
    @classmethod
    def sanitize_redirect(cls, url: str, allowed_hosts: set) -> str:
        """
        Sanitize redirect URL
        - ยอมรับเฉพาะ relative URLs หรือ allowed hosts
        """
        if not url:
            return '/'
        
        # Relative URLs ที่ขึ้นต้นด้วย / ปลอดภัย
        if url.startswith('/') and not url.startswith('//'):
            # ตรวจสอบ path traversal
            if '..' in url:
                return '/'
            return url
        
        # ตรวจสอบ absolute URLs
        m = re.match(r'^https?://([^/?#]+)', url, re.IGNORECASE)
        if not m:
            return '/'
        
        host = m.group(1).lower()
        # Remove port ถ้ามี
        host = re.sub(r':\d+$', '', host)
        
        if host in allowed_hosts:
            return url
        
        # ไม่ใช่ allowed host
        return '/'


# ทดสอบ security check
security_tests = [
    "https://legitimate.com/page",
    "javascript:alert(1)",
    "JAVASCRIPT:alert(1)",           # uppercase bypass attempt
    "java%73cript:alert(1)",         # encoded bypass
    "data:text/html,<h1>XSS</h1>",
    "https://192.168.1.100/admin",   # internal IP
    "https://example.com/../../etc/passwd",  # path traversal
    "%2522javascript:alert(1)",      # double encoded
]

print("URL Security Check:")
print("=" * 70)
for url in security_tests:
    result = URLSecurityChecker.check(url)
    icon = '✅' if result['is_safe'] else '⚠️'
    print(f"\n{icon} {url}")
    if result['issues']:
        for issue in result['issues']:
            print(f"   [{result['severity'].upper()}] {issue}")

# Redirect sanitization
print("\nRedirect Sanitization:")
allowed = {'example.com', 'www.example.com', 'api.example.com'}
redirect_tests = [
    '/home',
    '/profile',
    '../admin',                    # path traversal
    'https://example.com/page',   # allowed domain
    'https://evil.com/phish',     # external domain
    '//evil.com',                 # protocol-relative
]

for url in redirect_tests:
    safe = URLSecurityChecker.sanitize_redirect(url, allowed)
    icon = '✅' if safe == url else '⚠️'
    print(f"  {icon} {url!r:40} → {safe!r}")
```

---

## 12.6 URL Patterns ใน JavaScript

```javascript
// ============================================
// JavaScript URL Validation and Parsing
// ============================================

// Modern approach: ใช้ URL API (ดีที่สุด)
function parseURL(urlString) {
    try {
        const url = new URL(urlString);
        return {
            valid: true,
            scheme: url.protocol.slice(0, -1),  // ตัด trailing ':'
            host: url.hostname,
            port: url.port || null,
            path: url.pathname,
            query: url.search.slice(1) || null,  // ตัด leading '?'
            fragment: url.hash.slice(1) || null,  // ตัด leading '#'
            params: Object.fromEntries(url.searchParams),
        };
    } catch {
        return { valid: false, error: 'Invalid URL' };
    }
}

// Regex สำหรับ extract URLs จาก text
function extractURLs(text) {
    const pattern = /https?:\/\/[a-zA-Z0-9\-._~:/?#[\]@!$&'()*+,;=%]+(?<![.,;:!?"'])/g;
    return [...text.matchAll(pattern)].map(m => m[0]);
}

// Validate URL format (ไม่ใช้ URL API)
function isValidURL(url) {
    const pattern = /^https?:\/\/(?:[a-zA-Z0-9](?:[a-zA-Z0-9\-]{0,61}[a-zA-Z0-9])?\.)+[a-zA-Z]{2,}(?::\d{1,5})?(?:\/[^\s]*)?$/;
    return pattern.test(url.trim());
}

// URL Builder
class URLBuilder {
    constructor(base) {
        this.url = new URL(base);
    }
    
    addParam(key, value) {
        this.url.searchParams.append(key, value);
        return this;
    }
    
    setParam(key, value) {
        this.url.searchParams.set(key, value);
        return this;
    }
    
    setPath(path) {
        this.url.pathname = path;
        return this;
    }
    
    build() {
        return this.url.toString();
    }
}

// ทดสอบ
const testURLs = [
    'https://www.google.com/search?q=regex&lang=th',
    'http://api.example.com:3000/v1/users',
    'not-a-valid-url',
    'javascript:alert(1)',
];

testURLs.forEach(url => {
    const result = parseURL(url);
    console.log(`${result.valid ? '✅' : '❌'} ${url}`);
    if (result.valid) {
        console.log(`   host: ${result.host}, path: ${result.path}`);
    }
});

// Builder example
const apiUrl = new URLBuilder('https://api.example.com')
    .setPath('/v1/products')
    .addParam('page', '1')
    .addParam('limit', '20')
    .addParam('category', 'electronics')
    .build();
console.log('Built URL:', apiUrl);
```

---

## 12.7 Real-World: URL Processor

```python
import re
from collections import Counter
from urllib.parse import urlparse, parse_qs, urlencode, urlunparse

def analyze_url_list(urls: list) -> dict:
    """
    วิเคราะห์ชุด URLs ว่ามี patterns อะไรบ้าง
    """
    results = {
        'total': len(urls),
        'valid': 0,
        'schemes': Counter(),
        'tlds': Counter(),
        'domains': Counter(),
        'has_params': 0,
        'has_fragments': 0,
        'https_percentage': 0,
        'sample_invalid': [],
    }
    
    URL_RE = re.compile(
        r'^(?P<scheme>https?|ftp)://(?P<host>[^/?#:\s]+)(?::(?P<port>\d+))?'
        r'(?P<path>[^?#\s]*)(?:\?(?P<query>[^#\s]*))?(?:#(?P<frag>[^\s]*))?$'
    )
    
    for url in urls:
        m = URL_RE.match(url.strip())
        if not m:
            results['sample_invalid'].append(url[:50])
            continue
        
        results['valid'] += 1
        d = m.groupdict()
        
        scheme = d['scheme'].lower()
        host = d['host'].lower()
        
        results['schemes'][scheme] += 1
        results['domains'][host] += 1
        
        # Extract TLD
        tld_m = re.search(r'\.([a-z]{2,10})$', host)
        if tld_m:
            results['tlds'][tld_m.group(1)] += 1
        
        if d['query']:
            results['has_params'] += 1
        
        if d['frag']:
            results['has_fragments'] += 1
    
    if results['valid'] > 0:
        results['https_percentage'] = (
            results['schemes'].get('https', 0) / results['valid'] * 100
        )
    
    return results


# ทดสอบ
sample_urls = [
    "https://www.google.com/search?q=python",
    "https://github.com/python/cpython",
    "http://old-site.com/legacy-page",
    "https://api.example.com/v1/data?format=json",
    "https://docs.python.org/3/library/re.html",
    "https://stackoverflow.com/questions/tagged/regex",
    "http://example.co.th/product/123",
    "https://blog.medium.com/@author/article#comments",
    "not-a-url",
    "ftp://files.example.com/download",
]

analysis = analyze_url_list(sample_urls)
print("URL Analysis Report:")
print(f"  Total:          {analysis['total']}")
print(f"  Valid:          {analysis['valid']}")
print(f"  HTTPS:          {analysis['https_percentage']:.1f}%")
print(f"  With params:    {analysis['has_params']}")
print(f"  With fragments: {analysis['has_fragments']}")
print(f"\n  Top domains:")
for domain, count in analysis['domains'].most_common(5):
    print(f"    {domain}: {count}")
print(f"\n  Top TLDs:")
for tld, count in analysis['tlds'].most_common(5):
    print(f"    .{tld}: {count}")
```

---

## 12.8 สรุป Part 12

✅ **URL Structure** — scheme, authority, path, query, fragment  
✅ **Simple extraction** — `https?://\S+` สำหรับงานทั่วไป  
✅ **Full parser** — แยกทุกส่วนด้วย named groups  
✅ **Security check** — ตรวจสอบ JavaScript URLs, double encoding  
✅ **URL API (JS)** — ดีกว่า Regex สำหรับ parsing  

### Pattern แนะนำ

```python
import re

# Extract URLs จาก text
URL_EXTRACT = re.compile(
    r'https?://[a-zA-Z0-9\-._~:/?#\[\]@!$&\'()*+,;=%]+'
    r'(?<![.,;:!?"\'])'
)

# Validate URL (ง่าย)
URL_VALIDATE = re.compile(
    r'^https?://[a-zA-Z0-9\-._]+\.[a-zA-Z]{2,}(?:/[^\s]*)?$'
)
```

---

*[⬅ Part 11: Email Validation](part-11-email-validation.md) | [➡ Part 13: Phone Numbers](part-13-phone-numbers.md)*
