# Part 15: Password Validation — การตรวจสอบรหัสผ่าน

> **ระดับ:** พื้นฐาน-กลาง | **เวลาเรียน:** ~60 นาที | **ข้อกำหนด:** Part 01-14

---

## 15.1 Password Policy Standards

```
NIST SP 800-63B Guidelines (2017):
✅ Minimum 8 characters
✅ Maximum 64+ characters
✅ Allow all printable Unicode
✅ Check against known-breached passwords
❌ NO mandatory complexity rules
❌ NO periodic rotation
❌ NO hints

Common Enterprise Policy:
✅ Minimum 8 characters
✅ 1+ uppercase letter
✅ 1+ lowercase letter
✅ 1+ digit
✅ 1+ special character
✅ No username inside password
✅ No common dictionary words
```

---

## 15.2 Basic Password Validators

```python
import re
from typing import List
from dataclasses import dataclass

@dataclass
class PasswordRule:
    name: str
    pattern: re.Pattern
    message: str
    required: bool = True

class PasswordValidator:
    """
    Password validator with configurable rules
    """
    
    RULES = [
        PasswordRule(
            name='min_length',
            pattern=re.compile(r'^.{8,}$'),
            message='ต้องมีอย่างน้อย 8 ตัวอักษร',
        ),
        PasswordRule(
            name='max_length',
            pattern=re.compile(r'^.{1,128}$'),
            message='ต้องไม่เกิน 128 ตัวอักษร',
        ),
        PasswordRule(
            name='has_uppercase',
            pattern=re.compile(r'[A-Z]'),
            message='ต้องมีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว (A-Z)',
        ),
        PasswordRule(
            name='has_lowercase',
            pattern=re.compile(r'[a-z]'),
            message='ต้องมีตัวพิมพ์เล็กอย่างน้อย 1 ตัว (a-z)',
        ),
        PasswordRule(
            name='has_digit',
            pattern=re.compile(r'\d'),
            message='ต้องมีตัวเลขอย่างน้อย 1 ตัว (0-9)',
        ),
        PasswordRule(
            name='has_special',
            pattern=re.compile(r'[!@#$%^&*()\-_=+\[\]{}|;:,.<>?/`~\'"\\]'),
            message='ต้องมีอักขระพิเศษอย่างน้อย 1 ตัว',
        ),
        PasswordRule(
            name='no_spaces',
            pattern=re.compile(r'^\S+$'),
            message='ต้องไม่มีช่องว่าง',
        ),
        PasswordRule(
            name='no_repeating',
            pattern=re.compile(r'^(?!.*(.)(\1){2,})'),
            message='ต้องไม่มีตัวอักษรซ้ำกัน 3 ครั้งติดต่อกัน',
        ),
    ]
    
    WEAK_PASSWORDS = {
        'password', 'password1', '123456', '12345678', 'qwerty',
        'admin', 'letmein', 'welcome', 'monkey', 'dragon',
        'iloveyou', 'master', 'abc123', 'pass', 'test',
        'root', 'toor', 'login', 'guest', 'default',
    }
    
    @classmethod
    def validate(cls, password: str, 
                 username: str = None,
                 rules: List[str] = None) -> dict:
        
        errors = []
        passed = []
        
        if password.lower() in cls.WEAK_PASSWORDS:
            errors.append('รหัสผ่านนี้อ่อนแอเกินไป อยู่ในรายการรหัสผ่านที่ถูก crack บ่อย')
        
        if username and len(username) >= 3:
            if re.search(re.escape(username), password, re.IGNORECASE):
                errors.append(f'รหัสผ่านต้องไม่มีชื่อผู้ใช้ "{username}"')
        
        rule_names = rules or [r.name for r in cls.RULES]
        
        for rule in cls.RULES:
            if rule.name not in rule_names:
                continue
            if rule.pattern.search(password):
                passed.append(rule.name)
            else:
                errors.append(rule.message)
        
        strength = cls.calculate_strength(password)
        
        return {
            'valid': len(errors) == 0,
            'errors': errors,
            'passed': passed,
            'strength': strength,
            'strength_label': ['อ่อนมาก', 'อ่อน', 'ปานกลาง', 'แข็งแกร่ง', 'แข็งแกร่งมาก'][strength - 1],
        }
    
    @classmethod
    def calculate_strength(cls, password: str) -> int:
        score = 0
        if len(password) >= 8:  score += 1
        if len(password) >= 12: score += 1
        if re.search(r'[A-Z]', password) and re.search(r'[a-z]', password): score += 1
        if re.search(r'\d', password): score += 1
        if re.search(r'[!@#$%^&*()\-_=+\[\]{}|;:,.<>?/`~]', password): score += 1
        if len(password) >= 16: score += 1
        if re.search(r'[^\x00-\x7F]', password): score += 1
        return max(1, min(5, score - 1 if score > 1 else 1))


