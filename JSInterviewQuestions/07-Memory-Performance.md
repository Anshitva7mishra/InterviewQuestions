# 07 — Memory Management & Performance
### 15 Questions | Advanced

---

**Q1. How does JavaScript memory management work and what is the mark-and-sweep algorithm?** — Hard

**Answer:**
JavaScript uses automatic memory management — the developer does not manually allocate or free memory. The engine's garbage collector reclaims memory that is no longer reachable.

The mark-and-sweep algorithm (used by all modern JS engines):

Phase 1 — Marking:
- Start from "roots": global variables, variables currently on the call stack, and any variables closed over by live closures.
- Traverse all references reachable from these roots, marking every object as "live."

Phase 2 — Sweeping:
- Scan the entire heap.
- Any object NOT marked as live is unreachable — it is garbage and its memory is reclaimed.

Phase 3 — Compaction (optional):
- Defragment memory by moving live objects together, improving allocation locality.

```
Roots (globals, stack):
  |-> Object A (marked live)
        |-> Object B (marked live)
        |-> Object C (marked live)
              |-> Object D (marked live)

Object E (no path from roots) -> GARBAGE -> collected
Object F (no path from roots) -> GARBAGE -> collected
```

Modern enhancements in V8:
- Generational collection: Most objects die young. V8 divides the heap into young and old generations and uses different algorithms (Scavenge for young, Mark-Sweep-Compact for old).
- Incremental marking: Marking happens in small steps interleaved with program execution to reduce pause times.
- Concurrent marking: Marking runs on background threads while JS runs on the main thread.
- Parallel sweeping: Sweeping runs across multiple threads simultaneously.

**Spec Reference:** Not specified by ECMAScript — implementation detail. The spec only defines when objects can be collected (when they are not reachable).

**Follow-up:** Does the GC know exactly when an object becomes unreachable?

No — GC only collects during a GC run, which happens at intervals determined by the engine's heuristics (heap pressure, time since last collection, etc.). An object becoming unreachable does not immediately free memory.

**GOTCHA:** Setting a variable to `null` does not free the object's memory — it just removes that one reference. The GC will free the memory only at the next GC cycle, and only if no other references to the object exist.

---

**Q2. What are the most common sources of memory leaks in JavaScript?** — Hard

**Answer:**
Memory leaks occur when objects remain reachable (and therefore cannot be collected) even though they are no longer useful to the program.

1. Forgotten event listeners:
```js
// Leak: listener added but never removed
function setup() {
  const data = loadLargeDataset(); // large object
  document.getElementById("btn").addEventListener("click", function() {
    process(data); // 'data' is closed over — stays in memory as long as listener exists
  });
}
// Even if the button is removed from DOM, the listener (and its closure) persist
// because the global document still holds a reference to the event target map

// Fix: Remove listeners when no longer needed
const handler = () => process(data);
btn.addEventListener("click", handler);
// Later:
btn.removeEventListener("click", handler);
```

2. Detached DOM nodes:
```js
// Leak: DOM node removed from tree but still referenced in JS
let cachedElement = document.getElementById("modal");
document.body.removeChild(cachedElement);
// cachedElement still holds the element — it cannot be GC'd
// Fix: Set to null when done
cachedElement = null;
```

3. Timers and intervals not cleared:
```js
// Leak: setInterval with closure capturing a large object
function startPolling() {
  const cache = new Map(); // grows over time
  const id = setInterval(() => {
    cache.set(Date.now(), fetchData()); // cache grows indefinitely
  }, 1000);
  // If clearInterval(id) is never called, this runs forever
}
// Fix: always clearInterval/clearTimeout when done
```

4. Closures holding large objects unnecessarily:
```js
// Leak: outer variable held by closure even when only a small part is needed
function processLargeFile() {
  const largeBuffer = new ArrayBuffer(100_000_000); // 100MB
  const firstByte = new Uint8Array(largeBuffer)[0]; // we only need this

  return function() {
    return firstByte; // this closure holds a reference to largeBuffer!
  };
}
// Fix: Extract what you need before creating the closure
function processLargeFileFix() {
  const largeBuffer = new ArrayBuffer(100_000_000);
  const firstByte = new Uint8Array(largeBuffer)[0];
  // largeBuffer is now out of scope after this function returns
  return function() { return firstByte; };
}
```

