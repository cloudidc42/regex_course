# Part 16: IP Addresses — การจับคู่ IP Address

> **ระดับ:** พื้นฐาน-กลาง | **เวลาเรียน:** ~70 นาที | **ข้อกำหนด:** Part 01-15

---

## 16.1 IPv4 Address Structure

```
IPv4 = 4 octets × 8 bits = 32 bits total

ตัวอย่าง: 192.168.1.100

  192     .   168    .    1     .   100
[0-255]  [0-255]  [0-255]  [0-255]

Range: 0.0.0.0 → 255.255.255.255
```

---

## 16.2 IPv4 Patterns

```python
import re
from typing import Optional, List

class IPv4Validator:
    """
    Validate and parse IPv4 addresses
    """
    
    # Basic pattern (0-255 per octet)
    BASIC = re.compile(
        r'\b'
        r'(?:(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\.){3}'
        r'(?:25[0-5]|2[0-4]\d|[01]?\d\d?)'
        r'\b'
    )
    
    # Strict: full match only
    STRICT = re.compile(
        r'^'
        r'(?:(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\.){3}'
        r'(?:25[0-5]|2[0-4]\d|[01]?\d\d?)'
        r'$'
    )
    
    # With optional CIDR: 192.168.1.0/24
    WITH_CIDR = re.compile(
        r'^'
        r'(?P<ip>(?:(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\.){3}(?:25[0-5]|2[0-4]\d|[01]?\d\d?))'
        r'(?:/(?P<prefix>[0-9]|[12]\d|3[0-2]))?'
        r'$'
    )
    
    # Private address ranges (RFC 1918)
    PRIVATE_RANGES = [
        re.compile(r'^10\.\d{1,3}\.\d{1,3}\.\d{1,3}$'),
        re.compile(r'^172\.(1[6-9]|2\d|3[01])\.\d{1,3}\.\d{1,3}$'),
        re.compile(r'^192\.168\.\d{1,3}\.\d{1,3}$'),
    ]
    
    LOOPBACK   = re.compile(r'^127\.\d{1,3}\.\d{1,3}\.\d{1,3}$')
    LINK_LOCAL = re.compile(r'^169\.254\.\d{1,3}\.\d{1,3}$')
    MULTICAST  = re.compile(r'^2(?:2[4-9]|3\d)\.\d{1,3}\.\d{1,3}\.\d{1,3}$')
    
    @classmethod
    def validate(cls, ip: str) -> dict:
        if not cls.STRICT.match(ip):
            return {'valid': False, 'error': 'Invalid IPv4 format'}
        
        octets = [int(o) for o in ip.split('.')]
        
        addr_type = 'public'
        if cls.LOOPBACK.match(ip):
            addr_type = 'loopback'
        elif any(p.match(ip) for p in cls.PRIVATE_RANGES):
            addr_type = 'private'
        elif cls.LINK_LOCAL.match(ip):
            addr_type = 'link-local'
        elif cls.MULTICAST.match(ip):
            addr_type = 'multicast'
        elif octets[0] == 0:
            addr_type = 'this-network'
        elif ip == '255.255.255.255':
            addr_type = 'broadcast'
        
        return {
            'valid': True,
            'ip': ip,
            'octets': octets,
            'type': addr_type,
            'binary': '.'.join(f'{o:08b}' for o in octets),
            'integer': sum(o << (24 - 8*i) for i, o in enumerate(octets)),
        }
    
    @classmethod
    def extract_all(cls, text: str) -> List[str]:
        """Extract all IPv4 addresses from text"""
        return cls.BASIC.findall(text)
    
    @classmethod
    def is_in_subnet(cls, ip: str, network: str, prefix: int) -> bool:
        """Check if IP is in subnet"""
        def ip_to_int(addr):
            parts = addr.split('.')
            return sum(int(p) << (24 - 8*i) for i, p in enumerate(parts))
        
        ip_int = ip_to_int(ip)
        net_int = ip_to_int(network)
        mask = (0xFFFFFFFF << (32 - prefix)) & 0xFFFFFFFF
        return (ip_int & mask) == (net_int & mask)


# ทดสอบ IPv4
ipv4_tests = [
    "192.168.1.1", "10.0.0.1", "172.16.0.1", "127.0.0.1",
    "8.8.8.8", "255.255.255.255", "256.0.0.1",
    "192.168.1.256", "1.2.3", "0.0.0.0", "169.254.1.1", "224.0.0.1",
]

print("IPv4 Validation:")
print("=" * 65)
for ip in ipv4_tests:
    result = IPv4Validator.validate(ip)
    if result['valid']:
        print(f"  ✅ {ip:20} → type: {result['type']:12} int: {result['integer']}")
    else:
        print(f"  ❌ {ip:20} → {result['error']}")

# Extract from text
log_text = """
Failed login from 192.168.1.100
Successful auth: 10.0.0.50 → 8.8.8.8
Blocked: 203.0.113.5 (blacklisted)
Server: 172.20.0.1 connected to 172.20.0.254
"""
print("\nExtracted IPs:")
for ip in IPv4Validator.extract_all(log_text):
    print(f"  {ip}")

# Subnet check
print("\nSubnet Check (192.168.1.0/24):")
for ip in ["192.168.1.50", "192.168.1.200", "192.168.2.1", "10.0.0.1"]:
    in_subnet = IPv4Validator.is_in_subnet(ip, "192.168.1.0", 24)
    print(f"  {'✅' if in_subnet else '❌'} {ip}")
```

