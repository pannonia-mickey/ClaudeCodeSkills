# Node.js Runtime Internals

## Event Loop Deep Dive

### Phase Details

1. **Timers**: Executes callbacks from `setTimeout()` and `setInterval()`. A timer specifies a threshold after which a callback may execute, not the exact time.

2. **Pending Callbacks**: Executes I/O callbacks deferred to the next loop iteration (e.g., TCP error callbacks).

3. **Poll**: Retrieves new I/O events. Executes I/O-related callbacks. Node will block here when appropriate if there's nothing else scheduled.

4. **Check**: `setImmediate()` callbacks execute here, immediately after the poll phase.

5. **Close**: `socket.on('close', ...)` callbacks.

### Microtask Queue Priority

```typescript
// Microtasks run between EVERY event loop phase
// Priority: process.nextTick > Promise > queueMicrotask

setTimeout(() => console.log('1: setTimeout'), 0);
setImmediate(() => console.log('2: setImmediate'));
Promise.resolve().then(() => console.log('3: Promise'));
process.nextTick(() => console.log('4: nextTick'));
queueMicrotask(() => console.log('5: queueMicrotask'));

// Output: 4, 3, 5, 1, 2
// nextTick always runs first, then Promise microtasks, then macrotasks
```

### nextTick vs setImmediate vs setTimeout(0)

```typescript
// process.nextTick — runs BEFORE any I/O, highest priority microtask
// Use for: ensuring callback runs before next event loop phase
process.nextTick(() => { /* runs immediately after current operation */ });

// setImmediate — runs in check phase, AFTER I/O
// Use for: yielding to event loop after I/O operations
setImmediate(() => { /* runs in next check phase */ });

// setTimeout(fn, 0) — runs in timers phase, minimum 1ms delay
// Use for: deferring execution with minimum delay
setTimeout(() => { /* runs in next timers phase */ }, 0);

// Inside an I/O callback, setImmediate always runs before setTimeout
const fs = require('fs');
fs.readFile('file.txt', () => {
  setImmediate(() => console.log('immediate'));  // First
  setTimeout(() => console.log('timeout'), 0);   // Second
});
```

## libuv Thread Pool

```typescript
// libuv uses a thread pool (default 4 threads) for:
// - File system operations (fs.readFile, fs.writeFile)
// - DNS lookups (dns.lookup, NOT dns.resolve)
// - Crypto operations (crypto.pbkdf2, crypto.randomBytes)
// - Zlib compression

// Increase thread pool for I/O-heavy apps
// Set BEFORE requiring any module
process.env.UV_THREADPOOL_SIZE = '16'; // Max 1024

// These use the thread pool:
import { pbkdf2 } from 'node:crypto';
pbkdf2('password', 'salt', 100000, 64, 'sha512', (err, key) => { /* ... */ });

// These do NOT use the thread pool (use OS async I/O):
// - Network I/O (TCP, UDP, HTTP)
// - dns.resolve (uses c-ares)
// - Timers
```

## Event Loop Monitoring

```typescript
// Detect event loop blocking
let lastCheck = Date.now();
setInterval(() => {
  const now = Date.now();
  const lag = now - lastCheck - 1000; // Expected interval - 1000ms
  if (lag > 100) {
    console.warn(`Event loop lag: ${lag}ms`);
  }
  lastCheck = now;
}, 1000);

// Using monitorEventLoopDelay (Node 12+)
import { monitorEventLoopDelay } from 'node:perf_hooks';
const histogram = monitorEventLoopDelay({ resolution: 20 });
histogram.enable();

setInterval(() => {
  console.log({
    min: histogram.min / 1e6,     // Convert ns to ms
    max: histogram.max / 1e6,
    mean: histogram.mean / 1e6,
    p99: histogram.percentile(99) / 1e6,
  });
  histogram.reset();
}, 5000);
```

## Memory Management

```typescript
// V8 heap info
const mem = process.memoryUsage();
console.log({
  rss: `${(mem.rss / 1024 / 1024).toFixed(1)} MB`,          // Resident Set Size
  heapTotal: `${(mem.heapTotal / 1024 / 1024).toFixed(1)} MB`, // V8 total heap
  heapUsed: `${(mem.heapUsed / 1024 / 1024).toFixed(1)} MB`,  // V8 used heap
  external: `${(mem.external / 1024 / 1024).toFixed(1)} MB`,   // C++ objects bound to JS
});

// Increase heap limit
// node --max-old-space-size=4096 app.js

// Force garbage collection (for debugging only)
// node --expose-gc app.js
// global.gc();

// Common memory leaks:
// 1. Growing arrays/maps that are never cleaned
// 2. Event listeners not removed (EventEmitter)
// 3. Closures holding references to large objects
// 4. Unbounded caches without eviction
// 5. Streams not properly destroyed
```
