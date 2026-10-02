# Part 73: Email & Communication Parsing

> **ระดับ:** สูง-มืออาชีพ | **เวลาเรียน:** ~85 นาที | **ข้อกำหนด:** Part 01-72

---

## 73.1 RFC 5322 Email Address Parsing

```python
import re
from typing import Dict, List, Optional, Tuple

print("Email & Communication Parsing:")
print("=" * 60)

print("\n1. RFC 5322 email address validation:")

# Full RFC 5322 addr-spec (simplified but practical)
# local-part@domain
LOCAL_PART = re.compile(
    r'[a-zA-Z0-9!#$%&\'*+/=?^_`{|}~-]+'
    r'(?:\.[a-zA-Z0-9!#$%&\'*+/=?^_`{|}~-]+)*'
)

DOMAIN_PART = re.compile(
    r'(?:[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?)'
    r'(?:\.[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?)*'
    r'\.[a-zA-Z]{2,10}'
)

EMAIL_RFC5322 = re.compile(
    r'^(?P<local>[a-zA-Z0-9!#$%&\'*+/=?^_`{|}~-]+'
    r'(?:\.[a-zA-Z0-9!#$%&\'*+/=?^_`{|}~-]+)*)'
    r'@'
    r'(?P<domain>(?:[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?)'
    r'(?:\.[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?)*'
    r'\.[a-zA-Z]{2,10})$'
)

# Quoted local part (allows spaces and special chars)
EMAIL_QUOTED = re.compile(
    r'^"(?:[^"\\]|\\.)*"@'
    r'(?:[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?)'
    r'(?:\.[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?)*'
    r'\.[a-zA-Z]{2,10}$'
)

# Display name + email: "John Doe" <john@example.com>
EMAIL_WITH_NAME = re.compile(
    r'(?:(?P<name>["\']?[^<>"\']+["\']?)\s+)?'
    r'<(?P<email>[^>]+)>'
    r'|(?P<bare>[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,10})'
)

# Plus-addressing (subaddress extension)
EMAIL_PLUS = re.compile(
    r'^(?P<user>[a-zA-Z0-9._%+-]+)'
    r'\+(?P<tag>[a-zA-Z0-9._-]+)'
    r'@(?P<domain>[a-zA-Z0-9.-]+\.[a-zA-Z]{2,10})$'
)


def parse_email_address(addr: str) -> Dict:
    addr = addr.strip()

    # Try display name + angle bracket
    m = EMAIL_WITH_NAME.match(addr)
    if m:
        email = m.group('email') or m.group('bare')
        name  = m.group('name')
    else:
        email = addr
        name  = None

    if not email:
        return {'valid': False, 'raw': addr}

    # Validate email
    em = EMAIL_RFC5322.match(email.strip())
    if not em:
        # Try quoted
        em_q = EMAIL_QUOTED.match(email.strip())
        if em_q:
            return {'valid': True, 'email': email.strip(), 'display_name': name,
                    'local': email.split('@')[0], 'domain': email.split('@')[1],
                    'quoted': True}
        return {'valid': False, 'raw': addr, 'display_name': name}

    result = {
        'valid':        True,
        'email':        email.strip().lower(),
        'display_name': name.strip('" \'') if name else None,
        'local':        em.group('local'),
        'domain':       em.group('domain').lower(),
    }

    # Check for plus addressing
    pm = EMAIL_PLUS.match(email.strip())
    if pm:
        result['plus_user'] = pm.group('user')
        result['plus_tag']  = pm.group('tag')

    return result


test_addresses = [
    'alice@example.com',
    'Alice Smith <alice@example.com>',
    '"Alice Smith" <alice.smith@company.co.uk>',
    'alice+newsletter@example.com',
    'alice+orders+2024@shop.example.com',
    'user.name+tag@example.org',
    'invalid-email',
    '@nodomain.com',
    'no@tld',
    '"quoted local"@example.com',
    'user@[192.168.1.1]',
]

print(f"\n   {'Address':<45} {'Valid':<6} {'Info'}")
print(f"   {'-'*45} {'-'*6} {'-'*30}")
for addr in test_addresses:
    p = parse_email_address(addr)
    status = '✓' if p.get('valid') else '✗'
    if p.get('valid'):
        info = p['email']
        if p.get('plus_tag'):
            info += f" [tag={p['plus_tag']}]"
        if p.get('display_name'):
            info += f" [{p['display_name']}]"
    else:
        info = 'invalid'
    print(f"   {addr[:43]:<45} {status:<6} {info[:35]}")
