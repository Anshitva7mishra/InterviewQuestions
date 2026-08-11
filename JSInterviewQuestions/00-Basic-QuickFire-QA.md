# 00 — Basic JavaScript: Quick-Fire Interview Questions
### 80+ Questions | Beginner to Intermediate | Minute-to-Minute Format

> These are the questions you **will** be asked in the first 10 minutes of any JS interview.
> Every answer is short, confident, and interview-ready. Say these out loud.

---

## Variables & Types

**Q: What are the differences between `var`, `let`, and `const`?**

`var` is function-scoped, hoisted to `undefined`, and can be re-declared. `let` and `const` are block-scoped and in the Temporal Dead Zone before their declaration — `const` additionally cannot be reassigned. In modern code, I always use `const` by default and `let` when I need to reassign. I never use `var`.

---

**Q: What is `typeof` and what are its possible return values?**

`typeof` returns a string describing the type of a value. The possible values are `"undefined"`, `"boolean"`, `"number"`, `"bigint"`, `"string"`, `"symbol"`, `"function"`, and `"object"`. The well-known gotcha is `typeof null === "object"` — a historical bug that was never fixed.

---

**Q: What is the difference between `null` and `undefined`?**

`undefined` means a variable was declared but no value was assigned — it's the default uninitialized value. `null` is an intentional absence of value — you explicitly set it. `typeof undefined` is `"undefined"`, `typeof null` is `"object"`. They are `==` equal but not `===` equal.

---

**Q: What is `NaN` and how do you check for it?**

`NaN` stands for "Not a Number" — it's the result of invalid numeric operations like `"text" / 2` or `Math.sqrt(-1)`. It's the only value in JavaScript that is not equal to itself. To check for NaN, use `Number.isNaN(value)`. Avoid the global `isNaN()` because it first coerces the value to a number — `isNaN("hello")` returns `true` even though `"hello"` is a string, not `NaN`.

---

**Q: What is the difference between `==` and `===`?**

`===` (strict equality) compares value AND type — no coercion happens. `==` (abstract equality) coerces types before comparing, which leads to confusing results: `0 == false` is `true`, `"" == false` is `true`, `null == undefined` is `true`. I always use `===` unless I specifically need nullish equality (`value == null` checks for both `null` and `undefined`).

---

**Q: What are JavaScript's primitive types?**

There are 7 primitives: `string`, `number`, `bigint`, `boolean`, `undefined`, `null`, and `symbol`. Everything else is an object. Primitives are immutable — when you "modify" a string, you're creating a new one. Primitives are compared by value; objects are compared by reference.

---

**Q: What is type coercion?**

Type coercion is JavaScript's automatic conversion of values from one type to another. It happens implicitly in operations like `1 + "2"` (→ `"12"`, number coerced to string) and `"5" * 3` (→ `15`, string coerced to number). Explicit coercion is when you do it yourself with `Number()`, `String()`, `Boolean()`. Understanding coercion rules is essential for avoiding bugs with `==` and arithmetic operators.

---

**Q: What values are falsy in JavaScript?**

There are exactly 8 falsy values: `false`, `0`, `-0`, `0n` (BigInt zero), `""` (empty string), `null`, `undefined`, and `NaN`. Everything else is truthy — including `"0"`, `[]`, `{}`, and `function(){}`.

---

## Functions

**Q: What is a closure?**

A closure is a function that retains access to variables from its outer scope even after the outer function has finished executing. It captures a live reference to the outer lexical environment, not a snapshot. Closures are used for data privacy, memoization, partial application, and in React hooks.

```js
function makeCounter() {
  let count = 0;
  return () => ++count; // closes over `count`
}
const counter = makeCounter();
counter(); // 1
counter(); // 2
```

---

**Q: What is the difference between a function declaration and a function expression?**

A function declaration (`function foo() {}`) is fully hoisted — you can call it before it appears in the code. A function expression (`const foo = function() {}`) is not hoisted — you can only call it after the assignment. Arrow functions are also expressions. I prefer expressions with `const` to avoid accidental hoisting surprises.

---

**Q: What is an arrow function and how does it differ from a regular function?**

