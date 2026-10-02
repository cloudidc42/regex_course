# Part 14: Dates & Times — การจับคู่วันที่และเวลา

> **ระดับ:** พื้นฐาน-กลาง | **เวลาเรียน:** ~80 นาที | **ข้อกำหนด:** Part 01-13

---

## 14.1 ความซับซ้อนของ Date/Time Formats

```
รูปแบบวันที่ทั่วโลก:
- 2026-10-02         ISO 8601 (แนะนำ)
- 02/10/2026         UK/Thai format (DD/MM/YYYY)
- 10/02/2026         US format (MM/DD/YYYY)
- 02-10-2026         DD-MM-YYYY
- October 2, 2026    English long form
- 2 ตุลาคม 2569      Thai format
- 20261002           Compact ISO

รูปแบบเวลา:
- 14:30:00           24-hour
- 2:30 PM            12-hour AM/PM
- 14:30:00.000       With milliseconds
- 14:30:00+07:00     With timezone offset
- 2026-10-02T14:30:00Z  ISO 8601 datetime
```

---

## 14.2 ISO 8601 Date Patterns

```python
import re
from datetime import datetime, date, time, timedelta
from typing import Optional

# ============================================
# ISO 8601 Patterns
# ============================================

class ISO8601Parser:
    
    # Date: YYYY-MM-DD
    DATE_PATTERN = re.compile(
        r'^(?P<year>\d{4})'
        r'-(?P<month>0[1-9]|1[0-2])'
        r'-(?P<day>0[1-9]|[12]\d|3[01])$'
    )
    
    # Time: HH:MM:SS[.fff][Z|+HH:MM]
    TIME_PATTERN = re.compile(
        r'^(?P<hour>[01]\d|2[0-3])'
        r':(?P<minute>[0-5]\d)'
        r'(?::(?P<second>[0-5]\d)(?:\.(?P<ms>\d{1,6}))?)?'
        r'(?P<tz>Z|[+-][01]\d:[0-5]\d)?$'
    )
    
    # Datetime: YYYY-MM-DDTHH:MM:SS
    DATETIME_PATTERN = re.compile(
        r'^(?P<year>\d{4})'
        r'-(?P<month>0[1-9]|1[0-2])'
        r'-(?P<day>0[1-9]|[12]\d|3[01])'
        r'[T ]'
        r'(?P<hour>[01]\d|2[0-3])'
        r':(?P<minute>[0-5]\d)'
        r'(?::(?P<second>[0-5]\d)(?:\.(?P<ms>\d{1,9}))?)?'
        r'(?P<tz>Z|[+-][01]\d:[0-5]\d)?$'
    )
    
    # Extract datetimes from text
    DATETIME_EXTRACT = re.compile(
        r'\d{4}-(?:0[1-9]|1[0-2])-(?:0[1-9]|[12]\d|3[01])'
        r'(?:[T ][012]\d:[0-5]\d(?::[0-5]\d(?:\.\d+)?)?(?:Z|[+-]\d{2}:\d{2})?)?'
    )
    
    @classmethod
    def parse_date(cls, s: str) -> Optional[dict]:
        m = cls.DATE_PATTERN.match(s.strip())
        if not m:
            return None
        
        year, month, day = int(m.group('year')), int(m.group('month')), int(m.group('day'))
        
        # Validate actual date
        try:
            d = date(year, month, day)
            return {
                'date': d,
                'year': year,
                'month': month,
                'day': day,
                'weekday': d.strftime('%A'),
                'iso': d.isoformat(),
            }
        except ValueError as e:
            return {'error': str(e)}
    
    @classmethod
    def parse_datetime(cls, s: str) -> Optional[dict]:
        m = cls.DATETIME_PATTERN.match(s.strip())
        if not m:
            return None
        
        d = m.groupdict()
        
        try:
            year = int(d['year'])
            month = int(d['month'])
            day = int(d['day'])
            hour = int(d['hour'])
            minute = int(d['minute'])
            second = int(d['second'] or 0)
            ms_str = d.get('ms') or '0'
            # Normalize to microseconds
            microsecond = int(ms_str.ljust(6, '0')[:6])
            tz_str = d.get('tz')
            
            dt = datetime(year, month, day, hour, minute, second, microsecond)
            
            return {
                'datetime': dt,
                'iso': dt.isoformat(),
                'timezone': tz_str,
                'components': {
                    'year': year, 'month': month, 'day': day,
                    'hour': hour, 'minute': minute, 'second': second,
                    'microsecond': microsecond,
                },
            }
        except ValueError as e:
            return {'error': str(e)}
    
    @classmethod
    def extract_all(cls, text: str) -> list:
        """Extract all ISO datetimes from text"""
        return cls.DATETIME_EXTRACT.findall(text)


# ทดสอบ
dates = [
    "2026-10-02",
    "2026-13-01",    # invalid month
    "2026-02-29",    # invalid (2026 not leap year)
    "2024-02-29",    # valid (2024 is leap year)
    "2026-10-31",
]

print("ISO Date Parsing:")
for d in dates:
    result = ISO8601Parser.parse_date(d)
    if result and 'error' not in result and 'date' in result:
        print(f"  ✅ {d} → {result['weekday']}")
    else:
        err = result.get('error', 'No match') if result else 'No match'
        print(f"  ❌ {d} → {err}")

# Datetime examples
datetimes = [
    "2026-10-02T14:30:00",
    "2026-10-02T14:30:00Z",
    "2026-10-02T14:30:00+07:00",
    "2026-10-02 14:30:00.500",
    "2026-10-02T25:00:00",    # invalid hour
]

print("\nISO Datetime Parsing:")
for dt in datetimes:
    result = ISO8601Parser.parse_datetime(dt)
    if result and 'error' not in result:
        print(f"  ✅ {dt:35} → {result['iso']}")
    else:
        err = result.get('error', 'No match') if result else 'No match'
        print(f"  ❌ {dt:35} → {err}")

# Extract from text
log_text = """
Server log:
[2026-10-02T09:15:00Z] Server started
[2026-10-02T09:15:30.123Z] Connected
[2026-10-02T10:00:00+07:00] Job scheduled
2026-10-02 14:30:00 - Backup complete
"""

print("\nExtracted datetimes:")
for dt in ISO8601Parser.extract_all(log_text):
    print(f"  {dt}")
```

