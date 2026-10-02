# Part 58: Network & Protocol Parsing with Regex

> **ระดับ:** สูง | **เวลาเรียน:** ~75 นาที | **ข้อกำหนด:** Part 01-57

---

## 58.1 HTTP Request/Response Parsing

```python
import re
from typing import Dict, Optional, List

print("HTTP Protocol Parsing:")
print("=" * 60)

print("\n1. HTTP request parser:")

HTTP_REQUEST_LINE = re.compile(
    r'^(?P<method>GET|POST|PUT|DELETE|PATCH|HEAD|OPTIONS|TRACE|CONNECT)\s+'
    r'(?P<path>[^\s]+)\s+'
    r'HTTP/(?P<version>\d+\.\d+)$'
)

HTTP_HEADER = re.compile(
    r'^(?P<name>[A-Za-z0-9\-]+):\s*(?P<value>.+)$'
)

HTTP_STATUS_LINE = re.compile(
    r'^HTTP/(?P<version>\d+\.\d+)\s+'
    r'(?P<status>\d{3})\s+'
    r'(?P<reason>.+)$'
)


def parse_http_request(raw: str) -> Dict:
    lines = raw.strip().split('\n')
    result = {'headers': {}, 'body': ''}
    m = HTTP_REQUEST_LINE.match(lines[0].strip())
    if m:
        result.update(m.groupdict())
    i = 1
    while i < len(lines) and lines[i].strip():
        m = HTTP_HEADER.match(lines[i].strip())
        if m:
            result['headers'][m.group('name').lower()] = m.group('value')
        i += 1
    if i < len(lines):
        result['body'] = '\n'.join(lines[i+1:]).strip()
    return result


def parse_http_response(raw: str) -> Dict:
    lines = raw.strip().split('\n')
    result = {'headers': {}, 'body': ''}
    m = HTTP_STATUS_LINE.match(lines[0].strip())
    if m:
        result.update(m.groupdict())
        result['status'] = int(result['status'])
    i = 1
    while i < len(lines) and lines[i].strip():
        m = HTTP_HEADER.match(lines[i].strip())
        if m:
            result['headers'][m.group('name').lower()] = m.group('value')
        i += 1
    if i < len(lines):
        result['body'] = '\n'.join(lines[i+1:]).strip()
    return result


http_request = """GET /api/users?page=2&per_page=10 HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGci...token
Content-Type: application/json
Accept: application/json
X-Request-ID: abc-123-def

"""

parsed_req = parse_http_request(http_request)
print(f"\n   Parsed request:")
print(f"   Method:  {parsed_req.get('method')}")
print(f"   Path:    {parsed_req.get('path')}")
print(f"   Version: {parsed_req.get('version')}")
print(f"   Headers: {len(parsed_req['headers'])} headers")
for k, v in parsed_req['headers'].items():
    print(f"     {k}: {v[:40]}")

http_response = """HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 234
X-Request-ID: abc-123-def
Cache-Control: no-cache

{"users": [{"id": 1, "name": "Alice"}]}"""

parsed_resp = parse_http_response(http_response)
print(f"\n   Parsed response:")
print(f"   Status: {parsed_resp.get('status')} {parsed_resp.get('reason')}")
print(f"   Body:   {parsed_resp.get('body', '')[:50]}")
```

---

## 58.2 URL Parsing & Manipulation

