# 14 — Node.js Specifics
### 12 Questions | Intermediate

---

**Q1. What is the Node.js event loop and how does it differ from the browser's event loop?** — Hard

**Answer:**
Node.js uses libuv under the hood for its event loop, which has more phases than the browser's simpler loop.

```
Node.js Event Loop Phases (libuv):

┌─────────────────────────────────────────────┐
│                                             │
│  1. timers         — setTimeout, setInterval│
│  2. pending I/O    — I/O callbacks deferred │
│  3. idle/prepare   — internal use           │
│  4. poll           — retrieve new I/O events│
│  5. check          — setImmediate callbacks │
│  6. close callbacks— close events (socket)  │
│                                             │
│  Between each phase:                        │
│  → process.nextTick queue (runs FIRST)      │
│  → microtask queue (Promise callbacks)      │
└─────────────────────────────────────────────┘
```

```js
setTimeout(() => console.log("setTimeout"), 0);
setImmediate(() => console.log("setImmediate"));
Promise.resolve().then(() => console.log("Promise"));
process.nextTick(() => console.log("nextTick"));
console.log("sync");

// Output (in Node.js):
// sync          — synchronous code first
// nextTick      — nextTick queue drains before microtasks
// Promise       — microtask queue
// setTimeout    — timers phase
// setImmediate  — check phase (usually after setTimeout when both are 0ms)
```

Key Node.js-specific differences from browser:
```js
// 1. process.nextTick — does NOT exist in browsers:
process.nextTick(() => console.log("runs before any Promise"));

// 2. setImmediate — does NOT exist in standard browsers (IE only):
setImmediate(() => console.log("runs after I/O in this iteration"));

// 3. No rendering phase — browser has a "render" step between macro-tasks

// 4. setTimeout vs setImmediate order:
// In the main module: order is NON-DETERMINISTIC (depends on OS)
setTimeout(() => console.log("A"), 0);
setImmediate(() => console.log("B"));
// May print: A then B, OR B then A — depends on event loop phase timing

// Inside an I/O callback: setImmediate ALWAYS runs first:
fs.readFile("file.txt", () => {
  setTimeout(() => console.log("timeout"), 0);
  setImmediate(() => console.log("immediate"));
  // ALWAYS: "immediate" then "timeout"
});
```

**GOTCHA:** `process.nextTick()` runs BEFORE Promises even though both are "microtasks." Overloading `process.nextTick()` with recursive calls (calling `nextTick` inside a `nextTick` callback) can starve the event loop — I/O events will never process. Use `setImmediate` instead if you need to defer to the next iteration.

---

**Q2. What is the difference between `require()` and `import` in Node.js?** — Medium

**Answer:**
`require()` is synchronous CommonJS module loading. `import` is static ESM. Node.js supports both but they have important differences.

```js
// CommonJS (require) — synchronous:
const fs = require("fs");           // built-in
const path = require("path");
const myLib = require("./mylib");   // relative path

// ESM (import) — static:
import fs from "fs";
import { readFile } from "fs/promises";
import myLib from "./mylib.mjs";    // must include extension in ESM

// CJS interoperability:
// ESM can import CJS (but only the default export):
import pkg from "some-cjs-package"; // pkg = module.exports
// Named imports from CJS don't work reliably without build tools

// CJS CANNOT statically require() ESM files:
const esm = require("./esm-file.mjs"); // ERR_REQUIRE_ESM
// Must use dynamic import():
const esm = await import("./esm-file.mjs");

// Differences:
// require() — synchronous, can be conditional, can use variables:
const config = require(`./configs/${env}.json`);
if (DEBUG) require("./debugger");

// import — must be at top level, static (but dynamic import() is available):
const config = await import(`./configs/${env}.json`);

// __dirname and __filename — CJS only:
console.log(__dirname);  // CJS: works
// In ESM:
import { fileURLToPath } from "url";
import { dirname } from "path";
const __dirname = dirname(fileURLToPath(import.meta.url));

// Module caching:
require("./module"); // loads and caches
require("./module"); // returns CACHED version — not re-executed
// Clearing cache (advanced use):
delete require.cache[require.resolve("./module")];
require("./module"); // loads fresh
```

