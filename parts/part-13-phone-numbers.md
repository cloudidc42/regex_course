# Part 13: Phone Numbers — การจับคู่เบอร์โทรศัพท์

> **ระดับ:** พื้นฐาน-กลาง | **เวลาเรียน:** ~75 นาที | **ข้อกำหนด:** Part 01-10

---

## 13.1 ความซับซ้อนของเบอร์โทรศัพท์

เบอร์โทรศัพท์มีรูปแบบที่หลากหลายมากทั่วโลก:

```
เบอร์โทรไทย (ตัวอย่าง):
- 081-234-5678          มือถือ ais/dtac/true
- 02-123-4567           กทม.
- 053-123-456           ต่างจังหวัด
- 0812345678            ไม่มีเครื่องหมาย
- +66 81 234 5678       international format
- (02) 123-4567         ใส่ ()

เบอร์โทรสากล:
- +1 (555) 123-4567     US
- +44 20 7946 0958      UK
- +81 3-1234-5678       Japan
- +49 30 12345678       Germany
- +86 138-1234-5678     China
```

---

## 13.2 เบอร์โทรไทย (Thai Phone Numbers)

```python
import re

# ============================================
# Pattern สำหรับเบอร์โทรไทย
# ============================================

class ThaiPhoneParser:
    
    # มือถือ: 08x, 09x, 06x
    MOBILE_PATTERN = re.compile(
        r'(?:0[689]\d)'         # prefix: 06, 08, 09
        r'[-\s.]?'              # separator optional
        r'(\d{3})'              # 3 digits
        r'[-\s.]?'              # separator optional
        r'(\d{4})'              # 4 digits
    )
    
    # โทรบ้าน/สำนักงาน กทม.: 02-xxxx-xxxx
    BANGKOK_PATTERN = re.compile(
        r'0[-\s.]?2'            # 02
        r'[-\s.]?'
        r'(\d{3,4})'            # 3-4 digits
        r'[-\s.]?'
        r'(\d{4})'              # 4 digits
    )
    
    # ต่างจังหวัด: 0xx-xxx-xxxx
    PROVINCE_PATTERN = re.compile(
        r'0[3-57-9]\d'          # 03x-07x, 09x
        r'[-\s.]?'
        r'(\d{3})'
        r'[-\s.]?'
        r'(\d{3,4})'
    )
    
    # International format (Thailand)
    INTL_PATTERN = re.compile(
        r'\+66'
        r'[-\s.]?'
        r'(?:0?)'               # optional 0
        r'([689]\d)'            # prefix without 0
        r'[-\s.]?'
        r'(\d{3})'
        r'[-\s.]?'
        r'(\d{4})'
    )
    
    # รวมทั้งหมด
    ALL_THAI_PATTERN = re.compile(
        r'(?:'
        r'\+66[-\s.]?(?:0)?[689]\d[-\s.]?\d{3}[-\s.]?\d{4}'  # +66
        r'|0[689]\d[-\s.]?\d{3}[-\s.]?\d{4}'                  # mobile
        r'|0[-\s.]?2[-\s.]?\d{3,4}[-\s.]?\d{4}'              # Bangkok
        r'|0[3-57-9]\d[-\s.]?\d{3}[-\s.]?\d{3,4}'           # province
        r')'
    )
    
    @classmethod
    def extract(cls, text: str) -> list:
        """ดึงเบอร์โทรทั้งหมดจาก text"""
        matches = []
        for m in cls.ALL_THAI_PATTERN.finditer(text):
            phone = m.group()
            normalized = cls.normalize(phone)
            matches.append({
                'original': phone,
                'normalized': normalized,
                'position': m.span(),
            })
        return matches
    
    @classmethod
    def normalize(cls, phone: str) -> str:
        """Normalize เบอร์โทรไทยให้เป็นรูปแบบมาตรฐาน"""
        # เอาแค่ตัวเลข
        digits = re.sub(r'[^\d+]', '', phone)
        
        # แปลง +66 เป็น 0
        if digits.startswith('+66'):
            digits = '0' + digits[3:]
        
        # ลบ leading 0 ถ้ามีมากเกิน
        digits = re.sub(r'^00+', '0', digits)
        
        # Format
        if len(digits) == 10:
            if digits.startswith('02'):
                return f"{digits[:2]}-{digits[2:6]}-{digits[6:]}"
            return f"{digits[:3]}-{digits[3:6]}-{digits[6:]}"
        elif len(digits) == 9:  # บางจังหวัด
            return f"{digits[:3]}-{digits[3:6]}-{digits[6:]}"
        
        return digits
    
    @classmethod
    def validate(cls, phone: str) -> dict:
        """ตรวจสอบความถูกต้องของเบอร์โทร"""
        digits = re.sub(r'[^\d+]', '', phone)
        
        # แปลง international
        if digits.startswith('+66'):
            digits = '0' + digits[3:]
        
        result = {
            'phone': phone,
            'digits': digits,
            'valid': False,
            'type': None,
            'normalized': None,
        }
        
        # มือถือ: 10 หลัก เริ่มด้วย 06, 08, 09
        if re.match(r'^0[689]\d{8}$', digits):
            result['valid'] = True
            prefix = digits[:3]
            if prefix.startswith('06'):
                result['type'] = 'mobile_true_dtac'
            elif prefix.startswith('08'):
                result['type'] = 'mobile'
            elif prefix.startswith('09'):
                result['type'] = 'mobile'
            result['normalized'] = f"{digits[:3]}-{digits[3:6]}-{digits[6:]}"
        
        # กทม.: 9 หลัก เริ่มด้วย 02
        elif re.match(r'^02\d{7}$', digits):
            result['valid'] = True
            result['type'] = 'landline_bangkok'
            result['normalized'] = f"{digits[:2]}-{digits[2:6]}-{digits[6:]}"
        
        # ต่างจังหวัด: 9-10 หลัก
        elif re.match(r'^0[3-9]\d{7,8}$', digits):
            result['valid'] = True
            result['type'] = 'landline_province'
            if len(digits) == 9:
                result['normalized'] = f"{digits[:3]}-{digits[3:6]}-{digits[6:]}"
            else:
                result['normalized'] = f"{digits[:3]}-{digits[3:6]}-{digits[6:]}"
        
        return result


# ============================================
# ทดสอบ
# ============================================

test_phones = [
    "081-234-5678",
    "0812345678",
    "+66-81-234-5678",
    "+66 812 345 678",
    "02-123-4567",
    "(02) 123 4567",
    "053-123-456",
    "044-123-4567",
    "02 345 6789",
    "invalid-phone",
    "12345",
]

print("Thai Phone Validation:")
print("=" * 60)
for phone in test_phones:
    result = ThaiPhoneParser.validate(phone)
    icon = '✅' if result['valid'] else '❌'
    print(f"{icon} {phone:25} → {result.get('type', 'N/A'):25} {result.get('normalized', '')}")

# Extract from text
text = """
ติดต่อฝ่ายขาย: 081-234-5678
ฝ่ายบริการลูกค้า: 02-123-4567 หรือ 1234 (สายด่วน)
สำนักงานใหญ่: +66 2 123 4567
สาขาเชียงใหม่: 053-123-456
"""

print("\nExtract from text:")
found = ThaiPhoneParser.extract(text)
for p in found:
    print(f"  Found: {p['original']:25} → {p['normalized']}")
```

