# Part 92: Regex in Data Science & ML Pipelines

> **ระดับ:** ผู้เชี่ยวชาญ-ระดับโลก | **เวลาเรียน:** ~110 นาที | **ข้อกำหนด:** Part 01-91

---

## 92.1 Text Preprocessing for NLP

```python
import re
from typing import List, Dict, Tuple

print("Regex in Data Science & ML Pipelines:")
print("=" * 60)

print("\n1. Text preprocessing for NLP:")

# Remove HTML tags
HTML_TAG      = re.compile(r'<[^>]+>')
HTML_ENTITY   = re.compile(r'&(?:[a-zA-Z]+|#\d+|#x[0-9a-fA-F]+);')

# URL removal (for sentiment analysis — URLs add noise)
URL_PATTERN   = re.compile(r'https?://\S+|www\.\S+')

# Social media patterns
HASHTAG       = re.compile(r'#(\w+)')
MENTION       = re.compile(r'@(\w+)')

# Repeated characters normalization: "sooooo good" → "so good"
REPEATED_CHARS = re.compile(r'(.)\1{2,}')

# Contractions expansion
CONTRACTIONS = {
    re.compile(r"\bcan't\b",     re.IGNORECASE): "cannot",
    re.compile(r"\bwon't\b",     re.IGNORECASE): "will not",
    re.compile(r"\bdon't\b",     re.IGNORECASE): "do not",
    re.compile(r"\bisn't\b",     re.IGNORECASE): "is not",
    re.compile(r"\bI'm\b",       re.IGNORECASE): "I am",
    re.compile(r"\bI've\b",      re.IGNORECASE): "I have",
    re.compile(r"\bI'll\b",      re.IGNORECASE): "I will",
    re.compile(r"\bthey're\b",   re.IGNORECASE): "they are",
    re.compile(r"\bwe're\b",     re.IGNORECASE): "we are",
}

# Number normalization patterns
ORDINAL       = re.compile(r'\b(\d+)(?:st|nd|rd|th)\b', re.IGNORECASE)
PERCENTAGE    = re.compile(r'\b(\d+(?:\.\d+)?)\s*%')
CURRENCY_NUM  = re.compile(r'[$€£¥]\s*(\d+(?:[,\d]*)?(?:\.\d+)?)')


def preprocess_text(text: str, options: dict = None) -> str:
    opts = options or {
        'remove_html': True,
        'remove_urls': True,
        'normalize_repeats': True,
        'expand_contractions': True,
        'lowercase': True,
    }

    if opts.get('remove_html'):
        text = HTML_TAG.sub(' ', text)
        text = HTML_ENTITY.sub(' ', text)

    if opts.get('remove_urls'):
        text = URL_PATTERN.sub(' URL ', text)

    if opts.get('expand_contractions'):
        for pattern, replacement in CONTRACTIONS.items():
            text = pattern.sub(replacement, text)

    if opts.get('normalize_repeats'):
        text = REPEATED_CHARS.sub(r'\1\1', text)  # Keep 2 max

    if opts.get('lowercase'):
        text = text.lower()

    # Normalize whitespace
    text = re.sub(r'\s+', ' ', text).strip()

    return text


sample_texts = [
    '<p>This is <b>amazing</b>! Check https://example.com for more &amp; details.</p>',
    "I can't believe how sooooo good this movie is!!! Won't stop watching!!",
    'The stock fell 15% today. They\'re worried about the $2,500 quarterly target.',
    '@alice Great work! #python #regex are my favorites :heart:',
]

print(f"\n   NLP preprocessing:")
for text in sample_texts:
    processed = preprocess_text(text)
    print(f"\n   Input:  {text[:70]!r}")
    print(f"   Output: {processed[:70]!r}")
```

---

## 92.2 Feature Extraction from Text

