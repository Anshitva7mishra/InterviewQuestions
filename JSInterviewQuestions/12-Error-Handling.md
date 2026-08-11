# 12 — Error Handling & Error Types
### 10 Questions | Intermediate

---

**Q1. What are the built-in Error types in JavaScript and when is each thrown?** — Easy

**Answer:**
JavaScript has a hierarchy of built-in error types, all inheriting from `Error`.

```js
// Base Error:
new Error("Something went wrong");
// Properties: .message, .name, .stack, .cause (ES2022)

// TypeError — wrong type or invalid operation on a type:
null.property;                    // TypeError: Cannot read properties of null
undefined();                      // TypeError: undefined is not a function
Object.defineProperty(1, "a", {}); // TypeError: Object.defineProperty called on non-object

// ReferenceError — variable that doesn't exist in scope:
console.log(undeclaredVar);       // ReferenceError: undeclaredVar is not defined
// (Note: accessing undeclared var in non-strict is ReferenceError, not undefined)

// SyntaxError — invalid syntax (usually at parse time, not runtime):
eval("function(");                // SyntaxError: Unexpected token )
// Thrown by: eval(), new Function(), JSON.parse() with invalid JSON:
JSON.parse("{bad json}");         // SyntaxError: Unexpected token b

// RangeError — value outside allowed range:
new Array(-1);                    // RangeError: Invalid array length
(1).toFixed(200);                 // RangeError: toFixed() digits must be 0-100
function recurse() { recurse(); }
recurse();                        // RangeError: Maximum call stack size exceeded

// URIError — invalid URI:
decodeURIComponent("%");          // URIError: URI malformed

// EvalError — historically from eval(), rarely thrown in modern JS:
// Still exists for legacy reasons

// AggregateError (ES2021) — multiple errors wrapped together:
Promise.any([
  Promise.reject(new Error("A")),
  Promise.reject(new Error("B"))
]).catch(e => {
  e instanceof AggregateError; // true
  e.errors; // [Error("A"), Error("B")]
});
```

**Follow-up:** How do you create a custom error type?

```js
class ValidationError extends Error {
  constructor(message, field) {
    super(message);
    this.name = "ValidationError";
    this.field = field;
  }
}

class NetworkError extends Error {
  constructor(message, statusCode) {
    super(message);
    this.name = "NetworkError";
    this.statusCode = statusCode;
  }
}

try {
  throw new ValidationError("Required", "email");
} catch (e) {
  if (e instanceof ValidationError) {
    console.log(e.field); // "email"
  }
}
```

**GOTCHA:** `Error.prototype.name` defaults to `"Error"` on all custom subclasses unless you explicitly set `this.name = "MyError"` in the constructor. This affects the string representation of the error and `instanceof` checks don't use `.name` — they use the prototype chain — so setting `.name` is cosmetic/logging purposes only.

---

**Q2. What is the `error.cause` property and how does error chaining work?** — Medium

**Answer:**
`error.cause` (ES2022) allows you to attach the original error as the cause of a new, higher-level error. This preserves the full error chain while allowing you to wrap errors with more contextual messages.

```js
// ES2022 — passing cause option to Error constructor:
async function fetchUser(id) {
  try {
    const response = await fetch(`/api/users/${id}`);
    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    return response.json();
  } catch (originalError) {
    // Wrap with context, preserve original:
    throw new Error(`Failed to fetch user ${id}`, { cause: originalError });
  }
}

// Inspecting the chain:
try {
  await fetchUser(42);
} catch (e) {
  console.log(e.message);         // "Failed to fetch user 42"
  console.log(e.cause?.message);  // "HTTP 404" (the original error)
  console.log(e.cause?.cause);    // undefined or deeper cause
}

// Custom error classes with cause:
class DatabaseError extends Error {
  constructor(message, options) {
    super(message, options); // passes { cause } to Error
    this.name = "DatabaseError";
  }
}

// Full chain:
function queryUser(id) {
  try {
    // ... db operation
  } catch (dbErr) {
    throw new DatabaseError("User query failed", { cause: dbErr });
  }
}

// Logging the full chain:
function logErrorChain(err, depth = 0) {
  console.error(" ".repeat(depth * 2) + err.message);
  if (err.cause) logErrorChain(err.cause, depth + 1);
}
```

**Follow-up:** Why is error chaining better than just logging the original error?

Without chaining, you lose either the original error's stack trace (if you create a new error and discard the original) or the contextual information about what was happening (if you just rethrow the original). Error chaining gives you both: a new, contextually meaningful error at the top level, with the original preserved as `.cause` for debugging.

