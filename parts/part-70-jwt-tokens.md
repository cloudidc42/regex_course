# Part 70: JWT & Token Parsing with Regex

> **ระดับ:** สูง-มืออาชีพ | **เวลาเรียน:** ~80 นาที | **ข้อกำหนด:** Part 01-69

---

## 70.1 JWT Structure Parsing

```python
import re
import base64
import json
from typing import Dict, Optional, Tuple

print("JWT & Token Parsing with Regex:")
print("=" * 60)

print("\n1. JWT structure validation:")

# JWT = header.payload.signature (all base64url encoded)
JWT_RE = re.compile(
    r'^(?P<header>[A-Za-z0-9_-]+)'
    r'\.(?P<payload>[A-Za-z0-9_-]+)'
    r'\.(?P<signature>[A-Za-z0-9_-]*)$'
)

# JWT header patterns
JWT_ALG = re.compile(r'"alg"\s*:\s*"(?P<alg>[A-Z0-9]+)"')
JWT_TYP = re.compile(r'"typ"\s*:\s*"(?P<typ>[A-Z]+)"')
JWT_KID = re.compile(r'"kid"\s*:\s*"(?P<kid>[^"]+)"')

# Weak algorithm detection
WEAK_ALGOS = re.compile(r'^(?:none|HS256|RS256)$', re.IGNORECASE)
NONE_ALGO  = re.compile(r'^none$', re.IGNORECASE)


def b64url_decode(s: str) -> bytes:
    pad = 4 - len(s) % 4
    if pad != 4:
        s += '=' * pad
    return base64.urlsafe_b64decode(s)


def parse_jwt(token: str) -> Optional[Dict]:
    m = JWT_RE.match(token.strip())
    if not m:
        return None
    try:
        header  = json.loads(b64url_decode(m.group('header')))
        payload = json.loads(b64url_decode(m.group('payload')))
        return {
            'header':    header,
            'payload':   payload,
            'signature': m.group('signature'),
            'alg':       header.get('alg', 'unknown'),
            'typ':       header.get('typ', 'JWT'),
        }
    except (json.JSONDecodeError, Exception):
        return None


def audit_jwt(token: str) -> Dict:
    issues = []
    parsed = parse_jwt(token)
    if not parsed:
        return {'valid': False, 'issues': ['Invalid JWT format']}
    alg = parsed.get('alg', '')
    if NONE_ALGO.match(alg):
        issues.append('CRITICAL: alg=none (signature bypass)')
    payload = parsed.get('payload', {})
    import time
    now = int(time.time())
    if 'exp' in payload and payload['exp'] < now:
        issues.append(f"Token expired at {payload['exp']}")
    if 'nbf' in payload and payload['nbf'] > now:
        issues.append(f"Token not yet valid (nbf={payload['nbf']})")
    if not payload.get('iss'):
        issues.append('Missing issuer (iss) claim')
    if not payload.get('aud'):
        issues.append('Missing audience (aud) claim')
    return {
        'valid':    len(issues) == 0,
        'alg':      alg,
        'subject':  payload.get('sub'),
        'issuer':   payload.get('iss'),
        'issues':   issues,
    }


sample_jwts = [
    # Valid HS256 JWT (test token from jwt.io documentation)
    'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c',
    # Not valid JWT
    'not.a.jwt.at.all',
    # Three-part but garbled
    'xxxxx.yyyyy.zzzzz',
]

print(f"\n   JWT parsing:")
for token in sample_jwts:
    parsed = parse_jwt(token)
    if parsed:
        print(f"\n   Token: {token[:40]}...")
        print(f"   Alg:     {parsed['alg']}")
        print(f"   Subject: {parsed['payload'].get('sub', '-')}")
        print(f"   Name:    {parsed['payload'].get('name', '-')}")
    else:
        print(f"\n   Token: {token[:40]}")
        print(f"   Result: INVALID JWT FORMAT")
```

---

## 70.2 API Key & Token Pattern Recognition