5. Uncleaned WeakRef / FinalizationRegistry registrations.

6. Accumulation without bounds (unbounded caches):
```js
const cache = {}; // Grows forever — no eviction
function getUser(id) {
  if (!cache[id]) cache[id] = fetchUser(id); // cache grows indefinitely
  return cache[id];
}
// Fix: Use LRU cache with a max size, or WeakMap when keys are objects
```

**GOTCHA:** Memory leaks in SPAs (Single Page Applications) are more dangerous than in multi-page apps. In traditional apps, navigation reloads the page and clears everything. In SPAs, the app runs for hours or days — small leaks accumulate into significant problems.

---

**Q3. How do `WeakRef` and `FinalizationRegistry` work?** — Hard

**Answer:**
`WeakRef` holds a reference to an object without preventing garbage collection. You access the object via `.deref()` — which returns the object if still alive, or `undefined` if it has been collected.

`FinalizationRegistry` lets you register a callback that runs after a registered object is garbage collected, receiving a cleanup value you provide at registration.

```js
// WeakRef example — a cache that does not prevent GC:
class WeakCache {
  #cache = new Map();

  set(key, value) {
    this.#cache.set(key, new WeakRef(value));
  }

  get(key) {
    const ref = this.#cache.get(key);
    if (!ref) return undefined;
    const value = ref.deref();
    if (value === undefined) {
      this.#cache.delete(key); // Clean up dead entry
    }
    return value;
  }
}

// FinalizationRegistry example — cleanup callback after GC:
const registry = new FinalizationRegistry((cleanupToken) => {
  console.log(`Object with token "${cleanupToken}" was garbage collected`);
  // Release associated resources
  releaseResource(cleanupToken);
});

let obj = { data: "important" };
registry.register(obj, "myObject"); // register with cleanup token

obj = null; // remove strong reference — obj CAN now be GC'd
// At some indeterminate future time, the registry callback fires:
// "Object with token "myObject" was garbage collected"
```

Important caveats:
- The timing of GC (and therefore `FinalizationRegistry` callbacks) is non-deterministic — you cannot predict or force it.
- `WeakRef.deref()` can return `undefined` at any point after a GC run.
- Never use these for correctness — only for optimization (caching) or cleanup.
- Both were added in ES2021.

**Spec Reference:** ECMAScript section 27.4 — WeakRef Objects; section 27.5 — FinalizationRegistry Objects

**GOTCHA:** Even `WeakRef.deref()` returning non-undefined does not guarantee the object lives indefinitely — the GC could collect it between your check and your use. Always check the return value of `deref()` before using it, and do not store the dereffed value across async gaps.

---

**Q4. What is a memory heap snapshot and how do you use Chrome DevTools to find leaks?** — Hard

**Answer:**
A heap snapshot captures the entire JS heap at a point in time — all objects, their sizes, and the references between them. Comparing snapshots taken at different times reveals growing objects.

Step-by-step leak detection workflow:

1. Open Chrome DevTools > Memory tab.
2. Take a baseline snapshot (Snapshot 1) — this is your starting state.
3. Perform the suspected leaking action (navigate, open/close a modal, etc.).
4. Take another snapshot (Snapshot 2).
5. Select Snapshot 2, then in the dropdown choose "Comparison" — shows what was added/retained compared to Snapshot 1.
6. Look for objects with large "Delta" values — objects that grew significantly.
7. Click on suspect objects to see the "Retainer" tree — shows what is holding the reference keeping the object alive.

Key views in the heap snapshot:
- Summary: Objects grouped by constructor — lets you see all Arrays, Closures, HTMLElement instances, etc.
- Comparison: Delta between two snapshots.
- Containment: Object tree from roots.
- Statistics: Pie chart of heap by type.

Finding closure leaks:
- Look for `(closure)` entries in the Summary view.
- Expand a closure to see what variables it is capturing.
- The retainer tree shows what is holding the closure alive.