# ทดสอบ
test_passwords = [
    ("password", "alice"),
    ("Password1!", "bob"),
    ("P@ssw0rd", "charlie"),
    ("MyS3cur3P@$$w0rd!", "david"),
    ("alice123", "alice"),
    ("Tr0ub4dor&3", "user"),
]

print("Password Validation:")
print("=" * 70)
for pwd, usr in test_passwords:
    result = PasswordValidator.validate(pwd, username=usr)
    icon = '✅' if result['valid'] else '❌'
    strength_bar = '█' * result['strength'] + '░' * (5 - result['strength'])
    print(f"\n  {icon} '{pwd}'")
    print(f"     Strength: [{strength_bar}] {result['strength']}/5 - {result['strength_label']}")
    if result['errors']:
        for e in result['errors'][:2]:
            print(f"     ⚠️  {e}")
```

---

## 15.3 Password Strength Meter

```python
import re
import math
from typing import Dict

class PasswordStrengthMeter:
    """
    คำนวณ entropy และ crack time ของรหัสผ่าน
    """
    
    CHARSETS = {
        'lowercase': (re.compile(r'[a-z]'), 26),
        'uppercase': (re.compile(r'[A-Z]'), 26),
        'digits': (re.compile(r'\d'), 10),
        'symbols': (re.compile(r'[!@#$%^&*()\-_=+\[\]{}|;:,.<>?/`~]'), 32),
        'spaces': (re.compile(r'\s'), 1),
        'unicode': (re.compile(r'[^\x00-\x7F]'), 65536),
    }
    
    @classmethod
    def analyze(cls, password: str) -> Dict:
        length = len(password)
        pool_size = 0
        char_types = []
        
        for name, (pattern, size) in cls.CHARSETS.items():
            if pattern.search(password):
                pool_size += size
                char_types.append(name)
        
        entropy = length * math.log2(pool_size) if pool_size > 0 else 0
        guesses = 2 ** entropy
        crack_seconds = guesses / 10_000_000_000
        crack_time = cls._format_time(crack_seconds)
        patterns_found = cls._check_patterns(password)
        score = max(0, min(100, int(entropy * 1.5) - len(patterns_found) * 10))
        
        return {
            'length': length,
            'entropy': round(entropy, 1),
            'pool_size': pool_size,
            'char_types': char_types,
            'crack_time': crack_time,
            'patterns': patterns_found,
            'score': score,
            'grade': cls._grade(score),
        }
    
    @classmethod
    def _check_patterns(cls, password: str) -> list:
        found = []
        checks = [
            (r'(.)\1{2,}', 'ตัวอักษรซ้ำ (aaa, 111)'),
            (r'(012|123|234|345|456|567|678|789|890)', 'ตัวเลขเรียงต่อกัน'),
            (r'(?i)(qwerty|asdf|zxcv|qazwsx)', 'Keyboard walk pattern'),
            (r'(?i)(pass|word|secret|admin|user|login)', 'คำที่ใช้บ่อยในรหัสผ่าน'),
            (r'\d{4}$', 'ลงท้ายด้วยตัวเลข 4 หลัก'),
        ]
        for pattern, description in checks:
            if re.search(pattern, password, re.IGNORECASE):
                found.append(description)
        return found
    
    @staticmethod
    def _format_time(seconds: float) -> str:
        if seconds < 1: return "น้อยกว่า 1 วินาที"
        elif seconds < 60: return f"{seconds:.0f} วินาที"
        elif seconds < 3600: return f"{seconds/60:.0f} นาที"
        elif seconds < 86400: return f"{seconds/3600:.0f} ชั่วโมง"
        elif seconds < 2592000: return f"{seconds/86400:.0f} วัน"
        elif seconds < 31536000: return f"{seconds/2592000:.0f} เดือน"
        elif seconds < 3153600000: return f"{seconds/31536000:.0f} ปี"
        elif seconds < 3.15e13: return f"{seconds/3153600000:.0f} พันปี"
        else: return "หลายล้านปี (ปลอดภัยมาก)"
    
    @staticmethod
    def _grade(score: int) -> str:
        if score >= 80: return 'A'
        elif score >= 60: return 'B'
        elif score >= 40: return 'C'
        elif score >= 20: return 'D'
        else: return 'F'