---

## 16.3 IPv6 Address Patterns

```python
import re
from typing import Optional

class IPv6Validator:
    """Validate IPv6 addresses"""
    
    # Full IPv6: 8 groups of 4 hex digits
    FULL = re.compile(
        r'^(?:[0-9a-fA-F]{1,4}:){7}[0-9a-fA-F]{1,4}$'
    )
    
    # Compressed with ::
    COMPRESSED = re.compile(
        r'^'
        r'(?:[0-9a-fA-F]{1,4}:){1,7}:'
        r'|:(?::[0-9a-fA-F]{1,4}){1,7}'
        r'|(?:[0-9a-fA-F]{1,4}:){1,6}:[0-9a-fA-F]{1,4}'
        r'|(?:[0-9a-fA-F]{1,4}:){1,5}(?::[0-9a-fA-F]{1,4}){1,2}'
        r'|(?:[0-9a-fA-F]{1,4}:){1,4}(?::[0-9a-fA-F]{1,4}){1,3}'
        r'|(?:[0-9a-fA-F]{1,4}:){1,3}(?::[0-9a-fA-F]{1,4}){1,4}'
        r'|(?:[0-9a-fA-F]{1,4}:){1,2}(?::[0-9a-fA-F]{1,4}){1,5}'
        r'|[0-9a-fA-F]{1,4}:(?::[0-9a-fA-F]{1,4}){1,6}'
        r'|::'
        r'$'
    )
    
    # IPv4-mapped: ::ffff:192.168.1.1
    IPV4_MAPPED = re.compile(
        r'^::(?:ffff:)?'
        r'(?:(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\.){3}'
        r'(?:25[0-5]|2[0-4]\d|[01]?\d\d?)$',
        re.IGNORECASE
    )
    
    @classmethod
    def validate(cls, ip: str) -> dict:
        s = ip.strip().lower().strip('[]')
        s = re.sub(r'%\w+$', '', s)  # remove zone ID
        
        if cls.FULL.match(s):
            fmt = 'full'
        elif cls.IPV4_MAPPED.match(s):
            fmt = 'ipv4-mapped'
        elif cls.COMPRESSED.match(s):
            fmt = 'compressed'
        else:
            return {'valid': False, 'error': 'Invalid IPv6 format'}
        
        addr_type = 'global-unicast'
        if s in ('::1', '0:0:0:0:0:0:0:1'):
            addr_type = 'loopback'
        elif s.startswith('fe80'):
            addr_type = 'link-local'
        elif s.startswith('fc') or s.startswith('fd'):
            addr_type = 'unique-local'
        elif s.startswith('ff'):
            addr_type = 'multicast'
        elif s in ('::', '0:0:0:0:0:0:0:0'):
            addr_type = 'unspecified'
        elif fmt == 'ipv4-mapped':
            addr_type = 'ipv4-mapped'
        
        return {'valid': True, 'ip': s, 'format': fmt, 'type': addr_type}
    
    @classmethod
    def expand(cls, ip: str) -> Optional[str]:
        """Expand compressed IPv6 to full form"""
        s = ip.strip().lower().strip('[]')
        s = re.sub(r'%\w+$', '', s)
        
        if '::' in s:
            left, _, right = s.partition('::')
            left_parts = left.split(':') if left else []
            right_parts = right.split(':') if right else []
            missing = 8 - len(left_parts) - len(right_parts)
            parts = left_parts + ['0'] * missing + right_parts
        else:
            parts = s.split(':')
        
        if len(parts) != 8:
            return None
        
        try:
            return ':'.join(f'{int(p, 16):04x}' for p in parts)
        except ValueError:
            return None


# ทดสอบ IPv6
ipv6_tests = [
    "2001:0db8:85a3:0000:0000:8a2e:0370:7334",
    "2001:db8:85a3::8a2e:370:7334",
    "::1",
    "::",
    "::ffff:192.168.1.1",
    "fe80::1",
    "fc00::1",
    "ff02::1",
    "[2001:db8::1]",
    "invalid::address::here",
]

print("\nIPv6 Validation:")
print("=" * 65)
for ip in ipv6_tests:
    result = IPv6Validator.validate(ip)
    if result['valid']:
        print(f"  ✅ {ip:40} → {result['type']} ({result['format']})")
    else:
        print(f"  ❌ {ip:40} → {result['error']}")
```

