# Part 80: Log Analysis & SIEM Patterns

> **ระดับ:** ผู้เชี่ยวชาญ-ระดับโลก | **เวลาเรียน:** ~100 นาที | **ข้อกำหนด:** Part 01-79

---

## 80.1 Syslog & Common Log Format Parsing

```python
import re
from typing import Dict, List, Optional

print("Log Analysis & SIEM Patterns:")
print("=" * 60)

print("\n1. Standard log format parsers:")

# RFC 5424 Syslog
SYSLOG_RFC5424 = re.compile(
    r'^<(?P<pri>\d+)>(?P<version>\d+)\s+'
    r'(?P<timestamp>\S+)\s+'
    r'(?P<hostname>\S+)\s+'
    r'(?P<appname>\S+)\s+'
    r'(?P<procid>\S+)\s+'
    r'(?P<msgid>\S+)\s+'
    r'(?P<structured>\S+|-)\s+'
    r'(?P<msg>.*)',
    re.MULTILINE
)

# BSD Syslog (RFC 3164)
SYSLOG_RFC3164 = re.compile(
    r'^<(?P<pri>\d+)>'
    r'(?P<month>[A-Z][a-z]{2})\s+'
    r'(?P<day>\s?\d+)\s+'
    r'(?P<time>\d{2}:\d{2}:\d{2})\s+'
    r'(?P<hostname>\S+)\s+'
    r'(?P<process>[^:\[]+)(?:\[(?P<pid>\d+)\])?\s*:\s+'
    r'(?P<msg>.*)',
    re.MULTILINE
)

# Apache Combined Log Format
APACHE_COMBINED = re.compile(
    r'^(?P<ip>[\d.]+|::1)\s+'
    r'(?P<ident>\S+)\s+'
    r'(?P<user>\S+)\s+'
    r'\[(?P<time>[^\]]+)\]\s+'
    r'"(?P<method>\w+)\s+(?P<path>\S+)\s+HTTP/(?P<http_ver>[\d.]+)"\s+'
    r'(?P<status>\d+)\s+'
    r'(?P<bytes>\d+|-)\s+'
    r'"(?P<referer>[^"]*)"\s+'
    r'"(?P<ua>[^"]*)"'
)

# Nginx error log
NGINX_ERROR = re.compile(
    r'^(?P<time>\d{4}/\d{2}/\d{2}\s+\d{2}:\d{2}:\d{2})\s+'
    r'\[(?P<level>debug|info|notice|warn|error|crit|alert|emerg)\]\s+'
    r'(?P<pid>\d+)#(?P<tid>\d+):\s+'
    r'(?:\*(?P<cid>\d+)\s+)?'
    r'(?P<msg>.*)'
)


def parse_syslog_pri(pri: int):
    facility  = pri >> 3
    severity  = pri & 7
    facilities = {0:'kern', 1:'user', 2:'mail', 3:'daemon', 4:'auth',
                  5:'syslog', 6:'lpr', 7:'news', 10:'authpriv', 16:'local0'}
    severities = {0:'emerg', 1:'alert', 2:'crit', 3:'err',
                  4:'warning', 5:'notice', 6:'info', 7:'debug'}
    return facilities.get(facility, f'fac{facility}'), severities.get(severity, f'sev{severity}')


sample_logs = [
    '<34>1 2024-02-15T10:00:01.000Z web-server nginx 1234 - - Connection from 192.168.1.1',
    '<165>Feb 15 10:00:02 firewall sshd[5678]: Failed password for root from 10.0.0.1 port 22 ssh2',
    '10.0.0.50 - - [15/Feb/2024:10:00:03 +0700] "GET /admin.php HTTP/1.1" 404 0 "-" "sqlmap/1.7.8"',
    '2024/02/15 10:00:04 [error] 1234#0: *42 connect() failed (111: Connection refused)',
]

print(f"\n   Log format detection and parsing:")
for log in sample_logs:
    m5424  = SYSLOG_RFC5424.match(log)
    m3164  = SYSLOG_RFC3164.match(log)
    mhttp  = APACHE_COMBINED.match(log)
    mnginx = NGINX_ERROR.match(log)

    if m5424:
        pri = int(m5424.group('pri'))
        fac, sev = parse_syslog_pri(pri)
        print(f"   [RFC5424] {m5424.group('hostname')} | {fac}.{sev} | {m5424.group('msg')[:50]}")
    elif m3164:
        pri = int(m3164.group('pri'))
        fac, sev = parse_syslog_pri(pri)
        print(f"   [RFC3164] {m3164.group('hostname')} | {fac}.{sev} | {m3164.group('msg')[:50]}")
    elif mhttp:
        print(f"   [Apache] {mhttp.group('ip')} | {mhttp.group('method')} {mhttp.group('path')[:40]} | {mhttp.group('status')}")
    elif mnginx:
        print(f"   [Nginx] [{mnginx.group('level')}] {mnginx.group('msg')[:60]}")
    else:
        print(f"   [?] {log[:60]!r}")
```

