# Part 20: Search & Replace — การค้นหาและแทนที่

> **ระดับ:** กลาง | **เวลาเรียน:** ~60 นาที | **ข้อกำหนด:** Part 01-19

---

## 20.1 re.sub() พื้นฐาน

```python
import re

# re.sub(pattern, repl, string, count=0, flags=0)

text = "Hello World! Hello Python!"
result = re.sub(r'Hello', 'Hi', text)
print(f"Basic: {result}")
# -> "Hi World! Hi Python!"

# แทนที่ครั้งเดียว
result = re.sub(r'Hello', 'Hi', text, count=1)
print(f"Count=1: {result}")
# -> "Hi World! Hello Python!"

# Ignore case
result = re.sub(r'hello', 'Hi', text, flags=re.IGNORECASE)
print(f"Ignore case: {result}")

# Capture groups
phone = "โทร: 089-123-4567"
result = re.sub(r'(\d{3})-(\d{3})-(\d{4})', r'(\1) \2-\3', phone)
print(f"Phone format: {result}")
# -> "โทร: (089) 123-4567"

# Named groups
date_str = "วันที่: 02/10/2026"
result = re.sub(
    r'(?P<d>\d{2})/(?P<m>\d{2})/(?P<y>\d{4})',
    r'\g<y>-\g<m>-\g<d>',
    date_str
)
print(f"Date reformat: {result}")
# -> "วันที่: 2026-10-02"
```

---

## 20.2 Callable Replacement (Lambda)

```python
import re

# ใช้ function เป็น replacement

# Capitalize words
text = "hello world foo bar"
result = re.sub(r'\b\w+\b', lambda m: m.group(0).capitalize(), text)
print(f"Capitalize: {result}")
# -> "Hello World Foo Bar"

# แปลงเลขอารบิก -> อักษรไทย
THAI_NUMS = ['ศูนย์', 'หนึ่ง', 'สอง', 'สาม', 'สี่', 'ห้า', 'หก', 'เจ็ด', 'แปด', 'เก้า']
def to_thai_num(m):
    try:
        return THAI_NUMS[int(m.group(0))]
    except (ValueError, IndexError):
        return m.group(0)

result = re.sub(r'\d', to_thai_num, "รหัส: 12345")
print(f"Thai numbers: {result}")
# -> "รหัส: หนึ่งสองสามสี่ห้า"

# Markdown bold -> HTML
md = "This is **bold text** and __also bold__ here"
result = re.sub(r'\*\*(.+?)\*\*|__(.+?)__', lambda m: f'<b>{m.group(1) or m.group(2)}</b>', md)
print(f"MD to HTML: {result}")

# Count replacements
count = [0]
def replace_and_count(m):
    count[0] += 1
    return m.group(0).upper()

result = re.sub(r'\b[a-z]+\b', replace_and_count, "hello world foo")
print(f"Result: {result}, Count: {count[0]}")
```

---

## 20.3 Advanced Substitution Patterns

