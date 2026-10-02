# Part 34: Log Analysis — วิเคราะห์ Log Files ด้วย Regex

> **ระดับ:** กลาง-สูง | **เวลาเรียน:** ~65 นาที | **ข้อกำหนด:** Part 01-33

---

## 34.1 Web Server Log Parsers

```python
import re
from typing import List, Dict, Optional
from collections import Counter, defaultdict

class ApacheLogParser:
    """Parse Apache/Nginx access logs"""
    
    # Combined Log Format
    COMBINED = re.compile(
        r'(?P<ip>\S+)\s+'           # Client IP
        r'(?P<ident>\S+)\s+'        # Ident (usually -)
        r'(?P<user>\S+)\s+'         # Auth user (usually -)
        r'\[(?P<time>[^\]]+)\]\s+'  # Timestamp
        r'"(?P<request>[^"]+)"\s+'  # Request line
        r'(?P<status>\d{3})\s+'     # HTTP status
        r'(?P<bytes>\S+)'           # Bytes transferred
        r'(?:\s+"(?P<referer>[^"]*)"\s+"(?P<agent>[^"]*)")?'
    )
    
    REQUEST = re.compile(
        r'(?P<method>[A-Z]+)\s+(?P<path>[^\s?]+)(?:\?(?P<query>[^\s]*))?\s+HTTP/(?P<ver>[\d.]+)'
    )
    
    @classmethod
    def parse_line(cls, line: str) -> Optional[Dict]:
        m = cls.COMBINED.match(line.strip())
        if not m:
            return None
        
        d = m.groupdict()
        
        req_m = cls.REQUEST.match(d.get('request', ''))
        if req_m:
            d.update({
                'method': req_m.group('method'),
                'path': req_m.group('path'),
                'query': req_m.group('query'),
                'http_version': req_m.group('ver'),
            })
        
        d['status'] = int(d['status'])
        d['bytes'] = int(d['bytes']) if d['bytes'] != '-' else 0
        
        return d
    
    @classmethod
    def parse_log(cls, log_text: str) -> List[Dict]:
        entries = []
        for line in log_text.strip().split('\n'):
            if line:
                entry = cls.parse_line(line)
                if entry:
                    entries.append(entry)
        return entries
    
    @classmethod
    def analyze(cls, entries: List[Dict]) -> Dict:
        if not entries:
            return {}
        
        status_counts = Counter(e['status'] for e in entries)
        top_ips = Counter(e['ip'] for e in entries).most_common(5)
        top_paths = Counter(e.get('path', '') for e in entries).most_common(5)
        total_bytes = sum(e.get('bytes', 0) for e in entries)
        errors = [e for e in entries if e['status'] >= 400]
        error_rate = len(errors) / len(entries) * 100
        
        status_groups = defaultdict(int)
        for e in entries:
            status_groups[f"{e['status'] // 100}xx"] += 1
        
        return {
            'total_requests': len(entries),
            'total_bytes_mb': round(total_bytes / 1024 / 1024, 2),
            'error_rate_pct': round(error_rate, 1),
            'status_groups': dict(status_groups),
            'top_status': status_counts.most_common(5),
            'top_ips': top_ips,
            'top_paths': top_paths,
        }


# ทดสอบ
apache_log = '''192.168.1.100 - - [02/Oct/2024:10:00:01 +0700] "GET /api/users HTTP/1.1" 200 1234 "-" "Mozilla/5.0"
10.0.0.50 - alice [02/Oct/2024:10:00:02 +0700] "POST /api/login HTTP/1.1" 200 512 "https://example.com" "axios/1.4"
192.0.2.200 - - [02/Oct/2024:10:00:03 +0700] "GET /api/admin HTTP/1.1" 403 256 "-" "curl/7.68"
192.168.1.100 - - [02/Oct/2024:10:00:04 +0700] "GET /api/users/42 HTTP/1.1" 200 890 "-" "Mozilla/5.0"
10.0.0.99 - - [02/Oct/2024:10:00:05 +0700] "GET /notfound HTTP/1.1" 404 145 "-" "Googlebot/2.1"
192.0.2.200 - - [02/Oct/2024:10:00:06 +0700] "DELETE /api/users/1 HTTP/1.1" 401 120 "-" "curl/7.68"
192.168.1.100 - - [02/Oct/2024:10:00:07 +0700] "GET /api/products?category=tech&page=2 HTTP/1.1" 200 5678 "-" "Mozilla/5.0"
10.0.0.50 - alice [02/Oct/2024:10:00:08 +0700] "PUT /api/users/42 HTTP/1.1" 200 678 "-" "axios/1.4"
192.0.2.200 - - [02/Oct/2024:10:00:09 +0700] "GET /api/secret HTTP/1.1" 403 256 "-" "python-requests/2.28"
10.0.0.99 - - [02/Oct/2024:10:00:10 +0700] "GET /api/status HTTP/1.1" 500 89 "-" "HealthCheck/1.0"
'''

parser = ApacheLogParser
entries = parser.parse_log(apache_log)
stats = parser.analyze(entries)

print("Apache Log Analysis:")
print("=" * 60)
print(f"Total requests: {stats['total_requests']}")
print(f"Error rate: {stats['error_rate_pct']}%")
print(f"Status groups: {stats['status_groups']}")
print(f"Top IPs: {stats['top_ips']}")
print(f"Top paths: {stats['top_paths'][:3]}")
```

---

## 34.2 Application Log Analyzers

