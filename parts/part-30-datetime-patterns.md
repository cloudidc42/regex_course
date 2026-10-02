# Part 30: Date & Time Patterns — รูปแบบวันที่และเวลา

> **ระดับ:** กลาง | **เวลาเรียน:** ~60 นาที | **ข้อกำหนด:** Part 01-29

---

## 30.1 Date Format Patterns

```python
import re
from typing import Optional, Dict

class DatePatterns:
    """Patterns สำหรับรูปแบบวันที่ต่างๆ"""
    
    ISO_DATE      = re.compile(r'\b(\d{4})-(0[1-9]|1[0-2])-(0[1-9]|[12]\d|3[01])\b')
    ISO_DATETIME  = re.compile(
        r'\b(\d{4})-(0[1-9]|1[0-2])-(0[1-9]|[12]\d|3[01])'
        r'[T ]([01]\d|2[0-3]):([0-5]\d)(?::([0-5]\d)(?:\.(\d+))?)?'
        r'(Z|[+-][01]\d:[0-5]\d)?\b'
    )
    
    DATE_TH_BE    = re.compile(r'\b(0?[1-9]|[12]\d|3[01])[/.-](0?[1-9]|1[0-2])[/.-](\d{4})\b')
    DATE_US       = re.compile(r'\b(0?[1-9]|1[0-2])/(0?[1-9]|[12]\d|3[01])/(\d{4})\b')
    
    DATE_NATURAL  = re.compile(
        r'(?:(?:Mon|Tue|Wed|Thu|Fri|Sat|Sun)(?:day)?,?\s+)?'
        r'(?P<day>\d{1,2})(?:st|nd|rd|th)?\s+'
        r'(?P<month>Jan(?:uary)?|Feb(?:ruary)?|Mar(?:ch)?|Apr(?:il)?|May|Jun(?:e)?|'
        r'Jul(?:y)?|Aug(?:ust)?|Sep(?:tember)?|Oct(?:ober)?|Nov(?:ember)?|Dec(?:ember)?)'
        r',?\s+(?P<year>\d{4})',
        re.IGNORECASE
    )
    
    THAI_DATE = re.compile(
        r'(?P<day>\d{1,2})\s+'
        r'(?P<month>มกราคม|กุมภาพันธ์|มีนาคม|เมษายน|พฤษภาคม|มิถุนายน|'
        r'กรกฎาคม|สิงหาคม|กันยายน|ตุลาคม|พฤศจิกายน|ธันวาคม)'
        r'\s+(?:พ\.?ศ\.?\s*)?(?P<year>\d{4})'
    )
    
    THAI_MONTHS = {
        'มกราคม': 1, 'กุมภาพันธ์': 2, 'มีนาคม': 3, 'เมษายน': 4,
        'พฤษภาคม': 5, 'มิถุนายน': 6, 'กรกฎาคม': 7, 'สิงหาคม': 8,
        'กันยายน': 9, 'ตุลาคม': 10, 'พฤศจิกายน': 11, 'ธันวาคม': 12,
    }
    
    @classmethod
    def parse_thai_date(cls, text: str) -> Optional[Dict]:
        m = cls.THAI_DATE.search(text)
        if not m:
            return None
        
        year_be = int(m.group('year'))
        year_ce = year_be - 543 if year_be > 2400 else year_be
        
        return {
            'day': int(m.group('day')),
            'month': cls.THAI_MONTHS.get(m.group('month')),
            'month_name': m.group('month'),
            'year_be': year_be if year_be > 2400 else year_be + 543,
            'year_ce': year_ce,
        }
    
    @classmethod
    def extract_all_dates(cls, text: str) -> list:
        dates = []
        for m in cls.ISO_DATE.finditer(text):
            dates.append({'format': 'ISO', 'raw': m.group(0), 
                         'year': m.group(1), 'month': m.group(2), 'day': m.group(3)})
        for m in cls.DATE_TH_BE.finditer(text):
            dates.append({'format': 'DD/MM/YYYY', 'raw': m.group(0),
                         'day': m.group(1), 'month': m.group(2), 'year': m.group(3)})
        for m in cls.DATE_NATURAL.finditer(text):
            dates.append({'format': 'Natural', 'raw': m.group(0),
                         'day': m.group('day'), 'month': m.group('month'), 'year': m.group('year')})
        return dates


# ทดสอบ
test_texts = [
    "Meeting on 2026-10-02 at 14:30:00Z",
    "Invoice dated 15/09/2026 due 30/09/2026",
    "Conference on October 15th, 2026",
    "วันที่ 2 ตุลาคม พ.ศ. 2569 เวลา 14:30",
    "Born: 01/15/1990 (MM/DD/YYYY format)",
]

patterns = DatePatterns

print("Date Extraction:")
print("=" * 60)

for text in test_texts:
    dates = patterns.extract_all_dates(text)
    print(f"\n  Text: {text}")
    for d in dates:
        print(f"    [{d['format']}] {d['raw']}")
    thai = patterns.parse_thai_date(text)
    if thai:
        print(f"    [THAI] {thai['day']}/{thai['month']}/{thai['year_ce']} CE ({thai['year_be']} BE)")
```

