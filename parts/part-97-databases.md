# Part 97: Regex in Database Systems

> **ระดับ:** ผู้เชี่ยวชาญ-ระดับโลก | **เวลาเรียน:** ~115 นาที | **ข้อกำหนด:** Part 01-96

---

## 97.1 SQL Pattern Matching with Python

```python
import re
import sqlite3
from typing import List, Dict, Optional, Tuple

print("Regex in Database Systems:")
print("=" * 60)

print("\n1. Python + SQLite with regex functions:")

# SQLite: register Python regex as a user-defined function
def sqlite_regexp(pattern: str, value: str) -> bool:
    if value is None:
        return False
    try:
        return bool(re.search(pattern, str(value), re.IGNORECASE))
    except re.error:
        return False


# Build in-memory database for demonstration
conn = sqlite3.connect(':memory:')
conn.create_function('REGEXP', 2, sqlite_regexp)
cur = conn.cursor()

cur.executescript("""
CREATE TABLE users (
    id      INTEGER PRIMARY KEY,
    email   TEXT,
    phone   TEXT,
    username TEXT,
    country TEXT
);

INSERT INTO users VALUES
    (1, 'alice@example.com',   '+1-555-123-4567', 'alice_dev',    'US'),
    (2, 'bob.smith@corp.org',  '555.987.6543',    'bob.smith',    'US'),
    (3, 'charlie@sub.io',      '+66-81-234-5678', 'charlie99',    'TH'),
    (4, 'invalid-email',       'not-a-phone',     'ok_user',      'UK'),
    (5, 'diana@example.com',   '+44 20 7946 0958','diana_2024',   'UK'),
    (6, 'eve@test.co.th',      '02-345-6789',     'eve!user',     'TH');
""")


# Query using REGEXP
print("\n   Rows with valid email addresses:")
cur.execute("""
    SELECT id, email FROM users
    WHERE email REGEXP '^[a-zA-Z0-9._%+\\-]+@[a-zA-Z0-9.\\-]+\\.[a-zA-Z]{2,}$'
""")
for row in cur.fetchall():
    print(f"   id={row[0]}: {row[1]!r}")


print("\n   US phone numbers:")
cur.execute("""
    SELECT id, phone FROM users
    WHERE phone REGEXP '^\\+?1[\\s\\-]?\\(?\\d{3}\\)?[\\s.\\-]\\d{3}[\\s.\\-]\\d{4}$'
       OR phone REGEXP '^\\d{3}[.\\-]\\d{3}[.\\-]\\d{4}$'
""")
for row in cur.fetchall():
    print(f"   id={row[0]}: {row[1]!r}")


print("\n   Invalid usernames (contain non-word chars except underscore):")
cur.execute("""
    SELECT id, username FROM users
    WHERE username NOT REGEXP '^[a-zA-Z0-9_]{3,20}$'
""")
for row in cur.fetchall():
    print(f"   id={row[0]}: {row[1]!r}")

conn.close()
```

---

## 97.2 Schema Validation Patterns

