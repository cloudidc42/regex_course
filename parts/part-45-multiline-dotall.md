# Part 45: Multiline, DOTALL & Verbose Mode Flags

> **ระดับ:** กลาง | **เวลาเรียน:** ~60 นาที | **ข้อกำหนด:** Part 01-44

---

## 45.1 re.MULTILINE — ^ และ $ บน แต่ละบรรทัด

```python
import re

print("re.MULTILINE Flag:")
print("=" * 60)

text = """line one
line two
line three
first line again"""

# ===== Without MULTILINE =====
# ^ matches only start of string
# $ matches only end of string

print("\n1. Without MULTILINE (default):")
# ^ matches only start of entire string
starts = re.findall(r'^\w+', text)
ends   = re.findall(r'\w+$', text)
print(f"   ^\\w+  : {starts}")   # only 'line' at very start
print(f"   \\w+$  : {ends}")     # only last word

# ===== With MULTILINE =====
# ^ matches start of EACH line
# $ matches end of EACH line

print("\n2. With re.MULTILINE:")
starts_ml = re.findall(r'^\w+', text, re.MULTILINE)
ends_ml   = re.findall(r'\w+$', text, re.MULTILINE)
print(f"   ^\\w+  : {starts_ml}")   # first word on each line
print(f"   \\w+$  : {ends_ml}")     # last word on each line

# ===== Practical: extract numbered list items =====
print("\n3. Extract numbered list items:")
numbered = """1. First item
2. Second item
3. Third item
   3a. Sub-item
4. Fourth item"""

# Match lines starting with a number
items = re.findall(r'^\s*(\d+[a-z]?)\.\s+(.+)', numbered, re.MULTILINE)
for num, text_item in items:
    print(f"   {num}: {text_item}")

# ===== Extract section headers =====
print("\n4. Extract markdown headers:")
markdown = """# Title

## Section 1
Some content here.

## Section 2
More content.

### Subsection
Details here.

## Section 3
Final section."""

headers = re.findall(r'^(#{1,6})\s+(.+)', markdown, re.MULTILINE)
for level, title in headers:
    indent = "  " * (len(level) - 1)
    print(f"   {indent}H{len(level)}: {title}")

# ===== Remove leading whitespace from each line =====
print("\n5. Per-line substitution:")
indented = """    def hello():
        print(\"Hello\")
        return True"""

# Remove 4 spaces from start of each line
dedented = re.sub(r'^ {4}', '', indented, flags=re.MULTILINE)
print(f"   Before:\n{indented}")
print(f"\n   After (4-space dedent):\n{dedented}")
```

---

## 45.2 re.DOTALL — . จับได้ทุกอย่างรวม newline

```python
import re

print("\nre.DOTALL Flag:")
print("=" * 60)

html = """<html>
<head>
  <title>Test Page</title>
</head>
<body>
  <div class=\"content\">
    <p>First paragraph.</p>
    <p>Second paragraph
    with newline.</p>
  </div>
</body>
</html>"""

# ===== Without DOTALL =====
print("\n1. Without DOTALL:")
# .* does NOT match newline
without = re.findall(r'<p>(.*?)</p>', html)
print(f"   <p>.*?</p>  : {without}")  # misses multi-line p

# ===== With DOTALL =====
print("\n2. With re.DOTALL:")
with_dotall = re.findall(r'<p>(.*?)</p>', html, re.DOTALL)
print(f"   <p>.*?</p>  : {with_dotall}")  # captures multi-line

# ===== Alternative: [\\s\\S]* (works without DOTALL) =====
print("\n3. Alternative: [\\s\\S]* (no flag needed):")
# [\s\S] matches any char including newline
alt_result = re.findall(r'<p>([\s\S]*?)</p>', html)
print(f"   <p>[\\s\\S]*?</p>: {alt_result}")

# ===== Extract multi-line blocks =====
print("\n4. Extract multi-line code blocks:")
markdown_with_code = """
# Tutorial

Here is an example:

```python
def greet(name):
    return f\"Hello, {name}!\"