**GOTCHA:** When Node.js sees `"type": "module"` in package.json, `.js` files are treated as ESM and `require()` is not available. Using `require()` in such a file throws `ReferenceError: require is not defined`. If you need `require` in an ESM file, use `module.createRequire(import.meta.url)`.

---

**Q3. What are Node.js streams and why should you use them?** — Hard

**Answer:**
Streams are collections of data processed piece-by-piece (chunks), allowing you to handle large amounts of data without loading it all into memory.

```js
const { createReadStream, createWriteStream } = require("fs");
const { Transform } = require("stream");
const { pipeline } = require("stream/promises");

// Problem WITHOUT streams — loads entire file into memory:
const data = fs.readFileSync("huge-file.csv");         // 10GB → crashes
const processed = processData(data.toString());
fs.writeFileSync("output.csv", processed);

// WITH streams — processes in chunks of ~64KB:
const readStream = createReadStream("huge-file.csv");
const writeStream = createWriteStream("output.csv");

// Transform stream — transform data as it flows:
const uppercaseTransform = new Transform({
  transform(chunk, encoding, callback) {
    this.push(chunk.toString().toUpperCase());
    callback();
  }
});

// pipeline — connects streams with automatic error propagation and cleanup:
await pipeline(
  readStream,
  uppercaseTransform,
  writeStream
);
// On ANY stream error, all streams are destroyed — no leaks

// Stream types:
// Readable — source of data (fs.createReadStream, HTTP req, child_process stdout)
// Writable — destination (fs.createWriteStream, HTTP res, child_process stdin)
// Duplex  — both readable and writable (TCP socket, crypto.Cipher)
// Transform — reads, transforms, writes (zlib, crypto)

// Backpressure — critical concept:
const readable = createReadStream("input.txt");
const writable = createWriteStream("output.txt");

readable.on("data", (chunk) => {
  const canContinue = writable.write(chunk);
  if (!canContinue) {
    // writable buffer full — pause reading:
    readable.pause();
    writable.once("drain", () => readable.resume());
  }
});
readable.on("end", () => writable.end());
// Use pipeline() to handle backpressure automatically!

// Node.js 17+ — Web Streams API (WHATWG):
const { ReadableStream, WritableStream } = require("stream/web");
// Interop:
const nodeStream = Readable.fromWeb(webReadableStream);
const webStream = Readable.toWeb(nodeReadableStream);
```

**GOTCHA:** `pipe()` does NOT handle errors properly — if a destination stream errors, the source stream is NOT automatically destroyed, causing memory leaks. Always use `pipeline()` (from `stream/promises`) instead of `pipe()` in modern Node.js code. Also, forgetting to handle `"error"` events on streams crashes the process.

---

**Q4. What is the Node.js cluster module and how does it achieve multi-core utilization?** — Hard

**Answer:**
Node.js is single-threaded by design, but the `cluster` module allows you to spawn worker processes (one per CPU core) that all share the same server port.

```js
const cluster = require("cluster");
const http = require("http");
const os = require("os");
const numCPUs = os.cpus().length;

if (cluster.isPrimary) {
  // Master process — forks workers:
  console.log(`Primary ${process.pid} running, forking ${numCPUs} workers`);

  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  cluster.on("exit", (worker, code, signal) => {
    console.log(`Worker ${worker.process.pid} died (${signal || code}), restarting...`);
    cluster.fork(); // auto-restart dead workers
  });

  // IPC — master ↔ worker communication:
  cluster.on("message", (worker, message) => {
    console.log(`Worker ${worker.id} says:`, message);
  });

} else {
  // Worker process — each runs the HTTP server:
  http.createServer((req, res) => {
    res.writeHead(200);
    res.end(`Handled by worker ${process.pid}`);
  }).listen(3000);

  // Workers can send messages to master:
  process.send({ type: "log", data: `Worker ${process.pid} started` });
}

// How requests are distributed:
// Linux (default): Round-robin by the OS (fair distribution)
// Windows: The master distributes connections (less predictable)
// scheduler.policy setting can be used to control this

// Alternatives to cluster:
// 1. Worker Threads (same process, shared memory):
const { Worker } = require("worker_threads");
// Better for CPU-intensive tasks, not I/O servers

// 2. PM2 — process manager with cluster mode built-in:
// pm2 start app.js -i max  (auto-detects CPU count)
```

