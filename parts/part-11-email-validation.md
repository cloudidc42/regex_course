# Part 11: Email Validation — ตัวอย่างจริง

> **ระดับ:** พื้นฐาน-กลาง | **เวลาเรียน:** ~90 นาที | **ข้อกำหนด:** Part 01-10

---

## 11.1 ทำไม Email Validation ถึงซับซ้อน?

Email address มีมาตรฐาน RFC 5321 และ RFC 5322 ที่ซับซ้อนมาก:

```
ตัวอย่าง email ที่ valid ตาม RFC แต่ดูแปลก:
- "user name"@example.com        (quoted local part)
- user+tag@sub.domain.co.th      (plus addressing)
- user@[192.168.1.1]             (IP address domain)
- admin@xn--nxasmq6b.com        (internationalized domain)
- very.long.name.with.dots@example.com
- user@localhost                  (no TLD - valid in some contexts)
```

### ระดับของ Validation

```
ระดับ 1: Syntax check  → Regex
ระดับ 2: Domain check  → DNS MX record lookup
ระดับ 3: Mailbox check → SMTP VRFY / send test email
```

ในบทนี้เราจะใช้ Regex สำหรับ **ระดับ 1** ซึ่งครอบคลุมการใช้งานส่วนใหญ่

---

## 11.2 Pattern พื้นฐาน → ซับซ้อน

### Step 1: Pattern ง่ายที่สุด

```python
import re

# Pattern ง่ายมาก (แต่ไม่แม่นยำ)
simple = r'.+@.+\..+'
emails = ['user@example.com', 'bad-email', 'a@b.c', '@missing.com']
for e in emails:
    result = '✅' if re.match(simple, e) else '❌'
    print(f"{result} {e}")

# ✅ user@example.com
# ❌ bad-email
# ✅ a@b.c
# ✅ @missing.com  ← False positive!
```

### Step 2: เพิ่ม character restrictions

```python
import re

# Pattern กลาง
medium = r'^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$'

test_emails = [
    'user@example.com',          # ✅ valid
    'user.name@example.com',     # ✅ valid
    'user+tag@example.com',      # ✅ valid
    'USER@EXAMPLE.COM',          # ✅ valid (case insensitive)
    'user@sub.domain.co.th',     # ✅ valid
    '@missing.com',              # ❌ no local part
    'user@',                     # ❌ no domain
    'user@.com',                 # ✅ incorrect (false positive)
    'user..name@example.com',    # ✅ incorrect (consecutive dots)
    'user@example..com',         # ✅ incorrect (consecutive dots)
]

for e in test_emails:
    result = '✅' if re.match(medium, e) else '❌'
    print(f"{result} {e}")
```

### Step 3: Pattern ที่ดีขึ้น

```python
import re

def validate_email_v3(email: str) -> tuple[bool, str]:
    email = email.strip()
    
    # Pattern ครอบคลุมมากขึ้น
    pattern = re.compile(r"""
        ^
        (?P<local>
            [a-zA-Z0-9]                    # must start with alphanumeric
            (?:[a-zA-Z0-9._%+\-]{0,62}    # followed by allowed chars
            [a-zA-Z0-9])?                  # must end with alphanumeric
        )
        @
        (?P<domain>
            (?:[a-zA-Z0-9]
               (?:[a-zA-Z0-9\-]{0,61}[a-zA-Z0-9])?
               \.)+                        # one or more subdomains
            [a-zA-Z]{2,10}               # TLD
        )
        $
    """, re.VERBOSE | re.IGNORECASE)
    
    m = pattern.match(email)
    if not m:
        return False, "Invalid format"
    
    local = m.group('local')
    domain = m.group('domain')
    
    # ตรวจสอบเพิ่มเติม
    if '..' in local:
        return False, "Consecutive dots in local part"
    if local.startswith('.') or local.endswith('.'):
        return False, "Local part cannot start/end with dot"
    if len(local) > 64:
        return False, "Local part too long (max 64)"
    if len(domain) > 255:
        return False, "Domain too long (max 255)"
    
    return True, "Valid email"

# ทดสอบ
test_cases = [
    'user@example.com',
    'user.name+tag@gmail.com',
    'admin@sub.domain.co.th',
    'user..double@example.com',
    '.startdot@example.com',
    'toolong' + 'a' * 60 + '@example.com',
    'user@example.c',          # TLD too short
    'user@256' + '.a' * 63,   # domain too long
]

for email in test_cases:
    valid, msg = validate_email_v3(email)
    icon = '✅' if valid else '❌'
    print(f"{icon} {email[:50]:50} | {msg}")
```

