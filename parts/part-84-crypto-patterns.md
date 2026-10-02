# Part 84: Cryptographic Pattern Analysis

> **ระดับ:** ผู้เชี่ยวชาญ-ระดับโลก | **เวลาเรียน:** ~95 นาที | **ข้อกำหนด:** Part 01-83

---

## 84.1 Cryptographic Hash Patterns

```python
import re
from typing import Dict, List

print("Cryptographic Pattern Analysis:")
print("=" * 60)

print("\n1. Hash format detection and validation:")

# Hash format patterns
MD5_HASH    = re.compile(r'\b[0-9a-f]{32}\b', re.IGNORECASE)
SHA1_HASH   = re.compile(r'\b[0-9a-f]{40}\b', re.IGNORECASE)
SHA256_HASH = re.compile(r'\b[0-9a-f]{64}\b', re.IGNORECASE)
SHA512_HASH = re.compile(r'\b[0-9a-f]{128}\b', re.IGNORECASE)

# Bcrypt format: $2a$12$<22chars-salt><31chars-hash>
BCRYPT = re.compile(r'^\$2[aby]\$(?:0[4-9]|[12]\d|3[01])\$[./A-Za-z0-9]{53}$')

# Argon2 format
ARGON2 = re.compile(
    r'^\$argon2(?:i|d|id)\$v=(?:16|19)\$m=\d+,t=\d+,p=\d+\$'
    r'[A-Za-z0-9+/]+=*\$[A-Za-z0-9+/]+=*$'
)

# PBKDF2 format (Django style)
PBKDF2 = re.compile(
    r'^pbkdf2_(?:sha1|sha256|sha512)\$\d+\$[A-Za-z0-9]+\$[A-Za-z0-9+/]+=*$'
)

# scrypt format
SCRYPT = re.compile(
    r'^\$scrypt\$N=\d+,r=\d+,p=\d+\$[A-Za-z0-9+/]+=*\$[A-Za-z0-9+/]+=*$'
)

# Unix crypt formats
CRYPT_MD5    = re.compile(r'^\$1\$[./A-Za-z0-9]{8}\$[./A-Za-z0-9]{22}$')
CRYPT_SHA256 = re.compile(r'^\$5\$(?:rounds=\d+\$)?[./A-Za-z0-9]{1,16}\$[./A-Za-z0-9]{43}$')
CRYPT_SHA512 = re.compile(r'^\$6\$(?:rounds=\d+\$)?[./A-Za-z0-9]{1,16}\$[./A-Za-z0-9]{86}$')

# Windows NTLM hash (32 hex = same length as MD5)
NTLM_HASH = re.compile(r'^[0-9A-F]{32}$')  # UPPERCASE hint (not definitive)


def identify_hash(hash_str: str) -> str:
    if BCRYPT.match(hash_str):
        rounds = int(hash_str.split('$')[2])
        return f'bcrypt (cost={rounds}, {"STRONG" if rounds >= 12 else "WEAK"})'
    if ARGON2.match(hash_str):
        return 'argon2 (recommended for new systems)'
    if PBKDF2.match(hash_str):
        parts = hash_str.split('$')
        return f'PBKDF2 ({parts[0].split("_")[1].upper()}, iterations={parts[1]})'
    if SCRYPT.match(hash_str):
        return 'scrypt'
    if CRYPT_SHA512.match(hash_str):
        return 'SHA-512 crypt (Unix shadow)'
    if CRYPT_SHA256.match(hash_str):
        return 'SHA-256 crypt (Unix shadow)'
    if CRYPT_MD5.match(hash_str):
        return 'MD5 crypt (Unix, WEAK)'
    if NTLM_HASH.match(hash_str):
        return 'possible NTLM hash (WEAK - no salt)'
    if len(hash_str) == 128 and SHA512_HASH.match(hash_str):
        return 'SHA-512 (unsalted, WEAK for passwords)'
    if len(hash_str) == 64 and SHA256_HASH.match(hash_str):
        return 'SHA-256 (unsalted, WEAK for passwords)'
    if len(hash_str) == 40 and SHA1_HASH.match(hash_str):
        return 'SHA-1 (unsalted, WEAK for passwords)'
    if len(hash_str) == 32 and MD5_HASH.match(hash_str):
        return 'MD5 (unsalted, VERY WEAK for passwords)'
    return 'unknown hash format'


hash_samples = [
    '$2b$12$LQv3c1yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/LewdJxnkLvWxGn3Cm',
    '$2b$06$LQv3c1yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/LewdJxnkLvWxGn3Cm',
    '$argon2id$v=19$m=65536,t=3,p=4$c2FsdHZhbHVl$RdescudvJCsgt3ub+b+dWRWJTmaaJObG',
    'pbkdf2_sha256$260000$randomsalt123$hashedpasswordhere==',
    '5f4dcc3b5aa765d61d8327deb882cf99',
    '0a4d55a8d778e5022fab701977c5d840bbc486d0',
    '5e884898da28047151d0e56f8dc6292773603d0d6aabbdd62a11ef721d1542d8',
    'AABBCCDDEEFF00112233445566778899',
]

print(f"\n   Hash identification:")
for h in hash_samples:
    result = identify_hash(h)
    is_weak = 'WEAK' in result or 'VERY WEAK' in result
    marker = 'WEAK' if is_weak else 'OK'
    print(f"   [{marker}] {h[:50]!r}")
    print(f"     → {result}")
```

