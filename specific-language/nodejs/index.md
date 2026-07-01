# Node.js Core, Performance & Tools

Comprehensive guide covering Node.js architecture, streams, worker threads, and package management.

## 1. Buffers vs. Streams

Understanding how NodeJS handles data in memory is crucial for scale.

### A. Buffers
A **Buffer** represents a fixed-size chunk of memory allocated outside the V8 heap.
- **How it works**: Reads the entire file/payload into physical RAM at once before processing.
- **Problem**: If you try to read a 4GB file using `fs.readFile()`, a standard server with 2GB of RAM will crash instantly with an `Out of Memory` (OOM) error. V8 heap limits also apply.

### B. Streams
A **Stream** handles data chunk-by-chunk (in small pieces, typically 64KB buffers), allowing processing of files larger than the available physical RAM.
- **Types**: Readable, Writable, Duplex, Transform.
- **Example (No RAM issues)**:
  ```javascript
  const readable = fs.createReadStream('huge-4gb-file.txt');
  const writable = fs.createWriteStream('copy.txt');
  readable.pipe(writable); // Streams data smoothly with ~20MB RAM usage
  ```

#### Handling Backpressure
- **Backpressure** occurs when a Readable stream reads data much faster than the Writable stream can write it.
- **How it's resolved**: The Writable stream tells the Readable stream to pause reading when the internal write queue buffer is full (reaches `highWaterMark`). Once the queue is drained and written to disk, the Writable stream emits a `'drain'` event, telling the Readable stream to resume reading. Using `.pipe()` handles backpressure automatically.

## Package Management and node_modules
## 1. Core Architectural Comparison

| Dimension | npm (Default) | pnpm | Yarn (Classic v1) | Yarn (PnP - v2+) | Bun (Native) |
| :
