# Part 19: Data Extraction — การดึงข้อมูลจากข้อความ

> **ระดับ:** กลาง | **เวลาเรียน:** ~75 นาที | **ข้อกำหนด:** Part 01-18

---

## 19.1 Named Entity Extraction

```python
import re
from typing import List, Dict

class EntityExtractor:
    """Extract named entities from text using regex"""
    
    CURRENCY = re.compile(
        r'(?P<symbol>[\$€£¥฿]|THB|USD|EUR|GBP|JPY|BTC)\s*(?P<amount>[\d,]+(?:\.\d{1,2})?)'
        r'|(?P<amount2>[\d,]+(?:\.\d{1,2})?)\s*(?P<symbol2>บาท|ดอลลาร์|ยูโร)',
        re.IGNORECASE
    )
    
    # เบอร์บัตรประชาชนไทย (13 หลัก)
    THAI_ID = re.compile(r'\b(\d)-(\d{4})-(\d{5})-(\d{2})-(\d)\b')
    
    # ชื่อบุคคล (รูปแบบ Title + Name)
    THAI_PERSON = re.compile(
        r'\b(?:นาย|นาง(?:สาว)?|ด[ร.]|ศ\.|รศ\.|ผศ\.|คุณ)\s*'
        r'([ก-๙]+(?:\s+[ก-๙]+)*)',
        re.UNICODE
    )
    
    HASHTAG = re.compile(r'#(\w+)', re.UNICODE)
    MENTION = re.compile(r'@(\w+)')
    
    CREDIT_CARD = re.compile(
        r'\b(?:'
        r'4\d{3}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}'
        r'|5[1-5]\d{2}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}'
        r'|3[47]\d{2}[\s-]?\d{6}[\s-]?\d{5}'
        r')\b'
    )
    
    THAI_PLATE = re.compile(
        r'\b([ก-ฮ]{2,3})\s*(\d{1,4})\b'
        r'|\b(\d{1,4})\s*([ก-ฮ]{2,3})\b'
    )
    
    @classmethod
    def extract_all(cls, text: str) -> Dict:
        results = {
            'currencies': [],
            'hashtags': cls.HASHTAG.findall(text),
            'mentions': cls.MENTION.findall(text),
            'thai_ids': [],
            'thai_names': cls.THAI_PERSON.findall(text),
            'thai_plates': [],
        }
        
        for m in cls.CURRENCY.finditer(text):
            d = m.groupdict()
            symbol = d.get('symbol') or d.get('symbol2', '')
            amount = d.get('amount') or d.get('amount2', '')
            if amount:
                results['currencies'].append({
                    'raw': m.group(0),
                    'symbol': symbol,
                    'amount': float(amount.replace(',', '')),
                })
        
        for m in cls.THAI_ID.finditer(text):
            id_num = ''.join(m.groups())
            results['thai_ids'].append(id_num)
        
        for m in cls.THAI_PLATE.finditer(text):
            plate = ' '.join(g for g in m.groups() if g)
            results['thai_plates'].append(plate)
        
        return results
    
    @classmethod
    def redact(cls, text: str) -> str:
        """Redact sensitive data from text"""
        text = cls.CREDIT_CARD.sub('[CARD-REDACTED]', text)
        text = cls.THAI_ID.sub(r'\1-XXXX-XXXXX-XX-\5', text)
        text = re.sub(r'\b[\w.+-]+@[\w-]+\.[a-zA-Z]{2,}\b', '[EMAIL-REDACTED]', text)
        text = re.sub(r'\b(?:\+?66|0)[6-9]\d[\-\s.]?\d{3}[\-\s.]?\d{4}\b', '[PHONE-REDACTED]', text)
        return text


# ทดสอบ
test_texts = [
    "ราคาสินค้า $99.99 และ ฿1,500.00",
    "ติดต่อ @johndoe หรือ #Python #Regex",
    "เลขบัตร 3-1001-00123-45-6 สั่งซื้อแล้ว",
    "ยินดีต้อนรับ นาย สมชาย ใจดี และ ดร. สมหญิง รักดี",
    "ทะเบียน กข 1234 จอดอยู่",
]

print("Entity Extraction:")
print("=" * 60)
for text in test_texts:
    entities = EntityExtractor.extract_all(text)
    print(f"\n  Text: {text}")
    for key, values in entities.items():
        if values:
            if key == 'currencies':
                print(f"  {key}: {[f\"{v['symbol']}{v['amount']}\" for v in values]}")
            else:
                print(f"  {key}: {values}")

# Redaction test
sensitive_text = "ชื่อ: สมชาย โทร: 089-123-4567 บัตร: 4111 1111 1111 1111 เลขบัตร: 3-1001-00123-45-6"
print(f"\nOriginal: {sensitive_text}")
print(f"Redacted: {EntityExtractor.redact(sensitive_text)}")
```

