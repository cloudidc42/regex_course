# Part 90: Advanced Python `re` Module Techniques

> **ระดับ:** ผู้เชี่ยวชาญ-ระดับโลก | **เวลาเรียน:** ~105 นาที | **ข้อกำหนด:** Part 01-89

---

## 90.1 Named Groups, Conditionals & Inline Flags

```python
import re
from typing import Dict, Optional

print("Advanced Python re Module Techniques:")
print("=" * 60)

print("\n1. Named groups and backreferences:")

# Named groups for readable patterns
LOG_PATTERN = re.compile(
    r'(?P<timestamp>\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}(?:\.\d+)?Z?)\s+'
    r'(?P<level>DEBUG|INFO|WARNING|ERROR|CRITICAL)\s+'
    r'\[(?P<module>[^\]]+)\]\s+'
    r'(?P<message>.+)',
    re.IGNORECASE
)

# Named backreference in same pattern
MATCHING_TAGS = re.compile(r'<(?P<tag>[a-zA-Z][a-zA-Z0-9]*)[^>]*>.*?</(?P=tag)>', re.DOTALL)

# Named groups in substitution
def anonymize_log(log_line: str) -> str:
    """Replace IP addresses in log lines, keep structure."""
    IP_PATTERN = re.compile(
        r'(?P<pre>src=|from\s+|address\s+)(?P<ip>\d{1,3}(?:\.\d{1,3}){3})'
    )
    return IP_PATTERN.sub(lambda m: m.group('pre') + 'x.x.x.x', log_line)


log_lines = [
    '2024-01-15T10:30:45Z INFO [auth.module] User login successful',
    '2024-01-15T10:30:46.123Z ERROR [db.connection] Connection timeout',
    '2024-01-15T10:30:47Z WARNING [security] Rate limit exceeded',
    '2024-01-15T10:30:48Z DEBUG [api.handler] Request received',
]

print(f"\n   Named group log parsing:")
for line in log_lines:
    m = LOG_PATTERN.match(line)
    if m:
        print(f"   [{m.group('level'):<8}] {m.group('timestamp')} | {m.group('module')} | {m.group('message')[:40]}")

print(f"\n   Named backreference (matching tags):")
html_samples = [
    '<div class="test">content</div>',
    '<span>inner</span>',
    '<div>unclosed<span>',
    '<p>paragraph text</p>',
]
for html in html_samples:
    m = MATCHING_TAGS.search(html)
    if m:
        print(f"   Matched tag: <{m.group('tag')}> content: {m.group(0)[:40]!r}")

print(f"\n   Log anonymization:")
raw_logs = [
    'Connection from src=192.168.1.100 port 54321',
    'Blocked from 10.0.0.5 port 80',
    'Normal log entry without IPs',
]
for log in raw_logs:
    print(f"   {anonymize_log(log)}")


print("\n2. Inline flags:")

# Verbose mode for complex patterns
EMAIL_VERBOSE = re.compile(r'''
    (?x)                        # Verbose mode
    ^                           # Start of string
    (?P<local>                  # Local part
        [a-zA-Z0-9._%+\-]{1,64}  # Allowed chars, max 64
    )
    @                           # At sign
    (?P<domain>                 # Domain part
        [a-zA-Z0-9.\-]{1,253}    # Domain chars, max 253
    )
    \.                          # Dot before TLD
    (?P<tld>
        [a-zA-Z]{2,63}          # TLD: 2-63 alpha chars
    )
    $                           # End of string
''')

test_emails = ['user@example.com', 'admin@sub.domain.co.uk', 'invalid', '@bad.com']
print(f"\n   Verbose email regex:")
for email in test_emails:
    m = EMAIL_VERBOSE.match(email)
    if m:
        print(f"   OK: {email!r} → local={m.group('local')!r}, domain={m.group('domain')!r}")
    else:
        print(f"   FAIL: {email!r}")
```

---

## 90.2 Lookahead, Lookbehind & Overlapping Matches