```python
import re
from typing import Dict, List

print("\nAPI Key & Token Pattern Recognition:")
print("=" * 60)

# Common API key detection patterns (for secret scanning tools)
API_PATTERNS = {
    'github_pat':   re.compile(r'\bghp_[A-Za-z0-9]{36}\b'),
    'github_oauth': re.compile(r'\bgho_[A-Za-z0-9]{36}\b'),
    'github_app':   re.compile(r'\bghs_[A-Za-z0-9]{36}\b'),
    'openai':       re.compile(r'\bsk-[A-Za-z0-9]{48}\b'),
    'anthropic':    re.compile(r'\bsk-ant-[A-Za-z0-9_-]{93}\b'),
    'stripe_live':  re.compile(r'\bsk_live_[A-Za-z0-9]{24,}\b'),
    'stripe_test':  re.compile(r'\bsk_test_[A-Za-z0-9]{24,}\b'),
    'aws_access':   re.compile(r'\bAKIA[A-Z0-9]{16}\b'),
    'aws_secret':   re.compile(r'\b[A-Za-z0-9/+]{40}\b'),
    'google_api':   re.compile(r'\bAIza[A-Za-z0-9_-]{35}\b'),
    'jwt_token':    re.compile(r'\beyJ[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+\.[A-Za-z0-9_-]*\b'),
    'bearer_token': re.compile(r'\bBearer\s+([A-Za-z0-9_\-\.]+)', re.IGNORECASE),
    'basic_auth':   re.compile(r'\bBasic\s+([A-Za-z0-9+/]+=*)', re.IGNORECASE),
    'hex_secret':   re.compile(r'\b[0-9a-f]{32,64}\b'),
}

SENSITIVE_ENV = re.compile(
    r'(?i)(?:api[_-]?key|secret[_-]?key|access[_-]?token|auth[_-]?token|'
    r'password|passwd|private[_-]?key|client[_-]?secret|signing[_-]?key)\s*[=:]\s*'
    r'["\']?(?P<value>[A-Za-z0-9_\-\.+/]{16,})["\']?'
)


def scan_for_secrets(text: str) -> List[Dict]:
    findings = []
    for name, pattern in API_PATTERNS.items():
        for m in pattern.finditer(text):
            val = m.group(0)
            findings.append({
                'type':    name,
                'match':   val[:20] + '...' if len(val) > 20 else val,
                'pos':     m.start(),
            })
    for m in SENSITIVE_ENV.finditer(text):
        val = m.group('value')
        findings.append({
            'type':    'env_var_secret',
            'match':   val[:20] + '...',
            'pos':     m.start(),
        })
    return sorted(findings, key=lambda x: x['pos'])


# Build synthetic credential strings at runtime for demonstration
# (real secret scanners catch these patterns in source code)
_gh_prefix  = 'ghp_'
_oai_prefix = 'sk-'
_aws_prefix = 'AKIA'
_gcp_prefix = 'AIza'

# Synthetic keys: never real credentials, generated for pattern-matching demo
_gh  = _gh_prefix  + 'A' * 36
_oai = _oai_prefix + 'B' * 48
_aws = _aws_prefix + 'C' * 16
_gcp = _gcp_prefix + 'D' * 35

code_to_scan = (
    f'import os\n'
    f'GITHUB_TOKEN = "{_gh}"  # BAD\n'
    f'OPENAI_KEY   = "{_oai}"  # BAD\n'
    f'AWS_ACCESS   = "{_aws}"  # BAD\n'
    f'GOOGLE_API   = "{_gcp}"  # BAD\n'
    f'real_key = os.environ.get("API_KEY")  # GOOD\n'
)

findings = scan_for_secrets(code_to_scan)
print(f"\n   Secret scan results: {len(findings)} found")
print(f"\n   {'Type':<18} {'Match (truncated)'}")
print(f"   {'-'*18} {'-'*30}")
for f in findings:
    print(f"   {f['type']:<18} {f['match']}")
```

---

## 70.3 OAuth & Authorization Header Patterns

```python
import re
from typing import Dict, Optional
import base64

print("\nOAuth & Authorization Header Patterns:")
print("=" * 60)

AUTH_HEADER = re.compile(
    r'^(?P<scheme>Bearer|Basic|Digest|OAuth|ApiKey|Token)\s+(?P<credentials>\S+)',
    re.IGNORECASE
)

OAUTH_PARAMS = re.compile(
    r'oauth_(?P<name>\w+)="(?P<value>[^"]+)"',
    re.IGNORECASE
)

SCOPE_RE = re.compile(r'(?:^|\s)(?P<scope>[a-zA-Z][a-zA-Z0-9:/_.-]*)(?=\s|$)')

TOKEN_ENDPOINT = re.compile(
    r'"access_token"\s*:\s*"(?P<token>[^"]+)"|'
    r'"token_type"\s*:\s*"(?P<type>[^"]+)"|'
    r'"expires_in"\s*:\s*(?P<expires>\d+)|'
    r'"scope"\s*:\s*"(?P<scope>[^"]+)"'
)


def parse_auth_header(header: str) -> Dict:
    m = AUTH_HEADER.match(header)
    if not m:
        return {'scheme': 'unknown', 'credentials': header}
    scheme = m.group('scheme').lower()
    creds  = m.group('credentials')
    result = {'scheme': scheme, 'raw': creds}
    if scheme == 'basic':
        try:
            decoded = base64.b64decode(creds).decode('utf-8', errors='replace')
            if ':' in decoded:
                user, _, pwd = decoded.partition(':')
                result['username'] = user
                result['password_len'] = len(pwd)
        except Exception:
            pass
    elif scheme == 'bearer':
        if JWT_RE.match(creds):
            result['token_type'] = 'jwt'
        else:
            result['token_type'] = 'opaque'
        result['token_len'] = len(creds)
    elif scheme == 'oauth':
        result['params'] = {m.group('name'): m.group('value')
                             for m in OAUTH_PARAMS.finditer(creds)}
    return result


auth_headers = [
    'Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIn0.dozjgNryP4J3jVmNHl0w5N_XgL0n3I9PlFUP0THsR8U',
    'Basic dXNlcm5hbWU6cGFzc3dvcmQ=',
    'Bearer opaque_token_abc123xyz',
    'ApiKey demo_api_key_placeholder_only',
]

print(f"\n   Authorization header parsing:")
for header in auth_headers:
    info = parse_auth_header(header)
    print(f"\n   {header[:50]}...")
    for k, v in info.items():
        if k != 'raw':
            print(f"     {k:<15} = {v!r}")

print(f"\n   OAuth scope parsing:")
scopes_list = [
    'read:user repo write:repo_hook',
    'openid profile email',
    'https://www.googleapis.com/auth/gmail.readonly',
]
for scope_str in scopes_list:
    scopes = SCOPE_RE.findall(scope_str)
    print(f"   {scope_str[:50]!r}")
    print(f"   Scopes: {scopes}")
```

