# Part 22: Lookahead & Lookbehind — Zero-Width Assertions

> **ระดับ:** กลาง-สูง | **เวลาเรียน:** ~80 นาที | **ข้อกำหนด:** Part 01-21

---

## 22.1 Zero-Width Assertions คืออะไร

```
Zero-Width = ตรวจสอบตำแหน่ง แต่ไม่ "กิน" characters

Position assertions:
^        beginning of string/line
$        end of string/line
\b       word boundary
\B       non-word boundary

Lookaround assertions:
(?=...)   Positive Lookahead   — ตามด้วย...
(?!...)   Negative Lookahead   — ไม่ตามด้วย...
(?<=...) Positive Lookbehind  — นำหน้าด้วย...
(?<!...) Negative Lookbehind  — ไม่ได้นำหน้าด้วย...

ทั้งหมดนี้ไม่บริโภค characters → ไม่อยู่ใน match result
```

---

## 22.2 Positive Lookahead (?=...)

```python
import re

# (?=...) ตรวจว่า "ตามด้วย" pattern
text = "price 100 baht, price list available, price 200 USD"
m = re.findall(r'price(?=\s+\d)', text)
print(f"'price' before number: {m}")

m = re.findall(r'\d+(?=\s+baht)', text)
print(f"Numbers before 'baht': {m}")

# Password validation
def validate_password(password: str) -> list:
    issues = []
    if not re.search(r'(?=.*[A-Z])', password):
        issues.append("ต้องมีตัวพิมพ์ใหญ่")
    if not re.search(r'(?=.*[a-z])', password):
        issues.append("ต้องมีตัวพิมพ์เล็ก")
    if not re.search(r'(?=.*\d)', password):
        issues.append("ต้องมีตัวเลข")
    if not re.search(r'(?=.*[!@#$%^&*])', password):
        issues.append("ต้องมีสัญลักษณ์พิเศษ")
    if len(password) < 8:
        issues.append("ต้องมีอย่างน้อย 8 ตัวอักษร")
    return issues

passwords = ["abc123", "Password1", "P@ssw0rd", "Short1!", "GoodPass1!"]
print("\nPassword Validation:")
for pwd in passwords:
    issues = validate_password(pwd)
    status = "OK" if not issues else ", ".join(issues)
    print(f"  {pwd:15} {'OK' if not issues else status}")
```

---

## 22.3 Negative Lookahead (?!...)

```python
import re

# (?!...) ตรวจว่า "ไม่ตามด้วย" pattern
text = "foobar, football, foo fighter, foolish"
m = re.findall(r'foo(?!bar)', text)
print(f"'foo' not before 'bar': {m}")

urls = "http://unsafe.com https://safe.com http://another-unsafe.com"
insecure = re.findall(r'http(?!s)://[\w.-]+', urls)
print(f"Insecure URLs: {insecure}")

# ค้นหาคำที่ไม่ใช่ keyword
KEYWORDS = {'if', 'else', 'for', 'while', 'def', 'class', 'return'}
keyword_pattern = '|'.join(re.escape(k) for k in KEYWORDS)
code = "if x > 0: return calculate(x) else: for item in items: define(item)"

identifiers = re.findall(
    rf'\b(?!(?:{keyword_pattern})\b)[a-zA-Z_]\w*\b',
    code
)
print(f"\nNon-keyword identifiers: {identifiers}")
```

---

## 22.4 Positive Lookbehind (?<=...)

```python
import re

# (?<=...) ตรวจว่า "นำหน้าด้วย" (fixed-width)
prices = "Cost: $100 and $200, Budget: €500"
usd_prices = re.findall(r'(?<=\$)\d+(?:\.\d{2})?', prices)
print(f"USD prices: {usd_prices}")

emails = "admin@example.com, user@test.org, info@company.co.th"
domains = re.findall(r'(?<=@)[\w.-]+', emails)
print(f"Domains: {domains}")

text = "Dr. Smith and Mr. Jones met Mrs. Brown at the hospital"
TITLE = re.compile(r'(?<=(?:Mr|Mrs|Ms|Dr)\.\s)\w+')
names = TITLE.findall(text)
print(f"Names: {names}")

qs = "?name=Alice&age=30&city=Bangkok"
values = re.findall(r'(?<=[?&]\w+=)[^&]+', qs)
print(f"Values: {values}")

try:
    re.compile(r'(?<=\w+)\d+')    # ERROR: variable-length
except re.error as e:
    print(f"\nError: {e}")
```