---

## 14.3 Multi-Format Date Parser

```python
import re
from datetime import datetime, date
from typing import Optional, Tuple

# เดือนภาษาอังกฤษ
MONTH_NAMES = {
    'january': 1, 'jan': 1,
    'february': 2, 'feb': 2,
    'march': 3, 'mar': 3,
    'april': 4, 'apr': 4,
    'may': 5,
    'june': 6, 'jun': 6,
    'july': 7, 'jul': 7,
    'august': 8, 'aug': 8,
    'september': 9, 'sep': 9, 'sept': 9,
    'october': 10, 'oct': 10,
    'november': 11, 'nov': 11,
    'december': 12, 'dec': 12,
}

# เดือนภาษาไทย
THAI_MONTHS = {
    'มกราคม': 1, 'ม.ค.': 1,
    'กุมภาพันธ์': 2, 'ก.พ.': 2,
    'มีนาคม': 3, 'มี.ค.': 3,
    'เมษายน': 4, 'เม.ย.': 4,
    'พฤษภาคม': 5, 'พ.ค.': 5,
    'มิถุนายน': 6, 'มิ.ย.': 6,
    'กรกฎาคม': 7, 'ก.ค.': 7,
    'สิงหาคม': 8, 'ส.ค.': 8,
    'กันยายน': 9, 'ก.ย.': 9,
    'ตุลาคม': 10, 'ต.ค.': 10,
    'พฤศจิกายน': 11, 'พ.ย.': 11,
    'ธันวาคม': 12, 'ธ.ค.': 12,
}


class MultiFormatDateParser:
    """
    Parse วันที่จากหลาย formats
    """
    
    FORMATS = [
        # ISO 8601
        (re.compile(r'^(\d{4})-(\d{2})-(\d{2})$'), 'iso', 'YYYY-MM-DD'),
        # Compact ISO
        (re.compile(r'^(\d{4})(\d{2})(\d{2})$'), 'iso_compact', 'YYYYMMDD'),
        # UK/Thai: DD/MM/YYYY
        (re.compile(r'^(\d{1,2})[/\-.]( \d{1,2})[/\-.]( \d{4})$'), 'dmy', 'DD/MM/YYYY'),
        # US: MM/DD/YYYY
        (re.compile(r'^(\d{1,2})/(\d{1,2})/(\d{4})$'), 'mdy', 'MM/DD/YYYY'),
        # With month name: "2 October 2026" or "October 2, 2026"
        (re.compile(
            r'^(\d{1,2})\s+(' + '|'.join(MONTH_NAMES.keys()) + r')\s+(\d{4})$',
            re.IGNORECASE
        ), 'dmy_name', 'D Month YYYY'),
        (re.compile(
            r'^(' + '|'.join(MONTH_NAMES.keys()) + r')\s+(\d{1,2}),?\s+(\d{4})$',
            re.IGNORECASE
        ), 'mdy_name', 'Month D, YYYY'),
    ]
    
    THAI_FORMAT = re.compile(
        r'^(\d{1,2})\s*(' + '|'.join(re.escape(k) for k in THAI_MONTHS.keys()) + r')\s*(\d{4})$'
    )
    
    @classmethod
    def parse(cls, date_str: str) -> Optional[dict]:
        s = date_str.strip()
        
        # Try Thai format
        m = cls.THAI_FORMAT.match(s)
        if m:
            day = int(m.group(1))
            month_name = m.group(2)
            year_be = int(m.group(3))
            month = THAI_MONTHS.get(month_name)
            if month:
                year_ce = year_be - 543  # แปลงจาก พ.ศ. เป็น ค.ศ.
                try:
                    d = date(year_ce, month, day)
                    return cls._make_result(d, 'thai', date_str, year_be=year_be)
                except ValueError:
                    pass
        
        # Try other formats
        for pattern, fmt_type, fmt_desc in cls.FORMATS:
            m = pattern.match(s)
            if not m:
                continue
            
            groups = m.groups()
            
            try:
                if fmt_type == 'iso':
                    y, mo, d = int(groups[0]), int(groups[1]), int(groups[2])
                elif fmt_type == 'iso_compact':
                    y, mo, d = int(groups[0]), int(groups[1]), int(groups[2])
                elif fmt_type == 'dmy':
                    d, mo, y = int(groups[0]), int(groups[1]), int(groups[2])
                elif fmt_type == 'mdy':
                    mo, d, y = int(groups[0]), int(groups[1]), int(groups[2])
                elif fmt_type == 'dmy_name':
                    d = int(groups[0])
                    mo = MONTH_NAMES[groups[1].lower()]
                    y = int(groups[2])
                elif fmt_type == 'mdy_name':
                    mo = MONTH_NAMES[groups[0].lower()]
                    d = int(groups[1])
                    y = int(groups[2])
                else:
                    continue
                
                dt = date(y, mo, d)
                return cls._make_result(dt, fmt_type, date_str)
                
            except (ValueError, KeyError):
                continue
        
        return None
    
    @classmethod
    def _make_result(cls, d: date, fmt_type: str, original: str, year_be: int = None) -> dict:
        result = {
            'date': d,
            'original': original,
            'format': fmt_type,
            'iso': d.isoformat(),
            'year': d.year,
            'month': d.month,
            'day': d.day,
            'weekday': d.strftime('%A'),
            'weekday_th': ['จันทร์', 'อังคาร', 'พุธ', 'พฤหัสบดี', 'ศุกร์', 'เสาร์', 'อาทิตย์'][d.weekday()],
        }
        if year_be:
            result['year_be'] = year_be
            result['year_th'] = f"พ.ศ. {year_be}"
        return result
    
    @classmethod
    def to_thai_format(cls, d: date) -> str:
        """แปลงวันที่เป็น format ไทย"""
        thai_months = ['มกราคม', 'กุมภาพันธ์', 'มีนาคม', 'เมษายน',
                       'พฤษภาคม', 'มิถุนายน', 'กรกฎาคม', 'สิงหาคม',
                       'กันยายน', 'ตุลาคม', 'พฤศจิกายน', 'ธันวาคม']
        return f"{d.day} {thai_months[d.month-1]} {d.year + 543}"


# ทดสอบ
test_dates = [
    "2026-10-02",
    "20261002",
    "02/10/2026",
    "10/02/2026",
    "2 October 2026",
    "October 2, 2026",
    "2 ตุลาคม 2569",
    "2 ต.ค. 2569",
    "invalid date",
    "2026-13-01",    # invalid
]

print("Multi-Format Date Parsing:")
print("=" * 60)
for ds in test_dates:
    result = MultiFormatDateParser.parse(ds)
    if result and 'date' in result:
        print(f"  ✅ {ds:25} → ISO:{result['iso']} ({result['weekday_th']})")
    else:
        print(f"  ❌ {ds:25} → parse failed")

# Thai format conversion
from datetime import date as dt_date
today = dt_date(2026, 10, 2)
print(f"\n  Thai format: {MultiFormatDateParser.to_thai_format(today)}")
```

