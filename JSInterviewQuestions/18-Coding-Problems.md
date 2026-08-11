# 18 — Coding / Logic Problems
### 20 Questions | Intermediate

> Write clean, working code. Talk through your thought process. Mention time/space complexity.

---

**Q1. Implement `debounce(fn, delay)`.** — Medium

```js
function debounce(fn, delay) {
  let timerId = null;

  return function (...args) {
    clearTimeout(timerId); // cancel any pending call
    timerId = setTimeout(() => {
      fn.apply(this, args);
      timerId = null;
    }, delay);
  };
}

// Usage:
const search = debounce((query) => fetchResults(query), 300);
input.addEventListener("input", (e) => search(e.target.value));
// API is called only 300ms after the user stops typing

// With leading edge (fires immediately on first call, then debounces):
function debounceLeading(fn, delay) {
  let timerId = null;
  return function (...args) {
    if (!timerId) fn.apply(this, args); // fire immediately on first
    clearTimeout(timerId);
    timerId = setTimeout(() => { timerId = null; }, delay);
  };
}
```

**Complexity:** O(1) time. O(1) space (one timer ID).

---

**Q2. Implement `throttle(fn, interval)`.** — Medium

```js
function throttle(fn, interval) {
  let lastCall = 0;
  let timerId = null;

  return function (...args) {
    const now = Date.now();
    const remaining = interval - (now - lastCall);

    if (remaining <= 0) {
      // Enough time has passed — call immediately:
      clearTimeout(timerId);
      timerId = null;
      lastCall = now;
      fn.apply(this, args);
    } else if (!timerId) {
      // Schedule trailing call to catch the last invocation:
      timerId = setTimeout(() => {
        lastCall = Date.now();
        timerId = null;
        fn.apply(this, args);
      }, remaining);
    }
  };
}

// Usage:
const onScroll = throttle(() => updatePosition(), 100);
window.addEventListener("scroll", onScroll);
// updatePosition fires at most once every 100ms
```

**Key difference from debounce:** Throttle guarantees at least one call per `interval`. Debounce delays until activity stops.

---

**Q3. Implement `deepClone(obj)` without using `structuredClone`.** — Hard

```js
function deepClone(value, seen = new Map()) {
  // Primitives — return as-is:
  if (value === null || typeof value !== "object") return value;

  // Handle circular references:
  if (seen.has(value)) return seen.get(value);

  // Handle special types:
  if (value instanceof Date) return new Date(value.getTime());
  if (value instanceof RegExp) return new RegExp(value.source, value.flags);
  if (value instanceof Map) {
    const mapClone = new Map();
    seen.set(value, mapClone);
    value.forEach((v, k) => mapClone.set(deepClone(k, seen), deepClone(v, seen)));
    return mapClone;
  }
  if (value instanceof Set) {
    const setClone = new Set();
    seen.set(value, setClone);
    value.forEach(v => setClone.add(deepClone(v, seen)));
    return setClone;
  }

  // Arrays and plain objects:
  const clone = Array.isArray(value) ? [] : Object.create(Object.getPrototypeOf(value));
  seen.set(value, clone); // register BEFORE recursing to handle circular refs

  for (const key of Reflect.ownKeys(value)) {
    const descriptor = Object.getOwnPropertyDescriptor(value, key);
    if (descriptor.value !== undefined) {
      descriptor.value = deepClone(descriptor.value, seen);
    }
    Object.defineProperty(clone, key, descriptor);
  }

  return clone;
}

// Test:
const a = { x: 1, arr: [1, 2, { nested: true }], date: new Date() };
a.self = a; // circular reference
const b = deepClone(a);
b.arr[2].nested = false;
console.log(a.arr[2].nested); // true — not mutated
console.log(b.self === b);    // true — circular preserved
```

**Complexity:** O(n) time and space where n = total number of nested values.

---

**Q4. Implement `memoize(fn)`.** — Medium

