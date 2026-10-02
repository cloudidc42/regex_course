# Part 18: Log File Parsing — การวิเคราะห์ Log Files

> **ระดับ:** กลาง | **เวลาเรียน:** ~70 นาที | **ข้อกำหนด:** Part 01-17

---

## 18.1 Log Format Overview

```
Log formats ที่พบบ่อย:
Apache Combined:  IP - USER [DATE] "METHOD URL HTTP" STATUS SIZE "REF" "UA"
Nginx:            IP - USER [DATE] "METHOD URL HTTP" STATUS SIZE "REF" "UA"
Syslog:           DATE HOSTNAME PROCESS[PID]: MESSAGE
Python logging:   DATE TIME LEVEL LOGGER: MESSAGE
JSON logs:        {"timestamp":"...","level":"...","message":"..."}
Application:      [DATE] [LEVEL] [CLASS] - MESSAGE
```

---

## 18.2 Apache/Nginx Access Log Parser

```python
import re
from datetime import datetime
from collections import defaultdict
from typing import Optional, List

class AccessLogParser:
    """Parse Apache/Nginx access log entries"""
    
    # Combined Log Format
    COMBINED = re.compile(
        r'^(?P<ip>\S+)'            # IP address
        r'\s+(?P<ident>\S+)'       # ident (usually -)
        r'\s+(?P<user>\S+)'        # user (usually -)
        r'\s+\[(?P<time>[^\]]+)\]' # [date time zone]
        r'\s+"(?P<method>\S+)'     # "METHOD
        r'\s+(?P<path>\S+)'        # /path/to/resource
        r'\s+(?P<protocol>[^"]+)"' # HTTP/1.1"
        r'\s+(?P<status>\d{3})'    # status code
        r'\s+(?P<size>\d+|-)'      # bytes sent
        r'(?:\s+"(?P<referer>[^"]*)")?'    # "Referer"
        r'(?:\s+"(?P<useragent>[^"]*)")?'  # "User-Agent"
    )
    
    MONTH_MAP = {
        'Jan': 1, 'Feb': 2, 'Mar': 3, 'Apr': 4,
        'May': 5, 'Jun': 6, 'Jul': 7, 'Aug': 8,
        'Sep': 9, 'Oct': 10, 'Nov': 11, 'Dec': 12,
    }
    
    @classmethod
    def parse_line(cls, line: str) -> Optional[dict]:
        m = cls.COMBINED.match(line.strip())
        if not m:
            return None
        
        d = m.groupdict()
        size = int(d['size']) if d['size'] != '-' else 0
        status = int(d['status'])
        
        return {
            'ip': d['ip'],
            'user': d['user'] if d['user'] != '-' else None,
            'time_raw': d['time'],
            'method': d['method'],
            'path': d['path'],
            'protocol': d['protocol'].strip(),
            'status': status,
            'status_class': f"{status // 100}xx",
            'size': size,
            'referer': d.get('referer') or None,
            'useragent': d.get('useragent') or None,
        }
    
    @classmethod
    def parse_log(cls, log_content: str) -> List[dict]:
        entries = []
        for line in log_content.strip().split('\n'):
            if line.strip() and not line.startswith('#'):
                entry = cls.parse_line(line)
                if entry:
                    entries.append(entry)
        return entries
    
    @classmethod
    def analyze(cls, entries: List[dict]) -> dict:
        total = len(entries)
        if not total:
            return {}
        
        status_counts = defaultdict(int)
        ip_counts = defaultdict(int)
        path_counts = defaultdict(int)
        method_counts = defaultdict(int)
        total_bytes = 0
        errors = []
        
        for e in entries:
            status_counts[e['status_class']] += 1
            ip_counts[e['ip']] += 1
            path_counts[e['path']] += 1
            method_counts[e['method']] += 1
            total_bytes += e['size']
            if e['status'] >= 400:
                errors.append(e)
        
        return {
            'total_requests': total,
            'total_bytes': total_bytes,
            'status_distribution': dict(status_counts),
            'top_ips': sorted(ip_counts.items(), key=lambda x: x[1], reverse=True)[:5],
            'top_paths': sorted(path_counts.items(), key=lambda x: x[1], reverse=True)[:5],
            'method_distribution': dict(method_counts),
            'error_count': len(errors),
            'error_rate': len(errors) / total * 100,
        }


# ทดสอบ
sample_access_log = """192.168.1.100 - - [02/Oct/2026:14:30:00 +0700] "GET /index.html HTTP/1.1" 200 2048 "https://google.com" "Mozilla/5.0"
10.0.0.5 - admin [02/Oct/2026:14:30:01 +0700] "POST /api/login HTTP/1.1" 200 512 "-" "curl/7.88.0"
192.168.1.100 - - [02/Oct/2026:14:30:02 +0700] "GET /style.css HTTP/1.1" 200 4096 "https://mysite.com/" "Mozilla/5.0"
203.0.113.5 - - [02/Oct/2026:14:30:03 +0700] "GET /admin HTTP/1.1" 403 256 "-" "python-requests/2.28"
192.168.1.100 - - [02/Oct/2026:14:30:04 +0700] "GET /notfound HTTP/1.1" 404 128 "-" "Mozilla/5.0"
10.0.0.5 - - [02/Oct/2026:14:30:05 +0700] "POST /api/login HTTP/1.1" 401 64 "-" "curl/7.88.0"
192.168.1.100 - - [02/Oct/2026:14:30:06 +0700] "GET /api/data HTTP/2.0" 200 8192 "-" "Mozilla/5.0"
"""

entries = AccessLogParser.parse_log(sample_access_log)
analysis = AccessLogParser.analyze(entries)

print("Access Log Analysis:")
print("=" * 60)
print(f"\n  Total requests: {analysis['total_requests']}")
print(f"  Total bytes:    {analysis['total_bytes']:,}")
print(f"  Error rate:     {analysis['error_rate']:.1f}%")
print(f"\n  Status codes:   {analysis['status_distribution']}")
print(f"  Methods:        {analysis['method_distribution']}")
print("\n  Top IPs:")
for ip, count in analysis['top_ips']:
    print(f"    {ip:20} {count} requests")
print("\n  Top paths:")
for path, count in analysis['top_paths']:
    print(f"    {path:30} {count} requests")
```