---

## 14.4 Time Parser

```python
import re
from typing import Optional

class TimeParser:
    """Parse time strings in various formats"""
    
    # 24-hour format
    TIME_24H = re.compile(
        r'^(?P<hour>2[0-3]|[01]?\d)'
        r':(?P<minute>[0-5]\d)'
        r'(?::(?P<second>[0-5]\d)(?:\.(?P<ms>\d{1,6}))?)?$'
    )
    
    # 12-hour format
    TIME_12H = re.compile(
        r'^(?P<hour>1[0-2]|0?[1-9])'
        r':(?P<minute>[0-5]\d)'
        r'(?::(?P<second>[0-5]\d))?'
        r'\s*(?P<ampm>AM|PM|am|pm)$',
        re.IGNORECASE
    )
    
    # Timezone offset: +07:00 or -05:30
    TZ_OFFSET = re.compile(
        r'^(?P<sign>[+-])(?P<hours>[01]\d):(?P<minutes>[0-5]\d)$'
    )
    
    @classmethod
    def parse(cls, time_str: str) -> Optional[dict]:
        s = time_str.strip()
        
        # Try 12-hour
        m = cls.TIME_12H.match(s)
        if m:
            d = m.groupdict()
            hour = int(d['hour'])
            minute = int(d['minute'])
            second = int(d['second'] or 0)
            ampm = d['ampm'].upper()
            
            # Convert to 24-hour
            if ampm == 'AM':
                if hour == 12:
                    hour = 0
            else:  # PM
                if hour != 12:
                    hour += 12
            
            return {
                'hour_24': hour,
                'minute': minute,
                'second': second,
                'ampm': ampm,
                'format': '12h',
                'formatted_24h': f"{hour:02d}:{minute:02d}:{second:02d}",
                'formatted_12h': time_str,
            }
        
        # Try 24-hour
        m = cls.TIME_24H.match(s)
        if m:
            d = m.groupdict()
            hour = int(d['hour'])
            minute = int(d['minute'])
            second = int(d['second'] or 0)
            ms = int((d['ms'] or '0').ljust(6, '0')[:6])
            
            # 12-hour conversion
            if hour == 0:
                hour12, ampm = 12, 'AM'
            elif hour < 12:
                hour12, ampm = hour, 'AM'
            elif hour == 12:
                hour12, ampm = 12, 'PM'
            else:
                hour12, ampm = hour - 12, 'PM'
            
            return {
                'hour_24': hour,
                'minute': minute,
                'second': second,
                'microsecond': ms,
                'format': '24h',
                'formatted_24h': f"{hour:02d}:{minute:02d}:{second:02d}",
                'formatted_12h': f"{hour12}:{minute:02d}:{second:02d} {ampm}",
            }
        
        return None


# ทดสอบ
time_tests = [
    "14:30",
    "14:30:00",
    "14:30:00.500",
    "2:30 PM",
    "02:30 AM",
    "12:00 PM",    # noon
    "12:00 AM",    # midnight
    "23:59:59",
    "invalid",
]

print("Time Parsing:")
print("=" * 60)
for t in time_tests:
    result = TimeParser.parse(t)
    if result:
        print(f"  ✅ {t:20} → 24h: {result['formatted_24h']}  12h: {result['formatted_12h']}")
    else:
        print(f"  ❌ {t:20} → parse failed")
```

