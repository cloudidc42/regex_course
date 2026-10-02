# Part 28: Database & Query Patterns — รูปแบบฐานข้อมูล

> **ระดับ:** กลาง | **เวลาเรียน:** ~65 นาที | **ข้อกำหนด:** Part 01-27

---

## 28.1 SQL Query Analysis

```python
import re
from typing import Dict, List, Optional

class SQLAnalyzer:
    """วิเคราะห์ SQL queries ด้วย regex"""
    
    SELECT_STMT = re.compile(
        r'^\s*SELECT\s+(?P<cols>.*?)\s+FROM\s+(?P<table>\w+)'
        r'(?:\s+WHERE\s+(?P<where>.+?))?'
        r'(?:\s+ORDER\s+BY\s+(?P<order>.+?))?'
        r'(?:\s+LIMIT\s+(?P<limit>\d+))?'
        r'\s*;?\s*$',
        re.IGNORECASE | re.DOTALL
    )
    
    INSERT_STMT = re.compile(
        r'^\s*INSERT\s+INTO\s+(?P<table>\w+)'
        r'\s*\((?P<cols>[^)]+)\)'
        r'\s*VALUES\s*\((?P<values>[^)]+)\)',
        re.IGNORECASE
    )
    
    UPDATE_STMT = re.compile(
        r'^\s*UPDATE\s+(?P<table>\w+)'
        r'\s+SET\s+(?P<sets>.+?)'
        r'(?:\s+WHERE\s+(?P<where>.+?))?'
        r'\s*;?\s*$',
        re.IGNORECASE | re.DOTALL
    )
    
    DANGEROUS_PATTERNS = re.compile(
        r"(?:--|#|/\*|\*/|;\s*(?:DROP|DELETE|TRUNCATE|ALTER|CREATE)|"
        r"'\s*(?:OR|AND)\s+['\d]|UNION\s+(?:ALL\s+)?SELECT|"
        r"EXEC(?:UTE)?\s*\(?|INTO\s+(?:OUTFILE|DUMPFILE)|"
        r"LOAD_FILE\s*\()",
        re.IGNORECASE
    )
    
    IDENTIFIER = re.compile(r'`([^`]+)`|"([^"]+)"|([a-zA-Z_]\w*)')
    
    @classmethod
    def parse_select(cls, query: str) -> Optional[Dict]:
        m = cls.SELECT_STMT.match(query)
        if not m:
            return None
        
        cols_str = m.group('cols').strip()
        columns = [c.strip() for c in re.split(r',(?![^(]*\))', cols_str)]
        
        return {
            'type': 'SELECT',
            'table': m.group('table'),
            'columns': columns,
            'where': m.group('where'),
            'order_by': m.group('order'),
            'limit': int(m.group('limit')) if m.group('limit') else None,
        }
    
    @classmethod
    def detect_injection(cls, query: str) -> List[str]:
        threats = []
        for m in cls.DANGEROUS_PATTERNS.finditer(query):
            threats.append(m.group(0).strip())
        return threats
    
    @classmethod
    def extract_table_names(cls, query: str) -> List[str]:
        table_pattern = re.compile(
            r'(?:FROM|JOIN|INTO|UPDATE|TABLE)\s+([`"]?\w+[`"]?)',
            re.IGNORECASE
        )
        return [m.group(1).strip('`"') for m in table_pattern.finditer(query)]


# ทดสอบ
queries = [
    "SELECT id, name, email FROM users WHERE status='active' ORDER BY name LIMIT 10",
    "INSERT INTO orders (user_id, product, price) VALUES (123, 'Widget', 99.99)",
    "UPDATE users SET last_login=NOW() WHERE id=456",
    "SELECT * FROM users WHERE id=1 OR 1=1 --",
    "SELECT * FROM users UNION SELECT username, password FROM admin --",
]

analyzer = SQLAnalyzer

print("SQL Query Analysis:")
print("=" * 60)

for q in queries:
    print(f"\n  Query: {q[:70]}")
    
    parsed = analyzer.parse_select(q)
    if parsed:
        print(f"    Type:    {parsed['type']}")
        print(f"    Table:   {parsed['table']}")
        print(f"    Columns: {parsed['columns'][:3]}")
        if parsed['where']:
            print(f"    Where:   {parsed['where']}")
    
    threats = analyzer.detect_injection(q)
    if threats:
        print(f"    [THREAT] Injection detected: {threats}")
    
    tables = analyzer.extract_table_names(q)
    if tables:
        print(f"    Tables:  {tables}")
```

---

## 28.2 Database Connection String Parser

```python
import re
from typing import Dict, Optional

