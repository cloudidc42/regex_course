# Part 52: Regex for Log Analysis & Monitoring

> **ระดับ:** กลาง-สูง | **เวลาเรียน:** ~70 นาที | **ข้อกำหนด:** Part 01-51

---

## 52.1 Log Format Parsing

```python
import re
from typing import Dict, List, Optional
from datetime import datetime
from collections import Counter, defaultdict

print("Log Analysis with Regex:")
print("=" * 60)

# ===== Common log formats =====
print("\n1. Log format patterns:")

# Common Log Format (CLF): Apache/Nginx
CLF = re.compile(
    r'(?P<host>\S+)\s+'
    r'(?P<ident>\S+)\s+'
    r'(?P<user>\S+)\s+'
    r'\[(?P<time>[^\]]+)\]\s+'
    r'"(?P<request>[^"]+)"\s+'
    r'(?P<status>\d{3})\s+'
    r'(?P<size>\S+)'
)

# Extended Log Format (ELF): adds referer and user-agent
ELF = re.compile(
    r'(?P<host>\S+)\s+'
    r'(?P<ident>\S+)\s+'
    r'(?P<user>\S+)\s+'
    r'\[(?P<time>[^\]]+)\]\s+'
    r'"(?P<request>[^"]+)"\s+'
    r'(?P<status>\d{3})\s+'
    r'(?P<size>\S+)\s+'
    r'"(?P<referer>[^"]*)"\s+'
    r'"(?P<useragent>[^"]*)"'
)

# Syslog format
SYSLOG = re.compile(
    r'(?P<month>\w{3})\s+(?P<day>\d{1,2})\s+(?P<time>\d{2}:\d{2}:\d{2})\s+'
    r'(?P<host>\S+)\s+'
    r'(?P<process>[^\[:]+)(?:\[(?P<pid>\d+)\])?:\s+'
    r'(?P<message>.+)'
)

# JSON-structured log
JSON_LOG = re.compile(
    r'\{[^}]*"level"\s*:\s*"(?P<level>[^"]+)"[^}]*'
    r'"message"\s*:\s*"(?P<message>[^"]+)"[^}]*\}'
)

# ===== Parse sample logs =====
apache_logs = [
    '192.168.1.1 - alice [15/Oct/2024:09:30:00 +0700] "GET /api/users HTTP/1.1" 200 1234 "https://example.com" "Mozilla/5.0"',
    '10.0.0.5 - - [15/Oct/2024:09:30:01 +0700] "POST /api/login HTTP/1.1" 401 89 "-" "curl/7.68.0"',
    '172.16.0.3 - bob [15/Oct/2024:09:30:02 +0700] "DELETE /api/users/42 HTTP/1.1" 204 0 "-" "PostmanRuntime/7.29"',
]

print(f"\n   {'Host':<15} {'User':<8} {'Status':<8} {'Size':<8} {'Request'}")
print(f"   {'-'*15} {'-'*8} {'-'*8} {'-'*8} {'-'*35}")
for log in apache_logs:
    m = ELF.match(log)
    if m:
        d = m.groupdict()
        req_parts = d['request'].split(' ')
        path = req_parts[1] if len(req_parts) > 1 else d['request']
        print(f"   {d['host']:<15} {d['user']:<8} {d['status']:<8} {d['size']:<8} {path}")
```

---

## 52.2 Log Analytics Pipeline