Arrow functions have a shorter syntax and differ in three key ways: (1) they do not have their own `this` — they inherit `this` from the enclosing scope; (2) they cannot be used as constructors — `new (() => {})` throws; (3) they have no `arguments` object. Use arrow functions for callbacks and methods that don't need their own `this`. Avoid them as object methods or constructors.

---

**Q: What does `this` refer to in JavaScript?**

`this` depends on the call site — how and where a function is called, not where it's defined.

- Called as a method (`obj.fn()`): `this` = the object
- Called as a plain function (`fn()`): `this` = global object (or `undefined` in strict mode)
- With `call`/`apply`/`bind`: `this` = the explicitly provided value
- Arrow function: `this` = the enclosing lexical scope's `this`
- Constructor (`new Fn()`): `this` = the new instance

---

**Q: What is `bind()`, `call()`, and `apply()`?**

All three explicitly set `this` for a function. `call(thisArg, arg1, arg2)` invokes immediately with arguments listed. `apply(thisArg, [arg1, arg2])` invokes immediately with arguments as an array. `bind(thisArg, arg1)` returns a NEW function with `this` permanently bound — does not invoke immediately. I use `bind` for event handlers and callbacks where `this` would otherwise be lost.

---

**Q: What is the difference between rest parameters and the `arguments` object?**

`arguments` is an array-like object (not a real array) available in non-arrow functions — it contains all arguments passed. Rest parameters (`...args`) are a real `Array` containing only the parameters that weren't explicitly named. Rest parameters are cleaner, work in arrow functions, and support array methods directly. I always prefer rest parameters.

---

**Q: What is `IIFE` (Immediately Invoked Function Expression)?**

An IIFE is a function that is defined and immediately called. It creates a private scope that doesn't pollute the global namespace. Before ES6 modules, it was the main way to create encapsulation. Today, modules handle this, but IIFEs still appear in some patterns.

```js
(function() {
  const private = "not accessible outside";
})();
```

---

**Q: What is currying?**

Currying transforms a function that takes multiple arguments into a series of functions, each taking one argument. Instead of `add(1, 2)`, a curried version is `add(1)(2)`. It enables partial application — creating specialized functions from general ones.

```js
const multiply = a => b => a * b;
const double = multiply(2);
double(5); // 10
```

---

## Scope & Hoisting

**Q: What is hoisting?**

Hoisting is the process where the JavaScript engine, during the creation phase of an execution context, registers variable and function declarations before any code runs. `var` declarations are hoisted and initialized to `undefined`. Function declarations are fully hoisted (name + body). `let` and `const` declarations are hoisted but NOT initialized — they are in the Temporal Dead Zone until the declaration line runs.

---

**Q: What is the Temporal Dead Zone (TDZ)?**

The TDZ is the time between when a `let` or `const` variable's binding is created (at scope entry) and when its declaration line is reached. Accessing the variable in this zone throws a `ReferenceError`. It prevents you from using a variable before declaring it — a safeguard over `var`'s silent `undefined`.

```js
console.log(x); // ReferenceError — TDZ
let x = 5;
```

---

**Q: What is lexical scope?**

Lexical scope (also called static scope) means that the scope of a variable is determined by where it is defined in the source code — not where it's called. JavaScript uses lexical scope — a function can access variables from its parent scope based on its position in the code, regardless of how or from where it's called.

---

**Q: What is scope chain?**

When a variable is referenced, JavaScript first looks in the current scope. If not found, it looks in the parent scope, then the parent's parent, all the way to the global scope. This chain of nested scopes is the scope chain. If the variable isn't found anywhere, it throws a `ReferenceError`.

---

## Objects & Prototypes

**Q: What is the difference between a shallow copy and a deep copy?**

A shallow copy duplicates the top-level properties but nested objects are still shared references. A deep copy duplicates everything recursively — nested objects are fully independent. `Object.assign({}, obj)` and `{ ...obj }` produce shallow copies. `structuredClone(obj)` (ES2022) produces a true deep copy. `JSON.parse(JSON.stringify(obj))` works for simple objects but loses `undefined`, `Date`, `Map`, `Set`, and functions.

---

**Q: What is prototypal inheritance?**

