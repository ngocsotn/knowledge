# Web Frontend Security

Production-grade security and vulnerability protection for client-side applications.

## Core Web Security Vulnerabilities

### 1. Cross-Site Scripting (XSS)
- **Stored XSS:** Malicious script is permanently stored in database and rendered to other users.
- **Reflected XSS:** Script is reflected off a web server response (e.g., via query params) and executed in client browser.
- **DOM XSS:** Vulnerability sits entirely client-side, executing when user input dynamically modifies the DOM.
- **Mitigation:**
  1. Escape/Sanitize user input before rendering.
  2. Implement strict **Content Security Policy (CSP)** headers.
  3. Set `HttpOnly` on sensitive session cookies.

### 2. Cross-Site Request Forgery (CSRF)
- An attacker forces a logged-in user browser to execute state-changing HTTP requests silently.
- **Mitigation:**
  1. Enforce **`SameSite=Lax` or `SameSite=Strict`** cookie flags.
  2. Include dynamic, cryptographically signed custom request headers (like `X-CSRF-Token`).

### 3. Clickjacking
- An attacker overlaying an invisible iframe on top of a legitimate button, tricking users into clicking it.
- **Mitigation:** Configure `X-Frame-Options: DENY` or `Content-Security-Policy: frame-ancestors 'none'`.

## Interview Questions & Answers

### Q1: What is a Content Security Policy (CSP) and how does it prevent XSS?
- **Answer:** CSP is a HTTP response header (`Content-Security-Policy`) allowing site owners to explicitly restrict which sources of scripts, styles, images, and fonts the browser is allowed to load and execute. It blocks inline scripts (e.g., `<script>alert(1)</script>`) and untrusted external CDNs by default, rendering XSS injections completely inert.
