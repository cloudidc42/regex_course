# Part 74: Encoding & Obfuscation Pattern Detection

> **ระดับ:** สูง-มืออาชีพ | **เวลาเรียน:** ~90 นาที | **ข้อกำหนด:** Part 01-73

---

## 74.1 Base64 & URL Encoding Detection

```python
import re
import base64
import urllib.parse
from typing import Dict, List, Optional

print("Encoding & Obfuscation Pattern Detection:")
print("=" * 60)

print("\n1. Base64 detection and decoding:")

# Standard Base64
BASE64_STD = re.compile(r'^[A-Za-z0-9+/]+={0,2}$')
# URL-safe Base64
BASE64_URL = re.compile(r'^[A-Za-z0-9_-]+={0,2}$')
# Base64 with likely content (length multiple of 4, min 20 chars)
BASE64_LIKELY = re.compile(r'\b[A-Za-z0-9+/]{20,}={0,2}\b')


def detect_base64(text: str) -> Optional[str]:
    clean = text.strip()
    if not BASE64_STD.match(clean) and not BASE64_URL.match(clean):
        return None
    try:
        padded = clean + '=' * (4 - len(clean) % 4) if len(clean) % 4 else clean
        decoded = base64.b64decode(padded)
        return decoded.decode('utf-8', errors='replace')
    except Exception:
        try:
            decoded = base64.urlsafe_b64decode(padded)
            return decoded.decode('utf-8', errors='replace')
        except Exception:
            return None


# URL percent encoding
URL_ENCODED = re.compile(r'%[0-9A-Fa-f]{2}')
URL_ENCODED_BLOCK = re.compile(r'(?:%[0-9A-Fa-f]{2}){3,}')

# Double URL encoding
DOUBLE_ENCODED = re.compile(r'%25[0-9A-Fa-f]{2}')

# Unicode percent encoding (%uXXXX)
UNICODE_ENCODED = re.compile(r'%u[0-9A-Fa-f]{4}')


def decode_layers(text: str) -> List[Dict]:
    layers = [{'layer': 0, 'encoding': 'raw', 'value': text}]
    current = text

    for i in range(1, 4):
        decoded = urllib.parse.unquote(current)
        if decoded == current:
            break
        layers.append({'layer': i, 'encoding': 'url', 'value': decoded})
        current = decoded

    b64_matches = BASE64_LIKELY.findall(current)
    for b64 in b64_matches[:3]:
        result = detect_base64(b64)
        if result:
            layers.append({'layer': len(layers), 'encoding': 'base64', 'value': result})

    return layers


test_strings = [
    'Hello World',
    'SGVsbG8gV29ybGQ=',
    'Hello%20World',
    '%48%65%6C%6C%6F%20%57%6F%72%6C%64',
    '%2548%2565%256C%256C%256F',
    'SGVsbG8gV29ybGQgZnJvbSBCYXNlNjQ=',
    '%75%73%65%72%3d%61%64%6d%69%6e',
]

print(f"\n   {'Input':<40} {'Decoded'}")
print(f"   {'-'*40} {'-'*30}")
for s in test_strings:
    decoded = urllib.parse.unquote(s)
    b64 = detect_base64(s)
    result = b64 if b64 else (decoded if decoded != s else '-')
    print(f"   {s[:38]:<40} {result[:35]!r}")
```

---

## 74.2 HTML & Unicode Escape Detection

