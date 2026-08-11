# 17 — DOM & Browser APIs
### 15 Questions | All Levels

---

**Q1. What is the DOM and how does JavaScript interact with it?** — Easy

**Answer:**
The DOM (Document Object Model) is a programming interface for HTML and XML documents. It represents the document as a tree of objects — nodes — that JavaScript can read and manipulate.

```js
// DOM tree structure:
// document → <html> → <head>, <body> → child elements → text nodes

// Selecting elements:
document.getElementById("header");              // by ID (fastest)
document.querySelector(".card");                // first match (CSS selector)
document.querySelectorAll(".card");             // all matches (NodeList)
document.getElementsByClassName("btn");         // live HTMLCollection
document.getElementsByTagName("div");           // live HTMLCollection

// Creating and inserting elements:
const div = document.createElement("div");
div.className = "card";
div.textContent = "Hello World";

const parent = document.getElementById("container");
parent.appendChild(div);                         // add at end
parent.prepend(div);                            // add at start
parent.insertBefore(div, parent.firstChild);    // insert before
parent.replaceChild(div, parent.firstChild);    // replace
parent.removeChild(parent.lastChild);           // remove

// Modern insertion methods:
parent.append(div, "text node");                // insert nodes or strings
parent.before(div);                             // insert before parent
parent.after(div);                              // insert after parent
parent.remove();                                // remove from DOM (no parent needed)

// Reading and setting attributes:
div.getAttribute("class");                      // "card"
div.setAttribute("data-id", "123");
div.removeAttribute("hidden");
div.hasAttribute("disabled");                   // false

// Properties vs attributes:
// Properties: JS objects on the DOM element (live, fast)
// Attributes: HTML attributes in the markup (serialized)
div.className;          // property — reflects class attribute
div.id;                 // property
div.value;              // property — current value (for inputs)
div.getAttribute("value"); // attribute — original HTML value

// Traversal:
element.parentElement;          // parent element node
element.children;               // child ELEMENTS (no text nodes)
element.childNodes;             // all child nodes (including text)
element.firstElementChild;      // first child element
element.nextElementSibling;     // next sibling element
element.previousElementSibling; // previous sibling element
```

**GOTCHA:** `querySelectorAll()` returns a static `NodeList` (snapshot at query time). `getElementsByClassName()` and `getElementsByTagName()` return LIVE `HTMLCollection` objects — they update automatically as the DOM changes. This can cause infinite loops: `for (let el of document.getElementsByClassName("item")) el.className = "item done";` — the collection updates as you remove "item" class, potentially skipping elements.

---

**Q2. What is event bubbling and capturing?** — Medium

**Answer:**
DOM events propagate in three phases: capture (top-down), target, and bubble (bottom-up). By default, handlers run in the bubble phase.

```js
// Event flow:
// document → html → body → div → button  (capture phase, top-down)
// button                                  (target phase)
// button → div → body → html → document  (bubble phase, bottom-up)

// Default: bubble phase (useCapture = false):
button.addEventListener("click", handler);            // bubble (default)
button.addEventListener("click", handler, false);     // bubble (explicit)
button.addEventListener("click", handler, true);      // capture
button.addEventListener("click", handler, { capture: true }); // capture

// Stopping propagation:
button.addEventListener("click", (e) => {
  e.stopPropagation();     // stop bubble — parent handlers won't fire
  e.stopImmediatePropagation(); // stop all — even other handlers on THIS element
});

// Event delegation — listen on parent, handle for children:
document.getElementById("list").addEventListener("click", (e) => {
  const item = e.target.closest("li");  // find clicked li (or ancestor li)
  if (item) {
    item.classList.toggle("selected");
  }
});
// Works for li elements added DYNAMICALLY — no need to re-attach handlers

// e.target vs e.currentTarget:
// e.target — the element that was clicked (deepest in the tree)
// e.currentTarget — the element the handler is attached to

document.body.addEventListener("click", (e) => {
  console.log(e.target);        // the actual clicked element (e.g., button)
  console.log(e.currentTarget); // document.body — where handler lives
});
```