---

## 16.4 CIDR Notation

```python
import re
from typing import Optional

class CIDRParser:
    """Parse and validate CIDR notation"""
    
    IPV4_CIDR = re.compile(
        r'^'
        r'(?P<network>(?:(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\.){3}(?:25[0-5]|2[0-4]\d|[01]?\d\d?))'
        r'/'
        r'(?P<prefix>[0-9]|[12]\d|3[0-2])'
        r'$'
    )
    
    @classmethod
    def parse_ipv4(cls, cidr: str) -> Optional[dict]:
        m = cls.IPV4_CIDR.match(cidr.strip())
        if not m:
            return None
        
        network_str = m.group('network')
        prefix = int(m.group('prefix'))
        octets = [int(o) for o in network_str.split('.')]
        network_int = sum(o << (24 - 8*i) for i, o in enumerate(octets))
        mask_int = (0xFFFFFFFF << (32 - prefix)) & 0xFFFFFFFF
        actual_network = network_int & mask_int
        broadcast = actual_network | (~mask_int & 0xFFFFFFFF)
        
        def int_to_ip(n):
            return '.'.join(str((n >> (24 - 8*i)) & 0xFF) for i in range(4))
        
        num_hosts = 2 ** (32 - prefix) - 2 if prefix < 31 else (2 if prefix == 31 else 1)
        
        return {
            'cidr': cidr,
            'network': int_to_ip(actual_network),
            'broadcast': int_to_ip(broadcast),
            'prefix': prefix,
            'mask': int_to_ip(mask_int),
            'first_host': int_to_ip(actual_network + 1) if prefix < 31 else int_to_ip(actual_network),
            'last_host': int_to_ip(broadcast - 1) if prefix < 31 else int_to_ip(broadcast),
            'num_hosts': num_hosts,
            'num_addresses': 2 ** (32 - prefix),
        }


# ทดสอบ CIDR
cidr_tests = [
    "192.168.1.0/24", "10.0.0.0/8", "172.16.0.0/12",
    "192.168.1.100/30", "8.8.8.0/29", "0.0.0.0/0", "192.168.1.1/32",
]

print("\nCIDR Parsing:")
print("=" * 75)
for cidr in cidr_tests:
    result = CIDRParser.parse_ipv4(cidr)
    if result:
        print(f"\n  {cidr}")
        print(f"    Network:   {result['network']} / Mask: {result['mask']}")
        print(f"    Broadcast: {result['broadcast']}")
        print(f"    Hosts:     {result['first_host']} - {result['last_host']} ({result['num_hosts']:,})")
```

