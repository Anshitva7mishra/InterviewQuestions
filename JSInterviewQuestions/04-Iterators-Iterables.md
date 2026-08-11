# 04 — Iterators & Iterables Protocol
### 15 Questions | Intermediate

---

**Q1. What is the iterator protocol? Define it precisely.** — Medium

**Answer:**
The iterator protocol is an interface contract defined in the ECMAScript specification. An object conforms to the iterator protocol if it has a `next()` method that, when called with zero or one argument, returns an object conforming to the IteratorResult interface.

IteratorResult interface:
- `value`: Any JavaScript value. When `done` is `true`, this is the return value of the generator/iterator; when `done` is `false`, this is the yielded value.
- `done`: A boolean. `false` means more values are available. `true` means the iterator is exhausted.

```js
// Minimal valid iterator:
const manualIterator = {
  count: 0,
  next() {
    this.count++;
    if (this.count <= 3) {
      return { value: this.count, done: false };
    }
    return { value: undefined, done: true };
  }
};

manualIterator.next(); // { value: 1, done: false }
manualIterator.next(); // { value: 2, done: false }
manualIterator.next(); // { value: 3, done: false }
manualIterator.next(); // { value: undefined, done: true }
```

Optional iterator methods:
- `return(value)`: Called when iteration is terminated early (via `break`, `return`, or `throw` inside `for...of`). Allows the iterator to perform cleanup. Should return `{ done: true, value }`.
- `throw(error)`: Passes an error into the iterator (used by generators). Should return an IteratorResult.

**Spec Reference:** ECMAScript section 7.4 — Operations on Iterator Objects

**Follow-up:** What happens when you use `break` inside `for...of`?

The loop calls `iterator.return()` if it exists. This allows iterators that hold external resources (open file handles, network connections) to clean up properly. Generator objects implement `return()` — calling it causes the generator to execute any finally blocks and then terminate.

**GOTCHA:** Many built-in iterators are "one-shot" — once exhausted, they never produce values again. If you call `Symbol.iterator` again on the same iterator object (not the original iterable), you just get the same exhausted iterator. A fresh iterator requires calling `Symbol.iterator` on the original iterable.

---

**Q2. What is the iterable protocol? How does it differ from the iterator protocol?** — Medium

**Answer:**
The iterable protocol states that an object is iterable if it has a method at the key `Symbol.iterator` that, when called with no arguments, returns an object conforming to the iterator protocol.

The distinction:
- An **iterable** is an object you can iterate over — it produces iterators on demand. Examples: Array, String, Map, Set, generator functions.
- An **iterator** is the stateful object that tracks where you are in the iteration. It has `next()`.

An object can be both (self-referential iterator):
```js
// Iterable: has [Symbol.iterator]
// Iterator: has next()
// Self-referential: [Symbol.iterator]() returns itself

const range = {
  from: 1,
  to: 5,
  current: 1,

  [Symbol.iterator]() {
    return this; // returns itself — it IS the iterator
  },

  next() {
    if (this.current <= this.to) {
      return { value: this.current++, done: false };
    }
    return { value: undefined, done: true };
  }
};

for (const n of range) console.log(n); // 1, 2, 3, 4, 5
// But: range is now exhausted! Its 'current' state persists.
for (const n of range) console.log(n); // Nothing — already exhausted
```

Better design — separate iterable and iterator:
```js
const range = {
  from: 1,
  to: 5,

  [Symbol.iterator]() {
    let current = this.from;
    const to = this.to;
    return {
      next() {
        return current <= to
          ? { value: current++, done: false }
          : { value: undefined, done: true };
      }
    };
  }
};

for (const n of range) console.log(n); // 1-5
for (const n of range) console.log(n); // 1-5 again — fresh iterator each time
```

**Spec Reference:** ECMAScript section 7.4.1 — GetIterator

**GOTCHA:** Generator objects are both iterable AND iterator — `gen[Symbol.iterator]()` returns `gen` itself. This is why you can use a generator object directly in `for...of` or spread — it is both. But it also means a generator object is one-shot — once exhausted via `for...of`, it cannot be reset.