```js
function memoize(fn) {
  const cache = new Map();

  return function (...args) {
    // Use JSON for multi-arg key (works for JSON-serializable args):
    const key = JSON.stringify(args);
    if (cache.has(key)) {
      return cache.get(key);
    }
    const result = fn.apply(this, args);
    cache.set(key, result);
    return result;
  };
}

// With WeakMap for object args (avoids memory leaks):
function memoizeOne(fn) {
  let lastArgs = null;
  let lastResult = null;

  return function (...args) {
    if (lastArgs !== null && args.every((a, i) => a === lastArgs[i])) {
      return lastResult; // same args — return cached
    }
    lastArgs = args;
    lastResult = fn.apply(this, args);
    return lastResult;
  };
}

// Test:
const expensiveSquare = memoize((n) => {
  console.log("computing...");
  return n * n;
});
expensiveSquare(4); // "computing..." → 16
expensiveSquare(4); // (from cache) → 16
expensiveSquare(5); // "computing..." → 25
```

**GOTCHA:** `JSON.stringify` fails for functions, Symbols, circular refs, and doesn't distinguish `undefined` from missing. For production, use a proper serializer or a trie-based cache keyed per argument.

---

**Q5. Implement `Promise.all` from scratch.** — Hard

```js
function promiseAll(promises) {
  return new Promise((resolve, reject) => {
    if (!promises.length) return resolve([]);

    const results = new Array(promises.length);
    let remaining = promises.length;

    promises.forEach((p, index) => {
      // Wrap in Promise.resolve to handle non-Promise values:
      Promise.resolve(p).then((value) => {
        results[index] = value;
        remaining--;
        if (remaining === 0) resolve(results);
      }, reject); // reject immediately on first failure
    });
  });
}

// Test:
promiseAll([
  Promise.resolve(1),
  Promise.resolve(2),
  Promise.resolve(3)
]).then(console.log); // [1, 2, 3]

promiseAll([
  Promise.resolve(1),
  Promise.reject("error"),
  Promise.resolve(3)
]).catch(console.log); // "error"
```

---

**Q6. Flatten a deeply nested array.** — Easy

```js
// Recursive:
function flatDeep(arr) {
  return arr.reduce((acc, val) =>
    Array.isArray(val)
      ? acc.concat(flatDeep(val))
      : [...acc, val],
  []);
}

// Iterative (stack-based, avoids call stack overflow):
function flatDeepIterative(arr) {
  const result = [];
  const stack = [...arr];
  while (stack.length) {
    const item = stack.pop();
    if (Array.isArray(item)) {
      stack.push(...item); // push elements back to process
    } else {
      result.unshift(item); // add to front to preserve order
    }
  }
  return result;
}

// Built-in (ES2019):
[1, [2, [3, [4]]]].flat(Infinity); // [1, 2, 3, 4]

// Test:
flatDeep([1, [2, [3, [4, [5]]]]]); // [1, 2, 3, 4, 5]
```

**Complexity:** O(n) time, O(n) space (output array + call stack depth for recursive).

---

**Q7. Implement `curry(fn)` — convert any function to its curried form.** — Hard

```js
function curry(fn) {
  return function curried(...args) {
    if (args.length >= fn.length) {
      // Have enough args — call the original:
      return fn.apply(this, args);
    }
    // Not enough args — return a function that collects more:
    return function (...moreArgs) {
      return curried.apply(this, [...args, ...moreArgs]);
    };
  };
}

// Test:
function add(a, b, c) { return a + b + c; }

const curriedAdd = curry(add);
curriedAdd(1)(2)(3);     // 6
curriedAdd(1, 2)(3);     // 6
curriedAdd(1)(2, 3);     // 6
curriedAdd(1, 2, 3);     // 6

// Practical use — partial application:
const add10 = curriedAdd(10);
const add10and5 = add10(5);
add10and5(3); // 18
```

**Key insight:** `fn.length` gives the number of declared parameters. Currying works by accumulating arguments until that count is met.

---

**Q8. Given a string, find the first non-repeating character.** — Easy

```js
function firstNonRepeating(str) {
  const count = new Map();

  // Count occurrences:
  for (const char of str) {
    count.set(char, (count.get(char) ?? 0) + 1);
  }

  // Find first with count === 1:
  for (const char of str) {
    if (count.get(char) === 1) return char;
  }

  return null;
}

firstNonRepeating("aabbcde"); // "c"
firstNonRepeating("aabb");    // null
firstNonRepeating("stress");  // "t"
```

**Complexity:** O(n) time, O(k) space where k = unique characters (at most 26 for lowercase alpha).