```js
// Triggering leak for practice:
const listeners = [];
function addLeak() {
  const largeData = new Array(100000).fill("x"); // 100k strings
  document.addEventListener("resize", () => largeData.length); // closure over largeData
  // Without removeEventListener, this grows with each addLeak() call
}
// Taking heap snapshots before and after addLeak() calls shows growing "(closure)" count
```

**Follow-up:** What is the "Timeline" (Allocation instrumentation) view useful for?

It records heap allocations over time as a timeline, letting you identify when specific allocations occur and correlate them with user actions. Unlike snapshots (which are point-in-time), the timeline shows allocation patterns over a period.

**GOTCHA:** Not all object count growth is a leak. Some growth is expected caching, data loading, or normal growth. A leak is characterized by growth that continues even after the "causative" action is reversed — e.g., objects growing even after closing a modal that was supposed to clean up.

---

**Q5. What is layout thrashing and how do you prevent it?** — Hard

**Answer:**
Layout thrashing (also called "forced synchronous layout") occurs when JavaScript alternates between reading and writing DOM layout properties in a way that forces the browser to recalculate layout multiple times within a single frame.

Why it happens: The browser batches layout calculations. When you write to the DOM (changing width, height, class, style), it marks layout as dirty but does not immediately recalculate. If you then READ a layout-triggering property (offsetWidth, scrollTop, getBoundingClientRect), the browser MUST flush and recalculate layout first to give you an accurate value. If this read-write-read-write pattern repeats in a loop, layout recalculates on every read.

```js
// SLOW — layout thrashing: reads and writes interleaved
const items = document.querySelectorAll(".item");
items.forEach(item => {
  item.style.width = item.offsetWidth * 2 + "px"; // READ (offsetWidth) then WRITE
  // Each iteration: read forces layout flush, write invalidates, next read forces flush again
});
// If items.length = 100, this causes 100 layout recalculations

// FAST — batch reads first, then writes:
const widths = Array.from(items).map(item => item.offsetWidth); // READ ALL
items.forEach((item, i) => {
  item.style.width = widths[i] * 2 + "px"; // WRITE ALL
});
// Only 1 layout recalculation (triggered by the first read)
```

Using `requestAnimationFrame` to batch DOM changes:
```js
function updateLayout() {
  requestAnimationFrame(() => {
    // All reads first:
    const measurements = elements.map(el => el.getBoundingClientRect());
    // Then all writes:
    elements.forEach((el, i) => {
      el.style.transform = `translateX(${measurements[i].x + 10}px)`;
    });
  });
}
```

Properties that trigger layout (reading these forces layout flush):
- `offsetWidth`, `offsetHeight`, `offsetTop`, `offsetLeft`, `offsetParent`
- `scrollWidth`, `scrollHeight`, `scrollTop`, `scrollLeft`
- `clientWidth`, `clientHeight`, `clientTop`, `clientLeft`
- `getBoundingClientRect()`, `getComputedStyle()`
- `scrollIntoView()`, `focus()`

**GOTCHA:** Even reading a property ONCE outside of a loop can cause layout thrashing if a DOM write happened earlier in the same frame. The key rule: batch all DOM reads together before any DOM writes.

---

**Q6. What is the difference between `performance.now()` and `Date.now()`?** — Medium

**Answer:**
Both measure time, but for different purposes.

`Date.now()`:
- Returns milliseconds since January 1, 1970 UTC (Unix timestamp).
- Precision: Millisecond-level (1ms resolution) — actually lower precision in many environments due to security concerns.
- Can be affected by system clock changes (NTP synchronization, manual changes).
- Useful for: Timestamps, date arithmetic, recording when events occurred.

`performance.now()`:
- Returns milliseconds since the page/process started, as a high-resolution time.
- Precision: Sub-millisecond (microsecond-level, though browsers limit precision for security).
- Monotonic — never goes backward, not affected by system clock changes.
- Useful for: Benchmarking, measuring elapsed time, animation timing.

