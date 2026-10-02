# Part 26: Web Scraping — การดึงข้อมูลจากเว็บ

> **ระดับ:** กลาง | **เวลาเรียน:** ~70 นาที | **ข้อกำหนด:** Part 01-25

---

## 26.1 Web Scraping กับ Regex

```
คำเตือน: ใช้ BeautifulSoup หรือ lxml สำหรับ HTML ที่ซับซ้อน
Regex เหมาะสำหรับ:
- ข้อมูลที่มี pattern ชัดเจน
- API responses (plain text, non-nested)
- Simple data extraction
- Pre-processing ก่อนส่งให้ HTML parser
```

---

## 26.2 Extract Data จาก HTML

```python
import re
from typing import List, Dict, Optional

class WebDataExtractor:
    """ดึงข้อมูลจาก HTML/text ด้วย regex"""
    
    # Product patterns
    PRICE     = re.compile(r'(?:฿|THB|\$|USD|€)\s*([\d,]+(?:\.\d{2})?)', re.IGNORECASE)
    RATING    = re.compile(r'(\d+(?:\.\d+)?)\s*(?:/5|out of 5|stars?)', re.IGNORECASE)
    REVIEW_COUNT = re.compile(r'([\d,]+)\s*(?:reviews?|ratings?)', re.IGNORECASE)
    
    # Contact info
    PHONE_TH  = re.compile(r'(?:\+?66|0)[6-9]\d[\s\-.]?\d{3}[\s\-.]?\d{4}')
    EMAIL     = re.compile(r'\b[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}\b')
    LINE_ID   = re.compile(r'(?:Line|LINE)\s*(?:ID\s*:?\s*)?[@]?([\w.]+)', re.IGNORECASE)
    
    # Social media
    TWITTER   = re.compile(r'(?:twitter\.com/|@)(\w{1,15})', re.IGNORECASE)
    INSTAGRAM = re.compile(r'(?:instagram\.com/|@)([a-z0-9_.]+)', re.IGNORECASE)
    FACEBOOK  = re.compile(r'facebook\.com/(?:pages/)?([^/?&\s"]+)', re.IGNORECASE)
    
    @classmethod
    def extract_prices(cls, html: str) -> List[Dict]:
        prices = []
        for m in cls.PRICE.finditer(html):
            try:
                amount = float(m.group(1).replace(',', ''))
                currency = m.group(0)[0] if m.group(0)[0] in '฿$€' else m.group(0)[:3]
                prices.append({'raw': m.group(0), 'amount': amount, 'currency': currency})
            except ValueError:
                pass
        return prices
    
    @classmethod
    def extract_contacts(cls, text: str) -> Dict:
        return {
            'phones':  cls.PHONE_TH.findall(text),
            'emails':  cls.EMAIL.findall(text),
            'line_ids': cls.LINE_ID.findall(text),
        }
    
    @classmethod
    def extract_social(cls, text: str) -> Dict:
        return {
            'twitter':   cls.TWITTER.findall(text),
            'instagram': cls.INSTAGRAM.findall(text),
            'facebook':  cls.FACEBOOK.findall(text),
        }
    
    @classmethod
    def extract_table_data(cls, html: str) -> List[List[str]]:
        """ดึงข้อมูลจาก HTML table"""
        TAG  = re.compile(r'<[^>]+>')
        ROW  = re.compile(r'<tr[^>]*>(.*?)</tr>', re.IGNORECASE | re.DOTALL)
        CELL = re.compile(r'<t[dh][^>]*>(.*?)</t[dh]>', re.IGNORECASE | re.DOTALL)
        
        rows = []
        for row_m in ROW.finditer(html):
            cells = []
            for cell_m in CELL.finditer(row_m.group(1)):
                text = TAG.sub('', cell_m.group(1))
                text = re.sub(r'\s+', ' ', text).strip()
                cells.append(text)
            if cells:
                rows.append(cells)
        return rows


# ทดสอบ
html_page = """
<html>
<body>
  <h1>สินค้าราคาดี</h1>
  
  <div class="product">
    <h2>iPhone 15 Pro</h2>
    <p class="price">ราคา: ฿45,990</p>
    <p class="rating">4.8 out of 5 (1,234 reviews)</p>
    <p>ติดต่อ: 089-123-4567 หรือ admin@iphone-store.com</p>
    <p>LINE ID: @iphonestore</p>
    <p>Follow us: twitter.com/iphonestore</p>
  </div>
  
  <table>
    <tr><th>รุ่น</th><th>ราคา</th><th>สต็อก</th></tr>
    <tr><td>128GB</td><td>฿45,990</td><td>มีสินค้า</td></tr>
    <tr><td>256GB</td><td>฿51,990</td><td>มีสินค้า</td></tr>
    <tr><td>512GB</td><td>฿63,990</td><td>สินค้าหมด</td></tr>
  </table>
</body>
</html>
"""

extractor = WebDataExtractor

print("Web Data Extraction:")
print("=" * 60)

prices = extractor.extract_prices(html_page)
print(f"\nPrices:")
for p in prices:
    print(f"  {p['currency']}{p['amount']:,.0f}")

contacts = extractor.extract_contacts(html_page)
print(f"\nContacts:")
for key, values in contacts.items():
    if values:
        print(f"  {key}: {values}")

social = extractor.extract_social(html_page)
print(f"\nSocial:")
for key, values in social.items():
    if values:
        print(f"  {key}: {values}")

table = extractor.extract_table_data(html_page)
print(f"\nTable:")
for row in table:
    print(f"  {' | '.join(row)}")
```

