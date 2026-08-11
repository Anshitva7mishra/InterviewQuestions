# 19 — Scenario-Based / System Design
### 15 Questions | Advanced

> These questions test how you apply JavaScript knowledge to real architectural problems.
> There's no single right answer — explain your reasoning, trade-offs, and why.

---

**Q1. Design an autocomplete/typeahead search component. What JS patterns do you apply?** — Hard

**Answer:**

Key concerns: performance (API call throttling), UX (smooth updates), accessibility (keyboard navigation), and correctness (stale responses).

```js
class AutoComplete {
  #inputEl;
  #listEl;
  #controller = null;  // AbortController for cancellation
  #cache = new Map();
  #selectedIndex = -1;

  constructor(inputSelector, listSelector, fetchFn) {
    this.#inputEl = document.querySelector(inputSelector);
    this.#listEl = document.querySelector(listSelector);
    this.fetchFn = fetchFn;

    // Debounce: wait until user pauses typing (300ms):
    this.#inputEl.addEventListener("input", debounce(
      e => this.#search(e.target.value), 300
    ));

    // Keyboard navigation:
    this.#inputEl.addEventListener("keydown", e => this.#onKeyDown(e));
  }

  async #search(query) {
    const q = query.trim();
    if (q.length < 2) { this.#clear(); return; }

    // Cache hit — no network call needed:
    if (this.#cache.has(q)) {
      return this.#render(this.#cache.get(q));
    }

    // Cancel any in-flight request:
    this.#controller?.abort();
    this.#controller = new AbortController();

    try {
      const results = await this.fetchFn(q, this.#controller.signal);
      this.#cache.set(q, results); // cache result
      this.#render(results);
    } catch (err) {
      if (err.name !== "AbortError") {
        console.error("Search failed:", err);
      }
      // AbortError means a newer search superseded this one — ignore
    }
  }

  #render(items) {
    this.#selectedIndex = -1;
    this.#listEl.innerHTML = items
      .map((item, i) =>
        `<li role="option" data-index="${i}" tabindex="-1">${item.label}</li>`
      ).join("");
  }

  #onKeyDown(e) {
    const items = this.#listEl.querySelectorAll("li");
    if (!items.length) return;

    if (e.key === "ArrowDown") {
      e.preventDefault();
      this.#selectedIndex = Math.min(this.#selectedIndex + 1, items.length - 1);
      items[this.#selectedIndex]?.focus();
    } else if (e.key === "ArrowUp") {
      e.preventDefault();
      this.#selectedIndex = Math.max(this.#selectedIndex - 1, 0);
      items[this.#selectedIndex]?.focus();
    } else if (e.key === "Escape") {
      this.#clear();
    }
  }

  #clear() {
    this.#listEl.innerHTML = "";
    this.#selectedIndex = -1;
  }
}
```

**Patterns applied:**
- **Debounce** — prevent API call on every keystroke
- **AbortController** — cancel stale requests
- **Cache** — avoid duplicate network calls for same query
- **Private class fields** — encapsulation
- **Accessibility** — `role="option"`, keyboard navigation

---

**Q2. You have a `fetchUser(id)` function that's called thousands of times. How do you optimize it?** — Hard

**Answer:**

```js
// Problem: multiple components call fetchUser(123) independently
// Solution: request deduplication + cache with TTL

class UserStore {
  #cache = new Map();            // key: id, value: { data, expiry }
  #pending = new Map();          // key: id, value: Promise (in-flight)
  #ttl;

  constructor(ttlMs = 60_000) {
    this.#ttl = ttlMs;
  }

  async get(id) {
    // 1. Cache hit (not expired):
    const cached = this.#cache.get(id);
    if (cached && Date.now() < cached.expiry) {
      return cached.data;
    }

    // 2. Request deduplication — if already in flight, wait for it:
    if (this.#pending.has(id)) {
      return this.#pending.get(id);
    }

    // 3. New network request:
    const promise = fetch(`/api/users/${id}`)
      .then(r => r.json())
      .then(data => {
        this.#cache.set(id, { data, expiry: Date.now() + this.#ttl });
        this.#pending.delete(id); // remove from in-flight
        return data;
      })
      .catch(err => {
        this.#pending.delete(id); // clean up on failure too
        throw err;
      });

    this.#pending.set(id, promise);
    return promise;
  }

  invalidate(id) { this.#cache.delete(id); }
  clear() { this.#cache.clear(); this.#pending.clear(); }
}

const userStore = new UserStore(30_000); // 30 second TTL

// 50 simultaneous calls for the same user → only 1 network request:
await Promise.all(Array.from({ length: 50 }, () => userStore.get(42)));
```