```js
// Benchmarking with performance.now():
function benchmark(fn, iterations = 1000) {
  const start = performance.now();
  for (let i = 0; i < iterations; i++) fn();
  const end = performance.now();
  return (end - start) / iterations; // ms per iteration
}

const avgTime = benchmark(() => someExpensiveOperation());
console.log(`Average: ${avgTime.toFixed(4)}ms per call`);

// Date.now() is wrong for benchmarking:
const t1 = Date.now(); // May be affected by clock adjustments mid-measurement
doWork();
const t2 = Date.now();
// t2 - t1 could be negative if the system clock was adjusted backward!
```

Browser precision limiting:
- Modern browsers (since ~2018) reduce `performance.now()` precision to 100-microsecond resolution (0.1ms) to mitigate Spectre/side-channel timing attacks.
- `SharedArrayBuffer` access restores higher precision but requires cross-origin isolation headers.

**GOTCHA:** `performance.now()` is not the same in different browser tabs or workers — the origin is the page/worker start, not a shared epoch. Do not compare `performance.now()` values across different browsing contexts.

---

**Q7. What are typed arrays and why are they more performant than regular arrays for numeric data?** — Medium

**Answer:**
Typed arrays (Int8Array, Uint8Array, Int16Array, Uint16Array, Int32Array, Uint32Array, Float32Array, Float64Array, BigInt64Array, BigUint64Array) store binary data of a fixed numeric type in a contiguous block of memory.

Why they outperform regular arrays for numeric work:

1. Fixed type: Every element is exactly the same C-type. No boxing (HeapNumber), no type checking, no hidden classes needed.
2. Contiguous memory: Elements are stored back-to-back in an ArrayBuffer. CPU cache lines are used optimally.
3. No holes: No HOLEY element kinds — always dense/packed.
4. Direct hardware operations: CPUs have SIMD instructions optimized for contiguous typed data — JIT compilers can generate vectorized code.

```js
// Regular array — each number may be a HeapObject, not contiguous:
const regular = new Array(1_000_000).fill(0);

// Typed array — 4 bytes per element, fully contiguous 4MB buffer:
const typed = new Float32Array(1_000_000);

// Benchmark — typed arrays are typically 5-20x faster for numeric operations:
function sumTyped(arr) {
  let total = 0;
  for (let i = 0; i < arr.length; i++) total += arr[i];
  return total;
}
```

ArrayBuffer — the raw memory:
```js
const buffer = new ArrayBuffer(16); // 16 bytes of raw memory

// View the same buffer as different types:
const int32View = new Int32Array(buffer);  // 4 elements of 4 bytes each
const uint8View = new Uint8Array(buffer);  // 16 elements of 1 byte each
const float64View = new Float64Array(buffer); // 2 elements of 8 bytes each

// Writing via one view, reading via another (endian-aware):
int32View[0] = 0x01020304;
uint8View[0]; // 4 (little-endian: 0x04 is the first byte on x86)
```

Real-world uses: WebGL (vertex data), Web Audio API (audio samples), video processing, cryptography, scientific computing, WASM data exchange.

**GOTCHA:** Typed arrays do not support `push`, `pop`, `splice`, or other resizing methods — they are fixed-size. They also do not support `undefined` or `null` as values — reading an unset element gives `0`. Use `Array.from(typedArray)` to convert to a regular array if you need these features.

---

**Q8. What is the difference between `reflow` and `repaint` in the browser rendering pipeline?** — Medium

**Answer:**
The browser rendering pipeline has several stages. Understanding which operations trigger which stages is critical for performance.

Reflow (also called Layout):
- The browser recalculates the position and dimensions of elements in the document.
- Triggered by: changes that affect layout — adding/removing DOM nodes, changing dimensions (width, height, margin, padding, border), changing fonts, changing element visibility (display:none vs display:block), reading layout properties (which forces layout flush).
- Expensive because: a single element change can cause cascading recalculations through all its descendants and sometimes ancestors.
- Scope: Can be local (only part of the tree) or global (entire page).

Repaint:
- The browser redraws pixels for elements whose visual appearance changed but whose layout did NOT change.
- Triggered by: changes to color, background, visibility:hidden (vs display:none), box-shadow, border-color, outline.
- Less expensive than reflow — no position/size recalculations needed, but still requires compositing.