---

## 14.5 Date Range และ Duration Patterns

```python
import re
from datetime import datetime, timedelta

class DateRangeParser:
    """Parse date ranges จาก text"""
    
    # "from ... to ..."
    FROM_TO = re.compile(
        r'from\s+(\d{4}-\d{2}-\d{2})\s+to\s+(\d{4}-\d{2}-\d{2})',
        re.IGNORECASE
    )
    
    # "YYYY-MM-DD - YYYY-MM-DD"
    DASH_RANGE = re.compile(
        r'(\d{4}-\d{2}-\d{2})\s*[-–—]\s*(\d{4}-\d{2}-\d{2})'
    )
    
    # Duration: "3 days", "2 weeks", "1 month"
    DURATION = re.compile(
        r'(\d+)\s+(second|minute|hour|day|week|month|year)s?',
        re.IGNORECASE
    )
    
    @classmethod
    def parse_duration(cls, text: str) -> Optional[timedelta]:
        """แปลง duration text เป็น timedelta"""
        m = cls.DURATION.search(text)
        if not m:
            return None
        
        amount = int(m.group(1))
        unit = m.group(2).lower()
        
        multipliers = {
            'second': 1,
            'minute': 60,
            'hour': 3600,
            'day': 86400,
            'week': 604800,
            'month': 2592000,    # approximate (30 days)
            'year': 31536000,    # approximate (365 days)
        }
        
        seconds = amount * multipliers.get(unit, 0)
        return timedelta(seconds=seconds)
    
    @classmethod
    def extract_ranges(cls, text: str) -> list:
        ranges = []
        
        for m in cls.FROM_TO.finditer(text):
            ranges.append({
                'type': 'from_to',
                'start': m.group(1),
                'end': m.group(2),
                'text': m.group(0),
            })
        
        for m in cls.DASH_RANGE.finditer(text):
            ranges.append({
                'type': 'dash',
                'start': m.group(1),
                'end': m.group(2),
                'text': m.group(0),
            })
        
        return ranges


# ทดสอบ Duration
duration_tests = [
    "expires in 30 days",
    "valid for 2 weeks",
    "timeout after 5 minutes",
    "contract for 1 year",
    "session expires in 3600 seconds",
]

print("\nDuration Parsing:")
for text in duration_tests:
    td = DateRangeParser.parse_duration(text)
    if td:
        print(f"  '{text}' → {td}")

# ทดสอบ Range
range_text = """
Project phases:
- Phase 1: from 2026-01-01 to 2026-03-31
- Phase 2: 2026-04-01 - 2026-06-30
- Phase 3: from 2026-07-01 to 2026-09-30
"""

print("\nDate Ranges:")
for r in DateRangeParser.extract_ranges(range_text):
    print(f"  {r['start']} → {r['end']}  ({r['type']})")
```