In JavaScript, every object has a hidden `[[Prototype]]` link to another object (its prototype). When you access a property, if it's not on the object itself, JavaScript follows the prototype chain until it finds it or reaches `null`. Classes in JavaScript are syntactic sugar over this prototype-based system. `Object.create(proto)` creates an object with a specific prototype.

---

**Q: What is `Object.create()` and when would you use it?**

`Object.create(proto)` creates a new object with `proto` as its prototype. It gives you fine-grained control over prototype chains without constructors. Common use: `Object.create(null)` creates a truly empty object with no prototype — useful as a hash map/dictionary without the risk of inherited property collisions like `toString` or `hasOwnProperty`.

---

**Q: What is `Object.freeze()` vs `Object.seal()`?**

`Object.freeze(obj)` makes an object fully immutable — no properties can be added, removed, or changed. `Object.seal(obj)` prevents adding or removing properties, but allows changing existing property values. Both are shallow — nested objects are not frozen/sealed. For true deep immutability, freeze recursively or use a library like `immer`.

---

**Q: What is destructuring?**

Destructuring is a syntax to unpack values from arrays or properties from objects into distinct variables in a single statement.

```js
// Object destructuring:
const { name, age, role = "user" } = person; // with default

// Array destructuring:
const [first, , third] = [1, 2, 3]; // skip elements

// Nested:
const { address: { city } } = user;

// Function parameters:
function greet({ name, age }) { ... }
```

---

**Q: What is the spread operator (`...`) and rest syntax?**

The spread operator expands an iterable (array, string, Set) into individual elements. Rest syntax collects remaining elements into an array. They use the same `...` syntax but in different positions.

```js
// Spread — expand:
const merged = [...arr1, ...arr2];
const copied = { ...obj1, ...obj2 };
Math.max(...numbers);

// Rest — collect:
function sum(...nums) { return nums.reduce((a, b) => a + b, 0); }
const [first, ...rest] = [1, 2, 3, 4];
```

---

**Q: What is optional chaining (`?.`)?**

Optional chaining short-circuits property access, method calls, or subscript access when the left side is `null` or `undefined`, returning `undefined` instead of throwing a TypeError.

```js
const street = user?.address?.street; // undefined if user or address is null
user?.greet?.();                       // safe method call
```

---

**Q: What is nullish coalescing (`??`)?**

The nullish coalescing operator returns the right side only when the left side is `null` or `undefined`. Unlike `||`, it does NOT trigger on falsy values like `0` or `""`, which are often valid values.

```js
const port = config.port ?? 3000; // uses 3000 only if port is null/undefined
const count = 0 ?? 10;            // 0 — not replaced (unlike 0 || 10 = 10)
```

---

**Q: How do you clone an object?**

```js
// Shallow clone:
const copy1 = { ...original };
const copy2 = Object.assign({}, original);

// Deep clone:
const deepCopy = structuredClone(original); // modern, handles Date, Map, Set, circular refs

// Legacy deep clone (limited):
const deepCopy2 = JSON.parse(JSON.stringify(original)); // loses undefined, Date, functions
```

---

## Arrays

**Q: What is the difference between `map`, `filter`, and `reduce`?**

`map` transforms each element and returns a new array of the same length. `filter` returns a new array containing only elements that pass a test. `reduce` accumulates all elements into a single value (sum, object, array, etc.). All three are pure — they don't modify the original array.

```js
[1,2,3].map(n => n * 2);        // [2, 4, 6]
[1,2,3,4].filter(n => n % 2);   // [1, 3]
[1,2,3,4].reduce((a,b) => a+b); // 10
```

---

**Q: What is the difference between `forEach` and `map`?**

`forEach` iterates over an array and returns `undefined` — it's for side effects. `map` transforms each element and returns a new array. You cannot `break` out of either. If you need the return value (a new array), use `map`. If you just want to run code for each item, use `forEach` or `for...of`.

---

**Q: What does `flat()` and `flatMap()` do?**

`flat(depth)` creates a new array with sub-arrays flattened to the specified depth (default 1). `flat(Infinity)` flattens completely. `flatMap` is equivalent to `map` followed by `flat(1)` — useful when each element maps to zero or more results.

```js
[1, [2, [3]]].flat();        // [1, 2, [3]]
[1, [2, [3]]].flat(Infinity); // [1, 2, 3]
[1, 2, 3].flatMap(n => [n, n * 2]); // [1, 2, 2, 4, 3, 6]
```

