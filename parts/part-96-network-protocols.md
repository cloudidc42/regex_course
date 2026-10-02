# Part 96: Regex for Network Protocol Analysis

> **ระดับ:** ผู้เชี่ยวชาญ-ระดับโลก | **เวลาเรียน:** ~110 นาที | **ข้อกำหนด:** Part 01-95

---

## 96.1 Network Address & Protocol Patterns

```python
import re
from typing import Dict, List, Optional, Tuple

print("Regex for Network Protocol Analysis:")
print("=" * 60)

print("\n1. Network address patterns:")

# IPv4 with CIDR notation
IPV4_CIDR = re.compile(
    r'^(?P<ip>(?:(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\.){3}(?:25[0-5]|2[0-4]\d|[01]?\d\d?))'
    r'(?:/(?P<prefix>[0-2]?\d|3[0-2]))?$'
)

# IPv6 patterns
IPV6_FULL = re.compile(
    r'^(?:[0-9a-fA-F]{1,4}:){7}[0-9a-fA-F]{1,4}$'
)
IPV6_COMPRESSED = re.compile(
    r'^(?:[0-9a-fA-F]{0,4}:){2,7}[0-9a-fA-F]{0,4}$'
)

# MAC address
MAC_ADDRESS = re.compile(
    r'^(?P<mac>(?:[0-9A-Fa-f]{2}[:\-]){5}[0-9A-Fa-f]{2}|'
    r'[0-9A-Fa-f]{12})$'
)

def normalize_mac(mac: str) -> Optional[str]:
    """Normalize MAC address to XX:XX:XX:XX:XX:XX format."""
    m = MAC_ADDRESS.match(mac.strip())
    if not m:
        return None
    clean = re.sub(r'[:\-]', '', m.group('mac')).upper()
    if len(clean) != 12:
        return None
    return ':'.join(clean[i:i+2] for i in range(0, 12, 2))


PORT_RANGE = re.compile(r'^(?P<port>\d{1,5})$')

def validate_port(port_str: str) -> Optional[int]:
    m = PORT_RANGE.match(port_str.strip())
    if not m:
        return None
    port = int(m.group('port'))
    return port if 1 <= port <= 65535 else None


print(f"\n   IPv4 CIDR parsing:")
cidr_tests = ['192.168.1.0/24', '10.0.0.0/8', '172.16.0.0/12', '0.0.0.0/0', '255.255.255.255/32']
for cidr in cidr_tests:
    m = IPV4_CIDR.match(cidr)
    if m:
        prefix = m.group('prefix') or '32'
        print(f"   {cidr!r} → ip={m.group('ip')}, prefix=/{prefix}")


print(f"\n   MAC address normalization:")
mac_tests = ['00:1A:2B:3C:4D:5E', '00-1a-2b-3c-4d-5e', '001A2B3C4D5E', 'invalid-mac']
for mac in mac_tests:
    normalized = normalize_mac(mac)
    print(f"   {mac!r} → {normalized or 'INVALID'}")


print(f"\n   Port validation:")
port_tests = ['80', '443', '8080', '65535', '65536', '0', 'abc']
for port in port_tests:
    result = validate_port(port)
    print(f"   {port!r} → {result if result else 'INVALID'}")
```

---

## 96.2 HTTP Protocol Analysis

```python
import re
from typing import Dict, List, Optional, Tuple

print("\nHTTP Protocol Analysis:")
print("=" * 60)

# HTTP request line
HTTP_REQUEST_LINE = re.compile(
    r'^(?P<method>GET|POST|PUT|DELETE|PATCH|HEAD|OPTIONS|TRACE|CONNECT)\s+'
    r'(?P<path>[^\s]+)\s+'
    r'HTTP/(?P<version>\d\.\d)$'
)

# HTTP header field (RFC 7230)
HTTP_HEADER = re.compile(
    r'^(?P<name>[!#$%&\'*+\-.^_`|~0-9A-Za-z]+):\s*(?P<value>.+?)\s*$'
)

# Content-Type parsing
CONTENT_TYPE = re.compile(
    r'^(?P<type>[a-zA-Z0-9!#$&\-^_]+)/(?P<subtype>[a-zA-Z0-9!#$&\-^_.+]+)'
    r'(?:\s*;\s*(?P<params>.+))?$'
)
CT_PARAM = re.compile(r'(?P<key>[a-zA-Z0-9\-_]+)=(?:"(?P<qval>[^"]+)"|(?P<val>[^;\s]+))')


def parse_http_request(raw: str) -> Dict:
    lines = raw.strip().splitlines()
    result = {'headers': {}, 'body': None}

    if lines:
        m = HTTP_REQUEST_LINE.match(lines[0])
        if m:
            result.update({
                'method': m.group('method'),
                'path': m.group('path'),
                'version': m.group('version'),
            })

    body_start = None
    for i, line in enumerate(lines[1:], 1):
        if line == '':
            body_start = i + 1
            break
        m = HTTP_HEADER.match(line)
        if m:
            result['headers'][m.group('name').lower()] = m.group('value')

    if body_start and body_start < len(lines):
        result['body'] = '\n'.join(lines[body_start:])

    return result


sample_requests = [
    """GET /api/users?page=1&limit=10 HTTP/1.1
Host: example.com
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9
Content-Type: application/json
User-Agent: Python/3.11""",

    """POST /api/login HTTP/1.1
Host: example.com
Content-Type: application/json
Content-Length: 42

{"username": "alice", "password": "secret"}""",
]