```

---

## 73.2 Email Header Parsing (RFC 5322 Headers)

```python
import re
from typing import Dict, List

print("\nEmail Header Parsing:")
print("=" * 60)

# RFC 5322 header field
HEADER_FIELD = re.compile(
    r'^(?P<name>[A-Za-z][A-Za-z0-9-]*):\s*(?P<value>.+?)$',
    re.MULTILINE
)

# Folded header (continuation line starts with whitespace)
HEADER_FOLD = re.compile(r'\r?\n[ \t]+')

# From/To/CC with display names
FROM_TO = re.compile(
    r'(?:(?P<name>[^<,]+?)\s*<(?P<email>[^>]+)>|(?P<bare_email>[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}))',
    re.IGNORECASE
)

# Message-ID
MESSAGE_ID = re.compile(r'<(?P<id>[^>]+)>')

# Date header (RFC 2822)
DATE_HEADER = re.compile(
    r'(?P<dow>Mon|Tue|Wed|Thu|Fri|Sat|Sun),\s+'
    r'(?P<day>\d{1,2})\s+'
    r'(?P<month>Jan|Feb|Mar|Apr|May|Jun|Jul|Aug|Sep|Oct|Nov|Dec)\s+'
    r'(?P<year>\d{4})\s+'
    r'(?P<hour>\d{2}):(?P<min>\d{2})(?::(?P<sec>\d{2}))?\s+'
    r'(?P<tz>[+-]\d{4}|[A-Z]{2,3})',
    re.IGNORECASE
)

# Authentication results
DKIM_RESULT   = re.compile(r'dkim=(?P<result>pass|fail|none|neutral|temperror|permerror)', re.IGNORECASE)
SPF_RESULT    = re.compile(r'spf=(?P<result>pass|fail|softfail|none|neutral|temperror|permerror)', re.IGNORECASE)
DMARC_RESULT  = re.compile(r'dmarc=(?P<result>pass|fail|none)', re.IGNORECASE)


def parse_email_headers(raw_headers: str) -> Dict:
    # Unfold folded headers
    unfolded = HEADER_FOLD.sub(' ', raw_headers)
    headers  = {}
    for m in HEADER_FIELD.finditer(unfolded):
        name  = m.group('name').lower()
        value = m.group('value').strip()
        if name in headers:
            if isinstance(headers[name], list):
                headers[name].append(value)
            else:
                headers[name] = [headers[name], value]
        else:
            headers[name] = value

    result = {'headers': headers}

    # Parse From
    from_val = headers.get('from', '')
    fm = FROM_TO.search(from_val)
    if fm:
        result['from'] = {
            'name':  fm.group('name').strip() if fm.group('name') else None,
            'email': fm.group('email') or fm.group('bare_email'),
        }

    # Parse Date
    date_val = headers.get('date', '')
    dm = DATE_HEADER.search(date_val)
    if dm:
        result['date'] = dm.groupdict()

    # Parse Message-ID
    mid_val = headers.get('message-id', '')
    mm = MESSAGE_ID.search(mid_val)
    if mm:
        result['message_id'] = mm.group('id')

    # Authentication results
    auth_val = headers.get('authentication-results', '')
    result['auth'] = {
        'dkim':  DKIM_RESULT.search(auth_val),
        'spf':   SPF_RESULT.search(auth_val),
        'dmarc': DMARC_RESULT.search(auth_val),
    }
    for k in result['auth']:
        m = result['auth'][k]
        result['auth'][k] = m.group('result') if m else 'none'

    return result


sample_headers = """From: Alice Smith <alice@example.com>
To: Bob Jones <bob@company.com>, charlie@example.org
Subject: Meeting Tomorrow - Regex Workshop
Date: Thu, 15 Feb 2024 14:30:00 +0700
Message-ID: <20240215143000.12345@example.com>
MIME-Version: 1.0
Content-Type: multipart/mixed; boundary="boundary_42"
Authentication-Results: mx.example.com;
    dkim=pass header.d=example.com;
    spf=pass smtp.mailfrom=alice@example.com;
    dmarc=pass action=none header.from=example.com
X-Spam-Score: 0.5 required=5.0
Received: from mail.example.com (192.168.1.100)
    by mx.company.com with ESMTP; Thu, 15 Feb 2024 14:30:05 +0700"""