```python
import re

class TextTransformer:
    """Text transformation utilities"""
    
    @staticmethod
    def camel_to_snake(text: str) -> str:
        text = re.sub(r'([A-Z]+)([A-Z][a-z])', r'\1_\2', text)
        text = re.sub(r'([a-z0-9])([A-Z])', r'\1_\2', text)
        return text.lower()
    
    @staticmethod
    def snake_to_camel(text: str) -> str:
        components = text.split('_')
        return components[0] + ''.join(x.title() for x in components[1:])
    
    @staticmethod
    def snake_to_pascal(text: str) -> str:
        return ''.join(word.capitalize() for word in text.split('_'))
    
    @staticmethod
    def kebab_to_snake(text: str) -> str:
        return text.replace('-', '_')
    
    @staticmethod
    def normalize_whitespace(text: str) -> str:
        return re.sub(r'\s+', ' ', text).strip()
    
    @staticmethod
    def remove_accents(text: str) -> str:
        replacements = {
            r'[àáâãäå]': 'a', r'[èéêë]': 'e', r'[ìíîï]': 'i',
            r'[òóôõö]': 'o', r'[ùúûü]': 'u', r'[ýÿ]': 'y',
            r'[ÀÁÂÃÄÅ]': 'A', r'[ÈÉÊË]': 'E', r'[ÌÍÎÏ]': 'I',
            r'[ÒÓÔÕÖ]': 'O', r'[ÙÚÛÜ]': 'U', r'ñ': 'n', r'Ñ': 'N',
            r'[çÇ]': 'c',
        }
        for pattern, replacement in replacements.items():
            text = re.sub(pattern, replacement, text)
        return text
    
    @staticmethod
    def slug_from_title(title: str) -> str:
        slug = title.lower()
        slug = re.sub(r'[^\w\s-]', '', slug)
        slug = re.sub(r'[\s_-]+', '-', slug)
        return slug.strip('-')
    
    @staticmethod
    def truncate_text(text: str, max_len: int = 100, suffix: str = '...') -> str:
        if len(text) <= max_len:
            return text
        truncated = text[:max_len - len(suffix)]
        truncated = re.sub(r'\s+\S*$', '', truncated)
        return truncated + suffix
    
    @staticmethod
    def fix_smart_quotes(text: str) -> str:
        text = re.sub(r'[“”„‟«»]', '"', text)
        text = re.sub(r'[‘’‚‛‹›]', "'", text)
        return text
    
    @staticmethod
    def normalize_thai_text(text: str) -> str:
        text = re.sub(r'([฀-๿])\s+([฀-๿])', r'\1\2', text)
        return text


# ทดสอบ
print("Text Transformation:")
print("=" * 60)

cases = [
    ("getUserFullName", TextTransformer.camel_to_snake),
    ("get_user_full_name", TextTransformer.snake_to_camel),
    ("my-css-class-name", TextTransformer.kebab_to_snake),
    ("Hello   World   Test", TextTransformer.normalize_whitespace),
    ("Héllo Wörld café résumé", TextTransformer.remove_accents),
    ("My Amazing Blog Post Title!", TextTransformer.slug_from_title),
]

for orig, func in cases:
    result = func(orig)
    print(f"  {orig!r}")
    print(f"    -> {result!r}")
    print()

text = "This is a WARNING message with an ERROR inside"
highlighted = re.sub(r'\b(WARNING|ERROR)\b', r'**\1**', text)
print(f"Highlighted: {highlighted}")

long_text = "This is a very long text that needs to be truncated at a word boundary"
print(f"Truncated: {TextTransformer.truncate_text(long_text, max_len=40)!r}")
```

---

## 20.4 Template Engine (Regex-based)

```python
import re
from typing import Dict, Any

class SimpleTemplate:
    """Regex-based simple template engine"""
    
    VAR_PATTERN = re.compile(r'\{\{\s*(\w+)(?:\|([^}]*))?\s*\}\}')
    IF_PATTERN = re.compile(
        r'\{%\s*if\s+(\w+)\s*%\}(.*?)\{%\s*endif\s*%\}',
        re.DOTALL
    )
    FOR_PATTERN = re.compile(
        r'\{%\s*for\s+(\w+)\s+in\s+(\w+)\s*%\}(.*?)\{%\s*endfor\s*%\}',
        re.DOTALL
    )
    
    @classmethod
    def render(cls, template: str, context: Dict) -> str:
        result = template
        
        def render_for(m):
            var_name = m.group(1)
            list_name = m.group(2)
            body = m.group(3)
            items = context.get(list_name, [])
            return ''.join(cls.render(body, {**context, var_name: item}) for item in items)
        
        result = cls.FOR_PATTERN.sub(render_for, result)
        
        def render_if(m):
            condition = m.group(1)
            body = m.group(2)
            return cls.render(body, context) if context.get(condition) else ''
        
        result = cls.IF_PATTERN.sub(render_if, result)
        
        def replace_var(m):
            var_name = m.group(1)
            default = m.group(2) or ''
            value = context.get(var_name, default)
            return str(value) if value is not None else default
        
        result = cls.VAR_PATTERN.sub(replace_var, result)
        return result


# ทดสอบ
template = """
สวัสดี {{ name|ผู้ใช้ }}!

{% if is_premium %}
คุณเป็นสมาชิก Premium ระดับ {{ level }}
{% endif %}

รายการสินค้า:
{% for item in items %}
  - {{ item }}
{% endfor %}

ยอดรวม: ฿{{ total }}
"""

context = {
    'name': 'สมชาย',
    'is_premium': True,
    'level': 'Gold',
    'items': ['Python Book', 'Regex Course', 'Keyboard'],
    'total': '2,500',
}

output = SimpleTemplate.render(template, context)
print("Template Engine:")
print("=" * 60)
print(output)
```