Compositing (fastest):
- Some CSS properties (transform, opacity) are handled entirely on the GPU compositor thread without touching the main thread.
- These properties do NOT trigger reflow or repaint — the GPU repositions/fades existing rendered layers.
- Use `transform: translateX(100px)` instead of `left: 100px` for animations.

Performance hierarchy (fastest to slowest):
```
Compositing (transform, opacity) < Repaint (color changes) < Reflow (layout changes)
```

```js
// Triggering reflow (slow):
element.style.width = "200px";    // reflow + repaint
element.style.marginTop = "10px"; // reflow + repaint

// Triggering repaint only (medium):
element.style.color = "red";      // repaint only, no reflow

// Compositor-only (fast):
element.style.transform = "translateX(200px)"; // GPU compositor, no reflow/repaint
element.style.opacity = "0.5";    // GPU compositor, no reflow/repaint
```

**GOTCHA:** `visibility: hidden` triggers repaint (the element takes up space but is invisible). `display: none` triggers reflow (the element is removed from layout entirely and takes no space). `opacity: 0` is compositor-only — no reflow, no repaint, just GPU compositing.

---

**Q9. How do you implement an LRU (Least Recently Used) cache in JavaScript?** — Hard

**Answer:**
An LRU cache evicts the least recently used entry when capacity is exceeded. The optimal implementation uses a doubly-linked list for O(1) ordering combined with a Map for O(1) lookup.

```js
class LRUCache {
  #capacity;
  #map = new Map(); // key -> Node

  // Doubly linked list (head = most recent, tail = least recent):
  #head = { key: null, value: null, prev: null, next: null };
  #tail = { key: null, value: null, prev: null, next: null };

  constructor(capacity) {
    this.#capacity = capacity;
    this.#head.next = this.#tail;
    this.#tail.prev = this.#head;
  }

  #addToFront(node) {
    node.prev = this.#head;
    node.next = this.#head.next;
    this.#head.next.prev = node;
    this.#head.next = node;
  }

  #removeNode(node) {
    node.prev.next = node.next;
    node.next.prev = node.prev;
  }

  get(key) {
    if (!this.#map.has(key)) return -1;
    const node = this.#map.get(key);
    this.#removeNode(node);
    this.#addToFront(node); // Mark as most recently used
    return node.value;
  }

  put(key, value) {
    if (this.#map.has(key)) {
      const node = this.#map.get(key);
      node.value = value;
      this.#removeNode(node);
      this.#addToFront(node);
      return;
    }

    if (this.#map.size >= this.#capacity) {
      // Evict LRU: the node just before tail
      const lru = this.#tail.prev;
      this.#removeNode(lru);
      this.#map.delete(lru.key);
    }

    const newNode = { key, value, prev: null, next: null };
    this.#addToFront(newNode);
    this.#map.set(key, newNode);
  }

  get size() { return this.#map.size; }
}

const cache = new LRUCache(3);
cache.put("a", 1);
cache.put("b", 2);
cache.put("c", 3);
cache.get("a");    // 1 — 'a' is now most recently used
cache.put("d", 4); // evicts 'b' (least recently used)
cache.get("b");    // -1 — evicted
```

Simpler but slightly less performant — using Map's insertion order:
```js
class SimpleLRU {
  #capacity;
  #map = new Map();

  constructor(capacity) { this.#capacity = capacity; }

  get(key) {
    if (!this.#map.has(key)) return -1;
    const value = this.#map.get(key);
    this.#map.delete(key);
    this.#map.set(key, value); // Re-insert to end = most recent
    return value;
  }

  put(key, value) {
    if (this.#map.has(key)) this.#map.delete(key);
    if (this.#map.size >= this.#capacity) {
      this.#map.delete(this.#map.keys().next().value); // Delete oldest (first key)
    }
    this.#map.set(key, value);
  }
}
```

**GOTCHA:** Map preserves insertion order, and `.keys().next().value` gives the first inserted key — which is the LRU in the simple implementation. This works correctly because Map maintains insertion order. The simple version is O(n) for `put` with a full cache because `Map.delete` + `Map.set` moves an entry, but JavaScript Map operations are O(1) amortized, so the overall complexity is still acceptable.

