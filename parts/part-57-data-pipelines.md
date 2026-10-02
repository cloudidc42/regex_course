# Part 57: Data Pipelines & ETL with Regex

> **ระดับ:** กลาง-สูง | **เวลาเรียน:** ~75 นาที | **ข้อกำหนด:** Part 01-56

---

## 57.1 CSV / Delimited File Parsing

```python
import re
from typing import List, Dict, Iterator

print("CSV/Delimited Parsing with Regex:")
print("=" * 60)

print("\n1. Robust CSV field parser:")

CSV_FIELD = re.compile(
    r'"((?:[^"]|"")*)"| ([^,\n\r]*)'
)

def parse_csv_line(line: str) -> List[str]:
    fields = []
    pos = 0
    while pos <= len(line):
        m = CSV_FIELD.match(line, pos)
        if not m:
            break
        if m.group(1) is not None:
            fields.append(m.group(1).replace('""', '"'))
        else:
            fields.append(m.group(2) if m.group(2) is not None else '')
        pos = m.end()
        if pos < len(line) and line[pos] == ',':
            pos += 1
        else:
            break
    return fields


csv_lines = [
    'Alice,30,Bangkok',
    '"Smith, John",25,"New York"',
    '"He said ""Hello""",active,1234',
    'simple,,"empty middle"',
]

print(f"\n   {'Input':<45} {'Parsed'}")
print(f"   {'-'*45} {'-'*35}")
for line in csv_lines:
    parsed = parse_csv_line(line)
    print(f"   {line[:43]:<45} {parsed}")

print("\n\n2. TSV (Tab-Separated Values) parsing:")

TSV_SPLIT = re.compile(r'\t')

def parse_tsv(text: str, has_header: bool = True) -> List[Dict]:
    lines = text.strip().split('\n')
    if not lines:
        return []
    if has_header:
        headers = TSV_SPLIT.split(lines[0])
        rows = []
        for line in lines[1:]:
            values = TSV_SPLIT.split(line)
            rows.append(dict(zip(headers, values)))
        return rows
    else:
        return [TSV_SPLIT.split(line) for line in lines]


tsv_data = """name\tage\tcity
Alice\t30\tBangkok
Bob\t25\tChiang Mai
Carol\t35\tPhuket"""

rows = parse_tsv(tsv_data)
print(f"\n   Parsed TSV ({len(rows)} rows):")
for row in rows:
    print(f"   {row}")
```

---

## 57.2 Log ETL Pipeline

```python
import re
from collections import defaultdict, Counter
from typing import List, Dict, Optional

print("\nLog ETL Pipeline:")
print("=" * 60)

print("\n1. Extract → Transform → Load pattern:")

LOG_RE = re.compile(
    r'(?P<date>\d{4}-\d{2}-\d{2})\s+'
    r'(?P<time>\d{2}:\d{2}:\d{2})\s+'
    r'(?P<level>DEBUG|INFO|WARNING|ERROR|CRITICAL)\s+'
    r'\[(?P<module>[^\]]+)\]\s+'
    r'(?P<message>.+)'
)

def extract_log_line(line: str) -> Optional[Dict]:
    m = LOG_RE.match(line.strip())
    if not m:
        return None
    d = m.groupdict()
    d['datetime'] = f"{d['date']} {d['time']}"
    return d


ERROR_CODE = re.compile(r'Error code:\s*(\d+)', re.IGNORECASE)
DURATION   = re.compile(r'(\d+(?:\.\d+)?)\s*ms')
USER_ID    = re.compile(r'user_id[=:]?\s*(\d+)', re.IGNORECASE)
REQUEST_ID = re.compile(r'request_id[=:]?\s*([\w-]+)', re.IGNORECASE)

def transform_log(record: Dict) -> Dict:
    msg = record['message']
    ec = ERROR_CODE.search(msg)
    if ec:
        record['error_code'] = int(ec.group(1))
    dur = DURATION.search(msg)
    if dur:
        record['duration_ms'] = float(dur.group(1))
    uid = USER_ID.search(msg)
    if uid:
        record['user_id'] = int(uid.group(1))
    rid = REQUEST_ID.search(msg)
    if rid:
        record['request_id'] = rid.group(1)
    record['is_error'] = record['level'] in ('ERROR', 'CRITICAL')
    return record


def load_to_store(records: List[Dict]) -> Dict:
    store = {
        'by_level':   Counter(),
        'by_module':  Counter(),
        'errors':     [],
        'slow_calls': [],
        'total':      0,
    }
    for r in records:
        store['total'] += 1
        store['by_level'][r['level']]   += 1
        store['by_module'][r['module']] += 1
        if r.get('is_error'):
            store['errors'].append(r)
        if r.get('duration_ms', 0) > 100:
            store['slow_calls'].append(r)
    return store


log_lines = [
    "2024-10-15 09:00:01 INFO  [api.users] GET /users user_id=42 request_id=abc123 duration: 45ms",
    "2024-10-15 09:00:02 ERROR [db.conn] Connection failed Error code: 1045 user_id=42",
    "2024-10-15 09:00:03 INFO  [api.auth] Login successful user_id=99 45ms",
    "2024-10-15 09:00:04 WARNING [api.rate] Rate limit user_id=42 request_id=def456",
    "2024-10-15 09:00:05 INFO  [api.users] GET /users/42 250.5ms request_id=ghi789",
    "2024-10-15 09:00:06 CRITICAL [db.conn] DB unreachable Error code: 2003",
    "2024-10-15 09:00:07 INFO  [api.products] GET /products 12ms",
    "2024-10-15 09:00:08 ERROR [api.users] Validation failed Error code: 400 user_id=15",
    "not a log line - should be skipped",
]

extracted   = [r for line in log_lines if (r := extract_log_line(line))]
transformed = [transform_log(r) for r in extracted]
store       = load_to_store(transformed)

print(f"\n   ETL Results:")
print(f"   Total records: {store['total']}/{len(log_lines)} (1 skipped)")
print(f"   By level:  {dict(store['by_level'])}")
print(f"   By module: {dict(store['by_module'])}")
print(f"\n   Errors ({len(store['errors'])}):")
for e in store['errors']:
    code = e.get('error_code', 'N/A')
    print(f"   [{e['level']}] {e['module']}: Error {code} — {e['message'][:50]}")
print(f"\n   Slow calls (>100ms) ({len(store['slow_calls'])}):")
for s in store['slow_calls']:
    print(f"   {s['module']}: {s['duration_ms']}ms — {s['message'][:50]}")
```

