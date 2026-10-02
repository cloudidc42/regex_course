# Part 10: Flags และ Modifiers

> **ระดับ:** พื้นฐาน | **เวลาเรียน:** ~45 นาที | **ข้อกำหนด:** Part 01-09

---

## 10.1 Flags คืออะไร?

**Flags** (หรือ Modifiers) คือตัวเลือกที่เปลี่ยน behavior ของ Regex engine

---

## 10.2 Python Flags

```python
import re

# re.IGNORECASE (re.I)
text = "Hello WORLD hello World"
matches = re.findall(r'hello', text, re.IGNORECASE)
print(matches)  # ['Hello', 'hello', 'Hello']

# re.MULTILINE (re.M): ^ และ $ match แต่ละบรรทัด
text2 = "First\nSecond\nThird"
lines = re.findall(r'^\w+', text2, re.MULTILINE)
print(lines)  # ['First', 'Second', 'Third']

# re.DOTALL (re.S): . match newline ด้วย
text3 = "Start\nMiddle\nEnd"
match = re.search(r'Start.+End', text3, re.DOTALL)
print(match.group() if match else None)

# re.VERBOSE (re.X)
EMAIL_PATTERN = re.compile(r"""
    ^                           # Start of string
    (?P<local>
        [a-zA-Z0-9]             # Must start with alphanumeric
        [a-zA-Z0-9._%+\-]{0,63} # Followed by allowed chars
    )
    @                           # @ separator
    (?P<domain>
        (?:
            [a-zA-Z0-9]
            [a-zA-Z0-9\-]{0,61}
            [a-zA-Z0-9]
            |
            [a-zA-Z0-9]
        )
        (?:
            \.
            (?:
                [a-zA-Z0-9]
                [a-zA-Z0-9\-]{0,61}
                [a-zA-Z0-9]
                |
                [a-zA-Z0-9]
            )
        )*
    )
    \.
    (?P<tld>
        [a-zA-Z]{2,10}
    )
    $
""", re.VERBOSE | re.IGNORECASE)

test_emails = ['user@example.com', 'user.name+tag@company.co.th', 'invalid@@email.com']
for email in test_emails:
    m = EMAIL_PATTERN.match(email)
    if m:
        d = m.groupdict()
        print(f"✅ {email} | local={d['local']}, tld={d['tld']}")
    else:
        print(f"❌ {email}")

# re.ASCII (re.A): จำกัด \w \d \s ให้เป็น ASCII
text4 = "ราคา 100 บาท"
ascii_d = re.findall(r'\d', text4, re.ASCII)
print("ASCII \\d:", ascii_d)  # ['1', '0', '0']
```

---

## 10.3 Inline Flags

```python
import re

# (?i) = IGNORECASE
text = "HTML and html and Html"
matches = re.findall(r'(?i)html', text)
print(matches)  # ['HTML', 'html', 'Html']

# Scoped inline flags (?flags:pattern) — Python 3.6+
text4 = "Hello WORLD"
matches4 = re.findall(r'(?i:hello) (?-i:WORLD)', text4)
print(matches4)  # ['Hello WORLD']
```

---

## 10.4 JavaScript Flags

```javascript
const text = "Hello WORLD hello World";

// i - case insensitive
console.log(text.match(/hello/gi));  // ['Hello', 'hello', 'Hello']

// g - global (find all)
console.log("a1b2c3".match(/\d/));   // ['1']
console.log("a1b2c3".match(/\d/g));  // ['1', '2', '3']

// m - multiline
const multiline = "line1\nline2\nline3";
console.log(multiline.match(/^\w+/gm));  // ['line1', 'line2', 'line3']

// s - dotAll (ES2018)
const dotall = "start\nmiddle\nend";
console.log(/start.+end/s.test(dotall));  // true

// u - unicode (ES2015)
const emoji = "Hello 🎉 World";
console.log(emoji.match(/\p{Emoji}/gu));  // ['🎉']
```

---

## 10.5 PHP PCRE Flags

```php
<?php
// i - case insensitive
preg_match_all('/hello/i', 'Hello HELLO hello', $m);
echo implode(', ', $m[0]) . "\n";  // Hello, HELLO, hello

// m - multiline
preg_match_all('/^\w+/m', "line1\nline2\nline3", $m);
echo implode(', ', $m[0]) . "\n";  // line1, line2, line3

// u - unicode
preg_match_all('/[\x{0E00}-\x{0E7F}]+/u', 'สวัสดี Hello ไทย', $m);
echo implode(', ', $m[0]) . "\n";  // สวัสดี, ไทย
?>
```

---

## 10.6 แบบฝึกหัด Part 10

```python
import re

config = """
# Database settings
DB_HOST = localhost
DB_PORT = 5432
DB_NAME = production
DB_USER = Admin
"""

config_pattern = re.compile(r"""
    ^               # start of line
    (?P<key>
        [A-Z_]+     # uppercase letters and underscore
    )
    \s*=\s*         # = with optional spaces
    (?P<value>
        [^\s#]+     # non-whitespace, non-comment
    )
    .*              # ignore rest
    $               # end of line
""", re.VERBOSE | re.MULTILINE)

config_data = {}
for m in config_pattern.finditer(config):
    config_data[m.group('key')] = m.group('value')

print("Config:")
for k, v in config_data.items():
    print(f"  {k} = {v}")
```

---

## 10.7 สรุป Part 10

```
Flag (Python)      Flag (JS)   ความหมาย
───────────────────────────────────────────────────
re.IGNORECASE (I)  i           case-insensitive
re.MULTILINE (M)   m           ^ $ match each line
re.DOTALL (S)      s           . matches newline
re.VERBOSE (X)     (none)      allow whitespace/comments
re.ASCII (A)       (none)      \w\d\s = ASCII only
(none)             g           global (find all)
(none)             y           sticky
```

**Inline flags:** `(?i)`, `(?m)`, `(?s)`, `(?x)`, `(?a)` ใน Python  

*[⬅ Part 09: Lookahead/Lookbehind](part-09-lookahead-lookbehind.md) | [➡ Part 11: Email Validation](part-11-email-validation.md)*