**GOTCHA:** Before ES2022, `{ cause }` was ignored by the `Error` constructor — it's not set automatically. Libraries had to use patterns like `error.cause = originalError` manually. If you need to support older environments, set `.cause` explicitly on the new error object rather than relying on the constructor option.

---

**Q3. What is the difference between `throw` and rethrowing? When should you rethrow?** — Medium

**Answer:**
Throwing creates a new error. Rethrowing catches an error then re-throws it — either the same error object or a wrapped version.

```js
// Throw — generate a new error:
function divide(a, b) {
  if (b === 0) throw new RangeError("Cannot divide by zero");
  return a / b;
}

// Rethrow — catch and re-throw after some processing:
function processData(data) {
  try {
    return JSON.parse(data);
  } catch (e) {
    // Log, then rethrow — let the caller handle it:
    console.error("JSON parse failed:", e.message);
    throw e; // rethrow the same error
  }
}

// Selective rethrow — handle only known errors:
function handleRequest(req) {
  try {
    return riskyOperation(req);
  } catch (e) {
    if (e instanceof NetworkError) {
      // Known: handle specifically
      return { error: "Network issue, please retry" };
    }
    // Unknown: rethrow — don't swallow errors you don't understand
    throw e;
  }
}

// ANTI-PATTERN — catching and swallowing:
try {
  doSomething();
} catch (e) {
  // Silent catch — terrible: bugs disappear silently
}

// ANTI-PATTERN — catching everything and returning null:
function safeParse(data) {
  try {
    return JSON.parse(data);
  } catch {
    return null; // Caller has no idea why it failed
  }
}
// Better: throw a meaningful error with context
```

**When to rethrow:**
1. When you want to log or monitor but not handle the error
2. When filtering: handle only specific error types, rethrow the rest
3. When wrapping: convert low-level errors into domain errors while preserving cause
4. In async code: always rethrow in Promise chains if you can't recover

**GOTCHA:** When you rethrow `throw e`, the original stack trace is preserved. When you `throw new Error(e.message)`, you lose the original stack and get a new one from the point of the re-throw. Use `throw e` (or `throw new WrappedError(msg, { cause: e })`) to preserve debugging information.

---

**Q4. How does error handling work in async/await vs Promise chains?** — Medium

**Answer:**
Both approaches can handle rejections, but they have different ergonomics and different failure modes.

```js
// Promise chain — errors handled with .catch():
fetch("/api/data")
  .then(res => res.json())
  .then(data => processData(data))
  .catch(err => console.error("Failed:", err));

// async/await — errors handled with try/catch:
async function loadData() {
  try {
    const res = await fetch("/api/data");
    const data = await res.json();
    return processData(data);
  } catch (err) {
    console.error("Failed:", err);
  }
}

// Handling specific async errors:
async function getUser(id) {
  try {
    const user = await db.findUser(id);
    if (!user) throw new NotFoundError(`User ${id} not found`);
    return user;
  } catch (e) {
    if (e instanceof NotFoundError) {
      return null; // Expected — user doesn't exist
    }
    throw e; // Unexpected — rethrow DB errors
  }
}

// Parallel operations with individual error handling:
async function fetchMultiple() {
  const results = await Promise.allSettled([
    fetch("/api/a"),
    fetch("/api/b"),
    fetch("/api/c")
  ]);

  return results.map(result => {
    if (result.status === "fulfilled") return result.value;
    console.warn("One request failed:", result.reason);
    return null;
  });
}

// The "optional catch binding" — ES2019:
try {
  JSON.parse(input);
} catch {    // No (e) — you don't need the error variable
  return defaultValue;
}

// PROBLEM — unhandled promise rejections:
async function badExample() {
  fetch("/api").then(r => r.json()); // No .catch() — unhandled rejection
  // Node.js: UnhandledPromiseRejectionWarning
}
```

**Follow-up:** What happens to an error thrown inside a `setTimeout` called within an async function?

```js
async function example() {
  try {
    setTimeout(() => {
      throw new Error("async error"); // NOT caught by try/catch!
    }, 1000);
  } catch (e) {
    console.log("This never runs");
  }
}
// The error propagates to the global uncaught exception handler, not this try/catch.
// Solution: use process.on("uncaughtException") or wrap the callback in try/catch itself.
```

**GOTCHA:** An `async` function only catches errors from `await`ed Promises or synchronous throws within its own body. Errors in non-awaited callbacks (setTimeout, event listeners, etc.) escape the try/catch entirely. This is a very common source of "disappearing errors" in Node.js applications.