```python
import re
import sqlite3
from typing import Dict, List, Tuple, Optional
from dataclasses import dataclass, field

print("\nDatabase Schema Validation with Regex:")
print("=" * 60)


@dataclass
class ColumnConstraint:
    name: str
    pattern: re.Pattern
    error_msg: str
    nullable: bool = True


# Column-level regex constraints for a user registration table
COLUMN_CONSTRAINTS: Dict[str, ColumnConstraint] = {
    'email': ColumnConstraint(
        name='email',
        pattern=re.compile(r'^[a-zA-Z0-9._%+\-]{1,64}@[a-zA-Z0-9.\-]{1,253}\.[a-zA-Z]{2,63}$'),
        error_msg='Invalid email format',
        nullable=False,
    ),
    'username': ColumnConstraint(
        name='username',
        pattern=re.compile(r'^[a-zA-Z][a-zA-Z0-9_]{2,29}$'),
        error_msg='Username: 3-30 chars, start with letter, letters/digits/underscore only',
        nullable=False,
    ),
    'phone': ColumnConstraint(
        name='phone',
        pattern=re.compile(r'^\+\d{7,15}$'),
        error_msg='Phone must be E.164 format: +CCXXXXXXXXXX',
        nullable=True,
    ),
    'zip_code': ColumnConstraint(
        name='zip_code',
        pattern=re.compile(r'^\d{5}(?:-\d{4})?$'),
        error_msg='ZIP code must be 5 or 9 digits (NNNNN or NNNNN-NNNN)',
        nullable=True,
    ),
    'birth_date': ColumnConstraint(
        name='birth_date',
        pattern=re.compile(r'^\d{4}-(?:0[1-9]|1[0-2])-(?:0[1-9]|[12]\d|3[01])$'),
        error_msg='Birth date must be YYYY-MM-DD format',
        nullable=True,
    ),
}


def validate_row(row: Dict[str, Optional[str]]) -> List[str]:
    errors = []
    for col_name, constraint in COLUMN_CONSTRAINTS.items():
        val = row.get(col_name)
        if val is None:
            if not constraint.nullable:
                errors.append(f"{col_name}: required (cannot be null)")
        else:
            if not constraint.pattern.match(str(val)):
                errors.append(f"{col_name}: {constraint.error_msg} (got {val!r})")
    return errors


test_rows = [
    {'email': 'alice@example.com', 'username': 'alice_dev', 'phone': '+15551234567',
     'zip_code': '90210', 'birth_date': '1990-06-15'},

    {'email': 'invalid-email',     'username': '1bad_start', 'phone': '555-1234',
     'zip_code': '1234',  'birth_date': '1990-13-40'},

    {'email': 'bob@test.org',      'username': 'bob',  'phone': None,
     'zip_code': '12345-6789', 'birth_date': None},

    {'email': None, 'username': None, 'phone': '+66812345678',
     'zip_code': None, 'birth_date': '2000-02-29'},
]

print("\n   Row validation results:")
for i, row in enumerate(test_rows, 1):
    errors = validate_row(row)
    status = 'PASS' if not errors else f'FAIL ({len(errors)} error(s))'
    print(f"\n   Row {i}: {status}")
    for err in errors:
        print(f"     - {err}")
```

---

## 97.3 Query Log Analysis