parsed = parse_email_headers(sample_headers)
print(f"\n   Parsed headers:")
print(f"   From:       {parsed.get('from', {})}")
print(f"   Date:       {parsed.get('date', {}).get('dow')} {parsed.get('date', {}).get('day')} {parsed.get('date', {}).get('month')}")
print(f"   Message-ID: {parsed.get('message_id')}")
print(f"   Auth:       DKIM={parsed['auth']['dkim']}, SPF={parsed['auth']['spf']}, DMARC={parsed['auth']['dmarc']}")
```

---

## 73.3 MIME Multipart Parsing

```python
import re
from typing import Dict, List

print("\nMIME Multipart Parsing:")
print("=" * 60)

# MIME boundary
MIME_BOUNDARY = re.compile(r'boundary=["\']?(?P<boundary>[^\s"\';\.\r\n]+)["\']?', re.IGNORECASE)

# Content-Type with charset
CONTENT_TYPE_FULL = re.compile(
    r'^(?P<type>[a-z]+)/(?P<subtype>[a-z0-9.+-]+)'
    r'(?:;\s*charset=["\']?(?P<charset>[a-zA-Z0-9_-]+)["\']?)?'
    r'(?:;\s*boundary=["\']?(?P<boundary>[^\s"\';\.\r\n]+)["\']?)?',
    re.IGNORECASE
)

# Content-Disposition
CONTENT_DISP = re.compile(
    r'^(?P<disp>inline|attachment|form-data)'
    r'(?:;\s*name=["\']?(?P<name>[^"\';\.\r\n]+)["\']?)?'
    r'(?:;\s*filename=["\']?(?P<filename>[^"\';\.\r\n]+)["\']?)?',
    re.IGNORECASE
)


def parse_mime_part(raw_part: str) -> Dict:
    lines = raw_part.split('\n')
    headers = {}
    body_start = 0

    for i, line in enumerate(lines):
        line = line.rstrip('\r')
        if not line:
            body_start = i + 1
            break
        if ':' in line:
            name, _, value = line.partition(':')
            headers[name.lower().strip()] = value.strip()

    body = '\n'.join(lines[body_start:])
    ct   = headers.get('content-type', '')
    cd   = headers.get('content-disposition', '')
    cte  = headers.get('content-transfer-encoding', '7bit')

    ct_m = CONTENT_TYPE_FULL.match(ct)
    cd_m = CONTENT_DISP.match(cd)

    return {
        'content_type':     ct_m.group(0)[:50] if ct_m else ct,
        'mime_type':        f"{ct_m.group('type')}/{ct_m.group('subtype')}" if ct_m else None,
        'charset':          ct_m.group('charset') if ct_m else None,
        'disposition':      cd_m.group('disp') if cd_m else 'inline',
        'filename':         cd_m.group('filename') if cd_m else None,
        'part_name':        cd_m.group('name') if cd_m else None,
        'encoding':         cte.lower(),
        'body_length':      len(body),
    }


def extract_mime_parts(email_raw: str) -> List[Dict]:
    bm = MIME_BOUNDARY.search(email_raw)
    if not bm:
        return []
    boundary  = bm.group('boundary')
    delimiter = re.compile(r'--' + re.escape(boundary) + r'(?:--)?', re.MULTILINE)
    parts     = delimiter.split(email_raw)[1:]
    return [parse_mime_part(p) for p in parts if p.strip() and not p.strip().startswith('--')]


mime_email = """From: alice@example.com
To: bob@example.com
Subject: Test with Attachment
MIME-Version: 1.0
Content-Type: multipart/mixed; boundary="----=_Part_42"

------=_Part_42
Content-Type: text/plain; charset=UTF-8
Content-Transfer-Encoding: 7bit

Hello, this is the plain text body.

------=_Part_42
Content-Type: text/html; charset=UTF-8
Content-Transfer-Encoding: quoted-printable

<html><body><p>Hello =E2=80=94 this is HTML</p></body></html>

------=_Part_42
Content-Type: application/pdf
Content-Transfer-Encoding: base64
Content-Disposition: attachment; filename="document.pdf"

JVBERi0xLjQKJeLjz9MKCjEgMCBvYmoK

------=_Part_42--"""

parts = extract_mime_parts(mime_email)
print(f"\n   Found {len(parts)} MIME parts:")
for i, p in enumerate(parts, 1):
    print(f"\n   Part {i}:")
    print(f"     MIME:     {p.get('mime_type')}")
    print(f"     Encoding: {p.get('encoding')}")
    print(f"     Filename: {p.get('filename')}")
    print(f"     Size:     {p.get('body_length')} chars")