**Key concepts:** request deduplication (one in-flight Promise shared), TTL cache, cache invalidation.

---

**Q3. Implement a client-side rate limiter for API calls (max N requests per second).** — Hard

**Answer:**

```js
class RateLimiter {
  #queue = [];
  #running = 0;
  #maxConcurrent;
  #intervalMs;
  #callCount = 0;
  #windowStart = Date.now();
  #maxPerWindow;

  constructor({ maxConcurrent = 5, maxPerSecond = 10 } = {}) {
    this.#maxConcurrent = maxConcurrent;
    this.#maxPerWindow = maxPerSecond;
    this.#intervalMs = 1000;
  }

  async schedule(fn) {
    return new Promise((resolve, reject) => {
      this.#queue.push({ fn, resolve, reject });
      this.#drain();
    });
  }

  #drain() {
    while (this.#queue.length && this.#canRun()) {
      const { fn, resolve, reject } = this.#queue.shift();
      this.#running++;
      this.#callCount++;

      Promise.resolve()
        .then(() => fn())
        .then(resolve, reject)
        .finally(() => {
          this.#running--;
          this.#drain(); // process next queued item
        });
    }
  }

  #canRun() {
    const now = Date.now();
    // Reset window:
    if (now - this.#windowStart >= this.#intervalMs) {
      this.#callCount = 0;
      this.#windowStart = now;
    }
    return this.#running < this.#maxConcurrent &&
           this.#callCount < this.#maxPerWindow;
  }
}

const limiter = new RateLimiter({ maxConcurrent: 3, maxPerSecond: 5 });

// All these are automatically queued and rate-limited:
const results = await Promise.all(
  urls.map(url => limiter.schedule(() => fetch(url).then(r => r.json())))
);
```

---

**Q4. How would you implement an offline-first feature with background sync?** — Hard

**Answer:**

```js
// Service Worker + IndexedDB + Background Sync API

// In service-worker.js:
const DB_NAME = "offline-queue";
const STORE_NAME = "pending-requests";

// Register sync event:
self.addEventListener("sync", async (event) => {
  if (event.tag === "sync-pending-requests") {
    event.waitUntil(syncPendingRequests());
  }
});

async function syncPendingRequests() {
  const db = await openDB(DB_NAME);
  const tx = db.transaction(STORE_NAME, "readwrite");
  const store = tx.objectStore(STORE_NAME);
  const all = await store.getAll();

  for (const request of all) {
    try {
      await fetch(request.url, {
        method: request.method,
        headers: request.headers,
        body: request.body
      });
      await store.delete(request.id); // success — remove from queue
    } catch (err) {
      // Will retry on next sync event
      console.error("Sync failed for:", request.url);
    }
  }
}

// In main app — queue request if offline:
async function submitForm(data) {
  if (navigator.onLine) {
    // Online — send immediately:
    return fetch("/api/submit", {
      method: "POST",
      body: JSON.stringify(data)
    });
  }

  // Offline — save to IndexedDB:
  const db = await openDB(DB_NAME);
  await db.add(STORE_NAME, {
    id: crypto.randomUUID(),
    url: "/api/submit",
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(data),
    timestamp: Date.now()
  });

  // Register background sync:
  const registration = await navigator.serviceWorker.ready;
  await registration.sync.register("sync-pending-requests");

  return { queued: true }; // inform UI
}
```

**Trade-offs:** Background sync may delay request by minutes. Need to handle conflict resolution when syncing. IndexedDB has a ~50MB default storage limit. Always show the user their data was saved locally.

---

