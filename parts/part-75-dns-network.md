# Part 75: DNS & Network Pattern Analysis

> **ระดับ:** สูง-มืออาชีพ | **เวลาเรียน:** ~85 นาที | **ข้อกำหนด:** Part 01-74

---

## 75.1 IPv4 Address Parsing & Validation

```python
import re
from typing import Dict, List, Optional, Tuple

print("DNS & Network Pattern Analysis:")
print("=" * 60)

print("\n1. IPv4 address parsing:")

# Strict IPv4: each octet 0-255
IPV4_OCTET = r'(?:25[0-5]|2[0-4]\d|1\d{2}|[1-9]\d|\d)'
IPV4_STRICT = re.compile(
    r'^(?P<a>' + IPV4_OCTET + r')'
    r'\.(?P<b>' + IPV4_OCTET + r')'
    r'\.(?P<c>' + IPV4_OCTET + r')'
    r'\.(?P<d>' + IPV4_OCTET + r')$'
)

# CIDR notation
CIDR_V4 = re.compile(
    r'^(?P<ip>' + IPV4_OCTET + r'(?:\.' + IPV4_OCTET + r'){3})'
    r'/(?P<prefix>[12]?\d|3[0-2])$'
)

# IPv4 in free text
IPV4_IN_TEXT = re.compile(
    r'\b(?:25[0-5]|2[0-4]\d|1\d{2}|[1-9]\d|\d)'
    r'(?:\.(?:25[0-5]|2[0-4]\d|1\d{2}|[1-9]\d|\d)){3}\b'
)

# Private IP ranges
PRIVATE_RANGES = [
    (re.compile(r'^10\.'), '10.0.0.0/8 (RFC 1918)'),
    (re.compile(r'^172\.(?:1[6-9]|2\d|3[01])\.'), '172.16.0.0/12 (RFC 1918)'),
    (re.compile(r'^192\.168\.'), '192.168.0.0/16 (RFC 1918)'),
    (re.compile(r'^127\.'), '127.0.0.0/8 (loopback)'),
    (re.compile(r'^169\.254\.'), '169.254.0.0/16 (link-local)'),
    (re.compile(r'^0\.'), '0.0.0.0/8 (this network)'),
    (re.compile(r'^(?:22[4-9]|23\d)\.'), 'multicast'),
]


def classify_ipv4(ip: str) -> Dict:
    m = IPV4_STRICT.match(ip)
    if not m:
        return {'valid': False, 'ip': ip}

    result = {'valid': True, 'ip': ip, 'octets': [int(m.group(x)) for x in 'abcd']}
    for pattern, label in PRIVATE_RANGES:
        if pattern.match(ip):
            result['type'] = label
            result['public'] = False
            return result

    result['type'] = 'public'
    result['public'] = True
    return result


def cidr_to_range(cidr: str) -> Optional[Tuple[int, int]]:
    m = CIDR_V4.match(cidr)
    if not m:
        return None
    ip_parts = [int(p) for p in m.group('ip').split('.')]
    prefix   = int(m.group('prefix'))
    ip_int   = sum(p << (24 - 8 * i) for i, p in enumerate(ip_parts))
    mask     = (0xFFFFFFFF << (32 - prefix)) & 0xFFFFFFFF
    network  = ip_int & mask
    broadcast = network | (~mask & 0xFFFFFFFF)
    return network, broadcast


test_ips = [
    '192.168.1.1',
    '10.0.0.1',
    '172.16.50.100',
    '127.0.0.1',
    '8.8.8.8',
    '255.255.255.255',
    '256.0.0.1',
    '0.0.0.0',
    '169.254.1.100',
    '224.0.0.1',
]

print(f"\n   {'IP':<18} {'Valid':<7} {'Type'}")
print(f"   {'-'*18} {'-'*7} {'-'*30}")
for ip in test_ips:
    info = classify_ipv4(ip)
    status = '✓' if info['valid'] else '✗'
    kind   = info.get('type', 'invalid')
    print(f"   {ip:<18} {status:<7} {kind}")

cidrs = ['192.168.1.0/24', '10.0.0.0/8', '172.16.0.0/12', '8.8.8.0/30']
print(f"\n   CIDR ranges:")
for cidr in cidrs:
    r = cidr_to_range(cidr)
    if r:
        lo = '.'.join(str((r[0] >> (24 - 8*i)) & 0xFF) for i in range(4))
        hi = '.'.join(str((r[1] >> (24 - 8*i)) & 0xFF) for i in range(4))
        hosts = r[1] - r[0] - 1
        print(f"   {cidr:<22} {lo} → {hi} ({hosts} hosts)")
```

