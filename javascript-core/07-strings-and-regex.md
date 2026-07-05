# 22 — Strings and Regular Expressions

> **"Strings are everywhere in frontend code — user input, API responses, URLs, IDs, error messages. Regex feels like a foreign language at first, but the patterns you'll use 90% of the time fit on a single card. Know those patterns, and you stop reaching for verbose loops every time you need to validate an email or extract a number."**

🟢 **Level: Beginner → Intermediate**

---

## 📚 Table of Contents

1. [String Fundamentals](#1-string-fundamentals)
2. [Template Literals](#2-template-literals)
3. [String Methods Reference](#3-string-methods-reference)
4. [Regular Expressions — Syntax](#4-regular-expressions--syntax)
5. [Regex in JavaScript](#5-regex-in-javascript)
6. [Named Capture Groups](#6-named-capture-groups)
7. [Common Regex Patterns](#7-common-regex-patterns)
8. [Common Mistakes](#8-common-mistakes)
9. [Exercises](#9-exercises)

---

## 1. String Fundamentals

```javascript
// Strings are immutable sequences of UTF-16 code units
const str = "hello";
str[0] = "H"; // silently fails — strings are immutable
str.length; // 5

// Accessing characters
str[0]; // 'h'
str.charAt(0); // 'h'
str.at(-1); // 'o' — modern: negative index from end

// Strings are iterable
for (const char of "hello") {
  console.log(char); // 'h', 'e', 'l', 'l', 'o'
}

// Unicode — strings are UTF-16, multi-byte characters take 2 "code units"
"😀".length; // 2 (emoji is 2 UTF-16 code units)
[..."😀"].length; // 1 (spreading iterates by code POINT, not unit)

// String comparison (lexicographic, case-sensitive)
"abc" < "abd"; // true (c < d)
"Z" < "a"; // true — uppercase letters sort BEFORE lowercase in ASCII
"A".localeCompare("a"); // depends on locale settings
```

---

## 2. Template Literals

```javascript
const name = "Alice";
const age = 30;

// Basic interpolation
`Hello, ${name}!`; // "Hello, Alice!"
`In 10 years: ${age + 10}`; // "In 10 years: 40"

// Multi-line strings (newlines preserved)
const html = `
  <div>
    <h1>${name}</h1>
    <p>Age: ${age}</p>
  </div>
`.trim();

// Expressions (can be any JS expression)
`${Math.PI.toFixed(2)}`; // "3.14"
`${user.isAdmin ? "Admin" : "User"}`;
`${items.map((i) => `<li>${i}</li>`).join("\n")}`;

// Tagged template literals (advanced — function called with parsed template)
function highlight(strings, ...values) {
  return strings.reduce(
    (result, str, i) =>
      `${result}${str}${values[i] !== undefined ? `<mark>${values[i]}</mark>` : ""}`,
    "",
  );
}
highlight`Hello, ${name}! You are ${age} years old.`;
// "Hello, <mark>Alice</mark>! You are <mark>30</mark> years old."

// Raw strings (escapes not processed)
String.raw`\n is a newline`; // "\\n is a newline" (backslash n, not actual newline)
```

---

## 3. String Methods Reference

```javascript
const str = "  Hello, World!  ";

// SEARCHING
str.includes("World"); // true
str.startsWith("  Hello"); // true
str.endsWith("!  "); // true
str.indexOf("l"); // 4 (first occurrence)
str.lastIndexOf("l"); // 10 (last occurrence)
str.search(/world/i); // 9 (regex search, returns index or -1)

// EXTRACTING
str.slice(7, 12); // 'World'
str.slice(-7, -2); // 'World' (negative: from end)
str.substring(7, 12); // 'World' (no negative indices)
str.at(7); // 'W'

// MODIFYING (all return NEW strings — strings are immutable)
str.trim(); // 'Hello, World!' (removes leading/trailing whitespace)
str.trimStart(); // 'Hello, World!  '
str.trimEnd(); // '  Hello, World!'
str.toLowerCase(); // '  hello, world!  '
str.toUpperCase(); // '  HELLO, WORLD!  '
str.replace("World", "JS"); // replaces FIRST occurrence
str.replaceAll("l", "L"); // replaces ALL occurrences
str.padStart(20, "*"); // '***  Hello, World!  '
str.padEnd(20, "-"); // '  Hello, World!  ---'
str.repeat(2); // '  Hello, World!    Hello, World!  '

// SPLITTING AND JOINING
"a,b,c".split(","); // ['a', 'b', 'c']
"hello".split(""); // ['h', 'e', 'l', 'l', 'o']
"hello".split("", 3); // ['h', 'e', 'l'] (limit)
["a", "b", "c"].join(" - "); // 'a - b - c'

// MATCHING (returns matches)
"hello world".match(/\w+/g); // ['hello', 'world']
"abc123def456".matchAll(/\d+/g); // iterator of all matches with groups

// NUMBER ↔ STRING
String(42); // '42'
(42).toString(); // '42'
(42).toString(2); // '101010' (binary)
(255).toString(16); // 'ff' (hex)
Number("42"); // 42
parseInt("42px", 10); // 42 (stops at non-numeric)
parseFloat("3.14xyz"); // 3.14
```

---

## 4. Regular Expressions — Syntax

```javascript
// Create a regex
const re1 = /pattern/flags; // literal (preferred)
const re2 = new RegExp('pattern', 'flags'); // constructor (for dynamic patterns)

// FLAGS:
// g  — global: find ALL matches (not just first)
// i  — case-insensitive
// m  — multiline: ^ and $ match start/end of each LINE
// s  — dotAll: . matches \n too
// u  — unicode: full Unicode support
// d  — hasIndices: capture group index info (ES2022)

// CHARACTER CLASSES
/[abc]/     // matches 'a', 'b', or 'c'
/[a-z]/     // matches any lowercase letter
/[A-Za-z0-9]/ // alphanumeric
/[^abc]/    // NOT a, b, or c (negation)

// SHORTHAND CHARACTER CLASSES
/\d/   // digit: [0-9]
/\D/   // non-digit: [^0-9]
/\w/   // word char: [A-Za-z0-9_]
/\W/   // non-word
/\s/   // whitespace (space, tab, newline)
/\S/   // non-whitespace
/./    // any character EXCEPT newline (with s flag: includes newline)

// ANCHORS
/^hello/   // start of string (or line with m flag)
/world$/   // end of string (or line with m flag)
/\bhello\b/ // word boundary — matches 'hello' but not 'hello123'

// QUANTIFIERS
/a*/    // 0 or more
/a+/    // 1 or more
/a?/    // 0 or 1 (optional)
/a{3}/  // exactly 3
/a{2,4}/ // 2 to 4
/a{2,}/  // 2 or more

// GREEDY vs LAZY
/<.+>/    // greedy: matches as MUCH as possible
/<.+?>/   // lazy: matches as LITTLE as possible

// GROUPS
/(abc)/      // capturing group — captured in match result
/(?:abc)/    // non-capturing group — grouped but not captured
/(abc|def)/  // alternation: abc OR def

// LOOKAHEAD / LOOKBEHIND
/\d+(?= dollars)/ // positive lookahead: digits followed by " dollars"
/\d+(?! dollars)/ // negative lookahead
/(?<=\$)\d+/      // positive lookbehind: digits preceded by "$"
/(?<!\$)\d+/      // negative lookbehind
```

---

## 5. Regex in JavaScript

```javascript
// test(): returns boolean
/^\d+$/.test("12345"); // true
/^\d+$/.test("123a5"); // false

// exec(): returns first match with details (or null)
const match = /(\d{4})-(\d{2})-(\d{2})/.exec("2024-01-15");
// match[0] = '2024-01-15' (full match)
// match[1] = '2024', match[2] = '01', match[3] = '15'
// match.index = 0 (position in string)

// match(): with g flag returns ALL matches (no capture groups)
"hello world hello".match(/hello/g); // ['hello', 'hello']
// without g flag: same as exec()

// matchAll(): returns iterator of ALL matches WITH capture groups
const str = "2024-01-15 and 2024-03-20";
for (const m of str.matchAll(/(\d{4})-(\d{2})-(\d{2})/g)) {
  console.log(m[1], m[2], m[3]); // year, month, day for each date
}

// replace() with regex
"hello WORLD".replace(/[A-Z]+/g, (m) => m.toLowerCase()); // 'hello world'
"2024-01-15".replace(/(\d{4})-(\d{2})-(\d{2})/, "$3/$2/$1"); // '15/01/2024'
// $1, $2... reference captured groups in the replacement string

// replace() with function
"hello world".replace(/\b\w/g, (char) => char.toUpperCase()); // 'Hello World'

// split() with regex
"one1two2three".split(/\d/); // ['one', 'two', 'three']
"a  b   c".split(/\s+/); // ['a', 'b', 'c']
```

---

## 6. Named Capture Groups

```javascript
// Named groups: (?<name>pattern) — cleaner than positional indices
const dateRegex = /(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/;
const result = dateRegex.exec("2024-01-15");
const { year, month, day } = result.groups;
// year = '2024', month = '01', day = '15'

// In replace():
"2024-01-15".replace(
  /(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/,
  "$<day>/$<month>/$<year>",
); // '15/01/2024'

// In matchAll():
for (const { groups } of "2024-01-15 2024-03-20".matchAll(
  /(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/g,
)) {
  console.log(groups.year, groups.month, groups.day);
}
```

---

## 7. Common Regex Patterns

```javascript
// Email (simplified — real email validation is complex; use a library for production)
/^[^\s@]+@[^\s@]+\.[^\s@]+$/

// URL
/^https?:\/\/(www\.)?[-a-zA-Z0-9@:%._+~#=]{1,256}\.[a-zA-Z]{2,}\b/

// Phone (US format: (123) 456-7890 or 123-456-7890)
/^\(?\d{3}\)?[-.\s]\d{3}[-.\s]\d{4}$/

// Only digits
/^\d+$/

// Integer (positive or negative)
/^-?\d+$/

// Decimal number
/^-?\d+(\.\d+)?$/

// Date: YYYY-MM-DD
/^\d{4}-(0[1-9]|1[0-2])-(0[1-9]|[12]\d|3[01])$/

// Hex color
/^#([A-Fa-f0-9]{6}|[A-Fa-f0-9]{3})$/

// IPv4
/^(\d{1,3}\.){3}\d{1,3}$/

// Slug (URL-friendly string)
/^[a-z0-9]+(?:-[a-z0-9]+)*$/

// Password: min 8 chars, at least one uppercase, lowercase, digit
/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d).{8,}$/

// Strip HTML tags
str.replace(/<[^>]*>/g, '');

// Capitalize first letter of each word
str.replace(/\b\w/g, c => c.toUpperCase());

// Remove extra whitespace
str.replace(/\s+/g, ' ').trim();

// Extract all numbers from a string
str.match(/\d+(\.\d+)?/g)?.map(Number) ?? [];
```

---

## 8. Common Mistakes

### Mistake 1 — Forgetting the `g` flag for replace-all

```javascript
// ❌ Only replaces the FIRST occurrence without g flag
"aababab".replace(/a/, "X"); // 'Xababab'

// ✅ With g flag replaces all
"aababab".replace(/a/g, "X"); // 'XXbXbXb'

// ✅ Or use replaceAll() (string method, no regex needed for literals)
"aababab".replaceAll("a", "X"); // 'XXbXbXb'
```

### Mistake 2 — Reusing a stateful regex with `g` flag

```javascript
// ❌ Stateful regex: exec/test remembers lastIndex when g flag is used
const re = /\d+/g;
re.test("abc123"); // true  — lastIndex is now 6
re.test("abc123"); // false — starts searching at index 6!

// ✅ Re-create the regex each time, or reset lastIndex manually
re.lastIndex = 0;
re.test("abc123"); // true
```

### Mistake 3 — `split` on empty string for Unicode

```javascript
// ❌ Splits by UTF-16 code units — breaks emoji
"hello😀".split(""); // ['h','e','l','l','o','\uD83D','\uDE00'] — emoji split in two!

// ✅ Use spread or Array.from for Unicode-safe splitting
[..."hello😀"]; // ['h','e','l','l','o','😀']
```

---

## 9. Exercises

### Exercise 1 — Parse a query string

```javascript
// Given '?name=Alice&age=30&city=NY', parse it to { name: 'Alice', age: '30', city: 'NY' }
function parseQueryString(qs) {
  /* ... */
}
```

<details>
<summary>Solution</summary>

```javascript
function parseQueryString(qs) {
  return Object.fromEntries(
    qs
      .replace(/^\?/, "")
      .split("&")
      .map((pair) => pair.split("=").map(decodeURIComponent)),
  );
}
// Or simply:
function parseQueryString(qs) {
  return Object.fromEntries(new URLSearchParams(qs));
}
```

</details>

---

## 🔗 Related Topics

- [`16-variables-and-data-types.md`](./16-variables-and-data-types.md) — Strings as primitives
- [`24-es6-modern-syntax.md`](./24-es6-modern-syntax.md) — Template literal tagged templates
