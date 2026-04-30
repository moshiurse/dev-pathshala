# 🔍 Regular Expressions (Regex) — Engineer Deep Dive

> **Regex একটি ছোট্ট, ঘন ভাষা যা pattern matching-এ অসাধারণ powerful — কিন্তু একটু বেপরোয়া হলে catastrophic backtracking, ReDoS attack, বা বছরের পুরনো bug ডেকে আনে।**
> এই গাইডে আমরা regex-এর theory, syntax, performance pitfall, language-specific idiom এবং বাংলাদেশী phone number, NID, bKash transaction ID-এর pattern নিয়ে practical example দেখব।

---

## 📖 সূচিপত্র

- [Foundations: Regular Languages, NFA, DFA](#-foundations)
- [Engine Types: Backtracking vs DFA](#-engine-types)
- [POSIX BRE/ERE vs PCRE](#-posix-bre-ere-vs-pcre)
- [Syntax Tour](#-syntax-tour)
- [Common Patterns (BD Specific)](#-common-patterns)
- [Catastrophic Backtracking & ReDoS](#-catastrophic-backtracking--redos)
- [Per-Language Idioms](#-per-language-idioms)
- [Tooling](#-tooling)
- [When NOT to Use Regex](#-when-not-to-use-regex)
- [Real-World BD Examples](#-real-world-bd-examples)
- [Anti-patterns ও Checklist](#-anti-patterns-ও-checklist)

---

## 🧮 Foundations

### Regular Languages

Computer science-এ একটি language তখনই **regular** যখন সেটা কোনো regular expression বা finite automaton (DFA/NFA) দিয়ে describe করা যায়। Regular expression Stephen Kleene আবিষ্কার করেন (১৯৫০-এর দশক)।

**নিয়মিত languages-এর সীমাবদ্ধতা:**
- নেস্টেড balance match (যেমন matching `(((...)))`) regex-এ সম্ভব **না** — এটা context-free language।
- HTML/XML nested tag → regex দিয়ে parse করা অসম্ভব। (নিচে detail)

### Finite Automata

```
Regex:  a(b|c)*d

NFA visualization:

        ┌──── ε ────┐
        ▼           │
  ──► (S) ─a─► (1) ─b─► (1)  ─d─► ((F))   accept
                  └─c─► (1)─┘
```

**DFA (Deterministic FA)**: প্রতিটি state-এ প্রতিটি input symbol-এর জন্য exactly one transition।
**NFA (Nondeterministic FA)**: একই input-এ multiple transition বা ε-transition থাকতে পারে।

**Theory**: যেকোনো regex → NFA → DFA। DFA simulation O(n) input length-এ।

---

## ⚙️ Engine Types

### Backtracking Engines (NFA + recursion)

- **PCRE** (Perl, PHP `preg_*`, ridge cases for nginx, Apache)
- **Python `re`**, JavaScript, Java, .NET, Ruby
- Features: lookaround, backreferences, possessive quantifier, conditional
- Worst case: **exponential** time (catastrophic backtracking)

### DFA / Non-backtracking Engines

- **RE2** (Google) — used in Go `regexp`, Cloudflare WAF (post-2019), `grep -E`
- Features: subset of PCRE — **no backreferences, no lookaround** (intentional, for safety)
- Worst case: **linear** time, guaranteed
- Cloudflare 2019 outage caused by PCRE catastrophic backtracking — তারপর RE2 adopt

### Comparison

| | Backtracking (PCRE) | DFA (RE2) |
|-|---------------------|-----------|
| Lookaround | ✅ | ❌ |
| Backreferences | ✅ | ❌ |
| Worst case | exponential | linear |
| Memory | low | higher (NFA→DFA exp blowup possible) |
| Default in | most languages | Go, RE2-using systems |
| Untrusted input | ⚠️ ReDoS risk | safe |

**Rule of thumb**: User-supplied regex (search functionality) → RE2 ব্যবহার করুন। Hard-coded internal regex → PCRE ঠিক আছে যদি pattern test করা থাকে।

---

## 📚 POSIX BRE, ERE vs PCRE

| Feature | BRE | ERE | PCRE |
|---------|-----|-----|------|
| `+ ? \|` | escape needed | literal | literal |
| `()` group | `\( \)` | literal | literal |
| `{n,m}` | `\{ \}` | literal | literal |
| `\d \w \s` | ❌ | ❌ | ✅ |
| Lookaround | ❌ | ❌ | ✅ |
| Backreferences | ✅ `\1` | ✅ | ✅ |
| Examples | `grep`, `sed` | `grep -E`, `egrep` | Perl, PHP, JS-like |

```bash
# BRE — () literal, group needs escape
echo 'foo123' | grep '\(foo\)\([0-9]*\)'

# ERE — natural
echo 'foo123' | grep -E '(foo)([0-9]*)'

# PCRE — only with -P flag
echo 'foo123' | grep -P '(foo)(\d+)'
```

---

## 🧬 Syntax Tour

### Literals & Metacharacters

```
. ─ যেকোনো একটি char (newline বাদে; s flag-এ newline ও)
\ ─ escape

Metachar: . * + ? ^ $ ( ) [ ] { } | \
```

```
"১০০ টাকা"  match: \S+\s+\S+
"price=100" match: price=\d+
"a.b.c"     match (literal): a\.b\.c
```

### Character Classes

```
[abc]          a, b, বা c
[a-z]          lowercase
[A-Za-z0-9_]   = \w
[^abc]         a, b, c ছাড়া যেকোনো
\d             digit (0-9)         \D = non-digit
\w             word char           \W = non-word
\s             whitespace          \S = non-whitespace
\b             word boundary       \B = non-boundary

POSIX (PCRE):
[:alpha:] [:alnum:] [:digit:] [:space:] [:upper:] [:lower:]
[:xdigit:] [:punct:] [:cntrl:] [:print:]

# ব্যবহার: [[:alpha:]]+ (double bracket)
```

### Anchors

```
^   line/string start
$   line/string end
\A  string start (m flag-এ ও)
\Z  string end (trailing newline-এর আগে)
\z  absolute string end
\b  word boundary (\w↔\W)
\B  non-boundary
```

### Quantifiers

```
*       0+ (greedy)        *?    0+ (lazy)        *+    0+ (possessive)
+       1+                 +?    1+ lazy          ++    possessive
?       0 or 1             ??    lazy             ?+    possessive
{n}     exactly n
{n,}    n or more
{n,m}   n to m
```

**Greedy vs Lazy:**

```
String: "<b>Hello</b><i>World</i>"
<.+>     greedy → matches whole "<b>Hello</b><i>World</i>"
<.+?>    lazy   → matches "<b>"
```

**Possessive (PCRE):** কোনো backtrack হবে না — fast-fail।

```
\d++       1+ digit, no backtrack
```

### Groups

```
(abc)            capturing group, accessible as \1
(?:abc)          non-capturing
(?<name>abc)     named (PCRE: (?P<name>...) Python ও PHP)
\1, \2           backreferences (in pattern)
$1, $2           in replacement (or \1)

(?>abc)          atomic group — equivalent to no-backtrack
```

### Alternation

```
cat|dog|fish     match any of three
(yes|no)         grouped alternation
```

### Lookaround (zero-width assertions)

```
(?=...)    positive lookahead — পরে এটা থাকতে হবে
(?!...)    negative lookahead — পরে এটা থাকা যাবে না
(?<=...)   positive lookbehind — আগে এটা থাকতে হবে
(?<!...)   negative lookbehind — আগে এটা থাকা যাবে না
```

```
\d+(?=\s+টাকা)        digit followed by " টাকা" (টাকা match-এ আসবে না)
(?<=USD )\d+         "USD " এর পরের digit
\b(?!password)\w+    "password" নয় এমন word
```

### Atomic & Conditional (PCRE)

```
(?>X)              atomic — X-এ কোনো backtrack না
(?(1)yes|no)       conditional — group 1 match করলে yes else no
(?(<name>)yes|no)
```

### Flags

```
i  case-insensitive
m  multiline (^ $ matches each line)
s  dotall (. matches newline)
u  Unicode (PHP, JS, Python)
x  extended (whitespace + # comment)
g  global (JS, sed; PHP-এ default for replace_all)
y  sticky (JS) — match শুরু হবে exactly lastIndex থেকে
d  hasIndices (JS, indices for groups)
```

```javascript
const r = /\d+/giu;            // global, case-insens, unicode
```

```php
preg_match('/^\d+$/u', $input);   // u for UTF-8
```

### Unicode Property Escapes

```
\p{L}                  Letter
\p{N}                  Number
\p{Letter}             same as \p{L}
\p{Script=Bengali}     বাংলা script
\p{Script=Latin}
\p{Script=Arabic}
\p{Lu}                 Uppercase letter
\p{Mark}               diacritic mark

\P{L}                  not a letter
```

```javascript
"১০০ টাকা".match(/\p{Script=Bengali}+/gu);
// → ["টাকা"]   (১০০ ও Bengali script-এ আছে! check Unicode block 0980-09FF)

"১০০".match(/\p{Decimal_Number}/gu);
// → ["১", "০", "০"]
```

```php
preg_match_all('/\p{Bengali}+/u', 'Hello বাংলা World ঢাকা', $m);
// $m[0] = ['বাংলা', 'ঢাকা']
```

---

## 🛠️ Common Patterns

### Email — কাল্পনিক "complete" pattern পরিহার করুন

RFC 5322 pure regex 6000+ char. Practical:

```
^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$
```

**সবচেয়ে ভালো approach**: regex দিয়ে শুধু `contains @`-এর basic check করুন, তারপর confirmation email পাঠান। Real validation = email পৌঁছানো।

### URL Extraction

```
https?://[^\s<>"']+
```

PHP example:

```php
preg_match_all('#https?://[^\s<>"\']+#u', $text, $m);
```

(URL-এর শেষে dot/comma trim করা ভালো — `rtrim($u, '.,!?;:')`)

### Bangladesh Phone Number

বাংলাদেশী mobile pattern:
- Operator prefix: 013 (Grameenphone), 014 (Banglalink early), 015 (Teletalk), 016 (Airtel/Robi), 017 (GP), 018 (Robi/Airtel), 019 (Banglalink)
- Total 11 digit (`01X-XXXXXXXX`)
- Country code: `+880` বা `880`

```
^(?:\+?88)?01[3-9]\d{8}$
```

```php
function isValidBdMobile(string $n): bool {
    $n = preg_replace('/[\s\-()]/', '', $n);
    return (bool) preg_match('/^(?:\+?88)?01[3-9]\d{8}$/', $n);
}

// বিভিন্ন format আনুমোদিত করতে চাইলে normalize:
function normalizeBdMobile(string $n): ?string {
    $n = preg_replace('/[^\d+]/', '', $n);
    if (preg_match('/^(?:\+?880)?(1[3-9]\d{8})$/', $n, $m)) {
        return '01' . substr($m[1], 1);   // canonical: 01XXXXXXXXX
    }
    return null;
}
```

```javascript
// JS — separators allowed
const BD_MOBILE = /^\+?(?:880[\s-]?)?0?1[3-9](?:[\s-]?\d){8}$/;

function isBdMobile(s) {
  return BD_MOBILE.test(s.replace(/[\s\-()]/g, ''));
}
```

### NID (National ID)

বাংলাদেশী NID:
- পুরোনো: 13 বা 17 digit
- নতুন (Smart Card 2016+): 10 digit

```
^(\d{10}|\d{13}|\d{17})$
```

```php
function isValidBdNid(string $n): bool {
    $n = preg_replace('/\s/', '', $n);
    return (bool) preg_match('/^(\d{10}|\d{13}|\d{17})$/', $n);
}
```

⚠️ Regex শুধু format check; checksum/validity আলাদা — Election Commission API দিয়ে।

### bKash Transaction ID

bKash TrxID — 10 character alphanumeric uppercase:

```
^[0-9A-Z]{10}$
```

```php
function isBkashTrxId(string $id): bool {
    return (bool) preg_match('/^[0-9A-Z]{10}$/', $id);
}
```

(Pattern আরো নির্দিষ্ট হতে পারে — প্রায়ই capital letter + digit মিশ্রণ থাকে)

### TIN (Tax Identification Number)

12 digit:

```
^\d{12}$
```

### Postal Code (Bangladesh)

4 digit:

```
^\d{4}$
```

### Bangla Text Detection

Unicode block U+0980-U+09FF:

```
[\u0980-\u09FF]+
```

```javascript
function hasBengali(s) {
  return /[\u0980-\u09FF]/.test(s);
}

function isMixedScript(s) {
  return /[\u0980-\u09FF]/.test(s) && /[A-Za-z]/.test(s);
}
```

```php
preg_match('/[\x{0980}-\x{09FF}]+/u', $text);
// অথবা
preg_match('/\p{Bengali}+/u', $text);
```

### IP Address

#### IPv4 (strict)

```
^((25[0-5]|2[0-4]\d|1\d{2}|[1-9]?\d)\.){3}(25[0-5]|2[0-4]\d|1\d{2}|[1-9]?\d)$
```

```php
// অনেক সহজ: PHP filter
filter_var($ip, FILTER_VALIDATE_IP, FILTER_FLAG_IPV4);
```

#### IPv6

জটিল (8 groups, `::` shortcut, embedded IPv4)। Regex না — library use করুন:

```javascript
// Node.js
import net from 'node:net';
net.isIPv6('2001:db8::1');  // boolean

// অথবা npm ip-regex
import ipRegex from 'ip-regex';
const v6 = ipRegex.v6({ exact: true });
```

### Credit Card Detection (PCI Redaction)

```
\b(?:\d[ -]*?){13,19}\b
```

Issuer-specific:

```
Visa:        ^4\d{12}(\d{3})?$
Mastercard:  ^(5[1-5]\d{14}|2(2[2-9]\d|[3-6]\d{2}|7[01]\d|720)\d{12})$
Amex:        ^3[47]\d{13}$
```

Real validation: **Luhn algorithm**, regex নয়।

```php
function luhn(string $num): bool {
    $num = preg_replace('/\D/', '', $num);
    $sum = 0; $alt = false;
    for ($i = strlen($num) - 1; $i >= 0; $i--) {
        $n = (int) $num[$i];
        if ($alt) { $n *= 2; if ($n > 9) $n -= 9; }
        $sum += $n;
        $alt = !$alt;
    }
    return $sum % 10 === 0;
}
```

### Date/Time

```
ISO 8601 (basic):  ^\d{4}-\d{2}-\d{2}(T\d{2}:\d{2}:\d{2}(\.\d+)?(Z|[+-]\d{2}:\d{2})?)?$
```

⚠️ `2024-02-30` regex pass হয়! Date library use করুন (`Carbon`, `date-fns`, `dayjs`)।

### Password Strength (Multiple Lookaheads)

```
^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]{8,}$
```

- কমপক্ষে ১ lowercase
- কমপক্ষে ১ uppercase
- কমপক্ষে ১ digit
- কমপক্ষে ১ special char
- দৈর্ঘ্য ≥ 8

```javascript
const STRONG = /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]{8,}$/;

if (!STRONG.test(pwd)) {
  return 'পাসওয়ার্ড দুর্বল: ৮+ char, upper+lower+digit+special লাগবে';
}
```

⚠️ NIST 2017 guideline-এ এসব strict rule discourage করা হয়েছে; দৈর্ঘ্য + breach-list check (`pwned-passwords` API) ভালো।

### HTML Tag Stripping

```
<[^>]*>
```

```php
$clean = preg_replace('/<[^>]*>/', '', $html);
// অথবা better:
$clean = strip_tags($html);
```

⚠️ **HTML regex দিয়ে parse করবেন না!** [Famous SO answer](https://stackoverflow.com/a/1732454) — Cthulhu আসবে। DOM parser:

```php
$doc = new DOMDocument();
$doc->loadHTML($html);
```

```javascript
const dom = new DOMParser().parseFromString(html, 'text/html');
```

### Log Parsing (Logstash Grok)

Grok = predefined regex pattern + name।

```
%{IPORHOST:client} - - \[%{HTTPDATE:timestamp}\] "%{WORD:method} %{URIPATHPARAM:uri} HTTP/%{NUMBER:httpversion}" %{NUMBER:status} %{NUMBER:bytes}
```

দেখুন: `16-observability/elk-efk-stack.md`।

---

## 💥 Catastrophic Backtracking & ReDoS

### একটা Famous Example

```
Pattern: ^(a+)+$
Input:   "aaaaaaaaaaaaaaaaaa!"   (no match)

NFA তে:
- (a+) inside outer + → অজস্র way to split "aaaa" between inner and outer
- "!" শেষে fail করায় engine সব split try করে
- Time complexity: 2^n
- 30 a-চিহ্ন → কয়েক সেকেন্ড
- 40 → কয়েক মিনিট
- 50 → ঘন্টা
```

### Cloudflare 2019 Outage Story

19 জুলাই, 2019 — Cloudflare একটা WAF rule push করল:

```
(?:(?:\"|'|\]|\}|\\|\d|(?:nan|infinity|true|false|null|undefined|symbol|math)|\`|\-|\+)+[)]*;?((?:\s|-|~|!|{}|\|\||\+)*.*(?:.*=.*)))
```

এই pattern catastrophic backtracking-এ পড়ে → CPU 100% → 30 মিনিটের global outage। আজ Cloudflare RE2 ব্যবহার করে।

### ReDoS Attack

Attacker user input দিয়ে catastrophic regex trigger করে → server DoS।

**Vulnerable**: signup form-এ email validation `(.+)+@`। Attacker `aaaa....!` paste করে।

### Detection

```bash
# Node — safe-regex
npm i -g safe-regex
safe-regex '^(a+)+$'    # → false (unsafe)

# JavaScript — vuln-regex-detector
npm i vuln-regex-detector

# Python — repat
pip install repat
```

### Defense

#### ১. Possessive Quantifier / Atomic Group (PCRE)

```
^(?>a+)+$       atomic, no backtrack
^a++$           possessive
```

দুটোই linear time। `^(a+)+$` এর equivalent।

#### ২. Anchor + Specific Class

```
Bad:  ^(.+)+$
Good: ^[a-z]+$    (specific char class, no nested quant)
```

#### ৩. Timeout

```javascript
// Node-এ timeout
const re2 = require('re2');           // RE2 binding
const r = new re2(pattern);

// অথবা manual timeout
function timedMatch(re, str, ms = 100) {
  return new Promise((resolve, reject) => {
    const t = setTimeout(() => reject(new Error('regex timeout')), ms);
    try { resolve(re.test(str)); } finally { clearTimeout(t); }
  });
}
```

```python
import regex   # PyPI 'regex' (not stdlib 're')
regex.match(p, s, timeout=0.1)
```

```php
// PHP — backtrack limit
ini_set('pcre.backtrack_limit', 100000);
$result = preg_match($pattern, $input);
if ($result === false && preg_last_error() === PREG_BACKTRACK_LIMIT_ERROR) {
    // ReDoS attempted
}
```

#### ৪. RE2 for Untrusted Input

```javascript
const RE2 = require('re2');
const r = new RE2(userPattern);     // throws on incompatible
r.test(input);                       // linear time guaranteed
```

```go
// Go — re2 default
import "regexp"
r := regexp.MustCompile(`^[a-z]+$`)  // safe always
```

---

## 🌐 Per-Language Idioms

### PHP (PCRE)

```php
// Match
if (preg_match('/^01[3-9]\d{8}$/', $phone)) { ... }

// Match with capture
if (preg_match('/order-(\d+)/', $line, $m)) {
    $orderId = $m[1];
}

// Match all
preg_match_all('/\d+/', '1 2 3', $m);    // $m[0] = ['1','2','3']

// Replace
$cleaned = preg_replace('/\s+/', ' ', $input);

// Replace with callback
$result = preg_replace_callback(
    '/\b(\w+)\b/',
    fn($m) => strtoupper($m[1]),
    'hello world'
);

// Split
$parts = preg_split('/[\s,]+/', 'a, b  c,d');

// Named captures
preg_match('/(?<area>\d{3})-(?<num>\d{7})/', $s, $m);
echo $m['area'];

// UTF-8 — u modifier MUST
preg_match('/\p{Bengali}+/u', $text);

// Delimiters: any non-alnum non-backslash
preg_match('#https?://example\.com/path#', $url);
preg_match('~^\w+~', $s);
```

**PHP gotchas:**
- `pcre.backtrack_limit` (default 1M) ও `pcre.recursion_limit` (default 100K) — exceed হলে `false` return।
- UTF-8 হলে `u` flag বাধ্যতামূলক (নাহলে multi-byte char ভাঙবে)।
- `preg_quote($literal)` — user input regex-এ embed করার আগে।

### JavaScript / Node.js

```javascript
// Literal vs constructor
const r1 = /^\d+$/i;
const r2 = new RegExp('^\\d+$', 'i');     // double escape!

// match / matchAll
'abc 123 def 456'.match(/\d+/g);          // ['123', '456']
const all = [...'abc 123 def 456'.matchAll(/(\d+)/g)];
// each: ['123','123', index:4, ...]

// replace / replaceAll
'foo bar foo'.replace(/foo/g, 'baz');
'foo bar foo'.replaceAll('foo', 'baz');   // no regex needed for literal

// Replace with function
'a1 b2 c3'.replace(/(\w)(\d)/g, (_, l, d) => `${l.toUpperCase()}${+d * 2}`);

// Named captures
const m = 'order-123'.match(/order-(?<id>\d+)/);
console.log(m.groups.id);

// Sticky y flag — must start at lastIndex
const sticky = /\d+/y;
sticky.lastIndex = 4;
sticky.exec('abc 123');         // null (lastIndex 4 is space)

// d flag — match indices
const r = /(?<area>\d{3})-(\d{4})/d;
const m2 = '555-1234'.match(r);
console.log(m2.indices.groups.area);   // [0, 3]
```

**JS gotchas:**
- **`lastIndex` pitfall**: `/x/g.test()` consecutive call-এ different result (lastIndex change হয়)।

```javascript
const r = /foo/g;
r.test('foo');        // true,  lastIndex = 3
r.test('foo');        // false, lastIndex = 3, no match from 3
```

→ Stateless usage-এ flag-হীন regex বা প্রতিবার নতুন regex বানান।

- `.replace` first arg string হলে only first match replace; regex /g দিতে হবে।
- ES2018+: lookbehind `(?<=...)`, named groups, Unicode property `\p{...}` (with `u` flag)।

#### Node Libraries

```javascript
// validator.js — battle-tested
import validator from 'validator';
validator.isEmail('a@b.com');
validator.isMobilePhone('+8801712345678', 'bn-BD');

// ow — runtime check
import ow from 'ow';
ow(phone, ow.string.matches(/^01[3-9]\d{8}$/));
```

### Go (RE2)

```go
import "regexp"

var bdMobile = regexp.MustCompile(`^(?:\+?88)?01[3-9]\d{8}$`)

func IsBdMobile(s string) bool { return bdMobile.MatchString(s) }

// Named capture
r := regexp.MustCompile(`order-(?P<id>\d+)`)
m := r.FindStringSubmatch("order-123")
fmt.Println(m[r.SubexpIndex("id")])

// Replace
out := r.ReplaceAllString(input, "ORDER-${id}")

// FindAll
all := regexp.MustCompile(`\d+`).FindAllString("a1 b2 c3", -1)
```

**Go limitations**: RE2-only — no lookaround, no backreferences। Try `(?<=...)` → compile error। Workaround: split into multiple regex বা manually parse।

### Python

```python
import re
import regex                # third-party, richer

re.match(r'^\d+$', s)       # only at start
re.search(r'\d+', s)        # anywhere
re.fullmatch(r'\d+', s)     # whole string

# Named
m = re.match(r'(?P<id>\d+)', '123')
m.group('id')

# Compile reusable
PHONE = re.compile(r'^01[3-9]\d{8}$')
PHONE.match(s)

# replace with function
re.sub(r'\d+', lambda m: str(int(m.group()) * 2), 'a1 b2')

# Verbose mode (x flag)
P = re.compile(r"""
    ^(\+?88)?       # optional country code
    01[3-9]         # operator prefix
    \d{8}$          # 8 more digits
""", re.VERBOSE)
```

`regex` library: variable-length lookbehind, partial matching, fuzzy matching, timeout।

```python
import regex
regex.match(r'(?<=foo|bar)\d+', 'foo123', timeout=0.1)
```

### MySQL / Postgres

#### MySQL

```sql
SELECT * FROM users WHERE phone REGEXP '^01[3-9][0-9]{8}$';
SELECT REGEXP_REPLACE(name, '[^A-Za-z0-9]', '');
SELECT REGEXP_LIKE(email, '@example\\.com$');
```

⚠️ MySQL regex doesn't use indexes → slow on large tables। Full-text index বা generated column ভালো।

#### PostgreSQL (POSIX-flavored)

```sql
SELECT * FROM users WHERE phone ~ '^01[3-9][0-9]{8}$';   -- case-sens
SELECT * FROM users WHERE email ~* 'gmail';              -- case-insens
SELECT regexp_replace(name, '\s+', ' ', 'g');
SELECT regexp_matches(text, '\d+', 'g');
```

### grep / sed / awk

```bash
# grep
grep -E '^01[3-9][0-9]{8}$' phones.txt          # ERE
grep -P '^\d{11}$' phones.txt                   # PCRE (GNU only)
grep -o '\b\w*Foo\w*\b' file                    # only matched part
grep -v 'pattern' file                          # invert
grep -c 'ERROR' app.log                         # count

# sed
sed -E 's/[[:space:]]+/ /g' file
sed -nE '/^ERROR/p' app.log
sed -i.bak -E 's/old/new/g' file                # in-place + backup

# awk
awk '/^01[3-9]/ { print $1 }' file
awk -F, '{ if ($2 ~ /^[0-9]+$/) print }' csv
```

---

## 🔧 Tooling

### Online Debuggers

- **regex101.com** — best-in-class explanation, multi-flavor (PCRE, JS, Python, Go), ReDoS warning, profiling
- **regexr.com** — interactive, less detailed
- **debuggex.com** — visualize NFA railroad diagram
- **regexper.com** — railroad diagram only

### Editor Regex

- **VS Code**: Ctrl+F → `.*` icon, JS-flavored
- **IntelliJ/JetBrains**: Java-flavored, full lookaround support
- **Sublime/Vim**: PCRE-like (with Vim's quirks: `\(...\)` for group)

### CLI Test

```bash
# pcregrep
echo "01712345678" | pcregrep '^01[3-9]\d{8}$'

# grep -P
echo "01712345678" | grep -P '^01[3-9]\d{8}$'
```

### AST-based Alternatives

কখন regex enough নয়?
- Parsing-এ recursion (HTML, JSON, code)
- Context-sensitive (indent, scope)

বরং:
- **PEG / Parser combinators** (`peg.js`, `nearley`, PHP `phplrt/parser`)
- **ANTLR** — grammar → parser
- **Tree-sitter** — incremental parsing
- DOM/JSON-specific parser

---

## 🚫 When NOT to Use Regex

### ১. HTML / XML Parsing

```php
// ❌ এই 'simple' কাজও নষ্ট হবে
preg_match('/<a href="(.*?)">/', $html, $m);
// — `<a class="x" href="...">` ধরবে না
// — attribute order vary
// — comments, CDATA, mismatched quotes

// ✅ DOM
$doc = new DOMDocument();
@$doc->loadHTML($html);
$xpath = new DOMXPath($doc);
foreach ($xpath->query('//a/@href') as $href) { ... }
```

### ২. JSON

`json_decode` / `JSON.parse` use করুন। Regex দিয়ে JSON parse pure madness।

### ৩. Recursively nested

```
( ( a + b ) * c )       — balanced paren count
```

PCRE recursion (`(?R)`) দিয়ে possible কিন্তু complex। Better: stack-based parser।

### ৪. `string.includes()` যথেষ্ট

```javascript
// ❌
if (/foo/.test(s)) ...

// ✅
if (s.includes('foo')) ...
```

### ৫. "Real" Format Validation

| Format | Library |
|--------|---------|
| Email | send confirmation |
| URL | language built-in URL parser |
| IP | `filter_var`, `net.isIP` |
| Phone | `libphonenumber` (Google) |
| Date | `Carbon`, `date-fns`, `dayjs` |
| Credit card | Luhn + BIN range |
| UUID | language built-in |

```javascript
// libphonenumber-js — BD-aware
import { parsePhoneNumber } from 'libphonenumber-js';
const n = parsePhoneNumber('01712345678', 'BD');
n.isValid();         // true
n.format('E.164');   // +8801712345678
```

---

## 🇧🇩 Real-World BD Examples

### ১. Daraz Product Page Scrape — DON'T regex HTML

```python
# ❌
re.search(r'<h1[^>]*>(.+?)</h1>', html)

# ✅
from bs4 import BeautifulSoup
soup = BeautifulSoup(html, 'lxml')
title = soup.select_one('h1.pdp-mod-product-badge-title').text
```

### ২. Pathao OTP Extraction (Android SMS)

```kotlin
// SMS body: "Your Pathao OTP is 123456. Valid for 5 min."
val PATHAO_OTP = Regex("""Pathao[\s\S]*?(\d{4,6})""")

fun extract(sms: String): String? =
    PATHAO_OTP.find(sms)?.groupValues?.get(1)
```

```javascript
// React Native
const SMS_OTP = /\b(\d{4,6})\b/;
const otp = sms.match(SMS_OTP)?.[1];
```

### ৩. bKash Transaction ID Validate

```php
class BkashValidator {
    private const TRX_ID = '/^[0-9A-Z]{10}$/';

    public static function isValidTrxId(string $id): bool {
        return (bool) preg_match(self::TRX_ID, strtoupper(trim($id)));
    }
}
```

### ৪. Foodpanda Menu CSV Parsing — DON'T use regex

```php
// ❌
preg_split('/,/', $line);   // ভাঙবে যদি field-এ comma থাকে

// ✅
$rows = [];
if (($f = fopen($csv, 'r')) !== false) {
    while (($row = fgetcsv($f, 0, ',', '"', '\\')) !== false) {
        $rows[] = $row;
    }
}
```

### ৫. SSLCommerz Webhook Signature

Body থেকে fields parse করতে — regex নয়, key=value parser।

```php
parse_str($rawBody, $data);       // built-in
```

### ৬. ELK / Logstash Grok for Pathao API Logs

```
%{TIMESTAMP_ISO8601:ts} %{LOGLEVEL:lvl} \[%{DATA:reqid}\] %{IP:client} "%{WORD:method} %{URIPATH:path}" %{NUMBER:status:int} %{NUMBER:duration_ms:float}ms
```

### ৭. Bangla Profanity Filter

```php
$BAD_BN = ['গালি১', 'গালি২'];
$pattern = '/\b(?:' . implode('|', array_map('preg_quote', $BAD_BN)) . ')\b/u';
$clean = preg_replace($pattern, '****', $userComment);
```

⚠️ Word-boundary `\b` Bangla script-এ properly কাজ নাও করতে পারে — `\p{Bengali}` ব্যবহার করুন।

---

## ⚠️ Anti-patterns

❌ **HTML/XML/JSON regex দিয়ে parse** → use parser।
❌ **Greedy `.*` everywhere** — use `.*?` বা specific char class।
❌ **`^(a+)+$` ধরনের nested quant** → catastrophic backtracking।
❌ **User input directly regex-এ embed** → injection + ReDoS।

```javascript
// ❌
new RegExp(userInput);
// ✅
new RegExp(escapeRegex(userInput));
function escapeRegex(s) { return s.replace(/[.*+?^${}()|[\]\\]/g, '\\$&'); }
```

❌ **Email/URL/Phone-এ "perfect" regex** → বাইরের library use করুন।
❌ **No timeout / backtrack-limit** server-side untrusted input-এ।
❌ **PHP UTF-8 ছাড়া Bangla-তে regex** → `u` modifier missing।
❌ **JS `lastIndex` ignore** → `g`-flag re-use bug।
❌ **PCRE-only feature Go-তে expect** (lookaround) → compile error।
❌ **Verbose pattern comment-হীন** — `x` flag use করুন large pattern-এ।
❌ **Regex দিয়ে security validation** (SQL/XSS prevention) → use prepared statement, output encoding।
❌ **Same regex বার বার compile** → cache (`re.compile`, `preg_match` PHP কিছুটা auto cache, JS literal once)।
❌ **`\w` দিয়ে Bangla word match expect** → `\p{Bengali}`/`[\u0980-\u09FF]` lagbe।

---

## ✅ Senior Engineer Regex Checklist

**Design:**
- [ ] Pattern সঠিক engine-এ test করেছেন (PCRE vs RE2 vs JS)
- [ ] Anchored (`^`/`$`) — যেখানে দরকার
- [ ] Specific char class, generic `.*` কম
- [ ] Lookbehind variable-length জানেন (PCRE/Python `regex` only)
- [ ] Named capture, `(?:...)` non-capturing ব্যবহার

**Safety:**
- [ ] ReDoS-safe (safe-regex/vuln-regex tested)
- [ ] User input untrusted → RE2 / timeout / backtrack limit
- [ ] User regex input → escape (`preg_quote` / `escapeRegex`)
- [ ] Atomic group / possessive quantifier যেখানে দরকার

**Unicode:**
- [ ] PHP `u` flag, JS `u` flag
- [ ] `\p{Bengali}` Bangla text-এ
- [ ] `[\u0980-\u09FF]` fallback if Unicode property unsupported

**Practice:**
- [ ] HTML/XML/JSON: parser (DOMDocument, BS4, json_decode)
- [ ] Email/Phone/URL: library (libphonenumber, validator.js)
- [ ] Date: date library, regex নয়
- [ ] CSV: csv library, regex নয়

**Performance:**
- [ ] Hot path-এ pre-compiled (`preg_match`, `re.compile`, `regexp.MustCompile`)
- [ ] DB query-এ regex avoid; index/full-text use
- [ ] Large file: stream, not slurp

**Testing:**
- [ ] regex101.com / unit test দিয়ে edge cases
- [ ] Negative cases (should NOT match)
- [ ] Performance test on 10x normal input length
- [ ] Empty string, very long string, malformed input

**BD Specific:**
- [ ] Phone: `^(\+?88)?01[3-9]\d{8}$`
- [ ] NID: 10/13/17 digit handle
- [ ] Bangla: `\p{Bengali}` / `[\u0980-\u09FF]`
- [ ] PCRE `u` flag for UTF-8

---

> **মনে রাখুন**: Regex একটি ছোট্ট, ঘন language যা বিশাল লিভারেজ দেয় — কিন্তু একই সাথে একটি **footgun**। নিয়ম: pattern-কে যতটা সম্ভব **specific** এবং **anchored** রাখুন; user input untrusted হলে **RE2 / timeout** ব্যবহার করুন; HTML/JSON/email/phone/date-এর জন্য **proper library** ই সঠিক উত্তর। regex101.com আপনার বন্ধু — pattern লিখার আগে তাতে test করুন। বাংলাদেশী context-এ phone, NID, bKash TrxID এবং বাংলা script handling — এই গুলোর জন্য সঠিক pattern + UTF-8 awareness অপরিহার্য।
