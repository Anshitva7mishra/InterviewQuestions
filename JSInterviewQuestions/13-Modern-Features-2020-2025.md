# 13 — Modern JS (ES2020–ES2025)
### 20 Questions | Intermediate

---

**Q1. What is optional chaining (`?.`) and nullish coalescing (`??`)?** — Easy

**Answer:**
Optional chaining (`?.`, ES2020) short-circuits property access, method calls, and subscript access when the left-hand side is `null` or `undefined`, returning `undefined` instead of throwing.

Nullish coalescing (`??`, ES2020) returns the right-hand side only when the left is `null` or `undefined` — unlike `||`, it does not trigger on falsy values like `0` or `""`.

```js
const user = null;

// Without optional chaining — throws:
user.address.city; // TypeError: Cannot read properties of null

// With optional chaining — safe:
user?.address?.city;     // undefined (no throw)
user?.greet();           // undefined (no throw — method call)
user?.["name"];          // undefined (no throw — computed access)

// In arrays:
const arr = null;
arr?.[0];                // undefined

// Chained with function call:
const config = { getTimeout: null };
config.getTimeout?.();   // undefined — won't throw if getTimeout is null

// Real-world pattern:
const street = response?.data?.user?.address?.street ?? "Address not provided";

// Nullish coalescing — vs OR operator:
const count1 = 0 || 10;   // 10 — WRONG! 0 is falsy, replaced with 10
const count2 = 0 ?? 10;   // 0  — CORRECT! 0 is not null/undefined

const name1 = "" || "Anonymous";  // "Anonymous" — falsy string replaced
const name2 = "" ?? "Anonymous";  // ""           — empty string is not null/undefined

// Nullish assignment operator (??=, ES2021):
user.name ??= "Anonymous"; // Assigns only if user.name is null or undefined
// Equivalent to: user.name = user.name ?? "Anonymous";

// AND assignment (&&=) and OR assignment (||=) — ES2021:
user.isAdmin &&= checkAdminRole(); // Only calls checkAdminRole if isAdmin is truthy
user.name ||= "Default";          // Assigns only if user.name is falsy
```

**Follow-up:** What is the difference between `?.` and the `&&` short-circuit pattern?

`user && user.address && user.address.city` — the `&&` pattern returns a falsy value (e.g., `0`, `""`, `false`) short-circuited from the chain, not necessarily `undefined`. `user?.address?.city` always returns `undefined` when short-circuited. The `?.` version is also much shorter and handles all falsy values consistently.

**GOTCHA:** Optional chaining does NOT protect against errors inside the expression — only `null`/`undefined` on the left side. `user?.getCity()` still throws if `getCity` exists but throws internally. Also, `?.` cannot appear on the left side of an assignment: `user?.name = "Alice"` is a SyntaxError.

---

**Q2. What is `Promise.allSettled()`, `Promise.any()`, and `Promise.race()`?** — Medium

**Answer:**
ES2020 added `Promise.allSettled()` and ES2021 added `Promise.any()` to complete the Promise combinator API.

```js
const pa = Promise.resolve("A");
const pb = Promise.reject(new Error("B failed"));
const pc = Promise.resolve("C");

// Promise.all — rejects if ANY rejects (all-or-nothing):
await Promise.all([pa, pb, pc]); // Rejects immediately with Error("B failed")

// Promise.allSettled (ES2020) — waits for ALL, never rejects:
const results = await Promise.allSettled([pa, pb, pc]);
// [
//   { status: "fulfilled", value: "A" },
//   { status: "rejected",  reason: Error("B failed") },
//   { status: "fulfilled", value: "C" }
// ]

// Promise.race — resolves/rejects with the FIRST settled promise:
const winner = await Promise.race([
  new Promise(resolve => setTimeout(() => resolve("slow"), 1000)),
  new Promise(resolve => setTimeout(() => resolve("fast"), 100))
]);
// "fast" — first to settle wins

// Use case — timeout pattern with race:
function withTimeout(promise, ms) {
  const timeout = new Promise((_, reject) =>
    setTimeout(() => reject(new Error(`Timed out after ${ms}ms`)), ms)
  );
  return Promise.race([promise, timeout]);
}

// Promise.any (ES2021) — resolves with FIRST fulfilled, rejects only if ALL reject:
const first = await Promise.any([
  Promise.reject(new Error("A")),
  Promise.resolve("B"),
  Promise.resolve("C")
]);
// "B" — first fulfilled wins (ignores rejections)

// If ALL reject:
await Promise.any([
  Promise.reject(new Error("A")),
  Promise.reject(new Error("B"))
]);
// Throws AggregateError containing [Error("A"), Error("B")]

// Use case — try multiple CDN endpoints, use whichever responds first:
const data = await Promise.any([
  fetch("https://cdn1.example.com/data"),
  fetch("https://cdn2.example.com/data"),
  fetch("https://cdn3.example.com/data")
]);
```

**GOTCHA:** `Promise.race()` resolves/rejects with the first settled promise — but the other promises continue executing! They are not cancelled. This matters if they have side effects. To avoid memory leaks or unintended side effects, implement cancellation via `AbortController` for fetch-based races.

---

**Q3. What are BigInt and when would you use it?** — Medium

**Answer:**
`BigInt` (ES2020) is a primitive type for arbitrarily large integers. Regular `number` uses 64-bit IEEE 754 doubles, which can only safely represent integers up to 2^53 − 1 (`Number.MAX_SAFE_INTEGER = 9007199254740991`).