---

## 14.6 Log Timestamp Parser

```python
import re
from datetime import datetime
from typing import Optional

class LogTimestampParser:
    """
    Parse timestamps จาก log files ต่างๆ
    """
    
    FORMATS = {
        'apache': (
            re.compile(r'\[(\d{2}/\w+/\d{4}:\d{2}:\d{2}:\d{2}\s[+-]\d{4})\]'),
            '%d/%b/%Y:%H:%M:%S %z'
        ),
        'nginx': (
            re.compile(r'(\d{4}/\d{2}/\d{2}\s\d{2}:\d{2}:\d{2})'),
            '%Y/%m/%d %H:%M:%S'
        ),
        'syslog': (
            re.compile(r'(\w{3}\s+\d{1,2}\s\d{2}:\d{2}:\d{2})'),
            '%b %d %H:%M:%S'
        ),
        'iso': (
            re.compile(r'(\d{4}-\d{2}-\d{2}[T ]\d{2}:\d{2}:\d{2}(?:\.\d+)?(?:Z|[+-]\d{2}:\d{2})?)'),
            None  # handle separately
        ),
        'bracket_iso': (
            re.compile(r'\[(\d{4}-\d{2}-\d{2}[T ]\d{2}:\d{2}:\d{2}(?:\.\d+)?)\]'),
            '%Y-%m-%dT%H:%M:%S'
        ),
    }
    
    @classmethod
    def extract(cls, log_line: str) -> list:
        """Extract timestamps จาก log line"""
        found = []
        
        for fmt_name, (pattern, strpfmt) in cls.FORMATS.items():
            for m in pattern.finditer(log_line):
                ts_str = m.group(1)
                found.append({
                    'format': fmt_name,
                    'raw': ts_str,
                    'position': m.span(),
                })
        
        return found
    
    @classmethod
    def normalize(cls, log_line: str) -> str:
        """แปลง timestamp ใน log เป็น ISO 8601"""
        # ตัวอย่าง: แปลง Apache format → ISO
        apache_re = re.compile(
            r'\[(\d{2})/(\w{3})/(\d{4}):(\d{2}):(\d{2}):(\d{2})\s([+-]\d{4})\]'
        )
        
        MONTH_ABBR = {
            'Jan': '01', 'Feb': '02', 'Mar': '03', 'Apr': '04',
            'May': '05', 'Jun': '06', 'Jul': '07', 'Aug': '08',
            'Sep': '09', 'Oct': '10', 'Nov': '11', 'Dec': '12',
        }
        
        def apache_to_iso(m):
            day, month_abbr, year = m.group(1), m.group(2), m.group(3)
            hour, minute, second = m.group(4), m.group(5), m.group(6)
            tz = m.group(7)
            month = MONTH_ABBR.get(month_abbr, '01')
            tz_iso = f"{tz[:3]}:{tz[3:]}"
            return f"[{year}-{month}-{day}T{hour}:{minute}:{second}{tz_iso}]"
        
        return apache_re.sub(apache_to_iso, log_line)


# ทดสอบ
log_lines = [
    '192.168.1.1 - - [02/Oct/2026:14:30:00 +0700] "GET /index.html HTTP/1.1" 200 1234',
    '2026/10/02 14:30:00 [error] connection reset',
    '[2026-10-02T14:30:00] INFO Request received',
    'Oct  2 14:30:00 hostname service: message',
]

print("Log Timestamp Extraction:")
print("=" * 70)
for line in log_lines:
    print(f"\nLine: {line[:60]}...")
    timestamps = LogTimestampParser.extract(line)
    for ts in timestamps:
        print(f"  [{ts['format']}] {ts['raw']}")
    
    # Normalize Apache
    if '[' in line and '/Oct/' in line:
        normalized = LogTimestampParser.normalize(line)
        print(f"  Normalized: {normalized[:70]}")
```