```

And another:

```javascript
function greet(name) {
    return `Hello, ${name}!`;
}
```
"""

code_blocks = re.findall(r'```(\w+)\n(.*?)```', markdown_with_code, re.DOTALL)
for lang, code in code_blocks:
    print(f"\n   Language: {lang}")
    print(f"   Code:")
    for line in code.splitlines():
        print(f"     {line}")

# ===== DOTALL with anchors =====
print("\n5. Extract between delimiters (DOTALL):")
log = """
--- BEGIN CERTIFICATE ---
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA
n3Y3eT+VQ8YdXzUFNvjuePMBwkJHqA4n6r1m8TwWX
--- END CERTIFICATE ---

--- BEGIN PRIVATE KEY ---
MIIEvQIBADANBgkqhkiG9w0BAQEFAASCBK
--- END PRIVATE KEY ---
"""

certs = re.findall(
    r'--- BEGIN (\w+(?:\s+\w+)*) ---\n(.*?)\n--- END \\1 ---',
    log, re.DOTALL
)
for cert_type, content in certs:
    lines = content.strip().splitlines()
    print(f"\n   Type: {cert_type}")
    print(f"   Lines: {len(lines)}")
    print(f"   First: {lines[0][:30]}...")
```

---

## 45.3 re.VERBOSE — อ่านง่ายด้วย Comments

```python
import re

print("\nre.VERBOSE (re.X) Flag:")
print("=" * 60)

print("\n1. Complex pattern: compact vs verbose:")

# Compact (hard to read)
EMAIL_COMPACT = re.compile(
    r'^[a-zA-Z0-9!#$%&\'*+/=?^_`{|}~.-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
)

# Verbose (readable)
EMAIL_VERBOSE = re.compile(r'''
    ^                    # start of string
    [a-zA-Z0-9            # alphanumeric
     !#$%&\'*+/=?^_`     # allowed special chars (part 1)
     {|}~.\-              # allowed special chars (part 2)
    ]+                   # one or more chars
    @                    # @ separator
    [a-zA-Z0-9.\-]+      # domain name
    \.                   # literal dot
    [a-zA-Z]{2,}         # TLD: 2+ letters
    $                    # end of string
''', re.VERBOSE)

# ===== URL parser =====
print("\n2. URL parser (verbose):")
URL_PATTERN = re.compile(r'''
    ^
    (?P<scheme>https?|ftp)   # protocol
    ://                      # separator
    (?:                      # optional auth
        (?P<user>[^:@]+)     #   username
        (?::(?P<password>[^@]+))?  # optional password
        @
    )?
    (?P<host>                # hostname
        (?:[a-zA-Z0-9\-]+\.)+ # subdomains
        [a-zA-Z]{2,}         # TLD
    )
    (?::(?P<port>\d+))?      # optional port
    (?P<path>/[^?#]*)?       # optional path
    (?:\?(?P<query>[^#]*))?  # optional query string
    (?:\#(?P<fragment>.*))?  # optional fragment
    $
''', re.VERBOSE)

urls = [
    'https://api.example.com/v1/users',
    'https://user:pass@db.example.com:5432/mydb',
    'http://localhost:8080/api?key=value&page=1',
    'ftp://files.example.com/pub/data.tar.gz',
    'https://example.com/path#section1',
]

for url in urls:
    m = URL_PATTERN.match(url)
    if m:
        d = {k: v for k, v in m.groupdict().items() if v}
        print(f"\n   {url}")
        for k, v in d.items():
            print(f"     {k:<10}: {v}")

# ===== Date/Time parser =====
print("\n3. Date/Time parser (verbose):")
DATETIME_PATTERN = re.compile(r'''
    (?P<year>  \d{4})          # year
    [-/.]                       # separator
    (?P<month> 0[1-9]|1[0-2])  # month 01-12
    [-/.]                       # separator
    (?P<day>   0[1-9]|[12]\d|3[01])  # day 01-31
    (?:                         # optional time
        [T\s]                   # separator
        (?P<hour>   [01]\d|2[0-3])  # hour 00-23
        :
        (?P<minute> [0-5]\d)    # minute 00-59
        (?:
            :
            (?P<second> [0-5]\d)  # second 00-59
            (?:\.(?P<micro> \d+))?  # optional microseconds
        )?
        (?:\s*(?P<tz> [+-]\d{2}:?\d{2}|Z|UTC))?  # optional timezone
    )?
''', re.VERBOSE)

datetimes = [
    '2024-10-15',
    '2024/01/01T09:30:00',
    '2024-12-31 23:59:59.123456',
    '2024-10-15T08:00:00+07:00',
    '2024-10-15T00:00:00Z',
]

for dt in datetimes:
    m = DATETIME_PATTERN.match(dt)
    if m:
        d = {k: v for k, v in m.groupdict().items() if v}
        print(f"\n   {dt}")
        print(f"   → {d}")
```

---

## 45.4 Combining Flags

```python
import re

print("\nCombining Multiple Flags:")
print("=" * 60)

# ===== Flag combinations =====
print("\n1. re.MULTILINE | re.IGNORECASE:")
text = """ERROR: file not found
info: server started
WARNING: high memory usage
ERROR: connection refused"""

