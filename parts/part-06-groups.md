# Part 06: Groups และ Capturing

> **ระดับ:** พื้นฐาน | **เวลาเรียน:** ~75 นาที | **ข้อกำหนด:** Part 01-05

---

## 6.1 Groups คืออะไร?

**Group** คือการใช้ `( )` ล้อมรอบส่วนของ pattern เพื่อ:
1. **Capture** — บันทึกส่วนที่ match เพื่อใช้ภายหลัง
2. **Apply quantifiers** — ใช้ quantifier กับกลุ่มตัวอักษร
3. **Alternation** — ใช้ `|` ภายใน group
4. **Backreferences** — อ้างอิงสิ่งที่จับได้

---

## 6.2 Capturing Groups

```python
import re

text = "John Smith, age 30"
m = re.search(r'(\w+)\s+(\w+), age (\d+)', text)
if m:
    print(m.group(0))   # John Smith, age 30
    print(m.group(1))   # John
    print(m.group(2))   # Smith
    print(m.group(3))   # 30
    print(m.groups())   # ('John', 'Smith', '30')
    first, last, age = m.groups()
    print(f"Name: {first} {last}, Age: {age}")

# findall กับ groups
text2 = "Alice 25, Bob 30, Carol 28"
matches = re.findall(r'(\w+)\s+(\d+)', text2)
print(matches)  # [('Alice', '25'), ('Bob', '30'), ('Carol', '28')]
```

---

## 6.3 Non-Capturing Groups `(?:...)`

```python
import re

# ใช้ (?:...) เพื่อ apply quantifier กับกลุ่ม
text = "ha haha hahaha"
matches = re.findall(r'(?:ha)+', text)
print(matches)  # ['ha', 'haha', 'hahaha']
```

---

## 6.4 Named Groups `(?P<name>...)`

```python
import re

text = "Date: 2026-10-02 Time: 14:30:55"
m = re.search(
    r'Date:\s+(?P<year>\d{4})-(?P<month>\d{2})-(?P<day>\d{2})'
    r'\s+Time:\s+(?P<hour>\d{2}):(?P<minute>\d{2}):(?P<second>\d{2})',
    text
)

if m:
    print(m.group('year'))    # 2026
    d = m.groupdict()
    print(d)
    # {'year': '2026', 'month': '10', 'day': '02', 
    #  'hour': '14', 'minute': '30', 'second': '55'}
```

---

## 6.5 Groups ใน sub() และ replace

```python
import re

# Reorder date format DD/MM/YYYY → YYYY-MM-DD
text = "Date: 02/10/2026 and 15/12/2025"
reformatted = re.sub(r'(\d{2})/(\d{2})/(\d{4})', r'\3-\2-\1', text)
print(reformatted)  # Date: 2026-10-02 and 2025-12-15

# Named group ใน replacement
text2 = "Name: Smith, John"
reformatted2 = re.sub(r'(?P<last>\w+), (?P<first>\w+)', r'\g<first> \g<last>', text2)
print(reformatted2)  # Name: John Smith
```

---

## 6.6 ตัวอย่างจริง: Product Catalog Parser

```python
import re
from dataclasses import dataclass
from typing import List, Optional

@dataclass
class Product:
    sku: str
    name: str
    price: float
    currency: str = "THB"
    stock: Optional[int] = None
    category: Optional[str] = None

def parse_product_text(text: str) -> List[Product]:
    product_pattern = re.compile(
        r'\[(?P<sku>[A-Z]+-\d+)\]\s+'
        r'(?P<name>[^\|]+?)\s*\|\s*'
        r'(?P<currency>[฿$€£])(?P<price>[\d,]+(?:\.\d{2})?)'
        r'(?:\s*\|\s*(?P<stock>\d+)\s*units?)?'
        r'(?:\s*\|\s*#(?P<category>\w+))?',
        re.MULTILINE
    )
    products = []
    for m in product_pattern.finditer(text):
        d = m.groupdict()
        price = float(d['price'].replace(',', ''))
        stock = int(d['stock']) if d['stock'] else None
        products.append(Product(
            sku=d['sku'],
            name=d['name'].strip(),
            price=price,
            currency={'฿': 'THB', '$': 'USD', '€': 'EUR', '£': 'GBP'}[d['currency']],
            stock=stock,
            category=d['category'],
        ))
    return products

product_text = """
Catalog 2026:
[LAPTOP-001] MacBook Pro 14" M4 | ฿75,000.00 | 15 units | #laptop
[PHONE-042]  iPhone 16 Pro Max | ฿49,900 | 30 units | #smartphone
[ACCS-099]   USB-C Hub 7-port | ฿1,500.00 | 100 units | #accessories
"""

products = parse_product_text(product_text)
for p in products:
    stock_str = f"Stock: {p.stock}" if p.stock else "Stock: N/A"
    print(f"  {p.sku:12} | {p.name:30} | {p.currency} {p.price:>10,.2f} | {stock_str}")
```

---

## 6.7 Template Engine

```python
import re

def simple_template(template: str, variables: dict) -> str:
    def replace_var(m):
        parts = m.group(1).split('|')
        var_name = parts[0].strip()
        modifier = parts[1].strip() if len(parts) > 1 else None
        value = variables.get(var_name)
        if value is None:
            if modifier and modifier not in ('upper', 'lower'):
                return modifier
            return f"{{MISSING:{var_name}}}"
        value = str(value)
        if modifier == 'upper':
            return value.upper()
        elif modifier == 'lower':
            return value.lower()
        return value
    return re.sub(r'\{\{([^}]+)\}\}', replace_var, template)

template = "Dear {{title}} {{name|upper}}, Order #{{order_id}}."
variables = {'title': 'Mr.', 'name': 'John Smith', 'order_id': 'ORD-2026-001'}
print(simple_template(template, variables))
# Dear Mr. JOHN SMITH, Order #ORD-2026-001.
```

---

## 6.8 สรุป Part 06

✅ **`(...)`** — Capturing group  
✅ **`(?:...)`** — Non-capturing group  
✅ **`(?P<name>...)`** — Named group (Python)  
✅ **`m.group(n)`** — ดึง group ที่ n  
✅ **`m.groups()`** — tuple ของทุก groups  
✅ **`m.groupdict()`** — dict ของ named groups  
✅ **`\n` ใน replacement** — backreference  

*[⬅ Part 05: Anchors](part-05-anchors.md) | [➡ Part 07: Alternation](part-07-alternation.md)*