---

## 16.5 IP Address Classifier and Security

```python
import re
from typing import List

class IPSecurityClassifier:
    """Classify IPs for security purposes"""
    
    # Documentation/reserved ranges (RFC 5737)
    DOCUMENTATION = [
        re.compile(r'^192\.0\.2\.\d{1,3}$'),
        re.compile(r'^198\.51\.100\.\d{1,3}$'),
        re.compile(r'^203\.0\.113\.\d{1,3}$'),
    ]
    
    PRIVATE = [
        re.compile(r'^10\.\d{1,3}\.\d{1,3}\.\d{1,3}$'),
        re.compile(r'^172\.(1[6-9]|2\d|3[01])\.\d{1,3}\.\d{1,3}$'),
        re.compile(r'^192\.168\.\d{1,3}\.\d{1,3}$'),
    ]
    
    LOOPBACK   = re.compile(r'^127\.\d{1,3}\.\d{1,3}\.\d{1,3}$')
    MULTICAST  = re.compile(r'^2(?:2[4-9]|3\d)\.\d{1,3}\.\d{1,3}\.\d{1,3}$')
    LINK_LOCAL = re.compile(r'^169\.254\.\d{1,3}\.\d{1,3}$')
    
    KNOWN_MALICIOUS = {'1.2.3.4', '5.6.7.8'}  # example only
    
    @classmethod
    def classify(cls, ip: str) -> dict:
        classifications = []
        risk_level = 'low'
        
        if ip in cls.KNOWN_MALICIOUS:
            classifications.append('known-malicious')
            risk_level = 'critical'
        
        if cls.LOOPBACK.match(ip):
            classifications.append('loopback')
        if any(p.match(ip) for p in cls.PRIVATE):
            classifications.append('private-rfc1918')
        if cls.LINK_LOCAL.match(ip):
            classifications.append('link-local')
        if cls.MULTICAST.match(ip):
            classifications.append('multicast')
        if any(p.match(ip) for p in cls.DOCUMENTATION):
            classifications.append('documentation-reserved')
        
        if not classifications:
            classifications.append('public')
            if risk_level == 'low':
                risk_level = 'medium'
        
        return {
            'ip': ip,
            'classifications': classifications,
            'risk_level': risk_level,
            'is_routable': 'private-rfc1918' not in classifications and 
                          'loopback' not in classifications and
                          'link-local' not in classifications,
        }
    
    @classmethod
    def filter_trusted(cls, ip_list: List[str], trust_private: bool = True) -> dict:
        trusted, untrusted = [], []
        for ip in ip_list:
            info = cls.classify(ip)
            is_trusted = (
                'loopback' in info['classifications'] or
                (trust_private and 'private-rfc1918' in info['classifications'])
            )
            (trusted if is_trusted else untrusted).append(ip)
        return {'trusted': trusted, 'untrusted': untrusted}


# ทดสอบ
ips_to_classify = [
    "192.168.1.100", "10.0.0.1", "127.0.0.1", "8.8.8.8",
    "169.254.0.1", "203.0.113.1", "224.0.0.1", "1.2.3.4",
]

print("\nIP Security Classification:")
print("=" * 65)
for ip in ips_to_classify:
    info = IPSecurityClassifier.classify(ip)
    routable = "routable" if info['is_routable'] else "non-routable"
    risk = info['risk_level'].upper()
    classes = ', '.join(info['classifications'])
    print(f"  {ip:20} [{risk:8}] {routable}: {classes}")
```