---

## 11.3 Email Validator Class เต็มรูปแบบ

```python
import re
from dataclasses import dataclass, field
from typing import List, Optional
import unicodedata

@dataclass
class EmailValidationResult:
    is_valid: bool
    email: str
    normalized: Optional[str] = None
    local_part: Optional[str] = None
    domain: Optional[str] = None
    tld: Optional[str] = None
    errors: List[str] = field(default_factory=list)
    warnings: List[str] = field(default_factory=list)
    
    def __str__(self) -> str:
        status = "✅ VALID" if self.is_valid else "❌ INVALID"
        parts = [f"{status}: {self.email}"]
        if self.errors:
            parts.append(f"  Errors: {', '.join(self.errors)}")
        if self.warnings:
            parts.append(f"  Warnings: {', '.join(self.warnings)}")
        return "\n".join(parts)


class EmailValidator:
    """
    Email validator ตาม RFC 5321/5322 (simplified)
    
    Features:
    - Syntax validation
    - Length validation (RFC limits)
    - Domain label validation  
    - Consecutive dot detection
    - Common disposable domain detection
    - Thai and international domain support
    """
    
    # Pattern สำหรับ local part ที่ valid
    LOCAL_PATTERN = re.compile(
        r'^[a-zA-Z0-9]'                    # เริ่มต้น
        r'(?:[a-zA-Z0-9._%+\-]*'           # middle
        r'[a-zA-Z0-9])?$'                  # จบด้วย (optional ถ้ามีแค่ 1 ตัว)
    )
    
    # Pattern สำหรับ domain label (ส่วนระหว่าง dots)
    DOMAIN_LABEL_PATTERN = re.compile(
        r'^[a-zA-Z0-9]'
        r'(?:[a-zA-Z0-9\-]{0,61}[a-zA-Z0-9])?$'
    )
    
    # TLD ที่พบบ่อย (ไม่ครบทั้งหมด)
    COMMON_TLDS = {
        'com', 'org', 'net', 'edu', 'gov', 'mil', 'int',
        'th', 'co', 'ac', 'go', 'or', 'in', 'net',  # Thai ccTLD
        'io', 'app', 'dev', 'web', 'site', 'online',
        'uk', 'us', 'de', 'fr', 'jp', 'cn', 'au', 'ca',
    }
    
    # โดเมนที่ใช้ทิ้ง (disposable)
    DISPOSABLE_DOMAINS = {
        'mailinator.com', 'guerrillamail.com', 'tempmail.com',
        'throwaway.email', '10minutemail.com', 'yopmail.com',
        'maildrop.cc', 'dispostable.com', 'sharklasers.com',
    }
    
    # Role-based local parts
    ROLE_ACCOUNTS = {
        'admin', 'administrator', 'webmaster', 'info', 'support',
        'sales', 'contact', 'noreply', 'no-reply', 'postmaster',
        'abuse', 'security', 'privacy', 'help', 'service',
    }
    
    MAX_LOCAL_LENGTH = 64
    MAX_DOMAIN_LENGTH = 255
    MAX_TOTAL_LENGTH = 320
    
    @classmethod
    def validate(cls, email: str, 
                 allow_disposable: bool = True,
                 warn_role_accounts: bool = True) -> EmailValidationResult:
        """
        Validate email address
        
        Args:
            email: Email string ที่ต้องการตรวจสอบ
            allow_disposable: อนุญาต disposable emails หรือไม่
            warn_role_accounts: แจ้งเตือน role-based accounts หรือไม่
        """
        original = email
        email = email.strip()
        
        result = EmailValidationResult(
            is_valid=False,
            email=original,
        )
        
        # ตรวจสอบว่าว่างเปล่าหรือไม่
        if not email:
            result.errors.append("Email cannot be empty")
            return result
        
        # ตรวจสอบความยาวรวม
        if len(email) > cls.MAX_TOTAL_LENGTH:
            result.errors.append(f"Email too long (max {cls.MAX_TOTAL_LENGTH} chars)")
            return result
        
        # แบ่ง local part และ domain
        at_count = email.count('@')
        if at_count == 0:
            result.errors.append("Missing @ symbol")
            return result
        if at_count > 1:
            result.errors.append("Multiple @ symbols found")
            return result
        
        local, domain = email.rsplit('@', 1)
        
        # Validate local part
        local_errors = cls._validate_local(local)
        result.errors.extend(local_errors)
        
        # Validate domain
        domain_errors, domain_warnings = cls._validate_domain(domain)
        result.errors.extend(domain_errors)
        result.warnings.extend(domain_warnings)
        
        if result.errors:
            return result
        
        # Normalize
        normalized = f"{local.lower()}@{domain.lower()}"
        
        # เช็ค disposable
        domain_lower = domain.lower()
        if not allow_disposable and domain_lower in cls.DISPOSABLE_DOMAINS:
            result.errors.append(f"Disposable email domain: {domain_lower}")
            return result
        
        if domain_lower in cls.DISPOSABLE_DOMAINS:
            result.warnings.append("Disposable email domain detected")
        
        # เช็ค role accounts
        if warn_role_accounts and local.lower() in cls.ROLE_ACCOUNTS:
            result.warnings.append(f"Role-based email address: {local.lower()}")
        
        # แยก TLD
        domain_parts = domain.split('.')
        tld = domain_parts[-1]
        
        result.is_valid = True
        result.normalized = normalized
        result.local_part = local
        result.domain = domain
        result.tld = tld
        
        return result
    
    @classmethod
    def _validate_local(cls, local: str) -> List[str]:
        errors = []
        
        if not local:
            errors.append("Local part (before @) cannot be empty")
            return errors
        
        if len(local) > cls.MAX_LOCAL_LENGTH:
            errors.append(f"Local part too long (max {cls.MAX_LOCAL_LENGTH})")
        
        if '..' in local:
            errors.append("Consecutive dots not allowed in local part")
        
        if local.startswith('.') or local.endswith('.'):
            errors.append("Local part cannot start or end with a dot")
        
        # Check characters
        invalid_chars = re.findall(r'[^a-zA-Z0-9._%+\-]', local)
        if invalid_chars:
            chars_str = ''.join(set(invalid_chars))
            errors.append(f"Invalid characters in local part: '{chars_str}'")
        
        return errors
    
    @classmethod
    def _validate_domain(cls, domain: str) -> tuple[List[str], List[str]]:
        errors = []
        warnings = []
        
        if not domain:
            errors.append("Domain cannot be empty")
            return errors, warnings
        
        if len(domain) > cls.MAX_DOMAIN_LENGTH:
            errors.append(f"Domain too long (max {cls.MAX_DOMAIN_LENGTH})")
        
        if domain.startswith('.') or domain.endswith('.'):
            errors.append("Domain cannot start or end with a dot")
            return errors, warnings
        
        if '..' in domain:
            errors.append("Consecutive dots not allowed in domain")
        
        labels = domain.split('.')
        
        if len(labels) < 2:
            errors.append("Domain must have at least one dot")
            return errors, warnings
        
        # ตรวจสอบแต่ละ label
        for i, label in enumerate(labels):
            if not label:
                errors.append(f"Empty domain label at position {i+1}")
                continue
            
            if len(label) > 63:
                errors.append(f"Domain label too long: '{label}' (max 63)")
            
            if label.startswith('-') or label.endswith('-'):
                errors.append(f"Domain label cannot start/end with hyphen: '{label}'")
            
            if not re.match(r'^[a-zA-Z0-9\-]+$', label):
                # อาจเป็น IDN/punycode
                if not re.match(r'^[a-zA-Z0-9\-\u0080-￿]+$', label):
                    errors.append(f"Invalid characters in domain label: '{label}'")
        
        # ตรวจสอบ TLD
        tld = labels[-1].lower()
        if len(tld) < 2:
            errors.append(f"TLD too short: '{tld}' (min 2 chars)")
        elif len(tld) > 10:
            warnings.append(f"Unusual TLD length: '{tld}'")
        
        if not re.match(r'^[a-zA-Z]+$', tld):
            warnings.append(f"TLD contains non-letter characters: '{tld}'")
        
        return errors, warnings
    
    @classmethod
    def validate_bulk(cls, emails: List[str]) -> dict:
        """Validate รายการ emails"""
        results = {
            'valid': [],
            'invalid': [],
            'warnings': [],
            'total': len(emails),
        }
        
        for email in emails:
            r = cls.validate(email)
            if r.is_valid:
                results['valid'].append(r)
                if r.warnings:
                    results['warnings'].append(r)
            else:
                results['invalid'].append(r)
        
        results['valid_count'] = len(results['valid'])
        results['invalid_count'] = len(results['invalid'])
        results['valid_percentage'] = (
            len(results['valid']) / len(emails) * 100 
            if emails else 0
        )
        
        return results


# ============================================
# ทดสอบ EmailValidator
# ============================================

test_emails = [
    # Valid emails
    "user@example.com",
    "user.name+tag@gmail.com",
    "admin@company.co.th",
    "info@university.ac.th",
    "user123@sub.domain.org",
    
    # Invalid emails
    "@missing-local.com",
    "missing-domain@",
    "no-at-sign.com",
    "double@@at.com",
    "user@.startdot.com",
    "user@enddot.com.",
    "user@consecutive..dots.com",
    "local..double@example.com",
    
    # Special cases
    "noreply@company.com",          # Role account
    "test@mailinator.com",          # Disposable
    "user@very-long-domain-name-exceeding-sixty-three-characters-wow.com",
]

print("=" * 70)
print("Email Validation Results")
print("=" * 70)

for email in test_emails:
    r = EmailValidator.validate(email)
    print(r)
    print()

# Bulk validation
print("\n" + "=" * 70)
print("Bulk Validation Summary")
print("=" * 70)
bulk_results = EmailValidator.validate_bulk(test_emails)
print(f"Total: {bulk_results['total']}")
print(f"Valid: {bulk_results['valid_count']} ({bulk_results['valid_percentage']:.1f}%)")
print(f"Invalid: {bulk_results['invalid_count']}")
print(f"With warnings: {len(bulk_results['warnings'])}")
```