---

## 57.3 Config File Parsing

```python
import re
from typing import Dict, Any

print("\nConfig File Parsing:")
print("=" * 60)

print("\n1. INI-style config parser:")

INI_SECTION  = re.compile(r'^\[([^\]]+)\]$')
INI_KEYVAL   = re.compile(r'^(\w[\w.-]*)\s*[=:]\s*(.*?)(?:\s*#.*)?$')
INI_COMMENT  = re.compile(r'^\s*[;#]')
INI_CONTINUE = re.compile(r'^\s+(.+)')

def parse_ini(text: str) -> Dict[str, Dict[str, str]]:
    config = {'DEFAULT': {}}
    section = 'DEFAULT'
    last_key = None

    for line in text.splitlines():
        if INI_COMMENT.match(line) or not line.strip():
            last_key = None
            continue

        m_section = INI_SECTION.match(line.strip())
        if m_section:
            section = m_section.group(1)
            if section not in config:
                config[section] = {}
            last_key = None
            continue

        m_kv = INI_KEYVAL.match(line)
        if m_kv:
            key, value = m_kv.group(1).lower(), m_kv.group(2).strip()
            if len(value) >= 2 and value[0] == value[-1] and value[0] in ('"', "'"):
                value = value[1:-1]
            config[section][key] = value
            last_key = key
            continue

        m_cont = INI_CONTINUE.match(line)
        if m_cont and last_key:
            config[section][last_key] += '\n' + m_cont.group(1).strip()

    return config


ini_text = """
# App configuration
[database]
host    = localhost
port    = 5432
name    = myapp_db
user    = dbuser
password = s3cr3t!

[server]
host = 0.0.0.0
port = 8000
debug = true

allowed_origins =
    https://example.com
    https://www.example.com

[logging]
level   = INFO
format  = %(asctime)s - %(name)s - %(levelname)s - %(message)s
"""

config = parse_ini(ini_text)
print(f"\n   Parsed sections: {list(config.keys())}")
for section, values in config.items():
    if values and section != 'DEFAULT':
        print(f"\n   [{section}]")
        for k, v in values.items():
            val_display = repr(v) if '\n' in v else v
            print(f"     {k} = {val_display}")


print("\n\n2. .env file parser:")

ENV_LINE = re.compile(
    r'^'
    r'(?:export\s+)?'
    r'([A-Z_][A-Z0-9_]*)'
    r'\s*=\s*'
    r'(?:'
    r'"((?:[^"\\]|\\.)*)"'
    r"|'([^']*)'"  
    r'|([^#\n]*)'
    r')'
    r'(?:\s*#.*)?$',
    re.MULTILINE
)

def parse_dotenv(text: str) -> Dict[str, str]:
    env = {}
    for m in ENV_LINE.finditer(text):
        key = m.group(1)
        value = next((g for g in [m.group(2), m.group(3), m.group(4)] if g is not None), '')
        if m.group(2) is not None:
            value = re.sub(
                r'\\([nrt"\\])',
                lambda x: {'n':'\n','r':'\r','t':'\t','"':'"','\\':'\\'}[x.group(1)],
                value
            )
        env[key] = value.strip()
    return env


env_text = """
# Database
DB_HOST=localhost
DB_PORT=5432
DB_NAME="myapp_db"
DB_USER='dbuser'
DB_PASSWORD=s3cr3t!

# Server
SERVER_HOST=0.0.0.0
SERVER_PORT=8000
DEBUG=true

JWT_SECRET="my-super-secret"
export API_KEY=abcdef123456
"""

env_vars = parse_dotenv(env_text)
print(f"\n   Parsed .env ({len(env_vars)} variables):")
for k, v in env_vars.items():
    print(f"   {k}={v!r}")
```

---

## 57.4 สรุป Part 57

```
Data Pipeline Patterns with Regex:

1. ETL Pattern:
   Extract:   parse raw text → dicts with named groups
   Transform: enrich/normalize dicts → structured records
   Load:      aggregate into Counter/list

2. CSV Parsing:
   Quoted fields:   "((?:[^"]|"")*)"| ([^,\n\r]*)
   Inner double-quote: "" → "
   Handle: embedded commas, empty fields

3. Log ETL:
   Extract: named groups → dict
   Transform: search message for codes, durations, IDs
   Load: Counter for aggregation, filter for errors

4. INI Config:
   Section:    ^\[([^\]]+)\]$
   Key-value:  ^(\w+)\s*[=:]\s*(.*?)(?:\s*#.*)?$
   Multi-line: continuation lines start with whitespace

5. .env Parser:
   Key:         [A-Z_][A-Z0-9_]*
   Double-quote: "..." with \" escape
   Single-quote: '...' (no escapes)
   Unquoted:    value up to # comment
```

---

*[← Part 56: String Processing Pipelines](part-56-string-pipelines.md) | [→ Part 58: Network & Protocol Parsing](part-58-network-protocols.md)*
