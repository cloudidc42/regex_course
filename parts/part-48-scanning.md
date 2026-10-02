# Part 48: Scanning & Iterating — re.finditer(), re.scanner()

> **ระดับ:** กลาง | **เวลาเรียน:** ~60 นาที | **ข้อกำหนด:** Part 01-47

---

## 48.1 re.finditer() vs re.findall()

```python
import re
from typing import Iterator, List, Dict

print("re.finditer() vs re.findall():")
print("=" * 60)

text = """
Error at line 42: null pointer dereference
Warning at line 15: unused variable 'x'
Error at line 88: index out of bounds
Info at line 3: server started
Error at line 101: connection timeout
"""

ERROR_RE = re.compile(r'(?P<level>\w+) at line (?P<line>\d+): (?P<message>.+)')

# ===== findall() — returns list of strings/tuples =====
print("\n1. re.findall() — list of captures:")
results = ERROR_RE.findall(text)
for level, line, msg in results:
    print(f"   {level:<8} line {line:<5}: {msg}")

# ===== finditer() — iterator of Match objects =====
print("\n2. re.finditer() — Match objects:")
for m in ERROR_RE.finditer(text):
    d = m.groupdict()
    print(f"   pos={m.start():<6} {d['level']:<8} line {d['line']:<5}: {d['message']}")

# ===== Advantages of finditer() =====
print("\n3. finditer() advantages:")
print("   - Memory efficient (lazy, one match at a time)")
print("   - Full Match object: start(), end(), span(), groupdict()")
print("   - Can break early from iteration")
print("   - Can get position info for highlighting/replacement")

# Find first ERROR with early stop
print("\n4. Early stop with finditer():")
first_error = next(
    (m for m in ERROR_RE.finditer(text) if m.group('level') == 'Error'),
    None
)
if first_error:
    d = first_error.groupdict()
    print(f"   First error: line {d['line']}: {d['message']}")

# ===== Position information =====
print("\n5. Position info (for highlighting):")
source = "Python is great. Python regex is powerful. I love Python!"
PYTHON_RE = re.compile(r'Python')

positions = [(m.start(), m.end(), m.group()) for m in PYTHON_RE.finditer(source)]
print(f"   Text: {source!r}")
print(f"   Positions: {positions}")

# Highlight with markers
highlighted = source
offset = 0
for start, end, word in positions:
    s, e = start + offset, end + offset
    highlighted = highlighted[:s] + f'[{word}]' + highlighted[e:]
    offset += 2  # added 2 chars: [ and ]
print(f"   Highlighted: {highlighted}")
```

---

## 48.2 Scanning with State

```python
import re
from dataclasses import dataclass, field
from typing import List, Optional

print("\nStateful Scanning:")
print("=" * 60)

# ===== Token scanner / lexer =====
@dataclass
class Token:
    type: str
    value: str
    line: int
    col: int
    
    def __repr__(self):
        return f"Token({self.type}, {self.value!r}, {self.line}:{self.col})"


class Lexer:
    """Simple tokenizer using regex alternation"""
    
    # Order matters: longer/more specific patterns first
    TOKEN_PATTERNS = [
        ('FLOAT',    re.compile(r'\d+\.\d+')),
        ('INT',      re.compile(r'\d+')),
        ('STRING',   re.compile(r'"[^"\\\\]*(?:\\\\.[^"\\\\]*)*"')),
        ('BOOL',     re.compile(r'\b(?:true|false)\b')),
        ('NULL',     re.compile(r'\bnull\b')),
        ('LBRACE',   re.compile(r'\{')),
        ('RBRACE',   re.compile(r'\}')),
        ('LBRACKET', re.compile(r'\[')),
        ('RBRACKET', re.compile(r'\]')),
        ('COLON',    re.compile(r':')),
        ('COMMA',    re.compile(r',')),
        ('IDENT',    re.compile(r'[a-zA-Z_]\w*')),
        ('WS',       re.compile(r'[ \t]+')),
        ('NEWLINE',  re.compile(r'\n')),
    ]
    
    def tokenize(self, text: str) -> List[Token]:
        tokens = []
        pos = 0
        line = 1
        line_start = 0
        
        while pos < len(text):
            matched = False
            for token_type, pattern in self.TOKEN_PATTERNS:
                m = pattern.match(text, pos)
                if m:
                    if token_type not in ('WS',):  # skip whitespace
                        col = pos - line_start + 1
                        tokens.append(Token(token_type, m.group(), line, col))
                    if token_type == 'NEWLINE':
                        line += 1
                        line_start = m.end()
                    pos = m.end()
                    matched = True
                    break
            
            if not matched:
                col = pos - line_start + 1
                raise SyntaxError(f"Unexpected char {text[pos]!r} at line {line}, col {col}")
        
        return tokens


print("\n1. JSON Lexer:")
lexer = Lexer()
json_text = '{"name": "Alice", "age": 30, "active": true}'
tokens = lexer.tokenize(json_text)
for tok in tokens:
    print(f"   {tok.type:<12} {tok.value!r}")


# ===== Streaming scanner =====
class LogScanner:
    """Scan log file incrementally, extract structured events"""
    
    PATTERNS = {
        'error':    re.compile(r'(?P<ts>\d{4}-\d{2}-\d{2}[T ]\d{2}:\d{2}:\d{2})\s+ERROR\s+(?P<msg>.+)'),
        'warning':  re.compile(r'(?P<ts>\d{4}-\d{2}-\d{2}[T ]\d{2}:\d{2}:\d{2})\s+WARN\w*\s+(?P<msg>.+)'),
        'metric':   re.compile(r'(?P<key>\w+(?:\.\w+)*)=(?P<value>\d+(?:\.\d+)?)(?P<unit>ms|s|MB|GB|%)?'),
        'request':  re.compile(r'"(?P<method>GET|POST|PUT|DELETE|PATCH)\s+(?P<path>[^\s"]+)\s+HTTP'),
    }
    
    def scan(self, lines: List[str]) -> Dict[str, List]:
        results = {k: [] for k in self.PATTERNS}
        
        for line in lines:
            for kind, pattern in self.PATTERNS.items():
                m = pattern.search(line)
                if m:
                    results[kind].append({**m.groupdict(), 'raw': line.strip()})
        
        return results


print("\n2. Log Scanner:")
log_lines = [
    '2024-10-15T09:00:01 ERROR [db] Connection failed after 3 retries',
    '2024-10-15T09:00:02 INFO [web] "GET /api/users HTTP/1.1" 200 duration=45ms',
    '2024-10-15T09:00:03 WARNING [cache] Cache hit rate=0.45 low, evictions=12',
    '2024-10-15T09:00:04 ERROR [auth] Invalid token: token_expired',
    '2024-10-15T09:00:05 INFO [web] "POST /api/login HTTP/1.1" 200 duration=120ms',
    '2024-10-15T09:00:06 WARN [mem] Memory usage=87% high, free=128MB',
]

scanner = LogScanner()
results = scanner.scan(log_lines)

for kind, events in results.items():
    if events:
        print(f"\n   {kind.upper()} events ({len(events)}):")
        for event in events[:2]:
            display = {k: v for k, v in event.items() if k != 'raw'}
            print(f"     {display}")
```

