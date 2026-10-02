# Part 82: API Security Patterns

> **ระดับ:** ผู้เชี่ยวชาญ-ระดับโลก | **เวลาเรียน:** ~95 นาที | **ข้อกำหนด:** Part 01-81

---

## 82.1 REST API Input Validation

```python
import re
from typing import Dict, List

print("API Security Patterns:")
print("=" * 60)

print("\n1. REST API input validation patterns:")

# Resource ID validation
UUID_V4    = re.compile(r'^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$', re.IGNORECASE)
NUMERIC_ID = re.compile(r'^\d{1,20}$')
SLUG_ID    = re.compile(r'^[a-z0-9][a-z0-9\-]{0,62}[a-z0-9]$')
ULID       = re.compile(r'^[0-9A-Z]{26}$')

# HTTP method validation
HTTP_METHOD = re.compile(r'^(?:GET|POST|PUT|PATCH|DELETE|HEAD|OPTIONS)$')

# Content-Type validation
CONTENT_TYPE_JSON   = re.compile(r'^application/(?:json|vnd\.[^;]+\+json)(?:;\s*charset=utf-8)?$', re.IGNORECASE)
CONTENT_TYPE_FORM   = re.compile(r'^application/x-www-form-urlencoded(?:;\s*charset=utf-8)?$', re.IGNORECASE)
CONTENT_TYPE_MULTI  = re.compile(r'^multipart/form-data;\s*boundary=.+', re.IGNORECASE)

# Pagination parameter validation
PAGE_NUM  = re.compile(r'^\d{1,6}$')
PAGE_SIZE = re.compile(r'^(?:[1-9]|[1-9]\d|100)$')  # 1-100

# Sort/filter injection prevention
SORT_FIELD = re.compile(r'^[a-zA-Z][a-zA-Z0-9_]{0,49}$')
SORT_DIR   = re.compile(r'^(?:asc|desc)$', re.IGNORECASE)

# JSON field name safety (prevent prototype pollution)
PROTO_POLLUTE = re.compile(r'(?:__proto__|constructor|prototype)\s*(?:=|:)', re.IGNORECASE)


def validate_api_request(method: str, path: str, headers: Dict, params: Dict) -> List[str]:
    errors = []

    # Method check
    if not HTTP_METHOD.match(method):
        errors.append(f'Invalid HTTP method: {method!r}')

    # Content-Type for write methods
    if method in ('POST', 'PUT', 'PATCH'):
        ct = headers.get('Content-Type', '')
        if not (CONTENT_TYPE_JSON.match(ct) or CONTENT_TYPE_FORM.match(ct) or CONTENT_TYPE_MULTI.match(ct)):
            errors.append(f'Invalid Content-Type for {method}: {ct!r}')

    # Pagination
    if 'page' in params and not PAGE_NUM.match(str(params['page'])):
        errors.append(f"Invalid page number: {params['page']!r}")
    if 'limit' in params and not PAGE_SIZE.match(str(params['limit'])):
        errors.append(f"Invalid page size: {params['limit']!r} (must be 1-100)")

    # Sort safety
    if 'sort' in params and not SORT_FIELD.match(str(params['sort'])):
        errors.append(f"Invalid sort field: {params['sort']!r}")
    if 'order' in params and not SORT_DIR.match(str(params['order'])):
        errors.append(f"Invalid sort direction: {params['order']!r}")

    # Prototype pollution
    for k, v in params.items():
        if PROTO_POLLUTE.search(f'{k}={v}'):
            errors.append(f'Prototype pollution attempt: {k!r}')

    return errors


api_requests = [
    ('GET',    '/api/users',        {'Accept': 'application/json'},          {'page': '1', 'limit': '20', 'sort': 'name', 'order': 'asc'}),
    ('POST',   '/api/users',        {'Content-Type': 'application/json'},    {'page': '1'}),
    ('DELETE', '/api/users/1',      {'Authorization': 'Bearer token'},        {}),
    ('GET',    '/api/items',        {'Accept': 'application/json'},           {'limit': '999'}),
    ('POST',   '/api/data',         {'Content-Type': 'text/plain'},           {}),
    ('GET',    '/api/users',        {'Accept': 'application/json'},           {'sort': '__proto__', 'order': 'asc'}),
    ('PUT',    '/api/items/abc',    {'Content-Type': 'application/json'},     {'order': 'DROP TABLE'}),
]

print(f"\n   API request validation:")
for method, path, headers, params in api_requests:
    errors = validate_api_request(method, path, headers, params)
    params_str = ', '.join(f'{k}={v}' for k, v in params.items())[:40]
    if errors:
        print(f"\n   ⚠ {method} {path} [{params_str}]")
        for e in errors:
            print(f"     → {e}")
    else:
        print(f"   ✓ {method} {path} [{params_str}]")
```

---

## 82.2 GraphQL Security Patterns

