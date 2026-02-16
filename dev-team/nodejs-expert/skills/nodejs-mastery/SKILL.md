---
name: Node.js Mastery
description: This skill should be used when the user asks about "Node.js event loop", "Node streams", "worker threads", "Node.js cluster", "async patterns Node", "Node.js buffers", "Node process management", "PM2", or "Node.js internals". It covers runtime internals, async patterns, streams, worker threads, and process management.
---

# Node.js Runtime Mastery

## Event Loop Phases

```
   ┌───────────────────────────┐
┌─>│           timers          │  setTimeout, setInterval
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │  I/O callbacks deferred to next loop
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │  Internal use only
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           poll            │  Retrieve new I/O events; execute I/O callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           check           │  setImmediate callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      │  socket.on('close', ...)
   └───────────────────────────┘

Microtasks (process.nextTick, Promise) run between EVERY phase.
```

## Async Patterns

```typescript
// Promise.all — parallel independent operations
const [users, products] = await Promise.all([
  db.users.findMany(),
  db.products.findMany(),
]);

// Promise.allSettled — parallel, don't fail fast
const results = await Promise.allSettled([
  fetch('/api/a'),
  fetch('/api/b'),
  fetch('/api/c'),
]);
const successes = results.filter(r => r.status === 'fulfilled').map(r => r.value);

// Async iteration
for await (const chunk of readableStream) {
  await processChunk(chunk);
}

// Concurrency limit with p-limit
import pLimit from 'p-limit';
const limit = pLimit(5); // Max 5 concurrent
const results = await Promise.all(
  urls.map(url => limit(() => fetch(url)))
);
```

## Streams

```typescript
import { pipeline } from 'node:stream/promises';
import { createReadStream, createWriteStream } from 'node:fs';
import { createGzip } from 'node:zlib';
import { Transform } from 'node:stream';

// Pipeline — automatic error handling and cleanup
await pipeline(
  createReadStream('input.csv'),
  new Transform({
    transform(chunk, encoding, callback) {
      callback(null, chunk.toString().toUpperCase());
    },
  }),
  createGzip(),
  createWriteStream('output.csv.gz'),
);

// Readable stream as async iterator
const readable = createReadStream('data.jsonl');
for await (const line of readable) {
  const record = JSON.parse(line.toString());
  await processRecord(record);
}
```

## Worker Threads

```typescript
import { Worker, isMainThread, parentPort, workerData } from 'node:worker_threads';

if (isMainThread) {
  const worker = new Worker(new URL('./worker.ts', import.meta.url), {
    workerData: { input: largeDataSet },
  });
  worker.on('message', result => console.log('Result:', result));
  worker.on('error', err => console.error('Worker error:', err));
} else {
  const result = heavyComputation(workerData.input);
  parentPort!.postMessage(result);
}
```

## Graceful Shutdown

```typescript
async function gracefulShutdown(signal: string) {
  console.log(`Received ${signal}. Starting graceful shutdown...`);
  server.close(() => console.log('HTTP server closed'));
  await db.$disconnect();
  await redis.quit();
  process.exit(0);
}

process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown('SIGINT'));

// Prevent abrupt crashes
process.on('unhandledRejection', (reason) => {
  console.error('Unhandled Rejection:', reason);
  // Log but don't exit — let the error handler deal with it
});
```

## References

- [Runtime Internals](references/runtime-internals.md) — Event loop deep dive, libuv, microtasks vs macrotasks, nextTick vs setImmediate.
- [Streams and Buffers](references/streams-buffers.md) — Stream patterns, backpressure, Buffer API, pipeline composition.