---

## 14.7 Relative Date Expressions

```python
import re
from datetime import datetime, timedelta, date

class RelativeDateParser:
    """Parse relative date expressions"""
    
    # English relative dates
    ENG_RELATIVE = re.compile(
        r'(?P<direction>last|next|in|ago)?'
        r'\s*(?P<amount>\d+)\s*'
        r'(?P<unit>second|minute|hour|day|week|month|year)s?'
        r'(?:\s*ago)?',
        re.IGNORECASE
    )
    
    UNIT_SECONDS = {
        'second': 1, 'minute': 60, 'hour': 3600,
        'day': 86400, 'week': 604800,
        'month': 2592000, 'year': 31536000,
    }
    
    @classmethod
    def parse(cls, text: str, reference: datetime = None) -> Optional[datetime]:
        if reference is None:
            reference = datetime(2026, 10, 2, 14, 30, 0)
        
        text_lower = text.lower().strip()
        
        if text_lower in ('today', 'วันนี้'):
            return reference.replace(hour=0, minute=0, second=0, microsecond=0)
        if text_lower in ('yesterday', 'เมื่อวาน'):
            return (reference - timedelta(days=1)).replace(hour=0, minute=0, second=0, microsecond=0)
        if text_lower in ('tomorrow', 'พรุ่งนี้'):
            return (reference + timedelta(days=1)).replace(hour=0, minute=0, second=0, microsecond=0)
        if text_lower in ('now', 'ตอนนี้'):
            return reference
        
        m = cls.ENG_RELATIVE.search(text)
        if m:
            d = m.groupdict()
            amount = int(d['amount'])
            unit = d['unit'].lower()
            direction = (d['direction'] or '').lower()
            
            seconds = cls.UNIT_SECONDS.get(unit, 0) * amount
            
            if direction in ('last', 'ago') or 'ago' in text.lower():
                return reference - timedelta(seconds=seconds)
            else:
                return reference + timedelta(seconds=seconds)
        
        return None


# ทดสอบ
ref_date = datetime(2026, 10, 2, 14, 30, 0)
relative_tests = [
    "today", "yesterday", "tomorrow",
    "3 days ago", "in 2 weeks",
    "last 30 days", "next 1 year", "5 minutes ago",
]

print("\nRelative Date Parsing (ref: 2026-10-02 14:30:00):")
for text in relative_tests:
    result = RelativeDateParser.parse(text, ref_date)
    if result:
        print(f"  '{text:20}' → {result.strftime('%Y-%m-%d %H:%M:%S')}")
    else:
        print(f"  '{text:20}' → parse failed")
```