---

## 11.4 Email Patterns สำหรับ Use Cases ต่างๆ

```python
import re

# ============================================
# 1. Simple validation (ใช้ทั่วไป)
# ============================================
SIMPLE_EMAIL = re.compile(
    r'^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$'
)

# ============================================
# 2. Strict validation
# ============================================
STRICT_EMAIL = re.compile(r"""
    ^
    (?!.*\.\.)                   # ไม่มี consecutive dots
    [a-zA-Z0-9]                  # เริ่มด้วย alphanumeric
    [a-zA-Z0-9._%+\-]{0,62}     # ตัวกลาง
    [a-zA-Z0-9]?                 # จบด้วย alphanumeric (optional ถ้าสั้น)
    @
    (?!-)                        # domain ไม่เริ่มด้วย -
    (?:[a-zA-Z0-9][a-zA-Z0-9\-]{0,61}[a-zA-Z0-9]\.)+  # subdomains
    [a-zA-Z]{2,10}               # TLD
    $
""", re.VERBOSE)

# ============================================
# 3. Extract emails from text
# ============================================
EXTRACT_EMAIL = re.compile(
    r'\b[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}\b'
)

# ============================================
# 4. Extract email + name (format: "Name <email>")
# ============================================
EMAIL_WITH_NAME = re.compile(
    r'(?P<name>[^<]+?)?\s*<(?P<email>[^>]+)>'
    r'|(?P<email2>[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,})'
)

# ============================================
# 5. Gmail-specific (strip dots and + tags)
# ============================================
def normalize_gmail(email: str) -> str:
    """
    Normalize Gmail address:
    - user.name@gmail.com → username@gmail.com
    - user+tag@gmail.com  → user@gmail.com
    """
    m = re.match(r'^([^@+]+)(?:\+[^@]*)?@(gmail\.com|googlemail\.com)$', 
                 email, re.IGNORECASE)
    if m:
        local = m.group(1).replace('.', '')
        return f"{local}@gmail.com"
    return email

# ============================================
# ทดสอบ
# ============================================

# Extract emails from text
text = """
Contact list:
- Alice Smith <alice.smith@company.com>
- Bob Jones <bjones@example.co.th>
- carol@startup.io (preferred)
- david.doe+newsletter@gmail.com
- admin AT company DOT com (not valid)
"""

# Extract all emails
emails_found = EXTRACT_EMAIL.findall(text)
print("Found emails:", emails_found)

# Extract with names
print("\nEmail addresses with names:")
for m in EMAIL_WITH_NAME.finditer(text):
    if m.group('email'):
        name = (m.group('name') or '').strip()
        email = m.group('email')
        print(f"  Name: {name!r:25} Email: {email}")
    elif m.group('email2'):
        print(f"  Name: {'(no name)':25} Email: {m.group('email2')}")

# Gmail normalization
gmail_tests = [
    'u.s.e.r@gmail.com',
    'user+spam@gmail.com',
    'user.name+filter@googlemail.com',
    'user@yahoo.com',  # not gmail
]
print("\nGmail normalization:")
for e in gmail_tests:
    print(f"  {e:40} → {normalize_gmail(e)}")
```