# ทดสอบ
passwords = [
    "abc", "password", "Password1", "P@ssw0rd!",
    "Tr0ub4dor&3", "correct horse battery staple", "7Hj#mK9@pQ2&vL5nX",
]

print("\nPassword Strength Analysis:")
print("=" * 70)
for pwd in passwords:
    analysis = PasswordStrengthMeter.analyze(pwd)
    print(f"\n  Password: '{pwd}'")
    print(f"  Entropy:  {analysis['entropy']} bits | Pool: {analysis['pool_size']} chars")
    print(f"  Grade:    {analysis['grade']} ({analysis['score']}/100)")
    print(f"  Crack:    {analysis['crack_time']}")
    if analysis['patterns']:
        print(f"  Warnings: {', '.join(analysis['patterns'][:2])}")
```

---

## 15.4 Password Generator

```python
import re
import secrets
import string

class SecurePasswordGenerator:
    """Generate cryptographically secure passwords"""
    
    CHARS = {
        'lower': string.ascii_lowercase,
        'upper': string.ascii_uppercase,
        'digits': string.digits,
        'safe_symbols': '!@#$%^&*-_=+',
    }
    
    AMBIGUOUS = re.compile(r'[0O1lI|]')
    
    @classmethod
    def generate(cls, 
                 length: int = 16,
                 use_upper: bool = True,
                 use_lower: bool = True,
                 use_digits: bool = True,
                 use_symbols: bool = True,
                 avoid_ambiguous: bool = False,
                 min_of_each: int = 1) -> str:
        
        charset = ''
        required = []
        
        if use_lower:
            pool = cls.CHARS['lower']
            if avoid_ambiguous: pool = cls.AMBIGUOUS.sub('', pool)
            charset += pool
            required.append(pool)
        
        if use_upper:
            pool = cls.CHARS['upper']
            if avoid_ambiguous: pool = cls.AMBIGUOUS.sub('', pool)
            charset += pool
            required.append(pool)
        
        if use_digits:
            pool = cls.CHARS['digits']
            if avoid_ambiguous: pool = cls.AMBIGUOUS.sub('', pool)
            charset += pool
            required.append(pool)
        
        if use_symbols:
            charset += cls.CHARS['safe_symbols']
            required.append(cls.CHARS['safe_symbols'])
        
        if not charset:
            raise ValueError("At least one character type must be selected")
        
        password_chars = []
        for pool in required:
            for _ in range(min_of_each):
                password_chars.append(secrets.choice(pool))
        
        remaining = length - len(password_chars)
        for _ in range(max(0, remaining)):
            password_chars.append(secrets.choice(charset))
        
        secrets.SystemRandom().shuffle(password_chars)
        return ''.join(password_chars)
    
    @classmethod
    def generate_passphrase(cls, num_words: int = 4, separator: str = '-') -> str:
        """Generate memorable passphrase"""
        words = [
            'apple', 'brave', 'cloud', 'dance', 'eagle', 'flame',
            'grace', 'heart', 'ivory', 'jewel', 'kings', 'lunar',
            'maple', 'noble', 'ocean', 'pearl', 'queen', 'river',
            'storm', 'tiger', 'unity', 'vivid', 'water', 'xenon',
        ]
        selected = [secrets.choice(words).capitalize() for _ in range(num_words)]
        number = str(secrets.randbelow(100)).zfill(2)
        symbol = secrets.choice('!@#$%')
        return separator.join(selected + [number + symbol])
    
    @classmethod
    def generate_pin(cls, length: int = 6, no_repeating: bool = True) -> str:
        """Generate numeric PIN"""
        while True:
            pin = ''.join(str(secrets.randbelow(10)) for _ in range(length))
            if no_repeating:
                if re.search(r'(.)\1{2,}', pin):
                    continue
                digits = [int(d) for d in pin]
                diffs = [digits[i+1] - digits[i] for i in range(len(digits)-1)]
                if len(set(diffs)) == 1:
                    continue
            return pin