---

## 14.8 Date Validation Functions

```python
import re
from datetime import date

def is_leap_year(year: int) -> bool:
    """ตรวจสอบปีอธิกสุรทิน"""
    return (year % 4 == 0 and year % 100 != 0) or (year % 400 == 0)

def validate_date_range(date_str: str, 
                        min_year: int = 1900, 
                        max_year: int = 2100) -> dict:
    """Validate วันที่ว่าอยู่ใน range ที่กำหนดหรือไม่"""
    pattern = re.compile(r'^(\d{4})-(\d{2})-(\d{2})$')
    m = pattern.match(date_str.strip())
    
    if not m:
        return {'valid': False, 'error': 'Invalid date format (expected YYYY-MM-DD)'}
    
    year, month, day = int(m.group(1)), int(m.group(2)), int(m.group(3))
    
    if not (min_year <= year <= max_year):
        return {'valid': False, 'error': f'Year {year} out of range [{min_year}, {max_year}]'}
    
    if not (1 <= month <= 12):
        return {'valid': False, 'error': f'Invalid month: {month}'}
    
    days_in_month = [31, 29 if is_leap_year(year) else 28, 31, 30, 31, 30,
                     31, 31, 30, 31, 30, 31]
    
    max_day = days_in_month[month - 1]
    if not (1 <= day <= max_day):
        return {'valid': False, 'error': f'Invalid day {day} for {year}-{month:02d} (max {max_day})'}
    
    return {'valid': True, 'date': date(year, month, day)}


# ทดสอบ
date_tests = [
    "2026-10-02", "2024-02-29", "2023-02-29",
    "2026-13-01", "2026-04-31", "1899-01-01", "not-a-date",
]

print("\nDate Validation:")
for ds in date_tests:
    result = validate_date_range(ds)
    icon = '✅' if result['valid'] else '❌'
    if result['valid']:
        d = result['date']
        print(f"  {icon} {ds:15} → {d.strftime('%A, %d %B %Y')}")
    else:
        print(f"  {icon} {ds:15} → {result['error']}")

print("\nLeap Year Check:")
for y in [2020, 2021, 2024, 2025, 2026, 2100, 2000]:
    leap = is_leap_year(y)
    print(f"  {'✅' if leap else '  '} {y}: {'Leap year' if leap else 'Not leap year'}")
```

---

## 14.9 JavaScript Date Patterns

