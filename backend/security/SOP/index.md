# Same-Origin Policy (SOP)

Same-Origin Policy is the foundational security model of web browsers, designed to isolate potentially malicious documents.

---

## 1. Meaning & Core Concepts

### What is an "Origin"?
An origin is defined by the unique combination of three components:
1. **Protocol (Scheme):** `http`, `https`
2. **Domain (Host):** `example.com`, `api.example.com`
3. **Port:** `80`, `443`, `8080`

If **any** of these three parts differ, the origins are considered cross-origin.

### Same-Origin Comparison Table (Target: `https://example.com:443`)

| Target URL | Origin Match? | Reason |
| :--- | :---: | :--- |
| `https://example.com/about` | **Yes** | Same protocol, host, and port. |
| `https://example.com:8080/` | **No** | Different port (`8080` vs `443`). |
| `http://example.com/` | **No** | Different protocol (`http` vs `https`). |
| `https://api.example.com/` | **No** | Different subdomain/host. |

### Why SOP is Crucial
Without SOP, a malicious website running in one browser tab could execute JavaScript to read sensitive data (like account dashboards or private emails) from another tab where the user is logged into their bank or email account. SOP ensures scripts running on one origin cannot access or manipulate the DOM/data of another origin.

---

## 2. Exceptions & Limits to SOP

SOP does **not** block all cross-origin interactions. Browsers categorize resource access into three groups:

```
                 SOP CROSS-ORIGIN REQUEST PERMISSION MATRIX
       
    Request Type        | Browser Action  | Example / Mechanics
  ──────────────────────┼─────────────────┼─────────────────────────────────────────
  1. WRITES             |  ALLOWED        | - Form Submissions (HTML <form>)
     (State changes)    |                 | - Page Link Redirections (<a href>)
                        |                 | * Creates the vulnerability vector for CSRF!
  ──────────────────────┼─────────────────┼─────────────────────────────────────────
  2. EMBEDDINGS         |  ALLOWED        | - Script loading (<script src="...">)
     (Asset loading)    |                 | - Media rendering (<img src="...">)
                        |                 | - Frame containment (<iframe>)
  ──────────────────────┼─────────────────┼─────────────────────────────────────────
  3. READS              |  BLOCKED        | - Fetch / XHR response body queries
     (Data retrieval)   |                 | - Reading DOM nodes inside iframes
                        |                 | * Requires server CORS header override!
```

1. **Cross-Origin Writes:** Generally **allowed**. Links, redirects, and form submissions to another origin do not violate SOP.
2. **Cross-Origin Embedding:** Generally **allowed**.
   * `<script src="..."></script>` (JavaScript)
   * `<link rel="stylesheet" href="...">` (CSS)
   * `<img>`, `<video>`, `<audio>` (Media)
   * `<iframe>` (Embedded documents, though framing can be blocked using security headers like `X-Frame-Options` or CSP `frame-ancestors`).
3. **Cross-Origin Reads:** Strictly **blocked**. A script from origin A cannot read the response body of a network request (fetch/XHR) to origin B, nor can it read the DOM contents of an iframe loading origin B, unless origin B explicitly permits it (via CORS).

---

## 3. Popular Interview Questions & High-Impact Answers

### Q1: Does the Same-Origin Policy block requests from being sent, or does it block the response from being read?
* **Answer:** SOP (and CORS) blocks the **response from being read** by the client-side JavaScript. The browser *does* send the network request, and the server *does* process and return a response. However, the browser intercepts the response, checks the security policy, and blocks the script from reading any data if origins do not match.

### Q2: How can Origin A safely share data or communicate with Origin B client-side?
* **Answer:** There are two standard approaches:
  1. **PostMessage API:** Allows windows/iframes from different origins to pass text messages to each other securely using `window.postMessage()`. The receiver must validate the incoming event's `origin` property before processing.
  2. **CORS (Cross-Origin Resource Sharing):** A server-side mechanism that instructs the browser to bypass SOP reads for specified origins.

### Q3: What is the relation between SOP and Cookies?
* **Answer:** SOP does not directly govern cookie access. Cookies are separated by domain and path, not protocol or port. However, modern cookie flags like `SameSite` and `Secure` are used to align cookie behavior with origin security, preventing CSRF attacks by limiting cross-origin cookie transmission.