---

**Q3. What does `for...of` actually do under the hood?** — Medium

**Answer:**
`for...of` is syntactic sugar over the iterator protocol. It desugars to approximately:

```js
// for (const item of iterable) { body }
// Desugars to:

const iterator = iterable[Symbol.iterator]();
let result;

try {
  while (true) {
    result = iterator.next();
    if (result.done) break;
    const item = result.value;
    // body runs here
  }
} catch (err) {
  if (iterator.throw) {
    iterator.throw(err); // pass error into generator
  } else {
    throw err;
  }
} finally {
  if (!result.done && iterator.return) {
    iterator.return(); // cleanup on early exit
  }
}
```

This is why `for...of` works on anything with `Symbol.iterator` — including custom objects — and why `break`/`return`/`throw` inside the loop properly calls `iterator.return()` for cleanup.

```js
function* resource() {
  try {
    yield 1;
    yield 2;
    yield 3;
  } finally {
    console.log("Cleanup!"); // runs when iterator.return() is called
  }
}

const gen = resource();
for (const v of gen) {
  console.log(v); // 1
  break;          // triggers iterator.return() -> "Cleanup!" is logged
}
// Output: 1, then "Cleanup!"
```

**Follow-up:** Why can you not use `for...of` on a plain object?

Plain objects do not have `Symbol.iterator` defined. The spec did not add it to avoid ambiguity — should it iterate keys? values? entries? With arrays and strings, the default is clear (values). With plain objects, there is no single obvious default. Use `Object.entries(obj)`, `Object.keys(obj)`, or `Object.values(obj)` to get an iterable.

**GOTCHA:** `for...of` on a `Map` iterates `[key, value]` pairs (entries), NOT just values. `for (const [k, v] of map)` is the typical pattern. Iterating `for (const v of map)` gives you arrays `[key, value]` which may be unexpected.

---

**Q4. How does `yield*` work with delegation in generators?** — Hard

**Answer:**
`yield*` delegates to another iterable (or iterator). It iterates the delegated iterable, forwarding each value as a `yield` from the outer generator. When the delegated iterable is exhausted, `yield*` evaluates to the iterable's return value (the final `{ done: true, value }` result).

```js
function* inner() {
  yield "a";
  yield "b";
  return "inner done"; // This is the return value of the inner generator
}

function* outer() {
  yield 1;
  const result = yield* inner(); // delegates to inner; result = "inner done"
  yield 2;
  console.log("inner returned:", result); // "inner returned: inner done"
}

[...outer()]; // [1, "a", "b", 2]
// Note: "inner done" is NOT in the output — return values are not yielded
```

`yield*` works with any iterable:
```js
function* all() {
  yield* [1, 2, 3];    // yields 1, 2, 3
  yield* "hello";      // yields "h", "e", "l", "l", "o"
  yield* new Set([7, 8]); // yields 7, 8
}
```

Two-way communication through `yield*`:
```js
function* adder() {
  let total = 0;
  while (true) {
    const n = yield total;
    if (n === null) return total;
    total += n;
  }
}

function* proxy() {
  return yield* adder(); // passes .next(value) calls through to adder
}

const gen = proxy();
gen.next();    // { value: 0, done: false } — start
gen.next(5);   // { value: 5, done: false } — add 5
gen.next(3);   // { value: 8, done: false } — add 3
gen.next(null);// { value: 8, done: true }  — return total
```

**GOTCHA:** The return value of `yield*` is the return value of the DELEGATED generator — not any of the yielded values. Return values are distinguished from yielded values because `done: true` results are not produced as iterable items. This is a subtle but important distinction.

---

**Q5. What is the async iterator protocol and `for await...of`?** — Hard

**Answer:**
The async iterator protocol is the async counterpart to the sync iterator protocol. An async iterator has a `next()` method that returns a **Promise** resolving to `{ value, done }` instead of returning `{ value, done }` synchronously.