**Follow-up:** When would you use cluster vs worker threads?

Cluster: Best for HTTP server scaling — each worker is a completely independent process with its own memory. Crashes in one worker don't affect others. Naturally suited for stateless request/response workloads.

Worker Threads: Best for CPU-intensive tasks that need shared memory (SharedArrayBuffer). The threads share the same process and can pass data without serialization overhead using `Atomics` and `SharedArrayBuffer`.

**GOTCHA:** Workers in a cluster do NOT share memory — they are separate processes. Any in-memory state (like a cache or session store) must be stored externally (Redis, database). Also, cluster is not the same as thread-level parallelism — each worker still runs on a single thread and has its own event loop.

---

**Q5. What is the `Buffer` type in Node.js and how does it relate to typed arrays?** — Medium

**Answer:**
`Buffer` is Node.js's way to work with raw binary data (byte arrays). Before TypedArrays existed in the browser, Node.js needed its own binary type. Now `Buffer` is a subclass of `Uint8Array`.

```js
// Creating buffers:
Buffer.alloc(10);                    // 10 zero-filled bytes — safe
Buffer.alloc(10, 0xff);              // 10 bytes filled with 0xFF
Buffer.allocUnsafe(10);              // 10 uninitialized bytes — FAST but may contain old data
Buffer.from("hello", "utf8");        // from string
Buffer.from([0x48, 0x65, 0x6c]);     // from array of bytes
Buffer.from(arrayBuffer);            // from ArrayBuffer

// Reading and writing:
const buf = Buffer.from("Hello, World!", "utf8");
buf.length;           // 13 (bytes, not characters)
buf[0];               // 72 — ASCII/UTF-8 code for 'H'
buf.toString("utf8"); // "Hello, World!"
buf.toString("hex");  // "48656c6c6f2c20576f726c6421"
buf.toString("base64"); // "SGVsbG8sIFdvcmxkIQ=="

// Multi-byte reads:
const buf2 = Buffer.alloc(4);
buf2.writeUInt32BE(0xdeadbeef, 0); // write 32-bit big-endian
buf2.readUInt32BE(0);              // 0xdeadbeef (3735928559)

// Buffer is Uint8Array:
const buf3 = Buffer.from([1, 2, 3]);
buf3 instanceof Uint8Array; // true
buf3 instanceof Buffer;     // true

// Slice — shares memory (unlike array slice):
const original = Buffer.from([1, 2, 3, 4, 5]);
const slice = original.slice(1, 4); // [2, 3, 4] — shares memory!
slice[0] = 99;
original[1]; // 99 — modified the original!

// Safe copy: use subarray + Buffer.from:
const copy = Buffer.from(original.subarray(1, 4)); // independent copy

// Concatenating buffers:
const part1 = Buffer.from("Hello ");
const part2 = Buffer.from("World");
const combined = Buffer.concat([part1, part2]);
combined.toString(); // "Hello World"

// Security: Never use Buffer() constructor (deprecated in Node 6+):
new Buffer(100); // DANGEROUS — contains arbitrary old memory data
// Use Buffer.alloc() or Buffer.allocUnsafe() instead
```

**GOTCHA:** `Buffer.allocUnsafe()` is fast but the memory it returns may contain sensitive data from previous allocations (passwords, keys, etc.). Only use it when you will immediately overwrite all bytes. Also, `Buffer` indices are byte offsets — for multibyte strings (UTF-8), `buf.length` is the byte count, not the character count. `"hello 🌍".length` is 6 chars; `Buffer.from("hello 🌍").length` is 10 bytes.

---

**Q6. What are Node.js built-in modules you use most, and what do they provide?** — Easy

