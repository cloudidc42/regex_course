# Part 59: API Design Patterns & Documentation Regex

> **ระดับ:** สูง | **เวลาเรียน:** ~70 นาที | **ข้อกำหนด:** Part 01-58

---

## 59.1 OpenAPI / Swagger Pattern Validation

```python
import re
from typing import Dict, List, Optional

print("API Design Pattern Validation:")
print("=" * 60)

print("\n1. OpenAPI path parameter validation:")

OPENAPI_PARAM = re.compile(r'\{([a-zA-Z][a-zA-Z0-9_]*)\}')
OPENAPI_PATH  = re.compile(r'^(?:/(?:[a-zA-Z0-9\-._~!$&\'()*+,;=@]|\{[a-zA-Z][a-zA-Z0-9_]*\})*)+$')

def parse_openapi_path(path: str) -> Dict:
    params    = OPENAPI_PARAM.findall(path)
    valid     = bool(OPENAPI_PATH.match(path))
    regex_str = OPENAPI_PARAM.sub(r'(?P<\\1>[^/]+)', path)
    regex_str = '^' + regex_str + '$'
    return {'path': path, 'valid': valid, 'params': params, 'regex': regex_str}


openapi_paths = [
    '/users',
    '/users/{userId}',
    '/users/{userId}/posts/{postId}',
    '/orgs/{orgId}/repos/{repoName}',
]

print(f"\n   {'Path':<45} {'Valid':<8} {'Params'}")
print(f"   {'-'*45} {'-'*8} {'-'*25}")
for path in openapi_paths:
    info = parse_openapi_path(path)
    print(f"   {path:<45} {str(info['valid']):<8} {info['params']}")

test_path = '/users/{userId}/posts/{postId}'
info = parse_openapi_path(test_path)
fix_re = re.sub(r'\(\?P<\\\\1>', lambda m: m.group().replace('\\\\1', 'userId'), info['regex'])

print(f"\n   Manual test of path matching:")
test_re = re.compile(r'^/users/(?P<userId>[^/]+)/posts/(?P<postId>[^/]+)$')
for req in ['/users/42/posts/7', '/users/alice/posts/hello', '/comments/1']:
    m = test_re.match(req)
    if m:
        print(f"   ✓ {req!r} → {m.groupdict()}")
    else:
        print(f"   ✗ {req!r} → no match")
```

---

## 59.2 GraphQL Query Parsing

```python
import re
from typing import Dict, List

print("\nGraphQL Query Parsing:")
print("=" * 60)

GQL_OPERATION = re.compile(
    r'(?P<type>query|mutation|subscription)\s+(?P<name>[a-zA-Z_]\w*)'
    r'(?:\s*\((?P<vars>[^)]*)\))?'
    r'\s*\{',
    re.IGNORECASE
)

GQL_FRAGMENT = re.compile(
    r'fragment\s+(?P<name>[a-zA-Z_]\w*)\s+on\s+(?P<type>[a-zA-Z_]\w*)\s*\{',
    re.IGNORECASE
)

GQL_VARIABLE = re.compile(r'\$(?P<name>[a-zA-Z_]\w*)\s*:\s*(?P<type>[^\s,)]+)')


def parse_graphql(query: str) -> Dict:
    result = {'operations': [], 'fragments': []}
    for m in GQL_OPERATION.finditer(query):
        op = m.groupdict()
        if op.get('vars'):
            vars_found = GQL_VARIABLE.findall(op['vars'])
            op['variables'] = [{'name': v[0], 'type': v[1]} for v in vars_found]
        else:
            op['variables'] = []
        result['operations'].append(op)
    for m in GQL_FRAGMENT.finditer(query):
        result['fragments'].append(m.groupdict())
    return result


gql_examples = [
    """query GetUser($userId: ID!, $withPosts: Boolean = false) {
        user(id: $userId) { id name email }
    }""",
    """mutation CreatePost($input: PostInput!) {
        createPost(input: $input) { id title slug }
    }""",
    """subscription OnMessage($channelId: ID!) {
        messageReceived(channelId: $channelId) { id content }
    }
    fragment UserFields on User {
        id name email
    }""",
]

for gql in gql_examples:
    parsed = parse_graphql(gql)
    for op in parsed['operations']:
        print(f"\n   {op['type'].upper()} {op['name']}")
        for v in op['variables']:
            print(f"     ${v['name']}: {v['type']}")
    for frag in parsed['fragments']:
        print(f"   FRAGMENT {frag['name']} on {frag['type']}")
```