---

**Q10. What is the performance cost of closures and how do you minimize it?** — Medium

**Answer:**
Closures have costs in three areas:

1. Memory: The closure holds a reference to its entire outer Lexical Environment, keeping all closed-over variables alive. If a closure closes over a large object but only uses one property, the entire large object stays in memory.

2. Allocation: Each function definition allocates a new closure object (the function object plus its captured environment). In hot code, many closures per second increase GC pressure.

3. Property lookup: Accessing closed-over variables requires a scope chain traversal — going up to the outer environment. This is usually fast (V8 ICs cache it) but deeper chains cost more.

```js
// Expensive: closure created in a hot loop, closes over large data
function processItems(items, largeConfig) {
  return items.map(item => {
    // New closure per item, each captures 'largeConfig'
    return transform(item, largeConfig.settings, largeConfig.rules);
  });
}

// Better: extract what you need from large object outside the closure
function processItems(items, largeConfig) {
  const { settings, rules } = largeConfig; // extract once
  return items.map(item => transform(item, settings, rules));
  // Closures now capture only two small variables, not 'largeConfig'
}

// Even better for hot loops: avoid closures entirely
function processItems(items, settings, rules) {
  const result = [];
  for (let i = 0; i < items.length; i++) {
    result.push(transform(items[i], settings, rules));
  }
  return result;
}
```

V8 optimization of closures: V8 can sometimes analyze which variables in an outer scope are actually used by a closure and only capture those (not the entire environment). But this optimization is not guaranteed and should not be relied upon.

**GOTCHA:** The classic "closure in a loop" performance mistake: creating many similar closures that each close over the same data but could have been factored out. Each closure is a separate allocation. Prefer factory functions that take parameters over closures that implicitly capture them.

---

**Q11. What is the `performance.mark()` and `performance.measure()` API?** — Medium

**Answer:**
The Performance API provides precise browser-native timing tools that integrate with DevTools' Performance timeline.

```js
// Mark a point in time:
performance.mark("start-computation");

// ... do work ...
const result = heavyComputation();

performance.mark("end-computation");

// Measure the duration between two marks:
performance.measure("computation-duration", "start-computation", "end-computation");

// Get the measurement:
const measures = performance.getEntriesByName("computation-duration");
console.log(`Took: ${measures[0].duration.toFixed(2)}ms`);

// Get all performance entries:
performance.getEntriesByType("measure"); // all measures
performance.getEntries();                // all entries (marks, measures, resource timings)

// Clear marks when done:
performance.clearMarks();
performance.clearMeasures();
```

The Performance Timeline includes:
- Navigation timings (DOMContentLoaded, load, TTFB)
- Resource timings (when each script, image, font loaded)
- User marks and measures
- Long tasks (tasks > 50ms that block the main thread)
- Paint timings (First Contentful Paint, Largest Contentful Paint)

```js
// Observe paint timings:
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.log(entry.name, entry.startTime);
    // "first-contentful-paint", "largest-contentful-paint"
  }
});
observer.observe({ type: "paint", buffered: true });
```

**GOTCHA:** `performance.mark` names are shared across the page. Multiple calls to `performance.mark("start")` create multiple entries all named "start." Use `performance.getEntriesByName("start")` to get all of them. In production code, use namespaced mark names like `"mylib:start-process"`.

---

**Q12. What are Long Tasks and how do you detect and fix them?** — Hard

**Answer:**
A Long Task is any task that takes longer than 50ms to complete on the main thread. Long tasks block user interaction, cause janky animations, and degrade Core Web Vitals (specifically Total Blocking Time and Interaction to Next Paint).

Detecting Long Tasks:
```js
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.log(`Long task: ${entry.duration.toFixed(1)}ms`);
    console.log("Attribution:", entry.attribution);
    // Attribution shows which script/frame caused the long task
  }
});
observer.observe({ type: "longtask", buffered: true });
```

