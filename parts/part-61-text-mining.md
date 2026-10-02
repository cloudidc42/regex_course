# Part 61: Advanced Text Mining & NLP Patterns

> **ระดับ:** สูง | **เวลาเรียน:** ~80 นาที | **ข้อกำหนด:** Part 01-60

---

## 61.1 Tokenization & Word Segmentation

```python
import re
from collections import Counter
from typing import List, Dict, Iterator

print("Advanced Text Mining & NLP Patterns:")
print("=" * 60)

print("\n1. Tokenization strategies:")

WORD_TOKEN    = re.compile(r'\b[a-zA-Z]+\b')
WORD_NUM      = re.compile(r'\b[a-zA-Z0-9]+\b')
SENTENCE_END  = re.compile(r'(?<=[.!?])\s+(?=[A-Z])')
SENT_BOUNDARY = re.compile(
    r'(?<!\w\.\w.)(?<![A-Z][a-z]\.)(?<=\.|\?|!)\s'
)
CLAUSE        = re.compile(r'(?<=[,;:])\s+')

def tokenize_words(text: str) -> List[str]:
    return WORD_TOKEN.findall(text.lower())

def tokenize_sentences(text: str) -> List[str]:
    return [s.strip() for s in SENT_BOUNDARY.split(text) if s.strip()]

def tokenize_n_grams(tokens: List[str], n: int) -> List[tuple]:
    return [tuple(tokens[i:i+n]) for i in range(len(tokens)-n+1)]


sample_text = """
The quick brown fox jumps over the lazy dog. Natural language processing
is a field of AI and linguistics. Dr. Smith visited the U.S.A. last year.
Regex patterns can tokenize text efficiently! Are you ready?
"""

words     = tokenize_words(sample_text)
sentences = tokenize_sentences(sample_text)
bigrams   = tokenize_n_grams(words[:10], 2)
trigrams  = tokenize_n_grams(words[:10], 3)

print(f"\n   Words (first 12):  {words[:12]}")
print(f"   Sentences:  {len(sentences)}")
for i, s in enumerate(sentences):
    print(f"     [{i+1}] {s[:70]}")
print(f"\n   Bigrams:   {bigrams[:5]}")
print(f"   Trigrams:  {trigrams[:3]}")

print("\n\n2. Frequency analysis:")

word_freq = Counter(words)
print(f"\n   Top 10 words:")
for word, count in word_freq.most_common(10):
    bar = '█' * count
    print(f"   {word:<15} {count:2d} {bar}")
```

---

## 61.2 Named Entity Recognition (Regex-based)

