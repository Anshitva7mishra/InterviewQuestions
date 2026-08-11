# 11 — Regular Expressions Deep Dive
### 10 Questions | Intermediate

---

**Q1. What are regular expression flags and what does each one do?** — Easy

**Answer:**
Flags modify how a regex engine processes a pattern. They are appended after the closing `/` or passed as the second argument to `new RegExp()`.

```js
// Syntax:
/pattern/flags
new RegExp("pattern", "flags")

// All flags:
const s = "Hello\nWorld";

/hello/i.test(s);          // i — case-insensitive: true
/hello/gi;                 // g — global: find ALL matches (not just first)
/^world/m.test(s);         // m — multiline: ^ and $ match start/end of each LINE
/hello.world/s.test(s);    // s — dotAll: . matches \n (dot matches ALL characters)
/\p{Letter}/u.test("é");   // u — unicode: enables \p{} unicode properties, full unicode mode
/\p{Letter}/v.test("é");   // v — unicode sets (ES2024): superset of u, enables set operations
/(?<name>...)/d;           // d — indices: match object includes .indices (start/end positions)
/pattern/y;                // y — sticky: match must start at lastIndex, doesn't advance beyond it
```

Flag combinations:
```js
// Find all words case-insensitively across multiple lines:
const text = "Hello world\nHELLO WORLD";
const matches = text.match(/hello/gi); // ["Hello", "HELLO"]

// dotAll — . normally doesn't match \n:
/Hello.World/.test("Hello\nWorld");  // false (. skips \n)
/Hello.World/s.test("Hello\nWorld"); // true (dotAll)

// Sticky vs global:
const re = /\d+/y;
re.lastIndex = 3;
"abc123".match(re); // ["123"] — starts exactly at index 3

const reg = /\d+/g;
"abc123def456".match(reg); // ["123", "456"] — finds all
```

**Follow-up:** What is the difference between `g` (global) and `y` (sticky)?

`g` searches forward from `lastIndex` but keeps scanning until a match is found anywhere after that position. `y` requires that the match start exactly at `lastIndex` — if there is no match at that exact position, it fails immediately. Sticky is used for tokenizers where you want to parse input left-to-right without gaps.

**GOTCHA:** The `g` flag makes `RegExp.prototype.test()` stateful — it advances `lastIndex` on every successful match. Calling `regex.test()` in a loop with a `g` regex can produce alternating true/false results. Use `String.prototype.match()` or reset `lastIndex` to 0 between uses, or avoid `g` when you only need one match.

---

**Q2. What are capturing groups, non-capturing groups, and named capturing groups?** — Medium

**Answer:**
Groups organize parts of a pattern and control what gets captured into matches.

```js
// Capturing group (...) — saves the matched text:
const date = "2024-03-15";
const match = date.match(/(\d{4})-(\d{2})-(\d{2})/);
// match[0] = "2024-03-15" (full match)
// match[1] = "2024"       (group 1)
// match[2] = "03"         (group 2)
// match[3] = "15"         (group 3)

// Non-capturing group (?:...) — groups without capturing:
const re = /(?:https?|ftp):\/\/([\w.]+)/;
const url = "https://example.com".match(re);
// url[1] = "example.com" — only domain is captured (protocol skipped)

// Named capturing group (?<name>...) — ES2018:
const { groups } = "2024-03-15".match(/(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/);
console.log(groups.year);  // "2024"
console.log(groups.month); // "03"
console.log(groups.day);   // "15"

// Named groups in replace:
const swapped = "John Smith".replace(
  /(?<first>\w+)\s(?<last>\w+)/,
  "$<last>, $<first>"
);
// "Smith, John"

// Backreference — match the same text captured earlier:
// Match repeated words:
/\b(\w+)\s+\1\b/.test("the the");  // true — \1 references group 1
// Named backreference:
/(?<word>\w+)\s+\k<word>/.test("hello hello"); // true
```

**Follow-up:** When would you use a non-capturing group?

When you need grouping for alternation or quantifiers but don't need to extract the matched content. Non-capturing groups have less overhead (no allocation of a capture slot) and make the match array cleaner.

**GOTCHA:** Nested groups are numbered by order of their opening parenthesis. `/(a(b))(c)/` — group 1 is `(a(b))`, group 2 is `(b)`, group 3 is `(c)`. If you reorganize groups, all your numeric references break — another reason to prefer named groups.

---

**Q3. What are lookahead and lookbehind assertions?** — Hard