---

## 80.2 Security Event Correlation

```python
import re
from collections import defaultdict
from typing import Dict, List

print("\nSecurity Event Correlation:")
print("=" * 60)

AUTH_FAILURE = re.compile(
    r'(?:failed\s+(?:password|login|authentication)|'
    r'authentication\s+failure|'
    r'invalid\s+(?:password|user|credentials)|'
    r'logon\s+failure)',
    re.IGNORECASE
)

AUTH_SUCCESS = re.compile(
    r'(?:accepted\s+password|'
    r'session\s+opened|'
    r'successful\s+(?:login|authentication)|'
    r'logon\s+success)',
    re.IGNORECASE
)

EXTRACT_IP = re.compile(r'\b(?:from|src|source|ip[=:]?)\s*([\d.]+)', re.IGNORECASE)

PRIV_ESCALATION = re.compile(
    r'(?:sudo\s+|su\s+|runas\s+|privilege\s+escalat|'
    r'token\s+impersonation|seimpers|'
    r'getsystem|bypassuac)',
    re.IGNORECASE
)

LATERAL_MOVEMENT = re.compile(
    r'(?:psexec|wmiexec|smbexec|dcomexec|'
    r'remote\s+service\s+install|'
    r'net\s+use\s+\\\\|'
    r'invoke-command\s+-computer)',
    re.IGNORECASE
)


class EventCorrelator:
    def __init__(self):
        self.failures = defaultdict(list)
        self.alerts   = []

    def process_event(self, log_line: str, timestamp: float = 0.0):
        ip_m = EXTRACT_IP.search(log_line)
        ip = ip_m.group(1) if ip_m else 'unknown'

        if AUTH_FAILURE.search(log_line):
            self.failures[ip].append(timestamp)
            recent = [t for t in self.failures[ip] if timestamp - t < 60]
            self.failures[ip] = recent
            if len(recent) >= 5:
                self.alerts.append({
                    'type':  'BRUTE_FORCE',
                    'ip':    ip,
                    'count': len(recent),
                    'log':   log_line[:80],
                })

        if AUTH_SUCCESS.search(log_line) and len(self.failures.get(ip, [])) >= 3:
            self.alerts.append({
                'type':           'BRUTE_FORCE_SUCCESS',
                'ip':             ip,
                'prior_failures': len(self.failures[ip]),
                'log':            log_line[:80],
            })

        if PRIV_ESCALATION.search(log_line):
            self.alerts.append({'type': 'PRIV_ESC', 'ip': ip, 'log': log_line[:80]})

        if LATERAL_MOVEMENT.search(log_line):
            self.alerts.append({'type': 'LATERAL_MOVE', 'ip': ip, 'log': log_line[:80]})


corr = EventCorrelator()

security_events = [
    (1, "sshd: Failed password for root from 10.0.0.5 port 22"),
    (2, "sshd: Failed password for root from 10.0.0.5 port 22"),
    (3, "sshd: Failed password for admin from 10.0.0.5 port 22"),
    (4, "sshd: Failed password for admin from 10.0.0.5 port 22"),
    (5, "sshd: Failed password for user from 10.0.0.5 port 22"),
    (6, "sshd: Accepted password for admin from 10.0.0.5 port 22"),
    (7, "sudo: user admin executed: /bin/bash"),
    (8, "Windows: Logon failure for DOMAIN\\user from 192.168.1.20"),
    (9, "Windows: Logon failure for DOMAIN\\user from 192.168.1.20"),
    (10, "net use \\\\server\\share from 192.168.1.20"),
    (11, "Normal: Web request GET /index.html from 172.16.0.1"),
]

print(f"\n   Event correlation results:")
for ts, event in security_events:
    corr.process_event(event, float(ts))

for alert in corr.alerts:
    print(f"\n   \U0001f6a8 ALERT: {alert['type']}")
    for k, v in alert.items():
        if k != 'type':
            print(f"     {k}: {v}")
```