```python
import re
from typing import List, Tuple, Dict

print("\nNamed Entity Recognition (Regex-based):")
print("=" * 60)

DATE_PATTERNS = [
    re.compile(r'\b(?:January|February|March|April|May|June|July|August|'
               r'September|October|November|December)\s+\d{1,2},\s+\d{4}\b'),
    re.compile(r'\b\d{1,2}/\d{1,2}/\d{2,4}\b'),
    re.compile(r'\b\d{4}-\d{2}-\d{2}\b'),
    re.compile(r'\b(?:Mon|Tue|Wed|Thu|Fri|Sat|Sun)(?:day)?,\s+\w+ \d{1,2}\b'),
]

TIME_PATTERNS = [
    re.compile(r'\b\d{1,2}:\d{2}(?::\d{2})?\s*(?:AM|PM|am|pm)?\b'),
    re.compile(r'\b\d{1,2}\s*(?:AM|PM|am|pm)\b'),
]

MONEY_PATTERNS = [
    re.compile(r'(?:USD|THB|EUR|GBP|JPY|\$|€|\xa3|\xa5|฿)\s*[\d,]+(?:\.\d{2})?'),
    re.compile(r'[\d,]+(?:\.\d{2})?\s*(?:USD|THB|EUR|GBP|baht|บาท)'),
]

PROPER_NOUN = re.compile(r'\b([A-Z][a-z]+(?:\s+[A-Z][a-z]+)*)\b')

ORGANIZATION = re.compile(
    r'\b([A-Z][a-zA-Z&\s]+(?:Co\.?|Corp\.?|Inc\.?|Ltd\.?|LLC|Group|Bank|'
    r'Foundation|Institute|University|College))\b'
)

LOCATION = re.compile(
    r'\b(?:in|at|near|from|to)\s+([A-Z][a-z]+(?:\s+[A-Z][a-z]+)*)'
)


def extract_entities(text: str) -> Dict[str, List[str]]:
    entities = {
        'dates':      [],
        'times':      [],
        'money':      [],
        'locations':  [],
        'orgs':       [],
        'proper':     [],
    }
    for pat in DATE_PATTERNS:
        entities['dates'].extend(pat.findall(text))
    for pat in TIME_PATTERNS:
        entities['times'].extend(pat.findall(text))
    for pat in MONEY_PATTERNS:
        entities['money'].extend(pat.findall(text))
    entities['locations'] = [m.group(1) for m in LOCATION.finditer(text)]
    entities['orgs']      = [m.group(1) for m in ORGANIZATION.finditer(text)]
    entities['proper']    = [m.group(1) for m in PROPER_NOUN.finditer(text)
                             if m.group(1) not in ['I', 'The', 'A', 'An']]
    return {k: list(set(v)) for k, v in entities.items() if v}


ner_text = """
Alice Johnson joined Microsoft Corp. in January 15, 2023 at 9:30 AM.
The company paid USD 5,000 for training held in Bangkok, Thailand.
Bob Smith from Google Inc. attended the meeting at 14:00.
The conference in Silicon Valley started on 2024-03-10.
Sarah Lee visited the Bank of Thailand Foundation last Monday.
"""

entities = extract_entities(ner_text)
print(f"\n   Named entities found:")
for ent_type, ents in entities.items():
    print(f"   {ent_type:<12}: {ents}")
```

---

## 61.3 Pattern-based Sentiment Analysis

```python
import re
from typing import Dict, Tuple

print("\nPattern-based Sentiment Analysis:")
print("=" * 60)

POSITIVE_WORDS = re.compile(
    r'\b(?:good|great|excellent|amazing|wonderful|fantastic|awesome|'
    r'brilliant|perfect|love|beautiful|outstanding|superb|impressive|'
    r'pleased|happy|satisfied|recommend|helpful|useful|best)\b',
    re.IGNORECASE
)

NEGATIVE_WORDS = re.compile(
    r'\b(?:bad|terrible|awful|horrible|poor|worst|hate|ugly|broken|'
    r'useless|disappointing|frustrated|annoyed|waste|fail|failed|'
    r'broken|cheap|slow|buggy|problem|issue|error|bug)\b',
    re.IGNORECASE
)

NEGATION = re.compile(
    r"\b(?:not|no|never|cannot|can't|don't|doesn't|didn't|won't|"
    r"wouldn't|shouldn't|isn't|aren't|wasn't|weren't)\b\s+\w+",
    re.IGNORECASE
)

INTENSIFIERS = re.compile(
    r'\b(?:very|extremely|absolutely|totally|completely|incredibly|'
    r'remarkably|exceptionally|highly|quite|rather)\b',
    re.IGNORECASE
)


def analyze_sentiment(text: str) -> Dict:
    pos_matches = POSITIVE_WORDS.findall(text)
    neg_matches = NEGATIVE_WORDS.findall(text)
    neg_contexts = NEGATION.findall(text)
    intensifiers = INTENSIFIERS.findall(text)

    pos_score = len(pos_matches) * 1.0
    neg_score = len(neg_matches) * 1.0

    for nc in neg_contexts:
        words = nc.lower().split()
        if len(words) >= 2:
            if POSITIVE_WORDS.match(words[-1]):
                pos_score -= 1
                neg_score += 0.5
            elif NEGATIVE_WORDS.match(words[-1]):
                neg_score -= 1
                pos_score += 0.5

    for _ in intensifiers:
        if pos_score > neg_score:
            pos_score *= 1.2
        else:
            neg_score *= 1.2

    total = pos_score + neg_score
    if total == 0:
        label = 'neutral'
        confidence = 0.5
    elif pos_score > neg_score:
        label = 'positive'
        confidence = pos_score / total
    else:
        label = 'negative'
        confidence = neg_score / total

    return {
        'label':        label,
        'confidence':   round(confidence, 3),
        'pos_words':    pos_matches,
        'neg_words':    neg_matches,
        'negations':    len(neg_contexts),
        'intensifiers': intensifiers,
    }


reviews = [
    "This product is absolutely amazing! I love it and highly recommend it.",
    "Terrible experience. The service was awful and the product completely broken.",
    "Not bad at all. It's quite helpful but not the best I've used.",
    "Very disappointed. The app is slow and buggy. Won't buy again.",
    "Excellent quality. Extremely satisfied with this outstanding purchase!",
]

print(f"\n   {'Review':<55} {'Label':<10} {'Conf'}")
print(f"   {'-'*55} {'-'*10} {'-'*5}")
for review in reviews:
    result = analyze_sentiment(review)
    label_sym = '+' if result['label'] == 'positive' else ('-' if result['label'] == 'negative' else '~')
    short = review[:52] + '...' if len(review) > 55 else review
    print(f"   {short:<55} [{label_sym}] {result['label']:<8} {result['confidence']:.3f}")
```