**Q5. Design a real-time collaborative cursor-sharing feature (like Figma's multiplayer cursors).** — Hard

**Answer:**

```js
// Client-side — WebSocket-based cursor sharing

class CursorSync {
  #ws;
  #cursors = new Map(); // userId → DOM element
  #userId;
  #container;
  #throttledSend;

  constructor(wsUrl, container) {
    this.#userId = crypto.randomUUID();
    this.#container = container;
    this.#ws = new WebSocket(wsUrl);
    this.#throttledSend = throttle(data => this.#send(data), 50); // max 20 updates/sec

    this.#ws.addEventListener("message", e => this.#onMessage(JSON.parse(e.data)));
    this.#container.addEventListener("mousemove", e => this.#onMouseMove(e));
    this.#ws.addEventListener("close", () => this.#cleanup());
  }

  #onMouseMove(e) {
    const rect = this.#container.getBoundingClientRect();
    this.#throttledSend({
      type: "cursor",
      userId: this.#userId,
      x: ((e.clientX - rect.left) / rect.width) * 100,  // percentage
      y: ((e.clientY - rect.top) / rect.height) * 100,
    });
  }

  #onMessage({ type, userId, x, y, name, color }) {
    if (userId === this.#userId) return; // ignore own cursor

    if (type === "cursor") {
      this.#moveCursor(userId, x, y);
    } else if (type === "leave") {
      this.#removeCursor(userId);
    }
  }

  #moveCursor(userId, x, y) {
    if (!this.#cursors.has(userId)) {
      // Create cursor element:
      const el = document.createElement("div");
      el.className = "remote-cursor";
      el.style.cssText = `position:absolute; pointer-events:none; transition:left 50ms,top 50ms;`;
      this.#container.appendChild(el);
      this.#cursors.set(userId, el);
    }

    const el = this.#cursors.get(userId);
    // requestAnimationFrame for smooth rendering:
    requestAnimationFrame(() => {
      el.style.left = `${x}%`;
      el.style.top = `${y}%`;
    });
  }

  #removeCursor(userId) {
    this.#cursors.get(userId)?.remove();
    this.#cursors.delete(userId);
  }

  #send(data) {
    if (this.#ws.readyState === WebSocket.OPEN) {
      this.#ws.send(JSON.stringify(data));
    }
  }

  #cleanup() {
    this.#cursors.forEach(el => el.remove());
    this.#cursors.clear();
  }
}
```

**Key decisions:**
- **Throttle** mouse events to 20fps (50ms) — cursor positions at 60fps create too much traffic
- **Percentage coordinates** — work across different viewport sizes
- **`requestAnimationFrame`** — smooth cursor movement
- **CSS transition** — interpolates between position updates for fluid movement
- **Cleanup on disconnect** — remove stale cursors

---

**Q6. You have a heavy computation that blocks the UI thread. How do you fix it?** — Hard

**Answer:**

Three approaches, in increasing complexity:

```js
// Approach 1: Break work into chunks with setTimeout (simplest):
function processInChunks(items, processFn, chunkSize = 100) {
  let index = 0;

  return new Promise((resolve) => {
    const results = [];

    function processChunk() {
      const end = Math.min(index + chunkSize, items.length);
      while (index < end) {
        results.push(processFn(items[index++]));
      }

      if (index < items.length) {
        setTimeout(processChunk, 0); // yield to event loop
      } else {
        resolve(results);
      }
    }
    processChunk();
  });
}

// Approach 2: scheduler.postTask (modern browser API):
async function processWithScheduler(items) {
  const results = [];
  for (const item of items) {
    results.push(processItem(item));
    // Yield to browser between items if it has higher-priority work:
    await scheduler.postTask(() => {}, { priority: "background" });
  }
  return results;
}

// Approach 3: Web Worker (true parallelism — best for CPU-bound tasks):
// worker.js:
self.addEventListener("message", ({ data: { items } }) => {
  const results = items.map(heavyCompute);
  self.postMessage({ results });
});

// main.js:
function processInWorker(items) {
  return new Promise((resolve, reject) => {
    const worker = new Worker("./worker.js");
    worker.postMessage({ items });
    worker.addEventListener("message", ({ data }) => {
      resolve(data.results);
      worker.terminate();
    });
    worker.addEventListener("error", reject);
  });
}

// For large data — use SharedArrayBuffer to avoid copying:
const sharedBuffer = new SharedArrayBuffer(items.length * 4);
const sharedArray = new Int32Array(sharedBuffer);
```

**When to use each:**
- `setTimeout` chunking: Simple, no transfer cost, but still single-threaded
- `scheduler.postTask`: Cooperative with browser, priority control
- Web Worker: True parallel execution, best for math-heavy work (image processing, compression, parsing)

---

**Q7. How would you implement infinite scroll with dynamic content loading?** — Medium

**Answer:**

```js
class InfiniteScroll {
  #page = 1;
  #loading = false;
  #exhausted = false;
  #observer;
  #container;
  #sentinel;
  #fetchPage;

  constructor(container, fetchPage) {
    this.#container = container;
    this.#fetchPage = fetchPage;

    // Create sentinel element at the bottom of the list:
    this.#sentinel = document.createElement("div");
    this.#sentinel.setAttribute("aria-hidden", "true");
    container.appendChild(this.#sentinel);

    this.#observer = new IntersectionObserver(
      entries => { if (entries[0].isIntersecting) this.#loadNext(); },
      { rootMargin: "200px" } // trigger 200px before visible
    );
    this.#observer.observe(this.#sentinel);

    this.#loadNext(); // load first page immediately
  }

  async #loadNext() {
    if (this.#loading || this.#exhausted) return;
    this.#loading = true;
    this.#showSkeleton();

    try {
      const { items, hasMore } = await this.#fetchPage(this.#page);
      this.#hideSkeleton();
      this.#render(items);
      this.#page++;

      if (!hasMore) {
        this.#exhausted = true;
        this.#observer.disconnect();
        this.#showEndMessage();
      }
    } catch (err) {
      this.#hideSkeleton();
      this.#showError(err);
    } finally {
      this.#loading = false;
    }
  }

  #render(items) {
    const fragment = document.createDocumentFragment();
    items.forEach(item => {
      const el = this.#createCard(item);
      fragment.appendChild(el);
    });
    // Insert BEFORE the sentinel to keep it at the bottom:
    this.#container.insertBefore(fragment, this.#sentinel);
  }

  #showSkeleton() { /* add skeleton loader cards */ }
  #hideSkeleton() { /* remove skeleton cards */ }
  #showEndMessage() { /* "You've reached the end" */ }
  #showError(err) { /* "Failed to load. Retry?" */ }
}
```

**Key decisions:** `DocumentFragment` for batched DOM insertion. `rootMargin: "200px"` pre-fetches before user reaches the bottom. Loading flag prevents concurrent fetches.

---

**Q8. Explain how you would architect a client-side state management solution from scratch.** — Hard

**Answer:**

```js
// Minimal Redux-like store:

function createStore(reducer, initialState, ...middlewares) {
  let state = initialState;
  const listeners = new Set();

  // Apply middleware (enhance dispatch):
  let dispatch = (action) => {
    state = reducer(state, action);
    listeners.forEach(fn => fn(state));
    return action;
  };

  // Compose middlewares:
  if (middlewares.length) {
    const chain = middlewares.map(m => m({ getState, dispatch: d => dispatch(d) }));
    dispatch = chain.reduceRight((next, middleware) => middleware(next), dispatch);
  }

  function getState() { return state; }

  function subscribe(fn) {
    listeners.add(fn);
    fn(state); // immediately call with current state
    return () => listeners.delete(fn); // return unsubscribe
  }

  // Initialize with dummy action to populate initial state:
  dispatch({ type: "@@INIT" });

  return { getState, dispatch, subscribe };
}

// Logger middleware:
const logger = store => next => action => {
  console.log("Before:", store.getState());
  const result = next(action);
  console.log("After:", store.getState());
  return result;
};

// Thunk middleware (async actions):
const thunk = store => next => action => {
  if (typeof action === "function") {
    return action(store.dispatch, store.getState);
  }
  return next(action);
};

// Usage:
const store = createStore(
  (state, action) => {
    switch (action.type) {
      case "INCREMENT": return { ...state, count: state.count + 1 };
      case "SET_USER":  return { ...state, user: action.payload };
      default:          return state;
    }
  },
  { count: 0, user: null },
  thunk,
  logger
);

store.subscribe(state => console.log("State updated:", state));
store.dispatch({ type: "INCREMENT" });
store.dispatch(async (dispatch, getState) => {
  const user = await fetchUser(1);
  dispatch({ type: "SET_USER", payload: user });
});
```

---

**Q9. How would you implement a feature flag system in JavaScript?** — Medium

**Answer:**

```js
class FeatureFlags {
  #flags = new Map();
  #overrides = new Map(); // developer overrides via localStorage

  constructor(flagConfig) {
    for (const [name, config] of Object.entries(flagConfig)) {
      this.#flags.set(name, config);
    }
    // Load developer overrides from localStorage:
    try {
      const saved = JSON.parse(localStorage.getItem("feature_flags") ?? "{}");
      Object.entries(saved).forEach(([k, v]) => this.#overrides.set(k, v));
    } catch {}
  }

  isEnabled(flagName, context = {}) {
    // Override takes priority (for testing/dev):
    if (this.#overrides.has(flagName)) {
      return this.#overrides.get(flagName);
    }

    const flag = this.#flags.get(flagName);
    if (!flag) return false;
    if (!flag.enabled) return false;

    // Percentage rollout — stable per user:
    if (flag.rollout !== undefined) {
      const userId = context.userId ?? "anonymous";
      const hash = this.#stableHash(`${flagName}:${userId}`);
      return (hash % 100) < flag.rollout;
    }

    // User allowlist:
    if (flag.allowlist && context.userId) {
      return flag.allowlist.includes(context.userId);
    }

    return true;
  }

  override(flagName, value) {
    this.#overrides.set(flagName, value);
    const saved = Object.fromEntries(this.#overrides);
    localStorage.setItem("feature_flags", JSON.stringify(saved));
  }

  #stableHash(str) {
    let hash = 0;
    for (const char of str) hash = (hash * 31 + char.charCodeAt(0)) & 0xffffffff;
    return Math.abs(hash);
  }
}

const flags = new FeatureFlags({
  "new-dashboard":  { enabled: true, rollout: 20 },   // 20% of users
  "dark-mode":      { enabled: true, rollout: 100 },   // everyone
  "beta-feature":   { enabled: true, allowlist: ["user-123", "user-456"] },
  "old-feature":    { enabled: false }
});

// Usage:
if (flags.isEnabled("new-dashboard", { userId: currentUser.id })) {
  renderNewDashboard();
} else {
  renderOldDashboard();
}

// Dev override in console:
// flags.override("new-dashboard", true) — force-enable for testing
```

---

**Q10. How would you handle and recover from memory leaks in a long-running SPA?** — Hard

**Answer:**

Common memory leak sources and fixes:

```js
// Leak 1: Event listeners never removed:
class Component {
  #handlers = new Map();

  connectedCallback() {
    const handler = e => this.onClick(e);
    this.#handlers.set("click", handler);
    document.addEventListener("click", handler);
  }

  disconnectedCallback() {
    // MUST remove listeners when component is destroyed:
    document.removeEventListener("click", this.#handlers.get("click"));
    this.#handlers.clear();
  }
}

// Leak 2: Timers not cleared:
class PollingComponent {
  #timerId = null;

  start() {
    this.#timerId = setInterval(() => this.poll(), 5000);
  }

  destroy() {
    clearInterval(this.#timerId); // MUST clear on destroy
    this.#timerId = null;
  }
}

// Leak 3: Closures holding large objects:
// BAD:
function initHeavy() {
  const largeData = new Uint8Array(10_000_000); // 10MB
  document.getElementById("btn").addEventListener("click", () => {
    console.log(largeData.length); // closure keeps largeData alive forever!
  });
}

// GOOD — capture only what you need:
function initHeavy() {
  const largeData = new Uint8Array(10_000_000);
  const length = largeData.length; // capture primitive only
  document.getElementById("btn").addEventListener("click", () => {
    console.log(length); // largeData can now be GC'd
  });
}

// Leak 4: WeakRef for optional caches:
class ImageCache {
  #cache = new Map(); // key → WeakRef<Blob>

  set(key, blob) {
    this.#cache.set(key, new WeakRef(blob));
  }

  get(key) {
    const ref = this.#cache.get(key);
    const value = ref?.deref();
    if (!value) this.#cache.delete(key); // clean up dead ref
    return value ?? null;
  }
}

// Detection: Chrome DevTools → Memory → Take Heap Snapshot
// Compare two snapshots — look for growing object counts
// Use Performance Monitor panel to watch JS heap size in real-time
```

**Memory profiling checklist:**
1. Record a heap snapshot before and after navigating
2. Compare — look for retained objects that shouldn't exist
3. Check for detached DOM nodes (elements removed from DOM but held in memory)
4. Use `performance.measureUserAgentSpecificMemory()` (where available) for programmatic tracking

---

**Q11. Design a client-side job queue that runs tasks with priority and concurrency control.** — Hard

**Answer:**

```js
class JobQueue {
  #queues = {
    high:   [],
    normal: [],
    low:    []
  };
  #running = 0;
  #maxConcurrent;
  #paused = false;

  constructor(maxConcurrent = 3) {
    this.#maxConcurrent = maxConcurrent;
  }

  add(job, priority = "normal") {
    return new Promise((resolve, reject) => {
      this.#queues[priority].push({ job, resolve, reject });
      this.#tick();
    });
  }

  #tick() {
    if (this.#paused) return;

    while (this.#running < this.#maxConcurrent) {
      const task = this.#dequeue();
      if (!task) break;

      this.#running++;
      Promise.resolve()
        .then(() => task.job())
        .then(task.resolve, task.reject)
        .finally(() => { this.#running--; this.#tick(); });
    }
  }

  #dequeue() {
    // High → normal → low priority:
    for (const level of ["high", "normal", "low"]) {
      if (this.#queues[level].length) return this.#queues[level].shift();
    }
    return null;
  }

  pause() { this.#paused = true; }
  resume() { this.#paused = false; this.#tick(); }
  get size() {
    return Object.values(this.#queues).reduce((sum, q) => sum + q.length, 0);
  }
}

// Usage:
const queue = new JobQueue(2); // max 2 concurrent jobs

queue.add(() => uploadFile(file1), "high");
queue.add(() => sendAnalytics(data), "low");
queue.add(() => processImage(img), "normal");
// High-priority task runs first, low-priority waits
```

---

**Q12. How do you implement a virtual/windowed list for rendering 100,000 items efficiently?** — Hard

**Answer:**

```js
class VirtualList {
  #container;
  #items;
  #itemHeight;
  #visibleCount;
  #scrollTop = 0;
  #totalHeight;

  constructor(container, items, itemHeight) {
    this.#container = container;
    this.#items = items;
    this.#itemHeight = itemHeight;
    this.#totalHeight = items.length * itemHeight;
    this.#visibleCount = Math.ceil(container.clientHeight / itemHeight) + 2; // buffer

    this.#setup();
    this.#render();
  }

  #setup() {
    // Outer container — holds scroll position:
    this.#container.style.overflowY = "scroll";
    this.#container.style.position = "relative";

    // Inner spacer — sets total scroll height:
    this.spacer = document.createElement("div");
    this.spacer.style.height = `${this.#totalHeight}px`;
    this.#container.appendChild(this.spacer);

    // Viewport — clips to visible area:
    this.viewport = document.createElement("div");
    this.viewport.style.position = "sticky";
    this.viewport.style.top = "0";
    this.#container.prepend(this.viewport);

    this.#container.addEventListener("scroll", throttle(() => {
      this.#scrollTop = this.#container.scrollTop;
      this.#render();
    }, 16)); // ~60fps
  }

  #render() {
    const startIndex = Math.floor(this.#scrollTop / this.#itemHeight);
    const endIndex = Math.min(startIndex + this.#visibleCount, this.#items.length);
    const offsetY = startIndex * this.#itemHeight;

    // Render only visible items:
    const fragment = document.createDocumentFragment();
    for (let i = startIndex; i < endIndex; i++) {
      const el = document.createElement("div");
      el.style.cssText = `
        position: absolute;
        top: ${i * this.#itemHeight - offsetY}px;
        height: ${this.#itemHeight}px;
        width: 100%;
      `;
      el.textContent = this.#items[i].label;
      fragment.appendChild(el);
    }

    this.viewport.style.transform = `translateY(${offsetY}px)`;
    this.viewport.innerHTML = "";
    this.viewport.appendChild(fragment);
  }

  // Update a single item without full re-render:
  updateItem(index, newData) {
    this.#items[index] = newData;
    this.#render(); // could optimize to only update if in visible range
  }
}