```python
import re
from typing import List

print("\nLookahead, Lookbehind & Non-Capturing Groups:")
print("=" * 60)

# Positive lookahead: match X only if followed by Y
POS_LOOKAHEAD = re.compile(r'foo(?=bar)')

# Positive lookbehind: match X only if preceded by Y
POS_LOOKBEHIND = re.compile(r'(?<=\$)\d+(?:\.\d{2})?')

# Negative lookbehind: match 'script' not preceded by '/'
NEG_LOOKBEHIND = re.compile(r'(?<!//)\bscript\b', re.IGNORECASE)


# Password strength validation using lookaheads
def check_password_complexity(password: str) -> dict:
    return {
        'min_length':   len(password) >= 8,
        'has_upper':    bool(re.search(r'(?=[A-Z])', password)),
        'has_lower':    bool(re.search(r'(?=[a-z])', password)),
        'has_digit':    bool(re.search(r'(?=\d)', password)),
        'has_special':  bool(re.search(r'(?=[!@#$%^&*])', password)),
        'no_sequences': not bool(re.search(r'(?:012|123|234|345|456|567|678|789|abc|bcd|cde)', password, re.IGNORECASE)),
    }


# Overlapping matches via lookahead trick
def find_overlapping(pattern: str, text: str) -> List[str]:
    """Find all overlapping matches."""
    compiled = re.compile(r'(?=' + pattern + r')')
    return [text[m.start():m.start()+len(pattern)] for m in compiled.finditer(text)
            if m.start() + len(pattern) <= len(text)]


print(f"\n   Lookahead/lookbehind examples:")
samples = [
    ('foobar fooqwerty foobaz', 'POS lookahead (foo before bar)', POS_LOOKAHEAD),
    ('price: $29.99 and $100.00 total', 'POS lookbehind (number after $)', POS_LOOKBEHIND),
]
for text, desc, pattern in samples:
    matches = pattern.findall(text)
    print(f"\n   {desc}:")
    print(f"   Text: {text!r}")
    print(f"   Matches: {matches}")

print(f"\n   Password complexity:")
passwords = ['Password1!', 'weakpass', 'ALLCAPS1!', 'abc123456', 'GoodP@ss1']
for pwd in passwords:
    result = check_password_complexity(pwd)
    all_ok = all(result.values())
    failed = [k for k, v in result.items() if not v]
    status = 'STRONG' if all_ok else 'WEAK'
    print(f"   [{status}] {pwd!r}" + (f" (failed: {failed})" if failed else ""))

print(f"\n   Overlapping matches:")
overlap_text = 'abcabcabc'
pattern_str = 'abc'
overlapping = find_overlapping(pattern_str, overlap_text)
print(f"   Text: {overlap_text!r}, Pattern: {pattern_str!r}")
print(f"   Overlapping matches: {overlapping}")
```

---

## 90.3 Custom Tokenizers with Master Pattern