---

**Q5. What is the `finally` block and what are its guarantees and edge cases?** — Medium

**Answer:**
`finally` runs unconditionally after `try` and any `catch` — whether an error occurred, was caught, or rethrown. It is meant for cleanup operations.

```js
function readFile(path) {
  let file = null;
  try {
    file = openFile(path);        // May throw
    return file.read();           // May throw, but finally still runs
  } catch (e) {
    console.error("Read failed:", e.message);
    throw e;                       // Rethrowing — finally STILL runs
  } finally {
    if (file) file.close();       // Always cleaned up
    console.log("finally ran");
  }
}

// finally with return — the GOTCHA:
function gotcha() {
  try {
    return 1;
  } finally {
    return 2; // Overrides the return from try!
  }
}
gotcha(); // 2 — finally's return wins

// finally swallowing throws:
function silentFinally() {
  try {
    throw new Error("original");
  } finally {
    return "recovered"; // Swallows the original throw!
  }
}
silentFinally(); // Returns "recovered" — no error propagates!

// finally canceling a throw:
function cancelThrow() {
  try {
    throw new Error("error");
  } finally {
    // If finally does NOT throw or return, the original throw propagates
    console.log("cleanup"); // This runs, then original error propagates
  }
}

// Async finally:
async function fetchWithCleanup() {
  try {
    return await fetch("/api");
  } finally {
    await cleanup(); // finally can be async too — await is allowed here
  }
}
```

**Follow-up:** What is the difference between `try/catch/finally` and `using` (TC39 Stage 3)?

The `using` keyword (ES2026 proposal, Stage 3) provides automatic resource cleanup without `finally`:
```js
// Using declaration (if supported):
{
  using file = openFile(path);
  file.read(); // If this throws, file[Symbol.dispose]() is called automatically
} // file[Symbol.dispose]() called here

// Equivalent to finally pattern but cleaner syntax
```

**GOTCHA:** A `return` or `throw` inside `finally` silently overrides the original `return` or `throw` from `try` or `catch`. This is almost always a bug. Never use `return`, `throw`, `break`, or `continue` inside a `finally` block unless you have a very specific reason and are aware of the override behavior.

---

**Q6. What is `Promise.allSettled()` and when should you use it over `Promise.all()`?** — Medium

**Answer:**
`Promise.all()` rejects immediately when ANY promise rejects, causing all other results to be lost. `Promise.allSettled()` (ES2020) waits for ALL promises to complete regardless of outcome.

```js
const p1 = Promise.resolve("data A");
const p2 = Promise.reject(new Error("failed"));
const p3 = Promise.resolve("data C");

// Promise.all — fails fast:
try {
  const [a, b, c] = await Promise.all([p1, p2, p3]);
} catch (e) {
  // e = Error("failed")
  // We never know what p1 and p3 returned!
}

// Promise.allSettled — collects ALL results:
const results = await Promise.allSettled([p1, p2, p3]);
// [
//   { status: "fulfilled", value: "data A" },
//   { status: "rejected",  reason: Error("failed") },
//   { status: "fulfilled", value: "data C" }
// ]

results.forEach(result => {
  if (result.status === "fulfilled") {
    console.log("Success:", result.value);
  } else {
    console.error("Failed:", result.reason.message);
  }
});

// Use case — dashboard loading multiple independent widgets:
const [userResult, statsResult, notificationsResult] = await Promise.allSettled([
  fetchUser(),
  fetchStats(),
  fetchNotifications()
]);

// Each widget can independently succeed or fail:
const user = userResult.status === "fulfilled" ? userResult.value : defaultUser;
const stats = statsResult.status === "fulfilled" ? statsResult.value : [];
// Show partial dashboard — don't fail everything if notifications fail
```

Use `Promise.all()` when: all results are required and failure of any means the whole operation should fail (e.g., database transaction steps).

Use `Promise.allSettled()` when: you want to attempt multiple independent operations and process whatever succeeded (e.g., loading multiple independent resources).

**GOTCHA:** `Promise.all([])` with an empty array resolves immediately to `[]`. `Promise.allSettled([])` also resolves to `[]`. The difference matters when you build the array dynamically — an empty array is a valid input, not an error.

---

**Q7. How do you handle errors in event emitters and streams in Node.js?** — Hard

**Answer:**
Event emitters and streams use the `"error"` event convention rather than try/catch. Unhandled `"error"` events crash the Node.js process.