---

**Q: What is `Array.from()`?**

`Array.from()` creates a new array from an array-like object (NodeList, arguments, string) or iterable (Set, Map, generator). Optionally accepts a mapping function.

```js
Array.from("hello");                // ["h","e","l","l","o"]
Array.from({length: 5}, (_, i) => i); // [0, 1, 2, 3, 4]
Array.from(new Set([1,1,2,3]));     // [1, 2, 3]
```

---

**Q: What is the difference between `find()` and `filter()`?**

`find()` returns the FIRST element that matches the condition (or `undefined`). `filter()` returns an array of ALL matching elements (or empty array). Use `find()` when you only need one result.

---

**Q: What is `Array.prototype.some()` and `every()`?**

`some()` returns `true` if at least one element passes the test. `every()` returns `true` only if ALL elements pass. Both short-circuit — they stop iterating as soon as the answer is determined.

```js
[1, 2, 3].some(n => n > 2);  // true (stops at 3)
[1, 2, 3].every(n => n > 0); // true (checks all)
```

---

**Q: What is a sparse array?**

A sparse array has "holes" — indices with no value. `arr[5] = 1` on an empty array creates a sparse array with length 6 but only one value. Many array methods skip holes (`map`, `forEach`), while others don't (`join`). Avoid sparse arrays — use `Array(n).fill(undefined)` if you need a pre-allocated array.

---

## Async JavaScript

**Q: What is the event loop?**

The event loop is the mechanism that allows JavaScript (single-threaded) to perform non-blocking operations. It continuously checks the call stack — when empty, it picks the next task from the message queue and pushes it to the stack. Microtasks (Promise callbacks, `queueMicrotask`) always run before the next macro-task (setTimeout, setInterval, I/O callbacks).

---

**Q: What is a callback? What problem does "callback hell" describe?**

A callback is a function passed as an argument to another function, to be called when an async operation completes. Callback hell (also called "pyramid of doom") is when callbacks nest deeply — each async step inside the previous one — making code hard to read, error-handle, and maintain. Promises and async/await solve this problem.

---

**Q: What is a Promise and what are its states?**

A Promise is an object representing an asynchronous operation that will eventually succeed or fail. It has three states: `pending` (initial), `fulfilled` (succeeded with a value), and `rejected` (failed with a reason). Once settled, a Promise is immutable — it cannot change state again. You handle results with `.then()` (fulfillment), `.catch()` (rejection), and `.finally()` (always).

---

**Q: What is `async`/`await`?**

`async`/`await` is syntactic sugar over Promises that makes async code read like synchronous code. An `async` function always returns a Promise. `await` pauses execution of the async function until the Promise settles — the JS engine can still run other code while waiting. Errors are caught with `try`/`catch`.

```js
async function getUser(id) {
  try {
    const res = await fetch(`/api/users/${id}`);
    return await res.json();
  } catch (err) {
    console.error("Failed:", err);
  }
}
```

---

**Q: What is the difference between `Promise.all()` and `Promise.allSettled()`?**

`Promise.all()` runs promises in parallel and resolves when ALL fulfill — it rejects immediately if ANY single promise rejects. `Promise.allSettled()` also runs in parallel and waits for ALL to settle (regardless of outcome) — it never rejects. Use `allSettled` when you want results from all operations even if some failed.

---

**Q: What does `Promise.race()` do?**

`Promise.race()` resolves or rejects with the result of whichever promise settles FIRST. The other promises continue executing but their results are ignored. Commonly used for timeout patterns:

```js
const data = await Promise.race([
  fetch("/api/data"),
  new Promise((_, reject) => setTimeout(() => reject(new Error("Timeout")), 5000))
]);
```

---

**Q: What is the difference between `setTimeout(fn, 0)` and `Promise.resolve().then(fn)`?**

Both defer execution, but at different priority levels. `setTimeout(fn, 0)` queues `fn` as a macro-task — it runs after all microtasks are complete. `Promise.resolve().then(fn)` queues `fn` as a microtask — it runs before the next macro-task. Microtasks always run first.

---

**Q: What causes an "Unhandled Promise Rejection"?**