// Usage — 100,000 items, smooth 60fps scroll:
const list = new VirtualList(
  document.getElementById("list"),
  Array.from({ length: 100_000 }, (_, i) => ({ label: `Item ${i + 1}` })),
  48 // each item is 48px tall
);
```

**Key concepts:** Only render visible items (O(1) DOM nodes regardless of list size). Spacer div creates correct scroll height. `sticky` positioning pins viewport to top while scrolling.

---

**Q13. How would you implement client-side search with ranking (like a mini Ctrl+P)?** — Medium

**Answer:**

```js
class FuzzySearch {
  #items;
  #index; // pre-built search index

  constructor(items, keys = ["name"]) {
    this.#items = items;
    // Pre-process for faster search:
    this.#index = items.map((item, i) => ({
      i,
      searchable: keys.map(k => (item[k] ?? "").toLowerCase()).join(" ")
    }));
  }

  search(query, maxResults = 50) {
    if (!query.trim()) return this.#items.slice(0, maxResults);

    const q = query.toLowerCase();
    const scored = [];

    for (const { i, searchable } of this.#index) {
      const score = this.#score(searchable, q);
      if (score > 0) scored.push({ score, item: this.#items[i] });
    }

    return scored
      .sort((a, b) => b.score - a.score)
      .slice(0, maxResults)
      .map(({ item }) => item);
  }

  #score(text, query) {
    if (text.startsWith(query)) return 100;     // exact prefix → highest
    if (text.includes(query)) return 80;         // substring match
    if (this.#fuzzyMatch(text, query)) return 60; // fuzzy match
    return 0;
  }

  #fuzzyMatch(text, query) {
    let qi = 0;
    for (let i = 0; i < text.length && qi < query.length; i++) {
      if (text[i] === query[qi]) qi++;
    }
    return qi === query.length; // all query chars found in order
  }
}