---

**Q9. Implement `groupBy(arr, fn)`.** — Easy

```js
function groupBy(arr, fn) {
  return arr.reduce((groups, item) => {
    const key = fn(item);
    (groups[key] ??= []).push(item);
    return groups;
  }, {});
}

// Or using Map (supports non-string keys):
function groupByMap(arr, fn) {
  const map = new Map();
  for (const item of arr) {
    const key = fn(item);
    if (!map.has(key)) map.set(key, []);
    map.get(key).push(item);
  }
  return map;
}

// Test:
const people = [
  { name: "Alice", dept: "Eng" },
  { name: "Bob",   dept: "Mkt" },
  { name: "Carol", dept: "Eng" },
];

groupBy(people, p => p.dept);
// { Eng: [Alice, Carol], Mkt: [Bob] }
```

---

**Q10. Implement a function that composes multiple functions: `compose(f, g, h)(x)` = `f(g(h(x)))`.** — Medium

```js
// Compose — right to left:
function compose(...fns) {
  return function (x) {
    return fns.reduceRight((acc, fn) => fn(acc), x);
  };
}

// Pipe — left to right (more intuitive for data pipelines):
function pipe(...fns) {
  return function (x) {
    return fns.reduce((acc, fn) => fn(acc), x);
  };
}

// Test:
const double = x => x * 2;
const addOne = x => x + 1;
const square = x => x * x;

const transform = compose(addOne, double, square); // addOne(double(square(x)))
transform(3); // square(3)=9 → double(9)=18 → addOne(18)=19

const pipeline = pipe(square, double, addOne);     // same operations, left-to-right
pipeline(3); // 9 → 18 → 19
```

---

**Q11. Implement `EventEmitter` with `on`, `off`, `emit`, and `once`.** — Hard

```js
class EventEmitter {
  #listeners = new Map();

  on(event, handler) {
    if (!this.#listeners.has(event)) {
      this.#listeners.set(event, new Set());
    }
    this.#listeners.get(event).add(handler);
    return () => this.off(event, handler); // return unsubscribe
  }

  off(event, handler) {
    this.#listeners.get(event)?.delete(handler);
  }

  once(event, handler) {
    const wrapper = (...args) => {
      handler(...args);
      this.off(event, wrapper);
    };
    return this.on(event, wrapper);
  }

  emit(event, ...args) {
    this.#listeners.get(event)?.forEach(handler => {
      try { handler(...args); }
      catch (e) { console.error(e); }
    });
  }

  removeAllListeners(event) {
    if (event) this.#listeners.delete(event);
    else this.#listeners.clear();
  }

  listenerCount(event) {
    return this.#listeners.get(event)?.size ?? 0;
  }
}

// Test:
const ee = new EventEmitter();
const unsub = ee.on("data", d => console.log("received:", d));
ee.once("connect", () => console.log("connected!"));

ee.emit("data", 42);       // "received: 42"
ee.emit("connect");        // "connected!"
ee.emit("connect");        // (nothing — once only fires once)
unsub();                   // unsubscribe
ee.emit("data", 99);       // (nothing — handler removed)
```

---

**Q12. Implement `chunk(array, size)` — split array into chunks of given size.** — Easy

```js
function chunk(arr, size) {
  if (size <= 0) throw new RangeError("size must be positive");
  const result = [];
  for (let i = 0; i < arr.length; i += size) {
    result.push(arr.slice(i, i + size));
  }
  return result;
}

chunk([1, 2, 3, 4, 5], 2); // [[1,2], [3,4], [5]]
chunk([1, 2, 3], 5);       // [[1,2,3]]
chunk([], 3);               // []
```

**Complexity:** O(n) time, O(n) space.

---

**Q13. Implement a LRU (Least Recently Used) cache.** — Hard