When a Promise rejects and there is no `.catch()` handler or `try/catch` around the `await`, the rejection is unhandled. In browsers, this triggers a `unhandledrejection` event. In Node.js 15+, it crashes the process. Always attach error handlers to every Promise chain.

---

## Classes & OOP

**Q: How do JavaScript classes work under the hood?**

JavaScript classes (ES2015+) are syntactic sugar over the prototype-based inheritance system. A `class` creates a constructor function whose methods are placed on the prototype. `extends` sets up the prototype chain. `super()` calls the parent constructor. There is no separate class runtime — it all compiles down to functions and prototypes.

---

**Q: What is the difference between a class method and a class field function?**

A prototype method (`method() {}`) is defined once on the prototype — all instances share it. A class field arrow function (`method = () => {}`) is defined on EACH instance in the constructor — it captures `this` lexically. Field arrows are useful for event handlers but use more memory (one function per instance) and cannot be overridden via the prototype chain.

---

**Q: What are `static` class members?**

`static` methods and fields belong to the class itself — not to instances. They're called on the class: `MyClass.staticMethod()`. Use statics for utility functions related to the class, factory methods, and shared configuration. `this` inside a static method refers to the class, not an instance.

---

**Q: What is the difference between `public`, `private` (`#`), and `protected` class members in JavaScript?**

JavaScript has `public` (default) and `private` (`#` prefix) class members. Private fields are enforced by the engine — they cannot be accessed from outside the class body at all. JavaScript does NOT have `protected` — there is no language-level mechanism for "accessible to subclasses but not outsiders." Developers simulate it with naming conventions (`_protected`) or TypeScript's type-checking.

---

**Q: What is the `instanceof` operator?**

`instanceof` checks whether an object's prototype chain contains the `.prototype` of the constructor function. `obj instanceof Foo` returns `true` if `Foo.prototype` appears anywhere in `obj`'s prototype chain.

```js
[] instanceof Array;    // true
[] instanceof Object;   // true (Array.prototype chains to Object.prototype)
```

---

## Error Handling

**Q: What is `try`/`catch`/`finally`?**

`try` wraps code that might throw. `catch(e)` runs only when an error is thrown, with `e` being the error object. `finally` runs unconditionally — whether an error occurred or not — for cleanup. A `return` inside `finally` overrides any return in `try` or `catch`.

---

**Q: How do you create custom errors?**

Extend the built-in `Error` class, set `this.name`, and add any custom properties.

```js
class ValidationError extends Error {
  constructor(message, field) {
    super(message);
    this.name = "ValidationError";
    this.field = field;
  }
}
throw new ValidationError("Email is required", "email");
```

---

**Q: What is `Error.cause`?**

Added in ES2022, `Error.cause` lets you chain errors — wrapping a low-level error with a higher-level contextual one while preserving the original.

```js
try {
  JSON.parse(badInput);
} catch (original) {
  throw new Error("Config parsing failed", { cause: original });
}
```

---

## ES6+ Features

**Q: What are template literals?**

Template literals use backticks and support embedded expressions (`${expr}`), multi-line strings (no `\n` needed), and tagged templates (a function that processes the template). They replace string concatenation for clarity.

---

**Q: What is the difference between `for...in` and `for...of`?**

`for...in` iterates over the enumerable PROPERTY KEYS of an object (including inherited ones). `for...of` iterates over the VALUES of any iterable (arrays, strings, Maps, Sets, generators). Never use `for...in` on arrays — it may iterate prototype properties and doesn't guarantee order.

---

**Q: What is a Symbol?**

`Symbol` is a primitive type that creates a unique, immutable identifier. No two Symbols are equal, even with the same description. Symbols are used as unique object property keys (no name collisions) and for well-known behaviors (`Symbol.iterator`, `Symbol.toPrimitive`, etc.). They don't appear in `JSON.stringify`, `Object.keys`, or `for...in`.

---

**Q: What is `Map` and how is it different from a plain object?**

`Map` is a key-value collection where keys can be ANY type (objects, functions, primitives). Plain objects only support string or symbol keys (other types are coerced to strings). `Map` maintains insertion order, has a `.size` property, and has cleaner APIs (`.set`, `.get`, `.has`, `.delete`). Use `Map` when keys are non-strings or when you need ordered iteration.