---

## 20.5 Batch Find & Replace

```python
import re
from typing import List, Tuple

class BatchReplacer:
    """Perform multiple replacements efficiently"""
    
    def __init__(self, replacements: List[Tuple[str, str]]):
        self.replacements = [
            (re.compile(p, re.IGNORECASE), r)
            for p, r in replacements
        ]
    
    def apply(self, text: str) -> str:
        for pattern, replacement in self.replacements:
            text = pattern.sub(replacement, text)
        return text
    
    @classmethod
    def make_combined(cls, word_map: dict) -> 'BatchReplacer':
        """Single-pass replacer using combined alternation"""
        sorted_words = sorted(word_map.keys(), key=len, reverse=True)
        escaped = [re.escape(w) for w in sorted_words]
        combined_pattern = re.compile(
            r'\b(?:' + '|'.join(escaped) + r')\b',
            re.IGNORECASE
        )
        lower_map = {k.lower(): v for k, v in word_map.items()}
        
        class CombinedReplacer:
            def apply(self, text):
                def replace(m):
                    return lower_map.get(m.group(0).lower(), m.group(0))
                return combined_pattern.sub(replace, text)
        
        return CombinedReplacer()


# ทดสอบ
term_normalizer = BatchReplacer.make_combined({
    'กระทรวงสาธารณสุข': 'สธ.',
    'กระทรวงการคลัง': 'กค.',
    'กระทรวงศึกษาธิการ': 'ศธ.',
})

texts = [
    "กระทรวงสาธารณสุข ประกาศนโยบายใหม่",
    "กระทรวงการคลัง และ กระทรวงศึกษาธิการ ร่วมประชุม",
]

print("Batch Replacement:")
print("=" * 60)
for text in texts:
    result = term_normalizer.apply(text)
    print(f"  {text}")
    print(f"  -> {result}")
    print()

# Code refactoring
code_replacer = BatchReplacer([
    (r'\bprint\s*\(', 'logger.info('),
    (r'\bexcept\s+Exception\s*:', 'except Exception as e:'),
])

old_code = """
def process():
    print("Starting")
    try:
        result = do_work()
        print(f"Done: {result}")
    except Exception:
        print("Error occurred")
"""

print("Code Refactoring:")
print(code_replacer.apply(old_code))
```

---

## 20.6 Regex-based Formatter

```python
import re

class TextFormatter:
    """Format and normalize text using regex"""
    
    @staticmethod
    def normalize_phone_th(phone: str) -> str:
        digits = re.sub(r'[^\d+]', '', phone)
        digits = re.sub(r'^\+66', '0', digits)
        if re.match(r'^0[689]\d{8}$', digits):
            return f"{digits[0:3]}-{digits[3:6]}-{digits[6:]}"
        if re.match(r'^02\d{7,8}$', digits):
            return f"{digits[0:2]}-{digits[2:5]}-{digits[5:]}"
        return phone
    
    @staticmethod
    def format_credit_card(card: str, mask: bool = True) -> str:
        digits = re.sub(r'\D', '', card)
        if len(digits) != 16:
            return card
        if mask:
            return f"{digits[:4]} XXXX XXXX {digits[-4:]}"
        return f"{digits[:4]} {digits[4:8]} {digits[8:12]} {digits[12:]}"
    
    @staticmethod
    def add_thousands_separator(text: str) -> str:
        def format_num(m):
            num = m.group(0)
            if '.' in num or ',' in num:
                return num
            return f"{int(num):,}"
        return re.sub(r'\b\d{4,}\b', format_num, text)
    
    @staticmethod
    def mask_sensitive(text: str) -> str:
        # Email
        text = re.sub(
            r'\b([\w.+-]{1,3})[^@\s]*(@[\w-]+\.[a-zA-Z]{2,})\b',
            r'\1***\2', text
        )
        # Phone
        text = re.sub(r'\b(0\d)\d{4}(\d{4})\b', r'\1****\2', text)
        # Card numbers
        text = re.sub(
            r'\b(\d{4})\s?\d{4}\s?\d{4}\s?(\d{4})\b',
            r'\1 **** **** \2', text
        )
        return text


# ทดสอบ
print("Text Formatting:")
print("=" * 60)

phones = ["0891234567", "+66891234567", "089 123 4567", "02-2345678"]
print("\nPhone normalization:")
for p in phones:
    print(f"  {p:20} -> {TextFormatter.normalize_phone_th(p)}")

cards = ["4111111111111111", "5500 0000 0000 0004"]
print("\nCredit card masking:")
for c in cards:
    print(f"  {c:25} -> {TextFormatter.format_credit_card(c, mask=True)}")

pii_text = "Email: john.doe@example.com โทร: 0891234567 บัตร: 4111111111111111"
print(f"\nMask PII:")
print(f"  Original: {pii_text}")
print(f"  Masked:   {TextFormatter.mask_sensitive(pii_text)}")

text_with_nums = "มีผู้ใช้งาน 1234567 คน ยอดขาย 9876543 บาท"
print(f"\nNumber formatting:")
print(f"  Before: {text_with_nums}")
print(f"  After:  {TextFormatter.add_thousands_separator(text_with_nums)}")
```