```python
import re
from typing import Dict, List
from collections import Counter

print("\nFeature Extraction from Text:")
print("=" * 60)

# Named entity patterns (simple rule-based NER)
DATE_NER = re.compile(
    r'\b(?:Jan(?:uary)?|Feb(?:ruary)?|Mar(?:ch)?|Apr(?:il)?|May|Jun(?:e)?|'
    r'Jul(?:y)?|Aug(?:ust)?|Sep(?:tember)?|Oct(?:ober)?|Nov(?:ember)?|Dec(?:ember)?)'
    r'\s+\d{1,2},?\s+\d{4}|\d{4}-\d{2}-\d{2}|\d{1,2}/\d{1,2}/\d{4}',
    re.IGNORECASE
)

ORG_PATTERNS = [
    re.compile(r'\b[A-Z][a-z]+\s+(?:Inc|Ltd|Corp|LLC|GmbH|Co)\b\.?'),
    re.compile(r'\b(?:University|College|Institute)\s+of\s+[A-Z][a-zA-Z]+'),
]

LOCATION_HINTS = re.compile(
    r'\b(?:in|at|from|near|based\s+in)\s+([A-Z][a-zA-Z]+(?:\s+[A-Z][a-zA-Z]+)?)\b'
)

# Technical feature extraction
CODE_SNIPPET   = re.compile(r'`[^`]+`|```[\s\S]+?```')
VERSION_NUM    = re.compile(r'\bv?(\d+\.\d+(?:\.\d+)*(?:-[a-zA-Z0-9.+]+)?)\b')
CVE_ID         = re.compile(r'\bCVE-\d{4}-\d{4,7}\b', re.IGNORECASE)
RFC_REF        = re.compile(r'\bRFC\s*(\d{3,5})\b', re.IGNORECASE)


def extract_features(text: str) -> Dict:
    features = {}

    # Dates
    dates = DATE_NER.findall(text)
    if dates:
        features['dates'] = dates

    # Code references
    codes = CODE_SNIPPET.findall(text)
    if codes:
        features['code_snippets'] = len(codes)

    # Version numbers
    versions = VERSION_NUM.findall(text)
    if versions:
        features['versions'] = list(set(versions))

    # CVEs
    cves = CVE_ID.findall(text)
    if cves:
        features['cves'] = cves

    # RFCs
    rfcs = RFC_REF.findall(text)
    if rfcs:
        features['rfcs'] = rfcs

    # Organizations
    orgs = []
    for pattern in ORG_PATTERNS:
        orgs.extend(pattern.findall(text))
    if orgs:
        features['organizations'] = orgs

    # Locations (hint-based)
    locs = LOCATION_HINTS.findall(text)
    if locs:
        features['locations'] = locs

    # Basic text statistics
    words = re.findall(r'\b\w+\b', text)
    sentences = re.split(r'[.!?]+', text)
    features['word_count'] = len(words)
    features['sentence_count'] = len([s for s in sentences if s.strip()])
    features['avg_word_length'] = round(sum(len(w) for w in words) / len(words), 2) if words else 0

    return features


sample_docs = [
    """On January 15, 2024, Google Inc. released version 3.11.2 of their library.
    The update fixes CVE-2024-12345 and CVE-2024-67890. See RFC 7230 for HTTP spec.
    Based in Mountain View, the team used `re.compile()` for optimization.""",

    """The University of Cambridge published research on v2.0.0-beta of their model.
    Released on 2024-03-01, it addresses issues found in version 1.9.5.""",
]

print(f"\n   Feature extraction:")
for i, doc in enumerate(sample_docs, 1):
    features = extract_features(doc)
    print(f"\n   Document {i}: {doc[:60]!r}...")
    for key, val in features.items():
        print(f"     {key}: {val}")
```

---

## 92.3 Dataset Cleaning Patterns

```python
import re
from typing import List, Optional, Dict

print("\nDataset Cleaning Patterns:")
print("=" * 60)

# Phone number normalization (multiple formats → E.164)
PHONE_FORMATS = [
    re.compile(r'^\+?1?\s*\(?(\d{3})\)?[\s.\-](\d{3})[\s.\-](\d{4})$'),  # US
    re.compile(r'^\+?66\s*(\d{2})\s*(\d{3})\s*(\d{4})$'),                 # Thailand +66
    re.compile(r'^0(\d{2})\s*(\d{3})\s*(\d{4})$'),                         # TH local 0XX
]

def normalize_phone(phone: str) -> Optional[str]:
    phone = phone.strip()
    m = PHONE_FORMATS[0].match(phone)
    if m:
        return f"+1{m.group(1)}{m.group(2)}{m.group(3)}"
    m = PHONE_FORMATS[1].match(phone)
    if m:
        return f"+66{m.group(1)}{m.group(2)}{m.group(3)}"
    m = PHONE_FORMATS[2].match(phone)
    if m:
        return f"+66{m.group(1)}{m.group(2)}{m.group(3)}"
    return None


# Duplicate detection (fingerprint normalization)
def text_fingerprint(text: str) -> str:
    """Create fingerprint for near-duplicate detection."""
    t = text.lower()
    t = re.sub(r'[^\w\s]', '', t)
    t = re.sub(r'\s+', ' ', t).strip()
    words = sorted(set(t.split()))  # Sort + deduplicate
    return ' '.join(words[:20])


# PII detection for data masking
SSN_PATTERN  = re.compile(r'\b\d{3}[-\s]?\d{2}[-\s]?\d{4}\b')
CC_PATTERN   = re.compile(r'\b(?:4\d{3}|5[1-5]\d{2}|6011|3[47]\d{2})\s*[\s\-]?\d{4}\s*[\s\-]?\d{4}\s*[\s\-]?\d{4}\b')
EMAIL_MASK   = re.compile(r'([a-zA-Z0-9._%+\-]+)(@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,})')

def mask_pii(text: str) -> str:
    """Mask PII fields for safe dataset usage."""
    text = SSN_PATTERN.sub('XXX-XX-XXXX', text)
    text = CC_PATTERN.sub('XXXX-XXXX-XXXX-XXXX', text)
    text = EMAIL_MASK.sub(lambda m: m.group(1)[:2] + '***' + m.group(2), text)
    return text


phone_samples = [
    '(555) 123-4567',
    '555.123.4567',
    '+1-555-123-4567',
    '+66 81 234 5678',
    '081-234-5678',
    '02-345-6789',
]

print(f"\n   Phone normalization:")
for phone in phone_samples:
    normalized = normalize_phone(phone)
    print(f"   {phone!r:<20} → {normalized or 'INVALID'}")

print(f"\n   PII masking:")
pii_samples = [
    'SSN: 123-45-6789, Email: alice@example.com',
    'Contact: bob123@company.org, Card: 4111 1111 1111 1111',
    'Normal text with no PII',
]
for text in pii_samples:
    masked = mask_pii(text)
    print(f"   Before: {text!r}")
    print(f"   After:  {masked!r}")
    print()
```

---

## 92.4 สรุป Part 92

```
Regex in Data Science & ML Pipelines:

1. NLP preprocessing:
   HTML stripping: <[^>]+> removes tags; &amp; → entity decoder
   URL removal: https?://\S+ replaced with token URL
   Contraction expansion: compile dict of pattern → replacement
   Repeat normalization: (.)\1{2,} → keep max 2 repeats

2. Feature extraction (rule-based NER):
   Dates: named months + ordinals + ISO format
   Organizations: Inc/Ltd/Corp suffix patterns
   Technical: CVE-\d{4}-\d+, RFC \d+, version v1.2.3
   Combine regex with ML: use regex as features for downstream model

3. Dataset cleaning:
   Phone normalization: multiple formats → E.164 (+CC XXXXXXXXXX)
   Address parsing: number + street + type + city + state + zip
   Duplicate detection: text fingerprint = sorted unique words

4. PII masking:
   SSN: \b\d{3}[-\s]?\d{2}[-\s]?\d{4}\b
   Credit card: Visa/MC/Amex prefix + 16-digit groups
   Email: keep domain, mask local part: al***@example.com
   Always mask BEFORE storing dataset or sharing externally

5. Pipeline order:
   PII mask → HTML strip → URL replace → normalize → tokenize
   Apply regex operations in order: earlier steps simplify later ones
   Benchmark on representative samples before production deployment
```

---

*[← Part 91: Real-World Regex Applications](part-91-realworld.md) | [→ Part 93: Regex Testing & Quality Assurance](part-93-testing-qa.md)*
