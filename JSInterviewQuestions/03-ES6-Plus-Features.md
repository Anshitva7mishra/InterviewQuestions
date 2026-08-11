# 03 — ES6+ Features
### 20 Questions | All Levels

---

**Q1. What is destructuring and what are its advanced forms?** — Easy

**Answer:**
Destructuring is syntax that unpacks values from arrays or properties from objects into distinct variables in a single statement. It does not mutate the source.

Object destructuring:
```js
const user = { name: "Alice", age: 30, city: "London" };

// Basic:
const { name, age } = user;

// Rename:
const { name: userName } = user;

// Default value (used when property is undefined):
const { role = "viewer" } = user; // role is "viewer" — user.role is undefined

// Nested:
const { address: { city, zip = "N/A" } = {} } = user;

// Rest:
const { name: n, ...rest } = user; // rest = { age: 30, city: "London" }
```

Array destructuring:
```js
const [a, b, c] = [1, 2, 3];

// Skip elements:
const [first, , third] = [1, 2, 3]; // first=1, third=3

// Default values:
const [x = 0, y = 0] = [10]; // x=10, y=0

// Swap variables:
let p = 1, q = 2;
[p, q] = [q, p]; // p=2, q=1 — no temp variable needed

// Rest:
const [head, ...tail] = [1, 2, 3, 4]; // head=1, tail=[2,3,4]
```

Function parameter destructuring:
```js
function createUser({ name, age = 18, role = "user" } = {}) {
  return { name, age, role };
}
createUser({ name: "Bob" }); // { name: "Bob", age: 18, role: "user" }
```

**Spec Reference:** ECMAScript section 14.3.3 — Destructuring Binding Patterns

**Follow-up:** What happens when you destructure a `null` or `undefined` value?

It throws a `TypeError`. `const { x } = null` throws because the spec tries to call `ToObject(null)` and `ToObject` throws for null/undefined. This is why function parameter destructuring with `= {}` as default is a safety pattern — it protects against being called with no argument.

**GOTCHA:** Default values in destructuring only apply when the value is `undefined`, not when it is `null`. `const { x = 5 } = { x: null }` gives `x = null`, not `x = 5`.

---

**Q2. What is the spread operator and what are all its use cases?** — Easy

**Answer:**
The spread operator `...` expands an iterable into individual elements. It works with arrays, strings, Maps, Sets, generators — any object implementing `Symbol.iterator`.

Array spread:
```js
const a = [1, 2, 3];
const b = [4, 5, 6];

[...a, ...b];          // [1,2,3,4,5,6] — merge arrays
[...a];                // [1,2,3] — shallow copy
[0, ...a, 4];          // [0,1,2,3,4] — insert at any position

// Spread a string into characters:
[..."hello"];          // ["h","e","l","l","o"]

// Convert iterable to array:
[...new Set([1,1,2])]; // [1,2]
[...new Map([["a",1]]].entries()]; // [["a",1]]
```

Object spread (ES2018):
```js
const base = { a: 1, b: 2 };
const override = { b: 3, c: 4 };

// Merge (later properties win):
{ ...base, ...override }; // { a: 1, b: 3, c: 4 }

// Shallow clone:
const copy = { ...base };

// Add properties:
{ ...base, d: 5 }; // { a: 1, b: 2, d: 5 }
```

Function call spread:
```js
Math.max(...[1, 2, 3]);  // 3 — same as Math.max(1, 2, 3)
someFunction(...args, extraArg);
```

**Follow-up:** What is the difference between spread and `Object.assign`?

`Object.assign(target, source)` mutates `target` and copies only own, enumerable properties. Object spread `{ ...source }` creates a new object and also copies only own, enumerable properties. For shallow merging, they are functionally identical — the key difference is spread always creates a new object while `Object.assign` modifies an existing one.

**GOTCHA:** Object spread copies own enumerable properties only — non-enumerable properties and prototype properties are not copied. Getter functions on the source object are INVOKED (not copied as getters) and their return value is assigned. To copy getters, you must use `Object.defineProperties`.

---

**Q3. What is the rest operator and how does it differ from the spread operator?** — Easy

**Answer:**
The rest operator and spread operator use the same `...` syntax but are semantically opposite: spread expands, rest collects.