```js
class LRUCache {
  #capacity;
  #cache; // Map preserves insertion order

  constructor(capacity) {
    this.#capacity = capacity;
    this.#cache = new Map();
  }

  get(key) {
    if (!this.#cache.has(key)) return -1;
    // Move to end (most recently used):
    const value = this.#cache.get(key);
    this.#cache.delete(key);
    this.#cache.set(key, value);
    return value;
  }

  put(key, value) {
    if (this.#cache.has(key)) {
      this.#cache.delete(key); // remove to re-insert at end
    } else if (this.#cache.size >= this.#capacity) {
      // Evict least recently used (first item in Map):
      const firstKey = this.#cache.keys().next().value;
      this.#cache.delete(firstKey);
    }
    this.#cache.set(key, value); // insert at end (most recent)
  }
}

// Test:
const lru = new LRUCache(3);
lru.put("a", 1);
lru.put("b", 2);
lru.put("c", 3);
lru.get("a");     // 1 — "a" becomes most recently used
lru.put("d", 4);  // evicts "b" (LRU)
lru.get("b");     // -1 — evicted
lru.get("c");     // 3
```

**Complexity:** O(1) get and put — Map operations are O(1) average.

**Why Map works:** `Map` in V8 maintains insertion order. The first key in iteration is always the oldest (LRU). Delete + re-insert moves a key to the end (MRU).

---

**Q14. Implement `deepEqual(a, b)`.** — Medium

```js
function deepEqual(a, b) {
  // Strict primitive equality:
  if (a === b) return true;

  // Different types:
  if (typeof a !== typeof b) return false;

  // Null check (typeof null === "object"):
  if (a === null || b === null) return false;

  // Date:
  if (a instanceof Date && b instanceof Date) {
    return a.getTime() === b.getTime();
  }

  // RegExp:
  if (a instanceof RegExp && b instanceof RegExp) {
    return a.source === b.source && a.flags === b.flags;
  }

  // Array:
  if (Array.isArray(a) && Array.isArray(b)) {
    if (a.length !== b.length) return false;
    return a.every((item, i) => deepEqual(item, b[i]));
  }

  // Object (plain):
  if (typeof a === "object" && typeof b === "object") {
    const keysA = Object.keys(a);
    const keysB = Object.keys(b);
    if (keysA.length !== keysB.length) return false;
    return keysA.every(k => Object.prototype.hasOwnProperty.call(b, k) && deepEqual(a[k], b[k]));
  }

  return false;
}

deepEqual({ a: 1, b: [2, 3] }, { a: 1, b: [2, 3] }); // true
deepEqual({ a: 1 }, { a: 1, b: 2 });                  // false
deepEqual(new Date("2024"), new Date("2024"));          // true
```

---

**Q15. Implement `flatten` for a nested object (dot-notation keys).** — Hard

```js
function flattenObject(obj, prefix = "", result = {}) {
  for (const [key, value] of Object.entries(obj)) {
    const flatKey = prefix ? `${prefix}.${key}` : key;

    if (value !== null && typeof value === "object" && !Array.isArray(value)) {
      flattenObject(value, flatKey, result); // recurse
    } else {
      result[flatKey] = value;
    }
  }
  return result;
}

flattenObject({
  a: 1,
  b: { c: 2, d: { e: 3 } },
  f: [1, 2, 3]          // arrays are NOT flattened further
});
// { "a": 1, "b.c": 2, "b.d.e": 3, "f": [1, 2, 3] }

// Reverse — unflatten:
function unflattenObject(flat) {
  const result = {};
  for (const [dotKey, value] of Object.entries(flat)) {
    const keys = dotKey.split(".");
    let current = result;
    for (let i = 0; i < keys.length - 1; i++) {
      current[keys[i]] ??= {};
      current = current[keys[i]];
    }
    current[keys[keys.length - 1]] = value;
  }
  return result;
}
```

---

**Q16. Implement a `pipe` function that supports async functions.** — Hard

```js
function pipeAsync(...fns) {
  return function (input) {
    return fns.reduce(
      (promiseChain, fn) => promiseChain.then(fn),
      Promise.resolve(input)
    );
  };
}

// Or with async/await:
function pipeAsync2(...fns) {
  return async function (input) {
    let result = input;
    for (const fn of fns) {
      result = await fn(result);
    }
    return result;
  };
}

// Test:
const fetchUser = async (id) => ({ id, name: "Alice", roleId: 5 });
const fetchRole = async (user) => ({ ...user, role: "admin" });
const formatUser = async (user) => `${user.name} [${user.role}]`;

const getFormattedUser = pipeAsync(fetchUser, fetchRole, formatUser);
await getFormattedUser(42); // "Alice [admin]"
```