---

## 18.3 Error Log Parser

```python
import re
from collections import defaultdict
from typing import List, Optional

class ErrorLogParser:
    """Parse Apache/Nginx/Application error logs"""
    
    # Apache error format
    APACHE_ERROR = re.compile(
        r'\[(?P<day>\w+)\s+(?P<month>\w+)\s+(?P<date>\d+)\s+(?P<time>\d+:\d+:\d+\.\d+)\s+(?P<year>\d+)\]'
        r'\s+\[(?P<level>\w+)\]'
        r'(?:\s+\[pid\s+(?P<pid>\d+)\])?'
        r'(?:\s+\[client\s+(?P<client>[^\]]+)\])?'
        r'\s+(?P<message>.+)',
        re.IGNORECASE
    )
    
    # Nginx error format
    NGINX_ERROR = re.compile(
        r'(?P<year>\d{4})/(?P<month>\d{2})/(?P<day>\d{2})'
        r'\s+(?P<time>\d{2}:\d{2}:\d{2})'
        r'\s+\[(?P<level>\w+)\]'
        r'\s+(?P<pid>\d+)#(?P<tid>\d+):'
        r'(?:\s+\*(?P<cid>\d+))?'
        r'\s+(?P<message>.+)'
    )
    
    # Generic: [LEVEL] message
    GENERIC = re.compile(
        r'^(?P<timestamp>\d{4}-\d{2}-\d{2}[T ]\d{2}:\d{2}:\d{2}(?:\.\d+)?)?' 
        r'\s*\[?(?P<level>DEBUG|INFO|WARNING|WARN|ERROR|CRITICAL|FATAL|NOTICE)\]?'
        r'[:\s]+(?P<message>.+)',
        re.IGNORECASE
    )
    
    @classmethod
    def parse_line(cls, line: str) -> Optional[dict]:
        line = line.strip()
        if not line:
            return None
        
        for fmt, pattern in [('apache', cls.APACHE_ERROR), ('nginx', cls.NGINX_ERROR), ('generic', cls.GENERIC)]:
            m = pattern.match(line)
            if m:
                d = m.groupdict()
                return {
                    'format': fmt,
                    'level': (d.get('level') or 'info').upper(),
                    'message': (d.get('message') or line).strip(),
                    'raw': line,
                }
        
        return {'format': 'unknown', 'level': 'UNKNOWN', 'message': line, 'raw': line}
    
    @classmethod
    def filter_by_level(cls, entries: List[dict], min_level: str = 'WARNING') -> List[dict]:
        LEVELS = ['DEBUG', 'INFO', 'NOTICE', 'WARNING', 'WARN', 'ERROR', 'CRITICAL', 'FATAL']
        min_idx = LEVELS.index(min_level.upper()) if min_level.upper() in LEVELS else 3
        return [e for e in entries if e['level'] in LEVELS and LEVELS.index(e['level']) >= min_idx]
    
    @classmethod
    def extract_patterns(cls, entries: List[dict]) -> dict:
        """Extract common error patterns"""
        patterns = {
            'timeout': re.compile(r'timeout', re.IGNORECASE),
            'connection_refused': re.compile(r'connection refused', re.IGNORECASE),
            'memory': re.compile(r'out of memory|memory error', re.IGNORECASE),
            'permission': re.compile(r'permission denied|access forbidden', re.IGNORECASE),
            'not_found': re.compile(r'no such file|not found|404', re.IGNORECASE),
            'sql_error': re.compile(r'sql error|database error|query failed', re.IGNORECASE),
        }
        found = defaultdict(list)
        for entry in entries:
            for name, pattern in patterns.items():
                if pattern.search(entry['message']):
                    found[name].append(entry['message'][:80])
        return dict(found)


# ทดสอบ
error_log_content = """
2026-10-02 14:30:00 [ERROR] Database connection timeout after 30s
2026-10-02 14:30:05 [WARNING] High memory usage: 85% (3.4GB/4GB)
2026-10-02 14:30:10 [INFO] Backup completed successfully
2026-10-02 14:30:15 [CRITICAL] Permission denied: /var/data/config.db
2026-10-02 14:30:20 [ERROR] Connection refused: redis://localhost:6379
2026-10-02 14:30:25 [DEBUG] Cache miss for key: user_session_12345
2026-10-02 14:30:30 [ERROR] SQL Error: Query failed - table 'users' not found
2026-10-02 14:30:35 [WARNING] Slow query: 5.2s for SELECT * FROM orders
"""

entries = [ErrorLogParser.parse_line(line) for line in error_log_content.strip().split('\n') if line.strip()]
entries = [e for e in entries if e]

print("\nError Log Analysis:")
print("=" * 60)
level_counts = defaultdict(int)
for e in entries:
    level_counts[e['level']] += 1
    
for level, count in sorted(level_counts.items()):
    print(f"  {level:10} {count}")

errors_warnings = ErrorLogParser.filter_by_level(entries, 'WARNING')
print(f"\n  Errors/Warnings ({len(errors_warnings)}):")
for e in errors_warnings:
    print(f"    [{e['level']}] {e['message'][:60]}")

patterns_found = ErrorLogParser.extract_patterns(entries)
if patterns_found:
    print(f"\n  Error patterns found:")
    for ptype, messages in patterns_found.items():
        print(f"    {ptype}: {len(messages)} occurrences")
```