---

## 61.4 Keyword Extraction & TF-IDF-like Scoring

```python
import re
import math
from collections import Counter
from typing import List, Dict

print("\nKeyword Extraction:")
print("=" * 60)

WORD_RE   = re.compile(r'\b[a-z]{3,}\b')
STOP_WORDS = re.compile(
    r'^(?:the|a|an|and|or|but|in|on|at|to|for|of|with|by|'
    r'from|is|are|was|were|be|been|being|have|has|had|do|'
    r'does|did|will|would|could|should|may|might|this|that|'
    r'these|those|it|its|they|them|their|we|our|you|your|'
    r'i|me|my|he|she|his|her|him|who|what|when|where|why|how)$'
)


def extract_keywords(text: str, top_n: int = 10) -> List[Dict]:
    words = WORD_RE.findall(text.lower())
    words = [w for w in words if not STOP_WORDS.match(w)]
    freq  = Counter(words)
    total = sum(freq.values())
    scored = [(word, count / total, count) for word, count in freq.most_common(top_n * 2)]
    scored = [(w, tf * (1 + len(w) * 0.05), c) for w, tf, c in scored]
    scored.sort(key=lambda x: x[1], reverse=True)
    return [{'keyword': w, 'score': round(s, 4), 'count': c} for w, s, c in scored[:top_n]]


def compute_tfidf(documents: List[str], top_n: int = 5) -> Dict[str, List[Dict]]:
    doc_words  = [set(WORD_RE.findall(d.lower())) for d in documents]
    N          = len(documents)
    results    = {}
    for i, text in enumerate(documents):
        words = WORD_RE.findall(text.lower())
        words = [w for w in words if not STOP_WORDS.match(w)]
        tf    = Counter(words)
        total = sum(tf.values())
        scores = []
        for word, count in tf.most_common(top_n * 3):
            df    = sum(1 for dw in doc_words if word in dw)
            idf   = math.log(N / (df + 1)) + 1
            tfidf = (count / total) * idf
            scores.append({'word': word, 'tfidf': round(tfidf, 4), 'tf': count, 'df': df})
        scores.sort(key=lambda x: x['tfidf'], reverse=True)
        results[f'doc{i+1}'] = scores[:top_n]
    return results


documents = [
    "Python is a powerful programming language used for machine learning and data science applications.",
    "Regex patterns are essential tools for text processing and string manipulation in Python programming.",
    "Machine learning models require large datasets and computational resources for training and inference.",
    "Text mining extracts valuable information from unstructured documents using natural language processing.",
]

print(f"\n   Single-document keyword extraction:")
kws = extract_keywords(documents[0])
print(f"   From: \"{documents[0][:50]}...\"")
print(f"   {'Keyword':<20} {'Score':<8} {'Count'}")
print(f"   {'-'*20} {'-'*8} {'-'*5}")
for kw in kws[:6]:
    print(f"   {kw['keyword']:<20} {kw['score']:<8} {kw['count']}")

print(f"\n   TF-IDF across {len(documents)} documents:")
tfidf_results = compute_tfidf(documents)
for doc_id, keywords in tfidf_results.items():
    print(f"\n   {doc_id}: {[k['word'] for k in keywords[:4]]}")
```

