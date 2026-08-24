# Cross-Site Scripting (XSS)

XSS is a vulnerability where an attacker injects malicious client-side scripts (typically JavaScript) into trusted websites, which are then executed by unsuspecting users' browsers.

---

## 1. Types of XSS

XSS is categorized into three primary types based on how the malicious script is delivered and executed:

```
                  ┌───► Stored (Persistent) -> DB to target
                  │
XSS Vulnerability ├───► Reflected (Non-persistent) -> Query params/payload
                  │
                  └───► DOM-based -> Client-side execution sink
```

### 1. Stored XSS (Persistent XSS)
* **How it works:** Malicious input is saved directly on the target server (e.g., in a database, comment section, or forum post).
* **Delivery:** When other users visit the page containing the stored content, the malicious script is rendered in their browsers and executed.
* **Severity:** High. Affects any user who views the corrupted resource.

### 2. Reflected XSS (Non-persistent XSS)
* **How it works:** Malicious payload is part of the request sent to the server (typically as query parameters or form submissions).
* **Delivery:** The server immediately echoes (reflects) this input in the HTML response without proper sanitization.
* **Mechanism:** Attackers lure targets into clicking a specially crafted URL (e.g., `https://example.com/search?q=<script>...</script>`).

### 3. DOM-based XSS
* **How it works:** The vulnerability exists entirely on the client side. The server is not necessarily involved.
* **Delivery:** Client-side JavaScript reads data from an untrusted source (like `window.location.hash` or `document.referrer`) and passes it insecurely to a sink that executes code (e.g., `element.innerHTML = payload` or `eval()`).

---

## 2. Prevention & Mitigation Best Practices

XSS defense must be applied multi-layered:

```
                    HTML ESCAPING VS. RICH SANITIZATION
       
CONTEXT-AWARE HTML ESCAPING (Strict plain-text safety)
User Input ───> [Convert `<` to `&lt;`, `>` to `&gt;`] ───> Renders safely as text only
(e.g., `<script>` displays as flat, non-executable letters: &lt;script&gt;)

RICH HTML SANITIZATION (Permitting secure formatting, e.g. blog posts)
User Input ───> [Parse HTML to in-memory DOM Tree] ───> [Scrub bad nodes / attributes] 
                 - Keeps safe: `<b>`, `<i>`, `<p>`       - Expunges: `<script>`, `onerror=`
                                                      │
                                                      v
                                           Safe rich HTML returned
```

### 1. Context-Aware Output Encoding (Primary Defense)
Never insert user input directly into HTML without escaping. Use specific escaping based on where the variable is placed:
* **HTML Body:** Convert `<` to `&lt;`, `>` to `&gt;`, `&` to `&amp;`.
* **HTML Attributes:** Escape quotes (`"` and `'`) to prevent breaking out of attribute constraints.
* **JavaScript Variables:** Use JSON serialization or strict encoding when passing backend variables to client scripts.

### 2. Secure Input Sanitization (When HTML is Allowed)
If users must input rich text (e.g., a blog editor), use a strict HTML parser to sanitize inputs:
* Use verified libraries: **DOMPurify** (frontend) or **bluemonday** (Go backend).
* Strip out blacklisted tags/attributes: `<script>`, `<iframe>`, `onload=`, `onerror=`.

### 3. HTTP-Only Cookie Flags
Keep session identifiers and JWTs out of reach of client-side scripts.
* Use `HttpOnly` flag on cookies. This prevents JavaScript from accessing them via `document.cookie`, making session hijacking via XSS significantly harder.

### 4. Content Security Policy (CSP)
A browser-enforced header that restricts which scripts, styles, and connections can load or execute:
```http
Content-Security-Policy: default-src 'self'; script-src 'self' https://trustedscripts.com;
```
* Effectively blocks execution of inline scripts (e.g., `<script>alert(1)</script>`) and untrusted external CDNs.

---

## 3. Popular Interview Questions & High-Impact Answers

### Q1: What is the difference between HTML escaping and HTML sanitization?
* **Answer:** HTML **escaping** converts special characters into their safe HTML entity representations (e.g., turning `<` into `&lt;`), rendering them as safe plain text in the browser. HTML **sanitization** parses active HTML, stripping out unsafe tags, elements, and event handlers (like `<script>` or `onclick`) while preserving safe styling tags (like `<b>`, `<i>`, or `<p>`) so rich text can be displayed safely.

### Q2: How does HttpOnly cookie flag prevent session hijacking via XSS?
* **Answer:** If an attacker executes an arbitrary XSS script, their script can access all cookies stored in `document.cookie` and exfiltrate them to an external server. Setting the `HttpOnly` flag instructs the browser to block all client-side JavaScript access to that cookie. The browser will still attach the cookie to subsequent network requests, but scripts cannot read or copy it, neutralizing the impact of XSS on session theft.

### Q3: What is DOM-based XSS, and how is it different from Reflected XSS?
* **Answer:** In Reflected XSS, the payload travels to the backend server and is sent back inside the server's HTML response. In DOM-based XSS, the payload never reaches the server; it is consumed entirely on the client side. JavaScript reads the payload from a client source (e.g., URL hash) and updates the DOM using an unsafe sink (e.g., `innerHTML`), making it a pure frontend scripting vulnerability.