---

## 11.5 Email Validation ใน JavaScript

```javascript
// ============================================
// JavaScript Email Validation
// ============================================

// Simple pattern
const SIMPLE_EMAIL = /^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$/;

// Strict pattern (RFC 5321 subset)
const STRICT_EMAIL = /^(?!.*\.\.)(?:[a-zA-Z0-9!#$%&'*+/=?^_`{|}~\-]+(?:\.[a-zA-Z0-9!#$%&'*+/=?^_`{|}~\-]+)*)@(?:[a-zA-Z0-9](?:[a-zA-Z0-9\-]{0,61}[a-zA-Z0-9])?\.)+[a-zA-Z]{2,}$/;

function validateEmail(email, strict = false) {
    const trimmed = email.trim().toLowerCase();
    const pattern = strict ? STRICT_EMAIL : SIMPLE_EMAIL;
    
    if (!trimmed) return { valid: false, error: 'Email cannot be empty' };
    if (trimmed.length > 320) return { valid: false, error: 'Email too long' };
    
    const parts = trimmed.split('@');
    if (parts.length !== 2) return { valid: false, error: 'Invalid @ usage' };
    
    const [local, domain] = parts;
    
    if (local.length > 64) return { valid: false, error: 'Local part too long' };
    if (domain.length > 255) return { valid: false, error: 'Domain too long' };
    
    if (!pattern.test(trimmed)) return { valid: false, error: 'Invalid format' };
    
    return { valid: true, normalized: trimmed, local, domain };
}

// Extract emails from text
function extractEmails(text) {
    const pattern = /\b[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}\b/g;
    return [...text.matchAll(pattern)].map(m => m[0]);
}

// Form validation
class EmailFormValidator {
    static validate(email, options = {}) {
        const {
            required = true,
            allowDisposable = true,
            domains = null,     // whitelist of domains
            strict = false,
        } = options;
        
        const errors = [];
        
        if (!email && required) {
            errors.push('Email is required');
            return { valid: false, errors };
        }
        
        if (!email && !required) {
            return { valid: true, errors };
        }
        
        const result = validateEmail(email, strict);
        if (!result.valid) {
            errors.push(result.error);
            return { valid: false, errors };
        }
        
        // Domain whitelist check
        if (domains && !domains.includes(result.domain)) {
            errors.push(`Email must be from: ${domains.join(', ')}`);
        }
        
        return {
            valid: errors.length === 0,
            errors,
            normalized: result.normalized,
        };
    }
}

// ทดสอบ
const testCases = [
    'user@example.com',
    'user.name+tag@gmail.com',
    'admin@company.co.th',
    '',
    'invalid',
    '@missing.com',
    'user@',
    'user@domain.c',
];

console.log('Email Validation Results:');
testCases.forEach(email => {
    const result = EmailFormValidator.validate(email);
    const icon = result.valid ? '✅' : '❌';
    console.log(`${icon} "${email}": ${result.valid ? result.normalized : result.errors[0]}`);
});
```

---

## 11.6 Email Validation ใน PHP

```php
<?php
// ============================================
// PHP Email Validation
// ============================================

function validateEmail(string $email): array {
    $email = trim($email);
    $result = [
        'valid' => false,
        'email' => $email,
        'normalized' => null,
        'errors' => [],
    ];
    
    if (empty($email)) {
        $result['errors'][] = 'Email cannot be empty';
        return $result;
    }
    
    // PHP built-in filter
    if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
        $result['errors'][] = 'Invalid email format';
        return $result;
    }
    
    // เพิ่มเติม: ตรวจสอบ length
    [$local, $domain] = explode('@', $email, 2);
    
    if (strlen($local) > 64) {
        $result['errors'][] = 'Local part too long (max 64)';
        return $result;
    }
    
    if (strlen($domain) > 255) {
        $result['errors'][] = 'Domain too long (max 255)';
        return $result;
    }
    
    // ตรวจสอบ consecutive dots
    if (str_contains($local, '..') || str_contains($domain, '..')) {
        $result['errors'][] = 'Consecutive dots not allowed';
        return $result;
    }
    
    $result['valid'] = true;
    $result['normalized'] = strtolower($email);
    
    return $result;
}

// ดึง emails จาก text
function extractEmails(string $text): array {
    $pattern = '/\b[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}\b/';
    preg_match_all($pattern, $text, $matches);
    return $matches[0];
}

// Regex-based (ไม่ใช้ filter_var)
function validateEmailRegex(string $email): bool {
    $pattern = '/^[a-zA-Z0-9](?:[a-zA-Z0-9._%+\-]{0,62}[a-zA-Z0-9])?@'
             . '(?:[a-zA-Z0-9](?:[a-zA-Z0-9\-]{0,61}[a-zA-Z0-9])?\.)'
             . '+[a-zA-Z]{2,10}$/';
    
    return (bool) preg_match($pattern, trim($email));
}

// ทดสอบ
$testEmails = [
    'user@example.com',
    'user.name@company.co.th',
    'invalid@',
    '@missing.com',
    'user@domain..com',
];

foreach ($testEmails as $email) {
    $result = validateEmail($email);
    $icon = $result['valid'] ? '✅' : '❌';
    echo "{$icon} {$email}";
    if (!$result['valid']) {
        echo " → " . implode(', ', $result['errors']);
    }
    echo "\n";
}

// Extract
$text = "Contact alice@example.com or bob.jones@company.org for details";
$emails = extractEmails($text);
echo "\nFound emails: " . implode(', ', $emails) . "\n";
?>
```

---

## 11.7 Real-World: Email Normalizer

```python
import re
from typing import Optional

class EmailNormalizer:
    """
    Normalize email addresses สำหรับ deduplication
    """
    
    # Known providers ที่ ignore dots ใน local part
    DOT_INSENSITIVE_PROVIDERS = {
        'gmail.com', 'googlemail.com',
    }
    
    # Providers ที่มี alias domains
    DOMAIN_ALIASES = {
        'googlemail.com': 'gmail.com',
        'google.com': 'gmail.com',
    }
    
    @classmethod
    def normalize(cls, email: str) -> Optional[str]:
        email = email.strip().lower()
        
        if not re.match(r'^[^@]+@[^@]+\.[^@]+$', email):
            return None
        
        local, domain = email.rsplit('@', 1)
        
        # Resolve domain aliases
        domain = cls.DOMAIN_ALIASES.get(domain, domain)
        
        # Strip + tags (subaddressing)
        local = re.sub(r'\+.*$', '', local)
        
        # Remove dots for dot-insensitive providers
        if domain in cls.DOT_INSENSITIVE_PROVIDERS:
            local = local.replace('.', '')
        
        return f"{local}@{domain}"
    
    @classmethod
    def are_same(cls, email1: str, email2: str) -> bool:
        """ตรวจสอบว่าเป็น email เดียวกันหรือไม่"""
        n1 = cls.normalize(email1)
        n2 = cls.normalize(email2)
        
        if n1 is None or n2 is None:
            return False
        
        return n1 == n2
    
    @classmethod
    def deduplicate(cls, emails: list) -> list:
        """ลบ duplicate emails ออก"""
        seen = set()
        result = []
        
        for email in emails:
            normalized = cls.normalize(email)
            if normalized and normalized not in seen:
                seen.add(normalized)
                result.append(email)
        
        return result


# ทดสอบ
normalizer_tests = [
    ('user.name@gmail.com',  'username@gmail.com'),   # dots removed
    ('user+spam@gmail.com',  'user@gmail.com'),         # + tag removed
    ('user@googlemail.com',  'user@gmail.com'),         # alias resolved
    ('USER@GMAIL.COM',       'user@gmail.com'),         # lowercased
]

print("Email Normalization:")
for original, expected in normalizer_tests:
    normalized = EmailNormalizer.normalize(original)
    match = '✅' if normalized == expected else '❌'
    print(f"  {match} {original:35} → {normalized}")
    if normalized != expected:
        print(f"     Expected: {expected}")

# Deduplication
email_list = [
    'user@gmail.com',
    'u.s.e.r@gmail.com',
    'user+tag@gmail.com',
    'USER@GMAIL.COM',
    'alice@example.com',
    'bob@company.com',
    'alice@example.com',  # true duplicate
]

unique = EmailNormalizer.deduplicate(email_list)
print(f"\nOriginal count: {len(email_list)}")
print(f"Unique count: {len(unique)}")
print(f"Unique emails: {unique}")

# Check same
print("\nSame email check:")
pairs = [
    ('user@gmail.com', 'u.s.e.r@gmail.com'),
    ('user@gmail.com', 'user+spam@gmail.com'),
    ('user@gmail.com', 'user@yahoo.com'),
]
for e1, e2 in pairs:
    same = EmailNormalizer.are_same(e1, e2)
    print(f"  {e1!r} == {e2!r}: {same}")
```

---

## 11.8 Performance: Compiled vs Non-compiled

```python
import re
import time

# ============================================
# Compiled vs. Non-compiled Performance
# ============================================

EMAIL_PATTERN = re.compile(
    r'^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$'
)

test_emails = [f"user{i}@example{i % 100}.com" for i in range(10000)]

# Non-compiled
start = time.perf_counter()
for email in test_emails:
    re.match(r'^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$', email)
non_compiled_time = time.perf_counter() - start

# Compiled
start = time.perf_counter()
for email in test_emails:
    EMAIL_PATTERN.match(email)
compiled_time = time.perf_counter() - start

print(f"Non-compiled: {non_compiled_time:.4f}s")
print(f"Compiled:     {compiled_time:.4f}s")
print(f"Speedup:      {non_compiled_time/compiled_time:.1f}x")

# Note: Python caches compiled patterns internally,
# but explicit compile() is still faster and clearer
```

---

## 11.9 สรุป Part 11

✅ **Simple pattern** — ใช้ได้สำหรับงานทั่วไป  
✅ **Strict pattern** — ตรวจสอบตาม RFC มากขึ้น  
✅ **Custom validator class** — เพิ่ม business logic  
✅ **Email extraction** — `\b...\b` สำหรับดึงจาก text  
✅ **Normalization** — สำหรับ deduplication  
✅ **Compiled patterns** — performance ดีกว่า  

### Pattern ที่แนะนำสำหรับงานทั่วไป

```python
import re

# สำหรับ form validation
EMAIL_SIMPLE = re.compile(
    r'^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$'
)

# สำหรับ extract จาก text
EMAIL_EXTRACT = re.compile(
    r'\b[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}\b'
)

# ใช้งาน
def is_valid_email(email: str) -> bool:
    return bool(EMAIL_SIMPLE.match(email.strip()))

def extract_emails(text: str) -> list:
    return EMAIL_EXTRACT.findall(text)
```

---

*[⬅ Part 10: Flags](part-10-flags.md) | [➡ Part 12: URL Matching](part-12-url-matching.md)*
