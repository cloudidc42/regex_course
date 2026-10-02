# Part 68: Log Analysis & Monitoring with Regex

> **ระดับ:** กลาง-สูง | **เวลาเรียน:** ~75 นาที | **ข้อกำหนด:** Part 01-67

---

## 68.1 Common Log Format Parsing

```python
import re
from typing import Dict, List, Optional

print("Log Analysis & Monitoring with Regex:")
print("=" * 60)

print("\n1. Apache/Nginx Common Log Format:")

# Common Log Format:
# 127.0.0.1 - frank [10/Oct/2000:13:55:36 -0700] "GET /apache_pb.gif HTTP/1.0" 200 2326
APACHE_CLF = re.compile(
    r'^(?P<host>\S+)\s+'
    r'(?P<ident>\S+)\s+'
    r'(?P<user>\S+)\s+'
    r'\[(?P<time>[^\]]+)\]\s+'
    r'"(?P<method>\w+)\s+(?P<path>[^\s"]+)\s+HTTP/(?P<http_ver>[^"]+)"\s+'
    r'(?P<status>\d{3})\s+'
    r'(?P<size>\d+|-)'
    r'(?:\s+"(?P<referer>[^"]*)")?\s*'
    r'(?:"(?P<ua>[^"]*)")?'
)

# Combined Log Format (Apache/Nginx)
APACHE_COMBINED = re.compile(
    r'^(?P<host>\S+)\s+\S+\s+\S+\s+'
    r'\[(?P<time>[^\]]+)\]\s+'
    r'"(?P<method>\w+)\s+(?P<path>\S+)\s+HTTP/(?P<ver>[^"]+)"\s+'
    r'(?P<status>\d{3})\s+(?P<size>\d+|-)\s+'
    r'"(?P<referer>[^"]*)"\s+'
    r'"(?P<ua>[^"]*)"'
)

CLF_TIME = re.compile(
    r'(?P<day>\d{2})/(?P<month>\w{3})/(?P<year>\d{4}):'
    r'(?P<hour>\d{2}):(?P<min>\d{2}):(?P<sec>\d{2})\s+'
    r'(?P<tz>[+-]\d{4})'
)

MONTHS = {'Jan':1,'Feb':2,'Mar':3,'Apr':4,'May':5,'Jun':6,
          'Jul':7,'Aug':8,'Sep':9,'Oct':10,'Nov':11,'Dec':12}


def parse_clf_time(time_str: str) -> Optional[str]:
    m = CLF_TIME.match(time_str)
    if not m:
        return None
    return (f"{m.group('year')}-{MONTHS.get(m.group('month'),0):02d}-"
            f"{m.group('day')} {m.group('hour')}:{m.group('min')}:{m.group('sec')}")


def parse_apache_log(line: str) -> Optional[Dict]:
    m = APACHE_COMBINED.match(line) or APACHE_CLF.match(line)
    if not m:
        return None
    d = m.groupdict()
    d['status'] = int(d['status'])
    d['size']   = int(d['size']) if d.get('size') and d['size'] != '-' else 0
    d['time_iso'] = parse_clf_time(d.get('time', ''))
    return d


apache_logs = [
    '192.168.1.10 - alice [15/Jan/2024:10:23:45 +0700] "GET /api/users HTTP/1.1" 200 1234 "https://app.example.com" "Mozilla/5.0"',
    '10.0.0.5 - - [15/Jan/2024:10:23:46 +0700] "POST /api/login HTTP/1.1" 401 234 "-" "curl/7.68.0"',
    '203.0.113.42 - - [15/Jan/2024:10:23:47 +0700] "GET /admin/config HTTP/1.1" 403 0 "-" "python-requests/2.28"',
    '192.168.1.20 - bob [15/Jan/2024:10:23:48 +0700] "DELETE /api/data/42 HTTP/1.1" 204 0 "-" "axios/1.4.0"',
]

print(f"\n   {'Host':<16} {'Method':<8} {'Path':<22} {'Status':<8} {'Size'}")
print(f"   {'-'*16} {'-'*8} {'-'*22} {'-'*8} {'-'*8}")
for line in apache_logs:
    r = parse_apache_log(line)
    if r:
        print(f"   {r['host']:<16} {r['method']:<8} {r['path']:<22} {r['status']:<8} {r['size']}")
```

---

## 68.2 Application Log Parsing