---

## 16.6 Rate Limiting Pattern Matching

```python
import re
from collections import defaultdict
from datetime import datetime, timedelta
from typing import Dict, List

class IPRateLimiter:
    """ตรวจจับ IP ที่ส่ง request มากผิดปกติ"""
    
    # Apache/Nginx combined log format
    ACCESS_LOG = re.compile(
        r'^(?P<ip>\d{1,3}(?:\.\d{1,3}){3})'
        r'\s+\S+\s+\S+'
        r'\s+\[(?P<time>[^\]]+)\]'
        r'\s+"(?P<method>\w+)\s+(?P<path>\S+)\s+HTTP/[\d.]+"'
        r'\s+(?P<status>\d{3})'
        r'\s+(?P<size>\d+|-)'
    )
    
    def __init__(self, window_minutes: int = 5, max_requests: int = 100):
        self.window = timedelta(minutes=window_minutes)
        self.max_requests = max_requests
        self.requests: Dict[str, List[datetime]] = defaultdict(list)
    
    def record(self, ip: str, timestamp: datetime = None) -> dict:
        if timestamp is None:
            timestamp = datetime.now()
        cutoff = timestamp - self.window
        self.requests[ip] = [t for t in self.requests[ip] if t > cutoff]
        self.requests[ip].append(timestamp)
        count = len(self.requests[ip])
        return {'ip': ip, 'count': count, 'blocked': count > self.max_requests}
    
    def analyze_log_file(self, log_content: str) -> dict:
        """Analyze IP frequency from log"""
        ip_counts = defaultdict(int)
        suspicious_paths = defaultdict(int)
        login_re = re.compile(r'POST\s+/(?:login|auth|signin|admin)', re.IGNORECASE)
        
        for line in log_content.strip().split('\n'):
            m = self.ACCESS_LOG.match(line)
            if m:
                ip = m.group('ip')
                ip_counts[ip] += 1
                if login_re.search(line):
                    suspicious_paths[ip] += 1
        
        return {
            'total_requests': sum(ip_counts.values()),
            'unique_ips': len(ip_counts),
            'top_ips': sorted(ip_counts.items(), key=lambda x: x[1], reverse=True)[:10],
            'login_attempts': dict(sorted(suspicious_paths.items(), key=lambda x: x[1], reverse=True)),
        }


# Sample log analysis
sample_log = """192.168.1.100 - - [02/Oct/2026:14:30:00 +0700] "GET /index.html HTTP/1.1" 200 1234
10.0.0.5 - - [02/Oct/2026:14:30:01 +0700] "POST /login HTTP/1.1" 401 56
10.0.0.5 - - [02/Oct/2026:14:30:02 +0700] "POST /login HTTP/1.1" 401 56
10.0.0.5 - - [02/Oct/2026:14:30:03 +0700] "POST /login HTTP/1.1" 401 56
8.8.8.8 - - [02/Oct/2026:14:30:04 +0700] "GET /api/data HTTP/1.1" 200 4567
192.168.1.100 - - [02/Oct/2026:14:30:05 +0700] "GET /dashboard HTTP/1.1" 200 8901
10.0.0.5 - - [02/Oct/2026:14:30:06 +0700] "POST /admin/login HTTP/1.1" 403 0"""

limiter = IPRateLimiter()
result = limiter.analyze_log_file(sample_log)

print("\nLog Analysis:")
print(f"  Total requests: {result['total_requests']}")
print(f"  Unique IPs: {result['unique_ips']}")
print("\n  Top IPs:")
for ip, count in result['top_ips']:
    print(f"    {ip:20} {count} requests")
print("\n  Login attempts:")
for ip, count in result['login_attempts'].items():
    flag = '⚠️ SUSPICIOUS' if count >= 3 else ''
    print(f"    {ip:20} {count} attempts {flag}")
```

---

## 16.7 JavaScript IP Validation