**Answer:**
```js
// fs — file system:
const fs = require("fs");
const fsP = require("fs/promises");            // Promise-based (Node 10+)
await fsP.readFile("config.json", "utf8");
await fsP.writeFile("output.txt", data);
await fsP.mkdir("./new-dir", { recursive: true });
const stat = await fsP.stat("file.txt");       // file info

// path — cross-platform path utilities:
const path = require("path");
path.join("/users", "anshu", "docs");          // "/users/anshu/docs" (OS-specific)
path.resolve("./relative");                     // absolute path
path.basename("/foo/bar.txt");                  // "bar.txt"
path.extname("file.txt");                       // ".txt"
path.dirname("/users/anshu/file.txt");          // "/users/anshu"

// http/https — HTTP server and client:
const https = require("https");
https.get("https://api.example.com/data", (res) => {
  let data = "";
  res.on("data", c => data += c);
  res.on("end", () => console.log(JSON.parse(data)));
});

// url — URL parsing:
const { URL } = require("url");
const u = new URL("https://example.com:3000/path?q=1#hash");
u.hostname; // "example.com"
u.searchParams.get("q"); // "1"

// crypto — cryptographic operations:
const crypto = require("crypto");
crypto.randomUUID();                            // UUID v4
crypto.randomBytes(32).toString("hex");         // secure random hex
const hash = crypto.createHash("sha256").update("data").digest("hex");
const hmac = crypto.createHmac("sha256", "secret").update("data").digest("hex");

// os — operating system info:
const os = require("os");
os.cpus().length;       // number of CPU cores
os.totalmem();          // total memory in bytes
os.freemem();           // free memory in bytes
os.platform();          // "linux", "darwin", "win32"
os.homedir();           // "/home/user"

// child_process — spawn external processes:
const { exec, spawn, execFile } = require("child_process");
const { promisify } = require("util");
const execAsync = promisify(exec);
const { stdout } = await execAsync("ls -la");

// util — utility functions:
const util = require("util");
util.promisify(fs.readFile);            // convert callback to Promise
util.inspect(obj, { depth: null });     // deep inspect any object
util.format("%s has %d items", "list", 5); // "list has 5 items"
```

**GOTCHA:** Many older Node.js APIs use the error-first callback pattern (`(err, data) => {}`). Use `util.promisify()` to convert them to Promises, or use the `fs/promises` equivalents directly. Never mix callback-style and Promise-style in the same flow.

---

**Q7. How does Node.js handle environment variables and configuration?** — Easy

**Answer:**
```js
// Accessing environment variables via process.env:
const dbUrl = process.env.DATABASE_URL;       // string or undefined
const port = Number(process.env.PORT) || 3000; // always strings — must convert

// process.env is an object populated from the OS environment:
// $ DATABASE_URL=postgres://localhost/mydb node app.js
// process.env.DATABASE_URL → "postgres://localhost/mydb"

// Best practice — centralize and validate config:
function loadConfig() {
  const required = ["DATABASE_URL", "JWT_SECRET", "API_KEY"];
  const missing = required.filter(key => !process.env[key]);
  if (missing.length > 0) {
    throw new Error(`Missing environment variables: ${missing.join(", ")}`);
  }

  return {
    db: {
      url: process.env.DATABASE_URL,
      maxConnections: parseInt(process.env.DB_MAX_CONNECTIONS ?? "10", 10)
    },
    jwt: {
      secret: process.env.JWT_SECRET,
      expiresIn: process.env.JWT_EXPIRES ?? "1h"
    },
    port: parseInt(process.env.PORT ?? "3000", 10),
    isDev: process.env.NODE_ENV === "development"
  };
}

// .env files — commonly used with dotenv package:
// .env file:
// DATABASE_URL=postgres://localhost/mydb
// JWT_SECRET=supersecret

require("dotenv").config(); // loads .env into process.env
// Or: import "dotenv/config" (ESM)

// Node.js 20.6+ — built-in .env file loading:
// node --env-file=.env app.js
// No dotenv package needed!

// NODE_ENV convention:
process.env.NODE_ENV === "production";   // production mode
process.env.NODE_ENV === "development";  // development mode
process.env.NODE_ENV === "test";         // test mode

// Setting in scripts:
// package.json: "start": "NODE_ENV=production node app.js"
// cross-platform: "start": "cross-env NODE_ENV=production node app.js"
```