// Usage:
const commands = [
  { name: "Open File", shortcut: "Ctrl+O" },
  { name: "Find in Files", shortcut: "Ctrl+Shift+F" },
  { name: "Toggle Terminal", shortcut: "Ctrl+`" }
];

const searcher = new FuzzySearch(commands, ["name"]);
searcher.search("fil");   // [{ name: "Open File" }, { name: "Find in Files" }]
searcher.search("trm");   // [{ name: "Toggle Terminal" }] — fuzzy match
```

---

**Q14. You're building a form with complex validation. Design a reusable validation system.** — Medium

**Answer:**

```js
// Composable validator functions:
const validators = {
  required: msg => value =>
    (value !== null && value !== undefined && value !== "")
      ? null : (msg ?? "This field is required"),

  minLength: (n, msg) => value =>
    value.length >= n ? null : (msg ?? `Minimum ${n} characters`),

  maxLength: (n, msg) => value =>
    value.length <= n ? null : (msg ?? `Maximum ${n} characters`),

  pattern: (regex, msg) => value =>
    regex.test(value) ? null : (msg ?? "Invalid format"),

  email: msg => validators.pattern(
    /^[^\s@]+@[^\s@]+\.[^\s@]+$/,
    msg ?? "Invalid email address"
  ),

  min: (n, msg) => value =>
    Number(value) >= n ? null : (msg ?? `Minimum value is ${n}`),

  custom: (fn, msg) => value => fn(value) ? null : msg
};