Rest in function parameters:
```js
function sum(first, ...rest) {
  // 'rest' collects all arguments after 'first' into a real Array
  return rest.reduce((acc, n) => acc + n, first);
}
sum(1, 2, 3, 4); // 10

// rest must be the LAST parameter:
function f(a, ...b, c) {} // SyntaxError — rest must be last
```

Rest in destructuring:
```js
const [head, ...tail] = [1, 2, 3, 4]; // head=1, tail=[2,3,4]
const { a, ...remaining } = { a: 1, b: 2, c: 3 }; // remaining={b:2,c:3}
```

Rest vs `arguments` object:
```js
// arguments — old way, not available in arrow functions, not a real Array:
function old() { return Array.from(arguments).filter(Boolean); }

// rest — new way, real Array, works anywhere:
const modern = (...args) => args.filter(Boolean);
```

**GOTCHA:** Rest parameters create a genuine `Array` instance. The `arguments` object is array-like but not an actual Array — it lacks `.map()`, `.filter()`, etc. Prefer rest parameters in all modern code. Also, rest parameters do NOT include explicitly named parameters before them — only the "rest" of the arguments.

---

**Q4. Explain template literals, tagged templates, and what makes tagged templates powerful.** — Medium

**Answer:**
Template literals use backticks and allow embedded expressions with `${}` and multi-line strings:
```js
const name = "Alice";
const age = 30;
`Hello, ${name}! You are ${age > 18 ? "an adult" : "a minor"}.`
// "Hello, Alice! You are an adult."

// Multi-line without \n:
const html = `
  <div>
    <p>${name}</p>
  </div>
`;
```

Tagged template literals: a tag function intercepts the template literal and can process it arbitrarily:
```js
function tag(strings, ...values) {
  // strings: array of raw string parts
  // values: array of interpolated values
  return strings.raw[0]; // access raw (unprocessed) string
}

tag`Hello\n${name}!`
// strings: ["Hello\n", "!"], strings.raw: ["Hello\\n", "!"]
// values: ["Alice"]
```

Real-world uses of tagged templates:
```js
// 1. SQL sanitization — prevent injection:
function sql(strings, ...values) {
  return {
    text: strings.join("?"),
    values: values
  };
}
const query = sql`SELECT * FROM users WHERE id = ${userId}`;
// { text: "SELECT * FROM users WHERE id = ?", values: [userId] }

// 2. Styled components (CSS-in-JS):
const Button = styled.button`
  background: ${props => props.primary ? "blue" : "white"};
  color: ${props => props.primary ? "white" : "black"};
`;

// 3. HTML sanitization:
function html(strings, ...values) {
  return strings.reduce((acc, str, i) => {
    const val = values[i - 1];
    return acc + (val ? String(val).replace(/</g, "&lt;") : "") + str;
  });
}
```

**Spec Reference:** ECMAScript section 13.2.8 — Template Literals

**GOTCHA:** `strings.length` in a tag function is always `values.length + 1`. There is always one more string chunk than there are interpolated values. If the template starts and ends with expressions (`${a}${b}`), the strings array is `["", "", ""]` — three empty strings around two values.

---

**Q5. What are `Map` and `Set` and when should you use them over plain objects and arrays?** — Medium

**Answer:**
`Map` is a key-value collection where keys can be ANY type (not just strings/symbols), maintains insertion order, and has a proper `.size` property.

`Set` is a collection of unique values (any type), maintains insertion order, and has O(1) lookup.

```js
// Map — when keys are not strings:
const map = new Map();
const keyObj = { id: 1 };
const keyFn = function() {};

map.set(keyObj, "object key value");
map.set(keyFn, "function key value");
map.set(42, "number key value");

map.get(keyObj); // "object key value"
map.size;        // 3

// Iteration (maintains insertion order):
for (const [key, value] of map) { }
map.keys();   // iterator of keys
map.values(); // iterator of values
map.entries(); // iterator of [key, value] pairs

// Initialize from entries:
const m = new Map([["a", 1], ["b", 2]]);
```

```js
// Set — guaranteed uniqueness, fast lookup:
const set = new Set([1, 2, 2, 3, 3, 3]);
set.size; // 3 — duplicates removed

set.has(2);    // true — O(1) lookup
set.add(4);    // Set {1,2,3,4}
set.delete(1); // true

// Convert to array:
[...set]; // [1,2,3,4]
Array.from(set);

// Remove duplicates from array:
const unique = [...new Set(arr)];
```

