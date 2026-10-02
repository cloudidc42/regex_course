# Part 100: Regex Mastery — Capstone Project

> **ระดับ:** ผู้เชี่ยวชาญ-ระดับโลก | **เวลาเรียน:** ~150 นาที | **ข้อกำหนด:** Part 01-99
> **เป้าหมาย:** ประยุกต์ทุกทักษะจากหลักสูตรในโปรเจกต์เดียวที่ใช้งานได้จริง

---

## 100.1 Capstone: Security Log Analysis Engine

```python
"""
Regex Mastery Capstone: Security Log Analysis Engine
=====================================================
โปรเจกต์นี้รวมทักษะ regex ทั้งหมด:
  - Pattern compilation + named groups (Part 90)
  - Multi-format log parsing (Part 91, 99)
  - NLP preprocessing (Part 92)
  - ReDoS-safe patterns (Part 89)
  - Performance optimization (Part 95)
  - Network protocol analysis (Part 96)
  - Security detection patterns (Part 98)
  - IOC extraction (Part 99)
"""
import re
import html
import time
import urllib.parse
from typing import Dict, List, Optional, Set, Tuple
from dataclasses import dataclass, field
from collections import defaultdict, Counter

print("Regex Mastery Capstone: Security Log Analysis Engine")
print("=" * 60)


# ─────────────────────────────────────────────────────────────
# 1. Compiled pattern registry (Part 90, 95)
# ─────────────────────────────────────────────────────────────

class PatternRegistry:
    """Central registry for all compiled regex patterns."""

    # Network
    IPV4 = re.compile(r'\b(?:(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\.){3}(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\b')
    PRIVATE_IP = re.compile(
        r'^(?:10\.\d+\.\d+\.\d+|172\.(?:1[6-9]|2\d|3[01])\.\d+\.\d+|'
        r'192\.168\.\d+\.\d+|127\.\d+\.\d+\.\d+|0\.0\.0\.0)$'
    )
    PORT = re.compile(r'\b(?:dpt|dst_port|port)[=:\s]+(?P<port>\d{1,5})\b', re.IGNORECASE)

    # Timestamps
    TS_ISO    = re.compile(r'\b(?P<ts>\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}(?:\.\d+)?(?:Z|[+-]\d{2}:\d{2})?)\b')
    TS_SYSLOG = re.compile(r'\b(?P<ts>(?:Jan|Feb|Mar|Apr|May|Jun|Jul|Aug|Sep|Oct|Nov|Dec)\s+\d{1,2}\s+\d{2}:\d{2}:\d{2})\b')

    # Auth
    AUTH_USER = re.compile(r'\b(?:user|for|username)[=:\s]+(?P<user>[a-zA-Z0-9._\-]{1,64})\b', re.IGNORECASE)
    AUTH_FAIL = re.compile(r'\b(?:failed|invalid|error|denied)\b.*(?:password|login|auth)', re.IGNORECASE)
    AUTH_OK   = re.compile(r'\b(?:accepted|successful|success)\b.*(?:password|login|auth)', re.IGNORECASE)

    # HTTP
    HTTP_METHOD = re.compile(r'\b(?P<method>GET|POST|PUT|DELETE|PATCH|HEAD|OPTIONS)\s+(?P<path>[^\s"]+)\s+HTTP/(?P<ver>\d\.\d)\b')
    HTTP_STATUS = re.compile(r'\b(?P<status>[1-5]\d{2})\b')

    # Security IOCs
    MD5    = re.compile(r'\b[a-fA-F0-9]{32}\b')
    SHA256 = re.compile(r'\b[a-fA-F0-9]{64}\b')
    CVE    = re.compile(r'\bCVE-\d{4}-\d{4,7}\b', re.IGNORECASE)

    # Attack signatures (ReDoS-safe — no nested quantifiers)
    SQLI    = re.compile(r"(?:'\s*(?:OR|AND)\s+'?\d|UNION\s+(?:ALL\s+)?SELECT|\bSLEEP\s*\()", re.IGNORECASE)
    XSS     = re.compile(r'(?:<\s*script\b|\bon\w+\s*=|javascript\s*:)', re.IGNORECASE)
    PATH_TR = re.compile(r'(?:\.\.[\/\\]|%2e%2e|%252e)', re.IGNORECASE)
    CMD_INJ = re.compile(r'(?:[;&|`]\s*(?:cat|ls|id|whoami|nc|bash|sh|wget|curl)\b|\$\()', re.IGNORECASE)

P = PatternRegistry()
```

---

## 100.2 Log Event Parser

```python
@dataclass
class LogEvent:
    raw: str
    timestamp: Optional[str] = None
    source_ip: Optional[str] = None
    dest_ip: Optional[str] = None
    port: Optional[int] = None
    user: Optional[str] = None
    method: Optional[str] = None
    path: Optional[str] = None
    status: Optional[int] = None
    event_type: str = 'UNKNOWN'
    attack_types: List[str] = field(default_factory=list)
    iocs: Dict[str, Set[str]] = field(default_factory=lambda: defaultdict(set))
    severity: str = 'INFO'