---

## 18.4 Syslog Parser

```python
import re
from typing import Optional

class SyslogParser:
    """Parse syslog format messages"""
    
    # Traditional syslog: MMM DD HH:MM:SS hostname process[pid]: message
    SYSLOG = re.compile(
        r'^(?P<month>Jan|Feb|Mar|Apr|May|Jun|Jul|Aug|Sep|Oct|Nov|Dec)'
        r'\s+(?P<day>\d{1,2})'
        r'\s+(?P<time>\d{2}:\d{2}:\d{2})'
        r'\s+(?P<host>\S+)'
        r'\s+(?P<process>[^\[:]+)'
        r'(?:\[(?P<pid>\d+)\])?'
        r':\s+(?P<message>.+)',
        re.IGNORECASE
    )
    
    # RFC 5424 structured syslog
    RFC5424 = re.compile(
        r'^<(?P<priority>\d+)>'
        r'(?P<version>\d+)\s+'
        r'(?P<timestamp>\S+)\s+'
        r'(?P<host>\S+)\s+'
        r'(?P<app>\S+)\s+'
        r'(?P<procid>\S+)\s+'
        r'(?P<msgid>\S+)\s+'
        r'(?P<structured_data>\[.*?\]|-)\s*'
        r'(?P<message>.*)',
        re.DOTALL
    )
    
    SEVERITY_NAMES = {
        0: 'Emergency', 1: 'Alert', 2: 'Critical', 3: 'Error',
        4: 'Warning', 5: 'Notice', 6: 'Info', 7: 'Debug',
    }
    
    @classmethod
    def parse_priority(cls, priority: int) -> dict:
        facility = priority >> 3
        severity = priority & 7
        return {
            'facility_code': facility,
            'severity': cls.SEVERITY_NAMES.get(severity, str(severity)),
            'severity_code': severity,
        }
    
    @classmethod
    def parse_line(cls, line: str) -> Optional[dict]:
        line = line.strip()
        
        m = cls.RFC5424.match(line)
        if m:
            d = m.groupdict()
            pri = cls.parse_priority(int(d['priority']))
            return {'format': 'rfc5424', **d, **pri}
        
        m = cls.SYSLOG.match(line)
        if m:
            d = m.groupdict()
            return {
                'format': 'traditional',
                'month': d['month'], 'day': d['day'],
                'time': d['time'], 'host': d['host'],
                'process': d['process'].strip(),
                'pid': d.get('pid'),
                'message': d['message'],
            }
        
        return None
    
    @classmethod
    def detect_security_events(cls, message: str) -> list:
        """Detect security-relevant events in syslog"""
        patterns = {
            'ssh_failed_login': re.compile(r'Failed password for .* from', re.IGNORECASE),
            'ssh_success': re.compile(r'Accepted (?:password|publickey) for', re.IGNORECASE),
            'sudo_attempt': re.compile(r'sudo:\s+\w+', re.IGNORECASE),
            'login_failure': re.compile(r'authentication failure', re.IGNORECASE),
            'brute_force': re.compile(r'Maximum authentication attempts exceeded', re.IGNORECASE),
        }
        events = []
        for event_type, pattern in patterns.items():
            if pattern.search(message):
                events.append(event_type)
        return events


# ทดสอบ
syslog_lines = [
    "Oct  2 14:30:00 webserver sshd[1234]: Failed password for root from 203.0.113.5 port 22 ssh2",
    "Oct  2 14:30:05 webserver sudo: alice : TTY=pts/0 ; PWD=/home/alice ; USER=root ; COMMAND=/bin/cat /etc/shadow",
    "Oct  2 14:30:10 webserver sshd[1235]: Accepted publickey for deploy from 192.168.1.100 port 54321 ssh2",
    "<34>1 2026-10-02T14:30:20Z webserver app 1234 ID47 [exampleSDID@32473 iut=\"3\"] error connecting",
]

print("\nSyslog Parsing:")
print("=" * 65)
for line in syslog_lines:
    entry = SyslogParser.parse_line(line)
    if entry:
        print(f"\n  [{entry['format']}]")
        if entry['format'] == 'traditional':
            print(f"  Process: {entry['process']} / Host: {entry['host']}")
        print(f"  Message: {entry['message'][:60]}")
        
        events = SyslogParser.detect_security_events(entry['message'])
        if events:
            print(f"  [SECURITY] {', '.join(events)}")
```