---

## 19.2 Key-Value Extraction

```python
import re
from typing import Dict

class KeyValueExtractor:
    """Extract key-value pairs from various formats"""
    
    CONFIG = re.compile(
        r'^\s*(?P<key>[^=:\s#][^=:\s]*)\s*[=:]\s*(?P<value>[^#\n]+?)(?:\s*#.*)?$',
        re.MULTILINE
    )
    
    INI_SECTION = re.compile(r'^\[(?P<section>[^\]]+)\]', re.MULTILINE)
    QUERY_STRING = re.compile(r'([^&=\s]+)=([^&\s]*)')
    HTTP_HEADER = re.compile(r'^(?P<name>[\w-]+):\s+(?P<value>.+)$', re.MULTILINE)
    
    @classmethod
    def parse_config(cls, config_text: str) -> Dict:
        result = {}
        current_section = 'default'
        
        for line in config_text.split('\n'):
            section_m = cls.INI_SECTION.match(line)
            if section_m:
                current_section = section_m.group('section')
                result.setdefault(current_section, {})
                continue
            
            kv_m = cls.CONFIG.match(line)
            if kv_m:
                key = kv_m.group('key').strip()
                value = kv_m.group('value').strip().strip('"\'')
                result.setdefault(current_section, {})
                result[current_section][key] = value
        
        return result
    
    @classmethod
    def parse_query_string(cls, qs: str) -> Dict:
        from urllib.parse import unquote_plus
        params = {}
        for m in cls.QUERY_STRING.finditer(qs.lstrip('?')):
            key = unquote_plus(m.group(1))
            value = unquote_plus(m.group(2))
            if key in params:
                if not isinstance(params[key], list):
                    params[key] = [params[key]]
                params[key].append(value)
            else:
                params[key] = value
        return params
    
    @classmethod
    def parse_http_headers(cls, headers_text: str) -> Dict:
        headers = {}
        for m in cls.HTTP_HEADER.finditer(headers_text):
            name = m.group('name').lower()
            value = m.group('value').strip()
            headers[name] = value
        return headers


# ทดสอบ
config_text = """
# Application configuration
[server]
host = 0.0.0.0
port = 8080
debug = false

[database]
host = localhost
port = 5432
name = myapp_db

[cache]
backend = redis
url = redis://localhost:6379/0
timeout = 300
"""

config = KeyValueExtractor.parse_config(config_text)
print("Config Parsing:")
print("=" * 60)
for section, values in config.items():
    if values:
        print(f"\n  [{section}]")
        for k, v in values.items():
            print(f"    {k} = {v}")

qs = "?name=John+Doe&age=30&lang=Python&lang=JavaScript"
params = KeyValueExtractor.parse_query_string(qs)
print(f"\nQuery String:")
for k, v in params.items():
    print(f"  {k}: {v}")

headers_text = """
Content-Type: application/json
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9
X-Request-ID: req-abc-123
"""
headers = KeyValueExtractor.parse_http_headers(headers_text)
print(f"\nHTTP Headers:")
for k, v in headers.items():
    print(f"  {k}: {v[:50]}")
```

---

## 19.3 Table Data Extraction from Text