```python
import re
from typing import List, Dict
from collections import Counter

class AppLogParser:
    """Parse structured application logs"""
    
    STRUCTURED = re.compile(
        r'(?P<timestamp>\d{4}-\d{2}-\d{2}[T ]\d{2}:\d{2}:\d{2}(?:\.\d+)?(?:Z|[+-]\d{2}:?\d{2})?)\s+'
        r'(?P<level>DEBUG|INFO|WARN(?:ING)?|ERROR|FATAL|CRITICAL)\s+'
        r'(?:\[(?P<component>[^\]]+)\]\s+)?'
        r'(?P<message>.+)'
    )
    
    EXCEPTION = re.compile(
        r'(?P<exc_type>[A-Z][a-zA-Z0-9_]*(?:Error|Exception|Warning|Fault))'
        r'(?:\s*:\s*(?P<exc_msg>[^\n]+))?'
    )
    
    DURATION = re.compile(
        r'(?:took|duration|elapsed|latency)\s*[=:]?\s*'
        r'(?P<value>[\d.]+)\s*(?P<unit>ms|s|sec|seconds?|millis(?:econds?)?)'
        ,re.IGNORECASE
    )
    
    @classmethod
    def parse_structured(cls, log_text: str) -> List[Dict]:
        entries = []
        for line in log_text.strip().split('\n'):
            m = cls.STRUCTURED.match(line.strip())
            if m:
                entry = m.groupdict()
                dur_m = cls.DURATION.search(line)
                if dur_m:
                    val = float(dur_m.group('value'))
                    unit = dur_m.group('unit').lower()
                    if unit.startswith('ms') or unit.startswith('milli'):
                        val = val / 1000
                    entry['duration_sec'] = val
                entries.append(entry)
        return entries
    
    @classmethod
    def extract_errors(cls, log_text: str) -> List[Dict]:
        errors = []
        lines = log_text.split('\n')
        for i, line in enumerate(lines):
            exc_m = cls.EXCEPTION.search(line)
            if exc_m:
                errors.append({
                    'line': i + 1,
                    'type': exc_m.group('exc_type'),
                    'message': (exc_m.group('exc_msg') or '').strip(),
                })
        return errors
    
    @classmethod
    def performance_summary(cls, entries: List[Dict]) -> Dict:
        durations = sorted(e['duration_sec'] for e in entries if 'duration_sec' in e)
        if not durations:
            return {}
        n = len(durations)
        return {
            'count': n,
            'avg_ms': round(sum(durations) / n * 1000, 1),
            'p50_ms': round(durations[n // 2] * 1000, 1),
            'p90_ms': round(durations[int(n * 0.9)] * 1000, 1),
            'slow_requests': sum(1 for d in durations if d > 1.0),
        }


# ทดสอบ
app_log = """2024-10-02T10:00:01.123Z INFO  [api] GET /users - took 45ms - status=200
2024-10-02T10:00:01.456Z DEBUG [db]  Query executed in 12ms
2024-10-02T10:00:02.001Z INFO  [api] POST /auth/login - took 234ms - status=200
2024-10-02T10:00:02.567Z WARN  [api] Rate limit approaching for IP 192.0.2.100
2024-10-02T10:00:03.000Z ERROR [db]  ConnectionError: Connection pool exhausted after 5000ms
2024-10-02T10:00:03.100Z INFO  [api] GET /products - took 890ms - status=200
2024-10-02T10:00:04.000Z INFO  [api] GET /cache/stats - took 2ms - status=200
2024-10-02T10:00:04.500Z WARN  [api] Slow query detected: took 1250ms
2024-10-02T10:00:05.000Z INFO  [api] DELETE /sessions/abc123 - took 15ms
"""

app_parser = AppLogParser
entries = app_parser.parse_structured(app_log)

print("\nApplication Log Analysis:")
print("=" * 60)

level_counts = Counter(e['level'] for e in entries)
print(f"Log levels: {dict(level_counts)}")

errors = app_parser.extract_errors(app_log)
print(f"\nErrors ({len(errors)}):")
for e in errors:
    print(f"  Line {e['line']}: {e['type']} - {e['message'][:60]}")

perf = app_parser.performance_summary(entries)
if perf:
    print(f"\nPerformance (n={perf['count']}):")
    print(f"  avg={perf['avg_ms']}ms  p50={perf['p50_ms']}ms  p90={perf['p90_ms']}ms")
    print(f"  slow(>1s): {perf['slow_requests']}")
```

---

## 34.3 สรุป Part 34

```
Apache Combined Log Format:
(\S+) (\S+) (\S+) \[([^\]]+)\] "([^"]+)" (\d{3}) (\S+) "([^"]*)" "([^"]*)"
 IP   ident  user   timestamp    request   status bytes referer  user-agent

Structured App Log:
(\d{4}-\d{2}-\d{2}[T ]\d{2}:\d{2}:\d{2})\s+(DEBUG|INFO|WARN|ERROR)\s+(.+)
  timestamp                                   level                  message

Duration:
(?:took|duration|elapsed)\s*(?P<value>[\d.]+)\s*(?P<unit>ms|s|sec)

Exception:
[A-Z][a-zA-Z0-9_]*(?:Error|Exception|Warning)\s*:\s*(.+)

Tips:
- group entries ด้วย request_id เพื่อ trace single request
- percentile: durations.sort(); p90 = [int(n*0.9)]
- parse timestamp เป็น datetime เพื่อ filter/sort
```

---

*[← Part 33: API Patterns](part-33-api-patterns.md) | [→ Part 35: Text Processing](part-35-text-processing.md)*