// Field validator — runs multiple validators:
function validateField(value, rules) {
  for (const rule of rules) {
    const error = rule(value);
    if (error) return error; // return first error
  }
  return null;
}

// Form validator:
function validateForm(data, schema) {
  const errors = {};
  let isValid = true;

  for (const [field, rules] of Object.entries(schema)) {
    const error = validateField(data[field], rules);
    if (error) {
      errors[field] = error;
      isValid = false;
    }
  }

  return { isValid, errors };
}

// Usage:
const schema = {
  name: [
    validators.required(),
    validators.minLength(2),
    validators.maxLength(50)
  ],
  email: [
    validators.required(),
    validators.email()
  ],
  age: [
    validators.required("Age is required"),
    validators.min(18, "Must be 18 or older"),
    validators.custom(v => Number.isInteger(Number(v)), "Must be a whole number")
  ]
};

const result = validateForm({ name: "Al", email: "bad", age: "16" }, schema);
// { isValid: false, errors: { name: "Minimum 2 characters", email: "Invalid email address", age: "Must be 18 or older" } }
```

---

**Q15. How would you design a JavaScript SDK/library for third-party developers?** — Hard

**Answer:**

Key design principles for a public SDK:

```js
// sdk.js — A well-designed JavaScript SDK

class SDK {
  static #version = "1.0.0";
  #config;
  #initialized = false;