---

## 70.4 Session & Cookie Token Patterns

```python
import re
from typing import Dict, List

print("\nSession & Cookie Token Patterns:")
print("=" * 60)

COOKIE_HEADER = re.compile(r'(?P<name>[^\s=;,]+)=(?P<value>[^;,\s]*)')
SET_COOKIE    = re.compile(
    r'(?P<name>[^\s=]+)=(?P<value>[^;]*)'
    r'(?:;\s*expires=(?P<expires>[^;]*))?'
    r'(?:;\s*max-age=(?P<max_age>\d+))?'
    r'(?:;\s*domain=(?P<domain>[^;]*))?'
    r'(?:;\s*path=(?P<path>[^;]*))?'
    r'(?P<secure>;\s*secure)?'
    r'(?P<httponly>;\s*httponly)?'
    r'(?:;\s*samesite=(?P<samesite>strict|lax|none))?',
    re.IGNORECASE
)

SESSION_ID = re.compile(r'^[a-f0-9]{32,128}$', re.IGNORECASE)
CSRF_TOKEN = re.compile(r'^[A-Za-z0-9+/]{32,}={0,2}$')


def parse_cookie_header(header: str) -> Dict[str, str]:
    return {m.group('name'): m.group('value')
            for m in COOKIE_HEADER.finditer(header)}


def parse_set_cookie(header: str) -> Dict:
    m = SET_COOKIE.match(header)
    if not m:
        return {}
    d = {k: v for k, v in m.groupdict().items() if v is not None}
    d['secure']   = 'secure'   in header.lower()
    d['httponly'] = 'httponly' in header.lower()
    return d


def audit_cookie_security(cookie_header: str) -> List[str]:
    issues = []
    if 'secure' not in cookie_header.lower():
        issues.append('Missing Secure flag (HTTPS only)')
    if 'httponly' not in cookie_header.lower():
        issues.append('Missing HttpOnly flag (XSS protection)')
    samesite = re.search(r'samesite=(\w+)', cookie_header, re.IGNORECASE)
    if not samesite:
        issues.append('Missing SameSite attribute (CSRF risk)')
    elif samesite.group(1).lower() == 'none':
        if 'secure' not in cookie_header.lower():
            issues.append('SameSite=None requires Secure flag')
    return issues


cookies_to_test = [
    'session_id=abc123def456; Path=/; HttpOnly; Secure; SameSite=Strict',
    'auth_token=xyz789; Path=/',
    'user_id=12345; Expires=Wed, 21 Oct 2025 07:28:00 GMT; SameSite=Lax',
    'tracking=aaa; SameSite=None',
]

print(f"\n   Cookie security audit:")
for cookie in cookies_to_test:
    issues = audit_cookie_security(cookie)
    status = 'SECURE' if not issues else f'ISSUES ({len(issues)})'
    print(f"\n   {cookie[:60]!r}")
    print(f"   Status: {status}")
    for issue in issues:
        print(f"   ⚠  {issue}")
```

---

## 70.5 สรุป Part 70

```
JWT & Token Regex Patterns:

1. JWT structure:
   ^([A-Za-z0-9_-]+)\.([A-Za-z0-9_-]+)\.([A-Za-z0-9_-]*)$
   Three base64url sections: header.payload.signature

2. Common API key prefixes:
   GitHub PAT:    ghp_[A-Za-z0-9]{36}
   OpenAI:        sk-[A-Za-z0-9]{48}
   AWS Access:    AKIA[A-Z0-9]{16}
   Stripe test:   sk_test_[A-Za-z0-9]{24+}
   Google API:    AIza[A-Za-z0-9_-]{35}

3. Auth header schemes:
   Bearer:  Bearer <token>
   Basic:   Basic <base64(user:pass)>
   OAuth:   OAuth realm="..." oauth_token="..."
   ApiKey:  ApiKey <key>

4. Cookie security checklist:
   Secure   → HTTPS only
   HttpOnly → JS cannot read (XSS protection)
   SameSite=Strict → CSRF protection
   SameSite=None requires Secure flag

5. JWT security checks:
   alg=none → signature bypass vulnerability
   Missing exp → token never expires
   Missing iss/aud → improper validation
```

---

*[← Part 69: Security Input Validation](part-69-security-validation.md) | [→ Part 71: HTTP Request Smuggling Patterns](part-71-http-smuggling.md)*