**GOTCHA:** `process.env` values are ALWAYS strings — `process.env.PORT === 3000` is always `false`. Always use `parseInt()`, `Number()`, or `parseFloat()` to convert numeric environment variables. Also, environment variables are NOT validated at startup unless you explicitly check — undefined variables silently become `undefined`, which can cause obscure runtime errors much later.

---

**Q8. What is the Node.js `http` module and how do you build a basic HTTP server?** — Medium

**Answer:**
```js
const http = require("http");
const { URL } = require("url");

const server = http.createServer((req, res) => {
  // req: IncomingMessage (readable stream)
  // res: ServerResponse (writable stream)

  const url = new URL(req.url, `http://${req.headers.host}`);
  const method = req.method;
  const pathname = url.pathname;

  // Routing:
  if (method === "GET" && pathname === "/") {
    res.writeHead(200, { "Content-Type": "application/json" });
    res.end(JSON.stringify({ message: "Hello World" }));
    return;
  }

  if (method === "POST" && pathname === "/users") {
    // Read request body (it's a stream):
    let body = "";
    req.on("data", chunk => { body += chunk.toString(); });
    req.on("end", () => {
      try {
        const data = JSON.parse(body);
        // Process...
        res.writeHead(201, { "Content-Type": "application/json" });
        res.end(JSON.stringify({ id: 1, ...data }));
      } catch {
        res.writeHead(400);
        res.end("Invalid JSON");
      }
    });
    return;
  }

  // 404:
  res.writeHead(404, { "Content-Type": "application/json" });
  res.end(JSON.stringify({ error: "Not Found" }));
});

server.listen(3000, () => {
  console.log("Server running at http://localhost:3000");
});

// Graceful shutdown:
process.on("SIGTERM", () => {
  server.close(() => {
    console.log("Server closed");
    process.exit(0);
  });
});

// https — same API, different module:
const https = require("https");
const fs = require("fs");
https.createServer({
  key: fs.readFileSync("key.pem"),
  cert: fs.readFileSync("cert.pem")
}, (req, res) => {
  // same handler as http
}).listen(443);
```

**Follow-up:** Why use Express or Fastify instead of the raw `http` module?

Raw `http` requires you to manually parse routes, parse request bodies, handle query parameters, manage cookies, and more. Frameworks like Express add middleware (composable request/response transformers), router hierarchies, error handling middleware, and body parsers. Fastify additionally offers schema-based serialization/validation, much better performance, and first-class TypeScript support.

**GOTCHA:** Request body is a stream in raw Node.js — you MUST collect chunks and concatenate them. Forgetting this is a very common beginner mistake. Also, `res.end()` must always be called — if you forget it on some code path, the request hangs forever. Frameworks handle this automatically.

---

**Q9. What are Worker Threads in Node.js and when should you use them?** — Hard

**Answer:**
Worker Threads (Node.js 10.5+, stable 12+) allow true parallel JavaScript execution in separate threads within the same process. They share memory via `SharedArrayBuffer`.

```js
const { Worker, isMainThread, parentPort, workerData } = require("worker_threads");

// worker.js — runs in a worker thread:
if (!isMainThread) {
  const { numbers } = workerData;

  // CPU-intensive work — won't block main thread:
  let sum = 0;
  for (let i = 0; i < numbers.length; i++) {
    sum += numbers[i];
  }

  parentPort.postMessage({ result: sum });
  process.exit(0);
}

// main.js — spawns worker:
if (isMainThread) {
  const numbers = new Array(1_000_000).fill(1);

  const worker = new Worker(__filename, {
    workerData: { numbers } // data is CLONED to worker (structured clone)
  });

  worker.on("message", ({ result }) => {
    console.log("Sum:", result); // 1000000
  });

  worker.on("error", err => console.error("Worker error:", err));
  worker.on("exit", code => {
    if (code !== 0) console.error("Worker exited with code:", code);
  });
}

// Shared memory — faster than cloning for large data:
const sharedBuffer = new SharedArrayBuffer(4 * 1024 * 1024); // 4MB
const sharedArray = new Int32Array(sharedBuffer);

// Fill from main thread:
sharedArray.fill(1);