```python
import re
from typing import Dict, Optional, List
from urllib.parse import unquote

print("\nURL Parsing & Manipulation:")
print("=" * 60)

URL_RE = re.compile(
    r'^(?:(?P<scheme>[a-zA-Z][a-zA-Z0-9+\-.]*):///?)'
    r'(?:(?P<user>[^:@]+)(?::(?P<password>[^@]*))?@)?'
    r'(?P<host>[a-zA-Z0-9\-._~%!$&\'()*+,;=]+)'
    r'(?::(?P<port>\d+))?'
    r'(?P<path>\/[^\?#]*)?'
    r'(?:\?(?P<query>[^#]*))?'
    r'(?:#(?P<fragment>.*))?$'
)

QUERY_PARAM = re.compile(r'([^&=]+)=([^&]*)')

def parse_url(url: str) -> Dict:
    m = URL_RE.match(url)
    if not m:
        return {}
    d = m.groupdict()
    if d.get('query'):
        params = {}
        for pm in QUERY_PARAM.finditer(d['query']):
            key   = unquote(pm.group(1))
            value = unquote(pm.group(2))
            if key in params:
                if isinstance(params[key], list):
                    params[key].append(value)
                else:
                    params[key] = [params[key], value]
            else:
                params[key] = value
        d['query_params'] = params
    if d.get('port'):
        d['port'] = int(d['port'])
    return d


urls_to_test = [
    'https://api.example.com/v2/users?page=2&sort=name&filter=active',
    'http://user:pass@localhost:8080/app/path?debug=true#section',
    'ftp://files.example.com/public/readme.txt',
    'https://example.co.th/search?q=python+regex&lang=th',
]

for url in urls_to_test:
    parsed = parse_url(url)
    print(f"\n   URL: {url[:60]}")
    for k in ['scheme', 'host', 'port', 'path', 'query_params']:
        if parsed.get(k) is not None:
            print(f"   {k:<15}: {parsed[k]}")
```

---

## 58.3 IP Address & CIDR Parsing

```python
import re
from typing import List, Tuple

print("\nIP Address Parsing:")
print("=" * 60)

IPV4_RE = re.compile(
    r'^(?P<a>(?:25[0-5]|2[0-4]\d|[01]?\d\d?))\.'
    r'(?P<b>(?:25[0-5]|2[0-4]\d|[01]?\d\d?))\.'
    r'(?P<c>(?:25[0-5]|2[0-4]\d|[01]?\d\d?))\.'
    r'(?P<d>(?:25[0-5]|2[0-4]\d|[01]?\d\d?))$'
)

CIDR_RE = re.compile(
    r'^(?P<ip>(?:(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\.){3}(?:25[0-5]|2[0-4]\d|[01]?\d\d?))'
    r'/(?P<prefix>\d{1,2})$'
)

PRIVATE_RANGES = [
    re.compile(r'^10\.'),
    re.compile(r'^172\.(1[6-9]|2\d|3[01])\.'),
    re.compile(r'^192\.168\.'),
    re.compile(r'^127\.'),
    re.compile(r'^169\.254\.'),
]


def parse_ipv4(ip: str):
    m = IPV4_RE.match(ip)
    if not m:
        return None
    return tuple(int(m.group(g)) for g in ['a', 'b', 'c', 'd'])


def is_private(ip: str) -> bool:
    return any(r.match(ip) for r in PRIVATE_RANGES)


def parse_cidr(cidr: str):
    m = CIDR_RE.match(cidr)
    if not m:
        return None
    ip = m.group('ip')
    prefix = int(m.group('prefix'))
    if prefix > 32:
        return None
    octets = parse_ipv4(ip)
    if not octets:
        return None
    ip_int    = (octets[0] << 24) | (octets[1] << 16) | (octets[2] << 8) | octets[3]
    mask      = (0xFFFFFFFF << (32 - prefix)) & 0xFFFFFFFF
    network   = ip_int & mask
    broadcast = network | (~mask & 0xFFFFFFFF)
    hosts     = max(broadcast - network - 1, 0)
    return {
        'ip':        ip,
        'prefix':    prefix,
        'network':   '.'.join(str((network >> s) & 0xFF) for s in [24, 16, 8, 0]),
        'broadcast': '.'.join(str((broadcast >> s) & 0xFF) for s in [24, 16, 8, 0]),
        'hosts':     hosts,
        'private':   is_private(ip),
    }


print(f"\n1. IPv4 validation:")
ips = ['192.168.1.1', '10.0.0.1', '8.8.8.8', '256.0.0.1', '192.168.1']
print(f"\n   {'IP':<20} {'Valid':<8} {'Octets':<22} {'Private'}")
print(f"   {'-'*20} {'-'*8} {'-'*22} {'-'*8}")
for ip in ips:
    octets  = parse_ipv4(ip)
    valid   = octets is not None
    private = is_private(ip) if valid else '-'
    print(f"   {ip:<20} {str(valid):<8} {str(octets):<22} {private}")

print(f"\n2. CIDR parsing:")
cidrs = ['192.168.1.0/24', '10.0.0.0/8', '172.16.0.0/12']
for cidr in cidrs:
    info = parse_cidr(cidr)
    if info:
        print(f"\n   {cidr}")
        print(f"   Network: {info['network']}, Broadcast: {info['broadcast']}, Hosts: {info['hosts']}")
```