class ConnectionStringParser:
    """Parse database connection strings"""
    
    DSN = re.compile(
        r'^(?P<scheme>[a-z]+(?:\+[a-z]+)?)'
        r'://(?:(?P<user>[^:@]+)(?::(?P<password>[^@]+))?@)?'
        r'(?P<host>[^:/?#]+)(?::(?P<port>\d+))?'
        r'(?:/(?P<database>[^?]+))?'
        r'(?:\?(?P<params>.+))?$',
        re.IGNORECASE
    )
    
    SQLITE_FILE = re.compile(r'^sqlite:///(?P<path>.+)$', re.IGNORECASE)
    
    MONGO_DSN = re.compile(
        r'^mongodb(?:\+srv)?://'
        r'(?:(?P<user>[^:@]+)(?::(?P<password>[^@]+))?@)?'
        r'(?P<hosts>[^/?]+)'
        r'(?:/(?P<database>[^?]+))?'
        r'(?:\?(?P<options>.+))?$',
        re.IGNORECASE
    )
    
    @classmethod
    def parse(cls, dsn: str) -> Dict:
        m = cls.SQLITE_FILE.match(dsn)
        if m:
            return {'type': 'sqlite', 'path': m.group('path')}
        
        if dsn.startswith('mongodb'):
            m = cls.MONGO_DSN.match(dsn)
            if m:
                return {
                    'type': 'mongodb',
                    'user': m.group('user'),
                    'password': '***' if m.group('password') else None,
                    'hosts': m.group('hosts'),
                    'database': m.group('database'),
                }
        
        m = cls.DSN.match(dsn)
        if m:
            params = {}
            if m.group('params'):
                for kv in m.group('params').split('&'):
                    if '=' in kv:
                        k, v = kv.split('=', 1)
                        params[k] = v
            
            return {
                'type': m.group('scheme'),
                'user': m.group('user'),
                'password': '***' if m.group('password') else None,
                'host': m.group('host'),
                'port': int(m.group('port')) if m.group('port') else None,
                'database': m.group('database'),
                'params': params or None,
            }
        
        return {'type': 'unknown', 'raw': dsn}
    
    @classmethod
    def mask_password(cls, dsn: str) -> str:
        return re.sub(
            r'(://[^:@]+:)[^@]+(@)',
            r'\1***\2',
            dsn
        )


# ทดสอบ
connection_strings = [
    "postgresql://user:secret@localhost:5432/mydb",
    "mysql+pymysql://root:password@db.example.com:3306/appdb?charset=utf8",
    "sqlite:///./data/app.db",
    "mongodb://admin:pass123@mongo1:27017,mongo2:27017/mydb?replicaSet=rs0",
    "redis://localhost:6379/0",
]

print("Connection String Parser:")
print("=" * 60)
for cs in connection_strings:
    parsed = ConnectionStringParser.parse(cs)
    masked = ConnectionStringParser.mask_password(cs)
    print(f"\n  DSN:    {masked}")
    for k, v in parsed.items():
        if v is not None:
            print(f"    {k:10}: {v}")
```

---

## 28.3 Database Log Analysis

```python
import re
from typing import List, Dict
from collections import Counter

class DBLogParser:
    """Parse database log files"""
    
    PG_LOG = re.compile(
        r'(?P<timestamp>\d{4}-\d{2}-\d{2}\s+\d{2}:\d{2}:\d{2}\.\d+)\s+'
        r'(?P<tz>\w+)\s+\[(?P<pid>\d+)\]\s+'
        r'(?P<user>\w+)@(?P<db>\w+)\s+'
        r'(?P<level>ERROR|WARNING|LOG|FATAL|DETAIL|HINT|NOTICE):\s+'
        r'(?P<message>.+)'
    )
    
    @classmethod
    def parse_postgres_log(cls, log_text: str) -> List[Dict]:
        entries = []
        for m in cls.PG_LOG.finditer(log_text):
            entries.append({
                'timestamp': m.group('timestamp'),
                'pid': int(m.group('pid')),
                'user': m.group('user'),
                'database': m.group('db'),
                'level': m.group('level'),
                'message': m.group('message'),
            })
        return entries


# ทดสอบ
pg_log_sample = """2026-10-02 14:30:01.123 UTC [1234] app@mydb LOG:  duration: 145 ms  statement: SELECT * FROM users WHERE id=123
2026-10-02 14:30:02.456 UTC [1235] app@mydb ERROR:  duplicate key value violates unique constraint "users_email_key"
2026-10-02 14:30:03.789 UTC [1236] app@mydb WARNING:  there is no transaction in progress
2026-10-02 14:30:04.001 UTC [1234] app@mydb LOG:  duration: 2341 ms  statement: SELECT u.*, p.* FROM users u JOIN profiles p ON u.id=p.user_id
2026-10-02 14:30:05.234 UTC [1237] admin@mydb FATAL:  role "unknown_user" does not exist
"""

parser = DBLogParser
entries = parser.parse_postgres_log(pg_log_sample)

print("PostgreSQL Log Analysis:")
print("=" * 60)
print(f"\nTotal entries: {len(entries)}")

for e in entries:
    level_icon = {'ERROR': '✗', 'FATAL': '✗✗', 'WARNING': '⚠', 'LOG': '·'}.get(e['level'], '?')
    print(f"  {level_icon} [{e['level']:8}] {e['user']}@{e['database']}: {e['message'][:60]}")

