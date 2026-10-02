# Part 99: Regex for Log Forensics & SIEM

> **ระดับ:** ผู้เชี่ยวชาญ-ระดับโลก | **เวลาเรียน:** ~120 นาที | **ข้อกำหนด:** Part 01-98

---

## 99.1 Multi-Format Log Parsing

```python
import re
from typing import Dict, List, Optional
from datetime import datetime

print("Regex for Log Forensics & SIEM:")
print("=" * 60)

print("\n1. Multi-format log parsing:")

# Syslog RFC 5424
SYSLOG_RFC5424 = re.compile(
    r'^<(?P<priority>\d{1,3})>'
    r'(?P<version>\d)\s+'
    r'(?P<timestamp>\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}(?:\.\d+)?(?:Z|[+-]\d{2}:\d{2}))\s+'
    r'(?P<hostname>\S+)\s+'
    r'(?P<appname>\S+)\s+'
    r'(?P<procid>\S+)\s+'
    r'(?P<msgid>\S+)\s+'
    r'(?P<structured_data>-|\[.+?\])\s+'
    r'(?P<message>.+)$'
)

# Traditional syslog RFC 3164
SYSLOG_RFC3164 = re.compile(
    r'^<(?P<priority>\d{1,3})>'
    r'(?P<timestamp>(?:Jan|Feb|Mar|Apr|May|Jun|Jul|Aug|Sep|Oct|Nov|Dec)\s+\d{1,2}\s+\d{2}:\d{2}:\d{2})\s+'
    r'(?P<hostname>\S+)\s+'
    r'(?P<process>[^[:\s]+)(?:\[(?P<pid>\d+)\])?\s*:\s*'
    r'(?P<message>.+)$'
)

# CEF (Common Event Format) for SIEM
CEF_HEADER = re.compile(
    r'^CEF:(?P<version>\d+)\|'
    r'(?P<device_vendor>[^|]*)\|'
    r'(?P<device_product>[^|]*)\|'
    r'(?P<device_version>[^|]*)\|'
    r'(?P<signature_id>[^|]*)\|'
    r'(?P<name>[^|]*)\|'
    r'(?P<severity>[^|]*)\|'
    r'(?P<extension>.*)$'
)
CEF_EXTENSION = re.compile(r'(?P<key>[a-zA-Z][a-zA-Z0-9_]*)=(?P<value>(?:[^\\=]|\\.)*?)(?=\s+[a-zA-Z][a-zA-Z0-9_]*=|$)')


def parse_syslog(line: str) -> Optional[Dict]:
    for pattern in (SYSLOG_RFC5424, SYSLOG_RFC3164):
        m = pattern.match(line)
        if m:
            d = m.groupdict()
            pri = int(d.get('priority', 0))
            d['facility'] = pri >> 3
            d['severity'] = pri & 7
            return d
    return None


def parse_cef(line: str) -> Optional[Dict]:
    m = CEF_HEADER.match(line)
    if not m:
        return None
    result = m.groupdict()
    ext = {}
    for em in CEF_EXTENSION.finditer(result.get('extension', '')):
        ext[em.group('key')] = em.group('value').replace('\\=', '=').replace('\\|', '|')
    result['extension'] = ext
    return result


# Sample log lines
syslog_samples = [
    '<34>1 2024-01-15T10:30:01.123Z web01 sshd 1234 - - Failed password for alice from 10.0.0.5 port 22 ssh2',
    '<165>Jan 15 10:30:01 web01 kernel[0]: Out of memory: Kill process 1234 (python) score 900 or sacrifice child',
]

cef_sample = 'CEF:0|Cisco|ASA|9.14|106001|Inbound TCP connection denied|5|src=192.168.1.50 dst=10.10.10.1 spt=12345 dpt=443 proto=TCP'

print("\n   Syslog parsing:")
for line in syslog_samples:
    parsed = parse_syslog(line)
    if parsed:
        print(f"\n   {line[:70]!r}...")
        print(f"     hostname={parsed.get('hostname')!r}, message={parsed.get('message')[:50]!r}")

print("\n   CEF parsing:")
cef = parse_cef(cef_sample)
if cef:
    print(f"   Product: {cef['device_product']}, Name: {cef['name']}, Severity: {cef['severity']}")
    print(f"   Extension: {cef['extension']}")
```

---

## 99.2 Security Event Correlation