---

## 30.2 Time Patterns

```python
import re
from typing import Optional, Dict

class TimePatterns:
    """Patterns สำหรับรูปแบบเวลา"""
    
    TIME_24H  = re.compile(r'\b([01]?\d|2[0-3]):([0-5]\d)(?::([0-5]\d)(?:\.(\d+))?)?\b')
    TIME_12H  = re.compile(
        r'\b(1[0-2]|0?[1-9]):([0-5]\d)(?::([0-5]\d))?'
        r'\s*([AaPp][Mm])\b'
    )
    UNIX_TS = re.compile(r'\b(1[0-9]{9}|[2-9][0-9]{9})\b')
    
    @classmethod
    def parse_time_12h(cls, text: str) -> Optional[Dict]:
        m = cls.TIME_12H.search(text)
        if not m:
            return None
        
        hour = int(m.group(1))
        minute = int(m.group(2))
        second = int(m.group(3)) if m.group(3) else 0
        ampm = m.group(4).upper()
        
        if ampm == 'PM' and hour != 12:
            hour += 12
        elif ampm == 'AM' and hour == 12:
            hour = 0
        
        return {
            'hour_24': hour,
            'hour_12': int(m.group(1)),
            'minute': minute,
            'second': second,
            'ampm': ampm,
            'formatted_24h': f"{hour:02d}:{minute:02d}:{second:02d}",
        }
    
    @classmethod
    def parse_duration_seconds(cls, text: str) -> int:
        total = 0
        m = re.search(r'\b(\d+):(\d+):(\d+)\b', text)
        if m:
            return int(m.group(1)) * 3600 + int(m.group(2)) * 60 + int(m.group(3))
        m = re.search(r'\b(\d+):(\d+)\b', text)
        if m:
            return int(m.group(1)) * 60 + int(m.group(2))
        h = re.search(r'(\d+)\s*h(?:ours?)?', text, re.IGNORECASE)
        mn = re.search(r'(\d+)\s*m(?:in(?:utes?)?)?(?!\w)', text, re.IGNORECASE)
        s = re.search(r'(\d+)\s*s(?:ecs?|econds?)?', text, re.IGNORECASE)
        if h: total += int(h.group(1)) * 3600
        if mn: total += int(mn.group(1)) * 60
        if s: total += int(s.group(1))
        return total


# ทดสอบ
time_examples = [
    "Meeting starts at 2:30 PM",
    "Server uptime: 1h 23m 45s",
    "Duration: 01:30:00",
    "Call me at 9:00 AM or 3:45 PM",
    "Midnight = 12:00 AM = 00:00",
]

tp = TimePatterns

print("Time Parsing:")
print("=" * 60)

for text in time_examples:
    parsed = tp.parse_time_12h(text)
    duration = tp.parse_duration_seconds(text)
    print(f"\n  Text: {text}")
    if parsed:
        print(f"    12h: {parsed['hour_12']}:{parsed['minute']:02d} {parsed['ampm']}"
              f" → 24h: {parsed['formatted_24h']}")
    if duration:
        h, m, s = duration // 3600, (duration % 3600) // 60, duration % 60
        print(f"    Duration: {duration}s ({h:02d}:{m:02d}:{s:02d})")
```

---

## 30.3 Log Timestamp Normalization