---

## 80.3 HTTP Anomaly Scoring

```python
import re
from typing import Dict

print("\nHTTP Anomaly Scoring:")
print("=" * 60)

TIMESTAMP_LOG  = re.compile(r'\[(\d{2})/\w+/\d{4}:(\d{2}):(\d{2}):(\d{2})')
BUSINESS_HOURS = range(8, 18)

def is_off_hours(log_line: str) -> bool:
    m = TIMESTAMP_LOG.search(log_line)
    if m:
        return int(m.group(2)) not in BUSINESS_HOURS
    return False


SUSPICIOUS_UA = re.compile(
    r'^(?:Python-urllib|python-requests|Go-http-client|curl|wget|'
    r'Scrapy|axios|node-fetch|Java/)',
    re.IGNORECASE
)

MISSING_UA = re.compile(r'"-"\s*$|""\s*$')

UNUSUAL_METHOD = re.compile(
    r'"(?:TRACE|TRACK|DEBUG|CONNECT|PROPFIND|PROPPATCH|MKCOL)\s'
)


def score_http_anomaly(log_line: str) -> Dict:
    score = 0
    reasons = []

    if is_off_hours(log_line):
        score += 1
        reasons.append('off-hours access')

    # Extract UA from log
    ua_m = re.search(r'"([^"]+)"\s*$', log_line)
    if ua_m:
        ua = ua_m.group(1)
        if SUSPICIOUS_UA.search(ua):
            score += 2
            reasons.append(f'suspicious UA: {ua[:40]}')
        if ua in ('-', ''):
            score += 2
            reasons.append('missing user-agent')

    if UNUSUAL_METHOD.search(log_line):
        score += 3
        reasons.append('unusual HTTP method')

    return {'score': score, 'reasons': reasons}


test_logs = [
    '10.0.0.1 - - [15/Feb/2024:02:30:00 +0700] "GET /admin HTTP/1.1" 200 1024 "-" "python-requests/2.31.0"',
    '10.0.0.2 - - [15/Feb/2024:14:00:00 +0700] "GET /index.html HTTP/1.1" 200 512 "-" "Mozilla/5.0"',
    '10.0.0.3 - - [15/Feb/2024:23:59:00 +0700] "TRACE / HTTP/1.1" 200 0 "-" "-"',
    '10.0.0.4 - - [15/Feb/2024:09:00:00 +0700] "GET /api/users?id=1 HTTP/1.1" 200 256 "-" "curl/7.68.0"',
    '10.0.0.5 - - [15/Feb/2024:11:00:00 +0700] "GET /page HTTP/1.1" 200 400 "-" "Mozilla/5.0 Chrome/120"',
]

print(f"\n   HTTP anomaly scoring:")
for log in test_logs:
    result = score_http_anomaly(log)
    ip_m = re.match(r'([\d.]+)', log)
    ip = ip_m.group(1) if ip_m else '?'
    if result['score'] > 0:
        print(f"\n   ⚠ Score={result['score']} | {ip}")
        for r in result['reasons']:
            print(f"     → {r}")
    else:
        print(f"   ✓ Score=0 | {ip}")
```

---

## 80.4 Sigma Rule Parsing

