# Part 29: Network & Protocol Patterns — เครือข่ายและโปรโตคอล

> **ระดับ:** กลาง | **เวลาเรียน:** ~70 นาที | **ข้อกำหนด:** Part 01-28

---

## 29.1 IP Address Patterns

```python
import re
from typing import Optional, Tuple

class IPValidator:
    """ตรวจสอบ IP addresses ทุกรูปแบบ"""
    
    IPV4_OCTET = r'(?:25[0-5]|2[0-4]\d|1\d{2}|[1-9]\d|\d)'
    IPV4 = re.compile(
        r'\b(?:25[0-5]|2[0-4]\d|1\d{2}|[1-9]\d|\d)'
        r'(?:\.(?:25[0-5]|2[0-4]\d|1\d{2}|[1-9]\d|\d)){3}\b'
    )
    
    IPV4_CIDR = re.compile(
        r'\b(?:25[0-5]|2[0-4]\d|1\d{2}|[1-9]\d|\d)'
        r'(?:\.(?:25[0-5]|2[0-4]\d|1\d{2}|[1-9]\d|\d)){3}'
        r'/(?:[12]?\d|3[0-2])\b'
    )
    
    IPV6 = re.compile(
        r'(?:'
        r'(?:[0-9a-fA-F]{1,4}:){7}[0-9a-fA-F]{1,4}'
        r'|(?:[0-9a-fA-F]{1,4}:){1,7}:'
        r'|:(?::[0-9a-fA-F]{1,4}){1,7}'
        r'|(?:[0-9a-fA-F]{1,4}:){1,6}:[0-9a-fA-F]{1,4}'
        r'|::1'
        r')'
    )
    
    PRIVATE = re.compile(
        r'^(?:'
        r'10\.\d{1,3}\.\d{1,3}\.\d{1,3}'
        r'|172\.(?:1[6-9]|2\d|3[01])\.\d{1,3}\.\d{1,3}'
        r'|192\.168\.\d{1,3}\.\d{1,3}'
        r'|127\.\d{1,3}\.\d{1,3}\.\d{1,3}'
        r'|169\.254\.\d{1,3}\.\d{1,3}'
        r')$'
    )
    
    @classmethod
    def is_private(cls, ip: str) -> bool:
        return bool(cls.PRIVATE.match(ip))
    
    @classmethod
    def extract_all(cls, text: str) -> dict:
        return {
            'ipv4': cls.IPV4.findall(text),
            'ipv4_cidr': cls.IPV4_CIDR.findall(text),
            'ipv6': cls.IPV6.findall(text),
        }
    
    @classmethod
    def get_class(cls, ip: str) -> str:
        m = re.match(r'^(\d+)', ip)
        if not m:
            return 'Unknown'
        first = int(m.group(1))
        if first < 128:   return 'A'
        if first < 192:   return 'B'
        if first < 224:   return 'C'
        if first < 240:   return 'D (Multicast)'
        return 'E (Reserved)'


# ทดสอบ
sample_text = """
Network topology:
- Public server: 203.0.113.1/24
- Internal gateway: 192.168.1.1
- App servers: 10.0.1.10, 10.0.1.11, 10.0.1.12
- IPv6 endpoint: 2001:0db8:85a3:0000:0000:8a2e:0370:7334
- Loopback: 127.0.0.1 (::1)
- Broadcast: 255.255.255.255
- Link-local: 169.254.0.1
"""

validator = IPValidator
extracted = validator.extract_all(sample_text)

print("IP Address Extraction:")
print("=" * 60)

for ip_type, ips in extracted.items():
    if ips:
        print(f"\n  {ip_type.upper()}:")
        for ip in ips:
            private = " [PRIVATE]" if validator.is_private(ip) else ""
            class_ = validator.get_class(ip) if '.' in ip and '/' not in ip else ""
            class_info = f" (Class {class_})" if class_ else ""
            print(f"    {ip}{private}{class_info}")
```

---

## 29.2 Network Protocol Parsers