**GOTCHA:** `e.stopPropagation()` stops propagation to PARENT elements only. Other handlers on the SAME element still run. Use `e.stopImmediatePropagation()` to prevent all remaining handlers (including siblings on the same element). Also, some events do NOT bubble — `focus`, `blur`, `load`, `scroll` (with exceptions). Use the capturing phase or `focusin`/`focusout` (which do bubble) as alternatives.

---

**Q3. What is `event.preventDefault()` and when should you use it?** — Easy

**Answer:**
`event.preventDefault()` cancels the default browser action for an event — but does NOT stop propagation.

```js
// Form submit — prevent page reload:
form.addEventListener("submit", async (e) => {
  e.preventDefault();           // stop form from submitting to server
  const data = new FormData(e.target);
  await fetch("/api/submit", { method: "POST", body: data });
});

// Link — prevent navigation:
link.addEventListener("click", (e) => {
  e.preventDefault();
  // custom navigation logic (SPA routing)
  router.navigate(link.href);
});

// Context menu — prevent right-click menu:
document.addEventListener("contextmenu", (e) => {
  e.preventDefault();
  showCustomMenu(e.clientX, e.clientY);
});

// Input — prevent specific characters:
input.addEventListener("keydown", (e) => {
  if (e.key === "e" && input.type === "number") {
    e.preventDefault(); // prevent 'e' in number inputs (scientific notation)
  }
});

// Drag and drop — must preventDefault on dragover to allow drop:
dropZone.addEventListener("dragover", (e) => {
  e.preventDefault(); // REQUIRED — otherwise drop won't work
  e.dataTransfer.dropEffect = "copy";
});

// Check if event is cancelable:
e.cancelable; // true if preventDefault() has any effect
e.defaultPrevented; // true if preventDefault() was already called

// Passive event listeners — for performance (scroll events):
document.addEventListener("scroll", handler, { passive: true });
// { passive: true } tells browser to skip waiting for JS — better scroll perf
// With passive: true, calling e.preventDefault() throws or is ignored
```

**GOTCHA:** `event.preventDefault()` and `event.stopPropagation()` are completely independent — `preventDefault` affects the browser's default action, `stopPropagation` affects the event's travel through the DOM. You can call both, either, or neither independently.

---

**Q4. What is the difference between `innerHTML`, `textContent`, and `innerText`?** — Medium

**Answer:**
These three properties all deal with element content but have crucial differences in what they read/write and how.

```js
const div = document.getElementById("content");

// innerHTML — read/write HTML markup:
div.innerHTML;      // "<strong>Hello</strong> <em>World</em>"
div.innerHTML = "<strong>Bold</strong>"; // parses and inserts HTML

// textContent — read/write PLAIN TEXT (no HTML parsing):
div.textContent;    // "Hello World" — strips HTML tags
div.textContent = "<script>alert('xss')</script>"; // safe — treated as text
// Renders the literal string "<script>..." as text, NOT as HTML

// innerText — read/write RENDERED TEXT (CSS-aware):
div.innerText;      // "Hello World" — respects CSS (hidden elements excluded)
// div { display: none } → div.innerText returns ""
// div.textContent still returns the text

// Performance:
// Reading textContent — fast (just returns text nodes)
// Reading innerText — slow (causes layout reflow to check CSS)
// Writing innerHTML — parses HTML (moderate)
// Writing textContent — fastest (no HTML parsing)

// XSS risk:
const userInput = '<img src=x onerror="alert(1)">';

// DANGEROUS:
div.innerHTML = userInput; // executes the attack!

// SAFE:
div.textContent = userInput; // displays as literal text

// Inserting adjacent HTML safely:
div.insertAdjacentHTML("beforeend", "<p>New paragraph</p>"); // fine for trusted HTML
div.insertAdjacentText("beforeend", userInput); // safe for untrusted text

// Creating elements programmatically — safest for user content:
const p = document.createElement("p");
p.textContent = userInput; // always safe
div.appendChild(p);
```