An async iterable has `Symbol.asyncIterator` instead of (or in addition to) `Symbol.iterator`.

`for await...of` consumes async iterables:
```js
// for await (const item of asyncIterable) { body }
// Desugars to approximately:

const iterator = asyncIterable[Symbol.asyncIterator]();
while (true) {
  const { value, done } = await iterator.next();
  if (done) break;
  // body with item = value
}
```

Custom async iterable example:
```js
async function* countdown(from) {
  for (let i = from; i >= 0; i--) {
    await new Promise(resolve => setTimeout(resolve, 500)); // wait 500ms
    yield i;
  }
}

async function run() {
  for await (const n of countdown(5)) {
    console.log(n); // 5, 4, 3, 2, 1, 0 — one per 500ms
  }
}
```

Paginated API example — the canonical use case:
```js
async function* fetchAllPages(url) {
  let nextUrl = url;
  while (nextUrl) {
    const response = await fetch(nextUrl);
    const data = await response.json();
    yield data.items;
    nextUrl = data.nextPage || null;
  }
}

for await (const items of fetchAllPages("/api/products")) {
  renderItems(items);
}
```

**Spec Reference:** ECMAScript section 27.6 — Async Iterator Objects

**Follow-up:** Can `for await...of` iterate sync iterables?

Yes. `for await...of` works on both sync and async iterables. For sync iterables, it calls `Symbol.iterator`, gets a sync iterator, calls `.next()`, and wraps the result in `Promise.resolve()`. This means a small overhead of one microtask per item compared to `for...of`, but it works.

**GOTCHA:** `for await...of` requires an `async` function context — you cannot use it at the top level of a non-module script. In ES module files, top-level `await` allows `for await...of` directly at the module level.

---

**Q6. What is an infinite iterator and how do you safely consume one?** — Medium

**Answer:**
An infinite iterator is one that never sets `done: true` — it yields values indefinitely. It is safe to use only with consumers that break out early (`for...of` with `break`, `take(n)` utilities, destructuring).

```js
// Infinite counter:
function* naturals(start = 0) {
  let n = start;
  while (true) yield n++;
}

// Safe consumption patterns:
const gen = naturals();

// 1. Destructuring with exact count:
const [a, b, c] = naturals(); // a=0, b=1, c=2 — generator is NOT exhausted

// 2. for...of with break:
for (const n of naturals()) {
  if (n > 5) break;
  console.log(n); // 0,1,2,3,4,5
}

// 3. Utility: take N items:
function take(iterable, n) {
  const result = [];
  for (const item of iterable) {
    result.push(item);
    if (result.length >= n) break;
  }
  return result;
}
take(naturals(), 5); // [0,1,2,3,4]

// 4. Fibonacci infinite:
function* fibonacci() {
  let [a, b] = [0, 1];
  while (true) {
    yield a;
    [a, b] = [b, a + b];
  }
}

take(fibonacci(), 10); // [0,1,1,2,3,5,8,13,21,34]
```

**GOTCHA:** Never use `[...infiniteIterator]` or `Array.from(infiniteIterator)` without a mapping function — these consume the iterator to completion and will hang indefinitely (or until memory is exhausted). `Array.from({ length: 5 }, () => gen.next().value)` is a safe way to take a fixed number.

---

**Q7. How do you implement a lazy pipeline using generators?** — Hard

**Answer:**
Generators enable lazy evaluation — values are produced only when requested. This allows you to build processing pipelines over large or infinite data sets without loading everything into memory at once.

```js
// Lazy pipeline operators:
function* map(iterable, fn) {
  for (const item of iterable) yield fn(item);
}

function* filter(iterable, predicate) {
  for (const item of iterable) {
    if (predicate(item)) yield item;
  }
}

function* take(iterable, n) {
  let count = 0;
  for (const item of iterable) {
    if (count++ >= n) return;
    yield item;
  }
}

function* naturals() {
  let n = 1;
  while (true) yield n++;
}

// Lazy pipeline — processes only what is consumed:
const result = take(
  filter(
    map(naturals(), n => n * n),  // squares
    n => n % 2 === 0              // even squares
  ),
  5                               // only first 5
);

[...result]; // [4, 16, 36, 64, 100]
// Critically: only 10 numbers (1-10) were ever generated from naturals()
// No intermediate arrays were created — fully lazy
```