---

## 48.3 Match Object — Full API

```python
import re

print("\nMatch Object — Full API:")
print("=" * 60)

text = "2024-10-15T09:30:45+07:00 ERROR [auth.login] user=alice failed"
PATTERN = re.compile(
    r'(?P<date>\d{4}-\d{2}-\d{2})'
    r'T(?P<time>\d{2}:\d{2}:\d{2})'
    r'(?P<tz>[+-]\d{2}:\d{2})?'
    r'\s+(?P<level>\w+)'
    r'\s+\[(?P<module>[^\]]+)\]'
    r'\s+(?P<data>.+)'
)

m = PATTERN.match(text)
if m:
    print("\n  Match object methods:")
    print(f"  m.group()         = {m.group()!r[:50]}...")
    print(f"  m.group(0)        = {m.group(0)!r[:50]}...")
    print(f"  m.group('level')  = {m.group('level')!r}")
    print(f"  m.group(4)        = {m.group(4)!r}")
    print(f"  m.groups()        = {m.groups()}")
    print(f"  m.groupdict()     = {m.groupdict()}")
    print(f"  m.start()         = {m.start()}")
    print(f"  m.end()           = {m.end()}")
    print(f"  m.span()          = {m.span()}")
    print(f"  m.start('level')  = {m.start('level')}")
    print(f"  m.end('level')    = {m.end('level')}")
    print(f"  m.string          = {m.string!r[:40]}...")
    print(f"  m.re              = {m.re.pattern!r[:40]}")
    print(f"  m.lastgroup       = {m.lastgroup!r}")
    print(f"  m.lastindex       = {m.lastindex}")
    print(f"  m.pos             = {m.pos}")
    print(f"  m.endpos          = {m.endpos}")

# ===== expand() method =====
print("\n  m.expand() — template expansion:")
if m:
    expanded = m.expand(r'\g<level> from \g<module> at \g<date>')
    print(f"  Template: '\\g<level> from \\g<module> at \\g<date>'")
    print(f"  Expanded: {expanded!r}")


# ===== Partial matching with pos/endpos =====
print("\n  Search with pos/endpos:")
text = "A1B2C3D4E5"
DIGIT = re.compile(r'\d')

print(f"  Text: {text!r}")
# Search only in first 6 chars
for m in re.finditer(r'\d', text[:6]):
    print(f"  Digit at {m.start()}: {m.group()!r}")

# Use pos parameter
print(f"\n  finditer with pos=4:")
p = re.compile(r'\d')
for m in p.finditer(text, 4):
    print(f"  Digit at {m.start()}: {m.group()!r}")
```

---

## 48.4 Practical: Document Information Extractor