```python
import re
from typing import Optional

class TimestampNormalizer:
    """Normalize timestamps จาก log formats ต่างๆ"""
    
    APACHE_TS = re.compile(
        r'\[(\d{2})/([A-Za-z]{3})/(\d{4}):(\d{2}):(\d{2}):(\d{2})\s+([+-]\d{4})\]'
    )
    
    SYSLOG_TS = re.compile(
        r'(Jan|Feb|Mar|Apr|May|Jun|Jul|Aug|Sep|Oct|Nov|Dec)\s+(\d{1,2})\s+(\d{2}):(\d{2}):(\d{2})',
        re.IGNORECASE
    )
    
    ISO_TS = re.compile(
        r'(\d{4})-(\d{2})-(\d{2})T(\d{2}):(\d{2}):(\d{2})(?:\.(\d+))?(Z|[+-]\d{2}:?\d{2})?'
    )
    
    FORMATS = {
        'apache': re.compile(r'\d{2}/\w{3}/\d{4}:\d{2}:\d{2}:\d{2}\s+[+-]\d{4}'),
        'iso':    re.compile(r'\d{4}-\d{2}-\d{2}[T ]\d{2}:\d{2}:\d{2}'),
        'syslog': re.compile(r'(?:Jan|Feb|Mar|Apr|May|Jun|Jul|Aug|Sep|Oct|Nov|Dec)\s+\d{1,2}\s+\d{2}:\d{2}:\d{2}'),
        'unix':   re.compile(r'\b\d{10}\b'),
    }
    
    MONTH_ABBR = {
        'Jan': '01', 'Feb': '02', 'Mar': '03', 'Apr': '04',
        'May': '05', 'Jun': '06', 'Jul': '07', 'Aug': '08',
        'Sep': '09', 'Oct': '10', 'Nov': '11', 'Dec': '12',
    }
    
    @classmethod
    def detect_format(cls, text: str) -> Optional[str]:
        for fmt, pattern in cls.FORMATS.items():
            if pattern.search(text):
                return fmt
        return None
    
    @classmethod
    def normalize_apache(cls, text: str) -> str:
        def replace(m):
            day, mon, year = m.group(1), m.group(2), m.group(3)
            h, mi, s = m.group(4), m.group(5), m.group(6)
            tz = m.group(7)
            month = cls.MONTH_ABBR.get(mon[:1].upper() + mon[1:].lower(), '01')
            tz_formatted = f"{tz[:3]}:{tz[3:]}"
            return f"{year}-{month}-{day}T{h}:{mi}:{s}{tz_formatted}"
        return cls.APACHE_TS.sub(replace, text)
    
    @classmethod
    def extract_timestamp(cls, log_line: str) -> Optional[str]:
        for fmt, pattern in cls.FORMATS.items():
            m = pattern.search(log_line)
            if m:
                return m.group(0)
        return None


# ทดสอบ
log_lines = [
    '192.168.1.1 - - [02/Oct/2026:14:30:01 +0700] "GET /index.html HTTP/1.1" 200 1234',
    '2026-10-02T14:30:02.123+07:00 INFO Application started',
    'Oct  2 14:30:03 server sshd[1234]: Accepted publickey for user',
    '1727848203 DEBUG Processing request #456',
]

normalizer = TimestampNormalizer

print("Timestamp Normalization:")
print("=" * 60)

for line in log_lines:
    fmt = normalizer.detect_format(line)
    ts = normalizer.extract_timestamp(line)
    print(f"\n  [{fmt or 'unknown':8}] {ts}")
    if fmt == 'apache':
        normalized = normalizer.normalize_apache(line)
        new_ts = normalizer.extract_timestamp(normalized)
        print(f"  [normalized] {new_ts}")
```

---

## 30.4 สรุป Part 30

```
Date/Time Patterns:
ISO date:     \b\d{4}-(0[1-9]|1[0-2])-(0[1-9]|[12]\d|3[01])\b
ISO datetime: ISO date + T\d{2}:\d{2}:\d{2}(Z|[+-]\d{2}:\d{2})?
Time 24h:     \b([01]?\d|2[0-3]):([0-5]\d)(?::([0-5]\d))?\b
Time 12h:     \b(1[0-2]|0?[1-9]):([0-5]\d)(?::([0-5]\d))?\s*[AaPp][Mm]\b
Apache TS:    \[(\d{2})/(\w{3})/(\d{4}):(\d{2}):(\d{2}):(\d{2})\s+[+-]\d{4}\]
Duration:     (\d+):(\d+):(\d+) or (\d+)h\s*(\d+)m\s*(\d+)s

Thai dates:
- DD/MM/YYYY พ.ศ. format
- day monthName year (BE)
- Convert BE to CE: BE - 543

Tips:
- Validate month range (1-12) and day range (1-31)
- Use specific patterns to avoid false positives
- Normalize to ISO 8601 for comparison
- Store timestamps as UTC
```

---

*[← Part 29: Network Patterns](part-29-network-patterns.md) | [→ Part 31: Markdown Parsing](part-31-markdown-parsing.md)*