---

## 61.5 Text Classification by Regex Rules

```python
import re
from typing import Dict, List, Tuple

print("\nRule-based Text Classification:")
print("=" * 60)

TOPIC_RULES = {
    'technology': re.compile(
        r'\b(?:software|hardware|API|code|programming|python|database|'
        r'server|cloud|AI|machine\s+learning|neural|algorithm|framework|'
        r'developer|frontend|backend|devops|kubernetes|docker)\b',
        re.IGNORECASE
    ),
    'finance': re.compile(
        r'\b(?:stock|market|invest|portfolio|dividend|revenue|profit|'
        r'loss|asset|liability|equity|bond|fund|interest|rate|budget|'
        r'fiscal|quarterly|earnings|valuation|IPO|crypto)\b',
        re.IGNORECASE
    ),
    'health': re.compile(
        r'\b(?:medical|doctor|patient|hospital|treatment|medicine|drug|'
        r'disease|symptom|diagnosis|surgery|therapy|clinical|health|'
        r'vitamin|diet|exercise|mental|wellness|vaccine)\b',
        re.IGNORECASE
    ),
    'sports': re.compile(
        r'\b(?:player|team|game|match|score|goal|tournament|championship|'
        r'athlete|coach|season|league|win|loss|stadium|referee|penalty)\b',
        re.IGNORECASE
    ),
    'politics': re.compile(
        r'\b(?:government|president|minister|parliament|election|vote|'
        r'policy|political|party|democrat|republican|law|legislation|'
        r'senate|congress|treaty|diplomat|cabinet|administration)\b',
        re.IGNORECASE
    ),
}


def classify_text(text: str, top_n: int = 2) -> List[Tuple[str, int]]:
    scores = {}
    for topic, pattern in TOPIC_RULES.items():
        matches = pattern.findall(text)
        if matches:
            scores[topic] = len(matches)
    return sorted(scores.items(), key=lambda x: x[1], reverse=True)[:top_n]


test_texts = [
    "The Federal Reserve raised interest rates affecting stock market valuations and crypto investments.",
    "Python developers are building machine learning models using TensorFlow and deploying to Kubernetes clusters.",
    "The medical team diagnosed the patient and prescribed a new treatment involving vitamin therapy.",
    "The championship game saw the home team score three goals in the second half of the tournament.",
    "The president signed new legislation after parliament voted on the healthcare policy reform.",
    "The startup's API uses Python backend and Vue.js frontend, deployed on AWS cloud infrastructure.",
]

print(f"\n   {'Text (truncated)':<55} {'Top topics'}")
print(f"   {'-'*55} {'-'*30}")
for text in test_texts:
    topics = classify_text(text)
    label  = ', '.join(f"{t}({c})" for t, c in topics)
    short  = text[:52] + '...' if len(text) > 55 else text
    print(f"   {short:<55} {label}")
```

---

## 61.6 สรุป Part 61

```
Text Mining & NLP with Regex:

1. Tokenization:
   Words:     \b[a-zA-Z]+\b
   Sentences: sentence boundary detection
   N-grams:   sliding window over token list

2. Named Entity Recognition:
   Dates:  Month DD, YYYY | DD/MM/YYYY | YYYY-MM-DD
   Money:  (USD|$|฿)\s*[\d,]+(\.\d{2})?
   Orgs:   [A-Z][a-zA-Z&\s]+(Co|Corp|Inc|Ltd|LLC)
   Loc:    (?:in|at)\s+([A-Z][a-z]+ [A-Z][a-z]+)

3. Sentiment:
   Positive/negative word lists (compile once)
   Negation detection flips sentiment
   Intensifiers amplify score

4. Keyword extraction:
   Remove stop words (pre-compiled pattern)
   TF = count / total_words
   TF-IDF = TF x log(N/DF+1)+1

5. Rule-based classification:
   One pattern per topic with alternation
   Score = match count
   Sort by score → top categories
```

---

*[← Part 60: Database & SQL Patterns](part-60-database-sql.md) | [→ Part 62: Code Analysis & Parsing](part-62-code-analysis.md)*