```python
import re
from typing import Dict, List

print("\nSigma Rule Pattern Detection:")
print("=" * 60)

SIGMA_TITLE     = re.compile(r'^title:\s*(.+)', re.MULTILINE)
SIGMA_CONDITION = re.compile(r'^\s+condition:\s*(.+)', re.MULTILINE)

WIN_EVENT_IDS = {
    '4625': 'Account Logon Failure',
    '4648': 'Logon With Explicit Credentials',
    '4672': 'Special Privileges Assigned',
    '4688': 'Process Creation',
    '4698': 'Scheduled Task Created',
    '4720': 'User Account Created',
    '4732': 'Member Added to Security Group',
    '7045': 'Service Installed',
    '1102': 'Audit Log Cleared',
}

WIN_EVENTID_PATTERN = re.compile(r'EventID(?:\|contains)?:\s*(?:\'|")?( \d+)(?:\'|")?')


def parse_sigma_rule(sigma_yaml: str) -> Dict:
    title_m = SIGMA_TITLE.search(sigma_yaml)
    cond_m  = SIGMA_CONDITION.search(sigma_yaml)
    evtids  = re.findall(r"EventID[^:]*:\s*'?(\d+)", sigma_yaml)

    return {
        'title':     title_m.group(1).strip() if title_m else 'Unknown',
        'condition': cond_m.group(1).strip() if cond_m else 'Unknown',
        'event_ids': [(eid, WIN_EVENT_IDS.get(eid, 'Unknown')) for eid in evtids],
    }


sample_sigma_rules = [
    """
title: Windows Account Brute Force
logsource:
  product: windows
  service: security
detection:
  selection:
    EventID: '4625'
    LogonType: 3
  condition: selection | count() by TargetUserName > 10
""",
    """
title: Suspicious PowerShell Execution
logsource:
  product: windows
  category: process_creation
detection:
  selection:
    EventID: '4688'
    CommandLine|contains:
      - '-EncodedCommand'
      - '-enc '
      - 'IEX('
  condition: selection
""",
    """
title: New Local Admin Account
logsource:
  product: windows
  service: security
detection:
  account_created:
    EventID: '4720'
  added_to_group:
    EventID: '4732'
    GroupName: 'Administrators'
  condition: account_created and added_to_group
""",
]

print(f"\n   Sigma rule analysis:")
for rule in sample_sigma_rules:
    parsed = parse_sigma_rule(rule)
    print(f"\n   Rule: {parsed['title']}")
    print(f"   Condition: {parsed['condition']}")
    if parsed['event_ids']:
        for eid, ename in parsed['event_ids']:
            print(f"   EventID {eid}: {ename}")
```

---

## 80.5 สรุป Part 80

```
Log Analysis & SIEM Pattern Design:

1. Log format parsers:
   RFC 5424: <PRI>VERSION TIMESTAMP HOSTNAME APP PROCID MSGID SD MSG
   RFC 3164: <PRI>Mon DD HH:MM:SS hostname process[pid]: msg
   Apache:   IP - user [datetime] "METHOD /path HTTP/ver" status bytes "referer" "UA"
   Nginx:    YYYY/MM/DD HH:MM:SS [level] PID#TID: *CID msg

2. Syslog priority decoding:
   PRI = facility * 8 + severity
   Facilities: kern(0), user(1), mail(2), daemon(3), auth(4)
   Severities: emerg(0), alert(1), crit(2), err(3), warn(4), notice(5), info(6), debug(7)

3. Event correlation rules:
   Brute force: > 5 auth failures from same IP in 60 seconds
   BF success: auth failure spike followed by success
   Privilege escalation: sudo, su, token impersonation
   Lateral movement: psexec, wmiexec, net use \\\\server

4. Anomaly scoring:
   Off-hours access: +1 (before 08:00 or after 18:00)
   Suspicious UA: +2 (python-requests, curl, wget)
   Missing UA: +2 (empty/dash User-Agent)
   Unusual method: +3 (TRACE, DEBUG, PROPFIND)
   Block threshold: total score > 5

5. Sigma rule language:
   logsource: product + category/service
   detection: named selections + condition expression
   condition: AND/OR/NOT/count() logic over selections
   MITRE ATT&CK: technique tags for alert classification
```

---

*[← Part 79: Threat Intelligence](part-79-threat-intel.md) | [→ Part 81: Cloud Security Patterns](part-81-cloud-security.md)*