```python
import re
from typing import Dict, List, Tuple
from collections import defaultdict, Counter
from datetime import datetime, timedelta

print("\nSecurity Event Correlation:")
print("=" * 60)

# Authentication event patterns
AUTH_SUCCESS = re.compile(
    r'(?P<ts>\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2})'
    r'.*?(?:Accepted|successful\s+login|authentication\s+successful)'
    r'.*?(?:for\s+(?P<user>\w+)|user[=:\s]+(?P<user2>\w+))'
    r'.*?(?:from\s+(?P<src_ip>\d{1,3}(?:\.\d{1,3}){3}))',
    re.IGNORECASE
)
AUTH_FAILURE = re.compile(
    r'(?P<ts>\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2})'
    r'.*?(?:Failed|Invalid|failed\s+login|authentication\s+failed)'
    r'.*?(?:for\s+(?P<user>\w+)|user[=:\s]+(?P<user2>\w+))'
    r'.*?(?:from\s+(?P<src_ip>\d{1,3}(?:\.\d{1,3}){3}))',
    re.IGNORECASE
)


def analyze_auth_events(log_lines: List[str]) -> Dict:
    failures: Dict[str, List[str]] = defaultdict(list)
    successes: Dict[str, List[str]] = defaultdict(list)

    for line in log_lines:
        m = AUTH_FAILURE.search(line)
        if m:
            ip = m.group('src_ip') or ''
            user = m.group('user') or m.group('user2') or 'unknown'
            failures[ip].append(user)
            continue
        m = AUTH_SUCCESS.search(line)
        if m:
            ip = m.group('src_ip') or ''
            user = m.group('user') or m.group('user2') or 'unknown'
            successes[ip].append(user)

    alerts = []
    for ip, fail_users in failures.items():
        if len(fail_users) >= 5:
            alerts.append({
                'type': 'BRUTE_FORCE',
                'ip': ip,
                'attempts': len(fail_users),
                'users_tried': list(set(fail_users))[:5],
            })
        if ip in successes and len(fail_users) >= 3:
            alerts.append({
                'type': 'BRUTE_FORCE_SUCCESS',
                'ip': ip,
                'failures': len(fail_users),
                'success_users': successes[ip],
            })

    return {'failures': dict(failures), 'successes': dict(successes), 'alerts': alerts}


sample_auth_logs = [
    '2024-01-15T10:30:01 Failed password for alice from 192.168.1.50 port 22',
    '2024-01-15T10:30:02 Failed password for alice from 192.168.1.50 port 22',
    '2024-01-15T10:30:03 Failed password for root from 192.168.1.50 port 22',
    '2024-01-15T10:30:04 Failed password for admin from 192.168.1.50 port 22',
    '2024-01-15T10:30:05 Failed password for test from 192.168.1.50 port 22',
    '2024-01-15T10:30:06 Failed password for alice from 192.168.1.50 port 22',
    '2024-01-15T10:30:10 Accepted password for alice from 192.168.1.50 port 22',
    '2024-01-15T10:31:00 Accepted password for bob from 10.0.0.5 port 22',
]

print("\n   Auth event correlation:")
result = analyze_auth_events(sample_auth_logs)
for alert in result['alerts']:
    print(f"\n   ALERT [{alert['type']}] from {alert['ip']}")
    for k, v in alert.items():
        if k != 'type' and k != 'ip':
            print(f"     {k}: {v}")
```

---

## 99.3 IOC (Indicator of Compromise) Extraction