Composable pipeline using pipe:
```js
function pipe(...fns) {
  return function*(source) {
    let current = source;
    for (const fn of fns) current = fn(current);
    yield* current;
  };
}

const process = pipe(
  src => map(src, x => x * 2),
  src => filter(src, x => x > 5),
  src => take(src, 3)
);

[...process(naturals())]; // [6, 8, 10]
```

**Follow-up:** What is the advantage of lazy pipelines over array method chains?

Array method chains (`arr.map(...).filter(...).slice(...)`) create intermediate arrays at each step — if the input array has 1 million items, `.map()` creates 1 million items, then `.filter()` creates another array, etc. Lazy generators process one item at a time through the entire pipeline — no intermediate arrays, constant memory usage regardless of source size.

**GOTCHA:** Generators are stateful and one-shot. Once consumed, the pipeline cannot be re-run without recreating all the generators. This contrasts with array chains where you can call the chain again. If you need to consume a lazy pipeline multiple times, wrap it in a function.

---

**Q8. What is the difference between `return()` and `throw()` on a generator?** — Hard

**Answer:**
Generator objects have three control methods beyond `.next()`:

`.return(value)`: Terminates the generator and makes it return the given value. Triggers `finally` blocks in the generator.
```js
function* gen() {
  try {
    yield 1;
    yield 2;
    yield 3;
  } finally {
    console.log("finally ran"); // Always runs on return() or throw()
  }
}

const g = gen();
g.next();           // { value: 1, done: false }
g.return("done");   // logs "finally ran", returns { value: "done", done: true }
g.next();           // { value: undefined, done: true } — already terminated
```

`.throw(error)`: Injects an error into the generator at the current suspension point. If the generator has a `try/catch`, it can handle it and continue.
```js
function* withErrorHandling() {
  try {
    const value = yield "waiting";
    console.log("got:", value);
  } catch (err) {
    console.log("caught:", err.message);
    yield "recovered"; // can continue after catching
  }
}

const g = withErrorHandling();
g.next();                      // { value: "waiting", done: false }
g.throw(new Error("oops"));    // logs "caught: oops", { value: "recovered", done: false }
g.next();                      // { value: undefined, done: true }
```

If `.throw()` is called and the generator has no handler, the error propagates to the caller:
```js
function* bare() { yield 1; }
const g = bare();
g.next();
g.throw(new Error("boom")); // Error propagates — not caught inside generator
```

**GOTCHA:** The `for...of` loop calls `.return()` on the iterator when it exits early (via `break`, `return`, or `throw` from within the loop body). This ensures that generators always get a chance to run their `finally` blocks even when iteration is terminated early. Custom iterators should implement `.return()` if they hold any resources that need cleanup.

---

**Q9. How do you make a class iterable and why might you make it an async iterable?** — Medium

**Answer:**
Make a class iterable by implementing `[Symbol.iterator]()` as a method. Make it async iterable by implementing `[Symbol.asyncIterator]()`.