```python
import re
from typing import List, Dict

class TextTableExtractor:
    """Extract tabular data from plain text"""
    
    MD_SEPARATOR = re.compile(r'^\|[-|\s:]+\|$')
    MD_ROW = re.compile(r'^\|(.+)\|$')
    CSV_PATTERN = re.compile(r'(?:"([^"]*)"|((?:[^,\n]*)))(?:,|$)')
    
    @classmethod
    def parse_markdown_table(cls, text: str) -> List[Dict]:
        lines = text.strip().split('\n')
        headers = []
        data = []
        header_parsed = False
        
        for line in lines:
            line = line.strip()
            if not line.startswith('|'):
                continue
            
            if cls.MD_SEPARATOR.match(line):
                header_parsed = True
                continue
            
            m = cls.MD_ROW.match(line)
            if m:
                cells = [c.strip() for c in m.group(1).split('|')]
                if not header_parsed and not headers:
                    headers = cells
                elif header_parsed:
                    if headers:
                        row = dict(zip(headers, cells))
                    else:
                        row = {f'col{i}': v for i, v in enumerate(cells)}
                    data.append(row)
        
        return data
    
    @classmethod
    def parse_csv_line(cls, line: str) -> List[str]:
        fields = []
        for m in cls.CSV_PATTERN.finditer(line):
            quoted = m.group(1)
            unquoted = m.group(2)
            if quoted is not None:
                fields.append(quoted)
            elif unquoted is not None:
                fields.append(unquoted.strip())
        if fields and fields[-1] == '':
            fields.pop()
        return fields


# ทดสอบ
md_table = """
| ชื่อ      | อายุ | ตำแหน่ง        | เงินเดือน  |
|-----------|------|----------------|------------|
| สมชาย    | 28   | Developer      | 45,000     |
| สมหญิง   | 32   | Data Scientist | 65,000     |
| สมศรี    | 25   | Designer       | 40,000     |
"""

rows = TextTableExtractor.parse_markdown_table(md_table)
print("Markdown Table:")
print("=" * 60)
for row in rows:
    print(f"  {row}")

csv_lines = [
    'John,Doe,30,"New York, NY",Developer',
    '"Smith, Jr.",Jane,25,"Los Angeles, CA","Data Scientist"',
]

print("\nCSV Parsing:")
for line in csv_lines:
    fields = TextTableExtractor.parse_csv_line(line)
    print(f"  {fields}")
```

---

## 19.4 Extract Numbers and Measurements

```python
import re
from typing import List, Dict

class NumberExtractor:
    """Extract numbers, measurements, and statistics from text"""
    
    INTEGER = re.compile(r'[+-]?(?:\d{1,3}(?:,\d{3})+|\d+)(?!\.\d|\w)')
    FLOAT = re.compile(r'[+-]?(?:\d{1,3}(?:,\d{3})*\.\d+|\d+\.\d+)(?!\w)')
    PERCENTAGE = re.compile(r'(\d+(?:\.\d+)?)\s*%')
    
    MEASUREMENT = re.compile(
        r'([\d,]+(?:\.\d+)?)\s*'
        r'(km|m|cm|mm|kg|g|mg|L|mL|ml|kB|MB|GB|TB|°C|°F|K|'
        r'px|pt|em|rem|vh|vw|ms|s|min|hr|hrs|hour|hours|day|days|'
        r'Mbps|Gbps|MHz|GHz|rpm|kWh|W|kW|MW)',
        re.IGNORECASE
    )
    
    RANGE = re.compile(r'(\d+(?:\.\d+)?)\s*(?:to|-|–)\s*(\d+(?:\.\d+)?)')
    
    @classmethod
    def extract_measurements(cls, text: str) -> List[Dict]:
        results = []
        for m in cls.MEASUREMENT.finditer(text):
            value_str = m.group(1).replace(',', '')
            results.append({
                'raw': m.group(0),
                'value': float(value_str),
                'unit': m.group(2),
                'position': m.start(),
            })
        return results
    
    @classmethod
    def extract_stats(cls, text: str) -> Dict:
        numbers = []
        for m in re.finditer(r'-?(?:\d{1,3}(?:,\d{3})+|\d+)(?:\.\d+)?', text):
            try:
                val = float(m.group(0).replace(',', ''))
                numbers.append(val)
            except ValueError:
                pass
        
        if not numbers:
            return {}
        
        return {
            'count': len(numbers),
            'min': min(numbers),
            'max': max(numbers),
            'sum': sum(numbers),
            'mean': sum(numbers) / len(numbers),
        }
    
    @classmethod
    def normalize_number(cls, text: str) -> float:
        THAI_TO_ARABIC = str.maketrans('๐๑๒๓๔๕๖๗๘๙', '0123456789')
        text = text.translate(THAI_TO_ARABIC)
        text = re.sub(r'[,\s]', '', text)
        return float(text)


# ทดสอบ
texts = [
    "ความเร็ว 100 km/h อุณหภูมิ 37.5°C ขนาด 1024 MB",
    "ราคาเพิ่มขึ้น 15% จาก 45,000 เป็น 51,750 บาท",
    "latency: 42ms bandwidth: 100Mbps uptime: 99.9%",
]

print("Number Extraction:")
print("=" * 60)
for text in texts:
    print(f"\n  '{text}'")
    measurements = NumberExtractor.extract_measurements(text)
    if measurements:
        for m in measurements:
            print(f"    {m['value']} {m['unit']}")
    percents = NumberExtractor.PERCENTAGE.findall(text)
    if percents:
        print(f"    Percentages: {percents}")
```