---

## 18.5 Application Log Parser (Python logging)

```python
import re
from typing import List, Optional
from collections import defaultdict

class PythonLogParser:
    """Parse Python logging format logs"""
    
    PYTHON_LOG = re.compile(
        r'^(?P<date>\d{4}-\d{2}-\d{2})'
        r'\s+(?P<time>\d{2}:\d{2}:\d{2}(?:[.,]\d+)?)'
        r'(?:\s+-\s+(?P<name>\S+))?'
        r'\s+-\s+(?P<level>DEBUG|INFO|WARNING|ERROR|CRITICAL)'
        r'\s+-\s+(?P<message>.+)',
        re.IGNORECASE
    )
    
    TRACEBACK_START = re.compile(r'^Traceback \(most recent call last\):', re.IGNORECASE)
    TRACEBACK_LINE = re.compile(r'^\s+File "(.+)", line (\d+), in (.+)')
    EXCEPTION_LINE = re.compile(r'^(\w+(?:\.\w+)*Error|\w+Exception):\s*(.*)')
    
    @classmethod
    def parse_line(cls, line: str) -> Optional[dict]:
        m = cls.PYTHON_LOG.match(line.strip())
        if not m:
            return None
        d = m.groupdict()
        return {
            'timestamp': f"{d['date']} {d['time']}",
            'logger': d.get('name', 'root'),
            'level': d['level'].upper(),
            'message': d['message'],
        }
    
    @classmethod
    def parse_log_with_tracebacks(cls, content: str) -> List[dict]:
        """Parse log including multi-line tracebacks"""
        entries = []
        current_entry = None
        in_traceback = False
        traceback_lines = []
        
        for line in content.split('\n'):
            log_entry = cls.parse_line(line)
            
            if log_entry:
                if current_entry and traceback_lines:
                    current_entry['traceback'] = '\n'.join(traceback_lines)
                    traceback_lines = []
                    in_traceback = False
                
                if current_entry:
                    entries.append(current_entry)
                current_entry = log_entry
            
            elif current_entry:
                if cls.TRACEBACK_START.match(line):
                    in_traceback = True
                    traceback_lines = [line]
                elif in_traceback:
                    traceback_lines.append(line)
                    exc_m = cls.EXCEPTION_LINE.match(line)
                    if exc_m:
                        current_entry['exception_type'] = exc_m.group(1)
                        current_entry['exception_msg'] = exc_m.group(2)
        
        if current_entry:
            if traceback_lines:
                current_entry['traceback'] = '\n'.join(traceback_lines)
            entries.append(current_entry)
        
        return entries


# ทดสอบ
app_log = """
2026-10-02 14:30:00,123 - myapp.views - INFO - GET /api/users/ HTTP/1.1 200 1234
2026-10-02 14:30:01,456 - myapp.db - DEBUG - Query executed in 0.05s: SELECT * FROM users
2026-10-02 14:30:02,789 - myapp.auth - WARNING - Failed login attempt for user: alice
2026-10-02 14:30:03,012 - myapp.views - ERROR - Unhandled exception in view users_detail
Traceback (most recent call last):
  File "/app/views.py", line 45, in users_detail
    user = User.objects.get(pk=user_id)
myapp.models.ObjectDoesNotExist: User matching query does not exist.
2026-10-02 14:30:04,345 - myapp.cache - INFO - Cache hit for key: user_profile_42
"""

entries = PythonLogParser.parse_log_with_tracebacks(app_log)

print("\nApplication Log:")
print("=" * 60)
for e in entries:
    level_map = {'DEBUG': 'D', 'INFO': 'I', 'WARNING': 'W', 'ERROR': 'E', 'CRITICAL': 'C'}
    mark = level_map.get(e['level'], '?')
    print(f"  [{mark}] {e['message'][:55]}")
    if 'exception_type' in e:
        print(f"     Exception: {e['exception_type']}: {e['exception_msg'][:40]}")
```