---

## 13.3 International Phone Numbers

```python
import re
from dataclasses import dataclass
from typing import Optional

@dataclass
class PhoneInfo:
    original: str
    country_code: Optional[str]
    country_name: Optional[str]
    number: str
    normalized: str
    is_valid: bool

# Country code mapping (subset)
COUNTRY_CODES = {
    '1': 'US/Canada',
    '7': 'Russia/Kazakhstan',
    '20': 'Egypt',
    '27': 'South Africa',
    '30': 'Greece',
    '31': 'Netherlands',
    '32': 'Belgium',
    '33': 'France',
    '34': 'Spain',
    '36': 'Hungary',
    '39': 'Italy',
    '40': 'Romania',
    '41': 'Switzerland',
    '43': 'Austria',
    '44': 'United Kingdom',
    '45': 'Denmark',
    '46': 'Sweden',
    '47': 'Norway',
    '48': 'Poland',
    '49': 'Germany',
    '52': 'Mexico',
    '54': 'Argentina',
    '55': 'Brazil',
    '56': 'Chile',
    '57': 'Colombia',
    '58': 'Venezuela',
    '60': 'Malaysia',
    '61': 'Australia',
    '62': 'Indonesia',
    '63': 'Philippines',
    '64': 'New Zealand',
    '65': 'Singapore',
    '66': 'Thailand',
    '81': 'Japan',
    '82': 'South Korea',
    '84': 'Vietnam',
    '86': 'China',
    '90': 'Turkey',
    '91': 'India',
    '92': 'Pakistan',
    '94': 'Sri Lanka',
    '95': 'Myanmar',
    '98': 'Iran',
}

class InternationalPhoneParser:
    
    # E.164 format: +[country code][subscriber number]
    E164_PATTERN = re.compile(
        r'^\+(?P<cc>\d{1,3})(?P<number>\d{7,12})$'
    )
    
    # General international (with separators)
    INTL_PATTERN = re.compile(
        r'^\+(?P<cc>\d{1,3})'
        r'[\s.\-]?'
        r'(?:\((?P<area>\d+)\)[\s.\-]?)?'
        r'(?P<number>[\d\s.\-]{7,15})$'
    )
    
    # Extract any phone-like pattern from text
    PHONE_EXTRACT = re.compile(
        r'(?:'
        r'\+\d{1,3}[\s.\-]?\(?\d{1,4}\)?[\s.\-]?\d{3,4}[\s.\-]?\d{4}'  # International
        r'|\(?\d{3}\)?[\s.\-]?\d{3}[\s.\-]?\d{4}'                         # US/CA
        r'|\d{3,4}[\s.\-]\d{3,4}[\s.\-]\d{3,4}'                          # Generic
        r')'
    )
    
    @classmethod
    def parse(cls, phone: str) -> PhoneInfo:
        """Parse international phone number"""
        original = phone
        # Clean
        cleaned = re.sub(r'[\s.\-\(\)]', '', phone)
        
        # Try E.164
        m = cls.E164_PATTERN.match(cleaned)
        if m:
            cc = m.group('cc')
            number = m.group('number')
            country = COUNTRY_CODES.get(cc, 
                       COUNTRY_CODES.get(cc[:2], 
                       COUNTRY_CODES.get(cc[:1], 'Unknown')))
            
            return PhoneInfo(
                original=original,
                country_code=cc,
                country_name=country,
                number=number,
                normalized=f"+{cc}{number}",
                is_valid=True,
            )
        
        # Try general international
        m = cls.INTL_PATTERN.match(phone.strip())
        if m:
            cc = m.group('cc')
            number = re.sub(r'[\s.\-]', '', m.group('number'))
            country = COUNTRY_CODES.get(cc, 'Unknown')
            
            return PhoneInfo(
                original=original,
                country_code=cc,
                country_name=country,
                number=number,
                normalized=f"+{cc}{number}",
                is_valid=len(number) >= 7,
            )
        
        return PhoneInfo(
            original=original,
            country_code=None,
            country_name=None,
            number=cleaned,
            normalized=phone,
            is_valid=False,
        )
    
    @classmethod
    def to_e164(cls, phone: str, default_country_code: str = '66') -> Optional[str]:
        """แปลงเบอร์โทรให้เป็น E.164 format"""
        
        # ถ้ามี + อยู่แล้ว
        if phone.startswith('+'):
            result = cls.parse(phone)
            if result.is_valid:
                return result.normalized
            return None
        
        # ลบ separator
        digits = re.sub(r'[^\d]', '', phone)
        
        # เอา leading 0 ออก
        if digits.startswith('0'):
            digits = digits[1:]
        
        return f"+{default_country_code}{digits}"
    
    @classmethod
    def extract(cls, text: str) -> list:
        """Extract phone numbers จาก text"""
        results = []
        for m in cls.PHONE_EXTRACT.finditer(text):
            phone = m.group()
            info = cls.parse(phone)
            results.append(info)
        return results


# ทดสอบ
intl_phones = [
    "+66 81 234 5678",      # Thailand mobile
    "+1 (555) 123-4567",    # US
    "+44 20 7946 0958",     # UK
    "+81 3-1234-5678",      # Japan
    "+86 138-1234-5678",    # China  
    "+49 30 12345678",      # Germany
    "+65 6234 5678",        # Singapore
    "+60 3-1234-5678",      # Malaysia
    "+84 24 3456 7890",     # Vietnam
    "+62 21 1234 5678",     # Indonesia
]

print("\nInternational Phone Numbers:")
print("=" * 70)
for phone in intl_phones:
    info = InternationalPhoneParser.parse(phone)
    icon = '✅' if info.is_valid else '❌'
    country = info.country_name or 'Unknown'
    print(f"{icon} {phone:25} | {country:20} | E.164: {info.normalized}")

# Extract from mixed text
mixed_text = """
Call our offices:
- Bangkok: +66 2 123 4567
- London: +44 20 7946 0958  
- New York: +1 (212) 555-1234
- Tokyo: +81 3-1234-5678
Local: 081-234-5678
"""

print("\nExtracted phones:")
for info in InternationalPhoneParser.extract(mixed_text):
    print(f"  {info.original:25} → {info.normalized}")
```