# ทดสอบ
print("\nPassword Generation:")
print("=" * 70)

print("\n  Standard passwords (length 16):")
for i in range(3):
    print(f"    {SecurePasswordGenerator.generate(length=16)}")

print("\n  Avoid-ambiguous passwords (length 12):")
for i in range(3):
    print(f"    {SecurePasswordGenerator.generate(length=12, avoid_ambiguous=True)}")

print("\n  Passphrases:")
for i in range(3):
    print(f"    {SecurePasswordGenerator.generate_passphrase(num_words=4)}")

print("\n  PINs (6 digits):")
for i in range(5):
    print(f"    {SecurePasswordGenerator.generate_pin(6)}")
```

---

## 15.5 Password Policy Builder

```python
import re
from dataclasses import dataclass, field
from typing import List, Optional

@dataclass
class PolicyConfig:
    min_length: int = 8
    max_length: int = 128
    require_uppercase: bool = True
    require_lowercase: bool = True
    require_digit: bool = True
    require_special: bool = True
    allow_spaces: bool = False
    max_repeating: int = 3
    special_chars: str = '!@#$%^&*()\\-_=+[]{}|;:,.<>?/`~'
    blacklist: List[str] = field(default_factory=list)


class PasswordPolicyBuilder:
    """Build custom password policies"""
    
    def __init__(self, config: PolicyConfig = None):
        self.config = config or PolicyConfig()
    
    def validate(self, password: str, username: str = None) -> dict:
        errors = []
        c = self.config
        
        if password.lower() in [b.lower() for b in c.blacklist]:
            errors.append('รหัสผ่านอยู่ใน blacklist')
        
        if username and re.search(re.escape(username), password, re.IGNORECASE):
            errors.append('ห้ามใช้ชื่อผู้ใช้ในรหัสผ่าน')
        
        if len(password) < c.min_length:
            errors.append(f'ต้องมีอย่างน้อย {c.min_length} ตัวอักษร')
        if len(password) > c.max_length:
            errors.append(f'ต้องไม่เกิน {c.max_length} ตัวอักษร')
        
        if c.require_uppercase and not re.search(r'[A-Z]', password):
            errors.append('ต้องมีตัวพิมพ์ใหญ่ (A-Z)')
        if c.require_lowercase and not re.search(r'[a-z]', password):
            errors.append('ต้องมีตัวพิมพ์เล็ก (a-z)')
        if c.require_digit and not re.search(r'\d', password):
            errors.append('ต้องมีตัวเลข (0-9)')
        if c.require_special:
            escaped = re.escape(c.special_chars)
            if not re.search(f'[{escaped}]', password):
                errors.append('ต้องมีอักขระพิเศษ')
        if not c.allow_spaces and re.search(r'\s', password):
            errors.append('ห้ามมีช่องว่าง')
        if c.max_repeating > 0:
            if re.search(f'(.){{{c.max_repeating + 1},}}', password):
                errors.append(f'ห้ามมีตัวอักษรซ้ำกัน {c.max_repeating + 1}+ ครั้ง')
        
        return {'valid': len(errors) == 0, 'errors': errors}


# ตัวอย่าง policies
nist_config = PolicyConfig(
    min_length=8, max_length=64,
    require_uppercase=False, require_lowercase=False,
    require_digit=False, require_special=False,
    blacklist=['password', '12345678', 'qwertyui'],
)
nist_policy = PasswordPolicyBuilder(nist_config)

strict_config = PolicyConfig(
    min_length=12, max_length=64,
    require_uppercase=True, require_lowercase=True,
    require_digit=True, require_special=True,
    max_repeating=2,
)
strict_policy = PasswordPolicyBuilder(strict_config)

