# Part 01: บทนำ — Regex คืออะไร และทำไมต้องเรียน

> **ระดับ:** พื้นฐาน | **เวลาเรียน:** ~45 นาที | **ข้อกำหนด:** ไม่มี

---

## 1.1 Regex คืออะไร?

**Regular Expression** (หรือ Regex, RegExp, RE) คือภาษาขนาดเล็กสำหรับ **อธิบาย pattern ของข้อความ** โดยใช้สัญลักษณ์พิเศษในการระบุว่าต้องการค้นหา จับคู่ หรือแทนที่ข้อความแบบใด

### ตัวอย่างง่ายๆ

สมมติคุณต้องการตรวจสอบว่า email ถูกรูปแบบหรือไม่:

```
ข้อความ: "user@example.com"
Pattern: [a-z]+@[a-z]+\.[a-z]+
ผลลัพธ์: ✅ ตรงกัน (match)
```

```
ข้อความ: "notanemail"
Pattern: [a-z]+@[a-z]+\.[a-z]+
ผลลัพธ์: ❌ ไม่ตรง (no match)
```

---

## 1.2 ประวัติความเป็นมา

| ปี | เหตุการณ์ |
|----|----------|
| 1951 | Stephen Kleene นักคณิตศาสตร์สร้าง "Regular Sets" |
| 1968 | Ken Thompson นำ Regex มาใช้ใน `grep` ของ Unix |
| 1986 | POSIX กำหนดมาตรฐาน Regex |
| 1987 | Larry Wall สร้าง Perl พร้อม PCRE อันทรงพลัง |
| 1990s | Regex แพร่หลายในทุกภาษาโปรแกรม |
| 2000s | Regex เป็น standard ในทุก framework |

---

## 1.3 ทำไมต้องเรียน Regex?

### 🔴 ปัญหาที่พบในชีวิตจริง

**ก่อนใช้ Regex:**
```python
# ตรวจสอบ email แบบ manual (ยุ่งยากมาก)
def validate_email_bad(email):
    has_at = False
    has_dot_after_at = False
    at_position = -1
    
    for i, char in enumerate(email):
        if char == '@':
            if has_at:
                return False  # มี @ สองตัว
            has_at = True
            at_position = i
        elif char == '.' and at_position != -1:
            has_dot_after_at = True
    
    if not has_at:
        return False
    if not has_dot_after_at:
        return False
    if email.startswith('@'):
        return False
    if email.endswith('@'):
        return False
    # ... ยังมีอีกมาก
    return True

# โค้ด 30+ บรรทัด ยังไม่สมบูรณ์!
```

**หลังใช้ Regex:**
```python
import re

def validate_email_good(email):
    pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    return bool(re.match(pattern, email))

# 1 บรรทัด ครอบคลุมกว่ามาก!
```

---

## 1.4 Regex ใช้ทำอะไรได้บ้าง?

### ✅ การใช้งานหลัก

```
1. SEARCH    - ค้นหาข้อความที่ตรงกับ pattern
2. MATCH     - ตรวจสอบว่าข้อความตรงกับ pattern หรือไม่  
3. EXTRACT   - ดึงส่วนของข้อความออกมา
4. REPLACE   - แทนที่ข้อความ
5. SPLIT     - แบ่งข้อความตาม pattern
6. VALIDATE  - ตรวจสอบความถูกต้องของข้อมูล
```

### 📋 ตัวอย่างการใช้งานจริง

```python
import re

text = """
Contact us:
Email: support@company.com
Phone: +66-02-123-4567
Website: https://www.company.co.th
IP: 192.168.1.100
Date: 2026-10-15
"""

# 1. ค้นหา email ทั้งหมด
emails = re.findall(r'[\w.+-]+@[\w-]+\.[\w.]+', text)
print(f"Emails: {emails}")
# Output: Emails: ['support@company.com']

# 2. ค้นหาเบอร์โทร
phones = re.findall(r'\+?\d[\d\s\-]{8,}', text)
print(f"Phones: {phones}")
# Output: Phones: ['+66-02-123-4567']

# 3. ค้นหา URL
urls = re.findall(r'https?://[\w./\-]+', text)
print(f"URLs: {urls}")
# Output: URLs: ['https://www.company.co.th']

# 4. ค้นหา IP
ips = re.findall(r'\b\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}\b', text)
print(f"IPs: {ips}")
# Output: IPs: ['192.168.1.100']

# 5. ค้นหาวันที่
dates = re.findall(r'\d{4}-\d{2}-\d{2}', text)
print(f"Dates: {dates}")
# Output: Dates: ['2026-10-15']
```