const worker = new Worker("./compute.js", {
  workerData: { sharedBuffer } // buffer is SHARED, not cloned
});

// Atomics — thread-safe operations on shared memory:
Atomics.add(sharedArray, 0, 1);    // atomic increment
Atomics.load(sharedArray, 0);      // atomic read
Atomics.wait(sharedArray, 0, 0);   // block until value changes
Atomics.notify(sharedArray, 0, 1); // wake up one waiting thread

// Worker pool pattern — reuse workers for multiple tasks:
class WorkerPool {
  constructor(size, workerScript) {
    this.pool = Array.from({ length: size }, () => ({
      worker: new Worker(workerScript),
      idle: true
    }));
  }

  run(data) {
    const slot = this.pool.find(s => s.idle);
    if (!slot) throw new Error("No idle workers");
    slot.idle = false;
    return new Promise((resolve, reject) => {
      slot.worker.once("message", result => {
        slot.idle = true;
        resolve(result);
      });
      slot.worker.postMessage(data);
    });
  }
}
```

**Use cases for Worker Threads:**
- Image/video processing
- Cryptographic operations
- Parsing large files
- Machine learning inference
- Any CPU-bound work that would block the main event loop

**GOTCHA:** Worker threads cannot access DOM (they're in Node.js anyway), cannot share closures, and data passed via `postMessage` is structured-cloned (copied) unless you use `SharedArrayBuffer` with transferables. The main thread itself is NOT blocked during worker execution — that's the whole point.

---

**Q10. What is `AbortController` and how do you use it in Node.js?** — Medium

**Answer:**
`AbortController` is a Web API now available natively in Node.js 15+ that allows you to cancel asynchronous operations — fetch requests, streams, timers, and custom async work.

```js
// Native fetch with AbortController:
const controller = new AbortController();
const { signal } = controller;

// Start a fetch:
const fetchPromise = fetch("https://api.example.com/data", { signal });

// Cancel after 5 seconds:
const timeout = setTimeout(() => controller.abort(), 5000);

try {
  const res = await fetchPromise;
  clearTimeout(timeout);
  const data = await res.json();
  return data;
} catch (err) {
  if (err.name === "AbortError") {
    console.log("Request was cancelled");
  } else {
    throw err;
  }
}

// AbortSignal.timeout() — shorthand for timeout signal (Node 17.3+):
const res = await fetch("https://api.example.com", {
  signal: AbortSignal.timeout(5000) // auto-aborts after 5 seconds
});

// AbortSignal.any() — abort when ANY of multiple signals abort (Node 20+):
const userCancel = new AbortController();
const combined = AbortSignal.any([
  AbortSignal.timeout(10000),   // 10s timeout
  userCancel.signal              // manual cancel
]);
fetch("/api", { signal: combined });

// Custom async operations that respect abort:
async function longTask(signal) {
  for (let i = 0; i < 1000; i++) {
    if (signal.aborted) {
      throw new DOMException("Task aborted", "AbortError");
    }
    await doChunk(i);
  }
}

// Stream cancellation:
const { pipeline } = require("stream/promises");
const controller2 = new AbortController();
await pipeline(
  createReadStream("input.txt"),
  transform,
  createWriteStream("output.txt"),
  { signal: controller2.signal }
);
controller2.abort(); // cancels the pipeline
```

**GOTCHA:** Once aborted, an `AbortController` cannot be "un-aborted" — it's a one-way signal. Create a new `AbortController` for each new operation. Also, aborting doesn't automatically cancel any in-flight network requests that have already started — it signals to the consumer (fetch, etc.) which must check the signal. For native `fetch`, the cancellation is handled correctly. For custom code, always check `signal.aborted` in loops and cooperative cancellation points.

---

**Q11. What is `util.promisify()` and how does it work internally?** — Medium

**Answer:**
`util.promisify()` converts Node.js-style error-first callback functions into Promise-returning functions.

```js
const util = require("util");
const fs = require("fs");

// Original callback API:
fs.readFile("file.txt", "utf8", (err, data) => {
  if (err) throw err;
  console.log(data);
});

// Promisified:
const readFile = util.promisify(fs.readFile);
const data = await readFile("file.txt", "utf8");