```javascript
// ============================================
// JavaScript Date Validation and Parsing
// ============================================

function isValidISODate(str) {
    const pattern = /^\d{4}-(?:0[1-9]|1[0-2])-(?:0[1-9]|[12]\d|3[01])$/;
    if (!pattern.test(str)) return false;
    
    const [year, month, day] = str.split('-').map(Number);
    const date = new Date(year, month - 1, day);
    
    return date.getFullYear() === year &&
           date.getMonth() === month - 1 &&
           date.getDate() === day;
}

function parseDate(str) {
    const formats = [
        {
            pattern: /^(\d{4})-(\d{2})-(\d{2})$/,
            parse: m => new Date(+m[1], +m[2]-1, +m[3]),
            name: 'ISO'
        },
        {
            pattern: /^(\d{1,2})\/(\d{1,2})\/(\d{4})$/,
            parse: m => new Date(+m[3], +m[2]-1, +m[1]),  // DD/MM/YYYY
            name: 'DD/MM/YYYY'
        },
        {
            pattern: /^(\d{4})(\d{2})(\d{2})$/,
            parse: m => new Date(+m[1], +m[2]-1, +m[3]),
            name: 'YYYYMMDD'
        },
    ];
    
    for (const { pattern, parse, name } of formats) {
        const m = str.trim().match(pattern);
        if (m) {
            const date = parse(m);
            if (!isNaN(date.getTime())) {
                return { date, format: name, valid: true };
            }
        }
    }
    
    return { date: null, format: null, valid: false };
}

function formatDate(date, format = 'iso') {
    const y = date.getFullYear();
    const m = String(date.getMonth() + 1).padStart(2, '0');
    const d = String(date.getDate()).padStart(2, '0');
    
    const formats = {
        iso: `${y}-${m}-${d}`,
        dmy: `${d}/${m}/${y}`,
        mdy: `${m}/${d}/${y}`,
        thai: `${d}/${m}/${y + 543}`,  // พ.ศ.
    };
    
    return formats[format] || formats.iso;
}

// ทดสอบ
const testDates = ['2026-10-02', '02/10/2026', '20261002', 'not-a-date', '2026-13-01'];

testDates.forEach(d => {
    const result = parseDate(d);
    const icon = result.valid ? '✅' : '❌';
    if (result.valid) {
        console.log(`${icon} ${d.padEnd(15)} → ISO: ${formatDate(result.date)}`);
        console.log(`   Thai: ${formatDate(result.date, 'thai')}`);
    } else {
        console.log(`${icon} ${d}`);
    }
});
```

---

## 14.10 สรุป Part 14

```
Pattern ที่ใช้บ่อยสำหรับ Date/Time:
────────────────────────────────────────────────────────────
ISO Date:     \d{4}-(?:0[1-9]|1[0-2])-(?:0[1-9]|[12]\d|3[01])
ISO Datetime: \d{4}-\d{2}-\d{2}[T ]\d{2}:\d{2}:\d{2}
Time 24h:     (?:[01]\d|2[0-3]):[0-5]\d(?::[0-5]\d)?
Time 12h:     (?:1[0-2]|0?[1-9]):[0-5]\d\s*[AaPp][Mm]
Date DD/MM:   (?:0[1-9]|[12]\d|3[01])/(?:0[1-9]|1[0-2])/\d{4}
```

✅ **ISO 8601** — รูปแบบมาตรฐานสากล  
✅ **Multi-format parser** — รองรับหลาย format รวมถึงภาษาไทย  
✅ **Thai Buddhist Era** — แปลง พ.ศ. ↔ ค.ศ.  
✅ **Validation** — ตรวจสอบวันที่ที่ถูกต้องจริง (เช่น 29 ก.พ. ปีอธิกสุรทิน)  
✅ **Log timestamp** — Parse จาก Apache, Nginx, Syslog  
✅ **Relative dates** — "3 days ago", "in 2 weeks"  

---

*[⬅ Part 13: Phone Numbers](part-13-phone-numbers.md) | [➡ Part 15: Password Validation](part-15-password-validation.md)*