---

## 13.4 Phone Formatter และ Masker

```python
import re

class PhoneFormatter:
    """Format and mask phone numbers"""
    
    @staticmethod
    def format_thai_mobile(digits: str) -> str:
        """Format Thai mobile: 0xx-xxx-xxxx"""
        clean = re.sub(r'[^\d]', '', digits)
        if len(clean) == 10 and clean.startswith('0'):
            return f"{clean[:3]}-{clean[3:6]}-{clean[6:]}"
        return digits
    
    @staticmethod
    def format_us(digits: str) -> str:
        """Format US: (xxx) xxx-xxxx"""
        clean = re.sub(r'[^\d]', '', digits)
        if len(clean) == 10:
            return f"({clean[:3]}) {clean[3:6]}-{clean[6:]}"
        if len(clean) == 11 and clean.startswith('1'):
            return f"+1 ({clean[1:4]}) {clean[4:7]}-{clean[7:]}"
        return digits
    
    @staticmethod
    def mask(phone: str, show_last: int = 4) -> str:
        """ปิดบังเบอร์โทร ยกเว้น N หลักท้าย"""
        digits = re.sub(r'\d', 'X', phone)
        
        # แทน X ท้ายด้วยตัวเลขจริง
        original_digits = re.findall(r'\d', phone)
        masked_digits = ['X'] * (len(original_digits) - show_last) + original_digits[-show_last:]
        
        # ใส่กลับ
        digit_idx = 0
        result = []
        for char in phone:
            if char.isdigit():
                result.append(masked_digits[digit_idx])
                digit_idx += 1
            else:
                result.append(char)
        
        return ''.join(result)
    
    @staticmethod
    def normalize_separators(phone: str, separator: str = '-') -> str:
        """เปลี่ยน separator ให้เหมือนกัน"""
        digits_only = re.sub(r'[^\d+]', '', phone)
        return re.sub(r'(\d+)', r'\1', phone).replace(' ', separator).replace('.', separator).replace('-', separator)


# ทดสอบ
formatter = PhoneFormatter()

phones_to_format = [
    ("0812345678", "thai"),
    ("5551234567", "us"),
    ("12125551234", "us"),
]

print("Phone Formatting:")
for digits, fmt in phones_to_format:
    if fmt == "thai":
        formatted = formatter.format_thai_mobile(digits)
    else:
        formatted = formatter.format_us(digits)
    print(f"  {digits:15} → {formatted}")

# Masking
mask_tests = [
    "081-234-5678",
    "+66 81 234 5678",
    "02-123-4567",
]

print("\nPhone Masking (show last 4):")
for phone in mask_tests:
    masked = formatter.mask(phone, show_last=4)
    print(f"  {phone:20} → {masked}")
```