from collections import Counter
levels = Counter(e['level'] for e in entries)
print(f"\nLevel breakdown: {dict(levels)}")
```

---

## 28.4 ORM Query Builder Pattern

```python
import re
from typing import List, Tuple, Any, Optional

class QueryBuilder:
    """Simple query builder with regex validation"""
    
    SAFE_IDENTIFIER = re.compile(r'^[a-zA-Z_][a-zA-Z0-9_.]*$')
    SAFE_OPERATOR   = re.compile(r'^(?:=|!=|<>|>|<|>=|<=|LIKE|NOT\s+LIKE|IN|NOT\s+IN|IS\s+NULL|IS\s+NOT\s+NULL)$', re.IGNORECASE)
    
    def __init__(self, table: str):
        if not self.SAFE_IDENTIFIER.match(table):
            raise ValueError(f"Invalid table name: {table}")
        self._table = table
        self._conditions = []
        self._columns = ['*']
        self._limit = None
        self._order = []
    
    def select(self, *columns) -> 'QueryBuilder':
        for col in columns:
            if not self.SAFE_IDENTIFIER.match(col) and col != '*':
                raise ValueError(f"Invalid column name: {col}")
        self._columns = list(columns) if columns else ['*']
        return self
    
    def where(self, column: str, operator: str, value: Any) -> 'QueryBuilder':
        if not self.SAFE_IDENTIFIER.match(column):
            raise ValueError(f"Invalid column: {column}")
        if not self.SAFE_OPERATOR.match(operator.strip()):
            raise ValueError(f"Invalid operator: {operator}")
        self._conditions.append((column, operator, value))
        return self
    
    def order_by(self, column: str, direction: str = 'ASC') -> 'QueryBuilder':
        if not self.SAFE_IDENTIFIER.match(column):
            raise ValueError(f"Invalid column: {column}")
        direction = direction.upper()
        if direction not in ('ASC', 'DESC'):
            raise ValueError(f"Invalid direction: {direction}")
        self._order.append(f"{column} {direction}")
        return self
    
    def limit(self, n: int) -> 'QueryBuilder':
        if not isinstance(n, int) or n < 0:
            raise ValueError(f"Invalid limit: {n}")
        self._limit = n
        return self
    
    def build(self) -> tuple:
        cols = ', '.join(self._columns)
        sql = f"SELECT {cols} FROM {self._table}"
        params = []
        
        if self._conditions:
            parts = []
            for col, op, val in self._conditions:
                parts.append(f"{col} {op} %s")
                params.append(val)
            sql += " WHERE " + " AND ".join(parts)
        
        if self._order:
            sql += " ORDER BY " + ", ".join(self._order)
        
        if self._limit is not None:
            sql += f" LIMIT {self._limit}"
        
        return sql, params


# ทดสอบ
print("Query Builder:")
print("=" * 60)

try:
    sql, params = (
        QueryBuilder("users")
        .select("id", "name", "email")
        .where("status", "=", "active")
        .where("age", ">=", 18)
        .order_by("name")
        .limit(10)
        .build()
    )
    print(f"\nSQL:    {sql}")
    print(f"Params: {params}")
except ValueError as e:
    print(f"Error: {e}")

try:
    bad = QueryBuilder("users; DROP TABLE users")
    print("ERROR: Should have raised ValueError!")
except ValueError as e:
    print(f"\nInjection prevented: {e}")

try:
    bad = QueryBuilder("users").where("id", "= 1 OR 1=1 --", "x")
    print("ERROR: Should have raised ValueError!")
except ValueError as e:
    print(f"Bad operator prevented: {e}")
```

---

## 28.5 สรุป Part 28

```
SQL Patterns:
SELECT:  ^\s*SELECT\s+(.+?)\s+FROM\s+(\w+)...
INSERT:  ^\s*INSERT\s+INTO\s+(\w+)\s*\(([^)]+)\)\s*VALUES\s*\(([^)]+)\)
UPDATE:  ^\s*UPDATE\s+(\w+)\s+SET\s+(.+?)(?:\s+WHERE\s+(.+?))?
DSN:     ^([a-z+]+)://(?:([^:@]+):([^@]+)@)?([^:/?#]+)...

Security:
- Identifier validation: ^[a-zA-Z_][a-zA-Z0-9_.]*$
- Operator whitelist: =|!=|<>|>|<|>=|<=|LIKE|IN...
- ใช้ parameterized queries เสมอ (? หรือ %s)
- Regex ตรวจสอบ format เท่านั้น ไม่ใช้ escape SQL

Connection strings:
postgresql://user:pass@host:5432/dbname
mysql+pymysql://user:pass@host/dbname?charset=utf8
sqlite:///./path/to/db.sqlite3
mongodb://user:pass@host1,host2/dbname?replicaSet=rs0
```

---

*[← Part 27: Email Validation](part-27-email-validation.md) | [→ Part 29: Network Patterns](part-29-network-patterns.md)*