def normalize_input(text: str) -> str:
    """Defensive normalization: decode encodings before pattern matching."""
    text = text.replace('\x00', '')
    prev = None
    for _ in range(3):
        if text == prev:
            break
        prev = text
        text = urllib.parse.unquote(text)
    text = html.unescape(text)
    text = re.sub(r'/\*.*?\*/', ' ', text, flags=re.DOTALL)
    text = re.sub(r'[\s\x0b\x0c]+', ' ', text).strip()
    return text


def parse_log_event(line: str) -> LogEvent:
    event = LogEvent(raw=line)

    m = P.TS_ISO.search(line)
    if not m:
        m = P.TS_SYSLOG.search(line)
    if m:
        event.timestamp = m.group('ts')

    ips = P.IPV4.findall(line)
    if ips:
        event.source_ip = ips[0]
        if len(ips) > 1:
            event.dest_ip = ips[1]

    m = P.PORT.search(line)
    if m:
        port = int(m.group('port'))
        if 1 <= port <= 65535:
            event.port = port

    m = P.AUTH_USER.search(line)
    if m:
        event.user = m.group('user')

    m = P.HTTP_METHOD.search(line)
    if m:
        event.method = m.group('method')
        event.path   = m.group('path')
        tail = line[m.end():m.end()+20]
        sm = P.HTTP_STATUS.search(tail)
        if sm:
            event.status = int(sm.group('status'))

    if P.AUTH_FAIL.search(line):
        event.event_type = 'AUTH_FAILURE'
        event.severity = 'WARNING'
    elif P.AUTH_OK.search(line):
        event.event_type = 'AUTH_SUCCESS'

    normalized = normalize_input(line)
    attacks = []
    if P.SQLI.search(normalized):    attacks.append('SQLI')
    if P.XSS.search(normalized):     attacks.append('XSS')
    if P.PATH_TR.search(normalized): attacks.append('PATH_TRAVERSAL')
    if P.CMD_INJ.search(normalized): attacks.append('CMD_INJECTION')

    if attacks:
        event.attack_types = attacks
        event.severity = 'CRITICAL'
        event.event_type = 'ATTACK'

    for sha256 in P.SHA256.findall(line):
        event.iocs['sha256'].add(sha256)
    for md5 in P.MD5.findall(line):
        event.iocs['md5'].add(md5)
    for cve in P.CVE.findall(line):
        event.iocs['cve'].add(cve.upper())

    return event


print("\n2. Log event parsing with attack detection:")

sample_logs = [
    "2024-01-15T10:30:01Z web01 sshd: Failed password for root from 203.0.113.42 port 22",
    "2024-01-15T10:30:05Z web01 sshd: Accepted password for alice from 10.0.0.5 port 22",
    "2024-01-15T10:31:00Z nginx: 203.0.113.10 - GET /login?user=admin'%20OR%20'1'%3D'1 HTTP/1.1 200",
    "2024-01-15T10:31:05Z nginx: 203.0.113.10 - GET /index.php?file=../../etc/passwd HTTP/1.1 500",
    "2024-01-15T10:31:10Z nginx: 10.0.0.5 - GET /api/v1/users HTTP/1.1 200",
    "2024-01-15T10:32:00Z sysmon: Process created: hash=a3f1e2c4b5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5f6a7b8c9d0e1f2 CVE-2024-12345",
]

for line in sample_logs:
    ev = parse_log_event(line)
    sev_icon = {'CRITICAL': 'XX', 'WARNING': '!!', 'INFO': '  '}.get(ev.severity, '  ')
    print(f"\n   [{sev_icon}] {ev.event_type:<16} {ev.severity:<8} ts={ev.timestamp}")
    if ev.source_ip:
        is_priv = bool(P.PRIVATE_IP.match(ev.source_ip))
        print(f"     src={ev.source_ip} ({'private' if is_priv else 'external'})", end='')
        if ev.user:
            print(f", user={ev.user!r}", end='')
        print()
    if ev.attack_types:
        print(f"     ATTACKS: {', '.join(ev.attack_types)}")
    if ev.iocs:
        for ioc_type, vals in ev.iocs.items():
            print(f"     IOC [{ioc_type}]: {', '.join(list(vals)[:1])[:60]}")
```

---

## 100.3 Alert Engine & Statistics

```python
from collections import Counter

print("\n3. Alert engine — brute-force & attack source detection:")


def run_analysis(log_lines: List[str]) -> Dict:
    events = [parse_log_event(line) for line in log_lines]
    stats = {
        'total': len(events),
        'by_severity': Counter(e.severity for e in events),
        'by_type': Counter(e.event_type for e in events),
        'attack_counts': Counter(a for e in events for a in e.attack_types),
        'auth_failures': defaultdict(list),
        'alerts': [],
    }

    for ev in events:
        if ev.event_type == 'AUTH_FAILURE' and ev.source_ip:
            stats['auth_failures'][ev.source_ip].append(ev.user or 'unknown')

    for ip, users in stats['auth_failures'].items():
        if len(users) >= 3:
            stats['alerts'].append({
                'type': 'BRUTE_FORCE',
                'ip': ip,
                'count': len(users),
                'users': list(set(users)),
            })

    attack_ips = Counter(
        ev.source_ip for ev in events
        if ev.attack_types and ev.source_ip and not P.PRIVATE_IP.match(ev.source_ip)
    )
    for ip, count in attack_ips.most_common(3):
        stats['alerts'].append({'type': 'ATTACK_SOURCE', 'ip': ip, 'attack_count': count})

    return stats