---

## 84.2 TLS/SSL Certificate Pattern Analysis

```python
import re
from typing import Dict, List

print("\nTLS/SSL Certificate Pattern Analysis:")
print("=" * 60)

# PEM certificate markers
PEM_CERT  = re.compile(r'-----BEGIN CERTIFICATE-----\n([A-Za-z0-9+/\n]+=*)\n-----END CERTIFICATE-----', re.DOTALL)
PEM_KEY   = re.compile(r'-----BEGIN (?:RSA |EC )?PRIVATE KEY-----\n([A-Za-z0-9+/\n]+=*)\n-----END (?:RSA |EC )?PRIVATE KEY-----', re.DOTALL)
PEM_CSR   = re.compile(r'-----BEGIN CERTIFICATE REQUEST-----')
PEM_DH    = re.compile(r'-----BEGIN DH PARAMETERS-----')

# Common Name extraction from DN
DN_CN   = re.compile(r'CN\s*=\s*([^,/\r\n]+)')
DN_ORG  = re.compile(r'O\s*=\s*([^,/\r\n]+)')
DN_SAN  = re.compile(r'DNS:([^,\s]+)')

# Weak cipher suites
WEAK_CIPHERS = re.compile(
    r'(?:RC4|DES(?!-EDE3)|3DES|MD5|NULL|EXPORT|ANON|'
    r'SSL_?2|SSL_?3|TLS1(?:\.0)?(?!\.))',
    re.IGNORECASE
)

# Strong cipher pattern
STRONG_TLS = re.compile(
    r'TLS(?:v)?1\.[23]|'
    r'ECDHE-(?:RSA|ECDSA)-AES(?:128|256)-GCM-SHA(?:256|384)|'
    r'TLS_AES_(?:128|256)_GCM_SHA(?:256|384)',
    re.IGNORECASE
)

def is_likely_selfsigned(cert_text: str) -> bool:
    subjects = DN_CN.findall(cert_text)
    return len(set(subjects)) == 1 and len(subjects) >= 2

# Certificate transparency log detection
CT_LOG = re.compile(r'CT\s+Precertificate\s+SCTs|Certificate\s+Transparency')

# OCSP/CRL URL extraction
OCSP_URL = re.compile(r'OCSP\s*-\s*URI:\s*(https?://[^\s]+)')
CRL_URL  = re.compile(r'CRL\s+Distribution\s+Points.*?URI:(https?://[^\s]+)', re.DOTALL)

# Certificate pinning header
HPKP      = re.compile(r'Public-Key-Pins(?:-Report-Only)?:\s*([^\r\n]+)', re.IGNORECASE)
EXPECT_CT = re.compile(r'Expect-CT:\s*([^\r\n]+)', re.IGNORECASE)


cert_info_samples = [
    "Subject: CN=example.com, O=Example Corp, C=US\nIssuer: CN=Let's Encrypt R3, O=Let's Encrypt",
    'Subject: CN=test.local\nIssuer: CN=test.local',
    'TLSv1.0 cipher RC4-MD5',
    'TLS1.3 ECDHE-RSA-AES256-GCM-SHA384',
    'SSL2.0 EXPORT-DES-CBC',
]

print(f"\n   TLS/Certificate analysis:")
for sample in cert_info_samples:
    issues = []
    if is_likely_selfsigned(sample):
        issues.append('self-signed certificate')
    weak_m = WEAK_CIPHERS.search(sample)
    if weak_m:
        issues.append(f'weak cipher/protocol: {weak_m.group(0)!r}')
    if STRONG_TLS.search(sample):
        issues.append(f'strong TLS: {STRONG_TLS.search(sample).group(0)!r}')

    marker = 'OK' if not issues or all('strong' in i for i in issues) else 'WARN'
    print(f"\n   [{marker}] {sample[:60]!r}")
    for issue in issues:
        print(f"     → {issue}")
```