When to use Map over plain objects:
- Keys are non-string types (objects, functions, numbers)
- You frequently add and delete entries (Map is optimized for this)
- You need the insertion order guaranteed (objects also maintain insertion order for string keys in modern engines, but Map is the spec-guaranteed version)
- You need to iterate key-value pairs frequently

When to use Set over arrays:
- Membership testing (`has` is O(1) vs array's `includes` which is O(n))
- Deduplication
- Mathematical set operations (union, intersection, difference)

**GOTCHA:** `Map` is NOT JSON-serializable. `JSON.stringify(new Map([["a", 1]]))` returns `"{}"` — the Map content is lost. Convert to entries array: `JSON.stringify([...map.entries()])`.

---

**Q6. What are `WeakMap` and `WeakSet` and why do they exist?** — Hard

**Answer:**
`WeakMap` and `WeakSet` hold their keys/values weakly — the garbage collector can collect them even if they still appear in the collection, provided there are no other strong references.

`WeakMap`:
- Keys MUST be objects (or registered symbols in ES2023) — not primitives
- No `.size`, no `.keys()`, no iteration — by design
- When a key object is GC'd, the entry is automatically removed

`WeakSet`:
- Values MUST be objects
- Same restrictions as WeakMap — no iteration, no size

```js
// Use case 1: Private data per instance (pre-private class fields):
const privateData = new WeakMap();
class User {
  constructor(name, password) {
    privateData.set(this, { password }); // Not accessible externally
    this.name = name;
  }
  checkPassword(pwd) {
    return privateData.get(this).password === pwd;
  }
}
// When a User instance is GC'd, its entry in privateData is also GC'd automatically

// Use case 2: DOM node metadata:
const nodeData = new WeakMap();
function attachData(node, data) {
  nodeData.set(node, data);
}
// When the DOM node is removed and GC'd, its metadata is also cleaned up automatically
// If you used a regular Map, you'd have a memory leak

// Use case 3: Caching computed results per object:
const cache = new WeakMap();
function process(obj) {
  if (cache.has(obj)) return cache.get(obj);
  const result = /* expensive computation */ obj.data.map(transform);
  cache.set(obj, result);
  return result;
}
```

Why no iteration or size: If you could iterate a WeakMap, the GC would need to prevent collection during iteration. The design intentionally prevents this to keep GC behavior non-deterministic and efficient.

**Spec Reference:** ECMAScript section 27.3 — WeakMap Objects

**GOTCHA:** You cannot use primitives as WeakMap keys. `weakMap.set("string", value)` throws TypeError. Primitives cannot be GC'd (they have no identity), so a weak reference to a primitive would never be collected, defeating the purpose.

---

**Q7. What are default parameters and what are their subtle behaviors?** — Easy

**Answer:**
Default parameters specify fallback values for function parameters when the corresponding argument is `undefined` (not when it is `null` or any other falsy value).

```js
function createUser(name, role = "viewer", permissions = []) {
  return { name, role, permissions };
}

createUser("Alice");            // { name:"Alice", role:"viewer", permissions:[] }
createUser("Bob", "admin");     // { name:"Bob", role:"admin", permissions:[] }
createUser("Chris", null);      // { name:"Chris", role:null, permissions:[] }
// null does NOT trigger the default — only undefined does
createUser("Dan", undefined);   // { name:"Dan", role:"viewer", permissions:[] }
// undefined DOES trigger the default
```

Defaults can be expressions (evaluated lazily at call time):
```js
function timestamp(time = Date.now()) {
  return time;
}
timestamp();    // current time (evaluated when called)
timestamp(0);   // 0

// Defaults can reference earlier parameters:
function rect(width, height = width) {
  return { width, height }; // square if height not provided
}
rect(5);    // { width: 5, height: 5 }
rect(5, 3); // { width: 5, height: 3 }
```

Default parameters and the TDZ:
```js
// Cannot reference later parameters in defaults:
function f(a = b, b = 1) {} // ReferenceError when called without args
// b is in TDZ when a's default is evaluated
```

Effect on `function.length`: Default parameters reduce the reported function length.
```js
function f(a, b = 1, c) {}
f.length; // 1 — only parameters BEFORE the first default count
```

**GOTCHA:** Each call to the function evaluates default expressions independently. `function f(arr = [])` creates a NEW empty array for every call — unlike a function-level `const arr = []` which would share the same array. This is actually the correct, safe behavior — unlike Python where mutable defaults are shared.

---

**Q8. What are optional chaining (`?.`) and nullish coalescing (`??`) and how do they interact?** — Easy

**Answer:**
Optional chaining (`?.`) short-circuits and returns `undefined` if the value before `?.` is `null` or `undefined`, instead of throwing a `TypeError`.

```js
const user = { profile: { address: { city: "London" } } };

// Without optional chaining:
const city = user && user.profile && user.profile.address && user.profile.address.city;

// With optional chaining:
const city = user?.profile?.address?.city; // "London"
const zip = user?.profile?.address?.zip;   // undefined (no TypeError)

// Works for method calls:
user?.profile?.getFullName?.(); // undefined if method doesn't exist

// Works for array access:
const first = arr?.[0];

// Works with computed property access:
const key = "city";
user?.profile?.address?.[key]; // "London"

// Short-circuits the whole expression:
user?.profile?.address?.city?.toUpperCase(); // "LONDON" or undefined
```

Nullish coalescing (`??`) provides a default only when the left side is `null` or `undefined`:
```js
// ?? vs ||:
const count = 0;
count || 5;   // 5 — 0 is falsy, || triggers
count ?? 5;   // 0 — 0 is not nullish, ?? does not trigger

// Combined — the common pattern:
const city = user?.profile?.address?.city ?? "Unknown";
// If any step is null/undefined, city = "Unknown"
// If city exists but is "", city = "" (not "Unknown")
```

Nullish assignment (`??=`, ES2021):
```js
user.profile ??= {}; // Only assigns if user.profile is null or undefined
```

**Spec Reference:** ECMAScript section 13.5.6 — Optional Chaining

**GOTCHA:** `?.` is NOT the same as a safe-navigation operator that gracefully handles all errors. It only guards against `null`/`undefined` in the chain. If `user.profile` exists but `user.profile.getCity` is not a function, `user?.profile?.getCity?.()` returns `undefined`, not an error — but `user?.profile?.getCity()` (without the `?.` after) WOULD throw TypeError if `getCity` is not a function.

---

**Q9. What are class fields (public and private) and how do they differ from constructor assignments?** — Medium

**Answer:**
Class fields (ES2022) allow you to declare instance and static properties directly on the class body, outside the constructor.

Public fields:
```js
class Counter {
  count = 0;            // Public instance field — initialized per instance
  static instances = 0; // Public static field — shared across all instances

  constructor() {
    Counter.instances++;
    // 'count' is already initialized to 0 before this runs
  }

  increment() {
    this.count++;
  }
}

const c = new Counter();
c.count;             // 0
Counter.instances;   // 1
```

Private fields (`#`):
```js
class BankAccount {
  #balance = 0;      // Private — inaccessible outside class
  #owner;            // Private, no initial value (undefined until set)

  constructor(owner, initialBalance) {
    this.#owner = owner;
    this.#balance = initialBalance;
  }

  deposit(amount) {
    if (amount <= 0) throw new Error("Invalid amount");
    this.#balance += amount;
    return this;
  }

  get balance() { return this.#balance; }
}

const account = new BankAccount("Alice", 1000);
account.balance;    // 1000 — via getter
account.#balance;   // SyntaxError — truly private, not just convention
```

Difference from constructor assignment:
- `count = 0` as a class field creates the property on the instance BEFORE the constructor runs
- `this.count = 0` in the constructor creates it during construction
- Private fields MUST be declared in the class body — you cannot dynamically add them

Checking for private field existence:
```js
class C {
  #x;
  static isC(obj) {
    return #x in obj; // private field existence check (ES2022)
  }
}
```

**Spec Reference:** ECMAScript section 15.7.1 — Class Definitions

**GOTCHA:** Private fields are per-class, not per-instance. Two instances of the same class share the SAME private field slot definition, but each instance has its own value. Subclass instances do NOT have access to parent private fields — even through inherited methods that access `this.#field` (this works as long as `this` is an instance of the defining class).

---

**Q10. What are static class methods and fields, and what is the `static` block?** — Medium

**Answer:**
Static members belong to the class itself, not to instances. They are accessed via the class name.

```js
class MathUtils {
  static PI = 3.14159265358979;
  #instanceId;

  static #instanceCount = 0; // Private static field

  constructor() {
    MathUtils.#instanceCount++;
    this.#instanceId = MathUtils.#instanceCount;
  }

  static getInstanceCount() {
    return MathUtils.#instanceCount; // Static method can access private static fields
  }

  static circle_area(r) {
    return MathUtils.PI * r ** 2;
  }
}

MathUtils.PI;                   // 3.14159...
MathUtils.circle_area(5);       // 78.539...
MathUtils.getInstanceCount();   // number of instances created

new MathUtils();
MathUtils.getInstanceCount();   // 1
```

Static initialization blocks (ES2022):
```js
class Config {
  static debug;
  static logLevel;

  static {
    // Complex initialization logic — runs once when class is evaluated
    Config.debug = process.env.NODE_ENV !== "production";
    Config.logLevel = Config.debug ? "verbose" : "error";
    // Can access private static fields here
  }
}
```

Static inheritance:
```js
class Animal {
  static kingdom = "Animalia";
  static describe() { return `Kingdom: ${this.kingdom}`; }
}

class Dog extends Animal {
  static kingdom = "Animalia (Canidae)";
}

Dog.describe(); // "Kingdom: Animalia (Canidae)" — 'this' is Dog in static context
```

**GOTCHA:** Static methods are NOT inherited by instances. `new Dog().describe()` throws TypeError — `describe` is on the class, not on the prototype of instances. Static methods ARE inherited by subclass constructors via the class prototype chain (`Dog.__proto__ === Animal`).

---

**Q11. What is `Symbol.iterator` and how do you make a custom iterable?** — Medium

**Answer:**
`Symbol.iterator` is a well-known symbol that, when defined as a method on an object, makes that object iterable — meaning it can be used with `for...of`, spread, destructuring, and all built-in iteration consumers.

The `Symbol.iterator` method must return an iterator object with a `next()` method that returns `{ value, done }`.

```js
// Custom range iterable:
class Range {
  constructor(start, end, step = 1) {
    this.start = start;
    this.end = end;
    this.step = step;
  }

  [Symbol.iterator]() {
    let current = this.start;
    const end = this.end;
    const step = this.step;

    return {
      next() {
        if (current <= end) {
          const value = current;
          current += step;
          return { value, done: false };
        }
        return { value: undefined, done: true };
      }
    };
  }
}

const range = new Range(1, 10, 2);
[...range];                    // [1, 3, 5, 7, 9]
for (const n of range) { }    // iterates 1,3,5,7,9
const [first, second] = range; // first=1, second=3

// Using a generator for simplicity:
class Range2 {
  constructor(start, end) { this.start = start; this.end = end; }

  *[Symbol.iterator]() { // * makes this a generator method
    for (let i = this.start; i <= this.end; i++) yield i;
  }
}
```

**Spec Reference:** ECMAScript section 7.4 — Operations on Iterator Objects

**GOTCHA:** An iterable and an iterator are different concepts. An iterable has `Symbol.iterator` and returns an iterator. An iterator has `.next()` and returns `{value, done}`. They can be the same object (self-referential iterators) — generators produce objects that are both: the generator object's `Symbol.iterator` returns itself. But custom iterables often separate the two.

---

**Q12. What is `Promise.allSettled` and why was it added after `Promise.all`?** — Medium

**Answer:**
`Promise.all` was designed for the "everything must succeed" use case. It fails fast — the moment any one promise rejects, the whole operation rejects. This is correct for many scenarios but impossible when you need to know the outcome of ALL promises regardless of individual failures.

`Promise.allSettled` (ES2020) waits for all promises to settle (fulfill or reject) and returns an array of outcome descriptors:

```js
const results = await Promise.allSettled([
  fetch("/api/users"),
  fetch("/api/products"),
  fetch("/api/orders")  // This one might fail
]);

results.forEach(result => {
  if (result.status === "fulfilled") {
    console.log("Success:", result.value);   // The resolved value
  } else {
    console.log("Failed:", result.reason);    // The rejection reason
  }
});
```

Shape of results:
```js
// Fulfilled:
{ status: "fulfilled", value: <resolved-value> }

// Rejected:
{ status: "rejected", reason: <rejection-reason> }
```

Comparison:
```js
// Promise.all — one failure kills everything:
await Promise.all([success(), failure(), success()]);
// throws immediately when failure() rejects — you get nothing

// Promise.allSettled — always wait for all:
await Promise.allSettled([success(), failure(), success()]);
// [fulfilled, rejected, fulfilled] — you get partial results
```

**Follow-up:** Is there a way to get `Promise.all` behavior but still collect errors?

Yes — wrap each promise in a catch that converts rejections into error values:
```js
const results = await Promise.all(
  promises.map(p => p.catch(err => ({ error: err })))
);
```

**GOTCHA:** `Promise.allSettled` never rejects — it always fulfills with the array. The array itself contains objects describing whether each input promise fulfilled or rejected. You must check the `status` field on each result.

---

**Q13. What is `for...of` and what can it iterate over?** — Easy

**Answer:**
`for...of` is a loop that iterates over the values of any iterable object. It calls `Symbol.iterator` on the object to get an iterator, then calls `.next()` until `done: true`.

Built-in iterables:
- Arrays
- Strings (iterates Unicode code points, not UTF-16 code units)
- Maps (iterates `[key, value]` pairs)
- Sets (iterates values)
- Typed arrays
- Arguments object
- NodeList (DOM)
- Generators

```js
// Array:
for (const n of [1, 2, 3]) console.log(n); // 1, 2, 3

// String — handles Unicode correctly:
for (const char of "hello") console.log(char); // h, e, l, l, o
for (const char of "a\uD83D\uDE00") console.log(char); // a, (emoji as one unit)
// "a\uD83D\uDE00"[0] === "a", "a\uD83D\uDE00"[1] === "\uD83D" — broken code unit
// But for...of iterates full code points

// Map:
const m = new Map([["a", 1], ["b", 2]]);
for (const [key, val] of m) console.log(key, val);

// Destructuring:
for (const [i, v] of arr.entries()) console.log(i, v);

// Generator:
function* gen() { yield 1; yield 2; }
for (const v of gen()) console.log(v);

// break works correctly:
for (const n of [1, 2, 3]) {
  if (n === 2) break; // iterator's return() is called for cleanup
}
```

Plain objects are NOT iterable by default:
```js
for (const x of {}) {} // TypeError: {} is not iterable
// Use: for (const [key, val] of Object.entries(obj))
```

**GOTCHA:** `for...of` on a string iterates Unicode code points (characters), but bracket access `str[i]` gives UTF-16 code units. For strings with emoji or characters above U+FFFF, `for...of` gives correct results while index-based iteration can give broken surrogate pairs.

---

**Q14. What are generator functions and async generator functions?** — Hard

**Answer:**
Generator functions (`function*`) return an iterator that produces values lazily via `yield`.

Async generator functions (`async function*`) combine generators with async/await — each `yield` can `await` a promise before producing the next value. The consumer uses `for await...of` to iterate.

```js
// Sync generator:
function* numberStream(start, end) {
  for (let i = start; i <= end; i++) {
    yield i;
  }
}
for (const n of numberStream(1, 5)) console.log(n); // 1,2,3,4,5

// Async generator — yields values that require async work:
async function* paginatedFetch(baseUrl) {
  let page = 1;
  while (true) {
    const res = await fetch(`${baseUrl}?page=${page}`);
    const data = await res.json();
    if (data.items.length === 0) return; // done
    yield data.items; // yield an array of items from each page
    page++;
  }
}

// Consumer:
async function loadAll() {
  for await (const items of paginatedFetch("/api/products")) {
    // Process each page of items as they arrive
    items.forEach(displayItem);
  }
}
```

Real-world use case — infinite scroll with async generator:
```js
async function* scrollStream(target) {
  while (true) {
    yield new Promise(resolve => {
      const observer = new IntersectionObserver(([entry]) => {
        if (entry.isIntersecting) {
          observer.disconnect();
          resolve();
        }
      });
      observer.observe(target);
    });
  }
}

for await (const _ of scrollStream(sentinel)) {
  await loadMoreItems();
}
```

**Spec Reference:** ECMAScript section 15.5 — Async Generator Function Definitions

**GOTCHA:** `for await...of` works on both async iterables and regular sync iterables. It wraps each `next()` call result in `Promise.resolve()`. If you use `for await...of` on a regular array, it works but adds unnecessary microtask overhead.

---

**Q15. What is the difference between `class` syntax and the older prototype-based pattern?** — Medium

**Answer:**
ES6 `class` syntax is syntactic sugar over the prototype-based pattern. The underlying mechanism is identical — classes create constructor functions and set up prototype chains. There are a few behavioral differences though.

Old prototype-based pattern:
```js
function Animal(name) {
  this.name = name;
}
Animal.prototype.speak = function() {
  return `${this.name} makes a sound.`;
};

function Dog(name) {
  Animal.call(this, name); // super() call manually
}
Dog.prototype = Object.create(Animal.prototype);
Dog.prototype.constructor = Dog;

Dog.prototype.bark = function() {
  return `${this.name} barks.`;
};
```

ES6 class equivalent:
```js
class Animal {
  constructor(name) {
    this.name = name;
  }
  speak() {
    return `${this.name} makes a sound.`;
  }
}

class Dog extends Animal {
  bark() {
    return `${this.name} barks.`;
  }
}
```

Behavioral differences (classes are NOT just sugar in every way):
1. Classes must be called with `new` — constructor functions can be called without it
2. Class methods are non-enumerable by default — prototype methods added manually ARE enumerable
3. Classes are in the TDZ (like `let`) — function declarations are hoisted
4. Class bodies always run in strict mode
5. `super()` must be called before accessing `this` in derived class constructors
6. Static methods are properly inherited via the class prototype chain (`Dog.__proto__ === Animal`)

**GOTCHA:** In the old prototype pattern, you had to manually set `Dog.prototype.constructor = Dog` after setting up the prototype chain — otherwise `new Dog() instanceof Dog` would fail, and `constructor` would point to `Animal`. The `class` syntax handles this automatically.

---

**Q16. What are computed property names?** — Easy

**Answer:**
Computed property names allow you to use an expression as a property key in an object literal or class, wrapped in square brackets.

```js
const prefix = "user";
const id = 42;

const obj = {
  [prefix + "Name"]: "Alice",      // "userName": "Alice"
  [`${prefix}Id`]: id,             // "userId": 42
  [Symbol.iterator]: function*() { yield 1; }, // Well-known symbol as key
};

// Dynamic method names in classes:
const methodName = "greet";
class API {
  [methodName]() { return "Hello"; }
  [`get${prefix.charAt(0).toUpperCase() + prefix.slice(1)}`]() {}
  // Creates method: "getUser"
}

// Useful pattern — creating action type constants:
const actions = {
  ["FETCH_" + "USER"]: "FETCH_USER",
  ["FETCH_" + "PRODUCTS"]: "FETCH_PRODUCTS",
};
```

**GOTCHA:** Computed property names are evaluated at the time the object literal is created. If the expression has side effects, those run once during object creation. Also, `__proto__` as a computed property name (`{ ["__proto__"]: val }`) sets the prototype in non-strict code — the behavior differs from `{ __proto__: val }`.

---

**Q17. What are the `get` and `set` accessor keywords in object literals and classes?** — Medium

**Answer:**
Getters and setters define accessor properties — properties that invoke a function when read (`get`) or written (`set`), but appear as regular properties to the outside world.

```js
class Temperature {
  #celsius;

  constructor(celsius) {
    this.#celsius = celsius;
  }

  get celsius() {
    return this.#celsius;
  }

  set celsius(value) {
    if (value < -273.15) throw new RangeError("Below absolute zero");
    this.#celsius = value;
  }

  get fahrenheit() {
    return this.#celsius * 9/5 + 32;
  }

  set fahrenheit(value) {
    this.celsius = (value - 32) * 5/9; // Goes through the celsius setter for validation
  }
}

const t = new Temperature(100);
t.celsius;         // 100 — calls getter
t.fahrenheit;      // 212 — calls getter, computed from celsius
t.fahrenheit = 32; // calls setter, which calls celsius setter
t.celsius;         // 0

t.celsius = -300;  // RangeError: Below absolute zero
```

In object literals:
```js
const obj = {
  _count: 0,
  get count() { return this._count; },
  set count(v) { if (v >= 0) this._count = v; }
};
```

Getter-only (read-only computed property):
```js
class Circle {
  constructor(r) { this.r = r; }
  get area() { return Math.PI * this.r ** 2; } // No setter — read-only
}
```

**GOTCHA:** A getter without a setter is not the same as `Object.freeze`. In sloppy mode, assigning to a getter-only property silently fails. In strict mode, it throws a `TypeError`. If you have a getter, always consider whether you need a corresponding setter and what it should validate.

---

**Q18. What are logical assignment operators and when were they added?** — Medium

**Answer:**
Logical assignment operators combine logical operators with assignment. They are short-circuit assignments added in ES2021.

`||=` (Logical OR assignment):
```js
// Assigns only if the left side is falsy:
let a = null;
a ||= "default"; // a = "default" (null is falsy)

let b = "existing";
b ||= "default"; // b = "existing" (already truthy, right side never evaluated)
```

`&&=` (Logical AND assignment):
```js
// Assigns only if the left side is truthy:
let config = { debug: false };
config.logging &&= true; // config.logging is undefined (falsy) — nothing happens

let user = { name: "Alice" };
user.name &&= user.name.toUpperCase(); // user.name = "ALICE" (truthy, so assignment runs)
```

`??=` (Nullish assignment):
```js
// Assigns only if the left side is null or undefined:
let setting = 0;
setting ??= "default"; // setting = 0 (0 is not nullish)

let missing;
missing ??= "default"; // missing = "default" (undefined is nullish)
```

Short-circuit behavior — the right side is NOT evaluated if the condition is not met:
```js
let calls = 0;
let x = "existing";
x ||= (++calls, "new value"); // calls is still 0 — right side not evaluated
```

**GOTCHA:** `a ||= b` is NOT equivalent to `a = a || b`. The assignment only happens if the left side is falsy. `a || b` always evaluates both, but only `a = a || b` always runs the assignment. The short-circuit means `a ||= b` avoids the assignment setter being called when not needed — relevant for objects with setters.

---

**Q19. What is object shorthand and method shorthand?** — Easy

**Answer:**
ES6 introduced shorthand syntax for object properties and methods, making object literals more concise.

Property shorthand — when variable name matches property name:
```js
const name = "Alice";
const age = 30;

// Old:
const user = { name: name, age: age };

// Shorthand:
const user = { name, age }; // equivalent
```

Method shorthand — omit `function` keyword:
```js
// Old:
const obj = {
  greet: function() { return "hello"; },
  async fetchData: async function() { },
  *generate: function*() { }
};

// Shorthand:
const obj = {
  greet() { return "hello"; },
  async fetchData() { },
  *generate() { }
};
```

Key behavioral difference between shorthand methods and regular functions:
- Shorthand methods `{ greet() {} }` cannot be used as constructors (`new obj.greet()` throws TypeError) — they have no `[[Construct]]`
- Regular function properties `{ greet: function() {} }` CAN be used as constructors

**Follow-up:** Does method shorthand affect `this` binding?

No — method shorthand methods are still regular functions in terms of `this` binding. `this` is determined by how they are called, not by the shorthand syntax. Only arrow functions have lexically bound `this`.

**GOTCHA:** Object shorthand does NOT work in destructuring syntax to create new variables with different names. `const { name: name }` and `const { name }` are equivalent. But `const { name: firstName }` creates a variable named `firstName` with the value of `name` — that is NOT shorthand, it is renaming.

---

**Q20. What is `String.raw` and how does it work as a tag function?** — Medium

**Answer:**
`String.raw` is a built-in tag function for template literals that returns the raw string content — exactly what you typed, without processing escape sequences.

```js
// Without String.raw — escape sequences are processed:
console.log("Line 1\nLine 2"); // Actual newline between the two lines
console.log(`C:\Users\name`);   // C:\Users (with actual newline before "name"... wait)
// Actually \n and \U are not recognized so it prints literally in this case

// With String.raw — no escape processing:
String.raw`Line 1\nLine 2`; // "Line 1\nLine 2" — literal backslash n
String.raw`C:\Users\name`;  // "C:\Users\name" — no escape processing
String.raw`\u0041`;         // "\\u0041" — not converted to "A"
```

How it works internally:
```js
function String_raw(strings, ...values) {
  // strings.raw is an array of raw (unprocessed) string parts
  return strings.raw.reduce((acc, str, i) => {
    return acc + (values[i - 1] ?? "") + str;
  });
}
```

Practical uses:
- Windows file paths: `String.raw`C:\Users\Alice\Documents``
- Regular expression strings: `String.raw`\d+\.\d+`` — no need to double-escape
- LaTeX strings
- Any context where backslashes should be literal

**GOTCHA:** `String.raw` still processes interpolated expressions (`${...}`) — only the literal backslash sequences in the static parts are unprocessed. So `String.raw`\n${"hello"}`` produces `"\\nhello"` — the `\n` is literal but the interpolation still evaluates.

---

*Next: [04-Iterators-Iterables.md](./04-Iterators-Iterables.md)*