**Answer:**
Lookaround assertions match a position based on what comes before or after, without consuming the characters.

```js
// Lookahead (?=...) — "followed by" (positive):
// Match digits only if followed by "px":
"20px 30em 40px".match(/\d+(?=px)/g);   // ["20", "40"]

// Negative lookahead (?!...) — "NOT followed by":
"20px 30em 40px".match(/\d+(?!px)\b/g); // ["30"]

// Lookbehind (?<=...) — "preceded by" (positive) — ES2018:
// Match digits only if preceded by "$":
"$100 £200 $300".match(/(?<=\$)\d+/g);  // ["100", "300"]

// Negative lookbehind (?<!...):
"$100 £200 $300".match(/(?<!\$)\d+/g);  // ["200"]

// Practical example — password validation:
function validatePassword(pwd) {
  const hasUpper = /(?=.*[A-Z])/.test(pwd);       // lookahead for uppercase
  const hasLower = /(?=.*[a-z])/.test(pwd);       // lookahead for lowercase
  const hasDigit = /(?=.*\d)/.test(pwd);          // lookahead for digit
  const hasSpecial = /(?=.*[!@#$%^&*])/.test(pwd); // lookahead for special char
  const minLength = pwd.length >= 8;
  return hasUpper && hasLower && hasDigit && hasSpecial && minLength;
}

// Multiple lookaheads combined in one regex:
/^(?=.*[A-Z])(?=.*[a-z])(?=.*\d)(?=.*[!@#$%^&*]).{8,}$/
  .test("SecureP@ss1"); // true
```

Difference between capturing and lookaround:
```js
// Without lookahead — consumes characters:
"100px".match(/\d+px/);  // ["100px"] — "px" is consumed

// With lookahead — does NOT consume "px":
"100px".match(/\d+(?=px)/);  // ["100"] — only digits matched
// Next match can still see "px"
```

**GOTCHA:** Lookbehind is not supported in some older environments (Safari < 16.4). Variable-length lookbehind (e.g., `(?<=a+)`) is supported in modern engines but was not part of the original ES2018 spec — check compatibility. Unlike lookahead, lookbehind matches right-to-left internally, which can cause surprising behavior with backreferences inside them.

---

**Q4. What is the difference between greedy, lazy, and possessive quantifiers?** — Medium

**Answer:**
Quantifiers control how many times a pattern element repeats. The strategy for matching determines backtracking behavior.

```js
const html = "<div><span>text</span></div>";

// Greedy quantifier (*) — match as much as possible, then backtrack:
html.match(/<.+>/)[0];   // "<div><span>text</span></div>" — greedy grabs EVERYTHING

// Lazy quantifier (*?) — match as little as possible:
html.match(/<.+?>/)[0];  // "<div>" — stops at first >

// More examples:
"aXXXb".match(/a.*b/)[0];   // "aXXXb" — greedy
"aXXXb".match(/a.*?b/)[0];  // "aXXXb" — same here (no shorter match)

// Where lazy matters:
const str = "cat sat mat";
str.match(/[a-z]at/g);   // ["cat", "sat", "mat"] — greedy but single char anyway
str.match(/.*at/g);      // ["cat sat mat"] — greedy takes all
str.match(/.*?at/g);     // ["cat", " sat", " mat", ""] — lazy takes minimal

// All quantifiers have lazy versions:
// *  → *?   (zero or more, lazy)
// +  → +?   (one or more, lazy)
// ?  → ??   (zero or one, lazy)
// {n,m} → {n,m}?  (range, lazy)

// Possessive quantifiers (*+, ++, ?+) — greedy but NO backtracking (JS does NOT support these)
// Atomic groups (?>...) — similar concept, also not in JS native regex
// For no-backtrack in JS, use — workaround with careful pattern design
```

**Follow-up:** When does greedy vs lazy matter for performance?

Catastrophic backtracking can occur with nested greedy quantifiers on ambiguous patterns — e.g., `/(a+)+$/.test("aaaaaaab")`. The engine tries exponentially many combinations. Lazy quantifiers don't inherently prevent this. The solution is to rewrite ambiguous patterns, use atomic groups (via third-party libs), or simplify the structure.

**GOTCHA:** `.*` in a greedy regex will first consume the entire remaining string, then backtrack. For large inputs with complex patterns, this can be very slow. Use lazy `.*?` or more specific character classes (`[^>]*`) to limit scope.

---