```python
import re
from typing import Dict, List, Optional

class ProtocolParser:
    """Parse network protocol messages"""
    
    HTTP_REQUEST = re.compile(
        r'^(?P<method>GET|POST|PUT|DELETE|PATCH|HEAD|OPTIONS|TRACE|CONNECT)'
        r'\s+(?P<path>[^\s]+)'
        r'\s+HTTP/(?P<version>\d+\.\d+)$',
        re.IGNORECASE
    )
    
    HTTP_RESPONSE = re.compile(
        r'^HTTP/(?P<version>\d+\.\d+)'
        r'\s+(?P<code>\d{3})'
        r'\s+(?P<reason>.+)$'
    )
    
    HTTP_HEADER = re.compile(
        r'^(?P<name>[A-Za-z][A-Za-z0-9-]*)'
        r'\s*:\s*(?P<value>.+)$'
    )
    
    DNS_RECORD = re.compile(
        r'^(?P<name>[\w.-]+)\s+'
        r'(?P<ttl>\d+)\s+'
        r'(?P<class>IN)\s+'
        r'(?P<type>A|AAAA|CNAME|MX|TXT|NS|PTR|SOA|SRV)'
        r'\s+(?P<data>.+)$',
        re.MULTILINE
    )
    
    @classmethod
    def parse_http_request(cls, request_line: str) -> Optional[Dict]:
        m = cls.HTTP_REQUEST.match(request_line.strip())
        if not m:
            return None
        return {
            'method': m.group('method').upper(),
            'path': m.group('path'),
            'version': m.group('version'),
        }
    
    @classmethod
    def parse_http_headers(cls, headers_text: str) -> Dict[str, str]:
        headers = {}
        for line in headers_text.split('\n'):
            m = cls.HTTP_HEADER.match(line.strip())
            if m:
                name = m.group('name').title()
                headers[name] = m.group('value').strip()
        return headers
    
    @classmethod
    def parse_dns_records(cls, zone_text: str) -> List[Dict]:
        records = []
        for m in cls.DNS_RECORD.finditer(zone_text):
            records.append({
                'name': m.group('name'),
                'ttl': int(m.group('ttl')),
                'type': m.group('type'),
                'data': m.group('data').strip(),
            })
        return records


# ทดสอบ
http_requests = [
    "GET /api/users HTTP/1.1",
    "POST /auth/login HTTP/1.1",
    "DELETE /api/users/123 HTTP/2.0",
    "INVALID REQUEST",
]

print("HTTP Request Parser:")
print("=" * 60)
parser = ProtocolParser

for req in http_requests:
    parsed = parser.parse_http_request(req)
    if parsed:
        print(f"  {parsed['method']:8} {parsed['path']:<30} HTTP/{parsed['version']}")
    else:
        print(f"  [INVALID] {req}")

headers_text = """
Host: api.example.com
Content-Type: application/json
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.xxx
X-Request-ID: abc-123-def
Accept-Language: th-TH, en;q=0.9
"""
headers = parser.parse_http_headers(headers_text)
print(f"\nParsed HTTP Headers:")
for k, v in headers.items():
    if 'Authorization' in k:
        v = re.sub(r'(Bearer\s+)\S+', r'\1[REDACTED]', v)
    print(f"  {k}: {v}")

dns_zone = """
example.com.      3600  IN  A      93.184.216.34
www               3600  IN  CNAME  example.com.
mail              3600  IN  MX     10 mail.example.com.
example.com.      3600  IN  TXT    "v=spf1 include:_spf.example.com ~all"
"""

records = parser.parse_dns_records(dns_zone)
print(f"\nDNS Records:")
for r in records:
    print(f"  {r['type']:6} {r['name']:30} {r['data'][:40]}")
```

---

## 29.3 Firewall Log Analysis