```

---

## 73.4 Phone Number Parsing

```python
import re
from typing import Dict, Optional

print("\nPhone Number Parsing:")
print("=" * 60)

# E.164 international format
E164 = re.compile(r'^\+(?P<cc>\d{1,3})(?P<num>\d{7,14})$')

# Thai phone numbers
THAI_MOBILE   = re.compile(r'^(?:0[689]\d{8}|0[2-5]\d{7,8})$')
THAI_LANDLINE = re.compile(r'^0[2-5]\d{7}$')
THAI_SPECIAL  = re.compile(r'^(?:1[0-9]{3,4}|0[01]\d{0,3})$')

# Phone extraction from free text
PHONE_IN_TEXT = re.compile(
    r'(?:\+\d{1,3}[\s.-]?)?'
    r'(?:\(?\d{2,4}\)?[\s.-]?)?'
    r'\d{3,4}[\s.-]?\d{4}'
)


def normalize_phone(raw: str) -> Optional[str]:
    digits = re.sub(r'[^\d+]', '', raw)
    if digits.startswith('+'):
        return digits
    if digits.startswith('00'):
        return '+' + digits[2:]
    if len(digits) == 10 and digits.startswith('0'):
        return '+66' + digits[1:]  # Thailand default
    return digits


def classify_thai_phone(number: str) -> str:
    clean = re.sub(r'[\s\-()]', '', number)
    if THAI_MOBILE.match(clean):
        prefix = clean[:3]
        if prefix in ('061', '062', '063', '064', '065', '066'):
            return 'mobile (DTAC)'
        elif prefix in ('080', '081', '082', '083', '085', '086', '087', '088', '089'):
            return 'mobile (AIS/TRUE)'
        elif prefix in ('090', '091', '092', '093', '095', '096', '097', '098', '099'):
            return 'mobile (TRUE/DTAC)'
        return 'mobile'
    elif THAI_LANDLINE.match(clean):
        prefix = clean[:3]
        if prefix == '02':
            return 'landline (Bangkok)'
        return 'landline (province)'
    elif THAI_SPECIAL.match(clean):
        return 'special/short number'
    return 'unknown'


phone_numbers = [
    '+66812345678',
    '081-234-5678',
    '02-123-4567',
    '(02) 123-4567',
    '+1 (555) 123-4567',
    '+44 20 7946 0958',
    '00447911123456',
    '1800-123-456',
    '091 234 5678',
    'Call me at 062-987-6543 or 02-555-0123',
]

print(f"\n   {'Raw number':<35} {'Normalized':<20} {'Type'}")
print(f"   {'-'*35} {'-'*20} {'-'*20}")
for raw in phone_numbers[:-1]:
    norm = normalize_phone(raw)
    kind = classify_thai_phone(raw)
    print(f"   {raw:<35} {norm:<20} {kind}")

print(f"\n   Extraction from text:")
text = phone_numbers[-1]
found = PHONE_IN_TEXT.findall(text)
print(f"   Text: {text!r}")
print(f"   Found: {found}")
```

---

## 73.5 สรุป Part 73

```
Email & Communication Regex Patterns:

1. Email validation tiers:
   Basic:    ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$
   RFC 5322: local@domain (local = [a-zA-Z0-9!#$%&'*+/=?^_`{|}~-]+)
   Full:     also handles quoted locals, display names, plus-addressing

2. Plus addressing:
   user+tag@domain.com → filter/label emails
   Regex: ^([a-zA-Z0-9._%+-]+)\+([a-zA-Z0-9._-]+)@(domain)$

3. Email header authentication:
   DKIM: verify sender domain signature
   SPF:  verify IP allowed to send for domain
   DMARC: policy (pass/fail/none) combining DKIM + SPF

4. MIME structure:
   Content-Type: type/subtype; charset=UTF-8; boundary="..."
   Content-Disposition: attachment; filename="..."
   Content-Transfer-Encoding: base64|quoted-printable|7bit

5. Phone normalization:
   E.164: +[country_code][number]
   Thailand mobile: 0[689]XXXXXXXX → +66[689]XXXXXXXX
   Strip: [^\d+] then normalize leading 0/00

6. Quoted-Printable:
   =XX  → hex-encoded byte (e.g. =E2 = 0xE2)
   =\r\n → soft line break (continuation)
```

---

*[← Part 72: ReDoS & Performance](part-72-redos-performance.md) | [→ Part 74: Encoding & Obfuscation](part-74-encoding-obfuscation.md)*