---

## 58.4 Hostname Validation

```python
import re

print("\nHostname Validation:")
print("=" * 60)

HOSTNAME_RE = re.compile(
    r'^(?!-)(?:[a-zA-Z0-9](?:[a-zA-Z0-9\-]{0,61}[a-zA-Z0-9])?\.)*'
    r'[a-zA-Z0-9](?:[a-zA-Z0-9\-]{0,61}[a-zA-Z0-9])?$'
)

EMAIL_FULL = re.compile(
    r'^(?P<local>[a-zA-Z0-9._%+\-]+)'
    r'@'
    r'(?P<domain>(?:[a-zA-Z0-9\-]+\.)*[a-zA-Z0-9\-]+)'
    r'\.'
    r'(?P<tld>[a-zA-Z]{2,})$'
)

hostnames = [
    'example.com', 'sub.example.co.th', 'api.v2.example.com',
    '-invalid.com', 'valid123.org', 'x' * 64 + '.com',
]

print(f"\n   Hostname validation:")
print(f"   {'Hostname':<40} {'Valid'}")
print(f"   {'-'*40} {'-'*6}")
for h in hostnames:
    valid = bool(HOSTNAME_RE.match(h))
    display = h if len(h) < 38 else h[:35] + '...'
    print(f"   {display:<40} {valid}")

print(f"\n   Email structural parsing:")
emails = [
    'user@example.com',
    'first.last+tag@mail.example.co.th',
    'not-an-email',
]
print(f"   {'Email':<40} {'Local':<20} {'Domain':<20} {'TLD'}")
print(f"   {'-'*40} {'-'*20} {'-'*20} {'-'*6}")
for email in emails:
    m = EMAIL_FULL.match(email)
    if m:
        d = m.groupdict()
        print(f"   {email:<40} {d['local']:<20} {d['domain']:<20} {d['tld']}")
    else:
        print(f"   {email:<40} {'(invalid)'}")
```

---

## 58.5 สรุป Part 58

```
Network Protocol Regex Patterns:

1. HTTP:
   Request:  ^(METHOD) (path) HTTP/(version)$
   Header:   ^([A-Za-z0-9\-]+):\s*(.+)$
   Response: ^HTTP/(version) (status) (reason)$

2. URL components:
   scheme://user:pass@host:port/path?query#fragment
   Query params: ([^&=]+)=([^&]*)

3. IPv4:
   Octet:  25[0-5]|2[0-4]\d|[01]?\d\d?
   Full:   {octet}\.{octet}\.{octet}\.{octet}
   CIDR:   {ipv4}/(\d{1,2})

4. Private IP ranges:
   10.x.x.x          (RFC 1918 Class A)
   172.16-31.x.x      (RFC 1918 Class B)
   192.168.x.x        (RFC 1918 Class C)
   127.x.x.x          (loopback)
   169.254.x.x        (link-local)

5. Hostname:
   ^(?!-)(?:[a-zA-Z0-9](?:[a-zA-Z0-9\-]{0,61}[a-zA-Z0-9])?\.)*
   [a-zA-Z0-9](?:[a-zA-Z0-9\-]{0,61}[a-zA-Z0-9])?$
```

---

*[← Part 57: Data Pipelines & ETL](part-57-data-pipelines.md) | [→ Part 59: API Design & Documentation](part-59-api-design.md)*