---

## 59.3 API Documentation Extraction

```python
import re
from typing import Dict, List

print("\nAPI Documentation Extraction:")
print("=" * 60)

PARAM_DOC  = re.compile(r':param\s+(?:(?P<type>\w+)\s+)?(?P<name>\w+):\s*(?P<desc>[^\n]+)', re.MULTILINE)
RETURN_DOC = re.compile(r':returns?:\s*(?P<desc>[^\n]+)', re.MULTILINE)
RAISES_DOC = re.compile(r':raises?\s+(?P<exc>\w+):\s*(?P<desc>[^\n]+)', re.MULTILINE)
TYPE_DOC   = re.compile(r':type\s+(?P<name>\w+):\s*(?P<type>[^\n]+)', re.MULTILINE)


def extract_docstring_api(docstring: str) -> Dict:
    result = {'params': [], 'returns': [], 'raises': []}
    types = {m.group('name'): m.group('type').strip() for m in TYPE_DOC.finditer(docstring)}
    for m in PARAM_DOC.finditer(docstring):
        name = m.group('name')
        result['params'].append({
            'name': name,
            'type': m.group('type') or types.get(name, 'any'),
            'desc': m.group('desc').strip(),
        })
    for m in RETURN_DOC.finditer(docstring):
        result['returns'].append(m.group('desc').strip())
    for m in RAISES_DOC.finditer(docstring):
        result['raises'].append({'exception': m.group('exc'), 'desc': m.group('desc').strip()})
    return result


sample_docstring = """
    Create a new user account.

    :param str username: Unique username (3-20 characters)
    :param str email: User's email address
    :param password: Hashed password string
    :type password: str
    :param bool active: Whether account is active (default True)
    :returns: Created user object with id assigned
    :raises ValueError: If username or email is invalid
    :raises DuplicateError: If username or email already exists
    """

api_doc = extract_docstring_api(sample_docstring)
print(f"\n   Parameters:")
for p in api_doc['params']:
    print(f"   {p['name']:<15} [{p['type']:<8}] {p['desc']}")
print(f"\n   Returns: {api_doc['returns']}")
print(f"\n   Raises:")
for r in api_doc['raises']:
    print(f"   {r['exception']}: {r['desc']}")


print("\n\n2. Extract REST endpoints from Python source:")

ROUTE_DECORATOR = re.compile(
    r'@(?:app|router|blueprint)\.'
    r'(?P<method>get|post|put|delete|patch)\s*'
    r'\(\s*[\'"](?P<path>[^\'"]+)[\'"]\s*\)',
    re.IGNORECASE
)
FUNCTION_DEF = re.compile(r'(?:async\s+)?def\s+(?P<name>[a-zA-Z_]\w*)\s*\(')

source_code = '''
@app.get("/users")
async def list_users(): pass

@app.post("/users")
async def create_user(): pass

@router.get("/users/{user_id}")
def get_user(): pass

@router.put("/users/{user_id}")
async def update_user(): pass

@router.delete("/users/{user_id}")
async def delete_user(): pass
'''

print(f"\n   {'Method':<8} {'Path':<35} {'Handler'}")
print(f"   {'-'*8} {'-'*35} {'-'*20}")
lines = source_code.split('\n')
for i, line in enumerate(lines):
    m = ROUTE_DECORATOR.search(line)
    if m:
        for j in range(i+1, min(i+3, len(lines))):
            fn = FUNCTION_DEF.search(lines[j])
            if fn:
                print(f"   {m.group('method').upper():<8} {m.group('path'):<35} {fn.group('name')}")
                break
```