**Q5. What are the key `String` and `RegExp` methods for working with regex?** — Easy

**Answer:**
```js
const str = "Hello World hello world";
const re = /hello/gi;

// String methods that accept regex:

// .match() — returns array of matches; with g: all matches; without g: first match + groups
str.match(/hello/i);   // ["Hello", index: 0, input: "...", groups: undefined]
str.match(/hello/gi);  // ["Hello", "hello"] — global returns only matched strings

// .matchAll() — returns iterator of ALL match objects (requires g flag):
const iter = str.matchAll(/hello/gi);
for (const m of iter) {
  console.log(m[0], m.index); // "Hello" 0, then "hello" 12
}

// .search() — returns index of first match or -1:
str.search(/world/i);  // 6

// .replace() — replace first (without g) or all (with g):
str.replace(/hello/gi, "Hi");   // "Hi World Hi world"

// .replaceAll() — ES2021, replaces all (regex must have g):
str.replaceAll(/hello/gi, "Hi"); // same

// .split() — split on regex:
"one1two2three".split(/\d/);   // ["one", "two", "three"]
"one1two2three".split(/(\d)/); // ["one", "1", "two", "2", "three"] — captures included

// RegExp methods:

// .test() — returns boolean:
/hello/i.test("Hello World"); // true

// .exec() — returns single match object (stateful with g flag):
const globalRe = /\d+/g;
const text = "abc 123 def 456";
let m;
while ((m = globalRe.exec(text)) !== null) {
  console.log(`Found ${m[0]} at index ${m.index}`);
}
// "Found 123 at index 4"
// "Found 456 at index 12"
```

**GOTCHA:** `String.prototype.match()` with a `g` flag returns ONLY the matched strings — no index, no groups. To get full match objects for every match, use `String.prototype.matchAll()` (ES2020), which returns an iterator of exec-style objects. This is a very common source of confusion when someone tries to get group captures from a global match.

---

**Q6. What are Unicode property escapes and when do you need them?** — Hard

**Answer:**
Unicode property escapes (`\p{...}` and `\P{...}`) match characters based on their Unicode properties — character class, script, or category. They require the `u` or `v` flag.

```js
// \p{Property} — matches characters WITH that property
// \P{Property} — matches characters WITHOUT that property

// Match any letter in any language:
/\p{Letter}/u.test("é");    // true
/\p{Letter}/u.test("α");    // true (Greek alpha)
/\p{Letter}/u.test("1");    // false

// Match decimal digits (all unicode digit chars, not just 0-9):
/\p{Decimal_Number}/u.test("٣"); // true (Arabic digit 3)

// Match emoji:
/\p{Emoji}/u.test("😀");  // true
/\p{Emoji}/u.test("A");   // false

// Script-based matching:
/\p{Script=Cyrillic}/u.test("Привет"); // true
/\p{Script=Latin}/u.test("Hello");     // true
/\p{Script=Han}/u.test("你好");        // true — Chinese characters

// General categories:
/\p{Uppercase_Letter}/u.test("A");   // true
/\p{Lowercase_Letter}/u.test("a");   // true
/\p{Punctuation}/u.test("!");        // true

// Practical: validate that a string contains only letters (any language):
function isAllLetters(str) {
  return /^\p{Letter}+$/u.test(str);
}
isAllLetters("hello");   // true
isAllLetters("héllo");   // true  — accented letters OK
isAllLetters("hello1");  // false — digit
isAllLetters("こんにちは"); // true — Japanese hiragana

// Without u flag, \p is treated as literal "p" or an error
```

The `v` flag (ES2024) adds set operations:
```js
// Set intersection [A&&B] — characters in both A and B:
/[\p{Letter}&&\p{ASCII}]/v.test("a"); // true — ASCII letter
/[\p{Letter}&&\p{ASCII}]/v.test("é"); // false — non-ASCII letter

// Set subtraction [A--B] — A minus B:
/[\p{Letter}--[a-z]]/v.test("A"); // true — uppercase letters only
```

**GOTCHA:** Without the `u` or `v` flag, `\p{Letter}` is not a unicode escape — it may be treated as an identity escape or throw a syntax error depending on the engine. Always include `u` or `v` when using `\p{}`. Also, `\p{Emoji}` can be tricky because some emoji are sequences (joined by ZWJ), not single code points.

---

**Q7. What is catastrophic backtracking and how do you prevent it?** — Hard