```python
import re
import html
import urllib.parse
from typing import Dict, List

print("\nHTML & Unicode Escape Detection:")
print("=" * 60)

# HTML entities
HTML_NAMED  = re.compile(r'&(?P<name>[a-zA-Z][a-zA-Z0-9]{1,31});')
HTML_DEC    = re.compile(r'&#(?P<dec>\d{1,6});')
HTML_HEX    = re.compile(r'&#x(?P<hex>[0-9A-Fa-f]{1,6});')
HTML_ENTITY = re.compile(r'&(?:#(?:x[0-9A-Fa-f]{1,6}|\d{1,6})|[a-zA-Z][a-zA-Z0-9]{1,31});')

# Unicode escapes (various formats)
UNICODE_JS   = re.compile(r'\\u(?P<cp>[0-9A-Fa-f]{4})')
UNICODE_PY   = re.compile(r'\\u(?P<cp>[0-9A-Fa-f]{4})|\\U(?P<cp8>[0-9A-Fa-f]{8})')
UNICODE_CSS  = re.compile(r'\\(?P<cp>[0-9A-Fa-f]{1,6})\s?')

# Obfuscated script detection after entity decoding
SCRIPT_PATTERNS = re.compile(r'<\s*script|javascript:|eval\s*\(|document\.write', re.IGNORECASE)

# URL encoded
URL_ENCODED = re.compile(r'%[0-9A-Fa-f]{2}')
DOUBLE_ENCODED = re.compile(r'%25[0-9A-Fa-f]{2}')


def decode_html_entities(text: str) -> str:
    return html.unescape(text)


def decode_unicode_escapes(text: str) -> str:
    def replace_js_unicode(m):
        return chr(int(m.group('cp'), 16))
    return UNICODE_JS.sub(replace_js_unicode, text)


def detect_obfuscation(text: str) -> Dict:
    findings = {
        'html_entities': len(HTML_ENTITY.findall(text)),
        'unicode_escapes': len(UNICODE_JS.findall(text)),
        'url_encoded': len(URL_ENCODED.findall(text)),
        'double_encoded': bool(DOUBLE_ENCODED.search(text)),
    }

    decoded = decode_html_entities(text)
    decoded = decode_unicode_escapes(decoded)
    decoded_url = urllib.parse.unquote(decoded)

    findings['decoded_threat'] = bool(SCRIPT_PATTERNS.search(decoded_url))
    findings['decoded_value']  = decoded_url[:100]
    return findings


obfuscated_samples = [
    'Hello &amp; World &lt;b&gt;bold&lt;/b&gt;',
    '&lt;script&gt;alert(1)&lt;/script&gt;',
    '&#60;script&#62;alert(1)&#60;/script&#62;',
    '&#x3c;script&#x3e;alert(1)&#x3c;/script&#x3e;',
    r'\u003cscript\u003ealert(1)\u003c/script\u003e',
    r'\u006A\u0061\u0076\u0061\u0073\u0063\u0072\u0069\u0070\u0074\u003a\u0061\u006C\u0065\u0072\u0074\u0028\u0031\u0029',
    '%3Cscript%3Ealert(1)%3C/script%3E',
    '%253Cscript%253Ealert(1)',
]

print(f"\n   Obfuscation detection:")
for sample in obfuscated_samples:
    info = detect_obfuscation(sample)
    threat = ' \u26a0 THREAT' if info['decoded_threat'] else ''
    print(f"\n   {sample[:55]!r}")
    print(f"   HTML:{info['html_entities']} Unicode:{info['unicode_escapes']} URL:{info['url_encoded']} Double:{info['double_encoded']}{threat}")
    if info['decoded_threat']:
        print(f"   Decoded: {info['decoded_value'][:60]!r}")
```

---

## 74.3 JavaScript Obfuscation Patterns

```python
import re
from typing import Dict, List

print("\nJavaScript Obfuscation Patterns:")
print("=" * 60)

# eval() with encoded argument
EVAL_ENCODED = re.compile(
    r'\beval\s*\(\s*(?:unescape|decodeURIComponent|atob|String\.fromCharCode)\s*\(',
    re.IGNORECASE
)

# String.fromCharCode obfuscation
FROM_CHAR_CODE = re.compile(
    r'String\.fromCharCode\s*\((?:\s*\d+\s*,?)+\)',
    re.IGNORECASE
)

# Extract char codes
CHAR_CODES = re.compile(r'\d+')

# Hex string concatenation
HEX_STRING = re.compile(r'(?:\\x[0-9A-Fa-f]{2})+')

# Prototype pollution
PROTO_POLLUTION = re.compile(r'__proto__|constructor\[|prototype\[', re.IGNORECASE)

# Suspicious function calls
SUSPICIOUS_CALLS = re.compile(
    r'\b(?:eval|setTimeout|setInterval|Function|execScript|'
    r'document\.write|window\.location|document\.cookie)\s*\(',
    re.IGNORECASE
)


def decode_fromcharcode(code_str: str) -> str:
    codes = [int(n) for n in CHAR_CODES.findall(code_str)]
    try:
        return ''.join(chr(c) for c in codes if 0 <= c <= 0x10FFFF)
    except ValueError:
        return ''


def decode_hex_string(hex_str: str) -> str:
    return re.sub(r'\\x([0-9A-Fa-f]{2})',
                  lambda m: chr(int(m.group(1), 16)),
                  hex_str)


def analyze_js(code: str) -> Dict:
    findings = []
    if EVAL_ENCODED.search(code):
        findings.append('eval with encoded arg')
    if FROM_CHAR_CODE.search(code):
        findings.append('String.fromCharCode')
        for m in FROM_CHAR_CODE.finditer(code):
            decoded = decode_fromcharcode(m.group(0))
            if decoded:
                findings.append(f'  decoded: {decoded[:50]!r}')
    if HEX_STRING.search(code):
        findings.append('hex string escapes')
    if PROTO_POLLUTION.search(code):
        findings.append('prototype pollution')
    for m in SUSPICIOUS_CALLS.finditer(code):
        findings.append(f'suspicious call: {m.group(0)[:30]}')
    return {'findings': findings, 'risk': len(findings)}


js_samples = [
    'eval(unescape("%61%6c%65%72%74%28%31%29"))',
    "String.fromCharCode(97,108,101,114,116,40,49,41)",
    'var x = "\\x61\\x6c\\x65\\x72\\x74\\x28\\x31\\x29"',
    'document.write(String.fromCharCode(60,115,99,114,105,112,116,62))',
    'setTimeout(function(){eval(atob("YWxlcnQoMSk="))}, 1000)',
    'Object.prototype.__proto__ = {}',
    'window["eval"]("alert(1)")',
]

print(f"\n   JavaScript obfuscation analysis:")
for code in js_samples:
    result = analyze_js(code)
    print(f"\n   {code[:60]!r}")
    for f in result['findings']:
        print(f"   \u26a0  {f}")
    if not result['findings']:
        print(f"   \u2713 No obfuscation detected")
```