---

## 26.3 Parse API Responses

```python
import re
import json
from typing import List, Dict, Optional

class APIResponseParser:
    """Parse และ extract ข้อมูลจาก API responses"""
    
    # JSON-like patterns (ไม่ใช่ full JSON parser)
    JSON_STRING = re.compile(r'"([^"\\]|\\.)*"')
    JSON_NUMBER = re.compile(r'-?(?:0|[1-9]\d*)(?:\.\d+)?(?:[eE][+-]?\d+)?')
    
    # Common API patterns
    PAGINATION  = re.compile(r'"(?:page|current_page)"\s*:\s*(\d+)')
    TOTAL       = re.compile(r'"(?:total|total_count|count)"\s*:\s*(\d+)')
    NEXT_URL    = re.compile(r'"(?:next|next_url|next_page)"\s*:\s*"([^"]+)"')
    
    # Error patterns
    ERROR_CODE  = re.compile(r'"(?:error_code|code)"\s*:\s*"?(\w+)"?')
    ERROR_MSG   = re.compile(r'"(?:error|message|error_message)"\s*:\s*"([^"]+)"')
    
    @classmethod
    def extract_pagination(cls, response_text: str) -> Dict:
        result = {}
        
        m = cls.PAGINATION.search(response_text)
        if m: result['page'] = int(m.group(1))
        
        m = cls.TOTAL.search(response_text)
        if m: result['total'] = int(m.group(1))
        
        m = cls.NEXT_URL.search(response_text)
        if m: result['next'] = m.group(1)
        
        return result
    
    @classmethod
    def extract_error(cls, response_text: str) -> Optional[Dict]:
        code_m = cls.ERROR_CODE.search(response_text)
        msg_m  = cls.ERROR_MSG.search(response_text)
        
        if code_m or msg_m:
            return {
                'code': code_m.group(1) if code_m else None,
                'message': msg_m.group(1) if msg_m else None,
            }
        return None
    
    @classmethod
    def extract_field_values(cls, response_text: str, field_name: str) -> List:
        """ดึง values ของ field ที่กำหนด"""
        pattern = re.compile(
            rf'"{ re.escape(field_name)}"\s*:\s*'
            r'("(?:[^"\\]|\\.)*"|\d+(?:\.\d+)?|true|false|null)',
        )
        results = []
        for m in pattern.finditer(response_text):
            raw = m.group(1)
            if raw.startswith('"'):
                results.append(raw[1:-1])
            elif raw in ('true', 'false'):
                results.append(raw == 'true')
            elif raw == 'null':
                results.append(None)
            else:
                results.append(float(raw) if '.' in raw else int(raw))
        return results


# ทดสอบ
api_response = '''{
  "status": "success",
  "page": 2,
  "total": 150,
  "per_page": 20,
  "next": "/api/products?page=3",
  "data": [
    {"id": 1, "name": "iPhone 15", "price": 45990, "stock": true},
    {"id": 2, "name": "Samsung S24", "price": 38990, "stock": true},
    {"id": 3, "name": "Pixel 8", "price": 35990, "stock": false}
  ]
}'''

parser = APIResponseParser

print("API Response Parsing:")
print("=" * 60)

pagination = parser.extract_pagination(api_response)
print(f"\nPagination: {pagination}")

names  = parser.extract_field_values(api_response, 'name')
prices = parser.extract_field_values(api_response, 'price')
stocks = parser.extract_field_values(api_response, 'stock')

print(f"\nProducts:")
for name, price, stock in zip(names, prices, stocks):
    status = 'in stock' if stock else 'out of stock'
    print(f"  {name:20} ฿{price:>8,.0f}  {status}")

# Error response
error_response = '{"error_code": "NOT_FOUND", "message": "Resource not found", "status": 404}'
error = parser.extract_error(error_response)
print(f"\nError: {error}")
```