```js
const { EventEmitter } = require("events");
const { createReadStream } = require("fs");

// EventEmitter — MUST attach "error" listener:
const emitter = new EventEmitter();

// Without error handler — FATAL:
emitter.emit("error", new Error("boom")); // Process crashes!

// WITH error handler — handled safely:
emitter.on("error", (err) => {
  console.error("Emitter error:", err.message);
});
emitter.emit("error", new Error("boom")); // Handled gracefully

// Streams — must handle both "error" and close properly:
const stream = createReadStream("/nonexistent/file.txt");

stream.on("data", chunk => console.log(chunk));
stream.on("error", err => console.error("Stream error:", err.code)); // ENOENT
stream.on("close", () => console.log("Stream closed"));

// Using pipeline (preferred over manual pipe):
const { pipeline } = require("stream/promises");

async function processFile() {
  try {
    await pipeline(
      createReadStream("input.txt"),
      transform,
      createWriteStream("output.txt")
    );
  } catch (err) {
    console.error("Pipeline failed:", err.message);
    // pipeline automatically destroys all streams on error
  }
}

// EventEmitter.once("error") — for one-time error setup:
emitter.once("error", err => { /* handle */ });

// Catching unhandled rejections and exceptions globally (last resort):
process.on("uncaughtException", (err) => {
  console.error("Uncaught:", err);
  process.exit(1); // Always exit — state is unknown
});

process.on("unhandledRejection", (reason, promise) => {
  console.error("Unhandled rejection:", reason);
  // Node 15+: process crashes by default without this handler
});
```

**Follow-up:** What is the difference between `uncaughtException` and `unhandledRejection`?

`uncaughtException` fires for synchronous errors that escape all try/catch handlers (thrown in callbacks, global scope). `unhandledRejection` fires for Promise rejections with no `.catch()` or catch block. Node.js 15+ treats unhandled rejections as fatal crashes by default. Both should be used only for logging/alerting before a controlled shutdown — not for recovery, since application state is unknown.

**GOTCHA:** Throwing inside an `"error"` event handler crashes the process immediately, bypassing `uncaughtException`. If your error handler itself can throw, wrap it in its own try/catch.

---

**Q8. What is the `Error.stack` property and how does it vary across environments?** — Hard

**Answer:**
`Error.stack` is a non-standard (but universally implemented) property that contains the error message and a human-readable stack trace.

```js
function outer() {
  function inner() {
    throw new Error("Something failed");
  }
  inner();
}

try {
  outer();
} catch (e) {
  console.log(e.stack);
}

// V8 (Node.js / Chrome) format:
// Error: Something failed
//     at inner (file.js:3:11)
//     at outer (file.js:5:3)
//     at Object.<anonymous> (file.js:9:3)
//     at Module._compile (node:internal/modules/cjs/loader:1105:14)
//     ...

// SpiderMonkey (Firefox) format — slightly different:
// Something failed
// inner@file.js:3:11
// outer@file.js:5:3
// @file.js:9:3

// JavaScriptCore (Safari) format:
// inner@file.js:3:11
// outer@file.js:5:3
// global code@file.js:9:3

// V8 API — customize stack traces:
Error.prepareStackTrace = (err, callSites) => {
  return callSites.map(cs => ({
    fn: cs.getFunctionName(),
    file: cs.getFileName(),
    line: cs.getLineNumber(),
    col: cs.getColumnNumber()
  }));
};

const err = new Error("test");
console.log(err.stack); // Now an array of objects instead of string!

// Async stack traces — V8 async stack:
async function fetchData() {
  await Promise.resolve();
  throw new Error("async fail");
}

async function main() {
  await fetchData(); // V8 shows async stack frames in modern Node
}

// Capture stack at a specific point:
const frames = {};
Error.captureStackTrace(frames, someFunction); // excludes someFunction from trace
```

**GOTCHA:** `error.stack` is a string in most environments, but after setting `Error.prepareStackTrace`, it becomes whatever your function returns (an array in the example above). If you call `new Error()` in a hot path, stack trace generation has a significant performance cost in V8. In production, consider using `Error.captureStackTrace` with a `stackTraceLimit` of 0 for errors where you don't need traces.

---

**Q9. What is defensive programming and what error boundaries should you establish?** — Medium

**Answer:**
Defensive programming anticipates failure at boundaries between systems and user input, guarding against invalid data propagating into your application's core logic.