```js
// Sync iterable — in-memory data:
class NumberRange {
  constructor(start, end, step = 1) {
    this.start = start;
    this.end = end;
    this.step = step;
  }

  [Symbol.iterator]() {
    let current = this.start;
    const { end, step } = this;
    return {
      next() {
        if (current <= end) {
          const value = current;
          current += step;
          return { value, done: false };
        }
        return { value: undefined, done: true };
      },
      [Symbol.iterator]() { return this; } // self-referential for use as iterator too
    };
  }
}

[...new NumberRange(1, 10, 2)]; // [1,3,5,7,9]

// Async iterable — data that requires async work:
class DatabaseCursor {
  constructor(query) {
    this.query = query;
  }

  async *[Symbol.asyncIterator]() {
    let offset = 0;
    const batchSize = 100;

    while (true) {
      const rows = await db.query(`${this.query} LIMIT ${batchSize} OFFSET ${offset}`);
      if (rows.length === 0) return;
      for (const row of rows) yield row;
      offset += batchSize;
      if (rows.length < batchSize) return; // last batch
    }
  }
}

// Consumer:
const cursor = new DatabaseCursor("SELECT * FROM users");
for await (const user of cursor) {
  processUser(user);
}
```

**Follow-up:** Why does the iterator returned by `[Symbol.iterator]()` also implement `[Symbol.iterator]()` returning itself?

This makes the returned iterator also an iterable, which means it can be passed to `for...of` directly, spread, or any other iterable consumer. Without this, if you stored the iterator (`const it = range[Symbol.iterator]()`) and then tried `for (const v of it)`, it would fail because `it` would not be iterable. Many consumers internally call `Symbol.iterator` on their input.

**GOTCHA:** If your class implements both `Symbol.iterator` AND `Symbol.asyncIterator`, `for...of` uses the sync one and `for await...of` prefers the async one. If only the async one is defined, `for...of` (sync) will fail — they are separate protocols.

---

**Q10. What is the `entries()`, `keys()`, and `values()` pattern and why do arrays, maps, and strings all implement it?** — Easy

**Answer:**
`entries()`, `keys()`, and `values()` are standardized methods that return iterators over the respective views of a collection. They return lazy iterators, not arrays.

```js
// Array:
const arr = ["a", "b", "c"];
[...arr.keys()];    // [0, 1, 2]
[...arr.values()];  // ["a", "b", "c"] — same as [...arr]
[...arr.entries()]; // [[0,"a"], [1,"b"], [2,"c"]]

// Common pattern — iterate with index:
for (const [index, value] of arr.entries()) {
  console.log(index, value);
}

// Map:
const map = new Map([["x", 1], ["y", 2]]);
[...map.keys()];    // ["x", "y"]
[...map.values()];  // [1, 2]
[...map.entries()]; // [["x",1],["y",2]] — same as [...map]

// String:
const str = "hi";
[...str.keys()];    // [0, 1] — indices
[...str.values()];  // ["h", "i"] — characters
[...str.entries()]; // [[0,"h"], [1,"i"]]
```

`Object.keys()`, `Object.values()`, `Object.entries()` — note these are static methods on `Object`, not on the object instance, and they return arrays (not iterators) of own, enumerable string-keyed properties.

**GOTCHA:** `Array.prototype.values()` was not available in some older environments and required a polyfill. In Node.js, it was not available until later versions. `[...arr]` and `arr[Symbol.iterator]()` are equivalent alternatives. Today it is universally supported.

---

**Q11. What is the `Symbol.iterator` protocol on strings and why does it handle Unicode correctly?** — Medium

**Answer:**
Strings implement `Symbol.iterator` to iterate over Unicode code points (characters), not UTF-16 code units. This is the critical distinction for handling non-BMP characters (emoji, many CJK extension characters, mathematical symbols).

JavaScript strings are internally stored as UTF-16 sequences. Characters above U+FFFF require two UTF-16 code units called a surrogate pair. Index-based string access (`str[i]`) gives you individual code units, which means accessing a surrogate pair by index gives you broken half-characters.

The `Symbol.iterator` protocol knows about surrogate pairs and yields complete code points:

