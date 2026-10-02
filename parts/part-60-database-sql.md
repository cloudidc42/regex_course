# Part 60: Database & SQL Patterns with Regex

> **ระดับ:** สูง | **เวลาเรียน:** ~75 นาที | **ข้อกำหนด:** Part 01-59

---

## 60.1 SQL Query Analysis

```python
import re
from typing import Dict, List, Optional

print("SQL Query Analysis with Regex:")
print("=" * 60)

SQL_TYPE = re.compile(
    r'^\s*(?P<type>SELECT|INSERT|UPDATE|DELETE|CREATE|DROP|ALTER|'
    r'TRUNCATE|GRANT|REVOKE|BEGIN|COMMIT|ROLLBACK|EXPLAIN|WITH)',
    re.IGNORECASE
)

SQL_TABLE_RE = re.compile(
    r'(?:FROM|JOIN|INTO|UPDATE|TABLE|'
    r'ALTER\s+TABLE|DROP\s+TABLE|TRUNCATE\s+TABLE|CREATE\s+(?:TEMP(?:ORARY)?\s+)?TABLE'
    r')\s+(?:`?"?\'?)([\w.]+)(?:`?"?\'?)',
    re.IGNORECASE
)

SQL_COLUMNS = re.compile(r'SELECT\s+(.+?)\s+FROM', re.IGNORECASE | re.DOTALL)
SQL_LIMIT   = re.compile(r'\bLIMIT\s+(\d+)(?:\s+OFFSET\s+(\d+))?', re.IGNORECASE)


def analyze_sql(query: str) -> Dict:
    result = {'type': None, 'tables': [], 'columns': None, 'has_where': False,
              'limit': None, 'offset': None}
    m = SQL_TYPE.match(query)
    if m:
        result['type'] = m.group('type').upper()
    result['tables'] = list(set(SQL_TABLE_RE.findall(query)))
    m = SQL_COLUMNS.search(query)
    if m:
        result['columns'] = [c.strip() for c in m.group(1).strip().split(',')]
    result['has_where'] = bool(re.search(r'\bWHERE\b', query, re.IGNORECASE))
    m = SQL_LIMIT.search(query)
    if m:
        result['limit']  = int(m.group(1))
        result['offset'] = int(m.group(2)) if m.group(2) else 0
    return result


queries = [
    "SELECT id, name, email FROM users WHERE active = 1 ORDER BY name LIMIT 10 OFFSET 20",
    "INSERT INTO orders (user_id, product_id, quantity) VALUES (1, 42, 3)",
    "UPDATE users SET email = 'new@example.com' WHERE id = 5",
    "DELETE FROM sessions WHERE expires_at < NOW()",
    "SELECT u.name, COUNT(o.id) FROM users u LEFT JOIN orders o ON u.id = o.user_id GROUP BY u.id",
    "CREATE TABLE products (id INT PRIMARY KEY, name VARCHAR(255))",
]

for q in queries:
    info = analyze_sql(q)
    print(f"\n   {q[:65]}...")
    print(f"   Type:    {info['type']}")
    print(f"   Tables:  {info['tables']}")
    if info['columns']:
        cols_display = info['columns'][:3]
        print(f"   Columns: {cols_display}{'...' if len(info['columns']) > 3 else ''}")
    if info['limit'] is not None:
        print(f"   Limit:   {info['limit']}, Offset: {info['offset']}")
```

---

## 60.2 SQL Query Builder

```python
import re
from typing import Dict, List, Any

print("\nSQL Query Builder with Regex Validation:")
print("=" * 60)

SAFE_IDENTIFIER = re.compile(r'^[a-zA-Z_][a-zA-Z0-9_.]*$')
SAFE_OPERATOR   = re.compile(
    r'^(?:=|!=|<>|<|>|<=|>=|LIKE|NOT\s+LIKE|IN|NOT\s+IN|IS\s+NULL|IS\s+NOT\s+NULL|BETWEEN)$',
    re.IGNORECASE
)
SAFE_DIRECTION  = re.compile(r'^(?:ASC|DESC)$', re.IGNORECASE)


