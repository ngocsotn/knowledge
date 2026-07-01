# Memory-Efficient Buffer & Stream Handling

Deep dive into how node_modules, buffers, streams, and backpressure are managed at scale.

## Buffers vs. Streams
- **Buffers:** Memory chunks allocated in RAM to store raw binary data temporarily. Great for small files, but loads the entire file into memory, causing memory exhaustion at scale.
- **Streams:** Read and write data chunk-by-chunk without storing the entire payload in memory. Keeps memory footprint small and constant (e.g., 32KB).

## Handling Backpressure
- Backpressure occurs when the data reading source produces data faster than the destination writing sink can consume/write it.
- **Mitigation:** Pause the reading stream when the write buffer is full, and resume reading only when the write stream triggers the `drain` event.

```go
// Pipes incoming network payload directly to S3 without RAM buffering (Go)
_, err := io.Copy(s3UploadStream, fileUploadRequestReader)
```

## Interview Questions & Answers

### Q1: Under high concurrency, an attacker attempts to flood your storage bucket with heavy files, causing cost starvation. How do you prevent this?
- **Answer:** 
  1. **Strict Content-Length Signatures:** Use S3 policy constraints like `["content-length-range", 1024, 10485760]` to reject files larger than 10MB at S3's edge nodes.
  2. **API Rate-Limiting:** Apply token-bucket rate limits on presigned URL generation endpoints.
  3. **Object Expiry & Lifecycle Cleanups:** Configure S3 Lifecycle rules to clean up incomplete multipart uploads after 24 hours.