```python
import re
from typing import Dict, List

print("\nGraphQL Security Patterns:")
print("=" * 60)

# GraphQL introspection queries (should be disabled in production)
GQL_INTROSPECTION = re.compile(
    r'__schema\s*\{|__type\s*\(|__typename\s*\{',
    re.IGNORECASE
)

# GraphQL batching attack (many queries in one request)
GQL_BATCH_COUNT = re.compile(r'^\s*\[', re.MULTILINE)

# Field count (deep nesting / field limit)
def count_gql_fields(query: str) -> int:
    return len(re.findall(r'\b\w+\s*(?:\([^)]*\))?\s*\{', query))

# Depth counting via brace nesting
def max_gql_depth(query: str) -> int:
    depth = max_depth = 0
    for ch in query:
        if ch == '{':
            depth += 1
            max_depth = max(max_depth, depth)
        elif ch == '}':
            depth -= 1
    return max_depth

# SQL-like injection in GQL arguments
GQL_SQLI = re.compile(
    r'(?:\'|")\s*(?:OR|AND|UNION)\s+',
    re.IGNORECASE
)

# N+1 query indicator (list field with subfields)
GQL_N_PLUS_1 = re.compile(
    r'(?:users|posts|items|products|orders)\s*(?:\([^)]*\))?\s*\{'
    r'[^}]*(?:posts|comments|orders|items|products|friends)\s*\{',
    re.IGNORECASE | re.DOTALL
)


def audit_graphql_query(query: str) -> Dict:
    issues = []
    depth  = max_gql_depth(query)
    fields = count_gql_fields(query)

    if GQL_INTROSPECTION.search(query):
        issues.append('introspection query (should be disabled in prod)')
    if depth > 10:
        issues.append(f'excessive depth: {depth} (max recommended: 10)')
    if fields > 50:
        issues.append(f'excessive fields: {fields} (max recommended: 50)')
    if GQL_SQLI.search(query):
        issues.append('possible injection in arguments')
    if GQL_N_PLUS_1.search(query):
        issues.append('potential N+1 query pattern')

    return {
        'depth':   depth,
        'fields':  fields,
        'issues':  issues,
        'allowed': len(issues) == 0,
    }


gql_queries = [
    'query { users { id name email } }',
    'query { __schema { types { name } } }',
    '''query {
        users {
            posts {
                comments {
                    author {
                        posts {
                            comments {
                                author {
                                    name
                                }
                            }
                        }
                    }
                }
            }
        }
    }''',
    'query { user(id: "1 OR 1=1") { name } }',
    'query { users { id posts { id comments { id author { id posts { id } } } } } }',
]

print(f"\n   GraphQL security audit:")
for q in gql_queries:
    result = audit_graphql_query(q)
    short_q = q.replace('\n', ' ').replace('  ', ' ')[:60]
    if result['issues']:
        print(f"\n   ⚠ {short_q!r}")
        print(f"     depth={result['depth']}, fields={result['fields']}")
        for issue in result['issues']:
            print(f"     → {issue}")
    else:
        print(f"   ✓ {short_q!r} (depth={result['depth']})")
```

---

## 82.3 API Rate Limit & Abuse Detection