**Answer:**
Catastrophic backtracking (also called ReDoS — Regular Expression Denial of Service) occurs when a regex engine explores an exponential number of paths to determine a non-match, causing extreme slowness.

```js
// Vulnerable pattern — nested quantifiers on overlapping matches:
const vulnerable = /(a+)+$/;
vulnerable.test("aaaaaaaaab"); // This can take seconds or minutes!

// Why: "aaaaaab" — engine tries to split "aaa" between outer group repetitions:
// (a)(a)(a)(a)(a)(a) — then "b" fails → backtrack
// (aa)(a)(a)(a)(a)   — then "b" fails → backtrack
// ... exponential combinations of how to group the 'a's

// Another vulnerable pattern:
/(a|aa)+$/.test("aaaaab"); // Same problem — 'a' and 'aa' overlap

// More patterns that look safe but aren't:
/^(\w+\s?)*$/.test("aaaaaaaaaaaaaaaaaaa!"); // catastrophic

// HOW TO PREVENT:

// 1. Eliminate ambiguity — make groups non-overlapping:
/a+$/.test("aaaaab"); // fast — no alternatives for 'a+'

// 2. Use specific character classes instead of generic ones:
// Instead of: /^(\w+\s?)*$/
// Use:        /^\w+(\s\w+)*$/  — each space forces a boundary, no overlap

// 3. Use atomic groups or possessive quantifiers (not native in JS):
// In some regex engines: (?>a+) won't backtrack
// In JS — simulate with workarounds or use a library like re2 (via npm)

// 4. Use linear-time regex engines (Google re2 via npm):
const RE2 = require("re2");
const safe = new RE2(/(a+)+$/); // re2 guarantees linear time

// 5. Set execution time limits:
// In Node.js (experimental):
const { Worker } = require("worker_threads");
// Run regex in a worker with a timeout

// 6. Validate user input before using it in regex:
// If users can supply patterns, never use user input directly in a RegExp
```

Detecting vulnerable patterns:
- Nested quantifiers: `(X+)+`, `(X*)+`, `(X|Y)+` where X and Y can match the same characters
- Overlapping alternatives with quantifiers: `(a|aa)+`
- Repeated groups with optional parts: `(a?a)+`

**GOTCHA:** ReDoS vulnerabilities in server-side JavaScript (Node.js) can cause denial of service if user input is matched against a vulnerable regex. The `validator.js` library has had historical ReDoS CVEs. Always audit regex patterns used with untrusted input.

---

**Q8. How does `String.prototype.replaceAll` differ from `replace` with a global flag?** — Medium

**Answer:**
`replaceAll()` was added in ES2021 to provide an explicit way to replace all occurrences. Both approaches work, but there are key differences.

```js
const text = "the cat sat on the mat";

// .replace() with global regex — replaces all:
text.replace(/the/g, "a"); // "a cat sat on a mat"

// .replaceAll() with string — replaces all occurrences of the literal string:
text.replaceAll("the", "a"); // "a cat sat on a mat"
// Equivalent to: text.replace(/the/g, "a")

// KEY DIFFERENCE — replaceAll with string literal vs regex:
// .replace("the", ...) — only replaces FIRST occurrence:
text.replace("the", "a");    // "a cat sat on the mat" — only first!
// .replaceAll("the", ...) — replaces ALL:
text.replaceAll("the", "a"); // "a cat sat on a mat"

// replaceAll with regex REQUIRES the g flag:
text.replaceAll(/the/, "a");  // TypeError: String.prototype.replaceAll called with a non-global RegExp argument
text.replaceAll(/the/g, "a"); // OK

// Replace with function:
text.replaceAll(/\w+/g, word => word.toUpperCase());
// "THE CAT SAT ON THE MAT"

// replaceAll is useful when you have a dynamic string (not a regex):
const search = "special.chars[here]";
// Using string with replace would need RegExp escaping:
const escaped = search.replace(/[.*+?^${}()|[\]\\]/g, "\\$&");
text.replace(new RegExp(escaped, "g"), "X");
// With replaceAll — no escaping needed for literal string:
text.replaceAll(search, "X");
```

**Follow-up:** When would you prefer `replaceAll()` over `replace()` with `/g`?

Use `replaceAll()` when replacing a literal string (no pattern matching needed) — it is clearer in intent, doesn't require regex syntax escaping, and can't accidentally introduce regex metacharacter bugs. Use `replace()` with `/g` when you need pattern matching with regex features.