---

## 13.5 Phone Validation ใน JavaScript

```javascript
// ============================================
// JavaScript Phone Validation
// ============================================

class ThaiPhoneValidator {
    static PATTERNS = {
        mobile: /^0[689]\d{8}$/,
        bangkok: /^02\d{7}$/,
        province: /^0[3-57-9]\d{7,8}$/,
        intl: /^\+66[689]\d{7}$/,
    };
    
    static clean(phone) {
        return phone.replace(/[\s.\-()]/g, '').replace(/^\+66/, '0');
    }
    
    static validate(phone) {
        const cleaned = this.clean(phone);
        
        for (const [type, pattern] of Object.entries(this.PATTERNS)) {
            if (pattern.test(cleaned)) {
                return { valid: true, type, normalized: this.format(cleaned) };
            }
        }
        
        return { valid: false, type: null, normalized: null };
    }
    
    static format(digits) {
        if (digits.startsWith('02') && digits.length === 9) {
            return `${digits.slice(0,2)}-${digits.slice(2,6)}-${digits.slice(6)}`;
        }
        if (digits.length === 10) {
            return `${digits.slice(0,3)}-${digits.slice(3,6)}-${digits.slice(6)}`;
        }
        return digits;
    }
    
    static mask(phone, showLast = 4) {
        return phone.replace(/\d/g, (d, i, arr) => {
            const digitCount = arr.filter(c => /\d/.test(c)).length;
            const digitIdx = arr.slice(0, i).filter(c => /\d/.test(c)).length;
            return digitIdx < digitCount - showLast ? 'X' : d;
        });
    }
}

// ทดสอบ
const phones = [
    '081-234-5678',
    '0812345678',
    '+66 812345678',
    '02-123-4567',
    '053-123-456',
    'not-a-phone',
];

phones.forEach(phone => {
    const result = ThaiPhoneValidator.validate(phone);
    const icon = result.valid ? '✅' : '❌';
    console.log(`${icon} ${phone.padEnd(20)} → ${result.type || 'invalid'} ${result.normalized || ''}`);
});

// Masking example
console.log('\nMasked:', ThaiPhoneValidator.mask('081-234-5678'));
// Output: XXX-XXX-5678
```