---

## 19.5 Code Snippet Extraction

```python
import re
from typing import List, Dict

class CodeExtractor:
    """Extract code snippets from text/documentation"""
    
    MD_CODE_BLOCK = re.compile(
        r'```(?P<lang>\w+)?\n(?P<code>.*?)```',
        re.DOTALL
    )
    
    INLINE_CODE = re.compile(r'`([^`]+)`')
    
    PYTHON_DEF = re.compile(
        r'^(async\s+)?def\s+(\w+)\s*\(([^)]*)\)\s*(?:->\s*\S+)?\s*:',
        re.MULTILINE
    )
    
    PYTHON_CLASS = re.compile(
        r'^class\s+(\w+)(?:\s*\(([^)]*)\))?\s*:',
        re.MULTILINE
    )
    
    PYTHON_IMPORT = re.compile(
        r'^(?:from\s+(\S+)\s+)?import\s+(.+)',
        re.MULTILINE
    )
    
    @classmethod
    def extract_code_blocks(cls, markdown: str) -> List[Dict]:
        blocks = []
        for m in cls.MD_CODE_BLOCK.finditer(markdown):
            blocks.append({
                'language': m.group('lang') or 'text',
                'code': m.group('code').strip(),
                'lines': m.group('code').count('\n') + 1,
            })
        return blocks
    
    @classmethod
    def extract_python_structure(cls, code: str) -> Dict:
        return {
            'classes': cls.PYTHON_CLASS.findall(code),
            'functions': [(m.group(2), m.group(3)) for m in cls.PYTHON_DEF.finditer(code)],
            'imports': [(m.group(1), m.group(2)) for m in cls.PYTHON_IMPORT.finditer(code)],
        }
    
    @classmethod
    def find_todos(cls, code: str) -> List[Dict]:
        TODO = re.compile(
            r'#\s*(?P<type>TODO|FIXME|HACK|BUG|NOTE|XXX)\s*:?\s*(?P<message>[^\n]+)',
            re.IGNORECASE
        )
        return [{'type': m.group('type').upper(), 'message': m.group('message').strip(),
                 'line': code[:m.start()].count('\n') + 1}
                for m in TODO.finditer(code)]


# ทดสอบ
sample_code = '''
# TODO: Add error handling
def calculate_price(base_price, discount=0.0, tax_rate=0.07):
    # FIXME: discount validation is missing
    discounted = base_price * (1 - discount)
    return round(discounted * (1 + tax_rate), 2)

class PriceCalculator:
    def __init__(self, tax_rate=0.07):
        self.tax_rate = tax_rate
    # NOTE: simplified logic
    def calculate(self, price):
        return price * (1 + self.tax_rate)
'''

structure = CodeExtractor.extract_python_structure(sample_code)
todos = CodeExtractor.find_todos(sample_code)

print("Code Analysis:")
print("=" * 60)
print(f"\n  Classes: {structure['classes']}")
print(f"  Functions: {structure['functions']}")
print(f"\n  TODOs/FIXMEs:")
for todo in todos:
    print(f"    Line {todo['line']:3}: {todo['type']}: {todo['message']}")
```

---

## 19.6 Document Information Extraction

```python
import re
from typing import Dict, List

class DocumentExtractor:
    """Extract structured information from documents"""
    
    INVOICE_NO = re.compile(r'\b(?:invoice|inv|bill|receipt|order)\s*[#:]?\s*([A-Z0-9]{4,20})\b', re.IGNORECASE)
    AMOUNT_DUE = re.compile(r'(?:total|amount|due|pay|ยอด(?:ชำระ|รวม)?)\s*[:\s]*[฿\$€]?\s*([\d,]+(?:\.\d{1,2})?)', re.IGNORECASE)
    VAT_PATTERN = re.compile(r'(?:vat|ภาษี|tax)\s*(?:\d+%?)?\s*[:\s]*[฿\$€]?\s*([\d,]+(?:\.\d{1,2})?)', re.IGNORECASE)
    POSTAL_CODE = re.compile(r'\b(\d{5})\b')
    TRACKING = re.compile(
        r'\b(?:'
        r'[A-Z]{2}\d{9}[A-Z]{2}'
        r'|TH\d{12}'
        r'|\d{12,20}'
        r')\b'
    )
    
    @classmethod
    def extract_invoice_data(cls, text: str) -> Dict:
        result = {}
        
        inv_m = cls.INVOICE_NO.search(text)
        if inv_m:
            result['invoice_number'] = inv_m.group(1)
        
        amounts = cls.AMOUNT_DUE.findall(text)
        if amounts:
            result['amounts'] = [float(a.replace(',', '')) for a in amounts]
            result['total'] = max(result['amounts'])
        
        vat = cls.VAT_PATTERN.findall(text)
        if vat:
            result['vat'] = float(vat[0].replace(',', ''))
        
        codes = cls.POSTAL_CODE.findall(text)
        if codes:
            result['postal_codes'] = list(set(codes))
        
        tracking = cls.TRACKING.findall(text)
        if tracking:
            result['tracking'] = tracking
        
        return result


# ทดสอบ
invoice_text = """
ใบแจ้งหนี้ / INVOICE
Invoice No: INV-2026-001234
วันที่: 2 ตุลาคม 2569

บริษัท ABC จำกัด
123 ถนนสุขุมวิท แขวงคลองเตย เขตคลองเตย จังหวัดกรุงเทพมหานคร 10110

รายการ: บริการพัฒนาซอฟต์แวร์
ราคาสุทธิ: 50,000.00 บาท
VAT 7%: 3,500.00 บาท
ยอดรวมทั้งสิ้น: 53,500.00 บาท

Tracking: TH202610023456
"""

doc_data = DocumentExtractor.extract_invoice_data(invoice_text)
print("Invoice Data Extraction:")
print("=" * 60)
for key, value in doc_data.items():
    print(f"  {key}: {value}")
```

---

## 19.7 สรุป Part 19

```
Pattern สำคัญสำหรับ Data Extraction:
Currency:     [\$€£¥฿]\s*[\d,]+(?:\.\d{1,2})?
Thai ID:      \d-\d{4}-\d{5}-\d{2}-\d
Hashtag:      #(\w+)
Mention:      @(\w+)
Percentage:   (\d+(?:\.\d+)?)\s*%
Measurement:  ([\d,]+(?:\.\d+)?)\s*(km|m|kg|MB|GB|°C|ms|...)
Config KV:    (\w+)\s*[=:]\s*(.+)
Query param:  ([^&=]+)=([^&]*)
Code block:   ```(\w+)?\n(.*?)```  (DOTALL)
TODO:         #\s*(TODO|FIXME|BUG)\s*:?\s*(.+)
```

---

*[← Part 18: Log Parsing](part-18-log-parsing.md) | [→ Part 20: Search & Replace](part-20-search-replace.md)*