```python
import re
import time
from collections import defaultdict, deque
from typing import Dict

print("\nAPI Rate Limit & Abuse Detection:")
print("=" * 60)

# API key format validation
VALID_API_KEY = re.compile(r'^[A-Za-z0-9_\-]{20,64}$')
BEARER_TOKEN  = re.compile(r'^Bearer\s+([A-Za-z0-9._\-]+)$')
BASIC_AUTH    = re.compile(r'^Basic\s+([A-Za-z0-9+/]+=*)$')

# Authorization header parser
def parse_auth_header(auth_header: str) -> Dict:
    m_bearer = BEARER_TOKEN.match(auth_header)
    if m_bearer:
        return {'type': 'bearer', 'value': m_bearer.group(1)}

    m_basic = BASIC_AUTH.match(auth_header)
    if m_basic:
        import base64
        try:
            decoded = base64.b64decode(m_basic.group(1)).decode('utf-8')
            user, _, pwd = decoded.partition(':')
            return {'type': 'basic', 'user': user, 'password_len': len(pwd)}
        except Exception:
            return {'type': 'basic', 'error': 'invalid base64'}

    if auth_header.startswith('ApiKey '):
        key = auth_header[7:]
        return {'type': 'apikey', 'value': key, 'valid': bool(VALID_API_KEY.match(key))}

    return {'type': 'unknown', 'raw': auth_header[:40]}


class APIAbuseDetector:
    def __init__(self):
        self.ip_requests  = defaultdict(deque)   # ip -> timestamps
        self.ip_endpoints = defaultdict(set)      # ip -> set of endpoints
        self.ip_users     = defaultdict(set)      # ip -> set of usernames
        self.ip_errors    = defaultdict(int)      # ip -> 401/403 count

    def record(self, ip: str, endpoint: str, user: str, status: int, now: float):
        dq = self.ip_requests[ip]
        dq.append(now)
        while dq and dq[0] < now - 60:
            dq.popleft()

        self.ip_endpoints[ip].add(endpoint)
        if user:
            self.ip_users[ip].add(user)
        if status in (401, 403):
            self.ip_errors[ip] += 1

    def get_risk(self, ip: str) -> Dict:
        req_count    = len(self.ip_requests[ip])
        endpoint_cnt = len(self.ip_endpoints[ip])
        user_cnt     = len(self.ip_users[ip])
        error_cnt    = self.ip_errors[ip]

        score = 0
        reasons = []

        if req_count > 100:
            score += 3
            reasons.append(f'high req rate: {req_count}/60s')
        if endpoint_cnt > 20:
            score += 2
            reasons.append(f'endpoint scan: {endpoint_cnt} unique endpoints')
        if user_cnt > 5:
            score += 4
            reasons.append(f'credential stuffing: {user_cnt} different usernames')
        if error_cnt > 10:
            score += 3
            reasons.append(f'many auth errors: {error_cnt}')

        return {'ip': ip, 'score': score, 'reasons': reasons,
                'action': 'BLOCK' if score >= 7 else ('WARN' if score >= 3 else 'ALLOW')}


detector = APIAbuseDetector()

import time
now = time.monotonic()
simulated_requests = [
    ('10.0.0.1', '/api/v1/users/me', 'alice', 200, 0.1),
    ('10.0.0.1', '/api/v1/posts',    'alice', 200, 0.2),
    ('10.0.0.5', '/api/v1/login', 'alice', 401, 1.0),
    ('10.0.0.5', '/api/v1/login', 'bob',   401, 1.1),
    ('10.0.0.5', '/api/v1/login', 'carol', 401, 1.2),
    ('10.0.0.5', '/api/v1/login', 'dave',  401, 1.3),
    ('10.0.0.5', '/api/v1/login', 'eve',   401, 1.4),
    ('10.0.0.5', '/api/v1/login', 'frank', 200, 1.5),
    ('10.0.0.9', '/api/v1/users',   '', 200, 2.0),
    ('10.0.0.9', '/api/v1/admin',   '', 403, 2.1),
    ('10.0.0.9', '/api/v1/config',  '', 403, 2.2),
    ('10.0.0.9', '/api/v1/debug',   '', 404, 2.3),
    ('10.0.0.9', '/api/v1/health',  '', 200, 2.4),
    ('10.0.0.9', '/api/v1/metrics', '', 403, 2.5),
]

for ip, endpoint, user, status, offset in simulated_requests:
    detector.record(ip, endpoint, user, status, now + offset)

print(f"\n   API abuse detection:")
for ip in ['10.0.0.1', '10.0.0.5', '10.0.0.9']:
    risk = detector.get_risk(ip)
    action = risk['action']
    marker = '🚨' if action == 'BLOCK' else ('⚠' if action == 'WARN' else '✓')
    print(f"\n   {marker} [{action}] {ip} (score={risk['score']})")
    for r in risk['reasons']:
        print(f"     → {r}")
```

---

## 82.4 สรุป Part 82

```
API Security Pattern Design:

1. REST API validation:
   Resource IDs: UUID v4, numeric, slug, ULID — validate format strictly
   Content-Type: application/json for POST/PUT/PATCH (strict)
   Pagination: page=1-999999, limit=1-100 (prevent DoS via large pages)
   Sort/filter: allowlist field names, asc/desc only
   Prototype pollution: block __proto__, constructor, prototype keys

2. GraphQL security:
   Depth limit: max 10 levels of nesting (prevent deep recursion)
   Field limit: max 50 fields per query (prevent overloading)
   Introspection: disable in production (__schema, __type)
   Batching: limit array queries (prevent 100x amplification)
   N+1: detect list+sublist patterns, use DataLoader

3. Authentication patterns:
   Bearer: JWT in Authorization header (validate algorithm+claims)
   Basic: base64(user:password) — only over HTTPS
   ApiKey: 20-64 alphanumeric chars — rotate regularly
   API key in header preferred over query string (appears in logs)

4. API abuse detection:
   Credential stuffing: > 5 different usernames from same IP
   Brute force: > 10 failed auth from same IP in 60s
   Scanner: > 20 different endpoints from same IP
   Rate: > 100 requests/60s from same IP
   Block: score >= 7, Warn: score >= 3

5. Defense:
   Validate ALL inputs at API boundary (schema validation)
   Use allowlists not blocklists for field names
   Rate limit at API gateway (not just app layer)
   Log all 4xx/5xx with full context for SIEM
```

---

*[← Part 81: Cloud Security Patterns](part-81-cloud-security.md) | [→ Part 83: Binary & Hex Patterns](part-83-binary-hex.md)*
