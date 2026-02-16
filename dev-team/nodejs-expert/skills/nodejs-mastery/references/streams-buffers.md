# Node.js Streams and Buffers

## Stream Types

```typescript
import { Readable, Writable, Transform, Duplex, PassThrough } from 'node:stream';

// Readable — source of data
const readable = new Readable({
  read(size) {
    this.push('data chunk');
    this.push(null); // Signal end
  },
});

// Writable — destination for data
const writable = new Writable({
  write(chunk, encoding, callback) {
    process.stdout.write(chunk);
    callback(); // Signal ready for next chunk
  },
});

// Transform — read + write with transformation
const upperCase = new Transform({
  transform(chunk, encoding, callback) {
    callback(null, chunk.toString().toUpperCase());
  },
});

// Duplex — independent read and write (e.g., TCP socket)
// PassThrough — Transform that passes data unchanged (useful for tapping)
```

## Pipeline (Preferred Pattern)

```typescript
import { pipeline } from 'node:stream/promises';
import { createReadStream, createWriteStream } from 'node:fs';
import { createGzip, createGunzip } from 'node:zlib';

// Compress file
await pipeline(
  createReadStream('input.txt'),
  createGzip(),
  createWriteStream('input.txt.gz'),
);

// Decompress and transform
await pipeline(
  createReadStream('data.json.gz'),
  createGunzip(),
  new Transform({
    objectMode: true,
    transform(chunk, _, cb) {
      try {
        const records = JSON.parse(chunk.toString());
        for (const record of records) {
          this.push(JSON.stringify(record) + '\n');
        }
        cb();
      } catch (err) { cb(err as Error); }
    },
  }),
  createWriteStream('output.jsonl'),
);
```

## Async Iterators with Streams

```typescript
import { createReadStream } from 'node:fs';
import { createInterface } from 'node:readline';

// Line-by-line file processing
const rl = createInterface({
  input: createReadStream('large-file.csv'),
  crlfDelay: Infinity,
});

let lineCount = 0;
for await (const line of rl) {
  lineCount++;
  const fields = line.split(',');
  await processRecord(fields);
}
console.log(`Processed ${lineCount} lines`);
```

## Backpressure Handling

```typescript
// Writable.write() returns false when internal buffer is full
const writable = createWriteStream('output.txt');

function writeData(data: string[]): Promise<void> {
  return new Promise((resolve, reject) => {
    let i = 0;
    write();

    function write() {
      while (i < data.length) {
        const canContinue = writable.write(data[i]);
        i++;
        if (!canContinue) {
          // Buffer full — wait for drain
          writable.once('drain', write);
          return;
        }
      }
      resolve();
    }

    writable.on('error', reject);
  });
}

// pipeline() handles backpressure automatically — prefer it
```

## HTTP Streaming

```typescript
import { createReadStream } from 'node:fs';
import { stat } from 'node:fs/promises';

// Stream file as HTTP response (Express)
app.get('/download/:file', async (req, res) => {
  const filePath = `/files/${req.params.file}`;
  const fileStat = await stat(filePath);

  res.setHeader('Content-Length', fileStat.size);
  res.setHeader('Content-Type', 'application/octet-stream');

  const stream = createReadStream(filePath);
  stream.pipe(res);
  stream.on('error', (err) => {
    if (!res.headersSent) res.status(500).send('File read error');
  });
});

// Stream JSON array
app.get('/api/users/export', async (req, res) => {
  res.setHeader('Content-Type', 'application/json');
  res.write('[');

  let first = true;
  const cursor = db.users.findManyCursor();
  for await (const user of cursor) {
    if (!first) res.write(',');
    res.write(JSON.stringify(user));
    first = false;
  }

  res.write(']');
  res.end();
});
```

## Buffer Operations

```typescript
// Create buffers
const buf1 = Buffer.from('Hello', 'utf-8');
const buf2 = Buffer.alloc(10); // Zero-filled
const buf3 = Buffer.allocUnsafe(10); // Uninitialized (faster)

// Read/write
buf2.writeUInt32BE(0xdeadbeef, 0);
const value = buf2.readUInt32BE(0); // 3735928559

// Concatenate
const combined = Buffer.concat([buf1, buf2]);

// Slice (shared memory)
const slice = buf1.subarray(0, 3); // 'Hel' — modifying slice modifies buf1

// Copy (independent)
const copy = Buffer.from(buf1);

// Compare
Buffer.compare(buf1, buf2); // -1, 0, or 1
buf1.equals(copy); // true

// Encoding conversion
const base64 = buf1.toString('base64');
const hex = buf1.toString('hex');
const back = Buffer.from(base64, 'base64').toString('utf-8');
```