  /**
   * @param {Object} config
   * @param {string} config.apiKey - Required API key
   * @param {string} [config.baseUrl] - Optional custom endpoint
   * @param {boolean} [config.debug] - Enable debug logging
   */
  constructor(config = {}) {
    // Validate required config at initialization:
    if (!config.apiKey) throw new TypeError("SDK: apiKey is required");
    this.#config = {
      apiKey: config.apiKey,
      baseUrl: config.baseUrl ?? "https://api.example.com",
      debug: config.debug ?? false,
      timeout: config.timeout ?? 10_000
    };
  }

  async init() {
    if (this.#initialized) return this; // idempotent
    // Verify API key, load remote config, etc:
    await this.#verifyApiKey();
    this.#initialized = true;
    return this; // fluent
  }

  // Public API — simple, versioned, promise-based:
  async track(eventName, properties = {}) {
    this.#requireInit();
    this.#validateEventName(eventName);
    return this.#request("POST", "/events", { eventName, properties });
  }

  async identify(userId, traits = {}) {
    this.#requireInit();
    if (typeof userId !== "string") throw new TypeError("userId must be a string");
    return this.#request("POST", "/identify", { userId, traits });
  }

  // Private internals — not part of public API:
  async #request(method, path, body) {
    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), this.#config.timeout);

    try {
      const res = await fetch(`${this.#config.baseUrl}${path}`, {
        method,
        headers: {
          "Authorization": `Bearer ${this.#config.apiKey}`,
          "Content-Type": "application/json",
          "X-SDK-Version": SDK.#version
        },
        body: JSON.stringify(body),
        signal: controller.signal
      });

      if (!res.ok) throw new SDKError(`Request failed: ${res.status}`, res.status);
      return await res.json();
    } catch (err) {
      if (err.name === "AbortError") throw new SDKError("Request timed out", 408);
      throw err;
    } finally {
      clearTimeout(timeout);
    }
  }

  #requireInit() {
    if (!this.#initialized) throw new SDKError("Call sdk.init() before using the SDK");
  }

  #validateEventName(name) {
    if (typeof name !== "string" || !name.trim()) {
      throw new TypeError("eventName must be a non-empty string");
    }
  }

  async #verifyApiKey() { /* ... */ }

  static get version() { return SDK.#version; }
}

class SDKError extends Error {
  constructor(message, statusCode) {
    super(message);
    this.name = "SDKError";
    this.statusCode = statusCode;
  }
}

// Export — support both ESM and CJS:
export { SDK, SDKError };
export default SDK;
```

**SDK design checklist:**
- Clear, validated initialization
- Meaningful error types (`SDKError`, not generic `Error`)
- Idempotent `init()` — safe to call multiple times
- Timeout on every request
- `X-SDK-Version` header for server-side debugging
- Private fields for internals — clean public surface
- Promise-based, async/await friendly
- Works in both ESM and CJS environments
- Semantic versioning for the package

---

*End of JavaScript Interview Encyclopedia.*
*For React-specific questions, refer to a dedicated React interview guide.*