---

## 84.3 Cryptocurrency Address Patterns

```python
import re
from typing import Dict

print("\nCryptocurrency Address Patterns:")
print("=" * 60)

# Bitcoin address (P2PKH, P2SH, Bech32)
BTC_P2PKH  = re.compile(r'\b1[A-HJ-NP-Za-km-z1-9]{25,34}\b')
BTC_P2SH   = re.compile(r'\b3[A-HJ-NP-Za-km-z1-9]{25,34}\b')
BTC_BECH32 = re.compile(r'\bbc1[a-z0-9]{6,87}\b')

# Ethereum address
ETH_ADDR = re.compile(r'\b0x[0-9a-fA-F]{40}\b')

# Monero address (starts with 4, 95 chars)
XMR_ADDR = re.compile(r'\b4[0-9AB][1-9A-HJ-NP-Za-km-z]{93}\b')

# Litecoin address
LTC_ADDR = re.compile(r'\b[LM3][A-HJ-NP-Za-km-z1-9]{25,34}\b')

# Tron address (starts with T)
TRX_ADDR = re.compile(r'\bT[A-HJ-NP-Za-km-z1-9]{33}\b')


def detect_crypto_addresses(text: str) -> Dict:
    found = {}
    patterns = {
        'bitcoin_p2pkh':  BTC_P2PKH,
        'bitcoin_p2sh':   BTC_P2SH,
        'bitcoin_bech32': BTC_BECH32,
        'ethereum':       ETH_ADDR,
        'monero':         XMR_ADDR,
        'litecoin':       LTC_ADDR,
        'tron':           TRX_ADDR,
    }
    for name, pattern in patterns.items():
        matches = pattern.findall(text)
        if matches:
            found[name] = matches
    return found


ransomware_note = """
Your files have been encrypted.
Send 0.5 BTC to: 1A1zP1eP5QGefi2DMPTfTL5SLmv7Divf (original genesis address example)
Or ETH: 0x742d35Cc6634C0532925a3b844Bc454e4438f44e
For Monero: 44AFFq5kSiGBoZ4NMDwYtN18obc8AemS33DBLWs3H7otXft3XjrpDtQGv7SqSsaBYBb98uNbr2VBBEt7f2wfn3RVGQBEP3A
Recovery key will be sent after payment confirmed.
"""

found = detect_crypto_addresses(ransomware_note)
print(f"\n   Cryptocurrency address detection (ransomware note):")
for coin, addrs in found.items():
    for addr in addrs:
        print(f"   [{coin.upper()}] {addr[:60]}")
```

---

## 84.4 สรุป Part 84

```
Cryptographic Pattern Analysis:

1. Password hash strength ranking:
   BEST:  argon2id (memory-hard, resistant to GPU cracking)
   GOOD:  bcrypt cost>=12 (adaptive, time-tested)
   GOOD:  scrypt (memory-hard)
   OK:    PBKDF2-SHA256 (>= 260000 iterations)
   WEAK:  MD5-crypt ($1$), unsalted SHA-256/SHA-1
   BAD:   Plain MD5, unsalted SHA1 (rainbow table trivial)
   WORST: Plaintext, ROT13, base64 (not encryption)

2. TLS security:
   GOOD: TLS 1.3, TLS 1.2 + ECDHE + AES-GCM
   WEAK: TLS 1.0/1.1 (deprecated RFC 8996)
   BAD:  SSL 3.0, SSL 2.0 (POODLE, DROWN attacks)
   BAD:  RC4 (BEAST, Lucky13), DES, EXPORT ciphers, NULL
   Self-signed: acceptable only in dev/test environments

3. Cipher strength:
   Forward secrecy: ECDHE or DHE key exchange
   AEAD: AES-GCM, ChaCha20-Poly1305 (good)
   Non-AEAD: AES-CBC (IV padding oracle risk)

4. Cryptocurrency patterns:
   Bitcoin P2PKH: starts with 1 (legacy)
   Bitcoin P2SH: starts with 3 (wrapped segwit)
   Bitcoin Bech32: starts with bc1 (native segwit)
   Ethereum: 0x + 40 hex chars
   Monero: starts with 4, 95 chars (privacy coin)
   TRX (Tron): starts with T, 34 chars

5. Detection use cases:
   Ransomware: find BTC/ETH payment addresses in notes
   DLP: detect private keys in source code/logs
   Credential audit: detect weak hash algorithms
   TLS scan: detect deprecated protocols/ciphers
```

---

*[← Part 83: Binary & Hex Patterns](part-83-binary-hex.md) | [→ Part 85: Forensics & Memory Pattern Analysis](part-85-forensics.md)*
