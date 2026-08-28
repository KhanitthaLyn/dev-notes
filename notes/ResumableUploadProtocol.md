# Question

You upload a 5 GB file to Google Drive. Your internet disconnects at 97%.
When you reconnect, it resumes instead of starting over. How?

# Answer

# The Big Picture & Analogy

Imagine building a brick wall, one brick at a time, and the power suddenly goes out mid-way.

If you "don't remember" where you left off, you'd have to **tear down the whole wall and start over from scratch** — this is exactly what happens with a simple upload (sending the whole file in one request) when the connection drops.

But if you **have someone keeping track: "970 out of 1000 bricks laid"** — when the power comes back, you just continue from brick 971, no need to demolish anything.

This is the core of resumable upload: **split the file into small chunks, and have the server remember which chunks it has already received.**

# Why Do We Need It? (The Problem With Simple Uploads)

A basic upload (simple upload) works like this:
```
Client: sends the entire 5GB file in one HTTP request
Server: waits for all 5GB before responding
```

The problem: if the connection drops at 97% (4.95GB out of 5GB) — **the whole HTTP request is considered "incomplete."** Neither the client nor server can be certain how reliable that 97% already sent actually is (some bytes might have gotten corrupted along the way). The only safe option is to **discard everything and start over.**

Solving this requires three components working together.

# Core Logic & How It Works

**Component 1: Splitting the file into chunks**
```
5GB file → split into 5-10MB chunks → roughly 500-1000 chunks
```
Instead of sending the whole file in one request, it's sent as many separate HTTP requests, each carrying one chunk.

**Component 2: The server keeps an upload session + tracks progress**
```
POST /upload/initiate
Response: { "uploadId": "abc-123", "chunkSize": 5242880 }

PUT /upload/abc-123/chunk/0    <- first chunk
PUT /upload/abc-123/chunk/1    <- second chunk
...
PUT /upload/abc-123/chunk/970  <- disconnects here!
```
The server maintains **state for this upload session** (in a database or Redis), tracking how many chunks `uploadId: abc-123` has received, updated every time a chunk arrives — **not waiting until the whole file finishes to record anything.**

**Component 3: The client asks the server "how far did we get?" on reconnect**
```
GET /upload/abc-123/status
Response: { "receivedChunks": [0, 1, 2, ..., 969], "nextExpectedChunk": 970 }
```
When the internet comes back, the client **doesn't immediately restart the upload** — it first asks the server "which chunks have you received so far?" then sends **only the remaining chunks** (970 onward).

**Component 4: Checksums for integrity verification**
```
PUT /upload/abc-123/chunk/970
Headers: Content-MD5: 5d41402abc4b2a76b9719d911017c592
Body: [chunk binary data]
```
Every chunk includes a **checksum (MD5/SHA-256)** — the server computes the checksum of the received data and compares it. If they don't match (corrupted in transit), it **rejects just that chunk and asks the client to resend only that one** — no need to resend the whole file.

# Trade-offs & When to Use

| Approach | Pros | Cons |
|---|---|---|
| **Simple Upload** (whole file in one request) | Very simple to implement | Breaks frequently on large files or unstable networks, requiring a full restart |
| **Chunked/Resumable Upload** | Resilient to unstable networks, saves bandwidth on interruption | Much more complex — requires managing session state, and cleanup for abandoned sessions (orphaned uploads) |

**When to use Simple Upload:** small files (< 10-50MB) that upload quickly, within a few seconds — the risk of interruption is low, and the added complexity isn't worth it.

**When to use Resumable Upload:** large files (videos, backups), or when targeting users on unstable networks (mobile, weak signal areas) — Google Drive, YouTube, and Dropbox all use exactly this pattern.

**Additional trade-off to manage:** you need a **TTL/cleanup job** for abandoned upload sessions (e.g. a user leaves a tab open and never resumes) — otherwise storage fills up with orphaned partial files nobody's using.

# Real-World Scenario (Ecommerce Domain)

Say a Seller Dashboard needs to upload large product review videos (Live Commerce recordings):

```java
@RestController
public class ResumableUploadController {
    
    @PostMapping("/uploads/initiate")
    public UploadSession initiateUpload(@RequestBody InitiateRequest req) {
        String uploadId = UUID.randomUUID().toString();
        UploadSession session = new UploadSession(uploadId, req.getFileSize(), req.getFileName());
        uploadSessionRepo.save(session); // persist state in DB/Redis
        return session;
    }
    
    @PutMapping("/uploads/{uploadId}/chunks/{chunkIndex}")
    public ResponseEntity<?> uploadChunk(
        @PathVariable String uploadId, 
        @PathVariable int chunkIndex,
        @RequestHeader("Content-MD5") String checksum,
        @RequestBody byte[] chunkData) {
        
        // 1. always verify checksum first
        String actualChecksum = calculateMD5(chunkData);
        if (!actualChecksum.equals(checksum)) {
            return ResponseEntity.status(422).body("Checksum mismatch, please resend this chunk");
        }
        
        // 2. write the chunk to storage (S3 multipart upload, or a temp file)
        storageService.writeChunk(uploadId, chunkIndex, chunkData);
        
        // 3. update session state to mark this chunk as received
        uploadSessionRepo.markChunkReceived(uploadId, chunkIndex);
        
        return ResponseEntity.ok().build();
    }
    
    @GetMapping("/uploads/{uploadId}/status")
    public UploadStatus getStatus(@PathVariable String uploadId) {
        // client calls this on reconnect to know where to resume from
        return uploadSessionRepo.getReceivedChunks(uploadId);
    }
}
```

# Lead's Key Takeaway

> **"The core of a resumable system isn't just splitting work into small pieces — it's having a 'source of truth' that can always tell you exactly where progress currently stands, no matter where things crashed."**
>
> A good Lead sees this pattern as bigger than just "file upload" — it's the same underlying concept as **idempotency + checkpointing**, applicable across many contexts: batch jobs processing huge datasets (checkpointing which record was last processed), or distributed data migrations. Anywhere "a large task might get interrupted midway" should have a mechanism like this, rather than an all-or-nothing design.