```python
import re
from typing import List, Dict
from collections import Counter

class FirewallLogParser:
    """Parse firewall/IDS log entries"""
    
    IPTABLES = re.compile(
        r'(?P<timestamp>\w+\s+\d+\s+\d+:\d+:\d+)\s+'
        r'(?P<host>\S+)\s+kernel:\s+'
        r'(?:(?P<prefix>[A-Z_]+)\s+)?'
        r'IN=(?P<in_if>\S*)\s+OUT=(?P<out_if>\S*)'
        r'(?:\s+MAC=(?P<mac>\S+))?'
        r'\s+SRC=(?P<src>[^\s]+)\s+DST=(?P<dst>[^\s]+)'
        r'\s+LEN=(?P<len>\d+)\s+.*?'
        r'PROTO=(?P<proto>\w+)'
        r'(?:\s+SPT=(?P<spt>\d+)\s+DPT=(?P<dpt>\d+))?'
    )
    
    WELL_KNOWN_PORTS = {
        21: 'FTP', 22: 'SSH', 23: 'Telnet', 25: 'SMTP', 53: 'DNS',
        80: 'HTTP', 110: 'POP3', 143: 'IMAP', 443: 'HTTPS',
        3306: 'MySQL', 5432: 'PostgreSQL', 6379: 'Redis', 27017: 'MongoDB',
        3389: 'RDP', 1433: 'MSSQL', 8080: 'HTTP-Alt', 8443: 'HTTPS-Alt',
    }
    
    @classmethod
    def parse_iptables(cls, log_text: str) -> List[Dict]:
        entries = []
        for m in cls.IPTABLES.finditer(log_text):
            d = m.groupdict()
            dpt = int(d['dpt']) if d.get('dpt') else None
            entries.append({
                'timestamp': d['timestamp'],
                'action': d.get('prefix', 'UNKNOWN'),
                'src_ip': d['src'],
                'dst_ip': d['dst'],
                'proto': d['proto'],
                'dst_port': dpt,
                'service': cls.WELL_KNOWN_PORTS.get(dpt, 'Unknown') if dpt else None,
            })
        return entries
    
    @classmethod
    def analyze_threats(cls, entries: List[Dict]) -> Dict:
        blocked = [e for e in entries if 'DROP' in (e.get('action') or '')]
        src_counts = Counter(e['src_ip'] for e in blocked)
        src_port_scan = {}
        for e in blocked:
            src = e['src_ip']
            if src not in src_port_scan:
                src_port_scan[src] = set()
            if e['dst_port']:
                src_port_scan[src].add(e['dst_port'])
        scanners = {ip: ports for ip, ports in src_port_scan.items() if len(ports) > 2}
        return {
            'total_blocked': len(blocked),
            'top_sources': src_counts.most_common(5),
            'port_scanners': {ip: list(p) for ip, p in scanners.items()},
        }


# ทดสอบ
iptables_log = """Oct  2 14:30:01 fw01 kernel: DROP_INPUT IN=eth0 OUT= MAC=aa:bb:cc:dd:ee:ff SRC=192.0.2.100 DST=10.0.0.1 LEN=60 TTL=64 PROTO=TCP SPT=54321 DPT=22
Oct  2 14:30:02 fw01 kernel: DROP_INPUT IN=eth0 OUT= SRC=192.0.2.100 DST=10.0.0.1 LEN=60 TTL=64 PROTO=TCP SPT=54322 DPT=3306
Oct  2 14:30:03 fw01 kernel: DROP_INPUT IN=eth0 OUT= SRC=192.0.2.100 DST=10.0.0.1 LEN=60 TTL=64 PROTO=TCP SPT=54323 DPT=5432
Oct  2 14:30:04 fw01 kernel: ACCEPT IN=eth0 OUT= SRC=10.0.1.5 DST=10.0.0.1 LEN=60 PROTO=TCP SPT=48000 DPT=80
Oct  2 14:30:05 fw01 kernel: DROP_INPUT IN=eth0 OUT= SRC=198.51.100.200 DST=10.0.0.1 LEN=60 PROTO=TCP SPT=12345 DPT=443
"""

parser = FirewallLogParser
entries = parser.parse_iptables(iptables_log)

print("Firewall Log Analysis:")
print("=" * 60)
print(f"\nTotal entries: {len(entries)}")

for e in entries:
    action = e['action']
    icon = '✗' if 'DROP' in action else '✓'
    svc = f"[{e['service']}]" if e['service'] else ''
    print(f"  {icon} {e['src_ip']:<18} -> :{e['dst_port']:<6} {svc}")

analysis = parser.analyze_threats(entries)
print(f"\nThreat Analysis:")
print(f"  Total blocked: {analysis['total_blocked']}")
print(f"  Top attackers: {analysis['top_sources']}")
if analysis['port_scanners']:
    print(f"  Port scanners: {analysis['port_scanners']}")
```

---

## 29.4 สรุป Part 29

```
Network Patterns:
IPv4:      \b(?:25[0-5]|2[0-4]\d|1\d{2}|[1-9]\d|\d)(?:\.same){3}\b
IPv4/CIDR: same + /(?:[12]?\d|3[0-2])
Private:   ^(?:10\.|172\.(?:1[6-9]|2\d|3[01])\.|192\.168\.|127\.)
HTTP req:  ^(GET|POST|...) (/path) HTTP/(\d+\.\d+)$
HTTP hdr:  ^([A-Za-z][A-Za-z0-9-]*)\s*:\s*(.+)$
DNS:       ^([\w.-]+)\s+(\d+)\s+IN\s+(A|AAAA|CNAME|MX|...)\s+(.+)$
iptables:  SRC=(\S+)\s+DST=(\S+).*PROTO=(\w+).*SPT=(\d+)\s+DPT=(\d+)

Security:
- ตรวจสอบ IP ranges (private vs public)
- ระบุ port scan patterns
- Log analysis for anomaly detection
- Mask credentials in log output
```

---

*[← Part 28: Database Patterns](part-28-database-patterns.md) | [→ Part 30: Date & Time](part-30-datetime-patterns.md)*