**GOTCHA:** `innerHTML` fires `<script>` tags ONLY in some contexts (not via innerHTML in HTML5 — scripts set via innerHTML don't execute, but event handlers in attributes DO execute). However, `img onerror`, `svg onload`, and similar inline event handlers still execute. NEVER use `innerHTML` with untrusted user content. Use DOMPurify to sanitize before inserting.

---

**Q5. What is the difference between `window.onload`, `DOMContentLoaded`, and `defer`/`async` on script tags?** — Medium

**Answer:**
These control when JavaScript runs relative to page load.

```js
// DOMContentLoaded — fires when HTML is parsed and DOM is built
// (images and stylesheets may still be loading):
document.addEventListener("DOMContentLoaded", () => {
  // Safe to access all DOM elements
  const header = document.getElementById("header");
});

// window.onload (or 'load' event) — fires when EVERYTHING is loaded
// (including images, stylesheets, fonts, subframes):
window.addEventListener("load", () => {
  // Images are definitely loaded here
  const img = document.getElementById("hero");
  console.log(img.naturalWidth); // actual image width available
});

// window.beforeunload — fires when user tries to leave:
window.addEventListener("beforeunload", (e) => {
  if (hasUnsavedChanges) {
    e.preventDefault();     // show "Are you sure?" dialog
    e.returnValue = "";     // Chrome requires this
  }
});
```

Script loading strategies:
```html
<!-- Regular script — parser blocks, download + execute immediately: -->
<script src="app.js"></script>

<!-- async — download in parallel, execute immediately when ready (non-deterministic order): -->
<script async src="analytics.js"></script>

<!-- defer — download in parallel, execute AFTER DOMContentLoaded (in order): -->
<script defer src="app.js"></script>

<!-- type="module" — always deferred, has module scope: -->
<script type="module" src="app.mjs"></script>
```

```
Timeline comparison:
HTML parsing: ──────────────────────────────────────
                              ↓DOMContentLoaded

Regular:      ──[BLOCK]────[EXEC]──────────────────
async:        ──────────────[DL]─[EXEC]─────────────  (runs whenever DL finishes)
defer:        ──────────────[DL]────────────[EXEC]──  (runs after parsing)
module:       same as defer
```

**GOTCHA:** `async` scripts execute in download-completion order — not document order. If script B finishes downloading before A, B runs first. Use `defer` when order matters. Also, `DOMContentLoaded` can fire before the document is visually styled if CSS is still loading — accessing `getComputedStyle()` before CSS loads may return incorrect values.

---

**Q6. What is the Intersection Observer API and how do you use it for lazy loading?** — Medium

**Answer:**
`IntersectionObserver` efficiently detects when elements enter or exit the viewport (or another container), without requiring expensive scroll event listeners.

```js
// Basic usage:
const observer = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      console.log(entry.target, "is visible");
    }
  });
});

observer.observe(document.getElementById("section"));
observer.unobserve(element);  // stop observing one element
observer.disconnect();        // stop observing all elements

// Lazy loading images:
const imageObserver = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      const img = entry.target;
      img.src = img.dataset.src;        // load the actual image
      img.classList.add("loaded");
      imageObserver.unobserve(img);     // stop observing once loaded
    }
  });
}, {
  rootMargin: "200px"  // start loading 200px before the image enters the viewport
});

// Observe all lazy images:
document.querySelectorAll("img[data-src]").forEach(img => {
  imageObserver.observe(img);
});

// Options:
const options = {
  root: null,           // viewport (default)
  rootMargin: "0px",    // expand/shrink root bounds
  threshold: 0.5        // fire when 50% of element is visible
  // threshold: [0, 0.25, 0.5, 0.75, 1.0] — multiple thresholds
};

// Infinite scroll:
const sentinelObserver = new IntersectionObserver((entries) => {
  if (entries[0].isIntersecting) {
    loadMoreItems(); // load next page
  }
}, { threshold: 1.0 }); // fire when sentinel is fully visible

sentinelObserver.observe(document.getElementById("sentinel")); // element at bottom of list
```

**Follow-up:** Why is IntersectionObserver better than a scroll event listener?

Scroll listeners fire synchronously on every scroll tick — potentially 60+ times per second — requiring `requestAnimationFrame` throttling and manual viewport calculations. `IntersectionObserver` runs asynchronously off the main thread, callbacks are batched, and it handles transforms and scroll containers. It's vastly more performant.

**GOTCHA:** `IntersectionObserver` callbacks run ASYNCHRONOUSLY, not on every scroll frame. There may be a slight delay. Also, `entry.isIntersecting === false` doesn't mean it's never been in view — on first observation, callbacks fire regardless of intersection state, so always check `isIntersecting`.

---

**Q7. What is the `MutationObserver` API?** — Hard

**Answer:**
`MutationObserver` watches for changes to the DOM tree — element additions, removals, attribute changes, text content changes.

```js
const observer = new MutationObserver((mutations) => {
  mutations.forEach(mutation => {
    if (mutation.type === "childList") {
      mutation.addedNodes.forEach(node => {
        console.log("Node added:", node);
      });
      mutation.removedNodes.forEach(node => {
        console.log("Node removed:", node);
      });
    }
    if (mutation.type === "attributes") {
      console.log(`Attribute "${mutation.attributeName}" changed`);
      console.log("Old value:", mutation.oldValue);
      console.log("New value:", mutation.target.getAttribute(mutation.attributeName));
    }
    if (mutation.type === "characterData") {
      console.log("Text changed to:", mutation.target.textContent);
    }
  });
});

// Observe an element:
observer.observe(document.getElementById("container"), {
  childList: true,           // watch for added/removed children
  subtree: true,             // watch entire subtree (not just direct children)
  attributes: true,          // watch attribute changes
  attributeFilter: ["class", "style"], // only watch specific attributes
  attributeOldValue: true,   // record old attribute values
  characterData: true,       // watch text content changes
  characterDataOldValue: true // record old text values
});

observer.disconnect(); // stop observing

// Use case — "polyfill" custom element behavior:
const observer2 = new MutationObserver(mutations => {
  for (const mutation of mutations) {
    for (const node of mutation.addedNodes) {
      if (node.matches?.("[data-tooltip]")) {
        attachTooltip(node);
      }
    }
  }
});
observer2.observe(document.body, { childList: true, subtree: true });

// Use case — detect third-party DOM changes (ad injections, etc.):
const securityObserver = new MutationObserver(mutations => {
  mutations.forEach(m => {
    m.addedNodes.forEach(node => {
      if (node.tagName === "SCRIPT") {
        console.warn("External script injected:", node.src);
      }
    });
  });
});
securityObserver.observe(document.documentElement, { childList: true, subtree: true });
```

**GOTCHA:** `MutationObserver` callbacks run as microtasks — they can fire very frequently for rapid DOM changes. If your callback itself modifies the DOM, it can trigger more mutations in an infinite loop. Always add guards (e.g., a flag or check the mutation's properties) before modifying the DOM inside a mutation callback.

---

**Q8. How does `requestAnimationFrame` work and when should you use it?** — Medium

**Answer:**
`requestAnimationFrame` (rAF) synchronizes JavaScript code with the browser's rendering pipeline, running the callback just before the next repaint.

```js
// Basic animation loop:
function animate(timestamp) {
  // timestamp is DOMHighResTimeStamp — ms since page load
  ctx.clearRect(0, 0, canvas.width, canvas.height);
  drawFrame(timestamp);
  requestAnimationFrame(animate); // schedule next frame
}
requestAnimationFrame(animate); // start the loop

// Cancelling:
const id = requestAnimationFrame(callback);
cancelAnimationFrame(id);

// Smooth animation — timing-based (NOT frame-count based):
let startTime = null;
const duration = 1000; // 1 second

function animateBox(timestamp) {
  if (!startTime) startTime = timestamp;
  const elapsed = timestamp - startTime;
  const progress = Math.min(elapsed / duration, 1);

  // Ease in-out:
  const eased = progress < 0.5
    ? 2 * progress * progress
    : 1 - Math.pow(-2 * progress + 2, 2) / 2;

  box.style.transform = `translateX(${eased * 300}px)`;

  if (progress < 1) requestAnimationFrame(animateBox);
}
requestAnimationFrame(animateBox);

// Use rAF for batching DOM reads/writes:
// DOM reads first, then writes — avoids layout thrashing:
function updateLayout() {
  // Read phase — collect all measurements:
  const heights = elements.map(el => el.offsetHeight);

  // Write phase — apply changes:
  requestAnimationFrame(() => {
    elements.forEach((el, i) => {
      el.style.height = heights[i] + 10 + "px";
    });
  });
}

// Why rAF over setTimeout for animations:
// setTimeout(fn, 16) — imprecise, may skip or delay frames
// rAF — synced with display refresh (60Hz → ~16.67ms, 120Hz → ~8.33ms)
// rAF pauses when tab is hidden → saves CPU/battery
```

**GOTCHA:** `requestAnimationFrame` callbacks are NOT called when the tab is hidden or the page is off-screen. If you need continuous operation regardless of visibility (e.g., for networking), use `setInterval` instead. Also, rAF fires once per frame — to keep animating, you MUST call `requestAnimationFrame` again inside the callback.

---

**Q9. What is the Web Storage API (`localStorage` and `sessionStorage`)?** — Easy

**Answer:**
Web Storage provides a key-value store for browsers. Both use the same API, differing only in scope and persistence.

```js
// localStorage — persists until explicitly cleared (survives page reload, browser restart):
localStorage.setItem("token", "abc123");
localStorage.getItem("token");           // "abc123"
localStorage.removeItem("token");
localStorage.clear();                     // remove ALL items

// sessionStorage — persists for the TAB session (cleared when tab closes):
sessionStorage.setItem("draft", JSON.stringify(formData));
const draft = JSON.parse(sessionStorage.getItem("draft") ?? "null");

// Both only store strings:
localStorage.setItem("count", 42);       // stored as "42" (stringified)
typeof localStorage.getItem("count");    // "string"!

// Always parse/stringify objects:
const user = { id: 1, name: "Alice" };
localStorage.setItem("user", JSON.stringify(user));
const stored = JSON.parse(localStorage.getItem("user")); // back to object

// Storage quota:
// Typically 5-10 MB per origin. Exceeds quota → throws QuotaExceededError

// Storage event — fires when ANOTHER tab changes storage:
window.addEventListener("storage", (e) => {
  console.log("Key changed:", e.key);
  console.log("Old value:", e.oldValue);
  console.log("New value:", e.newValue);
  console.log("Origin:", e.url);
  // Does NOT fire in the tab that made the change!
});

// Limitations:
// - Synchronous API — blocks main thread for large operations
// - Same-origin only — no cross-domain access
// - Not accessible in Web Workers
// - Not secure for sensitive data (JS-accessible, XSS risk)

// Better for large or structured data: IndexedDB (async, larger quota)
// Better for sensitive data: httpOnly cookies (not JS-accessible)
```

**GOTCHA:** `localStorage.getItem()` returns `null` (not `undefined`) for missing keys. `JSON.parse(null)` returns `null`, so `JSON.parse(localStorage.getItem("missing"))` safely returns `null`. Never store sensitive data (tokens, passwords) in localStorage — any XSS vulnerability exposes everything stored there. Use `HttpOnly` cookies for auth tokens instead.

---

**Q10. What is the Fetch API and how does it compare to `XMLHttpRequest`?** — Medium

**Answer:**
`fetch()` is the modern Promise-based HTTP API replacing the older callback-based `XMLHttpRequest`.

```js
// Basic GET:
const res = await fetch("https://api.example.com/users");
if (!res.ok) throw new Error(`HTTP error: ${res.status}`);
const users = await res.json();

// POST with JSON:
const newUser = await fetch("/api/users", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ name: "Alice", email: "alice@example.com" })
}).then(res => res.json());

// With credentials (cookies, HTTP auth):
fetch("/api/profile", { credentials: "include" });

// Response methods:
res.json();       // parse body as JSON (returns Promise)
res.text();       // body as plain text
res.blob();       // body as Blob (binary data)
res.arrayBuffer(); // body as ArrayBuffer
res.formData();   // body as FormData
res.body;         // ReadableStream — for streaming large responses

// Streaming response (large file):
const reader = res.body.getReader();
while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  processChunk(value); // Uint8Array
}

// AbortController — cancellation:
const controller = new AbortController();
const response = fetch("/api/data", { signal: controller.signal });
setTimeout(() => controller.abort(), 5000); // cancel after 5s

// Upload progress — NOT natively supported by fetch (XHR has it):
// Use XMLHttpRequest or fetch with ReadableStream for upload progress

// XHR (legacy — avoid for new code):
const xhr = new XMLHttpRequest();
xhr.open("GET", "/api/data");
xhr.onload = () => console.log(JSON.parse(xhr.responseText));
xhr.onerror = () => console.error("XHR failed");
xhr.send();

// XHR advantages over fetch:
// - Upload progress events (xhr.upload.onprogress)
// - Synchronous mode (never use this!)
// - Better browser support (IE11)
```

**GOTCHA:** `fetch()` does NOT throw for HTTP error status codes (4xx, 5xx). It only rejects for network failures (no connection, CORS block, DNS failure). You MUST check `response.ok` (status 200-299) or `response.status` manually. This is the most common fetch mistake — assuming a rejected promise means an HTTP error.

---

**Q11. What is CORS and how does the browser enforce it?** — Hard

**Answer:**
CORS (Cross-Origin Resource Sharing) is a security mechanism that restricts cross-origin HTTP requests made from scripts. Browsers enforce it by checking response headers from the server.

```js
// Same-origin: same protocol + host + port
// https://app.com → https://app.com/api   ✅ same origin
// https://app.com → https://api.app.com   ❌ different subdomain
// https://app.com → http://app.com/api    ❌ different protocol

// Simple requests (no preflight): GET/HEAD/POST with simple headers
// Complex requests: PUT, DELETE, custom headers → triggers preflight

// Preflight OPTIONS request (browser sends automatically):
// OPTIONS /api/users HTTP/1.1
// Origin: https://app.com
// Access-Control-Request-Method: DELETE
// Access-Control-Request-Headers: Authorization

// Server must respond:
// Access-Control-Allow-Origin: https://app.com
// Access-Control-Allow-Methods: GET, POST, DELETE
// Access-Control-Allow-Headers: Authorization
// Access-Control-Max-Age: 86400  (cache preflight for 1 day)

// fetch with custom headers — triggers preflight:
await fetch("https://api.external.com/data", {
  headers: { "Authorization": "Bearer token" } // triggers CORS preflight
});

// Server CORS headers (Node.js/Express):
app.use((req, res, next) => {
  res.setHeader("Access-Control-Allow-Origin", "https://trusted-app.com");
  res.setHeader("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS");
  res.setHeader("Access-Control-Allow-Headers", "Content-Type, Authorization");
  res.setHeader("Access-Control-Allow-Credentials", "true");

  if (req.method === "OPTIONS") {
    res.sendStatus(204); // preflight response
    return;
  }
  next();
});

// Credentials (cookies) across origins:
// Client must set: credentials: "include"
// Server must set: Access-Control-Allow-Credentials: true
// Server CANNOT use: Access-Control-Allow-Origin: *  (must be specific origin)

fetch("https://api.example.com/profile", { credentials: "include" });
```

**GOTCHA:** `Access-Control-Allow-Origin: *` (wildcard) does NOT allow credentials (cookies, HTTP auth). If you need credentials AND CORS, the server must echo the specific `Origin` header value. Also, CORS is entirely a browser enforcement — server-to-server requests (Node.js `fetch`, curl) are not restricted by CORS. It only applies to browser-originated requests.

---

**Q12. What is the `History API` and how does it enable client-side routing?** — Medium

**Answer:**
The History API allows SPAs to change the URL without a page reload, maintain a navigable history, and handle browser back/forward navigation.

```js
// Navigate without reload:
history.pushState(state, title, url);
history.replaceState(state, title, url); // replace current entry (no new history item)

// Examples:
history.pushState({ page: "about" }, "", "/about");
// URL changes to /about, no reload, adds to history

history.replaceState({ page: "home" }, "", "/");
// URL changes to /, no reload, REPLACES current history entry (back button skips it)

// Handle back/forward:
window.addEventListener("popstate", (event) => {
  console.log("Location changed to:", location.pathname);
  console.log("State:", event.state); // the state object from pushState
  renderPage(location.pathname);
});

// Full SPA router skeleton:
class Router {
  #routes = new Map();

  add(path, handler) {
    this.#routes.set(path, handler);
    return this;
  }

  navigate(path, data = {}) {
    history.pushState(data, "", path);
    this.#render(path);
  }

  #render(path) {
    const handler = this.#routes.get(path) ?? this.#routes.get("*");
    if (handler) handler(path);
  }

  init() {
    window.addEventListener("popstate", () => this.#render(location.pathname));
    this.#render(location.pathname); // render current URL on page load
  }
}

const router = new Router()
  .add("/", () => renderHome())
  .add("/about", () => renderAbout())
  .add("*", () => render404());

router.init();

// Important: server must serve index.html for ALL routes when using HTML5 history API
// Otherwise, refreshing /about gives a 404 from the server

// Reading current state:
history.state;    // current state object
history.length;   // number of entries in history
location.pathname; // current path
location.search;   // query string (?q=...)
location.hash;     // hash (#section)
```

**GOTCHA:** When using the HTML5 History API in SPAs, the server MUST serve the same `index.html` for all routes. When a user navigates directly to `/about` or refreshes, the server receives a request for `/about` and must serve the app HTML — not a 404. This is the "catch-all" or "fallback" server configuration required for SPA routing.

---

**Q13. What is the `CustomEvent` API and how do you create custom DOM events?** — Medium

**Answer:**
`CustomEvent` allows you to create and dispatch your own events with custom data, enabling decoupled communication between components.

```js
// Create a custom event:
const event = new CustomEvent("user:login", {
  detail: { userId: 42, username: "alice" }, // custom data
  bubbles: true,      // event bubbles up the DOM tree
  cancelable: true,   // can be cancelled with preventDefault()
  composed: false     // doesn't cross Shadow DOM boundaries
});

// Dispatch on an element:
document.dispatchEvent(event);           // dispatch globally
element.dispatchEvent(event);           // dispatch on specific element

// Listen for the custom event:
document.addEventListener("user:login", (e) => {
  console.log("User logged in:", e.detail.username);
  console.log("User ID:", e.detail.userId);
});

// Real-world pattern — component communication:
class ShoppingCart {
  #items = [];

  addItem(item) {
    this.#items.push(item);
    // Notify other parts of the app:
    document.dispatchEvent(new CustomEvent("cart:itemAdded", {
      detail: { item, total: this.#items.length },
      bubbles: true
    }));
  }
}

// Header listens for cart updates:
document.addEventListener("cart:itemAdded", ({ detail }) => {
  document.getElementById("cart-count").textContent = detail.total;
});

// Cancelable custom events — caller can prevent action:
function requestClose(modal) {
  const event = new CustomEvent("modal:close", {
    detail: { modal },
    cancelable: true,
    bubbles: true
  });

  modal.dispatchEvent(event);

  if (!event.defaultPrevented) {
    modal.remove(); // only close if no handler prevented it
  }
}

// A form with unsaved changes prevents close:
modal.addEventListener("modal:close", (e) => {
  if (hasUnsavedChanges()) {
    e.preventDefault(); // block the close
    showSavePrompt();
  }
});
```

**GOTCHA:** `new Event()` cannot carry custom data — use `new CustomEvent()` with the `detail` property for that. Also, the `detail` property is read-only after creation — you can't modify it later. Always pass all needed data at creation time.

---

**Q14. What is `ResizeObserver` and when would you use it?** — Medium

**Answer:**
`ResizeObserver` watches for changes to an element's dimensions, running a callback whenever the element's content box (or border box) resizes.

```js
const observer = new ResizeObserver((entries) => {
  entries.forEach(entry => {
    const { width, height } = entry.contentRect;
    console.log(`Element resized: ${width}px × ${height}px`);

    // entry.contentBoxSize — array of content box sizes (for multi-column)
    // entry.borderBoxSize  — size including borders and padding
    // entry.devicePixelContentBoxSize — physical pixels (for canvas)
  });
});

observer.observe(element);
observer.observe(element, { box: "border-box" }); // observe border-box instead

observer.unobserve(element);
observer.disconnect();

// Use case 1 — responsive canvas:
const canvas = document.getElementById("chart");
const resizeObserver = new ResizeObserver(entries => {
  for (const entry of entries) {
    const { width, height } = entry.contentRect;
    canvas.width = width * devicePixelRatio;   // crisp on retina
    canvas.height = height * devicePixelRatio;
    redrawChart(); // re-render at new size
  }
});
resizeObserver.observe(canvas);

// Use case 2 — responsive components (container queries polyfill):
const cardObserver = new ResizeObserver(entries => {
  entries.forEach(({ target, contentRect }) => {
    target.classList.toggle("compact", contentRect.width < 400);
    target.classList.toggle("wide", contentRect.width >= 800);
  });
});
document.querySelectorAll(".card").forEach(card => cardObserver.observe(card));

// Use case 3 — virtual list item measurement:
const measureObserver = new ResizeObserver(entries => {
  entries.forEach(({ target, contentRect }) => {
    virtualList.setItemHeight(target.dataset.index, contentRect.height);
  });
});
```

**Follow-up:** What's the difference between ResizeObserver and checking `window.resize`?

`window.resize` only fires when the VIEWPORT changes — not when individual elements resize (due to CSS changes, DOM manipulation, or parent container resizing). `ResizeObserver` watches specific elements independently of the viewport. It's essential for component-level responsive design.

**GOTCHA:** `ResizeObserver` callbacks can trigger additional resizing (if you modify element styles inside the callback), causing an infinite loop. The browser detects this and logs a "ResizeObserver loop limit exceeded" warning — it won't crash, but the extra callbacks are skipped. Check if dimensions actually changed before applying updates.

---

**Q15. What is the `Performance API` and how do you measure rendering performance?** — Hard

**Answer:**
The `Performance API` provides precise, high-resolution timing and performance measurement capabilities.

```js
// High-resolution timestamps:
performance.now(); // ms since page load, sub-millisecond precision

// Measure a code block:
performance.mark("operation-start");
doExpensiveOperation();
performance.mark("operation-end");
performance.measure("operation", "operation-start", "operation-end");

const [measure] = performance.getEntriesByName("operation");
console.log(`Operation took: ${measure.duration}ms`);

// Navigation timing — page load breakdown:
const nav = performance.getEntriesByType("navigation")[0];
console.log("DNS lookup:", nav.domainLookupEnd - nav.domainLookupStart);
console.log("TCP connect:", nav.connectEnd - nav.connectStart);
console.log("TTFB:", nav.responseStart - nav.requestStart);  // Time To First Byte
console.log("DOM ready:", nav.domContentLoadedEventEnd - nav.startTime);
console.log("Page load:", nav.loadEventEnd - nav.startTime);

// Resource timing — individual resources:
performance.getEntriesByType("resource").forEach(res => {
  console.log(`${res.name}: ${res.duration.toFixed(2)}ms (${res.transferSize} bytes)`);
});

// Long Tasks API — detect tasks blocking main thread for >50ms:
const observer = new PerformanceObserver((list) => {
  list.getEntries().forEach(entry => {
    console.warn(`Long task detected: ${entry.duration}ms`);
    // entry.attribution — which script/element caused it
  });
});
observer.observe({ entryTypes: ["longtask"] });

// Largest Contentful Paint (LCP):
new PerformanceObserver(list => {
  const entries = list.getEntries();
  const lastEntry = entries[entries.length - 1];
  console.log("LCP:", lastEntry.startTime);
}).observe({ type: "largest-contentful-paint", buffered: true });

// Layout Shift (CLS):
let cumulativeLayoutShift = 0;
new PerformanceObserver(list => {
  list.getEntries().forEach(entry => {
    if (!entry.hadRecentInput) {
      cumulativeLayoutShift += entry.value;
    }
  });
}).observe({ type: "layout-shift", buffered: true });

// Core Web Vitals reporting:
// Use web-vitals library: import { getCLS, getFID, getLCP } from "web-vitals";
```

**GOTCHA:** `performance.now()` has reduced precision in some browsers (1ms resolution) due to Spectre/Meltdown mitigations — the origin must be cross-origin isolated (`COEP: require-corp` + `COOP: same-origin`) to get sub-millisecond precision. Also, `performance.mark()` and `performance.measure()` entries persist in the performance timeline until you clear them with `performance.clearMarks()` / `performance.clearMeasures()`.

---

*Next: [18-Coding-Problems.md](./18-Coding-Problems.md)*