---

## 1.5 Regex ใช้ในภาษาใดบ้าง?

| ภาษา | Syntax | หมายเหตุ |
|------|--------|---------|
| Python | `re.match(r'pattern', text)` | Built-in module `re` |
| JavaScript | `/pattern/flags` | Built-in syntax |
| PHP | `preg_match('/pattern/', $text)` | PCRE engine |
| Java | `Pattern.compile("pattern")` | `java.util.regex` |
| Go | `regexp.MustCompile("pattern")` | `regexp` package |
| Ruby | `/pattern/` | Built-in |
| Perl | `$text =~ /pattern/` | Origin ของ PCRE |
| Bash | `[[ "$text" =~ pattern ]]` | Built-in |
| SQL | `REGEXP`, `RLIKE`, `~` | ขึ้นกับ DB |
| .NET (C#) | `Regex.Match(text, "pattern")` | `System.Text.RegularExpressions` |

---

## 1.6 Regex Engine มีกี่ประเภท?

### ประเภทหลัก

```
┌─────────────────────────────────────────────────┐
│                 Regex Engines                   │
├────────────────┬────────────────────────────────┤
│   NFA-based    │     DFA-based                  │
│  (Backtracking)│  (Linear time)                 │
├────────────────┼────────────────────────────────┤
│ • PCRE         │ • RE2 (Google)                 │
│ • Python re    │ • Golang regexp                │
│ • Java         │ • grep (BRE/ERE)               │
│ • .NET         │                                │
│ • JavaScript   │                                │
│ • Perl         │                                │
└────────────────┴────────────────────────────────┘
```

**NFA (Non-deterministic Finite Automaton):**
- ยืดหยุ่นกว่า, รองรับ features มากกว่า
- อาจช้าถ้าเขียน pattern ไม่ดี (backtracking)

**DFA (Deterministic Finite Automaton):**
- เร็วกว่า, เวลา O(n) เสมอ
- รองรับ features น้อยกว่า (ไม่มี backreferences)

---

## 1.7 Regex Syntax เบื้องต้น (Preview)

ต่อไปนี้คือ building blocks ของ Regex (จะอธิบายละเอียดในส่วนต่อๆ ไป):

```
┌──────────────────────────────────────────────────────┐
│                 Regex Cheat Sheet                    │
├──────────┬───────────────────────────────────────────┤
│ Pattern  │ ความหมาย                                  │
├──────────┼───────────────────────────────────────────┤
│ .        │ ตัวอักษรใดก็ได้ (ยกเว้น newline)          │
│ \d       │ ตัวเลข [0-9]                              │
│ \w       │ ตัวอักษร/เลข/underscore [a-zA-Z0-9_]     │
│ \s       │ whitespace (space, tab, newline)           │
│ [abc]    │ a หรือ b หรือ c                           │
│ [^abc]   │ ไม่ใช่ a, b, หรือ c                      │
│ ^        │ จุดเริ่มต้นของบรรทัด                     │
│ $        │ จุดสิ้นสุดของบรรทัด                      │
│ *        │ 0 ครั้งขึ้นไป                             │
│ +        │ 1 ครั้งขึ้นไป                             │
│ ?        │ 0 หรือ 1 ครั้ง                            │
│ {n}      │ n ครั้งพอดี                               │
│ {n,m}    │ n ถึง m ครั้ง                             │
│ (abc)    │ Group: capture "abc"                      │
│ a|b      │ a หรือ b                                  │
│ (?=abc)  │ Lookahead: ตามด้วย abc                   │
│ (?!abc)  │ Negative lookahead: ไม่ตามด้วย abc       │
└──────────┴───────────────────────────────────────────┘
```

---

## 1.8 ตัวอย่างแรกของคุณ

เริ่มต้นด้วยตัวอย่างง่ายๆ ใน Python:

```python
import re

# ============================================
# ตัวอย่าง 1: Match ข้อความที่แน่นอน
# ============================================
text = "Hello, World!"
pattern = r"Hello"

if re.search(pattern, text):
    print("✅ พบคำว่า Hello")
else:
    print("❌ ไม่พบ")

# Output: ✅ พบคำว่า Hello


# ============================================
# ตัวอย่าง 2: ค้นหาตัวเลขทั้งหมด
# ============================================
text = "ราคา 1500 บาท ส่วนลด 200 บาท คงเหลือ 1300 บาท"
numbers = re.findall(r'\d+', text)
print(f"ตัวเลขที่พบ: {numbers}")

# Output: ตัวเลขที่พบ: ['1500', '200', '1300']


# ============================================
# ตัวอย่าง 3: แทนที่ข้อความ
# ============================================
text = "โทร 081-234-5678 หรือ 02-123-4567"
masked = re.sub(r'\d', 'X', text)
print(f"ปกปิดเลข: {masked}")

# Output: ปกปิดเลข: โทร XXX-XXX-XXXX หรือ XX-XXX-XXXX


# ============================================
# ตัวอย่าง 4: แยกข้อความ
# ============================================
data = "แอปเปิ้ล, มะม่วง; กล้วย | ส้ม"
fruits = re.split(r'[,;|]\s*', data)
print(f"ผลไม้: {fruits}")

# Output: ผลไม้: ['แอปเปิ้ล', 'มะม่วง', 'กล้วย', 'ส้ม']


# ============================================
# ตัวอย่าง 5: ดึงข้อมูลจาก Log
# ============================================
log = '2026-10-02 14:30:22 ERROR [auth] Login failed for user "admin" from 192.168.1.50'

# ดึงวันที่
date_match = re.search(r'(\d{4}-\d{2}-\d{2})', log)

# ดึง IP
ip_match = re.search(r'(\d{1,3}(?:\.\d{1,3}){3})', log)

# ดึง username
user_match = re.search(r'user "([^"]+)"', log)

if date_match:
    print(f"วันที่: {date_match.group(1)}")
if ip_match:
    print(f"IP: {ip_match.group(1)}")
if user_match:
    print(f"Username: {user_match.group(1)}")

# Output:
# วันที่: 2026-10-02
# IP: 192.168.1.50
# Username: admin
```

---

## 1.9 JavaScript: ตัวอย่างแรก

```javascript
// ============================================
// JavaScript Regex Examples
// ============================================

// ตัวอย่าง 1: Test pattern
const email = "user@example.com";
const emailPattern = /^[\w.+-]+@[\w-]+\.[\w.]+$/;
console.log(emailPattern.test(email)); // true

// ตัวอย่าง 2: ค้นหาทั้งหมด
const text = "Call us at 081-234-5678 or 02-999-8888";
const phones = text.match(/\d[\d-]{8,}/g);
console.log(phones); // ['081-234-5678', '02-999-8888']

// ตัวอย่าง 3: Replace
const dirty = "Hello   World    how   are  you";
const clean = dirty.replace(/\s+/g, ' ');
console.log(clean); // "Hello World how are you"

// ตัวอย่าง 4: Split
const csv = "name,age,city,country";
const headers = csv.split(/,/);
console.log(headers); // ['name', 'age', 'city', 'country']

// ตัวอย่าง 5: Named groups
const dateStr = "Today is 2026-10-02";
const dateRegex = /(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/;
const match = dateStr.match(dateRegex);
if (match) {
    const { year, month, day } = match.groups;
    console.log(`Year: ${year}, Month: ${month}, Day: ${day}`);
    // Year: 2026, Month: 10, Day: 02
}
```

---

## 1.10 PHP: ตัวอย่างแรก

```php
<?php
// ============================================
// PHP PCRE Regex Examples
// ============================================

$text = "สั่งซื้อสินค้าราคา 1,500.00 บาท จำนวน 3 ชิ้น";

// ตัวอย่าง 1: preg_match - ตรวจสอบ pattern
$pattern = '/\d[\d,]+\.\d{2}/';
if (preg_match($pattern, $text, $matches)) {
    echo "ราคา: " . $matches[0] . "\n";
    // ราคา: 1,500.00
}

// ตัวอย่าง 2: preg_match_all - ค้นหาทั้งหมด
$text2 = "รหัส SKU-001, SKU-042, SKU-999";
preg_match_all('/SKU-\d{3}/', $text2, $skus);
print_r($skus[0]);
// Array ( [0] => SKU-001 [1] => SKU-042 [2] => SKU-999 )

// ตัวอย่าง 3: preg_replace
$html = "<p>Hello <b>World</b></p>";
$plain = preg_replace('/<[^>]+>/', '', $html);
echo $plain . "\n";
// Hello World

// ตัวอย่าง 4: preg_split
$data = "apple::mango::banana::orange";
$fruits = preg_split('/:+/', $data);
print_r($fruits);
// Array ( [0] => apple [1] => mango [2] => banana [3] => orange )

// ตัวอย่าง 5: preg_replace_callback
$prices = "Items: $10, $25, $100";
$inflated = preg_replace_callback('/\$(\d+)/', function($m) {
    return '$' . ($m[1] * 1.1);  // เพิ่ม 10%
}, $prices);
echo $inflated . "\n";
// Items: $11, $27.5, $110
?>
```

---

## 1.11 เปรียบเทียบ Regex ระหว่าง Engine

```
ฟีเจอร์             PCRE    Python  Java    JS      Go/RE2
─────────────────────────────────────────────────────────
Lookahead           ✅      ✅      ✅      ✅      ✅
Lookbehind          ✅      ✅      ✅      ✅*     ✅*
Named groups        ✅      ✅      ✅      ✅      ✅
Backreferences      ✅      ✅      ✅      ✅      ❌
Atomic groups       ✅      ❌      ❌      ❌      ❌
Possessive quant.   ✅      ❌      ✅      ❌      ❌
Recursive patterns  ✅      ❌      ❌      ❌      ❌
Unicode properties  ✅      ✅      ✅      ✅      ✅
Conditional regex   ✅      ❌      ❌      ❌      ❌

* = limited support (fixed-width only)
```

---

## 1.12 Tools สำหรับฝึก Regex

### Online Tools

1. **regex101.com** ⭐⭐⭐⭐⭐
   - รองรับ PCRE, Python, JavaScript, Go
   - แสดง explanation ของ pattern
   - มี match information ละเอียด
   - มี debugger

2. **regexr.com** ⭐⭐⭐⭐
   - Interface สวย
   - มี Community patterns
   - Reference panel

3. **debuggex.com** ⭐⭐⭐⭐
   - Visualize Regex เป็น diagram
   - เห็น NFA graph

4. **regexone.com** ⭐⭐⭐⭐
   - Interactive tutorial
   - แบบฝึกหัดพร้อมเฉลย

### CLI Tools

```bash
# grep - ค้นหาในไฟล์
grep -E 'pattern' file.txt
grep -oP 'pattern' file.txt  # PCRE, -o = only matching

# sed - แทนที่
sed 's/pattern/replacement/g' file.txt

# awk - ประมวลผลข้อความ
awk '/pattern/ {print}' file.txt

# perl - PCRE เต็มรูปแบบ
perl -ne 'print if /pattern/' file.txt
echo "text" | perl -pe 's/pattern/replacement/g'
```

---

## 1.13 แบบฝึกหัด Part 01

### แบบฝึกหัดที่ 2: เขียน Python Code

```python
import re

text = """
Product: iPhone 15 Pro
Price: 45,900 บาท
Stock: 150 units
SKU: APP-IP15P-256-NTB
Date: 2026-09-15
Website: https://apple.com/th/iphone
Contact: sales@apple.co.th
"""

# เฉลย
prices = re.findall(r'\d{1,3}(?:,\d{3})+', text)
skus = re.findall(r'[A-Z]{2,}-[A-Z0-9]+-[A-Z0-9]+-[A-Z0-9]+', text)
dates = re.findall(r'\d{4}-\d{2}-\d{2}', text)
urls = re.findall(r'https?://\S+', text)
emails = re.findall(r'[\w.+-]+@[\w-]+\.[\w.]+', text)

print(f"Prices: {prices}")   # ['45,900']
print(f"SKUs: {skus}")       # ['APP-IP15P-256-NTB']
print(f"Dates: {dates}")     # ['2026-09-15']
print(f"URLs: {urls}")       # ['https://apple.com/th/iphone']
print(f"Emails: {emails}")   # ['sales@apple.co.th']
```

---

## 1.14 สรุป Part 01

✅ Regex คือภาษาอธิบาย pattern ของข้อความ  
✅ ใช้ได้ในทุกภาษาโปรแกรม  
✅ มีประโยชน์ในการ Search, Match, Extract, Replace, Validate  
✅ มี NFA และ DFA engine ที่แตกต่างกัน  
✅ Tools: regex101.com, grep, sed, python re module  

---

## 📌 Reference

```
Functions ที่ใช้บ่อยใน Python re:
─────────────────────────────────────────
re.match()       - จับตั้งแต่ต้นสตริง
re.search()      - ค้นหาที่ไหนก็ได้
re.findall()     - ค้นหาทั้งหมด, return list
re.finditer()    - ค้นหาทั้งหมด, return iterator
re.sub()         - แทนที่
re.subn()        - แทนที่ + นับครั้ง
re.split()       - แบ่งข้อความ
re.compile()     - compile pattern ล่วงหน้า
re.fullmatch()   - ต้องตรงทั้งหมด
re.escape()      - escape metacharacters
```

---

*[⬅ กลับไป README](../README.md) | [➡ ต่อไป Part 02: Basic Characters](part-02-basic-characters.md)*