class QueryBuilder:

    def __init__(self, table: str):
        if not SAFE_IDENTIFIER.match(table):
            raise ValueError(f"Invalid table name: {table!r}")
        self._table   = table
        self._columns = ['*']
        self._where   = []
        self._order   = []
        self._limit   = None
        self._offset  = None
        self._params  = []

    def select(self, *columns: str) -> 'QueryBuilder':
        for col in columns:
            if not SAFE_IDENTIFIER.match(col.strip()):
                raise ValueError(f"Invalid column: {col!r}")
        self._columns = list(columns)
        return self

    def where(self, column: str, operator: str, value: Any = None) -> 'QueryBuilder':
        if not SAFE_IDENTIFIER.match(column):
            raise ValueError(f"Invalid column: {column!r}")
        if not SAFE_OPERATOR.match(operator.strip()):
            raise ValueError(f"Invalid operator: {operator!r}")
        self._where.append(f"{column} {operator.upper()} ?")
        if value is not None:
            self._params.append(value)
        return self

    def order_by(self, column: str, direction: str = 'ASC') -> 'QueryBuilder':
        if not SAFE_IDENTIFIER.match(column):
            raise ValueError(f"Invalid column: {column!r}")
        if not SAFE_DIRECTION.match(direction):
            raise ValueError(f"Invalid direction: {direction!r}")
        self._order.append(f"{column} {direction.upper()}")
        return self

    def limit(self, n: int, offset: int = 0) -> 'QueryBuilder':
        self._limit  = n
        self._offset = offset
        return self

    def build(self):
        cols = ', '.join(self._columns)
        sql  = f"SELECT {cols} FROM {self._table}"
        if self._where:
            sql += ' WHERE ' + ' AND '.join(self._where)
        if self._order:
            sql += ' ORDER BY ' + ', '.join(self._order)
        if self._limit is not None:
            sql += f' LIMIT {self._limit}'
            if self._offset:
                sql += f' OFFSET {self._offset}'
        return sql, self._params


print("\n   Query builder examples:")
try:
    q, params = (
        QueryBuilder('users')
        .select('id', 'name', 'email')
        .where('active', '=', 1)
        .where('age', '>=', 18)
        .order_by('name', 'ASC')
        .limit(10, 0)
        .build()
    )
    print(f"\n   SQL:    {q}")
    print(f"   Params: {params}")
except ValueError as e:
    print(f"   Error: {e}")

print("\n   Injection prevention tests:")
test_cases = [
    ('table', 'users',                  True),
    ('table', "users; DROP TABLE--",    False),
    ('column', 'user_id',               True),
    ('column', "1 OR 1=1",              False),
    ('operator', '=',                   True),
    ('operator', '= 1; --',             False),
]
for kind, value, expected_ok in test_cases:
    if kind in ('table', 'column'):
        ok = bool(SAFE_IDENTIFIER.match(value))
    else:
        ok = bool(SAFE_OPERATOR.match(value))
    status = '✓ SAFE' if ok == expected_ok else '✗ FAIL'
    print(f"   {kind:<10} {value:<35} safe={ok} {status}")
```

---

## 60.3 Database Connection String Parsing

```python
import re
from typing import Dict

print("\nDatabase Connection String Parsing:")
print("=" * 60)

DB_URL = re.compile(
    r'^(?P<scheme>[a-zA-Z][a-zA-Z0-9+\-.]*)(?:\+(?P<driver>[a-zA-Z0-9]+))?://'
    r'(?:(?P<user>[^:@]+)(?::(?P<password>[^@]*))?@)?'
    r'(?P<host>[^/:?#]*)(?::(?P<port>\d+))?'
    r'(?:/(?P<database>[^?#]*))?'
    r'(?:\?(?P<options>[^#]*))?$'
)

DB_OPTIONS = re.compile(r'([^&=]+)=([^&]*)')
SQLITE_PATH = re.compile(r'^sqlite:///(?P<path>.+)$')


def parse_db_url(url: str) -> Dict:
    if url.startswith('sqlite'):
        m = SQLITE_PATH.match(url)
        if m:
            return {'scheme': 'sqlite', 'path': m.group('path')}
        return {'scheme': 'sqlite', 'path': ':memory:'}
    m = DB_URL.match(url)
    if not m:
        return {}
    d = m.groupdict()
    if d.get('port'):
        d['port'] = int(d['port'])
    if d.get('options'):
        d['options_parsed'] = {k: v for k, v in DB_OPTIONS.findall(d['options'])}
    return d