---

**Q: What is `Set` and when do you use it?**

`Set` is a collection of UNIQUE values. Adding duplicates has no effect. Use it for deduplication, membership testing (`set.has(value)`), and maintaining ordered unique sequences. Converting: `[...new Set(array)]` is the cleanest way to deduplicate an array.

---

**Q: What is `WeakMap` and `WeakSet`?**

`WeakMap` and `WeakSet` hold WEAK references to their keys (WeakMap) or values (WeakSet) — only objects are allowed. They don't prevent garbage collection. If the only reference to an object is a WeakMap key, the object can be collected and the entry disappears. Use WeakMap for private metadata attached to objects without memory leaks. They are not iterable.

---

**Q: What is `Proxy`?**

`Proxy` wraps an object and intercepts fundamental operations — property access (`get`), assignment (`set`), function calls (`apply`), and more — through handler traps. Used for validation, logging, reactive data systems (Vue 3), and access control. Calls the trap function instead of the default behavior.

---

**Q: What is `Reflect`?**

`Reflect` is a built-in object with static methods mirroring `Proxy` handler traps. It provides clean, functional equivalents to language operations: `Reflect.get(obj, prop)`, `Reflect.set(obj, prop, value)`, etc. Often used inside Proxy handlers to forward operations after interception.

---

**Q: What are generators?**

Generator functions use `function*` and can `yield` multiple values one at a time. They are lazy — they pause at each `yield` and resume when `.next()` is called. The value passed to `.next(val)` becomes the result of the previous `yield` expression. Used for iterators, async control flow, and infinite sequences.

---

**Q: What is `async/await` under the hood?**

`async`/`await` is syntax sugar over Promises implemented using generators. The compiler transforms async functions into a state machine — each `await` is a `yield` that suspends the generator. The engine resumes the generator when the awaited Promise settles, with the resolved value replacing the `await` expression. The original async function returns a Promise that resolves when the generator completes.

---

## Common Gotchas & Interview Traps

**Q: Why does `0.1 + 0.2 !== 0.3`?**

JavaScript uses IEEE 754 double-precision floating point. Neither 0.1 nor 0.2 can be represented exactly in binary — they're repeating fractions. The tiny rounding errors accumulate to `0.30000000000000004`. To compare floats: `Math.abs(a - b) < Number.EPSILON`.

---

**Q: What is the difference between `undefined` and not defined?**

`undefined` is a value — the variable exists but has no assigned value. "Not defined" means the variable doesn't exist at all in scope — accessing it throws `ReferenceError`. `typeof undeclaredVar` is `"undefined"` (safely), but `undeclaredVar === undefined` throws.

---

**Q: What does `delete` do?**

`delete obj.prop` removes a property from an object. It returns `true` if successful, `false` if the property is non-configurable. It does NOT affect variables — `delete x` (where `x` is a variable) returns `false` and does nothing. It does NOT free memory directly — garbage collection handles that when no references remain.

---

**Q: What is an immediately rejected/resolved Promise?**

`Promise.resolve(value)` creates an already-resolved Promise. `Promise.reject(reason)` creates an already-rejected Promise. Their handlers still run asynchronously (as microtasks), never synchronously — even if the Promise was already settled when you attach the handler.

---

**Q: Can you explain `event.target` vs `event.currentTarget`?**

`event.target` is the element that was actually clicked (the deepest element in the DOM that received the event). `event.currentTarget` is the element the event listener is attached to — it changes as the event bubbles. In a delegated event handler on a parent, `target` is the clicked child, `currentTarget` is the parent.

---

**Q: What is the difference between deep equal and reference equal?**

Reference equal (`===`) checks if two variables point to the SAME object in memory. Deep equal checks if two objects have the same structure and values — even if they're different objects. JavaScript has no built-in deep equal for objects; use `JSON.stringify` (limited) or libraries like Lodash's `_.isEqual`.

---

**Q: What is a "pure function"?**

A pure function (1) always returns the same output for the same inputs and (2) has no side effects — it doesn't modify external state, mutate arguments, make network calls, or interact with I/O. Pure functions are predictable, testable, and safe to memoize.

---

