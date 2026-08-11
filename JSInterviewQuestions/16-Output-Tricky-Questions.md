# 16 — Output-Based / Tricky Questions
### 25 Questions | Intermediate

---

**Q1. What is the output?** — Easy

```js
console.log(typeof null);
console.log(typeof undefined);
console.log(null == undefined);
console.log(null === undefined);
```

**Answer:** `"object"`, `"undefined"`, `true`, `false`

`typeof null === "object"` is a legacy bug from the first version of JavaScript — null's type tag was 0, same as object references. `null == undefined` is `true` because the spec defines that `null` and `undefined` are equal only to each other under abstract equality. `null === undefined` is `false` because they are different types.

---

**Q2. What is the output?** — Easy

```js
console.log(0.1 + 0.2 === 0.3);
console.log(0.1 + 0.2);
```

**Answer:** `false`, `0.30000000000000004`

IEEE 754 double-precision floating point cannot represent 0.1 or 0.2 exactly. The sum has accumulated rounding error. To compare floats: `Math.abs(0.1 + 0.2 - 0.3) < Number.EPSILON`.

---

**Q3. What is the output?** — Medium

```js
var x = 1;
function outer() {
  var x = 2;
  function inner() {
    console.log(x);
    var x = 3;
  }
  inner();
}
outer();
```

**Answer:** `undefined`

Inside `inner()`, `var x = 3` is hoisted. The declaration is hoisted to the top of `inner()`, but not the assignment. So when `console.log(x)` runs, `x` exists in `inner()`'s scope (due to hoisting) but is `undefined` (the assignment hasn't run yet). The outer `x = 2` is shadowed.

---

**Q4. What is the output?** — Medium

```js
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 0);
}
```

**Answer:** `3`, `3`, `3`

`var` is function-scoped, not block-scoped. All three callbacks close over the SAME `i` variable. By the time the callbacks run (after the loop completes), `i` is `3`. Fix: use `let` (block-scoped, creates new binding per iteration), or use an IIFE `((j) => setTimeout(() => console.log(j), 0))(i)`.

---

**Q5. What is the output?** — Medium

```js
const obj = { a: 1 };
const copy = obj;
copy.a = 99;
console.log(obj.a);
```

**Answer:** `99`

Objects are assigned by reference. `copy` and `obj` point to the same object in memory. Mutating `copy.a` mutates the shared object. To make a true copy: `const copy = { ...obj }` (shallow) or `structuredClone(obj)` (deep).

---

**Q6. What is the output?** — Hard

```js
function foo() {
  console.log(this.name);
}

const obj1 = { name: "Alice", foo };
const obj2 = { name: "Bob" };

obj1.foo();           // ?
obj1.foo.call(obj2);  // ?
const fn = obj1.foo;
fn();                 // ?
```

**Answer:** `"Alice"`, `"Bob"`, `undefined` (or error in strict mode)