db_urls = [
    'postgresql://alice:secret@localhost:5432/mydb',
    'postgresql+asyncpg://user:pass@db.example.com:5432/prod?sslmode=require&pool_size=10',
    'mysql://root:pass@127.0.0.1:3306/testdb',
    'sqlite:///./data/app.db',
    'sqlite:///:memory:',
    'redis://localhost:6379/0',
]

print(f"\n   Connection string parsing:")
for url in db_urls:
    d = parse_db_url(url)
    if d:
        print(f"\n   {url[:65]}")
        for k in ['scheme', 'driver', 'user', 'host', 'port', 'database', 'path']:
            if d.get(k):
                print(f"     {k:<12}: {d[k]}")
        if 'password' in d and d['password']:
            print(f"     password    : ***")
        if d.get('options_parsed'):
            print(f"     options     : {d['options_parsed']}")
```

---

## 60.4 Migration File Parsing

```python
import re
from typing import List, Dict

print("\nDatabase Migration Parsing:")
print("=" * 60)

MIGRATION_DEP = re.compile(
    r'\(\'(?P<app>[a-zA-Z_]\w*)\',\s*\'(?P<migration>[a-zA-Z0-9_]+)\'\)',
)
MIGRATION_OP = re.compile(
    r'migrations\.(?P<op>CreateModel|DeleteModel|AddField|RemoveField|'
    r'AlterField|RenameField|RenameModel|AddIndex|RemoveIndex|RunSQL)\(',
)
FIELD_DEF = re.compile(
    r"'(?P<name>\w+)',\s*models\.(?P<type>\w+Field)\((?P<args>[^)]*)\)"
)


def parse_migration(source: str) -> Dict:
    deps   = [f"{m.group('app')}.{m.group('migration')}" for m in MIGRATION_DEP.finditer(source)]
    ops    = MIGRATION_OP.findall(source)
    fields = [{'name': m.group('name'), 'type': m.group('type'), 'args': m.group('args')[:40]}
              for m in FIELD_DEF.finditer(source)]
    return {'dependencies': deps, 'operations': ops, 'fields': fields}


migration_source = """
class Migration(migrations.Migration):
    dependencies = [
        ('auth', '0001_initial'),
        ('myapp', '0003_auto_20241015_0900'),
    ]
    operations = [
        migrations.CreateModel(
            name='Product',
            fields=[
                ('id', models.AutoField(primary_key=True)),
                ('name', models.CharField(max_length=255)),
                ('price', models.DecimalField(max_digits=10, decimal_places=2)),
                ('created_at', models.DateTimeField(auto_now_add=True)),
            ],
        ),
        migrations.AddField(
            model_name='order',
            name='discount',
            field=models.DecimalField(default=0, max_digits=5, decimal_places=2),
        ),
    ]
"""

info = parse_migration(migration_source)
print(f"\n   Dependencies: {info['dependencies']}")
print(f"   Operations:   {info['operations']}")
print(f"\n   Fields:")
for f in info['fields']:
    print(f"   {f['name']:<20} {f['type']:<20} {f['args']}")
```

---

## 60.5 สรุป Part 60

```
Database Regex Patterns:

1. SQL Analysis:
   Statement type: ^\s*(SELECT|INSERT|UPDATE|DELETE|...)
   Table names:    (?:FROM|JOIN|INTO)\s+(\w+)
   Column list:    SELECT\s+(.+?)\s+FROM

2. SQL Safety:
   Safe identifier: ^[a-zA-Z_][a-zA-Z0-9_.]*$
   Safe operator:   ^(?:=|!=|<|>|LIKE|IN|...)$
   Never interpolate user input into SQL!
   Always use parameterized queries.

3. Connection strings:
   scheme+driver://user:pass@host:port/database?options
   PostgreSQL: postgresql(+asyncpg)?://...
   SQLite:     sqlite:///path | sqlite:///:memory:

4. Migration parsing:
   Dependencies: ('[app]', '[migration]')
   Operations:   migrations.(CreateModel|AddField|...)

5. Defense principle:
   Regex validates names/operators (whitelist)
   Parameterized queries prevent injection
```

---

*[← Part 59: API Design Patterns](part-59-api-design.md) | [→ Part 61: Advanced Text Mining](part-61-text-mining.md)*