```js
// The problem with large numbers:
9007199254740991 + 1; // 9007199254740992 — OK
9007199254740991 + 2; // 9007199254740992 — WRONG! Same result

// BigInt — arbitrary precision:
9007199254740991n + 1n; // 9007199254740992n — correct
9007199254740991n + 2n; // 9007199254740993n — correct

// Creating BigInt:
const big1 = 42n;                        // literal suffix
const big2 = BigInt(42);                 // constructor
const big3 = BigInt("9007199254740993"); // from string

// Operations:
5n + 3n;   // 8n
5n * 3n;   // 15n
5n / 3n;   // 1n — integer division (truncates)
5n % 3n;   // 2n
5n ** 3n;  // 125n

// Comparison — works across types:
1n === 1;   // false — different types (strict equality)
1n == 1;    // true  — coerces for abstract equality
1n < 2;     // true  — comparison coerces

// Cannot mix BigInt and Number in arithmetic:
1n + 1;     // TypeError: Cannot mix BigInt and other types

// Use cases:
// Cryptocurrency — wei amounts in ETH (10^18 gwei per ETH):
const wei = 1000000000000000000n; // 1 ETH in wei — safe with BigInt

// Database IDs from Postgres/MySQL that exceed MAX_SAFE_INTEGER:
const id = BigInt("18446744073709551615"); // MAX UINT64

// JSON doesn't support BigInt natively:
JSON.stringify({ id: 42n }); // TypeError: Do not know how to serialize a BigInt
// Workaround:
JSON.stringify({ id: 42n }, (key, value) =>
  typeof value === "bigint" ? value.toString() : value
);
```

**GOTCHA:** `BigInt` values cannot be used with `Math` methods — `Math.max(1n, 2n)` throws a TypeError. You must use comparison operators directly. Also, `typeof 42n === "bigint"` — it's its own type. Converting to/from regular numbers loses precision for very large values: `Number(9007199254740993n)` returns `9007199254740992` — the rounded float.

---

**Q4. What is `globalThis` and why was it introduced?** — Easy

**Answer:**
`globalThis` (ES2020) provides a consistent reference to the global object across all JavaScript environments. Before it, different environments had different names for the global object.

```js
// Before ES2020 — environment-specific:
// Browser: window, self, frames
// Node.js: global
// Web Workers: self
// Deno: globalThis (already supported)

// Common polyfill hack people used:
const globalObj = (typeof globalThis !== "undefined") ? globalThis
  : (typeof window !== "undefined") ? window
  : (typeof global !== "undefined") ? global
  : (typeof self !== "undefined") ? self
  : this; // last resort

// Now — just use globalThis everywhere:
globalThis.setTimeout;    // works in browser, Node, workers
globalThis.console.log;   // works everywhere

// Setting global variables (avoid this pattern, but it works):
globalThis.MY_CONFIG = { debug: true };
// Later:
MY_CONFIG.debug; // true — accessible as global

// Use case — cross-environment feature detection:
function isNode() {
  return typeof globalThis.process !== "undefined" &&
         typeof globalThis.process.versions?.node !== "undefined";
}

function isBrowser() {
  return typeof globalThis.window !== "undefined" &&
         typeof globalThis.document !== "undefined";
}
```