// How promisify works internally (simplified):
function promisify(original) {
  return function (...args) {
    return new Promise((resolve, reject) => {
      original.call(this, ...args, (err, ...results) => {
        if (err) {
          reject(err);
        } else if (results.length === 1) {
          resolve(results[0]);
        } else {
          resolve(results); // multi-arg callbacks
        }
      });
    });
  };
}

// util.promisify.custom — for functions that don't follow error-first convention:
function customAsync(value, callback) {
  setTimeout(() => callback(value * 2), 100); // no error argument!
}
customAsync[util.promisify.custom] = (value) =>
  new Promise(resolve => setTimeout(() => resolve(value * 2), 100));

const promisified = util.promisify(customAsync);
await promisified(21); // 42

// Better: just use fs/promises directly (Node 10+):
const { readFile: readFileP } = require("fs/promises");
const data2 = await readFileP("file.txt", "utf8");

// Promisify exec:
const { exec } = require("child_process");
const execAsync = util.promisify(exec);
const { stdout, stderr } = await execAsync("ls -la");
```

**GOTCHA:** `util.promisify()` only works with the Node.js error-first callback convention — the callback must be the LAST argument and must take `(err, result)` as its first two parameters. Functions with different conventions need `util.promisify.custom`. Also, promisified functions lose the ability to cancel — once you call one, the underlying I/O operation continues even if you don't await the Promise.

---

**Q12. What is the `async_hooks` module and how do you track async context?** — Hard

**Answer:**
`async_hooks` provides APIs to track the lifecycle of asynchronous resources. `AsyncLocalStorage` (built on top) is the practical tool for passing context (like a request ID or user session) through async operations without explicitly passing parameters.

```js
const { AsyncLocalStorage } = require("async_hooks");

// Create a storage instance — one per type of context:
const requestContext = new AsyncLocalStorage();

// Express middleware — set context at the request boundary:
app.use((req, res, next) => {
  const ctx = {
    requestId: crypto.randomUUID(),
    userId: req.user?.id,
    startTime: Date.now()
  };

  // Run the rest of the request handling WITHIN this context:
  requestContext.run(ctx, next);
});

// Anywhere in the call tree (sync or async!) — get the context:
function logWithContext(message) {
  const ctx = requestContext.getStore();
  console.log(`[${ctx?.requestId ?? "no-context"}] ${message}`);
}

// Works through any async boundary:
async function processUser(userId) {
  logWithContext(`Processing user ${userId}`); // Has requestId from middleware!
  await db.query(`SELECT * FROM users WHERE id = $1`, [userId]);
  logWithContext("Query complete");
}

// Router handler — no context passing needed:
app.get("/users/:id", async (req, res) => {
  const user = await processUser(req.params.id);
  // requestContext.getStore() returns the same ctx set in middleware
  res.json(user);
});

// Error logging with request context:
async function safeRun(fn) {
  try {
    return await fn();
  } catch (error) {
    const ctx = requestContext.getStore();
    logger.error({
      error: error.message,
      requestId: ctx?.requestId,
      userId: ctx?.userId
    });
    throw error;
  }
}

// AsyncLocalStorage lifecycle:
// .run(store, callback) — execute callback with store, propagates to all child async operations
// .getStore() — retrieve current store (returns undefined if not in .run() context)
// .enterWith(store) — set store for current context (avoid in most cases)
// .exit(callback) — execute callback without inheriting parent store
```

**Follow-up:** What did developers use before `AsyncLocalStorage`?

Before Node.js 12.17+, they used the low-level `async_hooks` API directly (verbose and error-prone), continuation-local storage (CLS) libraries, or passed context explicitly through every function parameter (prop-drilling for Node.js).

**GOTCHA:** `requestContext.getStore()` returns `undefined` if called outside any `.run()` context — not an error. Always null-check the store. Also, the store object is shared (not copied) across all async operations within a `.run()` call — mutating it affects all concurrent accesses. Use a new store object per context root (each request), and treat the store as read-only or use careful coordination if you must mutate.

---

*Next: [15-Patterns-Architecture.md](./15-Patterns-Architecture.md)*