```python
import re
from collections import Counter, defaultdict
from typing import Dict, List

print("\nLog Analytics Pipeline:")
print("=" * 60)

class LogAnalyzer:
    """Analyze web server logs with regex"""

    ELF = re.compile(
        r'(?P<host>\S+)\s+\S+\s+(?P<user>\S+)\s+'
        r'\[(?P<time>[^\]]+)\]\s+'
        r'"(?P<method>\w+)\s+(?P<path>[^\s"]+)[^"]*"\s+'
        r'(?P<status>\d{3})\s+(?P<bytes>\S+)'
        r'(?:\s+"(?P<referer>[^"]*)"\s+"(?P<ua>[^"]*)")?' 
    )

    # Classify paths
    STATIC_RE  = re.compile(r'\.(css|js|png|jpg|gif|ico|woff2?|svg)(\?.*)?$', re.IGNORECASE)
    API_RE     = re.compile(r'^/api/')
    ADMIN_RE   = re.compile(r'^/admin/')

    def __init__(self):
        self.records = []
        self.errors  = []

    def parse_line(self, line: str):
        m = self.ELF.match(line)
        if not m:
            return None
        d = m.groupdict()
        d['status_int'] = int(d['status'])
        d['bytes_int']  = int(d['bytes']) if d['bytes'] != '-' else 0
        d['path_type']  = self._classify_path(d['path'])
        return d

    def _classify_path(self, path: str) -> str:
        if self.STATIC_RE.search(path): return 'static'
        if self.API_RE.match(path):     return 'api'
        if self.ADMIN_RE.match(path):   return 'admin'
        return 'page'

    def load(self, lines):
        for line in lines:
            parsed = self.parse_line(line)
            if parsed:
                self.records.append(parsed)
            elif line.strip():
                self.errors.append(line)

    def summary(self):
        if not self.records:
            return {}

        status_counts = Counter(r['status'] for r in self.records)
        method_counts = Counter(r['method'] for r in self.records)
        path_types    = Counter(r['path_type'] for r in self.records)
        top_paths     = Counter(r['path'] for r in self.records).most_common(5)
        top_ips       = Counter(r['host'] for r in self.records).most_common(5)
        error_paths   = [r['path'] for r in self.records if r['status_int'] >= 400]
        total_bytes   = sum(r['bytes_int'] for r in self.records)

        return {
            'total_requests': len(self.records),
            'parse_errors':   len(self.errors),
            'status_codes':   dict(status_counts),
            'http_methods':   dict(method_counts),
            'path_types':     dict(path_types),
            'top_paths':      top_paths,
            'top_ips':        top_ips,
            'error_paths':    error_paths[:5],
            'total_bytes':    total_bytes,
            'error_rate':     len(error_paths) / len(self.records),
        }


# Test data
sample_logs = [
    '192.168.1.1 - alice [15/Oct/2024:09:00:01 +0700] "GET /api/users HTTP/1.1" 200 1234 "-" "Mozilla/5.0"',
    '192.168.1.1 - alice [15/Oct/2024:09:00:02 +0700] "GET /static/style.css HTTP/1.1" 304 0 "-" "Mozilla/5.0"',
    '10.0.0.5 - - [15/Oct/2024:09:00:03 +0700] "POST /api/login HTTP/1.1" 401 89 "-" "curl/7.68"',
    '192.168.1.2 - bob [15/Oct/2024:09:00:04 +0700] "GET /api/users HTTP/1.1" 200 1567 "-" "Firefox/120"',
    '10.0.0.5 - - [15/Oct/2024:09:00:05 +0700] "GET /admin/ HTTP/1.1" 403 256 "-" "curl/7.68"',
    '192.168.1.3 - - [15/Oct/2024:09:00:06 +0700] "GET /api/products HTTP/1.1" 200 4321 "-" "Chrome/119"',
    '192.168.1.1 - alice [15/Oct/2024:09:00:07 +0700] "DELETE /api/users/99 HTTP/1.1" 404 45 "-" "Mozilla/5.0"',
    '10.0.0.5 - - [15/Oct/2024:09:00:08 +0700] "POST /api/login HTTP/1.1" 401 89 "-" "curl/7.68"',
    '192.168.1.4 - carol [15/Oct/2024:09:00:09 +0700] "GET /api/orders HTTP/1.1" 200 2890 "-" "Safari/17"',
    '192.168.1.1 - alice [15/Oct/2024:09:00:10 +0700] "GET /static/app.js HTTP/1.1" 200 45678 "-" "Mozilla/5.0"',
]

analyzer = LogAnalyzer()
analyzer.load(sample_logs)
stats = analyzer.summary()

print(f"\n   Log Summary:")
print(f"   Total requests: {stats['total_requests']}")
print(f"   Parse errors:   {stats['parse_errors']}")
print(f"   Total bytes:    {stats['total_bytes']:,}")
print(f"   Error rate:     {stats['error_rate']:.1%}")
print(f"\n   Status codes:   {stats['status_codes']}")
print(f"   HTTP methods:   {stats['http_methods']}")
print(f"   Path types:     {stats['path_types']}")
print(f"\n   Top paths:")
for path, count in stats['top_paths']:
    print(f"     {count:3}x  {path}")
print(f"\n   Top IPs:")
for ip, count in stats['top_ips']:
    print(f"     {count:3}x  {ip}")
print(f"\n   Error paths: {stats['error_paths']}")
```

---

## 52.3 Anomaly Detection