```js
// Input validation at API boundaries:
function createUser({ name, email, age }) {
  // Guard clauses at the top:
  if (!name || typeof name !== "string") {
    throw new TypeError("name must be a non-empty string");
  }
  if (!email || !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
    throw new RangeError("email must be a valid email address");
  }
  if (!Number.isInteger(age) || age < 0 || age > 150) {
    throw new RangeError("age must be an integer between 0 and 150");
  }
  // Happy path — all inputs validated:
  return db.insert({ name, email, age });
}

// Defensive parsing:
function parseConfig(raw) {
  let config;
  try {
    config = JSON.parse(raw);
  } catch {
    throw new Error("Configuration file contains invalid JSON", { cause: e });
  }
  // Validate shape after parsing:
  if (!config.apiUrl || typeof config.apiUrl !== "string") {
    throw new Error("config.apiUrl is required and must be a string");
  }
  return config;
}

// Optional chaining + nullish coalescing for safe access:
const city = user?.address?.city ?? "Unknown";

// Nullish default values:
function render(options = {}) {
  const {
    title = "Untitled",
    maxItems = 10,
    showFooter = true
  } = options;
  // Never throws if options is missing properties
}

// Error boundaries in React (class component or library):
class ErrorBoundary extends React.Component {
  state = { hasError: false, error: null };

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, info) {
    logErrorToService(error, info.componentStack);
  }

  render() {
    if (this.state.hasError) {
      return <FallbackUI error={this.state.error} />;
    }
    return this.props.children;
  }
}
```

**GOTCHA:** Don't over-validate — validating too aggressively can turn valid inputs into errors (e.g., rejecting email addresses that are actually valid per RFC but not your regex). Balance strictness with practicality, and defer deep validation (e.g., whether an email actually works) to the system boundary where it matters (e.g., the email delivery service).

---

**Q10. How do you properly test error handling in JavaScript?** — Medium

**Answer:**
Testing error paths is as important as testing happy paths. Different testing patterns are needed for synchronous, asynchronous, and class-based errors.

```js
// Jest examples:

// Synchronous throw:
test("throws on invalid input", () => {
  expect(() => divide(1, 0)).toThrow(RangeError);
  expect(() => divide(1, 0)).toThrow("Cannot divide by zero");
  expect(() => divide(1, 0)).toThrow(/divide/); // regex match on message
});

// Async throws (async/await):
test("rejects on network error", async () => {
  await expect(fetchUser(-1)).rejects.toThrow("User not found");
  await expect(fetchUser(-1)).rejects.toBeInstanceOf(NotFoundError);
});

// Promise rejection:
test("rejects with specific error", () => {
  return expect(fetchUser(-1)).rejects.toMatchObject({
    message: "User not found",
    statusCode: 404
  });
});

// Error properties:
test("error has correct properties", () => {
  expect(() => createUser({ name: "", email: "x", age: 0 }))
    .toThrow(expect.objectContaining({
      name: "ValidationError",
      field: "name"
    }));
});

// Mocking to force errors:
jest.spyOn(global, "fetch").mockRejectedValue(new NetworkError("timeout", 408));

test("handles network timeout gracefully", async () => {
  const result = await safeFetch("/api");
  expect(result).toBeNull();
  expect(console.warn).toHaveBeenCalledWith(expect.stringContaining("timeout"));
});

// Testing finally/cleanup:
test("cleanup runs even on error", async () => {
  const cleanup = jest.fn();
  await expect(fetchWithCleanup(cleanup)).rejects.toThrow();
  expect(cleanup).toHaveBeenCalledTimes(1); // finally ran
});
```

**Follow-up:** What are the most common mistakes when writing error handling tests?

1. Only testing that an error is thrown, not its type or message
2. Forgetting to `await` the assertion for async throws
3. Not testing that cleanup code (finally blocks, resource disposal) actually ran
4. Testing the wrong level — testing internal errors instead of the public API's behavior
5. Not using `toThrow()` correctly — calling it like `expect(fn()).toThrow()` instead of `expect(() => fn()).toThrow()` (note the wrapping arrow function)

**GOTCHA:** A common Jest mistake: `expect(someFn()).toThrow()` — this calls `someFn()` immediately, so if it throws, the test itself throws before Jest can catch it. Always wrap the function call in an arrow function: `expect(() => someFn()).toThrow()`. For async functions: `await expect(asyncFn()).rejects.toThrow()` — here you do NOT wrap in an arrow function because you want the Promise, not a function that returns one.

---

*Next: [13-Modern-Features-2020-2025.md](./13-Modern-Features-2020-2025.md)*