**GOTCHA:** `replaceAll()` with a regex requires the `g` flag — it throws a `TypeError` if you pass a non-global regex. This is intentional: a non-global regex with `replaceAll` would be semantically confusing (you asked to replace all, but gave a non-global pattern).

---

**Q9. What is the `d` (indices) flag and what does `.indices` provide?** — Hard

**Answer:**
The `d` flag (ES2022) adds an `.indices` property to match results, providing start and end positions for each captured group. This is useful for source map generation, text editors, and linters.

```js
// Without d flag — you know WHAT matched but not WHERE each group matched:
const m1 = "2024-03-15".match(/(\d{4})-(\d{2})-(\d{2})/);
m1.index;   // 0 — start of full match only
// No way to get start/end of individual groups

// With d flag:
const m2 = "2024-03-15".match(/(\d{4})-(\d{2})-(\d{2})/d);
m2.indices;
// [
//   [0, 10],  // full match: chars 0-9
//   [0, 4],   // group 1 (year): chars 0-3
//   [5, 7],   // group 2 (month): chars 5-6
//   [8, 10]   // group 3 (day): chars 8-9
// ]

// With named groups:
const m3 = "2024-03-15".match(/(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/d);
m3.indices.groups;
// { year: [0, 4], month: [5, 7], day: [8, 10] }

// Practical use — highlight a match in an editor:
function highlightMatch(text, pattern) {
  const m = text.match(new RegExp(pattern, "d"));
  if (!m) return text;
  const [start, end] = m.indices[0];
  return text.slice(0, start) + "**" + text.slice(start, end) + "**" + text.slice(end);
}
highlightMatch("Hello World", "World"); // "Hello **World**"
```

**GOTCHA:** The `d` flag adds overhead — the engine must track position information for each group. Don't use it unless you actually need position data. Also, `indices` entries for optional groups that didn't participate in the match are `undefined`, not `[NaN, NaN]`.

---

**Q10. Write a regex to validate email, URL, and IPv4 address. What are the limitations?** — Medium

**Answer:**
Regex can handle common formats but cannot validate semantic correctness (e.g., whether a domain actually exists).

```js
// Email — basic but practical (not RFC 5321 compliant):
const emailRe = /^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$/;
emailRe.test("user@example.com");       // true
emailRe.test("user.name+tag@sub.domain.co.uk"); // true
emailRe.test("notanemail");             // false
emailRe.test("@nodomain.com");          // false
// Limitation: Does not handle quoted local parts ("user name"@example.com)
// or IP-literal hosts (user@[192.168.1.1])

// URL — practical subset:
const urlRe = /^(https?:\/\/)([\w\-]+\.)+[\w\-]+(\/[\w\-./?%&=+#]*)?$/i;
urlRe.test("https://example.com");           // true
urlRe.test("http://sub.domain.co.uk/path");  // true
urlRe.test("ftp://example.com");             // false
// Better approach: use URL constructor for validation:
function isValidUrl(str) {
  try {
    new URL(str);
    return true;
  } catch {
    return false;
  }
}

// IPv4 — strict validation (0-255 per octet):
const ipv4Re = /^(?:(?:25[0-5]|2[0-4]\d|[01]?\d\d?)\.){3}(?:25[0-5]|2[0-4]\d|[01]?\d\d?)$/;
ipv4Re.test("192.168.1.1");   // true
ipv4Re.test("255.255.255.255"); // true
ipv4Re.test("256.0.0.1");     // false — 256 > 255
ipv4Re.test("192.168.1");     // false — only 3 octets

// Breaking down the IPv4 octet regex:
// 25[0-5]   → 250-255
// 2[0-4]\d  → 200-249
// [01]?\d\d? → 0-199 (with optional leading 0 or 1)
```

**Follow-up:** Why is a fully RFC-compliant email regex a bad idea?

The fully RFC 5321/5322 compliant email regex is thousands of characters long and allows patterns like `"user name"@example.com` and `user@[IPv6:2001:db8::1]` that are technically valid but never used in practice. For real applications, use a simple practical regex combined with a confirmation email to verify deliverability.

**GOTCHA:** Never use regex alone for URL validation — use the `URL` constructor (available in browsers and Node.js 10+). It correctly handles encoding, internationalized domain names (IDN), and edge cases. `new URL(str)` throws a `TypeError` on invalid URLs and is much more reliable than any regex.

---

*Next: [12-Error-Handling.md](./12-Error-Handling.md)*