---

## 20.7 JavaScript Search & Replace

```javascript
const TextUtils = {
    titleCase(text) {
        return text.replace(/\b\w+/g, w => w.charAt(0).toUpperCase() + w.slice(1).toLowerCase());
    },
    
    camelToSnake(str) {
        return str
            .replace(/([A-Z]+)([A-Z][a-z])/g, '$1_$2')
            .replace(/([a-z\d])([A-Z])/g, '$1_$2')
            .toLowerCase();
    },
    
    snakeToCamel(str) {
        return str.replace(/_([a-z])/g, (_, c) => c.toUpperCase());
    },
    
    highlight(text, query, tag = 'mark') {
        if (!query) return text;
        const escaped = query.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
        const regex = new RegExp(`(${escaped})`, 'gi');
        return text.replace(regex, `<${tag}>$1</${tag}>`);
    },
    
    interpolate(template, data) {
        return template.replace(/\{\{\s*(\w+)\s*\}\}/g, (_, key) =>
            data.hasOwnProperty(key) ? data[key] : ''
        );
    },
    
    slugify(text) {
        return text
            .toLowerCase()
            .replace(/[^\w\s-]/g, '')
            .replace(/[\s_-]+/g, '-')
            .replace(/^-+|-+$/g, '');
    },
    
    maskEmail(email) {
        return email.replace(/^([\w.+-]{1,3})[^@]*(@.+)$/, '$1***$2');
    },
    
    extractNumbers(text) {
        return (text.match(/-?\d+(?:\.\d+)?/g) || []).map(Number);
    }
};

// ทดสอบ
console.log(TextUtils.titleCase("hello world from javascript"));
console.log(TextUtils.camelToSnake("getUserFullName"));
console.log(TextUtils.snakeToCamel("get_user_full_name"));
console.log(TextUtils.highlight("Python Regex is fun", "regex"));
console.log(TextUtils.interpolate("Hello {{name}}!", {name: "Alice"}));
console.log(TextUtils.slugify("My Amazing Post! 2026"));
console.log(TextUtils.maskEmail("john.doe@example.com"));
console.log(TextUtils.extractNumbers("age: 28, score: 95.5"));
```

---

## 20.8 สรุป Part 20

```
Substitution patterns สำคัญ:
re.sub(r, r, s):       แทนที่ทั้งหมด
re.sub(r, r, s, 1):    แทนที่ครั้งแรก
re.sub(r, fn, s):      ใช้ function
\1, \g<name>:          back-reference ใน replacement
camelCase:             ([A-Z]+)([A-Z][a-z]) -> \1_\2
Slug:                  [^\w\s-] -> '' แล้ว [\s_-]+ -> -
Thousands:             \b(\d+)\b -> lambda m: f"{int(m.group(0)):,}"
Mask email:            ([\w.]{1,3})[^@]* -> \1***
```

---

*[← Part 19: Data Extraction](part-19-data-extraction.md) | [→ Part 21: Advanced Groups](part-21-advanced-groups.md)*