```js
const emoji = "a\u{1F600}b"; // "a😀b" — emoji is 2 code units wide

// Index access — BROKEN for non-BMP:
emoji.length;   // 4 — counts code units, not code points
emoji[0];       // "a"
emoji[1];       // "\uD83D" — broken high surrogate
emoji[2];       // "\uDE00" — broken low surrogate
emoji[3];       // "b"

// for...of — CORRECT:
for (const char of emoji) {
  console.log(char);
}
// "a"
// "😀" — the full emoji as one unit
// "b"

// Spread also uses Symbol.iterator:
[...emoji];           // ["a", "😀", "b"] — 3 elements, correct
emoji.split("");      // ["a", "\uD83D", "\uDE00", "b"] — 4 elements, WRONG

// Correct string reversal:
const reversed = [...emoji].reverse().join(""); // "b😀a"
// vs. wrong:
emoji.split("").reverse().join(""); // broken — reverses code units, not code points
```

**Follow-up:** What is `Array.from(str)` vs `[...str]`?

Both use `Symbol.iterator` on the string and produce the same correct code-point-per-element array. `Array.from` additionally accepts a mapping function as the second argument, making it more flexible for transformation.

**GOTCHA:** `str.length` counts UTF-16 code units, not Unicode code points. A string with one emoji has `length === 2`. To get the true character count: `[...str].length` or `str.codePointAt(0)`. The newer `Intl.Segmenter` API (ES2022) handles grapheme clusters (visually single characters that may be multiple code points, like flag emojis).

---

**Q12. What is a "closeable" iterator and how do you write one?** — Hard

**Answer:**
A closeable iterator is one that implements the optional `return(value)` method. This method is called when a consumer terminates iteration early — via `break`, `return`, or an uncaught exception inside `for...of`. It gives the iterator a chance to release resources.

```js
// A "file reader" iterator that cleans up on early exit:
function createFileIterator(filename) {
  const handle = openFile(filename); // hypothetical sync file handle

  return {
    [Symbol.iterator]() { return this; },

    next() {
      const line = handle.readLine();
      if (line === null) {
        handle.close();
        return { value: undefined, done: true };
      }
      return { value: line, done: false };
    },

    return(value) {
      // Called on early termination — must close the handle
      handle.close();
      return { value, done: true }; // spec: must return an IteratorResult
    }
  };
}

const fileIter = createFileIterator("data.txt");
for (const line of fileIter) {
  if (line.startsWith("STOP")) break; // triggers return() for cleanup
  process(line);
}
// File handle is closed even though we broke out early
```

Generators implement `return()` automatically via their `finally` block:
```js
function* fileLines(filename) {
  const handle = openFile(filename);
  try {
    let line;
    while ((line = handle.readLine()) !== null) {
      yield line;
    }
  } finally {
    handle.close(); // Runs on: normal completion, break, return, or throw
  }
}
```

**GOTCHA:** The `return()` method MUST return an object with `done` and `value` properties, or a TypeError is thrown. The `finally` block behavior in generators makes them the safest way to implement closeable iterators — `return()` is handled for you automatically.

---

**Q13. How does destructuring use the iterator protocol?** — Medium

**Answer:**
Array destructuring uses the iterator protocol, not direct index access. It calls `Symbol.iterator()` on the right-hand side, then calls `.next()` for each binding position.

```js
// This works on ANY iterable, not just arrays:
const [a, b, c] = new Set([1, 2, 3]);     // a=1, b=2, c=3
const [x, y] = "hello";                    // x="h", y="e"
const [first, second] = new Map([["a",1],["b",2]]); // first=["a",1], second=["b",2]

function* gen() { yield 10; yield 20; yield 30; }
const [p, q, r] = gen();  // p=10, q=20, r=30

// Skipping with holes:
const [, second2, , fourth] = gen(); // calls next() 4 times, discards values 1 and 3
```

Rest in destructuring calls the iterator to completion:
```js
const [head, ...tail] = gen(); // head=10, tail=[20, 30]
// tail captures all remaining values
```

Destructuring with too few values:
```js
const [a2, b2, c2, d2] = [1, 2];
// a2=1, b2=2, c2=undefined, d2=undefined
// next() is called 4 times; last two return { done: true }
// Variables for done iterators get value undefined
```

Default values in array destructuring:
```js
const [x2 = 0, y2 = 0] = [1];
// x2=1 (from iterator), y2=0 (default, since iterator was done)
```