test_cases = ["correcthorsebatterystaple", "P@ssw0rd!23", "weak", "P@ssword!"]

print("\nCustom Password Policies:")
for pwd in test_cases:
    nist_r = nist_policy.validate(pwd)
    strict_r = strict_policy.validate(pwd)
    print(f"\n  '{pwd}'")
    print(f"    NIST:   {'✅' if nist_r['valid'] else '❌'} {', '.join(nist_r['errors'][:1])}")
    print(f"    Strict: {'✅' if strict_r['valid'] else '❌'} {', '.join(strict_r['errors'][:1])}")
```

---

## 15.6 JavaScript Password Validation

```javascript
// ============================================
// JavaScript Password Validation
// ============================================

class PasswordValidator {
    static RULES = [
        { name: 'minLength',   pattern: /^.{8,}$/,         message: 'ต้องมีอย่างน้อย 8 ตัวอักษร' },
        { name: 'uppercase',   pattern: /[A-Z]/,            message: 'ต้องมีตัวพิมพ์ใหญ่' },
        { name: 'lowercase',   pattern: /[a-z]/,            message: 'ต้องมีตัวพิมพ์เล็ก' },
        { name: 'digit',       pattern: /\d/,               message: 'ต้องมีตัวเลข' },
        { name: 'special',     pattern: /[!@#$%^&*\-_=+]/,  message: 'ต้องมีอักขระพิเศษ' },
        { name: 'noSpaces',    pattern: /^\S+$/,            message: 'ห้ามมีช่องว่าง' },
        { name: 'noRepeating', pattern: /^(?!.*(.)(\1){2})/, message: 'ห้ามมีตัวซ้ำ 3+ ครั้ง' },
    ];
    
    static validate(password, options = {}) {
        const { username, rules = this.RULES } = options;
        const errors = [];
        const passed = [];
        
        if (username && password.toLowerCase().includes(username.toLowerCase())) {
            errors.push('ห้ามใช้ชื่อผู้ใช้ในรหัสผ่าน');
        }
        
        for (const rule of rules) {
            if (rule.pattern.test(password)) {
                passed.push(rule.name);
            } else {
                errors.push(rule.message);
            }
        }
        
        const strength = this.calculateStrength(password);
        
        return {
            valid: errors.length === 0,
            errors, passed, strength,
            strengthLabel: ['อ่อนมาก', 'อ่อน', 'ปานกลาง', 'แข็งแกร่ง', 'แข็งแกร่งมาก'][strength - 1],
        };
    }
    
    static calculateStrength(password) {
        let score = 0;
        if (password.length >= 8)  score++;
        if (password.length >= 12) score++;
        if (/[A-Z]/.test(password) && /[a-z]/.test(password)) score++;
        if (/\d/.test(password))   score++;
        if (/[!@#$%^&*\-_=+]/.test(password)) score++;
        return Math.min(5, Math.max(1, score));
    }
    
    // Real-time password strength indicator
    static createRealtimeValidator(inputEl, feedbackEl) {
        inputEl.addEventListener('input', (e) => {
            const result = this.validate(e.target.value);
            const colors = ['#ff4444', '#ff8800', '#ffcc00', '#88cc00', '#00aa44'];
            
            const bar = feedbackEl.querySelector('.strength-bar');
            if (bar) {
                bar.style.width = `${(result.strength / 5) * 100}%`;
                bar.style.backgroundColor = colors[result.strength - 1];
            }
            
            const errList = feedbackEl.querySelector('.errors');
            if (errList) {
                errList.innerHTML = result.errors.map(e => `<li>${e}</li>`).join('');
            }
        });
    }
}

// ทดสอบ
const passwords = [
    ['password', 'alice'],
    ['P@ssw0rd!', 'bob'],
    ['MySecure#Pass123', 'charlie'],
];

passwords.forEach(([pwd, usr]) => {
    const result = PasswordValidator.validate(pwd, { username: usr });
    const icon = result.valid ? '✅' : '❌';
    console.log(`${icon} '${pwd}' [strength: ${result.strength}/5]`);
    if (result.errors.length) console.log(`   Errors: ${result.errors.join(', ')}`);
});
```

---

## 15.7 PHP Password Validation

```php
<?php
// ============================================
// PHP Password Validation
// ============================================

class PasswordValidator {
    
    private static $rules = [
        'minLength'  => ['/^.{8,}$/',          'ต้องมีอย่างน้อย 8 ตัวอักษร'],
        'hasUpper'   => ['/[A-Z]/',             'ต้องมีตัวพิมพ์ใหญ่'],
        'hasLower'   => ['/[a-z]/',             'ต้องมีตัวพิมพ์เล็ก'],
        'hasDigit'   => ['/\d/',               'ต้องมีตัวเลข'],
        'hasSpecial' => ['/[!@#$%^&*\-_=+]/',  'ต้องมีอักขระพิเศษ'],
        'noSpaces'   => ['/^\S+$/',             'ห้ามมีช่องว่าง'],
    ];
    
    public static function validate(string $password, string $username = ''): array {
        $errors = [];
        $passed = [];
        
        if (!empty($username) && stripos($password, $username) !== false) {
            $errors[] = 'ห้ามใช้ชื่อผู้ใช้ในรหัสผ่าน';
        }
        
        foreach (self::$rules as $ruleName => [$pattern, $message]) {
            if (preg_match($pattern, $password)) {
                $passed[] = $ruleName;
            } else {
                $errors[] = $message;
            }
        }
        
        return [
            'valid'    => empty($errors),
            'errors'   => $errors,
            'passed'   => $passed,
            'strength' => self::calculateStrength($password),
        ];
    }
    
    private static function calculateStrength(string $password): int {
        $score = 0;
        if (strlen($password) >= 8)  $score++;
        if (strlen($password) >= 12) $score++;
        if (preg_match('/[A-Z]/', $password) && preg_match('/[a-z]/', $password)) $score++;
        if (preg_match('/\d/', $password)) $score++;
        if (preg_match('/[!@#$%^&*\-_=+]/', $password)) $score++;
        return min(5, max(1, $score));
    }
    
    public static function hash(string $password): string {
        return password_hash($password, PASSWORD_ARGON2ID, [
            'memory_cost' => 65536,
            'time_cost'   => 4,
            'threads'     => 2,
        ]);
    }
    
    public static function verify(string $password, string $hash): bool {
        return password_verify($password, $hash);
    }
}

// ทดสอบ
$tests = [
    ['password',      'alice'],
    ['P@ssw0rd!',     'bob'],
    ['MyS3cur3P@$$!', 'charlie'],
];

foreach ($tests as [$pwd, $usr]) {
    $result = PasswordValidator::validate($pwd, $usr);
    $icon = $result['valid'] ? '✅' : '❌';
    echo "$icon '$pwd' (strength: {$result['strength']}/5)\n";
    foreach (array_slice($result['errors'], 0, 2) as $err) {
        echo "   ⚠️  $err\n";
    }
}
?>
```

---

## 15.8 สรุป Part 15

```
Pattern สำคัญสำหรับ Password Validation:
────────────────────────────────────────────────────────────
Uppercase:       [A-Z]
Lowercase:       [a-z]
Digit:           \d  หรือ  [0-9]
Special chars:   [!@#$%^&*()\-_=+\[\]{}|;:,.<>?/`~]
Min 8 chars:     ^.{8,}$
No spaces:       ^\S+$
No repeating:    ^(?!.*(.)\1{2,})
Complex (all):   ^(?=.*[A-Z])(?=.*[a-z])(?=.*\d)(?=.*[!@#$%]).{8,}$
```

✅ **Password Validator** — ตรวจสอบตาม policy  
✅ **Strength Meter** — คำนวณ entropy และ crack time  
✅ **Secure Generator** — สร้างรหัสผ่านด้วย `secrets` module  
✅ **Policy Builder** — สร้าง policy แบบ custom ได้  
✅ **Passphrase** — รหัสผ่านแบบ memorable  

---

*[⬅ Part 14: Dates & Times](part-14-dates-times.md) | [➡ Part 16: IP Addresses](part-16-ip-addresses.md)*