---

## 13.6 สรุป Part 13

✅ **Thai mobile**: `0[689]\d{8}` — 10 หลัก  
✅ **Thai Bangkok**: `02\d{7}` — 9 หลัก  
✅ **Thai province**: `0[3-9]\d{7,8}` — 9-10 หลัก  
✅ **International E.164**: `\+\d{1,3}\d{7,12}`  
✅ **Flexible separators**: `[-\s.]?` ระหว่างกลุ่มตัวเลข  
✅ **Normalization**: เปลี่ยนทุก format เป็น standard  
✅ **Masking**: ปิดบังหลักแรก ๆ  

```python
import re

# Quick reference patterns สำหรับ Thai phones
THAI_MOBILE = re.compile(r'^(?:\+66|0)[689]\d{8}$')
THAI_PHONE_ANY = re.compile(
    r'(?:\+66|0)[689]\d[-.\s]?\d{3}[-.\s]?\d{4}'  # mobile
    r'|(?:\+66|0)2\d[-.\s]?\d{4}[-.\s]?\d{4}'     # Bangkok
    r'|(?:\+66|0)[3-9]\d[-.\s]?\d{3}[-.\s]?\d{4}' # province
)
```

---

*[⬅ Part 12: URL Matching](part-12-url-matching.md) | [➡ Part 14: Dates & Times](part-14-dates-times.md)*