```python
import re
from typing import List, Tuple

print("\nCustom Tokenizers:")
print("=" * 60)


def tokenize_expression(expr: str) -> List[Tuple[str, str]]:
    """Tokenize a mathematical/logical expression."""
    TOKEN_PATTERNS = [
        ('NUMBER',    r'\d+(?:\.\d+)?(?:[eE][+-]?\d+)?'),
        ('STRING',    r'"(?:[^"\\]|\\.)*"|\' (?:[^\'\\]|\\.)*\''),
        ('IDENT',     r'[a-zA-Z_]\w*'),
        ('OP',        r'[+\-*/^%]|<=|>=|!=|==|<|>|&&|\|\|'),
        ('LPAREN',    r'\('),
        ('RPAREN',    r'\)'),
        ('COMMA',     r','),
        ('WHITESPACE', r'\s+'),
        ('UNKNOWN',    r'.'),
    ]

    master_pattern = re.compile(
        '|'.join(f'(?P<{name}>{pat})' for name, pat in TOKEN_PATTERNS)
    )

    tokens = []
    for m in master_pattern.finditer(expr):
        kind = m.lastgroup
        value = m.group()
        if kind != 'WHITESPACE':
            tokens.append((kind, value))

    return tokens


def tokenize_sql(sql: str) -> List[Tuple[str, str]]:
    SQL_TOKENS = [
        ('KEYWORD',   r'\b(?:SELECT|FROM|WHERE|AND|OR|NOT|IN|LIKE|IS|NULL|JOIN|LEFT|RIGHT|INNER|OUTER|ON|GROUP|BY|ORDER|HAVING|LIMIT|OFFSET|INSERT|INTO|VALUES|UPDATE|SET|DELETE|CREATE|TABLE|DROP|ALTER|UNION|ALL|DISTINCT|AS)\b'),
        ('NUMBER',    r'\d+(?:\.\d+)?'),
        ('STRING',    r"'(?:[^'\\]|\\.)*'"),
        ('IDENT',     r'[a-zA-Z_]\w*(?:\.[a-zA-Z_]\w*)*'),
        ('OP',        r'>=|<=|<>|!=|=|>|<|\|\|||&&'),
        ('PUNCT',     r'[(),;*]'),
        ('WHITESPACE', r'\s+'),
        ('COMMENT',   r'--[^\n]*|/\*.*?\*/'),
        ('UNKNOWN',    r'.'),
    ]

    master = re.compile(
        '|'.join(f'(?P<{name}>{pat})' for name, pat in SQL_TOKENS),
        re.IGNORECASE | re.DOTALL
    )

    tokens = []
    for m in master.finditer(sql):
        kind = m.lastgroup
        value = m.group()
        if kind not in ('WHITESPACE', 'COMMENT'):
            tokens.append((kind, value.upper() if kind == 'KEYWORD' else value))

    return tokens


expressions = [
    '(a + b) * c^2 >= 100',
    'name == "Alice" && age > 25',
    'sum(x, y, 2.5e3)',
]

print(f"\n   Expression tokenizer:")
for expr in expressions:
    tokens = tokenize_expression(expr)
    print(f"\n   {expr!r}")
    print(f"   → {tokens}")


sql_queries = [
    "SELECT id, name FROM users WHERE age > 18",
    "SELECT * FROM products WHERE name LIKE '%phone%' ORDER BY price LIMIT 10",
]

print(f"\n   SQL tokenizer:")
for sql in sql_queries:
    tokens = tokenize_sql(sql)
    print(f"\n   {sql!r}")
    keywords = [v for k, v in tokens if k == 'KEYWORD']
    print(f"   → Keywords: {keywords}")
    print(f"   → All tokens: {[(k, v[:20]) for k, v in tokens[:8]]}")
```

---

## 90.4 สรุป Part 90

```
Advanced Python re Module:

1. Named groups (?P<name>...):
   - Better readability than positional groups
   - Access via m.group('name') instead of m.group(1)
   - Backreference: (?P=name) in pattern
   - In substitution: \g<name> or use lambda with m.group('name')

2. Inline flags:
   (?i) = case-insensitive (IGNORECASE)
   (?m) = multiline (^ and $ match each line)
   (?s) = dotall (. matches newline too)
   (?x) = verbose (whitespace and # comments ignored)
   Combine: (?im) = IGNORECASE + MULTILINE

3. Lookahead/lookbehind:
   (?=Y)  = positive lookahead: X must be followed by Y
   (?!Y)  = negative lookahead: X must NOT be followed by Y
   (?<=Y) = positive lookbehind: X must be preceded by Y (fixed width)
   (?<!Y) = negative lookbehind: X must NOT be preceded by Y

4. Performance tips:
   Compile patterns: re.compile() once, reuse
   Anchor when possible: ^ and $ reduce backtracking
   Specific chars: [abc] instead of (a|b|c) for single chars
   Non-capturing: (?:...) instead of (...) when group not needed
   re2 library: linear-time guarantee for untrusted input

5. Custom tokenizers:
   Build master pattern: '|'.join(f'(?P<{name}>{pat})' for ...)
   Use m.lastgroup to get token type
   Process each match with finditer for efficiency
   Skip whitespace/comments in the token stream
```

---

*[← Part 89: Regex Performance & ReDoS](part-89-redos.md) | [→ Part 91: Real-World Regex Applications](part-91-realworld.md)*