print(f"\n   HTTP request parsing:")
for raw in sample_requests:
    parsed = parse_http_request(raw)
    print(f"\n   {parsed.get('method')} {parsed.get('path')} HTTP/{parsed.get('version')}")
    for hname, hval in list(parsed['headers'].items())[:3]:
        print(f"     {hname}: {hval[:50]}")
    if parsed.get('body'):
        print(f"     Body: {parsed['body'][:40]!r}")


ct_samples = [
    'application/json',
    'text/html; charset=utf-8',
    'multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxk',
]

print(f"\n   Content-Type parsing:")
for ct in ct_samples:
    m = CONTENT_TYPE.match(ct)
    if m:
        params = {}
        if m.group('params'):
            for pm in CT_PARAM.finditer(m.group('params')):
                params[pm.group('key')] = pm.group('qval') or pm.group('val')
        print(f"   {ct!r}")
        print(f"     type={m.group('type')!r}, subtype={m.group('subtype')!r}, params={params}")
```

---

## 96.3 Network Traffic Pattern Detection

```python
import re
from typing import Dict, List

print("\nNetwork Traffic Pattern Detection:")
print("=" * 60)

# Phishing domain patterns
PHISHING_PATTERNS = [
    re.compile(r'(?:paypal|amazon|apple|microsoft|google|bank).*-(?:login|signin|verify|secure)', re.IGNORECASE),
    re.compile(r'(?:login|signin|verify|account|secure).*\.(?:info|xyz|top|click|pw|tk|ml|ga|cf)', re.IGNORECASE),
    re.compile(r'\d{1,3}-\d{1,3}-\d{1,3}-\d{1,3}'),  # IP-like domain
]

# URL structure analysis
URL_PARTS = re.compile(
    r'^(?P<scheme>https?|ftp)://'
    r'(?:(?P<userinfo>[^@]+)@)?'
    r'(?P<host>[^:/\?\#]+)'
    r'(?::(?P<port>\d+))?'
    r'(?P<path>/[^\?\#]*)?'
    r'(?:\?(?P<query>[^\#]*))?'
    r'(?:\#(?P<fragment>.*))?$'
)

def analyze_url(url: str) -> Dict:
    m = URL_PARTS.match(url)
    if not m:
        return {'error': 'invalid URL'}

    host = m.group('host') or ''
    result = {
        'scheme':   m.group('scheme'),
        'host':     host,
        'port':     m.group('port'),
        'path':     m.group('path'),
        'query':    m.group('query'),
        'fragment': m.group('fragment'),
        'userinfo': m.group('userinfo'),
    }

    flags = []
    if m.group('userinfo'):
        flags.append('CREDENTIALS_IN_URL')
    if m.group('port') and m.group('port') not in ('80', '443', '8080', '8443'):
        flags.append(f'NON_STANDARD_PORT:{m.group("port")}')
    for p in PHISHING_PATTERNS:
        if p.search(host):
            flags.append('PHISHING_DOMAIN')
            break
    if re.match(r'\d+\.\d+\.\d+\.\d+', host):
        flags.append('IP_ADDRESS_HOST')

    result['security_flags'] = flags
    return result


urls_to_analyze = [
    'https://example.com/api/v1/users?page=1',
    'http://admin:password@internal.example.com:8080/admin',
    'https://paypal-login.verify-account.xyz/secure',
    'ftp://192.168.1.100/files/data.zip',
    'https://github.com/user/repo#readme',
]

print(f"\n   URL security analysis:")
for url in urls_to_analyze:
    result = analyze_url(url)
    print(f"\n   {url!r}")
    print(f"     host={result.get('host')!r}, port={result.get('port')}")
    if result.get('security_flags'):
        for flag in result['security_flags']:
            print(f"     FLAG: {flag}")
```

---

## 96.4 สรุป Part 96

```
Regex for Network Protocol Analysis:

1. Network addressing:
   IPv4 with CIDR: validate octet ranges (0-255) in Python, not regex
   IPv6: full (7 colons), compressed (::), v4-mapped (::ffff:x.x.x.x)
   MAC: normalize to XX:XX:XX:XX:XX:XX; strip :-. separators
   Port: 1-65535; IANA well-known 0-1023, registered 1024-49151

2. HTTP parsing:
   Request line: METHOD path HTTP/version
   Header: name: value (RFC 7230 field-name = token)
   Content-Type: type/subtype; param=value
   Cookie: name=value; Path=...; Secure; HttpOnly; SameSite=...

3. DNS patterns:
   Label: [a-zA-Z0-9]([a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?
   FQDN: labels joined by dots, optionally ending with dot
   Wildcard: *.domain.tld (only leftmost label)

4. URL security analysis:
   Credentials in URL: user:pass@ → log but never store
   Non-standard port: flag for review (C2 beacons use high ports)
   IP address host: suspicious for web apps, common for IoT
   Phishing: brand-name + action-word + suspicious TLD

5. Pattern building principles:
   Build network patterns bottom-up: label → domain → URL
   Validate formats with regex, ranges in Python code
   Named groups enable structured extraction without index magic
```

---

*[← Part 95: Regex Performance Optimization](part-95-performance.md) | [→ Part 97: Regex in Database Systems](part-97-databases.md)*