```python
import re
from typing import Dict, List, Optional
from collections import Counter

print("\nApplication Log Parsing:")
print("=" * 60)

PYTHON_LOG = re.compile(
    r'^(?P<date>\d{4}-\d{2}-\d{2})\s+'
    r'(?P<time>\d{2}:\d{2}:\d{2}),(?P<ms>\d{3})\s+-\s+'
    r'(?P<module>\S+)\s+-\s+'
    r'(?P<level>DEBUG|INFO|WARNING|ERROR|CRITICAL)\s+-\s+'
    r'(?P<message>.+)$'
)

SYSLOG = re.compile(
    r'^(?P<month>\w{3})\s+(?P<day>\d{1,2})\s+'
    r'(?P<time>\d{2}:\d{2}:\d{2})\s+'
    r'(?P<host>\S+)\s+'
    r'(?P<process>[^\[:\s]+)(?:\[(?P<pid>\d+)\])?:\s+'
    r'(?P<message>.+)$'
)

TRACEBACK    = re.compile(r'^Traceback \(most recent call last\):')
TRACE_FILE   = re.compile(r'^\s+File "(?P<file>[^"]+)", line (?P<line>\d+)')
TRACE_EXCEPT = re.compile(r'^(?P<exc>[A-Za-z][A-Za-z0-9_]*(?:Error|Exception|Warning|Fault)):\s*(?P<msg>.+)$')


def parse_log_line(line: str) -> Optional[Dict]:
    for parser in [PYTHON_LOG, SYSLOG]:
        m = parser.match(line)
        if m:
            return m.groupdict()
    return None


def analyze_logs(lines: List[str]) -> Dict:
    stats = {'total': len(lines), 'by_level': Counter(), 'errors': []}
    for line in lines:
        r = parse_log_line(line)
        if r:
            level = r.get('level', 'UNKNOWN').upper()
            stats['by_level'][level] += 1
            if level in ('ERROR', 'CRITICAL'):
                stats['errors'].append({'level': level, 'message': r.get('message', '')[:80]})
    return stats


app_logs = [
    "2024-01-15 10:23:45,123 - app.api - INFO - GET /api/users 200 45ms",
    "2024-01-15 10:23:46,234 - app.auth - WARNING - Failed login for user alice@example.com",
    "2024-01-15 10:23:47,345 - app.db - ERROR - Connection timeout after 30s",
    "2024-01-15 10:23:48,456 - app.api - INFO - POST /api/data 201 12ms",
    "2024-01-15 10:23:49,567 - app.cache - DEBUG - Cache miss for key user:1234",
    "2024-01-15 10:23:50,678 - app.worker - CRITICAL - Queue overflow: 10000 items",
    "2024-01-15 10:23:51,789 - app.api - ERROR - Unhandled exception in request handler",
]

stats = analyze_logs(app_logs)
print(f"\n   Log analysis:")
print(f"   Total lines: {stats['total']}")
print(f"   By level:    {dict(stats['by_level'])}")
print(f"\n   Errors:")
for e in stats['errors']:
    print(f"   [{e['level']}] {e['message']}")
```

---

## 68.3 Performance & Timing Pattern Extraction

```python
import re
from typing import List, Dict
from collections import defaultdict
import statistics

print("\nPerformance & Timing Analysis:")
print("=" * 60)

RESPONSE_TIME = re.compile(
    r'(?:took|in|duration|elapsed|time)\s+(?P<value>[\d.]+)\s*(?P<unit>ms|s|\xb5s)',
    re.IGNORECASE
)

MEMORY_RE = re.compile(
    r'(?:memory|mem|ram|heap)\s*(?:usage|used|free)?\s*[=:]\s*(?P<value>[\d.]+)\s*(?P<unit>KB|MB|GB|bytes?)',
    re.IGNORECASE
)


def extract_timings(logs: List[str]) -> Dict:
    timings = defaultdict(list)
    for line in logs:
        m = RESPONSE_TIME.search(line)
        if m:
            value = float(m.group('value'))
            unit  = m.group('unit')
            if unit == 's':           value *= 1000
            elif unit == '\xb5s':     value /= 1000
            timings['response_ms'].append(value)
        m = MEMORY_RE.search(line)
        if m:
            value = float(m.group('value'))
            unit  = m.group('unit').upper()
            if 'KB' in unit:    value /= 1024
            elif 'GB' in unit:  value *= 1024
            elif 'BYTES' in unit: value /= (1024 * 1024)
            timings['memory_mb'].append(value)
    return dict(timings)


perf_logs = [
    "2024-01-15 10:00:01 - Request GET /api/v1/users took 45ms",
    "2024-01-15 10:00:02 - DB query duration 120ms - SELECT * FROM users",
    "2024-01-15 10:00:03 - Cache lookup elapsed 2ms for key=session:abc",
    "2024-01-15 10:00:04 - Slow query detected: 1500ms",
    "2024-01-15 10:00:05 - Request POST /api/v1/upload took 2.3s",
    "2024-01-15 10:00:06 - Memory usage: 512MB heap used",
    "2024-01-15 10:00:07 - Request GET /api/v1/reports took 340ms",
    "2024-01-15 10:00:08 - Memory usage: 768MB heap used",
    "2024-01-15 10:00:09 - DB query elapsed 85ms",
    "2024-01-15 10:00:10 - Request GET /api/v1/search took 0.8s",
]

timings = extract_timings(perf_logs)
for metric, values in timings.items():
    if values:
        print(f"\n   {metric}:")
        print(f"   Count: {len(values)}, Min: {min(values):.1f}, Max: {max(values):.1f}, Mean: {statistics.mean(values):.1f}")
        if len(values) > 1:
            print(f"   Stdev: {statistics.stdev(values):.1f}")
```

---

## 68.4 Error Pattern Detection & Alerting