**Q: What is memoization?**

Memoization is an optimization technique that caches the results of expensive function calls and returns the cached result for the same inputs. It's a form of trading memory for speed.

```js
function memoize(fn) {
  const cache = new Map();
  return function (...args) {
    const key = JSON.stringify(args);
    if (cache.has(key)) return cache.get(key);
    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}
```

---

**Q: What is debouncing and throttling?**

Both limit how often a function runs in response to rapid events.

**Debounce** — delays execution until after a quiet period. The function only runs after the events stop. Use for search-as-you-type input (fire the API call after the user stops typing for 300ms).

**Throttle** — limits to at most once per interval. Use for scroll handlers, resize handlers — you want updates but not every millisecond.

---

**Q: What is the difference between `Object.keys()`, `Object.values()`, and `Object.entries()`?**

`Object.keys(obj)` returns an array of own enumerable property names. `Object.values(obj)` returns an array of own enumerable property values. `Object.entries(obj)` returns an array of `[key, value]` pairs. All three only include own (non-inherited) enumerable properties. Non-enumerable and Symbol-keyed properties are excluded.

---

**Q: What is JSON and how do you serialize/deserialize?**

JSON (JavaScript Object Notation) is a text format for data interchange. `JSON.stringify(value)` converts a JS value to a JSON string. `JSON.parse(string)` parses JSON back to JS. `undefined`, functions, and Symbols are silently omitted during stringify. Circular references throw a TypeError.

---

**Q: What is the difference between `parseInt()` and `Number()`?**

`parseInt(str, radix)` parses left-to-right, stopping at the first non-numeric character. `Number("123px")` returns `NaN` — it requires the entire string to be valid. Always specify the radix in `parseInt` (use `10` for decimal) to avoid `parseInt("010")` being parsed as octal in older engines.

```js
parseInt("123px", 10); // 123 — stops at 'p'
Number("123px");       // NaN — can't parse full string
```

---

**Q: What is `Array.isArray()` and why use it instead of `typeof` or `instanceof`?**

`typeof []` returns `"object"` — not useful. `[] instanceof Array` fails across iframes (different global contexts have different `Array` constructors). `Array.isArray([])` is always reliable regardless of realm/context and is the recommended approach.

---

**Q: What does `Object.assign()` do?**

`Object.assign(target, ...sources)` copies own enumerable properties from one or more source objects into the target object. It modifies and returns the target. It's a shallow copy — nested objects are copied by reference. `Object.assign({}, obj)` is a common pattern for shallow cloning.

---

**Q: How does `JSON.stringify()` handle special values?**

- `undefined`, functions, Symbols → silently omitted from objects, become `null` in arrays
- `NaN`, `Infinity`, `-Infinity` → become `null`
- `Date` objects → converted to ISO string (but `JSON.parse` returns a string, not Date)
- Circular references → throws `TypeError`
- `BigInt` → throws `TypeError`
- `Map`, `Set` → serialized as `{}`

---

**Q: What is `typeof` vs `instanceof`?**

`typeof` works on any value and returns a string type name — best for primitives. `instanceof` checks the prototype chain — only works for objects and requires the constructor to be from the same context. Use `typeof` for type guards on primitives, `instanceof` for class instances, `Array.isArray()` for arrays.

---

**Q: What is the `in` operator?**

`"prop" in obj` returns `true` if the property exists on the object OR anywhere in its prototype chain. Use `Object.hasOwn(obj, "prop")` to check ONLY own properties.

---

**Q: What is short-circuit evaluation?**

Logical operators `&&` and `||` don't always evaluate both sides. `a && b` — if `a` is falsy, `b` is never evaluated. `a || b` — if `a` is truthy, `b` is never evaluated. This is used for conditional rendering (`condition && <Component/>`) and default values (`config.timeout || 5000`).

---

**Q: What is the `void` operator?**

`void expression` evaluates the expression and always returns `undefined`. Mainly used in old-style `href="javascript:void(0)"` to prevent navigation, or to explicitly discard a return value. In modern code, rarely needed.

---

**Q: What is `toString()` vs `valueOf()` for type coercion?**

When JavaScript needs to convert an object to a primitive: for numeric contexts, `valueOf()` is tried first. For string contexts, `toString()` is tried first. If neither produces a primitive, the other is tried. You can override both to customize how your objects behave in coercion.

