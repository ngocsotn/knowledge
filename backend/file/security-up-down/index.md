# File Upload Security

Production-grade security and validation patterns for uploaded assets.

: The RCE & XS Threats

Allowing users to upload files to a server is one of the highest-risk operations in web engineering. Without rigid security gateways, attackers can exploit file uploads to compromise host servers or execute client-side attacks.

### Core Attack Vectors:

#### 1. Remote Code Execution (RCE) via Execution Bypass
* **The Vulnerability:** An attacker uploads a script file (e.g., `backdoor.php`, `exploit.jsp`, or `shell.asp`) disguised with an allowed image extension. If the web server (Nginx/Apache) is configured to execute scripts inside the uploads directory, the attacker can execute system shell commands by requesting the file's public URL:
  `GET https://app.com/uploads/backdoor.php?cmd=whoami`
* **Double Extensions:** Attackers attempt to bypass extension filters using names like `malicious.php.png` or `exploit.png.php`. If the server's regex parser is poorly written or reads extensions from left to right, it accepts the file and executes it.

#### 2. Cross-Site Scripting (XSS) via MIME Sniffing
* **The Vulnerability:** An attacker uploads an HTML file with embedded javascript, but renames the extension to `avatar.jpg`. When a client requests this image, if the server returns a generic `Content-Type: application/octet-stream` or if the client browser has **MIME Sniffing** enabled:
  1. The browser reads the file's first few bytes, ignores the `.jpg` extension, and detects HTML tags.
  2. The browser renders the page as HTML and executes the embedded Javascript.
  3. **The Result:** Full XSS exploit in the domain's context, allowing cookie theft or API session hijacking.

#### 3. Zip Slip (Directory Traversal via Archive Extraction)
* **The Vulnerability:** When allowing users to upload compressed archives (`.zip`, `.tar.gz`), an attacker embeds files containing directory traversal characters in their internal file paths (e.g., `../../../../var/www/html/index.php`).
* When the backend extracts the archive without validating path prefixes, the extractor overwrites critical system files outside the target directory, executing code or crashing the host.

#### 4. Host-Level Malware Propagation
* **The Vulnerability:** Users upload infected PDFs or executables. Other legitimate users download these files, propagating malware and ransomware across the enterprise organization.

---

## 2. Production-Grade Backend File Upload Defense

To secure the backend against malicious file uploads, you must implement a multi-layered validation pipeline. Never rely on client-side validation alone.

```
Incoming File 
     │
     ▼
[Step 1: Size & Extension Gating] ──► Reject if > 10MB or extension not in Whitelist
     │
     ▼
[Step 2: Magic Bytes Check] ───────► Read first 512 bytes, assert matching File Signature
     │
     ▼
[Step 3: Filename Sanitization] ───► Discard original name, generate secure random UUID
     │
     ▼
[Step 4: Live Malware Scan] ───────► Scan payload stream using ClamAV API
     │
     ▼
[Step 5: Storage Isolation] ────────► Write payload to isolated, non-executable Object Storage (S3)
```

### A. Strict Extension Whitelisting (Never Blacklist)
Never use a blacklist (e.g., blocking `.php` or `.sh`) because attackers will find bypasses (e.g., `.php5`, `.phtml`, `.shtml`).
* Maintain a strict, explicit **Whitelist** of allowed extensions:
  ```go
  var AllowedExtensions = map[string]bool{
      ".jpg":  true,
      ".jpeg": true,
      ".png":  true,
      ".pdf":  true,
  }
  ```

### B. Cryptographic Magic Bytes (File Signature) Validation
Attackers can easily rename `shell.php` to `shell.jpg`. To verify the true structure of the file, you must validate its **Magic Bytes (File Signatures)**—the fixed sequence of hex bytes stored at the very beginning of the file's binary stream.

| File Type | Extension | Common Hex Signature (Magic Bytes) |
| :--- | :---: | :--- |
| **JPEG / JPG** | `.jpg` / `.jpeg` | `FF D8 FF` |
| **PNG** | `.png` | `89 50 4E 47 0D 0A 1A 0A` |
| **PDF** | `.pdf` | `25 50 44 46` (representing `%PDF`) |
| **GIF** | `.gif` | `47 49 46 38` (representing `GIF8`) |