```python
import re
from typing import Dict, List
from collections import Counter

print("\nError Pattern Detection:")
print("=" * 60)

ERROR_PATTERNS = {
    'oom':          re.compile(r'out.of.memory|OutOfMemoryError|OOM', re.IGNORECASE),
    'timeout':      re.compile(r'(?:connection|request|query)\s+timed?\s+out|TimeoutError', re.IGNORECASE),
    'db_error':     re.compile(r'(?:SQL|database|DB)\s+(?:error|exception|failed)', re.IGNORECASE),
    'auth_fail':    re.compile(r'(?:auth(?:entication)?|login)\s+(?:failed|error|denied)', re.IGNORECASE),
    'rate_limit':   re.compile(r'rate\s+limit(?:ed)?|too\s+many\s+requests|429', re.IGNORECASE),
    'disk_full':    re.compile(r'(?:disk|filesystem|volume)\s+(?:full|out\s+of\s+space)|ENOSPC', re.IGNORECASE),
    'null_pointer': re.compile(r'NullPointerException|AttributeError.*NoneType', re.IGNORECASE),
    'ssl_error':    re.compile(r'SSL(?:\s+certificate)?(?:\s+(?:error|failed|expired|invalid))', re.IGNORECASE),
}

TIMESTAMP_RE = re.compile(r'\d{4}-\d{2}-\d{2}[T ]\d{2}:\d{2}:\d{2}')


def classify_error(message: str) -> List[str]:
    return [name for name, pattern in ERROR_PATTERNS.items() if pattern.search(message)]


def analyze_error_log(log_lines: List[str]) -> Dict:
    results = {'total_errors': 0, 'by_type': Counter(), 'samples': {}}
    for line in log_lines:
        if any(lvl in line.upper() for lvl in ('ERROR', 'CRITICAL', 'FATAL')):
            results['total_errors'] += 1
            types = classify_error(line)
            for t in types:
                results['by_type'][t] += 1
                if t not in results['samples']:
                    results['samples'][t] = line[:80]
            if not types:
                results['by_type']['uncategorized'] += 1
    return results


mixed_error_logs = [
    "2024-01-15 10:01:00 ERROR - Database connection timed out after 30s",
    "2024-01-15 10:01:05 ERROR - SQL error: too many connections to database",
    "2024-01-15 10:01:10 INFO  - Health check OK",
    "2024-01-15 10:01:15 ERROR - Authentication failed for user bob@example.com",
    "2024-01-15 10:01:20 ERROR - Out of memory: kill process 1234 or sacrifice child",
    "2024-01-15 10:01:25 WARNING - Rate limit exceeded: 100 req/min",
    "2024-01-15 10:01:30 ERROR - SSL certificate expired for api.partner.com",
    "2024-01-15 10:01:35 CRITICAL - Disk full on /var/log: ENOSPC",
    "2024-01-15 10:01:40 ERROR - NullPointerException in UserService.getById()",
    "2024-01-15 10:01:45 ERROR - Request timed out after 5000ms",
    "2024-01-15 10:01:50 ERROR - Authentication error: invalid JWT token",
    "2024-01-15 10:01:55 ERROR - Unknown error occurred processing request",
]

result = analyze_error_log(mixed_error_logs)
print(f"\n   Error analysis summary:")
print(f"   Total errors: {result['total_errors']}")
print(f"\n   By type:")
for err_type, count in result['by_type'].most_common():
    sample = result['samples'].get(err_type, '')[:50]
    print(f"   {err_type:<18} {count:>3}x  \"{sample}\"")
```

---

## 68.5 สรุป Part 68

```
Log Analysis Regex Patterns:

1. Apache CLF:
   ^(\S+) \S+ \S+ \[([^\]]+)\] "(\w+) (\S+) HTTP/([\d.]+)" (\d{3}) (\d+|-)

2. Python logging:
   ^(\d{4}-\d{2}-\d{2}) (\d{2}:\d{2}:\d{2}),(\d{3}) - (\S+) - (DEBUG|INFO|WARNING|ERROR|CRITICAL) - (.+)$

3. Syslog:
   ^(\w{3}) +(\d{1,2}) (\d{2}:\d{2}:\d{2}) (\S+) ([^\[\s:]+)(?:\[(\d+)\])?: (.+)$

4. Performance:
   Duration: ([\d.]+)\s*(ms|s|µs)
   Memory:   ([\d.]+)\s*(KB|MB|GB|bytes?)

5. Error classification:
   OOM:     out.of.memory|OutOfMemoryError|OOM
   Timeout: (connection|request|query)\s+timed?\s+out
   Auth:    (auth|login)\s+(failed|error|denied)

6. Stack trace:
   Start:     ^Traceback \(most recent call last\):
   File ref:  ^\s+File "([^"]+)", line (\d+)
   Exception: ^([A-Za-z]\w*(?:Error|Exception)): (.+)$
```

---

*[← Part 67: Web Scraping & HTML Parsing](part-67-web-scraping.md) | [→ Part 69: Security Input Validation](part-69-security-validation.md)*