Method invocation: `this` is the object before the dot → Alice. `call(obj2)` explicitly sets `this` → Bob. When assigned to a plain variable and called, `this` is the global object (`window.name` is `""` in browsers, `undefined` in Node's strict mode).

---

**Q7. What is the output?** — Hard

```js
console.log(1 + "2" + 3);
console.log(1 + 2 + "3");
console.log("5" - 3);
console.log("5" * "3");
console.log(true + true);
console.log([] + []);
console.log({} + []);
```

**Answer:** `"123"`, `"33"`, `2`, `15`, `2`, `""`, `"[object Object]"` (or `0` if `{}` is parsed as empty block)

- `1 + "2"` → `"12"` (string coercion), then `"12" + 3` → `"123"`
- `1 + 2` → `3` (both numbers), then `3 + "3"` → `"33"`
- `"5" - 3` → `5 - 3 = 2` (subtraction coerces to number)
- `"5" * "3"` → `5 * 3 = 15` (multiplication coerces)
- `true + true` → `1 + 1 = 2` (booleans coerce to numbers)
- `[] + []` → `"" + "" = ""`
- `{} + []` — in a statement context, `{}` is an empty block, so `+[]` → `+""` → `0`; in expression context `({}) + []` → `"[object Object]"`

---

**Q8. What is the output?** — Hard

```js
const promise = new Promise((resolve) => {
  console.log("1");
  resolve("2");
  console.log("3");
});

promise.then(v => console.log(v));
console.log("4");
```

**Answer:** `1`, `3`, `4`, `2`

Promise executor runs synchronously (1, 3). Then `.then()` callback is a microtask — it's queued after the synchronous code completes. "4" logs next (synchronous), then the microtask queue drains (2).

---

**Q9. What is the output?** — Hard

```js
async function fetchData() {
  return 42;
}

const result = fetchData();
console.log(result);
console.log(result instanceof Promise);
```

**Answer:** `Promise { 42 }`, `true`

`async` functions ALWAYS return a Promise — even if you return a plain value. The value is wrapped in a resolved Promise. To get `42`, you need `await fetchData()` or `.then(v => console.log(v))`.

---

**Q10. What is the output?** — Medium

```js
let a = { x: 1 };
let b = a;
a = { x: 2 };
console.log(b.x);
```

**Answer:** `1`

`b` was assigned a reference to the original `{ x: 1 }` object. Reassigning `a` to `{ x: 2 }` creates a NEW object and makes `a` point to it. `b` still holds the reference to the original object. This differs from Q5 where the same object was mutated.

---

**Q11. What is the output?** — Hard

```js
console.log([] == false);
console.log([] == ![]);
console.log([] == 0);
```

**Answer:** `true`, `true`, `true`

- `[] == false` → `[] == 0` (false → 0) → `"" == 0` ([] → "") → `0 == 0` → `true`
- `![]` → `false` ([] is truthy, negated = false), then `[] == false` → `true`
- `[] == 0` → `"" == 0` → `0 == 0` → `true`

The Abstract Equality algorithm's coercion rules are notoriously complex. This is why `===` (strict equality) is always preferred.

---

**Q12. What is the output?** — Medium

```js
const arr = [1, 2, 3];
arr[10] = 11;
console.log(arr.length);
console.log(arr[5]);
```

**Answer:** `11`, `undefined`

Setting index 10 extends the array length to 11 (max index + 1). Indices 3-9 are "holes" (empty slots), not `undefined` values — though reading them returns `undefined`. `arr.length` is always max index + 1.

---

**Q13. What is the output?** — Hard

```js
function createFunctions() {
  const fns = [];
  for (let i = 0; i < 3; i++) {
    fns.push(() => i);
  }
  return fns;
}

const [f1, f2, f3] = createFunctions();
console.log(f1(), f2(), f3());
```

**Answer:** `0 1 2`

`let` in a `for` loop creates a NEW binding for each iteration. Each arrow function closes over its own unique `i`. This is the fix to the classic `var` loop problem (Q4).

---

**Q14. What is the output?** — Hard

```js
class Counter {
  count = 0;

  increment = () => { this.count++; };
  decrement() { this.count--; }
}

const c = new Counter();
const { increment, decrement } = c;

increment();
try { decrement(); } catch(e) { console.log("error"); }
console.log(c.count);
```

**Answer:** `error`, `1`

`increment` is a class field arrow function — it captures `this` lexically from the constructor context, always pointing to the instance. `decrement` is a prototype method — when destructured and called without a receiver, `this` is `undefined` in strict mode (classes are always strict), causing `TypeError`. `c.count` is `1` because `increment()` worked.

---

**Q15. What is the output?** — Medium

```js
console.log(typeof NaN);
console.log(NaN === NaN);
console.log(isNaN("hello"));
console.log(Number.isNaN("hello"));
```

**Answer:** `"number"`, `false`, `true`, `false`

`typeof NaN === "number"` — NaN is technically a number (Not-a-Number is still the Number type). `NaN !== NaN` — NaN is the only value not equal to itself. `isNaN("hello")` coerces "hello" to `NaN` first, then checks → `true`. `Number.isNaN("hello")` does NOT coerce — "hello" is not NaN, it's a string → `false`. Always use `Number.isNaN()`.

---

**Q16. What is the output?** — Hard

```js
const obj = {
  value: 42,
  getValue() {
    return this.value;
  },
  getValueArrow: () => {
    return this.value;
  }
};

console.log(obj.getValue());
console.log(obj.getValueArrow());
```

**Answer:** `42`, `undefined`

`getValue()` is a regular method — `this` is `obj` when called as `obj.getValue()`. `getValueArrow` is an arrow function defined in the object literal — at that point, `this` is the outer context (the global object or `undefined` in strict mode). Object literals do NOT create their own `this` context. In a browser, `this.value` is `window.value` which is `undefined`. This is why you should NOT use arrow functions as object methods when you need `this`.

---

**Q17. What is the output?** — Hard

```js
Promise.resolve(1)
  .then(v => v + 1)
  .then(v => { throw new Error("oops"); })
  .catch(e => { console.log("caught:", e.message); return 42; })
  .then(v => console.log("resolved:", v));
```

**Answer:** `caught: oops`, `resolved: 42`

`.then(v => { throw new Error("oops") })` — throwing inside a `.then()` callback causes the returned Promise to reject. `.catch()` handles the rejection, logs it, and returns `42`. A `.catch()` that doesn't rethrow converts a rejection back to a resolved Promise. The final `.then()` receives `42`.

---

**Q18. What is the output?** — Medium

```js
const set = new Set([1, 1, 2, 2, 3]);
console.log(set.size);
console.log([...set]);
console.log(set.has(2));
```

**Answer:** `3`, `[1, 2, 3]`, `true`

`Set` automatically deduplicates values. `size` is the number of unique elements. Spreading `set` gives an array of unique values in insertion order. `has()` checks for membership.

---

**Q19. What is the output?** — Hard

```js
let x = 1;
function outer() {
  let x = 2;
  return function inner() {
    return ++x;
  };
}
const fn = outer();
console.log(fn());
console.log(fn());
console.log(fn());
```

**Answer:** `3`, `4`, `5`

`inner` closes over `outer`'s `x = 2`. Each call to `fn()` increments and returns the SAME `x` variable (pre-increment). `++x` first increments, then returns: 2→3, 3→4, 4→5.

---

**Q20. What is the output?** — Hard

```js
class A {
  constructor() {
    console.log(new.target.name);
  }
}

class B extends A {
  constructor() {
    super();
  }
}

new A(); // ?
new B(); // ?
```

**Answer:** `"A"`, `"B"`

`new.target` inside a constructor refers to the class that was directly `new`-ed. When `new A()` is called, `new.target.name` is `"A"`. When `new B()` is called and `super()` runs (inside `A`'s constructor), `new.target` is still `B` — the originally constructed class. This allows base classes to know which subclass is being instantiated.

---

**Q21. What is the output?** — Hard

```js
const obj = {};
Object.defineProperty(obj, "x", {
  get() { return this._x ?? 0; },
  set(v) { this._x = v * 2; }
});

obj.x = 5;
console.log(obj.x);
obj.x++;
console.log(obj.x);
```

**Answer:** `10`, `21`

The setter doubles the value: `set(5)` → `_x = 10`. The getter returns `_x` → `10`. `obj.x++` is equivalent to `obj.x = obj.x + 1` → `set(10 + 1)` = `set(11)` → `_x = 22`. But wait — `obj.x` returns `_x = 22`, so `console.log(obj.x)` → `22`? Let's re-trace: `obj.x++` expands to `obj.x = obj.x + 1`. `obj.x` (get) returns `10`. So `10 + 1 = 11`. Then `obj.x = 11` (set) → `_x = 22`. So `obj.x` (get) returns `22`. **Answer: `10`, `22`.**

---

**Q22. What is the output?** — Hard

```js
function* gen() {
  const x = yield 1;
  const y = yield x + 2;
  yield y + 3;
}

const g = gen();
console.log(g.next());
console.log(g.next(10));
console.log(g.next(20));
```

**Answer:**
```
{ value: 1, done: false }
{ value: 12, done: false }
{ value: 23, done: false }
```

First `next()` — runs to first `yield 1`. Returns `{ value: 1 }`.
Second `next(10)` — resumes, `x = 10` (the argument becomes the result of `yield`). Runs to `yield x + 2` = `yield 12`. Returns `{ value: 12 }`.
Third `next(20)` — resumes, `y = 20`. Runs to `yield y + 3` = `yield 23`. Returns `{ value: 23 }`.

The key insight: the value PASSED to `next()` becomes the return value of the previous `yield` expression.

---

**Q23. What is the output?** — Hard

```js
const p1 = new Promise(resolve => setTimeout(() => resolve("p1"), 1000));
const p2 = new Promise(resolve => setTimeout(() => resolve("p2"), 500));
const p3 = new Promise((_, reject) => setTimeout(() => reject("p3"), 700));

Promise.race([p1, p2, p3]).then(console.log).catch(console.log);
```

**Answer:** `"p2"`

`Promise.race` settles with the FIRST settled promise. p2 resolves at 500ms, p3 rejects at 700ms, p1 resolves at 1000ms. p2 wins with `"p2"` → `.then(console.log)` → logs `"p2"`. p3's rejection is ignored (race is already settled).

---

**Q24. What is the output?** — Hard

```js
const symbol1 = Symbol("key");
const symbol2 = Symbol("key");
const globalSym = Symbol.for("shared");
const globalSym2 = Symbol.for("shared");

console.log(symbol1 === symbol2);
console.log(globalSym === globalSym2);
console.log(typeof symbol1);
```

**Answer:** `false`, `true`, `"symbol"`

`Symbol()` creates a UNIQUE symbol every time — even with the same description. `Symbol.for()` looks up or creates a symbol in the global registry — the same key always returns the same symbol. `typeof symbol === "symbol"`.

---

**Q25. What is the output?** — Hard

```js
class Foo {
  static bar = "static";
  bar = "instance";

  getBar() { return this.bar; }
  static getStaticBar() { return this.bar; }
}

const foo = new Foo();
console.log(foo.bar);
console.log(foo.getBar());
console.log(Foo.bar);
console.log(Foo.getStaticBar());
console.log(foo.getStaticBar?.());
```

**Answer:** `"instance"`, `"instance"`, `"static"`, `"static"`, `undefined`

Instance field `bar = "instance"` is set on each instance in the constructor. Static field `bar = "static"` lives on the class itself. `foo.bar` returns the instance field. `Foo.bar` returns the static field. `foo.getStaticBar?.()` — static methods are not on instances, so `foo.getStaticBar` is `undefined`, and `?.()` safely returns `undefined`.

---

*Next: [17-DOM-Browser.md](./17-DOM-Browser.md)*