# Case-insensitive match at start of each line
errors = re.findall(r'^(?:error|warning):\s+(.+)', text, re.MULTILINE | re.IGNORECASE)
print(f"   Errors/Warnings: {errors}")


print("\n2. re.DOTALL | re.MULTILINE:")
config = """
# Server config
[server]
host = localhost  # local
port = 8080

[database]
url = postgresql://localhost/db
# Connection pool
pool = 5
"""

# Extract sections (any content, any number of lines)
sections = re.findall(
    r'^\[(\w+)\]\n(.*?)(?=^\[|\Z)',
    config,
    re.MULTILINE | re.DOTALL
)
for name, content in sections:
    lines = [l.strip() for l in content.splitlines() if l.strip() and not l.strip().startswith('#')]
    print(f"\n   [{name}]:")
    for line in lines:
        print(f"     {line}")


print("\n3. Inline flags (?imsx):")
# Inline flags don't need re.X etc.
# (?i) = IGNORECASE, (?m) = MULTILINE, (?s) = DOTALL, (?x) = VERBOSE

# (?i) inline
result = re.findall(r'(?i)python', 'Python PYTHON python pYtHoN')
print(f"   (?i)python: {result}")

# (?m) inline  
text2 = "line1\nline2\nline3"
starts = re.findall(r'(?m)^\w+', text2)
print(f"   (?m)^\\w+: {starts}")

# (?x) inline with comments
pattern = re.compile(r'''(?x)
    (\d{4})   # year
    -
    (\d{2})   # month
    -
    (\d{2})   # day
''')
m = pattern.match('2024-10-15')
if m:
    print(f"   (?x) date: year={m.group(1)}, month={m.group(2)}, day={m.group(3)}")

# Per-group inline flags
text3 = "Hello World HELLO WORLD hello world"
# (?i:...) applies IGNORECASE only to that group
mixed = re.findall(r'(?i:hello)\s+(?-i:world)', text3)  # hello (any case) + world (lowercase)
print(f"   (?i:hello)\\s+world: {mixed}")


print("\n4. Flag Table:")
FLAG_TABLE = [
    ("re.IGNORECASE", "re.I",  "(?i)", "Case-insensitive matching"),
    ("re.MULTILINE",  "re.M",  "(?m)", "^ and $ match each line"),
    ("re.DOTALL",     "re.S",  "(?s)", ". matches newline"),
    ("re.VERBOSE",    "re.X",  "(?x)", "Ignore whitespace, allow comments"),
    ("re.ASCII",      "re.A",  "(?a)", "\\w \\d \\s match ASCII only"),
    ("re.UNICODE",    "re.U",  "(?u)", "\\w \\d \\s match Unicode (default)"),
    ("re.LOCALE",     "re.L",  "(?L)", "Locale-dependent matching"),
]

print(f"\n  {'Full Name':<20} {'Short':<8} {'Inline':<8} {'Description'}")
print(f"  {'-'*20} {'-'*8} {'-'*8} {'-'*40}")
for full, short, inline, desc in FLAG_TABLE:
    print(f"  {full:<20} {short:<8} {inline:<8} {desc}")
```

---

## 45.5 สรุป Part 45

```
Flags Quick Reference:

re.MULTILINE (re.M, (?m)):
- ^ → ตรงกับ start ของ ทุก line
- $ → ตรงกับ end ของ ทุก line
- ใช้กับ: parse config, extract headers, per-line substitution

re.DOTALL (re.S, (?s)):
- . → ตรงกับ ทุก character รวม \\n
- ใช้กับ: multi-line HTML/XML, block extraction
- Alternative: [\\s\\S] (ไม่ต้องใช้ flag)

re.VERBOSE (re.X, (?x)):
- whitespace ถูก ignore (ใช้ indent ได้)
- # = comment ถึงปลายบรรทัด
- ใช้กับ: complex patterns ที่ต้องอธิบาย

re.IGNORECASE (re.I, (?i)):
- ตรง a-z กับ A-Z
- ใช้กับ: case-insensitive search

Combining: re.MULTILINE | re.DOTALL | re.IGNORECASE

Inline flags (?i), (?m), (?s) - apply mid-pattern
Per-group: (?i:pattern) applies only to that group
Turn off:  (?-i:pattern)

When to use DOTALL vs [\\s\\S]:
- re.DOTALL: cleaner, but affects entire pattern
- [\\s\\S]: explicit, works per-character
- (?s:...): applies DOTALL only to part of pattern
```

---

*[← Part 44: Unicode & Internationalization](part-44-unicode.md) | [→ Part 46: Named Groups & Backreferences Advanced](part-46-named-groups.md)*