---

## 75.2 IPv6 Address Parsing

```python
import re
from typing import Dict

print("\nIPv6 Address Parsing:")
print("=" * 60)

# Proper IPv6 with :: handling
IPV6_PROPER = re.compile(
    r'^(?:'
    r'(?:[0-9A-Fa-f]{1,4}:){7}[0-9A-Fa-f]{1,4}|'
    r'(?:[0-9A-Fa-f]{1,4}:){1,7}:|'
    r':(?::[0-9A-Fa-f]{1,4}){1,7}|'
    r'(?:[0-9A-Fa-f]{1,4}:){1,6}:[0-9A-Fa-f]{1,4}|'
    r'(?:[0-9A-Fa-f]{1,4}:){1,5}(?::[0-9A-Fa-f]{1,4}){1,2}|'
    r'(?:[0-9A-Fa-f]{1,4}:){1,4}(?::[0-9A-Fa-f]{1,4}){1,3}|'
    r'(?:[0-9A-Fa-f]{1,4}:){1,3}(?::[0-9A-Fa-f]{1,4}){1,4}|'
    r'(?:[0-9A-Fa-f]{1,4}:){1,2}(?::[0-9A-Fa-f]{1,4}){1,5}|'
    r'[0-9A-Fa-f]{1,4}:(?::[0-9A-Fa-f]{1,4}){1,6}|'
    r'::(?:[0-9A-Fa-f]{1,4}:){0,6}[0-9A-Fa-f]{1,4}|'
    r'::|'
    r'::ffff:(?:25[0-5]|2[0-4]\d|[01]?\d\d?)(?:\.(?:25[0-5]|2[0-4]\d|[01]?\d\d?)){3}'
    r')$'
)

IPV6_LOOPBACK   = re.compile(r'^::1$')
IPV6_LINK_LOCAL = re.compile(r'^fe80:', re.IGNORECASE)
IPV6_MULTICAST  = re.compile(r'^ff', re.IGNORECASE)
IPV6_MAPPED_V4  = re.compile(r'^::ffff:', re.IGNORECASE)
IPV6_UNIQUE_LOCAL = re.compile(r'^f[cd]', re.IGNORECASE)


def classify_ipv6(addr: str) -> Dict:
    clean = addr.strip('[]').split('%')[0]  # remove scope id
    if not IPV6_PROPER.match(clean):
        return {'valid': False, 'addr': addr}

    addr_type = 'global unicast'
    if IPV6_LOOPBACK.match(clean):
        addr_type = 'loopback'
    elif IPV6_LINK_LOCAL.match(clean):
        addr_type = 'link-local'
    elif IPV6_MULTICAST.match(clean):
        addr_type = 'multicast'
    elif IPV6_MAPPED_V4.match(clean):
        addr_type = 'IPv4-mapped'
    elif IPV6_UNIQUE_LOCAL.match(clean):
        addr_type = 'unique local (ULA)'

    return {'valid': True, 'addr': clean, 'type': addr_type}


test_v6 = [
    '2001:db8::1',
    '::1',
    'fe80::1%eth0',
    'ff02::1',
    '::ffff:192.168.1.1',
    'fc00::1',
    '2001:0db8:85a3:0000:0000:8a2e:0370:7334',
    '2001:db8:85a3::8a2e:370:7334',
    '::',
    'not:valid:ipv6',
]

print(f"\n   {'IPv6 Address':<45} {'Valid':<7} {'Type'}")
print(f"   {'-'*45} {'-'*7} {'-'*25}")
for addr in test_v6:
    info = classify_ipv6(addr)
    status = '✓' if info['valid'] else '✗'
    kind = info.get('type', 'invalid')
    print(f"   {addr:<45} {status:<7} {kind}")
```

---

## 75.3 Domain Name & DNS Record Parsing