**GOTCHA:** Object destructuring does NOT use the iterator protocol — it uses property access (ToObject + [[Get]]). Only array destructuring patterns use iterators. `const { a } = someMap` would try to access `someMap.a`, not iterate the Map. `const [a] = someMap` would use the Map's `Symbol.iterator`.

---

**Q14. What happens when you spread an iterable vs an array-like?** — Medium

**Answer:**
Spread (`...`) and `for...of` use the iterator protocol (`Symbol.iterator`). They do NOT work on array-like objects that lack `Symbol.iterator`.

An array-like object has a `length` property and numeric indices but no `Symbol.iterator`:
```js
const arrayLike = { 0: "a", 1: "b", 2: "c", length: 3 };

// This fails — no Symbol.iterator:
[...arrayLike]; // TypeError: arrayLike is not iterable

// But this works — Array.from handles array-likes:
Array.from(arrayLike); // ["a", "b", "c"]

// The arguments object is array-like AND iterable (in modern JS):
function f() {
  [...arguments]; // Works — arguments has Symbol.iterator
  Array.from(arguments); // Also works
}

// DOM NodeList — is iterable in modern browsers:
[...document.querySelectorAll("div")]; // Works

// String — iterable:
[..."hello"]; // ["h","e","l","l","o"] — uses Symbol.iterator, handles Unicode
```

`Array.from` handles both:
- If the argument has `Symbol.iterator`, it uses the iterator protocol.
- If it only has `length` + indices (array-like), it uses index-based access.

```js
Array.from({ length: 5 }, (_, i) => i * 2); // [0,2,4,6,8]
// Second argument is a map function, applied to each element
```

**GOTCHA:** Adding `Symbol.iterator` to an object retroactively makes it spreadable and `for...of`-able. This is how `Map`, `Set`, `String`, and custom iterables all work with spread. But there is no way to make an object work with spread without implementing `Symbol.iterator`.

---

**Q15. What is `Iterator.prototype` and the proposed Iterator helpers?** — Hard

**Answer:**
In current JavaScript, iterators from different sources (arrays, maps, generators) do not share a common helper API. You cannot do `generator.map(fn)` or `generator.filter(fn)` — you have to write utility functions manually or consume to an array first.

The **Iterator Helpers proposal** (Stage 4, going into ES2025) adds a shared `Iterator.prototype` with built-in methods, making all iterators composable like arrays:

```js
// With Iterator Helpers (ES2025):
function* naturals() {
  let n = 0;
  while (true) yield n++;
}

naturals()
  .map(n => n * n)         // lazy map over generator
  .filter(n => n % 2 === 0) // lazy filter
  .take(5)                  // take first 5
  .toArray();               // materialize: [0, 4, 16, 36, 64]

// Available methods on Iterator.prototype:
// .map(fn)         — transform values
// .filter(fn)      — skip values
// .take(n)         — take first n
// .drop(n)         — skip first n
// .flatMap(fn)     — flatten one level
// .reduce(fn, init) — fold to single value
// .forEach(fn)     — side effects
// .some(fn)        — short-circuit any
// .every(fn)       — short-circuit all
// .find(fn)        — first matching
// .toArray()       — materialize to array
// .from(iterable) — static: convert any iterable to Iterator
```

Until full support, use a polyfill or manual utility functions:
```js
// Current way — utility functions:
function* map(iter, fn) { for (const x of iter) yield fn(x); }
function* filter(iter, fn) { for (const x of iter) { if (fn(x)) yield x; } }
function* take(iter, n) { let i = 0; for (const x of iter) { if (i++ >= n) return; yield x; } }
```

**GOTCHA:** Iterator Helpers are lazy — they return new iterators, not arrays. The computation only happens when you pull values. `.toArray()` is what materializes the result. This is different from array methods like `.map()` which eagerly produce a new array immediately.

---

*Next: [05-Advanced-Async.md](./05-Advanced-Async.md)*