---

**Q: What are getters and setters?**

Getters (`get propName()`) and setters (`set propName(val)`) allow you to define computed/reactive properties that look like regular property accesses but execute code.

```js
class Circle {
  constructor(r) { this.radius = r; }
  get area() { return Math.PI * this.radius ** 2; }      // computed
  get diameter() { return this.radius * 2; }
  set diameter(d) { this.radius = d / 2; }               // derived setter
}
const c = new Circle(5);
c.area;       // computed — no ()
c.diameter = 20; // sets radius to 10
```

---

**Q: What is the difference between `Object.defineProperty()` and regular assignment?**

Regular assignment creates an enumerable, configurable, writable property. `Object.defineProperty()` gives you full control over property descriptors: `enumerable` (shows in loops), `configurable` (can be deleted/redefined), `writable` (can be changed), plus `get`/`set` for accessors. Used internally by frameworks for reactive property observation.

---

**Q: How does `Array.prototype.sort()` work and what is its default behavior?**

`sort()` sorts in-place and returns the array. Without a comparator, it converts elements to strings and sorts lexicographically — `[10, 9, 1].sort()` gives `[1, 10, 9]` because `"10" < "9"`. Always provide a comparator for numbers: `arr.sort((a, b) => a - b)`. The algorithm is not guaranteed to be stable across all JS engines (V8 uses TimSort, which is stable).

---

**Q: What is `Object.fromEntries()`?**

`Object.fromEntries(iterable)` is the inverse of `Object.entries()` — it creates an object from an iterable of `[key, value]` pairs. Useful for transforming objects via Map or filter operations.

```js
const doubled = Object.fromEntries(
  Object.entries(obj).map(([k, v]) => [k, v * 2])
);
```

---

**Q: What is the comma operator?**

The comma operator evaluates both operands left to right and returns the value of the LAST one. Rarely used intentionally in modern code, but common in `for` loop initializations: `for (let i = 0, j = 10; i < j; i++, j--)`.

---

## Quick-Fire One-Liners

| Question | Answer |
|----------|--------|
| What does `!!value` do? | Converts any value to its boolean equivalent |
| What is `+value`? | Unary `+` coerces to number: `+"42"` → `42`, `+true` → `1` |
| What is `~index`? | Bitwise NOT — `~arr.indexOf(x)` is falsy only when index is -1 |
| What does `a ?? b ?? c` do? | Returns first non-null/undefined value in the chain |
| Can `const` hold a mutable value? | Yes — `const` prevents reassignment, not mutation |
| Is JavaScript pass-by-value or reference? | Primitives: by value. Objects: by reference (the reference is passed by value) |
| What is the prototype of a function? | `Function.prototype` |
| What is `Function.prototype.call` vs `Function.prototype.apply`? | `call` takes args individually; `apply` takes an array |
| Can you `await` a non-Promise value? | Yes — `await 42` returns `42` (wrapped in resolved Promise) |
| What does `Array.prototype.splice()` return? | An array of the removed elements (mutates original) |
| What is `Object.keys` on an array? | Returns string indices: `Object.keys([1,2,3])` → `["0","1","2"]` |
| What is `arguments` in an arrow function? | Arrow functions have no `arguments` — it would reference outer scope's |
| What does `new` do? | Creates new object, sets `[[Prototype]]`, runs constructor, returns `this` (unless constructor returns object) |
| What is `Symbol.iterator`? | The well-known symbol that makes an object iterable (used by `for...of`) |
| What is `Number.EPSILON`? | The smallest difference between two representable floats: ~2.22e-16 |
| What is `Number.MAX_SAFE_INTEGER`? | `2^53 - 1` = 9,007,199,254,740,991 — beyond this, integer arithmetic is imprecise |

---

*This file covers the essentials. For deeper dives, see:*
- *[00-Execution-Internals.md](./00-Execution-Internals.md) — how JS engines work*
- *[01-Core-Fundamentals.md](./01-Core-Fundamentals.md) — closures, scope, this*
- *[16-Output-Tricky-Questions.md](./16-Output-Tricky-Questions.md) — output-based tricky Qs*