```python
import re
from typing import Dict, List

print("\nDomain Name & DNS Record Parsing:")
print("=" * 60)

# Domain name validation (RFC 1123)
DOMAIN_LABEL = re.compile(r'^[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?$')
DOMAIN_FULL   = re.compile(
    r'^(?:[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?\.)*'
    r'[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?'
    r'\.[a-zA-Z]{2,63}\.?$'
)

# DNS zone file record types
DNS_A = re.compile(
    r'^(?P<name>\S+)\s+(?P<ttl>\d+)?\s*IN\s+A\s+(?P<ip>\d+\.\d+\.\d+\.\d+)',
    re.IGNORECASE
)

DNS_MX = re.compile(
    r'^(?P<name>\S+)\s+(?P<ttl>\d+)?\s*IN\s+MX\s+(?P<priority>\d+)\s+(?P<exchange>\S+)',
    re.IGNORECASE
)

DNS_TXT = re.compile(
    r'^(?P<name>\S+)\s+(?P<ttl>\d+)?\s*IN\s+TXT\s+"(?P<value>[^"]+)"',
    re.IGNORECASE
)

DNS_CNAME = re.compile(
    r'^(?P<name>\S+)\s+(?P<ttl>\d+)?\s*IN\s+CNAME\s+(?P<target>\S+)',
    re.IGNORECASE
)

# SPF record analysis
SPF_RECORD = re.compile(r'^v=spf1\s+(?P<mechanisms>.+)$', re.IGNORECASE)
SPF_MECH   = re.compile(
    r'(?P<qualifier>[+\-~?])?'
    r'(?P<mechanism>all|include|a|mx|ip4|ip6|exists|redirect|exp)'
    r'(?::(?P<value>\S+))?',
    re.IGNORECASE
)

# DMARC record
DMARC_TAG = re.compile(r'(?P<tag>[a-z]+)=(?P<val>[^;]+)', re.IGNORECASE)


def parse_spf(txt_value: str) -> Dict:
    m = SPF_RECORD.match(txt_value)
    if not m:
        return {'valid': False}
    mechanisms = []
    for mm in SPF_MECH.finditer(m.group('mechanisms')):
        mechanisms.append({
            'qualifier': mm.group('qualifier') or '+',
            'mechanism': mm.group('mechanism').lower(),
            'value':     mm.group('value'),
        })
    return {'valid': True, 'mechanisms': mechanisms}


zone_records = [
    'example.com. 3600 IN A 93.184.216.34',
    'www 300 IN CNAME example.com.',
    'example.com. 3600 IN MX 10 mail.example.com.',
    'example.com. 3600 IN MX 20 mail2.example.com.',
    'example.com. 3600 IN TXT "v=spf1 include:_spf.google.com ip4:93.184.216.0/24 ~all"',
    '_dmarc.example.com. 3600 IN TXT "v=DMARC1; p=reject; rua=mailto:dmarc@example.com; pct=100"',
]

print(f"\n   DNS record parsing:")
for record in zone_records:
    for pattern, name in [(DNS_A, 'A'), (DNS_MX, 'MX'), (DNS_CNAME, 'CNAME'), (DNS_TXT, 'TXT')]:
        m = pattern.match(record)
        if m:
            d = m.groupdict()
            print(f"\n   [{name}] {record[:60]}")
            if name == 'TXT' and 'value' in d:
                val = d['value']
                if val.startswith('v=spf1'):
                    spf = parse_spf(val)
                    print(f"   SPF mechanisms: {[me['mechanism'] for me in spf['mechanisms']]}")
                elif val.startswith('v=DMARC1'):
                    tags = {mm.group('tag'): mm.group('val').strip()
                            for mm in DMARC_TAG.finditer(val)}
                    print(f"   DMARC policy: p={tags.get('p')}, pct={tags.get('pct')}")
            break
```

---

## 75.4 Network Security Patterns

