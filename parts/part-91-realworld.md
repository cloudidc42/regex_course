# Part 91: Real-World Regex Applications

> **ระดับ:** ผู้เชี่ยวชาญ-ระดับโลก | **เวลาเรียน:** ~110 นาที | **ข้อกำหนด:** Part 01-90

---

## 91.1 Log Parsing & Analysis Pipeline

```python
import re
from collections import defaultdict
from typing import Dict, List, Optional

print("Real-World Regex Applications:")
print("=" * 60)

print("\n1. Web server log parsing (Apache/Nginx Combined Log Format):")

# Apache/Nginx combined log format
COMBINED_LOG = re.compile(
    r'(?P<ip>\d{1,3}(?:\.\d{1,3}){3})\s+'
    r'(?P<ident>[^\s]+)\s+'
    r'(?P<user>[^\s]+)\s+'
    r'\[(?P<time>[^\]]+)\]\s+'
    r'"(?P<method>[A-Z]+)\s+(?P<path>[^\s"]+)\s+HTTP/(?P<http_ver>[\d.]+)"\s+'
    r'(?P<status>\d{3})\s+'
    r'(?P<size>\d+|-)\s*'
    r'(?:"(?P<referer>[^"]*)")?\s*'
    r'(?:"(?P<ua>[^"]*)")?',
    re.IGNORECASE
)

# Syslog format
SYSLOG = re.compile(
    r'(?P<month>\w{3})\s+(?P<day>\d{1,2})\s+(?P<time>\d{2}:\d{2}:\d{2})\s+'
    r'(?P<host>\S+)\s+'
    r'(?P<service>[^\[]+)\[(?P<pid>\d+)\]:\s+'
    r'(?P<message>.+)'
)

def parse_log_line(line: str) -> Optional[Dict]:
    m = COMBINED_LOG.match(line)
    if m:
        size = m.group('size')
        return {
            'ip':      m.group('ip'),
            'method':  m.group('method'),
            'path':    m.group('path'),
            'status':  int(m.group('status')),
            'size':    int(size) if size != '-' else 0,
            'ua':      m.group('ua') or '',
        }
    return None


def analyze_access_log(log_lines: List[str]) -> Dict:
    stats = {
        'total': 0,
        'status_codes': defaultdict(int),
        'top_paths': defaultdict(int),
        'top_ips': defaultdict(int),
        'errors': [],
    }

    for line in log_lines:
        entry = parse_log_line(line)
        if entry:
            stats['total'] += 1
            stats['status_codes'][entry['status']] += 1
            stats['top_paths'][entry['path']] += 1
            stats['top_ips'][entry['ip']] += 1
            if entry['status'] >= 400:
                stats['errors'].append(entry)

    return stats


# Sample Apache combined log lines
sample_logs = [
    '192.168.1.100 - alice [15/Jan/2024:10:30:45 +0000] "GET /index.html HTTP/1.1" 200 1234 "https://example.com" "Mozilla/5.0"',
    '10.0.0.5 - - [15/Jan/2024:10:30:46 +0000] "POST /api/login HTTP/1.1" 200 89 "-" "curl/7.68.0"',
    '192.168.1.200 - - [15/Jan/2024:10:30:47 +0000] "GET /admin/config HTTP/1.1" 403 0 "-" "python-requests/2.28.0"',
    '192.168.1.100 - alice [15/Jan/2024:10:30:48 +0000] "GET /dashboard HTTP/1.1" 200 5678 "https://example.com" "Mozilla/5.0"',
    '10.0.0.6 - - [15/Jan/2024:10:30:49 +0000] "GET /nonexistent HTTP/1.1" 404 0 "-" "Googlebot/2.1"',
    '10.0.0.7 - - [15/Jan/2024:10:30:50 +0000] "GET /../../etc/passwd HTTP/1.1" 400 0 "-" "curl/7.68.0"',
]

stats = analyze_access_log(sample_logs)
print(f"\n   Access log analysis ({stats['total']} requests):")
print(f"   Status codes: {dict(stats['status_codes'])}")
print(f"   Top IPs: {dict(list(stats['top_ips'].items())[:3])}")
print(f"   Errors ({len(stats['errors'])}): {[e['path'] for e in stats['errors']]}")
```

---

## 91.2 Configuration File Parsing