```python
import re
from typing import Dict, List, Optional, Tuple
from collections import Counter, defaultdict

print("\nDatabase Query Log Analysis:")
print("=" * 60)

# MySQL slow query log patterns
SLOW_QUERY_TIME  = re.compile(r'#\s+Query_time:\s+(?P<query_time>[\d.]+)\s+Lock_time:\s+(?P<lock_time>[\d.]+)\s+Rows_sent:\s+(?P<rows_sent>\d+)\s+Rows_examined:\s+(?P<rows_examined>\d+)')
SLOW_QUERY_SQL   = re.compile(r'^(?!#|use |SET )(SELECT|INSERT|UPDATE|DELETE|REPLACE|CREATE|DROP|ALTER|TRUNCATE).+', re.IGNORECASE | re.MULTILINE)
SLOW_QUERY_TIMESTAMP = re.compile(r'^#\s+Time:\s+(?P<ts>\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2})', re.MULTILINE)

# Table name extraction from SQL
TABLE_FROM     = re.compile(r'\bFROM\s+`?(?P<table>[a-zA-Z_][a-zA-Z0-9_]*)`?', re.IGNORECASE)
TABLE_JOIN     = re.compile(r'\bJOIN\s+`?(?P<table>[a-zA-Z_][a-zA-Z0-9_]*)`?', re.IGNORECASE)
TABLE_INTO     = re.compile(r'\bINTO\s+`?(?P<table>[a-zA-Z_][a-zA-Z0-9_]*)`?', re.IGNORECASE)
TABLE_UPDATE   = re.compile(r'\bUPDATE\s+`?(?P<table>[a-zA-Z_][a-zA-Z0-9_]*)`?', re.IGNORECASE)

# SQL injection pattern detection (defensive: for WAF/SIEM, not offensive)
SQLI_PATTERNS = [
    re.compile(r"'\s*(?:OR|AND)\s+'?\d+'?\s*=\s*'?\d+", re.IGNORECASE),
    re.compile(r"--\s*$|;\s*--", re.MULTILINE),
    re.compile(r'\bUNION\b.*\bSELECT\b', re.IGNORECASE | re.DOTALL),
    re.compile(r"\bEXEC(?:UTE)?\s*\(", re.IGNORECASE),
    re.compile(r'\bINFORMATION_SCHEMA\b', re.IGNORECASE),
    re.compile(r"0x[0-9a-fA-F]{4,}"),  # Hex encoding
    re.compile(r'\bSLEEP\s*\(\s*\d+', re.IGNORECASE),  # Time-based blind
    re.compile(r'\bWAITFOR\s+DELAY\b', re.IGNORECASE),
]

SQLI_NAMES = [
    'boolean_based', 'comment_terminator', 'union_select',
    'exec_function', 'information_schema', 'hex_encoding',
    'time_based_sleep', 'time_based_waitfor',
]


def extract_tables(sql: str) -> List[str]:
    tables = set()
    for pattern in (TABLE_FROM, TABLE_JOIN, TABLE_INTO, TABLE_UPDATE):
        for m in pattern.finditer(sql):
            tables.add(m.group('table').lower())
    return sorted(tables)


def detect_sqli(user_input: str) -> List[str]:
    """Detect potential SQL injection patterns in user input (defensive)."""
    findings = []
    for name, pattern in zip(SQLI_NAMES, SQLI_PATTERNS):
        if pattern.search(user_input):
            findings.append(name)
    return findings


sample_slow_log = """
# Time: 2024-01-15T10:30:01.000000Z
# User@Host: app[app] @ web01 []
# Query_time: 12.345678  Lock_time: 0.000234  Rows_sent: 50  Rows_examined: 1500000
SELECT u.id, u.email, o.total FROM users u JOIN orders o ON u.id = o.user_id WHERE o.created_at > '2024-01-01' ORDER BY o.total DESC LIMIT 50;

# Time: 2024-01-15T10:30:45.000000Z
# Query_time: 0.045000  Lock_time: 0.000012  Rows_sent: 1  Rows_examined: 8
SELECT * FROM products WHERE id = 42;

# Time: 2024-01-15T10:31:00.000000Z
# Query_time: 8.900000  Lock_time: 2.100000  Rows_sent: 200  Rows_examined: 4000000
UPDATE orders SET status = 'shipped' WHERE created_at < '2024-01-10';
"""

print("\n   Slow query log parsing:")
for m in SLOW_QUERY_TIME.finditer(sample_slow_log):
    qt = float(m.group('query_time'))
    re_count = int(m.group('rows_examined'))
    print(f"\n   Query_time={qt:.3f}s, Rows_examined={re_count:,}")
    if qt > 5.0:
        print(f"     ALERT: Very slow query ({qt:.1f}s)")
    if re_count > 1_000_000:
        print(f"     ALERT: Full scan candidate ({re_count:,} rows examined)")


# Table access frequency from queries
all_sql = SLOW_QUERY_SQL.findall(sample_slow_log)
table_counter: Counter = Counter()
for sql in all_sql:
    for table in extract_tables(sql):
        table_counter[table] += 1

print(f"\n   Table access frequency:")
for table, count in table_counter.most_common():
    print(f"   {table}: {count} queries")


# SQL injection detection samples
print(f"\n   SQL injection detection (defensive):")
suspicious_inputs = [
    "alice@example.com",
    "' OR '1'='1",
    "admin'--",
    "1 UNION SELECT null,null,username,password FROM users--",
    "1; EXEC xp_cmdshell('whoami')--",
    "SLEEP(5)-- ",
    "1 AND 1=1",
]

for inp in suspicious_inputs:
    findings = detect_sqli(inp)
    if findings:
        print(f"   SUSPICIOUS: {inp!r}")
        print(f"     Patterns: {', '.join(findings)}")
    else:
        print(f"   CLEAN:      {inp!r}")
```

---

## 97.4 Migration Script Analysis