expanded_logs = sample_logs + [
    "2024-01-15T10:30:02Z web01 sshd: Failed password for alice from 203.0.113.42 port 22",
    "2024-01-15T10:30:03Z web01 sshd: Failed password for admin from 203.0.113.42 port 22",
    "2024-01-15T10:30:04Z web01 sshd: Failed password for test from 203.0.113.42 port 22",
    "2024-01-15T10:31:02Z nginx: 203.0.113.10 - POST /search?q=<script>alert(1)</script> HTTP/1.1 400",
]

result = run_analysis(expanded_logs)
print(f"\n   Analysis results ({result['total']} events):")
print(f"   By severity: {dict(result['by_severity'])}")
print(f"   By type:     {dict(result['by_type'])}")
print(f"   Attacks:     {dict(result['attack_counts'])}")

if result['alerts']:
    print(f"\n   ALERTS ({len(result['alerts'])}):")
    for alert in result['alerts']:
        print(f"   [{alert['type']}] IP={alert['ip']}", end='')
        if 'count' in alert:
            print(f", failures={alert['count']}, users={alert.get('users', [])[:3]}", end='')
        if 'attack_count' in alert:
            print(f", attacks={alert['attack_count']}", end='')
        print()
```

---

## 100.4 สรุปหลักสูตร — What You've Learned

```
╔══════════════════════════════════════════════════════════════╗
║          REGEX MASTERY COURSE — COMPLETE                     ║
║          Parts 1-100 | Professional / World-Class Level      ║
╚══════════════════════════════════════════════════════════════╝

FOUNDATION (Parts 1-30)
  re.match / re.search / re.findall / re.finditer / re.sub
  Character classes: [abc], [^abc], \d \w \s and negations
  Quantifiers: * + ? {n,m} and greedy vs lazy (??)
  Anchors: ^ $ \b \A \Z
  Groups: () capturing, (?:...) non-capturing, (?P<name>...)

INTERMEDIATE (Parts 31-60)
  Lookahead (?=...) / (?!...) / lookbehind (?<=...) / (?<!...)
  Backreferences: \1 or (?P=name) for matched group reuse
  Inline flags: (?i) (?m) (?s) (?x) for verbose patterns
  Alternation and priority: (a|b|c) — first match wins
  re.MULTILINE: ^ and $ match each line, not just string ends

ADVANCED (Parts 61-88)
  Atomic groups and possessive quantifiers (via workarounds)
  Custom tokenizers with master alternation + m.lastgroup
  Large-scale text processing with generators
  Unicode and locale-aware matching: re.UNICODE
  Combining Python logic with regex for hybrid validation

PROFESSIONAL (Parts 89-100)
  Part 89:  ReDoS prevention — catastrophic backtracking, safe replacements
  Part 90:  Advanced re module — named groups, verbose, tokenizers
  Part 91:  Real-world log/config parsing
  Part 92:  NLP preprocessing, rule-based NER, PII masking
  Part 93:  Regex testing & QA — unittest, property-based, fuzz
  Part 94:  DevOps — Conventional Commits, Docker, Terraform, Prometheus
  Part 95:  Performance — compile, anchor, char class vs alternation
  Part 96:  Network protocols — IP, MAC, HTTP, DNS, URL security
  Part 97:  Databases — SQLite REGEXP, schema validation, query logs
  Part 98:  Security — normalization, XSS/SQLi/path traversal/secrets
  Part 99:  SIEM/forensics — syslog, CEF, IOC extraction, correlation
  Part 100: Capstone — full security log analysis engine

THE 10 GOLDEN RULES OF PRODUCTION REGEX:
  1. Compile once at module level: PATTERN = re.compile(...)
  2. Normalize before matching (URL decode, HTML entities, null bytes)
  3. Use named groups: (?P<name>...) for maintainable extraction
  4. Anchor when you can: ^ $ \b prevent partial matches
  5. Prefer [abc] over (a|b|c) for single-character classes
  6. Avoid nested quantifiers: (a+)+ is ReDoS waiting to happen
  7. Use non-capturing (?:...) when you don't need the group value
  8. Validate ranges in Python, not regex: 0 <= int(m.group()) <= 255
  9. Write tests: VALID, INVALID, and GROUPS test cases minimum
  10. Defense in depth: regex is ONE layer, not the only defense

จบหลักสูตร Regex ระดับมืออาชีพ — Professional / World-Class Level
```

---

*[← Part 99: Regex for Log Forensics & SIEM](part-99-log-forensics.md)*

---

> **หลักสูตรนี้ครอบคลุม 100 Parts** | Regex fundamentals → advanced patterns → real-world applications → security & forensics
> เนื้อหาทั้งหมดเป็น Python ที่ runnable 100% พร้อม output จริง