```python
import re
from collections import Counter, defaultdict
from typing import List, Dict

print("\nAnomaly Detection in Logs:")
print("=" * 60)

class SecurityAnalyzer:
    """Detect suspicious patterns in logs"""

    SQL_INJECT_RE = re.compile(
        r"(?:'|\"|;|--|/\*|\*/|union\s+select|select\s+.*\s+from|"
        r"insert\s+into|drop\s+table|xp_cmdshell|exec\s*\()",
        re.IGNORECASE
    )

    PATH_TRAVERSAL_RE = re.compile(r'(?:\.\./|\.\.\\ |%2e%2e)', re.IGNORECASE)

    SCANNER_UA_RE = re.compile(
        r'(?:nikto|sqlmap|nmap|masscan|zgrab|gobuster|dirbuster|'
        r'wfuzz|burpsuite|owasp|metasploit)',
        re.IGNORECASE
    )

    def __init__(self):
        self.ip_401    = defaultdict(int)
        self.ip_404    = defaultdict(int)
        self.sql_hits  = []
        self.traversal = []
        self.scanners  = []

    def analyze_line(self, line: str, parsed: Dict):
        ip     = parsed.get('host', '')
        path   = parsed.get('path', '')
        status = parsed.get('status_int', 0)
        ua     = parsed.get('ua', '') or ''

        if status == 401:
            self.ip_401[ip] += 1
        if status == 404:
            self.ip_404[ip] += 1
        if self.SQL_INJECT_RE.search(path):
            self.sql_hits.append({'ip': ip, 'path': path})
        if self.PATH_TRAVERSAL_RE.search(path):
            self.traversal.append({'ip': ip, 'path': path})
        if self.SCANNER_UA_RE.search(ua):
            self.scanners.append({'ip': ip, 'ua': ua[:50]})

    def alerts(self, brute_threshold=3, scan_threshold=3):
        alerts = []
        for ip, count in self.ip_401.items():
            if count >= brute_threshold:
                alerts.append({'type': 'BRUTE_FORCE', 'severity': 'HIGH', 'ip': ip,
                                'detail': f"{count} failed auth attempts"})
        for ip, count in self.ip_404.items():
            if count >= scan_threshold:
                alerts.append({'type': 'SCAN_DETECTED', 'severity': 'MEDIUM', 'ip': ip,
                                'detail': f"{count} not-found requests"})
        for hit in self.sql_hits:
            alerts.append({'type': 'SQL_INJECTION', 'severity': 'CRITICAL', 'ip': hit['ip'],
                           'detail': f"Suspicious path: {hit['path'][:50]}"})
        for s in self.scanners:
            alerts.append({'type': 'SCANNER_UA', 'severity': 'HIGH', 'ip': s['ip'],
                           'detail': f"Scanner UA: {s['ua']}"})
        return sorted(alerts, key=lambda x: {'CRITICAL':0,'HIGH':1,'MEDIUM':2,'LOW':3}[x['severity']])


security_logs = [
    '192.168.1.1 - - [15/Oct/2024:09:00:01 +0700] "GET /index.html HTTP/1.1" 200 1234 "-" "Mozilla/5.0"',
    '10.0.0.99 - - [15/Oct/2024:09:00:02 +0700] "POST /login HTTP/1.1" 401 45 "-" "curl/7.68"',
    '10.0.0.99 - - [15/Oct/2024:09:00:03 +0700] "POST /login HTTP/1.1" 401 45 "-" "curl/7.68"',
    '10.0.0.99 - - [15/Oct/2024:09:00:04 +0700] "POST /login HTTP/1.1" 401 45 "-" "curl/7.68"',
    "10.0.0.50 - - [15/Oct/2024:09:00:05 +0700] \"GET /api/users?id=1' UNION SELECT * FROM users-- HTTP/1.1\" 400 56 \"-\" \"sqlmap/1.0\"",
    '10.0.0.51 - - [15/Oct/2024:09:00:06 +0700] "GET /files/../../../../etc/passwd HTTP/1.1" 403 45 "-" "python-requests"',
    '10.0.0.52 - - [15/Oct/2024:09:00:07 +0700] "GET /admin HTTP/1.1" 404 0 "-" "Nikto/2.1.6"',
    '10.0.0.52 - - [15/Oct/2024:09:00:08 +0700] "GET /phpmyadmin HTTP/1.1" 404 0 "-" "Nikto/2.1.6"',
    '10.0.0.52 - - [15/Oct/2024:09:00:09 +0700] "GET /.env HTTP/1.1" 404 0 "-" "Nikto/2.1.6"',
]

analyzer2 = LogAnalyzer()
security_analyzer = SecurityAnalyzer()

for line in security_logs:
    parsed = analyzer2.parse_line(line)
    if parsed:
        security_analyzer.analyze_line(line, parsed)

alerts = security_analyzer.alerts()
print(f"\n   Security Alerts ({len(alerts)} total):")
for alert in alerts:
    sev = alert['severity']
    print(f"\n   [{sev}] {alert['type']}")
    print(f"   IP: {alert['ip']}")
    print(f"   {alert['detail']}")
```

---

## 52.4 สรุป Part 52

```
Log Format Patterns:
CLF: host ident user [time] "request" status bytes
ELF: CLF + "referer" "user-agent"
Syslog: month day time host process[pid]: message

Key patterns:
[^\]]+  — content inside brackets (timestamps)
[^"]+   — content inside quotes (requests, UA)
\S+     — non-whitespace (host, ident, size)
\d{3}   — 3-digit status code

Analytics pipeline:
1. Parse each line → dict with named groups
2. Classify (static/api/admin/page)
3. Aggregate with Counter/defaultdict
4. Report top N with .most_common(N)

Security detection:
- Brute force: count 401 per IP > threshold
- Scanner: count 404 per IP > threshold
- SQLi: search for UNION SELECT, ' OR, etc.
- Path traversal: search for ../
- Scanner UA: known tool signatures

Performance:
- Compile patterns once, not per line
- Use re.search() for detection, re.match() for parsing
- Process logs line-by-line (streaming) for large files
- Use Counter for aggregation (O(n) one pass)
```

---

*[← Part 51: Web Framework Integration](part-51-web-frameworks.md) | [→ Part 53: Regex Security Patterns](part-53-security-patterns.md)*