```python
import re
from typing import Dict, List, Tuple

print("\nDatabase Migration Script Analysis:")
print("=" * 60)

# DDL statement patterns
DDL_CREATE_TABLE = re.compile(
    r'CREATE\s+(?:TABLE|INDEX|VIEW|SEQUENCE)\s+(?:IF\s+NOT\s+EXISTS\s+)?'
    r'`?(?P<name>[a-zA-Z_][a-zA-Z0-9_]*)`?',
    re.IGNORECASE
)
DDL_ALTER_TABLE = re.compile(
    r'ALTER\s+TABLE\s+`?(?P<table>[a-zA-Z_][a-zA-Z0-9_]*)`?\s+'
    r'(?P<action>ADD|DROP|MODIFY|RENAME|CHANGE)\s+(?:COLUMN\s+)?`?(?P<col>[a-zA-Z_][a-zA-Z0-9_]*)?`?',
    re.IGNORECASE
)
DDL_DROP = re.compile(
    r'DROP\s+(?P<type>TABLE|INDEX|VIEW|SEQUENCE)\s+(?:IF\s+EXISTS\s+)?`?(?P<name>[a-zA-Z_][a-zA-Z0-9_]*)`?',
    re.IGNORECASE
)

# Migration version numbers
MIGRATION_FILE = re.compile(r'^(?P<version>\d{4,14})_(?P<description>[a-z0-9_]+)\.(?:sql|py)$', re.IGNORECASE)


def analyze_migration(sql: str) -> Dict:
    result = {
        'creates':    [],
        'alters':     [],
        'drops':      [],
        'is_destructive': False,
    }

    for m in DDL_CREATE_TABLE.finditer(sql):
        result['creates'].append(m.group('name'))

    for m in DDL_ALTER_TABLE.finditer(sql):
        result['alters'].append({
            'table': m.group('table'),
            'action': m.group('action').upper(),
            'column': m.group('col'),
        })

    for m in DDL_DROP.finditer(sql):
        result['drops'].append({'type': m.group('type'), 'name': m.group('name')})
        result['is_destructive'] = True

    return result


sample_migration = """
-- Migration: 20240115_add_user_profile_fields
ALTER TABLE users ADD COLUMN avatar_url VARCHAR(500) NULL;
ALTER TABLE users ADD COLUMN bio TEXT NULL;
ALTER TABLE users MODIFY COLUMN email VARCHAR(320) NOT NULL;

CREATE TABLE IF NOT EXISTS user_sessions (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id     INT NOT NULL,
    token       VARCHAR(128) NOT NULL,
    created_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    expires_at  DATETIME NOT NULL,
    ip_address  VARCHAR(45) NULL,
    UNIQUE(token)
);

CREATE INDEX idx_sessions_user ON user_sessions(user_id);
CREATE INDEX idx_sessions_token ON user_sessions(token);

DROP TABLE IF EXISTS legacy_user_tokens;
"""

print("\n   Migration analysis:")
result = analyze_migration(sample_migration)
print(f"   Creates: {result['creates']}")
for alt in result['alters']:
    print(f"   Alter:   {alt['table']}.{alt['column']} [{alt['action']}]")
for drop in result['drops']:
    print(f"   Drop:    {drop['type']} {drop['name']}")
if result['is_destructive']:
    print("   WARNING: Migration contains destructive DDL (DROP)")


# Migration file naming validation
migration_files = [
    '20240115_add_user_profile.sql',
    '001_initial_schema.sql',
    '20240116000000_create_sessions.py',
    'invalid_migration.sql',
    '20240117_fix_indexes.SQL',
]

print(f"\n   Migration file name validation:")
for fname in migration_files:
    m = MIGRATION_FILE.match(fname)
    if m:
        print(f"   OK:  {fname!r} (version={m.group('version')}, desc={m.group('description')!r})")
    else:
        print(f"   BAD: {fname!r}")
```

---

## 97.5 สรุป Part 97

```
Regex in Database Systems:

1. SQLite + Python regex:
   conn.create_function('REGEXP', 2, lambda p,v: bool(re.search(p,v)))
   WHERE column REGEXP 'pattern' in SQL queries
   Use IGNORECASE in Python function for case-insensitive matching

2. Schema validation:
   Compile patterns at module level (not per-row)
   Validate at application layer before INSERT/UPDATE
   Distinguish nullable vs required with separate checks
   Column constraints: email, username, phone E.164, ZIP, date

3. Query log analysis:
   Parse slow query log: Query_time, Lock_time, Rows_examined
   Alert thresholds: query_time > 2s, rows_examined > 500k
   Extract table names with FROM/JOIN/INTO/UPDATE patterns
   Track table access frequency for index recommendations

4. SQL injection detection (defensive patterns):
   Boolean-based: ' OR '1'='1
   Comment terminator: -- or ;--
   UNION SELECT: stacked query exfiltration
   Time-based: SLEEP(n) / WAITFOR DELAY
   Always use parameterized queries to prevent injection
   Regex detection = WAF/SIEM layer, NOT the only defense

5. Migration analysis:
   Parse DDL: CREATE TABLE/INDEX/VIEW, ALTER TABLE, DROP
   Flag destructive operations (DROP) for review
   Validate migration file naming: version_description.sql
   Extract column definitions: name type(size) modifiers
```

---

*[← Part 96: Regex for Network Protocol Analysis](part-96-network-protocols.md) | [→ Part 98: Regex for Security & WAF Bypass Detection](part-98-security-waf.md)*