---

## 18.6 JSON Log Parser

```python
import re
import json
from typing import Optional, List

class JSONLogParser:
    """Parse structured JSON logs"""
    
    TIMESTAMP_FIELDS = ['timestamp', 'time', '@timestamp', 'datetime', 'date', 'ts']
    LEVEL_FIELDS = ['level', 'severity', 'log_level', 'loglevel']
    MESSAGE_FIELDS = ['message', 'msg', 'text', 'log', 'body']
    
    @classmethod
    def parse_line(cls, line: str) -> Optional[dict]:
        line = line.strip()
        if not line.startswith('{'):
            m = re.search(r'\{[^{}]*(?:\{[^{}]*\}[^{}]*)?\}', line)
            if not m:
                return None
            line = m.group(0)
        
        try:
            data = json.loads(line)
            if not isinstance(data, dict):
                return None
            
            normalized = {'raw': data}
            
            for field in cls.TIMESTAMP_FIELDS:
                if field in data:
                    normalized['timestamp'] = str(data[field])
                    break
            
            for field in cls.LEVEL_FIELDS:
                if field in data:
                    normalized['level'] = str(data[field]).upper()
                    break
            
            for field in cls.MESSAGE_FIELDS:
                if field in data:
                    normalized['message'] = str(data[field])
                    break
            
            for key in ['error', 'stack', 'user_id', 'request_id', 'service', 'host']:
                if key in data:
                    normalized[key] = data[key]
            
            return normalized
        except Exception:
            return None


# ทดสอบ
json_logs = [
    '{"timestamp":"2026-10-02T14:30:00Z","level":"info","message":"Server started","service":"api"}',
    '{"@timestamp":"2026-10-02T14:30:01Z","severity":"error","msg":"DB connection failed","error":"timeout"}',
    '{"ts":1727874600,"level":"warn","message":"High CPU","cpu_percent":87.5}',
    'not json at all',
]

print("\nJSON Log Parsing:")
print("=" * 60)
for line in json_logs:
    entry = JSONLogParser.parse_line(line)
    if entry:
        ts = entry.get('timestamp', 'N/A')
        level = entry.get('level', 'UNKNOWN')
        msg = entry.get('message', 'N/A')
        print(f"  OK [{level:7}] {msg[:45]}")
    else:
        print(f"  FAIL: {line[:40]!r}")
```

---

## 18.7 Log Statistics and Alerting