```javascript
// ============================================
// JavaScript IP Address Validation
// ============================================

class IPValidator {
    
    static IPv4_PATTERN = /^(?:(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\.){3}(?:25[0-5]|2[0-4]\d|[01]?\d\d?)$/;
    
    static IPv6_PATTERN = /^(?:[0-9a-fA-F]{1,4}:){7}[0-9a-fA-F]{1,4}$|^(?:[0-9a-fA-F]{1,4}:){1,7}:$|^:(?::[0-9a-fA-F]{1,4}){1,7}$|^::$/;
    
    static CIDR_PATTERN = /^(?:(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\.){3}(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\/([0-9]|[12]\d|3[0-2])$/;
    
    static isIPv4(ip) {
        return this.IPv4_PATTERN.test(ip);
    }
    
    static isIPv6(ip) {
        return this.IPv6_PATTERN.test(ip.replace(/^\[|\]$/g, ''));
    }
    
    static isPrivate(ip) {
        const patterns = [
            /^10\.\d{1,3}\.\d{1,3}\.\d{1,3}$/,
            /^172\.(1[6-9]|2\d|3[01])\.\d{1,3}\.\d{1,3}$/,
            /^192\.168\.\d{1,3}\.\d{1,3}$/,
            /^127\.\d{1,3}\.\d{1,3}\.\d{1,3}$/,
        ];
        return patterns.some(p => p.test(ip));
    }
    
    static classify(ip) {
        if (!this.isIPv4(ip)) return { valid: false };
        return {
            valid: true,
            isPrivate: this.isPrivate(ip),
            isLoopback: /^127\./.test(ip),
            isLinkLocal: /^169\.254\./.test(ip),
            isMulticast: /^2(?:2[4-9]|3\d)\./.test(ip),
        };
    }
    
    static extractFromText(text) {
        const pattern = /\b(?:(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\.){3}(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\b/g;
        return [...new Set(text.match(pattern) || [])];
    }
}

// ทดสอบ
const ips = ['192.168.1.1', '8.8.8.8', '127.0.0.1', '256.0.0.1', '::1'];

ips.forEach(ip => {
    if (IPValidator.isIPv4(ip)) {
        const info = IPValidator.classify(ip);
        const type = info.isLoopback ? 'loopback' : info.isPrivate ? 'private' : 'public';
        console.log(`✅ ${ip} → ${type}`);
    } else if (IPValidator.isIPv6(ip)) {
        console.log(`✅ ${ip} → IPv6`);
    } else {
        console.log(`❌ ${ip} → invalid`);
    }
});
```

---

## 16.8 สรุป Part 16

```
Pattern สำคัญสำหรับ IP Addresses:
────────────────────────────────────────────────────────────
IPv4 (strict):  ^(?:(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\.){3}(?:25[0-5]|2[0-4]\d|[01]?\d\d?)$
IPv4 (extract): \b(?:(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\.){3}(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\b
IPv4+CIDR:      <ipv4>/(?:[0-9]|[12]\d|3[0-2])
IPv6 loopback:  ^::1$
Private 10.x:   ^10\.\d{1,3}\.\d{1,3}\.\d{1,3}$
Private 172.x:  ^172\.(1[6-9]|2\d|3[01])\.\d{1,3}\.\d{1,3}$
Private 192.x:  ^192\.168\.\d{1,3}\.\d{1,3}$
```

✅ **IPv4 Validator** — ตรวจสอบ 0-255 per octet  
✅ **IPv6 Validator** — รองรับ full, compressed, IPv4-mapped  
✅ **CIDR Parser** — คำนวณ network, broadcast, hosts  
✅ **Security Classifier** — จำแนก private/public/loopback  
✅ **Rate Limiter** — ตรวจจับ IP ที่ส่ง request มากผิดปกติ  

---

*[⬅ Part 15: Password Validation](part-15-password-validation.md) | [➡ Part 17: HTML Parsing](part-17-html-parsing.md)*