Breaking up long tasks using `scheduler.yield()` (modern) or setTimeout:
```js
// SLOW — single long task blocking everything:
function processAllItems(items) {
  for (const item of items) {
    expensiveProcessing(item); // if 10,000 items × 1ms = 10,000ms task
  }
}

// FAST — yield to browser between chunks:
async function processAllItemsAsync(items) {
  const CHUNK_SIZE = 50;

  for (let i = 0; i < items.length; i += CHUNK_SIZE) {
    const chunk = items.slice(i, i + CHUNK_SIZE);
    chunk.forEach(expensiveProcessing);

    // Yield to the browser — allows rendering, user interactions, etc.:
    if (i + CHUNK_SIZE < items.length) {
      await scheduler.yield(); // Modern browsers
      // Fallback: await new Promise(resolve => setTimeout(resolve, 0));
    }
  }
}
```

`scheduler.yield()` (Scheduler API):
- Pauses the current task and returns control to the browser.
- Has higher priority than `setTimeout(fn, 0)` — the continuation runs in the next task with "user-visible" priority.
- Prevents UI jank caused by long processing loops.

**GOTCHA:** Breaking a long task into smaller tasks using `setTimeout(fn, 0)` works but has a minimum delay (~1ms). Each chunk becomes a separate macrotask. If you have 1000 chunks, that is 1000 macrotasks with ~1ms delays each — potentially 1 second of actual clock time, even if the CPU work is fast. Use `scheduler.yield()` for better scheduling control.

---

**Q13. What is the impact of excessive closures on garbage collection?** — Hard

**Answer:**
Closures prevent garbage collection of their captured variables. This creates two categories of problems:

1. Expected retention (correct but expensive):
```js
function createHandlers() {
  const cache = new Map(); // Legitimately needed
  return {
    process: (key) => cache.get(key),
    store: (key, val) => cache.set(key, val)
  };
}
// cache lives as long as either closure is alive — correct behavior
```

2. Unexpected retention (memory leak):
```js
function attachHandler(element) {
  const context = {
    largeData: new Array(100000).fill("x"), // 100K strings
    userId: "123" // the only thing actually needed
  };

  element.addEventListener("click", () => {
    // Only uses userId, but 'context' (including largeData) is captured
    sendEvent(context.userId);
  });
}

// Fix: capture only what is needed
function attachHandler(element) {
  const userId = "123"; // extract only needed value
  const largeData = new Array(100000).fill("x");
  doSomethingWith(largeData); // use it here, then it can be GC'd
  // largeData is NOT captured by the closure
  element.addEventListener("click", () => sendEvent(userId));
}
```

V8 closure variable capture optimization: V8 performs "context allocation analysis" — if a closure only uses some variables from the outer scope, it may create a context object that contains only those variables (not all of them). However, this optimization is not guaranteed and depends on the complexity of the scope. Do not rely on it.

The closure chain problem:
```js
function outer() {
  const huge = new Array(1_000_000); // 1M items
  function middle() {
    const medium = new Array(1_000); // 1K items — middle captures 'huge' via outer's context
    function inner() {
      return medium.length; // inner captures 'medium' AND 'huge' via the context chain
    }
    return inner;
  }
  return middle;
}
// inner() keeps both 'huge' and 'medium' alive via the lexical scope chain
```

**GOTCHA:** Even if an inner function does not explicitly reference a variable from an outer scope, V8 may still keep the variable alive if it appears in the outer function's scope and V8 cannot prove it is unnecessary (e.g., due to `eval` in the scope chain, or complex control flow). Simplify scopes to help the optimizer.

---

**Q14. What is `structuredClone` and how does it work internally?** — Medium

**Answer:**
`structuredClone(value)` is a global function (ES2022) that performs a complete deep clone using the "Structured Clone Algorithm" — the same algorithm used internally for `postMessage`, IndexedDB storage, and `history.pushState`.

What it handles correctly (unlike `JSON.parse(JSON.stringify())`):
- `Date` objects — cloned as Date, not converted to string
- `RegExp` — cloned with flags
- `Map` and `Set` — cloned with all entries
- `ArrayBuffer`, `TypedArray`, `DataView` — cloned as binary data
- Circular references — handled without infinite loops
- `Error` objects — cloned with message and stack