#### Implementation in Go:
```go
func ValidateMagicBytes(fileHeader []byte, expectedExtension string) bool {
    if len(fileHeader) < 4 {
        return false
    }
    // Check PDF Signature
    if expectedExtension == ".pdf" {
        return bytes.Equal(fileHeader[:4], []byte{0x25, 0x50, 0x44, 0x46})
    }
    // Check PNG Signature
    if expectedExtension == ".png" {
        return bytes.Equal(fileHeader[:4], []byte{0x89, 0x50, 0x4E, 0x47})
    }
    // Check JPEG Signature
    if expectedExtension == ".jpg" || expectedExtension == ".jpeg" {
        return bytes.Equal(fileHeader[:3], []byte{0xFF, 0xD8, 0xFF})
    }
    return false
}
```

### C. Filename Sanitization & Storage Path Isolation
1. **Randomize Filenames:** Never preserve the user's original filename (e.g., `../../index.php` or `exploit.php`). Discard it entirely and generate a cryptographically secure random UUID v4 with the whitelisted extension:
   `filename = uuid.New().String() + ".png"`
2. **Storage Isolation (Crucial):** Never store uploaded files inside the web server's application directory or under an execution tree. Save them directly to an **isolated cloud object store** (such as AWS S3 or Google Cloud Storage) or a completely separate, non-executable storage volume.
3. **No-Execution Mounts:** If local storage is required, mount the upload disk partition with the `noexec` flag at the operating system level, ensuring no files inside that folder can be executed by the OS.

### D. Automated Anti-Virus Scanning
Before finalizing an upload, stream the file's byte stream to an anti-virus daemon (like **ClamAV**) or a third-party scanning API. If infected, instantly abort the write and discard the buffer.

---

## Public URL Security & Asset Protection (CDN Gateways)

Exposing raw, public S3 bucket URLs to the internet is highly insecure. If bucket contents are public, attackers can scrape all files, and malicious users can consume massive bucket bandwidth, spiking cloud costs.

### Production Solution: CDN Origin Access Control (OAC)

```
  Client Browser ───► [CDN (CloudFront)] ───► (OAC Cache Hit / Decrypt) ───► [Private S3 Bucket]
                             │                                                     ▲
                  (Verifies Signed Cookie / URL)                                   │
                             │                                                     │
                             └─────────── Blocks Direct Public Access ─────────────┘
```

1. **Private Bucket Policy:** Configure the S3 bucket to block 100% of public internet access.
2. **CDN Gateway:** Route all media traffic through a Content Delivery Network (CDN) like **AWS CloudFront** or **Cloudflare**.
3. **Origin Access Control (OAC):** Configure CloudFront with OAC (or legacy OAI). The private S3 bucket is configured with a policy that allows read operations **only** if the request carries a cryptographic signature originating from the CDN's physical edge nodes.
4. **Secure Distribution with CDN Signed URLs or Cookies:**
   * For private user media (e.g., medical documents, invoices, premium course content), generate a **CloudFront Signed Cookie** or **CloudFront Signed URL** on the backend.
   * Edge nodes verify the cryptographic signature of the cookie/URL. If valid, the CDN serves the file from its cache or pulls it from S3. Direct calls to S3 from the public internet are instantly blocked (HTTP 403).

---

## Interview Questions & Answers

### Q1: What is "MIME Sniffing", and how do you protect against a browser executing XSS through an uploaded image?
- **Answer:** MIME Sniffing is a browser mechanism where the browser ignores the server's `Content-Type` header and inspects the raw binary bytes of a file to guess its format. If an attacker uploads an HTML file with malicious Javascript but names it `image.jpg`, a browser performing MIME sniffing will read the HTML tags, render it as HTML, and execute the Javascript, executing an XSS exploit on the domain.
- **Protection:** 
  1. Always set the `X-Content-Type-Options: nosniff` header on all file delivery responses, forcing browsers to respect the declared `Content-Type` strictly.
  2. Serve uploaded files from a completely **isolated sandbox domain** (e.g., `https://my-cdn.com`) separate from the main application domain (`https://app.com`), isolating cookie contexts.
  3. Force download of sensitive files (like PDF or attachments) by sending the `Content-Disposition: attachment; filename=...` header.