---

## 22.5 Negative Lookbehind (?<!...)

```python
import re

# (?<!...) ตรวจว่า "ไม่ได้นำหน้าด้วย"
text = "The cat chased the other there by the river"
the_words = re.findall(r'(?<!\w)the(?!\w)', text, re.IGNORECASE)
print(f"Standalone 'the': {the_words}")

ip = "IP: 192.168.1.1, port: 8080, code: 200"
non_ip_nums = re.findall(r'(?<![\d.])\b\d+\b(?![\d.])', ip)
print(f"Non-IP numbers: {non_ip_nums}")

paths = ["/home/user/file.txt", "../secret/file", "/var/log/app.log", "../../etc/passwd"]
print("\nPath validation:")
for path in paths:
    if re.search(r'(?<!\.)\.\.', path):
        print(f"  UNSAFE: {path}")
    else:
        print(f"  SAFE:   {path}")
```

---

## 22.6 Combining Lookarounds

```python
import re

STRONG_PASSWORD = re.compile(
    r'^'
    r'(?=.*[A-Z])'
    r'(?=.*[a-z])'
    r'(?=.*\d)'
    r'(?=.*[!@#$%^&*])'
    r'(?!.*\s)'
    r'(?!.*(.)(\1){2})'
    r'.{8,32}'
    r'$'
)

passwords = ["P@ssw0rd", "weakpass", "ALLCAPS1!", "P@ss   word1", "P@ssssword1", "Tr0ub4dor!"]
print("Strong Password Check:")
for pwd in passwords:
    ok = bool(STRONG_PASSWORD.match(pwd))
    print(f"  {'OK' if ok else 'FAIL'} {pwd!r}")

# Extract text between delimiters (not including)
text = "/* This is a comment */ and /* Another one */"
comments = re.findall(r'(?<=/\*\s?).*?(?=\s?\*/)', text, re.DOTALL)
print(f"\nComments: {[c.strip() for c in comments]}")

# Add commas to numbers
def add_commas(number_str: str) -> str:
    return re.sub(r'(?<=\d)(?=(\d{3})+(?!\d))', ',', number_str)

numbers = ['1234567', '9876543210', '123']
print("\nNumber formatting:")
for n in numbers:
    print(f"  {n} -> {add_commas(n)}")
```

---

## 22.7 JavaScript Lookaround

```javascript
// Positive lookahead
const text = "price 100 baht, price 200 USD";
const nums = text.match(/\d+(?=\s+baht)/g);
console.log('Before baht:', nums);  // ['100']

// Negative lookahead
const urls = "http://unsafe.com https://safe.com";
const insecure = urls.match(/http(?!s):\/\/[\w.-]+/g);
console.log('Insecure:', insecure);

// Positive lookbehind (ES2018+)
const emails = "admin@example.com user@test.org";
const domains = emails.match(/(?<=@)[\w.-]+/g);
console.log('Domains:', domains);

// Add commas to numbers
function addCommas(n) {
    return String(n).replace(/\B(?=(\d{3})+(?!\d))/g, ',');
}
console.log(addCommas(1234567));    // '1,234,567'

// Password validation
function validatePassword(pwd) {
    return [
        { rule: 'uppercase', ok: /(?=.*[A-Z])/.test(pwd) },
        { rule: 'lowercase', ok: /(?=.*[a-z])/.test(pwd) },
        { rule: 'digit',     ok: /(?=.*\d)/.test(pwd) },
        { rule: 'special',   ok: /(?=.*[!@#$%^&*])/.test(pwd) },
        { rule: 'length',    ok: pwd.length >= 8 },
    ];
}
console.log(validatePassword('P@ssw0rd'));
```

---

## 22.8 สรุป Part 22

```
Lookaround:
(?=...)   Positive lookahead  — ตามด้วย
(?!...)   Negative lookahead  — ไม่ตามด้วย
(?<=...)  Positive lookbehind — นำหน้าด้วย (fixed-width)
(?<!...)  Negative lookbehind — ไม่ได้นำหน้าด้วย

Use cases:
- Password validation: รวม lookaheads หลายตัว
- Insert separators: (?<=\d)(?=(\d{3})+(?!\d))
- Extract without delimiters: (?<=@)[\w.]+
- Split camelCase: (?<=[a-z])(?=[A-Z])
- URL security: http(?!s)://
```

---

*[← Part 21: Advanced Groups](part-21-advanced-groups.md) | [→ Part 23: Regex Performance](part-23-regex-performance.md)*