What it cannot clone:
- Functions (throws `DataCloneError`)
- DOM nodes (throws `DataCloneError`)
- `Symbol` values (throws `DataCloneError`)
- Properties of `WeakMap`, `WeakSet`, `WeakRef`
- Object prototypes — the clone always has `Object.prototype` as its prototype (class instances lose their class)

```js
// Correct deep clone:
const original = {
  date: new Date(),
  regex: /hello/gi,
  map: new Map([["a", 1]]),
  buffer: new ArrayBuffer(8),
  nested: { arr: [1, 2, 3] }
};

const clone = structuredClone(original);
clone.date instanceof Date;          // true
clone.map instanceof Map;            // true
clone.date === original.date;        // false — separate objects
clone.nested === original.nested;    // false — deep copy
clone.nested.arr === original.nested.arr; // false — deep copy

// Circular reference handling:
const circular = {};
circular.self = circular;
const clonedCircular = structuredClone(circular); // Works!
clonedCircular.self === clonedCircular; // true — circular reference preserved
```

Transferable objects — move instead of copy (for performance):
```js
const buffer = new ArrayBuffer(1024 * 1024); // 1MB
// Transfer ownership — buffer becomes detached in the original context:
const clone = structuredClone(buffer, { transfer: [buffer] });
buffer.byteLength; // 0 — transferred, original is now detached
clone.byteLength;  // 1048576 — clone has the data
```

**GOTCHA:** Class instances lose their prototype via `structuredClone`. `structuredClone(new MyClass())` returns a plain object with the same properties, not an instance of `MyClass`. If you need to clone class instances preserving their type, implement a custom `clone()` method.

---

**Q15. What is garbage collection tuning and what Node.js flags control GC behavior?** — Hard

**Answer:**
While you cannot control JavaScript's garbage collector directly, Node.js exposes V8 flags that allow you to tune GC behavior for specific workloads.

Key V8 GC flags in Node.js:

`--max-old-space-size=<MB>`: Sets the maximum size of the old generation heap in MB. Default is approximately 1.5GB on 64-bit systems. Increase for memory-intensive applications.
```bash
node --max-old-space-size=4096 app.js  # 4GB old space
```

`--max-semi-space-size=<MB>`: Sets the size of each semi-space in the young generation. Larger semi-spaces reduce GC frequency but increase GC pause duration.
```bash
node --max-semi-space-size=64 app.js  # 64MB semi-spaces
```

`--gc-interval=<N>`: Force GC every N allocations (for debugging only).

`--expose-gc`: Exposes `global.gc()` function for manual GC triggering (debugging only):
```bash
node --expose-gc app.js
```
```js
// In code — force GC (debugging/testing only, never in production):
global.gc();
```

Programmatic heap stats:
```js
const v8 = require("v8");
console.log(v8.getHeapStatistics());
// {
//   total_heap_size: N,
//   used_heap_size: N,
//   external_memory: N,
//   heap_size_limit: N,
//   ...
// }

// Monitor for memory issues:
setInterval(() => {
  const { used_heap_size, heap_size_limit } = v8.getHeapStatistics();
  const usagePercent = (used_heap_size / heap_size_limit) * 100;
  if (usagePercent > 90) {
    console.warn(`Heap at ${usagePercent.toFixed(1)}% — potential memory pressure`);
  }
}, 30000);
```

**Follow-up:** When should you increase `--max-old-space-size`?

When your application legitimately needs to hold large amounts of data in memory (large in-memory databases, image processing, machine learning inference) AND you have confirmed there are no memory leaks via heap profiling. Increasing this value without fixing leaks just delays the inevitable crash.

**GOTCHA:** Calling `global.gc()` in production code defeats the purpose of automatic memory management and can cause significant performance degradation. V8's GC timing is carefully tuned — forcing GC at arbitrary times disrupts those heuristics. Only use it in tests to verify that objects are actually collectible, or in benchmarks to start from a clean state.

---

*Next: [08-Concurrency-Workers.md](./08-Concurrency-Workers.md)*