---

## 74.4 Encoding Chain Tracing

```python
import re
import base64
import html
import urllib.parse
from typing import List, Dict

print("\nEncoding Chain Tracing:")
print("=" * 60)

BASE64_STD = re.compile(r'^[A-Za-z0-9+/]+={0,2}$')


def trace_encoding_chain(payload: str, max_depth: int = 5) -> List[Dict]:
    chain = []
    current = payload

    for depth in range(max_depth):
        # Try URL decode
        decoded = urllib.parse.unquote(current)
        if decoded != current:
            chain.append({'step': depth, 'encoding': 'url', 'value': decoded[:60]})
            current = decoded
            continue

        # Try Base64
        if BASE64_STD.match(current.strip()):
            try:
                padded = current.strip()
                if len(padded) % 4:
                    padded += '=='
                b64_decoded = base64.b64decode(padded).decode('utf-8', errors='replace')
                if len(b64_decoded) > 3 and sum(1 for c in b64_decoded if c.isprintable()) / max(len(b64_decoded),1) > 0.8:
                    chain.append({'step': depth, 'encoding': 'base64', 'value': b64_decoded[:60]})
                    current = b64_decoded
                    continue
            except Exception:
                pass

        # Try HTML entity decode
        html_decoded = html.unescape(current)
        if html_decoded != current:
            chain.append({'step': depth, 'encoding': 'html_entity', 'value': html_decoded[:60]})
            current = html_decoded
            continue

        break

    return chain


payloads_to_trace = [
    'SGVsbG8gV29ybGQ=',
    'JTNDc2NyaXB0JTNFYWxlcnQoMSklM0MlMkZzY3JpcHQlM0U=',
    '%3Cscript%3Ealert%281%29%3C%2Fscript%3E',
    '%2525%2541%2544%254D%2549%254E',
]

print(f"\n   Encoding chain tracing:")
for payload in payloads_to_trace:
    chain = trace_encoding_chain(payload)
    print(f"\n   Original: {payload[:50]!r}")
    for step in chain:
        print(f"   Step {step['step']+1} [{step['encoding']}]: {step['value']!r}")
    if not chain:
        print(f"   No encoding chain detected")

print(f"\n   XOR brute-force demo:")
message = b'Hello, World!'
xor_key = 0x42
encoded_xor = bytes(b ^ xor_key for b in message)
print(f"   XOR-encoded bytes (key=0x{xor_key:02X}): {encoded_xor.hex()}")

# Brute-force key
for key in range(1, 256):
    decoded = bytes(b ^ key for b in encoded_xor)
    try:
        text = decoded.decode('utf-8')
        printable = sum(1 for c in text if c.isprintable())
        if printable / len(text) > 0.9:
            print(f"   Key=0x{key:02X}: {text!r}")
            break
    except UnicodeDecodeError:
        pass
```

---

## 74.5 สรุป Part 74

```
Encoding & Obfuscation Detection:

1. Base64 indicators:
   - Length divisible by 4 (or padded with =)
   - Characters: [A-Za-z0-9+/=] or [A-Za-z0-9_-=] (URL-safe)
   - Multiple decode iterations for nested encoding

2. URL encoding tiers:
   Single:  %41 → 'A'
   Double:  %2541 → '%41' → 'A'
   Triple:  %252541 → '%2541' → '%41' → 'A'
   Pattern: %25 prefix = double-encoded %

3. HTML entities:
   Named:   &amp; &lt; &gt; &quot;
   Decimal: &#60; &#62;  (< >)
   Hex:     &#x3c; &#x3e; (< >)
   All decode to same characters — used to bypass filters

4. Unicode escapes:
   JS:  \u0041 → 'A'
   CSS: \41 or \000041 → 'A'
   Always decode before XSS scanning

5. fromCharCode obfuscation:
   String.fromCharCode(97,108,101,114,116) → 'alert'
   Extract codes, map to chr()

6. Encoding chain defense:
   Decode ALL layers before validating
   Max depth: 3-5 iterations
   After decoding: apply XSS/SQLi/CMDi checks

7. XOR encoding:
   Simple single-byte XOR: byte ^ key
   Brute force: try all 255 keys, check printability
   Indicators: repeating byte pattern, non-ASCII entropy
```

---

*[← Part 73: Email & Communication Parsing](part-73-email-parsing.md) | [→ Part 75: DNS & Network Patterns](part-75-dns-network.md)*