---

## 59.4 Semantic Versioning

```python
import re
from typing import Optional, Dict

print("\nSemantic Versioning (SemVer):")
print("=" * 60)

SEMVER = re.compile(
    r'^(?P<major>0|[1-9]\d*)'
    r'\.(?P<minor>0|[1-9]\d*)'
    r'\.(?P<patch>0|[1-9]\d*)'
    r'(?:-(?P<prerelease>(?:0|[1-9]\d*|\d*[a-zA-Z-][0-9a-zA-Z-]*)'
    r'(?:\.(?:0|[1-9]\d*|\d*[a-zA-Z-][0-9a-zA-Z-]*))*))?'
    r'(?:\+(?P<build>[0-9a-zA-Z-]+(?:\.[0-9a-zA-Z-]+)*))?$'
)

def parse_semver(version: str) -> Optional[Dict]:
    m = SEMVER.match(version.lstrip('v'))
    if not m:
        return None
    d = m.groupdict()
    return {
        'major':      int(d['major']),
        'minor':      int(d['minor']),
        'patch':      int(d['patch']),
        'prerelease': d['prerelease'],
        'build':      d['build'],
        'stable':     d['prerelease'] is None,
    }


def compare_semver(v1: str, v2: str) -> int:
    p1, p2 = parse_semver(v1), parse_semver(v2)
    if not p1 or not p2:
        return 0
    for field in ['major', 'minor', 'patch']:
        if p1[field] != p2[field]:
            return -1 if p1[field] < p2[field] else 1
    if p1['prerelease'] and not p2['prerelease']:
        return -1
    if not p1['prerelease'] and p2['prerelease']:
        return 1
    return 0


versions = ['1.0.0', '2.3.4', '0.1.0-alpha', '1.0.0-beta.2',
            '1.0.0-rc.1', '2.0.0+build.123', 'v3.0.0', 'invalid', '1.2']

print(f"\n   {'Version':<25} {'Valid':<8} {'Major':<8} {'Minor':<8} {'Patch':<8} {'Stable'}")
print(f"   {'-'*25} {'-'*8} {'-'*8} {'-'*8} {'-'*8} {'-'*8}")
for v in versions:
    p = parse_semver(v)
    if p:
        print(f"   {v:<25} {'True':<8} {p['major']:<8} {p['minor']:<8} {p['patch']:<8} {str(p['stable'])}")
    else:
        print(f"   {v:<25} {'False'}")

print(f"\n   Comparisons:")
for v1, v2 in [('1.0.0', '2.0.0'), ('1.0.0-alpha', '1.0.0'), ('2.0.0', '1.9.9')]:
    r = compare_semver(v1, v2)
    sym = '<' if r < 0 else ('>' if r > 0 else '=')
    print(f"   {v1} {sym} {v2}")
```

---

## 59.5 สรุป Part 59

```
API Design Regex Patterns:

1. OpenAPI paths:
   Parameter:  \{([a-zA-Z][a-zA-Z0-9_]*)\}
   Convert:    {param} → (?P<param>[^/]+)

2. GraphQL:
   Operation: (query|mutation|subscription)\s+(\w+)
   Variable:  \$(\w+)\s*:\s*([^\s,)]+)
   Fragment:  fragment\s+(\w+)\s+on\s+(\w+)

3. Docstring (Sphinx):
   :param type name: description
   :returns: description
   :raises ExcType: description

4. Route extraction:
   @app.(get|post|put|delete)\('(/[^']+)'\)
   Followed by: def\s+(\w+)\s*\(

5. SemVer:
   ^(major)\.(minor)\.(patch)(-prerelease)?(+build)?$
   Stable = no prerelease segment
   Comparison: major > minor > patch > prerelease
```

---

*[← Part 58: Network & Protocol Parsing](part-58-network-protocols.md) | [→ Part 60: Database & SQL Patterns](part-60-database-sql.md)*