```python
import re
from typing import Dict, List

print("\nNetwork Security Patterns:")
print("=" * 60)

# Well-known ports
WELL_KNOWN_PORTS = {
    21: 'FTP', 22: 'SSH', 23: 'Telnet', 25: 'SMTP',
    53: 'DNS', 80: 'HTTP', 110: 'POP3', 143: 'IMAP',
    443: 'HTTPS', 465: 'SMTPS', 587: 'Submission',
    993: 'IMAPS', 995: 'POP3S', 3306: 'MySQL',
    5432: 'PostgreSQL', 6379: 'Redis', 27017: 'MongoDB',
    8080: 'HTTP-alt', 8443: 'HTTPS-alt',
}

# URL with optional port
URL_WITH_PORT = re.compile(
    r'(?P<scheme>https?|ftp|ws|wss)://'
    r'(?P<host>[a-zA-Z0-9.-]+|\[[0-9A-Fa-f:]+\])'
    r'(?::(?P<port>[1-9]\d{0,4}))?'
    r'(?P<path>/[^\s?#]*)?'
    r'(?:\?(?P<query>[^\s#]*))?'
    r'(?:#(?P<fragment>\S*))?'
)

# Nmap port scan output
NMAP_PORT_LINE = re.compile(
    r'^(?P<port>\d+)/(?P<proto>tcp|udp)\s+'
    r'(?P<state>open|closed|filtered|open\|filtered)\s+'
    r'(?P<service>\S+)(?:\s+(?P<version>.+))?$'
)


def analyze_url(url: str) -> Dict:
    m = URL_WITH_PORT.match(url)
    if not m:
        return {'valid': False, 'url': url}
    port = m.group('port')
    scheme = m.group('scheme')
    if not port:
        port = '443' if scheme in ('https', 'wss') else '80'
    port_num = int(port)
    service = WELL_KNOWN_PORTS.get(port_num, f'custom-{port}')
    return {
        'valid':   True,
        'scheme':  scheme,
        'host':    m.group('host'),
        'port':    port_num,
        'service': service,
        'path':    m.group('path') or '/',
        'query':   m.group('query'),
    }


urls = [
    'https://api.example.com/v1/users?limit=10',
    'http://192.168.1.1:8080/admin',
    'https://[::1]:443/secure',
    'ws://realtime.example.com/socket',
    'http://db.internal:5432/query',
    'https://example.com:8443/app',
]

print(f"\n   URL analysis:")
for url in urls:
    info = analyze_url(url)
    if info['valid']:
        is_private = info['host'].startswith(('192.168.', '10.', '172.16.', '::1', 'db.internal'))
        risk = ' ⚠ INTERNAL' if is_private else ''
        print(f"   {url[:55]}")
        print(f"   → {info['scheme']}:{info['port']} ({info['service']}){risk}")

nmap_lines = [
    '22/tcp   open  ssh     OpenSSH 8.9p1',
    '80/tcp   open  http    nginx 1.22.0',
    '443/tcp  open  https   nginx 1.22.0',
    '3306/tcp open  mysql   MySQL 8.0.30',
    '8080/tcp open  http    Apache Tomcat',
]

print(f"\n   Nmap output parsing:")
for line in nmap_lines:
    m = NMAP_PORT_LINE.match(line)
    if m:
        port = int(m.group('port'))
        service = WELL_KNOWN_PORTS.get(port, m.group('service'))
        version = m.group('version') or ''
        risk = ' ⚠ EXPOSED' if port in (3306, 27017, 6379, 5432) else ''
        print(f"   Port {port:<6} {service:<15} {version[:30]}{risk}")
```

---

## 75.5 สรุป Part 75

```
DNS & Network Regex Patterns:

1. IPv4 validation (strict):
   Octet: 25[0-5]|2[0-4]\d|1\d{2}|[1-9]\d|\d
   Full:  (octet\.){3}octet

2. Private IP ranges (RFC 1918):
   10.0.0.0/8      → ^10\.
   172.16.0.0/12   → ^172\.(1[6-9]|2\d|3[01])\.
   192.168.0.0/16  → ^192\.168\.
   127.0.0.0/8     → ^127\. (loopback)
   169.254.0.0/16  → ^169\.254\. (link-local)

3. IPv6 special ranges:
   ::1               → loopback
   fe80::/10         → link-local
   ff00::/8          → multicast
   ::ffff:x.x.x.x    → IPv4-mapped
   fc00::/7          → unique local (ULA)

4. Domain label rules (RFC 1123):
   ^[a-zA-Z0-9]([a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?$
   Max 63 chars per label, max 253 total

5. DNS TXT security records:
   SPF:   v=spf1 <mechanisms> [~all|-all|+all]
   DMARC: v=DMARC1; p=none|quarantine|reject
   DKIM:  v=DKIM1; p=<base64 public key>

6. Port risk assessment:
   Well-known: 21,22,23,25,53,80,443,...
   Database:   3306,5432,6379,27017 → never public
   Admin:      8080,8443,9090 → verify auth
   Telnet(23): always flag as insecure
```

---

*[← Part 74: Encoding & Obfuscation](part-74-encoding-obfuscation.md) | [→ Part 76: WAF Detection Patterns](part-76-waf-detection.md)*