---

## 26.4 URL Analysis

```python
import re
from typing import Dict, List
from urllib.parse import unquote

class URLAnalyzer:
    """วิเคราะห์และจัดการ URLs"""
    
    URL_FULL = re.compile(
        r'(?P<scheme>https?|ftp)://'
        r'(?:(?P<user>[^:@]+)(?::(?P<password>[^@]+))?@)?'
        r'(?P<host>(?:[a-zA-Z0-9-]+\.)+[a-zA-Z]{2,}|localhost|\d{1,3}(?:\.\d{1,3}){3})'
        r'(?::(?P<port>\d+))?'
        r'(?P<path>/[^?#]*)?'
        r'(?:\?(?P<query>[^#]*))?'
        r'(?:#(?P<fragment>.*))?'
    )
    
    QUERY_PARAM = re.compile(r'(?:^|&)([^=&]+)(?:=([^&]*))?')
    
    @classmethod
    def parse(cls, url: str) -> Dict:
        m = cls.URL_FULL.match(url)
        if not m:
            return {'valid': False, 'raw': url}
        
        d = m.groupdict()
        d['valid'] = True
        
        # Parse query string
        if d['query']:
            params = {}
            for pm in cls.QUERY_PARAM.finditer(d['query']):
                key = unquote(pm.group(1))
                val = unquote(pm.group(2) or '')
                if key in params:
                    if not isinstance(params[key], list):
                        params[key] = [params[key]]
                    params[key].append(val)
                else:
                    params[key] = val
            d['params'] = params
        
        return d
    
    @classmethod
    def extract_all_urls(cls, text: str) -> List[str]:
        return [m.group(0) for m in cls.URL_FULL.finditer(text)]
    
    @classmethod
    def is_same_domain(cls, url1: str, url2: str) -> bool:
        m1 = cls.URL_FULL.match(url1)
        m2 = cls.URL_FULL.match(url2)
        if not m1 or not m2:
            return False
        return m1.group('host') == m2.group('host')
    
    @classmethod
    def normalize(cls, url: str) -> str:
        """Normalize URL"""
        url = re.sub(r'/+$', '', url)
        url = re.sub(r':80(?=/|$)', '', url)
        url = re.sub(r':443(?=/|$)', '', url)
        def lower_host(m):
            return m.group(0).lower()
        url = re.sub(r'^https?://[^/]+', lower_host, url)
        return url


# ทดสอบ
urls = [
    "https://api.example.com/v1/users?page=1&limit=20&sort=name",
    "http://user:pass@db.local:5432/mydb",
    "https://shop.example.co.th/products/123?ref=homepage#reviews",
    "ftp://files.company.com/uploads/data.csv",
]

print("URL Analysis:")
print("=" * 60)
for url in urls:
    parsed = URLAnalyzer.parse(url)
    print(f"\n  URL: {url[:60]}")
    print(f"    scheme: {parsed.get('scheme')}")
    print(f"    host:   {parsed.get('host')}")
    print(f"    path:   {parsed.get('path')}")
    if parsed.get('params'):
        print(f"    params: {parsed.get('params')}")
```

---

## 26.5 สรุป Part 26

```
Web Scraping Patterns:
Price:    (?:฿|\$|€)\s*([\d,]+(?:\.\d{2})?)
Phone TH: (?:\+?66|0)[6-9]\d[\s\-.]?\d{3}[\s\-.]?\d{4}
Email:    \b[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}\b
URL full: (?P<scheme>https?)://(?P<host>[...])(?P<path>\/[^?#]*)...
HTML tag: <(\w+)[^>]*>(.*?)<\/\1>  (DOTALL)
JSON field: "field"\s*:\s*(".*?"|\d+|true|false|null)

Best practices:
- ใช้ BeautifulSoup/lxml สำหรับ complex HTML
- Regex เหมาะกับ well-structured text
- Always validate URLs before following
- Respect robots.txt
```

---

*[← Part 25: File Processing](part-25-file-processing.md) | [→ Part 27: Email Validation](part-27-email-validation.md)*