---

**Q17. Implement `retryWithBackoff(fn, maxRetries, baseDelay)`.** — Hard

```js
async function retryWithBackoff(fn, maxRetries = 3, baseDelay = 100) {
  let lastError;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (err) {
      lastError = err;

      if (attempt === maxRetries) break; // out of retries

      // Exponential backoff with jitter:
      const delay = baseDelay * Math.pow(2, attempt) + Math.random() * 100;
      await new Promise(resolve => setTimeout(resolve, delay));
      console.log(`Retry ${attempt + 1}/${maxRetries} after ${delay.toFixed(0)}ms`);
    }
  }

  throw new Error(`Failed after ${maxRetries} retries: ${lastError.message}`, {
    cause: lastError
  });
}

// Usage:
const data = await retryWithBackoff(
  () => fetch("/api/data").then(r => { if (!r.ok) throw new Error(r.statusText); return r.json(); }),
  3,     // max 3 retries (4 total attempts)
  200    // base delay 200ms (will be 200, 400, 800ms)
);
```

---

**Q18. Implement a function to find all permutations of a string.** — Hard

```js
function permutations(str) {
  if (str.length <= 1) return [str];
  const result = [];

  for (let i = 0; i < str.length; i++) {
    const char = str[i];
    const remaining = str.slice(0, i) + str.slice(i + 1);
    const subPerms = permutations(remaining);
    for (const perm of subPerms) {
      result.push(char + perm);
    }
  }

  return result;
}

permutations("abc");
// ["abc", "acb", "bac", "bca", "cab", "cba"]

// Deduplicated (for strings with repeated chars):
function uniquePermutations(str) {
  return [...new Set(permutations(str))];
}

uniquePermutations("aab"); // ["aab", "aba", "baa"]
```

**Complexity:** O(n!) time (n! permutations), O(n) space per recursion depth, O(n × n!) total space for results.

---

**Q19. Implement `treeMap` — walk a tree and transform each node's value.** — Hard

```js
// Tree structure: { value, children: [...] }
function treeMap(node, transformFn) {
  if (!node) return null;
  return {
    value: transformFn(node.value),
    children: (node.children ?? []).map(child => treeMap(child, transformFn))
  };
}

// Test:
const tree = {
  value: 1,
  children: [
    { value: 2, children: [
        { value: 4, children: [] },
        { value: 5, children: [] }
    ]},
    { value: 3, children: [
        { value: 6, children: [] }
    ]}
  ]
};

const doubled = treeMap(tree, v => v * 2);
// { value: 2, children: [{ value: 4, ...}, { value: 6, ...}] }

// BFS traversal (iterative):
function bfsTree(root) {
  const result = [];
  const queue = [root];
  while (queue.length) {
    const node = queue.shift();
    result.push(node.value);
    queue.push(...(node.children ?? []));
  }
  return result;
}

bfsTree(tree); // [1, 2, 3, 4, 5, 6]
```

---

**Q20. Implement `observable(obj)` — make an object's properties reactive.** — Hard

```js
function observable(obj) {
  const listeners = new Map(); // property → Set of callbacks

  function watch(prop, callback) {
    if (!listeners.has(prop)) listeners.set(prop, new Set());
    listeners.get(prop).add(callback);
    return () => listeners.get(prop)?.delete(callback); // unwatch
  }

  const proxy = new Proxy(obj, {
    get(target, prop) {
      if (prop === "$watch") return watch;
      return target[prop];
    },
    set(target, prop, value) {
      const oldValue = target[prop];
      target[prop] = value;
      if (oldValue !== value) {
        listeners.get(prop)?.forEach(cb => cb(value, oldValue));
      }
      return true;
    }
  });

  return proxy;
}

// Test:
const state = observable({ count: 0, name: "Alice" });

const unwatch = state.$watch("count", (newVal, oldVal) => {
  console.log(`count: ${oldVal} → ${newVal}`);
});

state.count = 1;  // "count: 0 → 1"
state.count = 5;  // "count: 1 → 5"
state.count = 5;  // (no log — same value)
unwatch();
state.count = 10; // (no log — unwatched)
```

---

*Next: [19-Scenario-Based.md](./19-Scenario-Based.md)*