```python
import re
from typing import Dict, List, Tuple, Any

print("\nConfiguration File Parsing:")
print("=" * 60)

# INI/properties file parser
INI_SECTION   = re.compile(r'^\[(?P<section>[^\]]+)\]$')
INI_KEY_VALUE = re.compile(r'^(?P<key>[^=;#\s][^=]*?)\s*=\s*(?P<value>.+?)(?:\s*[;#].*)?$')
INI_COMMENT   = re.compile(r'^\s*[;#]')

def parse_ini(content: str) -> Dict[str, Dict[str, str]]:
    result = {'DEFAULT': {}}
    current_section = 'DEFAULT'

    for raw_line in content.splitlines():
        line = raw_line.strip()
        if not line or INI_COMMENT.match(line):
            continue
        m_sec = INI_SECTION.match(line)
        if m_sec:
            current_section = m_sec.group('section')
            result[current_section] = {}
            continue
        m_kv = INI_KEY_VALUE.match(line)
        if m_kv:
            result[current_section][m_kv.group('key').strip()] = m_kv.group('value').strip()

    return result


# Environment variable file (.env) parser
ENV_VAR = re.compile(
    r'^(?:export\s+)?'
    r'(?P<key>[A-Z_][A-Z0-9_]*)'
    r'\s*=\s*'
    r'(?P<value>(?:"(?:[^"\\]|\\.)*")|(?:\'(?:[^\'\\]|\\.)*\')|(?:[^\s#]*))'
    r'(?:\s*#.*)?$',
    re.IGNORECASE
)

def parse_env_file(content: str) -> Dict[str, str]:
    result = {}
    for line in content.splitlines():
        line = line.strip()
        if not line or line.startswith('#'):
            continue
        m = ENV_VAR.match(line)
        if m:
            key = m.group('key')
            val = m.group('value')
            if (val.startswith('"') and val.endswith('"')) or \
               (val.startswith("'") and val.endswith("'")):
                val = val[1:-1]
            result[key] = val
    return result


sample_ini = """
[database]
host = localhost
port = 5432
name = myapp_db
; Production password stored in vault
password = ${DB_PASSWORD}

[cache]
backend = redis
host = localhost
port = 6379
ttl = 3600
"""

sample_env = """
# Application settings
APP_NAME=MyApp
DEBUG=false
DATABASE_URL="postgresql://user:pass@localhost/mydb"
SECRET_KEY='my-secret-key-here'
export API_KEY=abc123xyz
MAX_CONNECTIONS=100
"""

print(f"\n   INI file parsing:")
ini_data = parse_ini(sample_ini)
for section, kv in ini_data.items():
    if kv:
        print(f"   [{section}]")
        for k, v in kv.items():
            print(f"     {k} = {v}")

print(f"\n   .env file parsing:")
env_data = parse_env_file(sample_env)
for k, v in env_data.items():
    masked = v[:4] + '***' if len(v) > 4 else '***'
    print(f"   {k} = {masked}")
```

---

## 91.3 Data Extraction & Transformation

```python
import re
from typing import List, Dict, Tuple

print("\nData Extraction & Transformation:")
print("=" * 60)

# Extract structured data from unstructured text
PRICE_PATTERN = re.compile(
    r'(?P<currency>[£$€¥]|USD|EUR|GBP|JPY|THB)\s*'
    r'(?P<amount>\d{1,3}(?:[,\s]\d{3})*(?:\.\d{1,4})?)',
    re.IGNORECASE
)

# Markdown table to list of dicts
MARKDOWN_TABLE_ROW = re.compile(r'\|([^|]+(?:\|[^|]+)*)\|')

def parse_markdown_table(text: str) -> List[Dict]:
    lines = [l.strip() for l in text.strip().splitlines() if l.strip()]
    if len(lines) < 3:
        return []

    headers = [h.strip() for h in lines[0].strip('|').split('|')]
    rows = []
    for line in lines[2:]:
        if not MARKDOWN_TABLE_ROW.match(line):
            continue
        cells = [c.strip() for c in line.strip('|').split('|')]
        if len(cells) == len(headers):
            rows.append(dict(zip(headers, cells)))

    return rows


# Extract all prices from text
def extract_prices(text: str) -> List[Dict]:
    results = []
    for m in PRICE_PATTERN.finditer(text):
        amount_str = m.group('amount').replace(',', '').replace(' ', '')
        results.append({
            'currency': m.group('currency'),
            'amount': float(amount_str),
            'raw': m.group(0),
        })
    return results


def normalize_text(text: str) -> str:
    """Normalize text for comparison."""
    t = text.lower()
    t = re.sub(r'[^\w\s]', ' ', t)
    t = re.sub(r'\s+', ' ', t).strip()
    return t


price_text = """
Product prices:
- Laptop: $1,299.99 USD
- Mouse: £24.99
- Keyboard: EUR 79.00
- Headphones: ¥8,500
- USB Cable: $9.99
"""

print(f"\n   Price extraction:")
prices = extract_prices(price_text)
for p in prices:
    print(f"   {p['currency']} {p['amount']:.2f} (from: {p['raw']!r})")


markdown_table = """
| Name    | Age | Role      |
|---------|-----|-----------|
| Alice   | 30  | Engineer  |
| Bob     | 25  | Designer  |
| Charlie | 35  | Manager   |
"""

print(f"\n   Markdown table parsing:")
table_data = parse_markdown_table(markdown_table)
for row in table_data:
    print(f"   {row}")


samples = [
    'Hello,   World!!!',
    'Python  3.11  is  great.',
    '  Multiple   spaces   everywhere  ',
]
print(f"\n   Text normalization:")
for s in samples:
    print(f"   {s!r}")
    print(f"   → {normalize_text(s)!r}")
```

---

## 91.4 สรุป Part 91

```
Real-World Regex Applications:

1. Log parsing:
   Apache combined format: IP + user + timestamp + method + path + status + size
   Use named groups for clarity: (?P<ip>...) (?P<status>...)
   Syslog: month day time host service[pid]: message
   JSON logs: key:value extraction with type detection

2. Configuration parsing:
   INI: [section] headers + key=value pairs + ; # comments
   .env: KEY=value or "quoted value" or 'single quoted'
   Pattern: skip empty lines, skip comments, then match key=value

3. Data extraction:
   Prices: multi-currency with optional thousands separator
   Dates: DD/MM/YYYY, YYYY-MM-DD, Month DD YYYY (named groups)
   Always normalize after extraction: strip extra chars, convert types

4. Table/structured data:
   Markdown tables: split on | after stripping leading/trailing |
   CSV: re.split(r',(?=(?:[^"]*"[^"]*")*[^"]*$)') for quoted fields

5. Text normalization:
   Lowercase → remove punctuation → collapse whitespace
   For comparison, not display — preserves semantic content
   Use re.sub() chains rather than complex single patterns
```

---

*[← Part 90: Advanced Python re Module](part-90-advanced-re.md) | [→ Part 92: Regex in Data Science & ML Pipelines](part-92-data-science.md)*