```python
import re
from typing import Dict, List, Set

print("\nIOC Extraction from Logs:")
print("=" * 60)

# IOC patterns for threat intelligence
IOC_IPV4   = re.compile(r'\b(?:(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\.){3}(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\b')
IOC_DOMAIN = re.compile(r'\b(?:[a-zA-Z0-9\-]+\.)+(?:com|net|org|io|gov|edu|mil|co|ru|cn|de|uk|fr|br|au|jp|info|biz|xyz|top|club|online|site)\b')
IOC_MD5    = re.compile(r'\b[a-fA-F0-9]{32}\b')
IOC_SHA1   = re.compile(r'\b[a-fA-F0-9]{40}\b')
IOC_SHA256 = re.compile(r'\b[a-fA-F0-9]{64}\b')
IOC_URL    = re.compile(r'https?://[^\s<>"\']+')
IOC_EMAIL  = re.compile(r'\b[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}\b')
IOC_CVE    = re.compile(r'\bCVE-\d{4}-\d{4,7}\b', re.IGNORECASE)

# Private/reserved IP ranges
PRIVATE_IP = re.compile(
    r'^(?:10\.\d+\.\d+\.\d+|'
    r'172\.(?:1[6-9]|2\d|3[01])\.\d+\.\d+|'
    r'192\.168\.\d+\.\d+|'
    r'127\.\d+\.\d+\.\d+|'
    r'169\.254\.\d+\.\d+)'
)


def extract_iocs(text: str) -> Dict[str, Set[str]]:
    """Extract all IOC types from text, excluding private IPs."""
    iocs: Dict[str, Set[str]] = {
        'ip_external': set(), 'ip_internal': set(),
        'domain': set(), 'url': set(), 'email': set(),
        'md5': set(), 'sha1': set(), 'sha256': set(), 'cve': set(),
    }

    for ip in IOC_IPV4.findall(text):
        if PRIVATE_IP.match(ip):
            iocs['ip_internal'].add(ip)
        else:
            iocs['ip_external'].add(ip)

    # SHA256 before SHA1 before MD5 to avoid subset matches
    remaining = text
    for sha256 in IOC_SHA256.findall(remaining):
        iocs['sha256'].add(sha256)
        remaining = remaining.replace(sha256, ' ' * len(sha256))
    for sha1 in IOC_SHA1.findall(remaining):
        iocs['sha1'].add(sha1)
        remaining = remaining.replace(sha1, ' ' * len(sha1))
    for md5 in IOC_MD5.findall(remaining):
        iocs['md5'].add(md5)

    for domain in IOC_DOMAIN.findall(text):
        iocs['domain'].add(domain.lower())
    for url in IOC_URL.findall(text):
        iocs['url'].add(url)
    for email in IOC_EMAIL.findall(text):
        iocs['email'].add(email.lower())
    for cve in IOC_CVE.findall(text):
        iocs['cve'].add(cve.upper())

    return {k: v for k, v in iocs.items() if v}


incident_report = """
Incident: Malware infection on 192.168.1.105
External C2 server contacted: 203.0.113.42, 198.51.100.7
Malware hash (SHA256): a3f1e2c4b5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5f6a7b8c9d0e1f2
Dropper hash (MD5): d41d8cd98f00b204e9800998ecf8427e
C2 domain: evil-c2.xyz, malware-host.top
Phishing email from: attacker@phish.ru
Related CVE: CVE-2024-12345, CVE-2023-98765
Download URL: http://203.0.113.42/payload.exe
Internal pivot: 10.0.0.50, 172.16.0.100, 192.168.2.55
"""

print("\n   IOC extraction from incident report:")
iocs = extract_iocs(incident_report)
for ioc_type, values in sorted(iocs.items()):
    print(f"\n   [{ioc_type.upper()}] ({len(values)} items):")
    for val in sorted(values):
        print(f"     {val}")
```

---

## 99.4 สรุป Part 99

```
Regex for Log Forensics & SIEM:

1. Multi-format log parsing:
   Syslog RFC 5424: <priority>version timestamp hostname appname procid msgid structured_data message
   Syslog RFC 3164: <priority>timestamp hostname process[pid]: message
   CEF: CEF:version|vendor|product|ver|sig|name|severity|key=value...
   JSON logs: use regex for field extraction fallback, prefer json.loads()

2. Priority decoding:
   priority = facility * 8 + severity
   facility: 0=kernel, 1=user, 3=system daemon, 4=auth, 16-23=local0-7
   severity: 0=Emergency, 1=Alert, 2=Critical, 3=Error, 4=Warning

3. Auth event correlation:
   Brute force: N failures from same IP within time window
   Credential stuffing: many different usernames from one IP
   Success after failures: brute force win indicator
   Impossible travel: same user from geographically distant IPs

4. IOC extraction:
   IP: distinguish public vs private/RFC1918 ranges
   Hash priority: SHA256 (64 hex) > SHA1 (40) > MD5 (32)
     — match longer first to avoid SHA256 matching as MD5
   Domain: extract TLD-qualified domains, not all hostnames
   CVE: CVE-YYYY-NNNNN format, case-insensitive

5. SIEM integration:
   Parse → normalize → correlate → alert
   Enrich IPs with geo/ASN lookups after extraction
   Store structured events; regex is the extraction layer only
   Time normalization: all timestamps to UTC ISO 8601
```

---

*[← Part 98: Regex for Security & WAF Bypass Detection](part-98-security-waf.md) | [→ Part 100: Regex Mastery — Capstone Project](part-100-capstone.md)*