```python
import re
from typing import Dict, List, Any

print("\nDocument Information Extractor:")
print("=" * 60)

class DocumentExtractor:
    """Extract structured data from unstructured text"""
    
    # Email
    EMAIL = re.compile(r'\b[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}\b')
    
    # Thai phone
    PHONE_TH = re.compile(r'(?:0|\+66)\s?[6-9]\d{8}')
    
    # URL
    URL = re.compile(
        r'https?://[a-zA-Z0-9\-._~:/?#\[\]@!$&\'()*+,;=%]+'
    )
    
    # Thai national ID
    THAI_ID = re.compile(r'\b\d{1}-\d{4}-\d{5}-\d{2}-\d\b')
    
    # Date formats
    DATE = re.compile(
        r'\b(?:'
        r'\d{4}[-/]\d{2}[-/]\d{2}'      # 2024-10-15 or 2024/10/15
        r'|\d{2}[-/]\d{2}[-/]\d{4}'      # 15-10-2024 or 15/10/2024
        r'|\d{1,2}\s+(?:Jan|Feb|Mar|Apr|May|Jun|Jul|Aug|Sep|Oct|Nov|Dec)\w*\s+\d{4}'  # 15 Oct 2024
        r')\b',
        re.IGNORECASE
    )
    
    # Thai baht amount
    AMOUNT = re.compile(r'(?:฿|บาท|THB)\s*[\d,]+(?:\.\d{2})?|[\d,]+(?:\.\d{2})?\s*(?:บาท|THB)', re.IGNORECASE)
    
    # IP address
    IP = re.compile(r'\b(?:\d{1,3}\.){3}\d{1,3}\b')
    
    @classmethod
    def extract_all(cls, text: str) -> Dict[str, List[str]]:
        results = {}
        
        for field, pattern in [
            ('emails',     cls.EMAIL),
            ('phones',     cls.PHONE_TH),
            ('urls',       cls.URL),
            ('thai_ids',   cls.THAI_ID),
            ('dates',      cls.DATE),
            ('amounts',    cls.AMOUNT),
            ('ips',        cls.IP),
        ]:
            found = [m.group() for m in pattern.finditer(text)]
            if found:
                results[field] = list(dict.fromkeys(found))  # deduplicate preserving order
        
        return results
    
    @classmethod
    def redact(cls, text: str, fields: List[str] = None) -> str:
        """Redact sensitive fields from text"""
        REDACT_MAP = {
            'emails':   (cls.EMAIL,    '[EMAIL REDACTED]'),
            'phones':   (cls.PHONE_TH, '[PHONE REDACTED]'),
            'thai_ids': (cls.THAI_ID,  '[ID REDACTED]'),
        }
        
        if fields is None:
            fields = list(REDACT_MAP.keys())
        
        result = text
        for field in fields:
            if field in REDACT_MAP:
                pattern, replacement = REDACT_MAP[field]
                result = pattern.sub(replacement, result)
        
        return result


# ===== Test =====
sample_document = """
ใบเสนอราคา

ลูกค้า: นางสาวสมหญิง ใจดี
อีเมล: somying@company.co.th
โทรศัพท์: 089-123-4567
เลขบัตร: 1-1234-12345-12-3

รายการ:
1. Python training course - ฿15,000.00
2. Regex workshop - 5,500 บาท
Total: THB 20,500.00

Website: https://www.example.co.th/courses
Payment by: 31 Dec 2024

Server IP: 192.168.1.100
Log time: 2024-10-15T09:30:00

Contact: admin@example.co.th or +66891234567
"""

print("\n1. Extraction:")
extractor = DocumentExtractor
results = extractor.extract_all(sample_document)
for field, values in results.items():
    print(f"\n  {field}:")
    for v in values:
        print(f"    {v!r}")

print("\n\n2. Redaction:")
redacted = extractor.redact(sample_document)
# Show only lines that changed
orig_lines = sample_document.strip().splitlines()
red_lines  = redacted.strip().splitlines()
for o, r in zip(orig_lines, red_lines):
    if o != r:
        print(f"  Before: {o.strip()!r}")
        print(f"  After:  {r.strip()!r}")
        print()
```

---

## 48.5 สรุป Part 48

```
findall() vs finditer():
findall()   → list of strings or tuples (all at once, memory)
finditer()  → iterator of Match objects (lazy, one at a time)

When to use finditer():
- Large text (memory efficient)
- Need position info (m.start(), m.end(), m.span())
- Early termination (next() or break)
- Need match groups AND positions

Match Object Key Methods:
m.group()           → entire match (= m.group(0))
m.group('name')     → named group
m.groups()          → tuple of all captured groups
m.groupdict()       → dict of named groups
m.start()           → start position
m.end()             → end position
m.span()            → (start, end)
m.expand(template)  → replace with \\g<name> references
m.string            → original string
m.re                → compiled pattern

Scanner pattern:
for token_type, pattern in PATTERNS:
    m = pattern.match(text, pos)
    if m:
        tokens.append(Token(token_type, m.group(), ...))
        pos = m.end()
        break

finditer() with pos:
pattern.finditer(text, pos)          → start from pos
pattern.finditer(text, pos, endpos)  → restricted range
```

---

*[← Part 47: Substitution & Replacement](part-47-substitution.md) | [→ Part 49: Atomic Groups & Possessive Quantifiers](part-49-atomic-possessive.md)*