```python
import re
from collections import defaultdict
from typing import List

class LogAnalyzer:
    """Analyze log files for patterns and anomalies"""
    
    ERROR_THRESHOLD = 0.1
    SLOW_QUERY = re.compile(r'(?:query|request|response).*?(\d+\.?\d*)\s*(?:ms|s|seconds?)', re.IGNORECASE)
    
    @classmethod
    def detect_anomalies(cls, entries: List[dict]) -> List[dict]:
        anomalies = []
        total = len(entries)
        if not total:
            return []
        
        error_count = sum(1 for e in entries if e.get('level') in ('ERROR', 'CRITICAL'))
        error_rate = error_count / total
        
        if error_rate > cls.ERROR_THRESHOLD:
            anomalies.append({
                'type': 'high_error_rate',
                'value': f"{error_rate:.1%}",
                'severity': 'warning' if error_rate < 0.3 else 'critical',
            })
        
        message_counts = defaultdict(int)
        for e in entries:
            if e.get('level') in ('ERROR', 'CRITICAL'):
                msg = e.get('message', '')[:80]
                message_counts[msg] += 1
        
        for msg, count in message_counts.items():
            if count > 5:
                anomalies.append({
                    'type': 'repeated_error',
                    'message': msg,
                    'count': count,
                    'severity': 'warning',
                })
        
        return anomalies
    
    @classmethod
    def summarize(cls, entries: List[dict]) -> str:
        total = len(entries)
        levels = defaultdict(int)
        for e in entries:
            levels[e.get('level', 'UNKNOWN')] += 1
        
        lines = [f"Log Summary: {total} entries"]
        for level in ['CRITICAL', 'ERROR', 'WARNING', 'INFO', 'DEBUG']:
            count = levels.get(level, 0)
            if count > 0:
                pct = count / total * 100
                bar = '#' * min(20, int(pct / 5)) + '.' * max(0, 20 - int(pct / 5))
                lines.append(f"  {level:10} {bar} {count:4} ({pct:.1f}%)")
        
        return '\n'.join(lines)


# ทดสอบ
combined_entries = [
    {'level': 'INFO', 'message': 'Request processed', 'timestamp': '14:30:00'},
    {'level': 'ERROR', 'message': 'Database query failed', 'timestamp': '14:30:01'},
    {'level': 'ERROR', 'message': 'Database query failed', 'timestamp': '14:30:02'},
    {'level': 'ERROR', 'message': 'Database query failed', 'timestamp': '14:30:03'},
    {'level': 'WARNING', 'message': 'Slow query: 5200ms for SELECT', 'timestamp': '14:30:04'},
    {'level': 'INFO', 'message': 'Cache hit', 'timestamp': '14:30:05'},
    {'level': 'ERROR', 'message': 'Database query failed', 'timestamp': '14:30:06'},
    {'level': 'CRITICAL', 'message': 'Out of memory', 'timestamp': '14:30:07'},
    {'level': 'ERROR', 'message': 'Database query failed', 'timestamp': '14:30:08'},
    {'level': 'ERROR', 'message': 'Database query failed', 'timestamp': '14:30:09'},
]

print("\nLog Analyzer:")
print("=" * 60)
print("\n" + LogAnalyzer.summarize(combined_entries))

anomalies = LogAnalyzer.detect_anomalies(combined_entries)
if anomalies:
    print("\n  Anomalies Detected:")
    for a in anomalies:
        severity = a['severity'].upper()
        print(f"  [{severity}] {a['type']}: ", end='')
        if a['type'] == 'high_error_rate':
            print(f"{a['value']}")
        else:
            print(f"'{a['message'][:40]}' x{a['count']}")
```

---

## 18.8 สรุป Part 18

```
Pattern สำคัญสำหรับ Log Parsing:
Apache access:  ^(\S+) \S+ \S+ \[([^\]]+)\] "(\w+) (\S+) ([^"]+)" (\d{3}) (\d+|-)
Log level:      \b(DEBUG|INFO|WARNING|ERROR|CRITICAL)\b
IP in log:      (?:from|client)((?:\d{1,3}\.){3}\d{1,3})
Duration:       (\d+\.?\d*)\s*(?:ms|s|seconds?)
Timestamp:      \d{4}-\d{2}-\d{2}[T ]\d{2}:\d{2}:\d{2}
Stack trace:    ^Traceback \(most recent call last\):
Exception:      ^(\w+(?:\.\w+)*(?:Error|Exception)):\s*(.+)
```

---

*[← Part 17: HTML Parsing](part-17-html-parsing.md) | [→ Part 19: Data Extraction](part-19-data-extraction.md)*