**GOTCHA:** Inside a module (ESM), `this` at the top level is `undefined`, not the global object. In non-strict mode scripts, `this` is `window` in browsers. `globalThis` is the safe, consistent choice regardless of mode. Do not mistake it for `window` — in Node.js, `globalThis !== window` (window doesn't exist).

---

**Q5. What are logical assignment operators (`&&=`, `||=`, `??=`) introduced in ES2021?** — Medium

**Answer:**
Logical assignment operators combine logical operations with assignment, providing short-circuit evaluation with assignment.

```js
// ||= — assign only if left side is falsy:
let a = 0;
a ||= 5;   // a = 5 (0 is falsy)
a ||= 10;  // a = 5 (5 is truthy — no assignment)
// Equivalent to: a = a || 5; or a || (a = 5);

// &&= — assign only if left side is truthy:
let b = 5;
b &&= 10;  // b = 10 (5 is truthy)
let c = 0;
c &&= 10;  // c = 0 (0 is falsy — no assignment)
// Equivalent to: b = b && 10; or b && (b = 10);

// ??= — assign only if left side is null or undefined:
let d = null;
d ??= "default"; // d = "default" (null triggers)
let e = 0;
e ??= "default"; // e = 0 (0 is not null/undefined — no assignment)
let f = "";
f ??= "default"; // f = "" (empty string is not null/undefined — no assignment)
// Equivalent to: d = d ?? "default"; or d ?? (d = "default");

// Practical patterns:
// Initialize cache entry only if missing:
cache.users ??= {};
cache.users[id] ??= await fetchUser(id);

// Enable feature if admin:
user.canEdit &&= hasPermission(user, "edit");

// Default configuration:
function initConfig(config) {
  config.timeout ??= 5000;
  config.retries ??= 3;
  config.debug ??= false;
  return config;
}

// Key distinction from regular assignment:
// Regular:  x = x || y   — always evaluates and assigns
// Logical:  x ||= y       — only assigns if x is falsy (no assignment when no-op)
// The difference matters for setters with side effects!
class Reactive {
  set value(v) { console.log("setter called!"); this._v = v; }
  get value() { return this._v; }
}
const r = new Reactive();
r._v = 5;
r.value = r.value || 10;  // "setter called!" even though 5 is truthy
r.value ||= 10;           // NO "setter called!" — short-circuits before assignment
```

**GOTCHA:** `||=` does NOT behave like `??=`. `user.name ||= "Anonymous"` replaces `""` (empty string) and `0` with the default. If those are valid values, use `??=` instead. This distinction mirrors the `||` vs `??` difference.

---

**Q6. What is `String.prototype.replaceAll()` (ES2021) and what problem does it solve?** — Easy

**Answer:**
`replaceAll()` explicitly replaces all occurrences of a string or regex, making the intent clear and avoiding the need for a global regex just to replace all occurrences of a simple string.

```js
const text = "the cat and the dog ate the cake";

// Before ES2021 — replace all needed a regex with /g:
text.replace(/the/g, "a");  // "a cat and a dog ate a cake"

// OR — split+join hack:
text.split("the").join("a");  // Same result but hacky

// ES2021 — explicit and readable:
text.replaceAll("the", "a"); // "a cat and a dog ate a cake"

// Works with strings (replaces all) and regex (must use g flag):
text.replaceAll(/the/g, "a"); // OK
text.replaceAll(/the/, "a");  // TypeError: replaceAll called with non-global RegExp

// Replacement function:
"a1b2c3".replaceAll(/\d/g, n => `[${n}]`); // "a[1]b[2]c[3]"

// Special replacement patterns (same as replace):
"hello world".replaceAll("o", "[$&]"); // "hell[o] w[o]rld" — $& inserts matched text
"a-b-c".replaceAll("-", " → ");       // "a → b → c"

// Useful for sanitizing special chars in dynamic strings:
function escapeHtml(str) {
  return str
    .replaceAll("&", "&amp;")
    .replaceAll("<", "&lt;")
    .replaceAll(">", "&gt;")
    .replaceAll('"', "&quot;")
    .replaceAll("'", "&#39;");
}
```

**GOTCHA:** If you pass a regex to `replaceAll` without the `g` flag, it throws `TypeError`. This is intentional — asking to "replace all" with a non-global regex is contradictory. Also, `replaceAll` with a string does NOT accept regex metacharacters — the string is treated literally. `"a.b.c".replaceAll(".", "-")` gives `"a-b-c"`, not `"---"` (unlike what a regex `.` would do).

---

**Q7. What is `Object.hasOwn()` (ES2022) and why is it better than `hasOwnProperty`?** — Medium

**Answer:**
`Object.hasOwn(obj, prop)` is a static method that safely checks if an object has a property as its own (not inherited). It was introduced to fix several foot-guns with `hasOwnProperty`.

```js
// The problems with hasOwnProperty:

// 1. Object.create(null) — has NO prototype, so no hasOwnProperty:
const pureMap = Object.create(null);
pureMap.key = "value";
pureMap.hasOwnProperty("key"); // TypeError: pureMap.hasOwnProperty is not a function

// 2. hasOwnProperty can be overridden by a property:
const evil = { hasOwnProperty: () => true };
evil.hasOwnProperty("anything"); // always true — the method is shadowed!

// 3. Verbose indirect call needed for safety:
Object.prototype.hasOwnProperty.call(obj, "key"); // works but verbose

// ES2022 — Object.hasOwn():
Object.hasOwn(pureMap, "key");  // true — works on null-prototype objects
Object.hasOwn(evil, "anything"); // false — not fooled by overriding

const obj = { a: 1 };
Object.hasOwn(obj, "a");              // true — own property
Object.hasOwn(obj, "toString");       // false — inherited from Object.prototype
Object.hasOwn(Object.create(obj), "a"); // false — 'a' is on prototype, not own

// With Object.defineProperty:
const obj2 = {};
Object.defineProperty(obj2, "hidden", { value: 1, enumerable: false });
Object.hasOwn(obj2, "hidden");     // true — own even if non-enumerable
"hidden" in obj2;                  // true
obj2.hasOwnProperty("hidden");    // true (when it's safe)

// In practice — replacing the old idiom:
// Old: if (Object.prototype.hasOwnProperty.call(obj, key))
// New: if (Object.hasOwn(obj, key))
```

**GOTCHA:** `"key" in obj` checks the FULL prototype chain — it returns `true` for inherited properties. `Object.hasOwn()` only returns `true` for own (directly defined) properties. Use `in` when you care about inheritance, `hasOwn` when you specifically want own properties.

---

**Q8. What is the `at()` method on arrays and strings (ES2022)?** — Easy

**Answer:**
`Array.prototype.at()` and `String.prototype.at()` allow indexing from the end of a sequence using negative indices, eliminating the need for `arr[arr.length - n]` patterns.

```js
const arr = [10, 20, 30, 40, 50];

// Traditional last-element access:
arr[arr.length - 1];     // 50
arr[arr.length - 2];     // 40

// With .at():
arr.at(-1);  // 50  — last element
arr.at(-2);  // 40  — second to last
arr.at(0);   // 10  — same as arr[0]
arr.at(1);   // 20  — same as arr[1]

// Out of bounds returns undefined:
arr.at(100);  // undefined
arr.at(-100); // undefined

// Works on strings:
"Hello".at(-1);  // "o"
"Hello".at(0);   // "H"

// Works on TypedArrays:
new Int32Array([1, 2, 3]).at(-1); // 3

// Practical use — clean chaining:
[1, 2, 3].map(x => x * 2).at(-1);  // 6 — last element of mapped result
// Before: const arr2 = [1, 2, 3].map(x => x * 2); arr2[arr2.length - 1];

// With optional chaining:
const last = users?.at(-1)?.name; // Safe access to last user's name
```

**Follow-up:** Why not just use `arr[arr.length - 1]`?

The bracket syntax is error-prone: you must repeat the array name, and for complex expressions (like `getItems().at(-1)`), you'd need to store the result in a variable first. The `.at()` method integrates cleanly into chains and has consistent behavior with strings and typed arrays.

**GOTCHA:** `arr.at(-0)` is the same as `arr.at(0)` because `-0 === 0` in JavaScript. There is no "from-the-end index 0" using `.at()`. Also, `.at()` is a method call — `arr.at` on its own is a function reference, not the last element. Don't confuse it with a property access.

---

**Q9. What are `Array.prototype.findLast()` and `findLastIndex()` (ES2023)?** — Easy

**Answer:**
`findLast()` and `findLastIndex()` search arrays from the end, equivalent to `find()` and `findIndex()` but in reverse order.

```js
const items = [
  { id: 1, status: "done" },
  { id: 2, status: "pending" },
  { id: 3, status: "done" },
  { id: 4, status: "pending" }
];

// Find FIRST matching item (from start):
items.find(x => x.status === "done");      // { id: 1, status: "done" }
items.findIndex(x => x.status === "done"); // 0

// Find LAST matching item (from end):
items.findLast(x => x.status === "done");      // { id: 3, status: "done" }
items.findLastIndex(x => x.status === "done"); // 2

// No match — returns undefined / -1 just like find/findIndex:
items.findLast(x => x.status === "cancelled");      // undefined
items.findLastIndex(x => x.status === "cancelled"); // -1

// Before ES2023 — verbose workarounds:
// Option 1 — copy and reverse (slow for large arrays):
[...items].reverse().find(x => x.status === "done");

// Option 2 — manual loop:
let lastDone;
for (let i = items.length - 1; i >= 0; i--) {
  if (items[i].status === "done") { lastDone = items[i]; break; }
}

// findLast avoids the copy overhead and is more readable
```

**GOTCHA:** `findLast()` does NOT mutate the original array — unlike `reverse()` which mutates in-place (and why `[...items].reverse().find()` uses spread to copy). `findLast()` is purely read-only.

---

**Q10. What is `Array.prototype.toSorted()`, `toReversed()`, `toSpliced()`, and `with()` (ES2023)?** — Medium

**Answer:**
ES2023 introduced non-mutating equivalents of `sort()`, `reverse()`, and `splice()`, plus a new `with()` method for non-mutating index assignment.

```js
const arr = [3, 1, 4, 1, 5, 9, 2, 6];

// sort() — MUTATES the original:
arr.sort((a, b) => a - b); // arr is now [1, 1, 2, 3, 4, 5, 6, 9]

// toSorted() — returns NEW sorted array, original unchanged:
const original = [3, 1, 4, 1, 5];
const sorted = original.toSorted((a, b) => a - b);
// original: [3, 1, 4, 1, 5] — UNCHANGED
// sorted:   [1, 1, 3, 4, 5]

// toReversed() — returns NEW reversed array:
const rev = [1, 2, 3].toReversed();
// [3, 2, 1] — original [1, 2, 3] unchanged

// toSpliced() — returns NEW array with elements added/removed:
const arr2 = [1, 2, 3, 4, 5];
const spliced = arr2.toSpliced(1, 2, "a", "b");
// arr2:    [1, 2, 3, 4, 5]    — unchanged
// spliced: [1, "a", "b", 4, 5]

// with() — returns NEW array with one element replaced:
const arr3 = [1, 2, 3, 4, 5];
const updated = arr3.with(2, 99);   // replace index 2 with 99
// arr3:    [1, 2, 3, 4, 5]    — unchanged
// updated: [1, 2, 99, 4, 5]
// Negative indices work:
arr3.with(-1, 99); // [1, 2, 3, 4, 99]

// Why this matters — immutable state management (React/Redux):
// Before: create a copy then mutate
const newItems = [...state.items];
newItems.sort(compareFn);
setState({ items: newItems });

// After: use non-mutating methods directly:
setState({ items: state.items.toSorted(compareFn) });
```

**Follow-up:** Are these methods available for TypedArrays?

Yes — `TypedArray.prototype.toSorted()`, `toReversed()`, and `with()` are all available. `toSpliced()` is not on TypedArrays because TypedArrays have fixed length.

**GOTCHA:** `with()` and `toSpliced()` support negative indices, but they still follow the same bounds as `at()` — out-of-range indices throw a `RangeError`, unlike bracket access which returns `undefined`.

---

**Q11. What is the `structuredClone()` global function (Node.js 17+, browsers 2022)?** — Medium

**Answer:**
`structuredClone()` creates a deep copy of an object using the Structured Clone Algorithm — the same algorithm used by `postMessage` and IndexedDB. It handles far more types than JSON serialization.

```js
// JSON.parse/stringify — shallow workaround with severe limitations:
const clone1 = JSON.parse(JSON.stringify(obj));
// Problems:
// - Loses undefined values
// - Loses functions
// - Loses Date (becomes string)
// - Loses Map, Set (becomes {}, [])
// - Doesn't handle circular references (throws)

// structuredClone — proper deep clone:
const original = {
  date: new Date(),
  map: new Map([["a", 1]]),
  set: new Set([1, 2, 3]),
  buffer: new ArrayBuffer(8),
  nested: { deep: { value: 42 } },
  arr: [1, 2, [3, 4]]
};

const cloned = structuredClone(original);
cloned.nested.deep.value = 999;
original.nested.deep.value; // 42 — not mutated

// Circular references handled:
const circular = {};
circular.self = circular;
const clonedCircular = structuredClone(circular);
clonedCircular.self === clonedCircular; // true — circular preserved

// What structuredClone CANNOT handle:
const bad = {
  fn: () => {},             // Functions — throws DataCloneError
  symbol: Symbol("test"),   // Symbols — throws DataCloneError
  error: new Error("test"), // Error objects — throws DataCloneError
};
structuredClone(bad); // DataCloneError

// Transferables — move ownership instead of copying (for performance):
const buffer = new ArrayBuffer(1024 * 1024); // 1MB
const transferred = structuredClone(buffer, { transfer: [buffer] });
// buffer is now detached (empty) — ownership transferred to transferred
```

**GOTCHA:** `structuredClone` throws `DataCloneError` for functions, Symbols, DOM nodes (in workers), and WeakMap/WeakSet. If your object might contain functions (e.g., objects with methods), use a custom deep clone. Libraries like `lodash.cloneDeep` handle more cases but have their own limitations.

---

**Q12. What are `Array.fromAsync()` and `Object.groupBy()` / `Map.groupBy()` (ES2024)?** — Medium

**Answer:**
ES2024 added several useful methods for working with async iterables and grouping data.

```js
// Array.fromAsync() — creates an array from an async iterable:
async function* asyncNumbers() {
  yield 1;
  yield 2;
  yield 3;
}

const arr = await Array.fromAsync(asyncNumbers()); // [1, 2, 3]

// Also works with promises:
const resolved = await Array.fromAsync([
  Promise.resolve(1),
  Promise.resolve(2),
  Promise.resolve(3)
]); // [1, 2, 3]

// With a mapping function:
const doubled = await Array.fromAsync(asyncNumbers(), x => x * 2); // [2, 4, 6]

// Before: needed for...of loop + push:
const arr2 = [];
for await (const item of asyncNumbers()) arr2.push(item);

// Object.groupBy() — group array into object by key:
const items = [
  { name: "Alice", dept: "Engineering" },
  { name: "Bob",   dept: "Marketing" },
  { name: "Carol", dept: "Engineering" },
  { name: "Dave",  dept: "Marketing" },
];

const grouped = Object.groupBy(items, item => item.dept);
// {
//   Engineering: [{ name: "Alice", ... }, { name: "Carol", ... }],
//   Marketing:   [{ name: "Bob",   ... }, { name: "Dave",  ... }]
// }

// Map.groupBy() — like Object.groupBy but returns a Map (allows non-string keys):
const byDept = Map.groupBy(items, item => item.dept);
byDept.get("Engineering"); // [Alice, Carol objects]

// With non-string keys (only possible with Map.groupBy):
const byLength = Map.groupBy([1, 2, 3, 10, 20], n => n > 9 ? "big" : "small");
```

**GOTCHA:** `Object.groupBy()` requires the callback to return a string (or a value coercible to string). Using non-string keys results in `[object Object]` collisions. `Map.groupBy()` accepts any key type. Also, both methods were originally proposed as `Array.prototype.group()` but renamed to `Object.groupBy()` due to web compatibility concerns with a web polyfill that already used that name on Array.

---

**Q13. What is `Promise.withResolvers()` (ES2024)?** — Medium

**Answer:**
`Promise.withResolvers()` returns an object containing a new Promise along with its `resolve` and `reject` functions as properties — eliminating the "deferred pattern" boilerplate.

```js
// Before — verbose deferred pattern:
let resolve, reject;
const promise = new Promise((res, rej) => {
  resolve = res;
  reject = rej;
});
// resolve and reject are now accessible outside the Promise constructor

// ES2024 — Promise.withResolvers():
const { promise, resolve, reject } = Promise.withResolvers();
// All three available immediately, cleanly destructured

// Use case — event-driven completion:
function waitForClick(element) {
  const { promise, resolve } = Promise.withResolvers();
  element.addEventListener("click", resolve, { once: true });
  return promise;
}

await waitForClick(document.getElementById("submit-btn"));
console.log("Button clicked!");

// Use case — streaming to Promise:
function collectStream(readable) {
  const { promise, resolve, reject } = Promise.withResolvers();
  const chunks = [];
  readable.on("data", chunk => chunks.push(chunk));
  readable.on("end", () => resolve(Buffer.concat(chunks)));
  readable.on("error", reject);
  return promise;
}

// Use case — exposing resolve/reject to external code:
class AsyncQueue {
  #waiting = [];

  take() {
    const { promise, resolve } = Promise.withResolvers();
    this.#waiting.push(resolve);
    return promise;
  }

  put(item) {
    const resolve = this.#waiting.shift();
    if (resolve) resolve(item);
  }
}
```

**GOTCHA:** The Promise returned by `Promise.withResolvers()` is a standard Promise. The `resolve` and `reject` functions behave identically to those inside the `new Promise()` constructor — calling `resolve` multiple times or after `reject` has no effect. The difference is purely ergonomic.

---

**Q14. What are private class fields and methods (`#`) (ES2022)?** — Medium

**Answer:**
Private class fields and methods (prefixed with `#`) are truly private to the class — inaccessible from outside by any means, including subclasses and `Object.keys()`.

```js
class BankAccount {
  #balance = 0;           // private field
  #transactionHistory = []; // private field

  constructor(initialBalance) {
    this.#balance = initialBalance;
  }

  #recordTransaction(amount, type) { // private method
    this.#transactionHistory.push({ amount, type, date: new Date() });
  }

  deposit(amount) {
    if (amount <= 0) throw new RangeError("Amount must be positive");
    this.#balance += amount;
    this.#recordTransaction(amount, "deposit");
  }

  withdraw(amount) {
    if (amount > this.#balance) throw new Error("Insufficient funds");
    this.#balance -= amount;
    this.#recordTransaction(amount, "withdrawal");
  }

  get balance() { return this.#balance; }
}

const account = new BankAccount(1000);
account.balance;          // 1000 — public getter
account.deposit(500);
account.#balance;         // SyntaxError — CANNOT access private field from outside
account["#balance"];      // undefined — not a string key, doesn't exist

// Private fields are NOT on the prototype:
Object.keys(account);                  // []
Object.getOwnPropertyNames(account);   // [] — private fields not visible
account.hasOwnProperty("#balance");    // false

// Private static fields and methods:
class IdGenerator {
  static #count = 0;
  static #validate(n) { return n > 0; }

  static generate() {
    return ++IdGenerator.#count;
  }
}

// Checking if a private field exists (brand checking):
class Tagged {
  #tag = true;
  static isTagged(obj) {
    try {
      obj.#tag; // throws if obj doesn't have this private field
      return true;
    } catch {
      return false;
    }
  }
}
// Better — use `in` operator for private fields (ES2022):
class Tagged2 {
  #tag = true;
  static isTagged(obj) {
    return #tag in obj; // true if obj has this private field
  }
}
```

**Follow-up:** What is the difference between private fields (`#`) and convention-based private (`_`)?

`_prefix` is just a naming convention — the property is still fully public and accessible. `#name` is enforced by the language — the JavaScript engine itself prevents external access. There is no way to access a `#field` from outside the class body, not even through Proxy or `Object.getOwnPropertyDescriptor`.

**GOTCHA:** Private fields cannot be accessed in subclasses without a getter in the parent. `class Child extends Parent` — `Child` cannot read `Parent`'s `#privateField`. This is deliberate — true encapsulation. Also, private fields must be declared in the class body (with `#name` or `#name = value`) before they can be used in methods — unlike public properties which can be assigned anywhere.

---

**Q15. What is `using` / `await using` — the Explicit Resource Management proposal (ES2026)?** — Hard

**Answer:**
The `using` declaration (TC39 Stage 4, ES2026) provides automatic resource cleanup using `Symbol.dispose` (sync) and `Symbol.asyncDispose` (async), similar to Python's `with` statement or C#'s `using`.

```js
// Symbol.dispose — synchronous cleanup:
class FileHandle {
  constructor(path) {
    this.path = path;
    this.fd = fs.openSync(path, "r");
    console.log(`Opened: ${path}`);
  }

  read() { return fs.readFileSync(this.fd, "utf8"); }

  [Symbol.dispose]() {
    fs.closeSync(this.fd);
    console.log(`Closed: ${this.path}`);
  }
}

// using — auto-disposes at end of block:
{
  using file = new FileHandle("./data.txt");
  const content = file.read();
  console.log(content);
} // file[Symbol.dispose]() called automatically here

// Even if an error is thrown:
try {
  using file = new FileHandle("./data.txt");
  throw new Error("Something failed");
} catch (e) {
  // file is still disposed!
}

// await using — async cleanup:
class DatabaseConnection {
  constructor(url) { this.url = url; }
  async connect() { /* ... */ }
  async query(sql) { /* ... */ }

  async [Symbol.asyncDispose]() {
    await this.connection.close();
    console.log("DB connection closed");
  }
}

async function runQuery() {
  await using db = new DatabaseConnection(DB_URL);
  await db.connect();
  return await db.query("SELECT * FROM users");
} // db[Symbol.asyncDispose]() called automatically

// DisposableStack — manage multiple resources:
using stack = new DisposableStack();
const a = stack.use(new ResourceA());
const b = stack.use(new ResourceB());
// When stack is disposed, b is disposed first, then a (LIFO order)
```

**GOTCHA:** `using` requires the value to have a `[Symbol.dispose]` method — if it doesn't, it throws immediately at the `using` declaration point, not when trying to dispose. For `null`/`undefined`, `using x = null` is allowed and simply does nothing on disposal. The `using` keyword is a new reserved word only in certain contexts, so older transpilers may not handle it.

---

**Q16. What is `Array.prototype.group()` → `Object.groupBy()` naming evolution and what does it do?** — Medium

**Answer:**
This question reveals knowledge of TC39 process and web compatibility. The feature was first proposed as `Array.prototype.group()`, then renamed to `Object.groupBy()` due to a naming conflict with an existing Sugar.js library polyfill that was already used by many websites.

```js
// Final API — Object.groupBy() and Map.groupBy() (ES2024):
const people = [
  { name: "Alice", age: 30 },
  { name: "Bob",   age: 25 },
  { name: "Carol", age: 30 },
  { name: "Dave",  age: 25 }
];

// Object.groupBy — returns plain object:
const byAge = Object.groupBy(people, p => p.age);
// {
//   "25": [{ name: "Bob", age: 25 }, { name: "Dave", age: 25 }],
//   "30": [{ name: "Alice", age: 30 }, { name: "Carol", age: 30 }]
// }

// Map.groupBy — returns Map (preserves insertion order, allows non-string keys):
const byAgeMap = Map.groupBy(people, p => p.age);
byAgeMap.get(30); // [Alice, Carol] — key is number 30, not string "30"

// Complex grouping:
const inventory = [
  { item: "apple",  category: "fruit",  qty: 10 },
  { item: "banana", category: "fruit",  qty: 0  },
  { item: "carrot", category: "veggie", qty: 5  },
  { item: "date",   category: "fruit",  qty: 3  },
];

const { fruit, veggie } = Object.groupBy(inventory, i => i.category);
// fruit:  [apple, banana, date]
// veggie: [carrot]

// Multi-level grouping:
const grouped = Object.fromEntries(
  Object.entries(Object.groupBy(inventory, i => i.category))
    .map(([cat, items]) => [
      cat,
      Object.groupBy(items, i => i.qty > 0 ? "inStock" : "outOfStock")
    ])
);
```

**Follow-up:** Why was the naming change significant for TC39?

This was a landmark case of web compat breaking a spec proposal. Sugar.js had added `Array.prototype.group` to `Array.prototype` as a polyfill/extension. Many websites using Sugar.js would have broken if TC39 added a built-in with different semantics. The lesson: TC39 must check if a proposed method name is already used in popular libraries before finalizing. The rename to `Object.groupBy()` was the pragmatic solution.

**GOTCHA:** `Object.groupBy()` uses the string coercion of the key — number `30` becomes string `"30"`. To preserve key types, use `Map.groupBy()`. The callback receives `(element, index)` — the second argument `index` is available but rarely needed.

---

**Q17. What is `RegExp.prototype.flags` and the `v` flag (ES2024)?** — Hard

**Answer:**
The `v` (unicode sets) flag (ES2024) is a superset of the `u` flag, enabling string-based matching, set operations in character classes, and improved unicode handling.

```js
// v flag enables set operations in character classes:

// Set intersection [A&&B]:
/[\p{Letter}&&[A-Z]]/v.test("A"); // true — letter AND uppercase ASCII
/[\p{Letter}&&[A-Z]]/v.test("a"); // false — letter but not uppercase ASCII
/[\p{Letter}&&[A-Z]]/v.test("1"); // false — not a letter

// Set subtraction [A--B]:
/[\p{Letter}--[A-Za-z]]/v.test("é"); // true — non-ASCII letter
/[\p{Letter}--[A-Za-z]]/v.test("a"); // false — excluded by --[A-Za-z]

// Nested character classes:
/[[0-9][a-f]]/v.test("c"); // true — hex digit

// String properties (v only, not u):
/\p{Emoji_Keycap_Sequence}/v.test("1️⃣"); // true — emoji sequence (multi-codepoint)
// u flag cannot match multi-codepoint unicode properties

// RegExp.prototype.flags — read all flags as a string:
const re = /hello/gim;
re.flags; // "gim" — always alphabetically sorted: d, g, i, m, s, u, v, y
// Added in ES6 (ES2015), but v flag added in 2024

// Other recent additions:
// hasIndices property:
const re2 = /test/d;
re2.hasIndices; // true

// dotAll property:
const re3 = /test/s;
re3.dotAll; // true

// unicode, sticky, global, ignoreCase, multiline:
/test/gui.unicode;     // true
/test/y.sticky;        // true
```

**GOTCHA:** The `v` and `u` flags are mutually exclusive — you cannot use both together. The `v` flag makes unrecognized unicode property names a SyntaxError (stricter than `u`). Also, the `v` flag changes the behavior of some existing character class syntax — `{` and `}` inside `[...]` have special meaning, so literal `{` or `}` must be escaped as `\{` or `\}`.

---

**Q18. What is top-level `await` in ES modules (ES2022)?** — Medium

**Answer:**
Top-level `await` allows `await` at the module's top level — outside any `async` function. A module with top-level `await` is itself async — modules that import it will wait for it to resolve before executing.

```js
// Before — had to wrap in async IIFE:
(async () => {
  const data = await fetch("/api/config").then(r => r.json());
  console.log(data);
})();

// ES2022 — top-level await (in .mjs or type="module"):
const data = await fetch("/api/config").then(r => r.json());
console.log(data); // Works at module level

// Use case — conditional polyfill loading:
const { platform } = await import(
  navigator.userAgent.includes("Chrome")
    ? "./chrome-apis.mjs"
    : "./fallback-apis.mjs"
);

// Use case — database connection at module load time:
// db.mjs:
import { createPool } from "mysql2/promise";
export const pool = await createPool({ host: "localhost", database: "app" });

// app.mjs:
import { pool } from "./db.mjs";
// pool is already connected — db.mjs waited for the pool before this import resolved

// Use case — feature detection with dynamic import:
let JSONParse;
try {
  const { parse } = await import("./fast-json.mjs");
  JSONParse = parse;
} catch {
  JSONParse = JSON.parse; // fallback
}

// HOW IT WORKS — module graph:
// A module with top-level await is "async" in the module graph.
// Parent modules that import it get a "pending" link until the child settles.
// Sibling modules continue loading in parallel.
```

**Follow-up:** What are the performance implications of top-level await?

If module A has top-level `await` and modules B and C both import A, then B and C both wait for A's async initialization before executing. This can create a waterfall of initialization delays if not designed carefully. Top-level `await` should be used for necessary async initialization (like DB connections), not for trivial operations that could be deferred.

**GOTCHA:** Top-level `await` only works in ES Modules — it's a `SyntaxError` in CommonJS files or classic scripts. In bundlers like Webpack, it requires specific configuration. Also, a module that has top-level `await` that never resolves (a hanging Promise) will block all modules that import it indefinitely.

---

**Q19. What is `Error.cause` and how does `AggregateError` work (ES2021)?** — Medium

**Answer:**
These two features improve error handling expressiveness. `Error.cause` (ES2022) chains errors with context; `AggregateError` (ES2021) wraps multiple errors into one.

```js
// Error.cause — wrap errors with contextual message:
async function fetchUser(id) {
  try {
    return await db.query(`SELECT * FROM users WHERE id = ?`, [id]);
  } catch (dbError) {
    throw new Error(`Failed to fetch user with id=${id}`, { cause: dbError });
  }
}

// Inspecting the cause chain:
try {
  await fetchUser(42);
} catch (e) {
  console.log(e.message);       // "Failed to fetch user with id=42"
  console.log(e.cause.message); // "Connection refused" (the db error)
  console.log(e.cause.code);    // "ECONNREFUSED"
}

// AggregateError — for multiple concurrent failures:
// Used by Promise.any() when ALL promises reject:
try {
  await Promise.any([
    Promise.reject(new Error("Server A unreachable")),
    Promise.reject(new Error("Server B unreachable")),
    Promise.reject(new TypeError("Server C timeout"))
  ]);
} catch (agg) {
  agg instanceof AggregateError; // true
  agg.message;                   // "All promises were rejected"
  agg.errors.length;             // 3
  agg.errors[0].message;         // "Server A unreachable"
  agg.errors[2] instanceof TypeError; // true
}

// Creating AggregateError manually:
throw new AggregateError(
  [new Error("Validation A failed"), new Error("Validation B failed")],
  "Multiple validation errors occurred"
);

// Logging full cause chain:
function printChain(err, indent = 0) {
  console.error(" ".repeat(indent * 2) + `[${err.name}] ${err.message}`);
  if (err.cause) printChain(err.cause, indent + 1);
  if (err.errors) err.errors.forEach(e => printChain(e, indent + 1));
}
```

**GOTCHA:** `AggregateError.prototype.errors` is an array of the original error objects. Unlike the `cause` property on regular errors, the `message` on `AggregateError` is typically generic ("All promises were rejected") and the detail is in `.errors`. Always inspect `.errors` when handling `AggregateError`.

---

**Q20. What are the most impactful ES2020-2025 features for day-to-day development?** — Medium

**Answer:**
A curated summary of the most practically impactful features:

```js
// ✅ ES2020 — Game changers:
// 1. Optional chaining — eliminates defensive null-checking boilerplate:
user?.profile?.avatar?.url ?? "/default-avatar.png";

// 2. Nullish coalescing — correct default for 0, "":
const timeout = config.timeout ?? 5000;

// 3. Promise.allSettled — handle mixed success/failure results:
const results = await Promise.allSettled(fetchAll(urls));

// 4. BigInt — safe large number handling:
const id = BigInt(rowId);

// ✅ ES2021:
// 5. Logical assignment — clean initialization patterns:
config.timeout ??= 5000;
user.permissions &&= filterValid(user.permissions);

// 6. String.replaceAll — clear intent:
text.replaceAll("foo", "bar");

// 7. Promise.any — first success, use as fallback:
const data = await Promise.any(mirrors.map(fetch));

// ✅ ES2022:
// 8. Private class fields — true encapsulation:
class Store { #state = {}; get(key) { return this.#state[key]; } }

// 9. Top-level await — cleaner module initialization:
const config = await loadConfig();

// 10. Object.hasOwn — safe own-property check:
if (Object.hasOwn(obj, key)) { ... }

// 11. Array .at() — clean negative indexing:
arr.at(-1); // last element

// 12. Error.cause — meaningful error chains:
throw new Error("DB failed", { cause: originalError });

// ✅ ES2023:
// 13. findLast / findLastIndex:
events.findLast(e => e.type === "click");

// 14. toSorted / toReversed / with — immutable array operations:
setState(prev => ({ items: prev.items.toSorted() }));

// ✅ ES2024:
// 15. Object.groupBy / Map.groupBy — data organization:
const byStatus = Object.groupBy(tasks, t => t.status);

// 16. Promise.withResolvers — clean deferred patterns:
const { promise, resolve, reject } = Promise.withResolvers();
```

**GOTCHA:** Feature availability varies. Always check `caniuse.com` or the Node.js compatibility table before using cutting-edge features in production without transpilation. TypeScript adds these features with their own release cycle (separate from the ES spec). Some features (like `using`) require both engine support AND transpiler support (Babel/TypeScript).

---

*Next: [14-NodeJS-Specifics.md](./14-NodeJS-Specifics.md)*
