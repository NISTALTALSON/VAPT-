# HTTP, HTTPS & Web Security — Deep Dive (Mastery Level)
### Companion Document to "The Complete Networking Guide"

---

## Table of Contents
1. What Is HTTP? (The Big Picture)
2. HTTP Request/Response Structure
3. HTTP Methods (Verbs) — Full Mastery
4. HTTP Status Codes — Full Mastery
5. HTTP Headers — Full Mastery
6. Cookies — Deep Dive
7. Sessions — Deep Dive
8. Cookies vs Sessions vs Tokens
9. JWT (JSON Web Tokens) — Deep Dive
10. Caching — Deep Dive
11. SSL & TLS — Deep Dive
12. The TLS Handshake (Full Detail)
13. HTTPS Certificates — How They Actually Work
14. Certificate Authorities & Chain of Trust
15. Common Web Security Concepts
16. Quick-Reference Cheat Sheet

---

## 1. What Is HTTP? (The Big Picture)

**HTTP (HyperText Transfer Protocol)** is the application-layer protocol that powers the web — it defines how clients (browsers, apps) and servers exchange data: requesting web pages, submitting forms, loading images, calling APIs.

### Key Characteristics
- **Client-server model:** The client (browser) initiates a request; the server responds.
- **Stateless:** Each HTTP request is independent — the server doesn't inherently "remember" previous requests from the same client. (This is *why* cookies/sessions/tokens exist — to work around statelessness.)
- **Text-based (originally):** HTTP/1.1 messages are human-readable text. Modern HTTP/2 and HTTP/3 use binary framing for efficiency, but the conceptual structure (methods, headers, status codes) remains the same.
- **Runs over TCP** (HTTP/1.1, HTTP/2) or **over QUIC/UDP** (HTTP/3 — designed to reduce latency).
- Default port **80** (HTTP) or **443** (HTTPS — HTTP encrypted via TLS).

---

## 2. HTTP Request/Response Structure

### Anatomy of an HTTP Request
```
GET /products?id=42 HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0
Accept: text/html
Cookie: sessionid=abc123

(optional body — used in POST/PUT/PATCH)
```

| Part | Description |
|---|---|
| Method | The action requested (GET, POST, etc.) |
| Path | The resource being requested (`/products?id=42`) |
| HTTP Version | HTTP/1.1, HTTP/2, HTTP/3 |
| Headers | Metadata about the request (see Section 5) |
| Body | Optional data sent with the request (form data, JSON, etc.) |

### Anatomy of an HTTP Response
```
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 1256
Set-Cookie: sessionid=abc123; HttpOnly; Secure

<html>...</html>
```

| Part | Description |
|---|---|
| Status Line | HTTP version + status code + status text |
| Headers | Metadata about the response |
| Body | The actual content returned (HTML, JSON, image, etc.) |

---

## 3. HTTP Methods (Verbs) — Full Mastery

| Method | Purpose | Safe? | Idempotent? | Has Body? |
|---|---|---|---|---|
| **GET** | Retrieve a resource | Yes | Yes | No (shouldn't) |
| **POST** | Create a new resource / submit data | No | No | Yes |
| **PUT** | Replace/update a resource entirely | No | Yes | Yes |
| **PATCH** | Partially update a resource | No | No* | Yes |
| **DELETE** | Remove a resource | No | Yes | Usually no |
| **HEAD** | Same as GET but returns only headers, no body | Yes | Yes | No |
| **OPTIONS** | Asks what methods/operations are supported on a resource | Yes | Yes | No |
| **CONNECT** | Establishes a tunnel (used for HTTPS proxying) | No | No | No |
| **TRACE** | Echoes back the received request (diagnostic, rarely used, security risk) | Yes | Yes | No |

*"Safe" = doesn't modify server state; "Idempotent" = calling it multiple times has the same effect as calling it once.

### Deep-Dive on the Core 5

**GET**
- Requests data **without** modifying anything on the server.
- Parameters are passed in the URL (query string: `?id=42&sort=asc`).
- Can be cached, bookmarked, and stays in browser history — **never** put sensitive data (passwords, tokens) in a GET URL.
- Example: Loading a webpage, fetching a list of products.

**POST**
- Sends data to the server to **create** a new resource or trigger an action (submit a form, upload a file, log in).
- Data goes in the **request body**, not the URL — more suitable for large or sensitive data (though still needs HTTPS for actual security).
- Not idempotent: submitting the same POST twice (e.g., double-clicking "Submit Order") can create two separate resources — this is why forms often disable the submit button after one click.

**PUT**
- Replaces an **entire** existing resource with the data provided. If you PUT a partial object, whatever fields you omit may be wiped/reset (because it's a *full* replacement).
- **Idempotent:** sending the same PUT request 5 times results in the same end state as sending it once.
- Example: `PUT /users/42` with a full user object replaces user 42 entirely.

**PATCH**
- Updates **only the specified fields** of a resource, leaving the rest untouched.
- Example: `PATCH /users/42` with `{"email": "new@example.com"}` updates just the email, nothing else.
- Not guaranteed idempotent in all implementations (e.g., a PATCH that says "increment counter by 1" gives a different result each time it's called), though many APIs implement it idempotently in practice.

**DELETE**
- Removes the specified resource.
- Idempotent in principle: deleting an already-deleted resource should still result in "it's gone" (though the server might return a 404 on the second call — the *end state* is the same).

### GET vs POST — The Classic Interview Question
| Aspect | GET | POST |
|---|---|---|
| Data location | URL query string | Request body |
| Visibility | Visible in URL, browser history, logs | Hidden in body |
| Bookmarkable | Yes | No |
| Cacheable | Yes (by default) | Not by default |
| Data limit | Limited by URL length (~2000 chars typical) | Effectively unlimited |
| Use case | Fetching/reading data | Submitting/creating data |

---

## 4. HTTP Status Codes — Full Mastery

Status codes are grouped into 5 classes by their first digit:

| Class | Category | Meaning |
|---|---|---|
| **1xx** | Informational | Request received, still processing |
| **2xx** | Success | Request succeeded |
| **3xx** | Redirection | Further action needed to complete the request |
| **4xx** | Client Error | The request has an error (client's fault) |
| **5xx** | Server Error | The server failed to fulfill a valid request (server's fault) |

### Must-Know Codes by Category

**1xx — Informational**
| Code | Meaning |
|---|---|
| 100 | Continue — initial part received, client should continue |
| 101 | Switching Protocols (e.g., upgrading to WebSocket) |

**2xx — Success**
| Code | Meaning |
|---|---|
| **200** | OK — standard success response |
| **201** | Created — resource successfully created (typical POST response) |
| **204** | No Content — success, but nothing to return in the body (common for DELETE) |

**3xx — Redirection**
| Code | Meaning |
|---|---|
| **301** | Moved Permanently — resource has a new permanent URL (browsers/search engines update their records) |
| **302** | Found (Temporary Redirect) — resource temporarily at a different URL |
| **304** | Not Modified — cached version is still valid, no need to re-download (used with caching, see Section 10) |
| 307 | Temporary Redirect (method/body must NOT change, unlike 302 in older implementations) |

**4xx — Client Errors**
| Code | Meaning |
|---|---|
| **400** | Bad Request — malformed request syntax |
| **401** | Unauthorized — authentication required/failed (you're not logged in, or credentials invalid) |
| **403** | Forbidden — you ARE authenticated, but don't have permission for this resource |
| **404** | Not Found — resource doesn't exist |
| 405 | Method Not Allowed — e.g., trying POST on a GET-only endpoint |
| **409** | Conflict — request conflicts with current server state (e.g., duplicate entry) |
| 415 | Unsupported Media Type |
| **429** | Too Many Requests — rate limiting triggered |

**5xx — Server Errors**
| Code | Meaning |
|---|---|
| **500** | Internal Server Error — generic catch-all server failure |
| 501 | Not Implemented |
| **502** | Bad Gateway — a server acting as a gateway/proxy got an invalid response from an upstream server |
| **503** | Service Unavailable — server temporarily overloaded or down for maintenance |
| **504** | Gateway Timeout — upstream server didn't respond in time |

### 401 vs 403 — The Classic Confusion
- **401 Unauthorized** = "I don't know who you are" (not authenticated — log in first)
- **403 Forbidden** = "I know who you are, but you're not allowed here" (authenticated, but lacking permission)

---

## 5. HTTP Headers — Full Mastery

Headers carry metadata about the request or response — they don't contain the actual "content," but describe *how* to handle it.

### Common Request Headers
| Header | Purpose |
|---|---|
| `Host` | Which domain is being requested (required in HTTP/1.1 — allows multiple sites on one IP) |
| `User-Agent` | Identifies the client software (browser/OS) making the request |
| `Accept` | What content types the client can handle (`text/html`, `application/json`) |
| `Accept-Language` | Preferred language(s) |
| `Authorization` | Credentials for authentication (e.g., `Bearer <token>`, Basic Auth) |
| `Cookie` | Sends previously stored cookies back to the server |
| `Content-Type` | The format of the data in the request body (`application/json`, `multipart/form-data`) |
| `Content-Length` | Size of the request body in bytes |
| `Referer` | The URL of the page that linked to the current request |
| `If-None-Match` / `If-Modified-Since` | Used for cache validation (see Section 10) |

### Common Response Headers
| Header | Purpose |
|---|---|
| `Content-Type` | Format of the returned data (`text/html`, `application/json`, `image/png`) |
| `Content-Length` | Size of the response body |
| `Set-Cookie` | Instructs the browser to store a cookie |
| `Cache-Control` | Caching rules/directives (see Section 10) |
| `ETag` | A version identifier for a resource, used for cache validation |
| `Location` | Used with 3xx redirects — tells the client where to go |
| `Access-Control-Allow-Origin` | CORS header — controls which origins can access this resource cross-domain |
| `Strict-Transport-Security` (HSTS) | Forces browsers to only connect via HTTPS from now on |
| `X-Content-Type-Options` | Security header preventing MIME-type sniffing attacks |
| `Content-Security-Policy` | Restricts what sources scripts/styles/etc. can load from (anti-XSS) |

---

## 6. Cookies — Deep Dive

A **cookie** is a small piece of data that a server sends to a browser (via the `Set-Cookie` response header), which the browser then **stores and automatically sends back** on every subsequent request to that same domain.

Cookies exist to solve HTTP's fundamental statelessness — they let a server "remember" something about the client across multiple otherwise-independent requests.

### Cookie Attributes
```
Set-Cookie: sessionid=abc123; Domain=example.com; Path=/; 
            Expires=Wed, 01 Jan 2025 00:00:00 GMT; 
            Secure; HttpOnly; SameSite=Strict
```

| Attribute | Purpose |
|---|---|
| `Domain` | Which domain(s) the cookie applies to |
| `Path` | Restricts the cookie to specific URL paths |
| `Expires` / `Max-Age` | When the cookie should be deleted (omit both = "session cookie," deleted when browser closes) |
| `Secure` | Cookie is only sent over HTTPS connections, never plain HTTP |
| `HttpOnly` | Cookie cannot be accessed via JavaScript (`document.cookie`) — protects against XSS attacks stealing it |
| `SameSite` | Controls whether the cookie is sent on cross-site requests (`Strict`, `Lax`, or `None`) — a key defense against CSRF attacks |

### Types of Cookies
| Type | Description |
|---|---|
| **Session Cookie** | No expiration set — deleted when the browser closes |
| **Persistent Cookie** | Has an expiration date — survives browser restarts (e.g., "remember me" logins) |
| **First-Party Cookie** | Set by the domain you're actually visiting |
| **Third-Party Cookie** | Set by a different domain than the one you're visiting (e.g., ad trackers embedded in a page) — increasingly blocked by modern browsers for privacy |

---

## 7. Sessions — Deep Dive

A **session** is server-side storage that tracks a user's state (logged in, cart contents, preferences) across multiple requests.

### How Sessions Typically Work
1. User logs in successfully.
2. Server creates a session record (stored in server memory, a database, or a cache like Redis) containing user data, and generates a unique **session ID**.
3. Server sends that session ID to the browser via a `Set-Cookie` header (e.g., `sessionid=abc123`).
4. Browser automatically includes that cookie on every future request to the site.
5. Server looks up `abc123` in its session store on each request to know "which user is this, and what's their current state."

### Key Point: The cookie itself is just a *reference* (like a claim ticket) — the actual session **data lives on the server**, not in the cookie. This is the core difference from tokens like JWT (see below), where the data itself often travels with the client.

### Session Expiration & Security
- Sessions typically expire after a period of inactivity (e.g., 30 minutes) for security.
- **Session hijacking:** If an attacker steals a valid session ID (e.g., via XSS or network sniffing on unencrypted HTTP), they can impersonate the user — this is exactly why `HttpOnly`, `Secure`, and HTTPS everywhere matter so much.

---

## 8. Cookies vs Sessions vs Tokens

| Concept | What it is | Where data lives |
|---|---|---|
| **Cookie** | The transport mechanism — a small piece of data stored by the browser and auto-sent with requests | Browser (can carry the data itself, OR just an ID) |
| **Session** | A server-side record tracking user state, referenced by a cookie containing a session ID | Server (cookie just holds a pointer/ID) |
| **Token (e.g., JWT)** | A self-contained, portable credential that carries the actual data/claims within itself, cryptographically signed | Client-side (server doesn't need to store anything) |

**The core architectural tradeoff:**
- **Session-based auth:** Server must store/manage session state for every logged-in user (more server memory/database load, but easy to instantly revoke access by deleting the session).
- **Token-based auth (JWT):** Server stores nothing — it just verifies the token's signature on each request (scales better across multiple servers/microservices, but harder to instantly revoke a single token before it naturally expires).

---

## 9. JWT (JSON Web Tokens) — Deep Dive

A **JWT** is a compact, self-contained, URL-safe token format used to securely transmit claims (pieces of information, like "this is user #42, and they're an admin") between two parties — commonly used for stateless authentication in modern APIs.

### JWT Structure — Three Parts, Separated by Dots
```
eyJhbGciOiJIUzI1NiJ9.eyJ1c2VySWQiOjQyfQ.SflKxwRJSMeKKF2QT4fw
└──────  Header  ──────┘└──── Payload ────┘└─── Signature ───┘
```

| Part | Contents | Purpose |
|---|---|---|
| **Header** | Algorithm used (e.g., `HS256`, `RS256`) and token type (`JWT`) | Describes how the token is signed |
| **Payload** | The actual **claims** — data like `userId`, `role`, `exp` (expiration), `iat` (issued at) | The information being transmitted |
| **Signature** | A cryptographic signature over the header+payload, using a secret key (or private key) | Proves the token hasn't been tampered with |

**Important:** The header and payload are just **Base64-encoded**, NOT encrypted — anyone can decode and read them (try it at jwt.io). The **signature** is what guarantees integrity/authenticity — it does not hide the content.

### How JWT Authentication Works (Typical Flow)
1. User logs in with username/password.
2. Server verifies credentials, then generates a JWT containing claims (`userId`, `role`, `exp`), and **signs it** with a secret key known only to the server.
3. Server sends the JWT back to the client (often stored in `localStorage`, a cookie, or an app's secure storage).
4. On every subsequent request, the client includes the JWT — typically in the header:
   ```
   Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
   ```
5. Server **verifies the signature** (using the same secret/public key) to confirm the token is authentic and unmodified, then trusts the claims inside without needing a database lookup.

### JWT Pros & Cons
| Pros | Cons |
|---|---|
| Stateless — no server-side session storage needed | Can't be easily revoked before expiration (server has no record of "active" tokens by default) |
| Scales well across multiple servers/microservices (any server with the secret can verify it) | Payload is readable by anyone (not encrypted) — never put secrets/passwords in it |
| Self-contained — carries its own expiration and claims | If the secret key leaks, an attacker can forge valid tokens |

### Symmetric vs Asymmetric Signing
- **HS256 (Symmetric):** Same secret key used to both sign and verify — simple, but every service that needs to verify tokens must also have the secret (risk if leaked).
- **RS256 (Asymmetric):** A private key signs the token; a public key verifies it — the public key can be shared freely without risk, since only the private key can create valid signatures. Preferred for multi-service/microservice architectures.

---

## 10. Caching — Deep Dive

**Caching** stores a copy of a resource (webpage, image, API response) so it doesn't have to be re-fetched/re-generated on every single request — dramatically improving speed and reducing server load.

### Where Caching Happens
```
Browser Cache → CDN/Proxy Cache → Server-Side Cache → Origin Server
```

### Key Caching Headers

**`Cache-Control`** — the primary directive controlling caching behavior:
| Directive | Meaning |
|---|---|
| `no-store` | Never cache this, at all |
| `no-cache` | Can cache, but MUST revalidate with the server before using the cached copy |
| `public` | Can be cached by any cache (browser, CDN, proxies) |
| `private` | Can only be cached by the end user's browser, not shared caches (like CDNs) |
| `max-age=3600` | Cache is valid for 3600 seconds before needing revalidation |

### Cache Validation (Conditional Requests)
Even when a cached copy has "expired," the client doesn't always need to re-download the entire resource — it can ask "has this changed at all?" using validators:

- **ETag:** A unique identifier (like a fingerprint/hash) for a specific version of a resource.
  - Client sends `If-None-Match: "<etag>"` on its next request.
  - If the resource hasn't changed, the server responds with **304 Not Modified** (empty body — saves bandwidth) instead of resending the full resource.
- **Last-Modified / If-Modified-Since:** Similar concept, but based on a timestamp instead of a hash.

### Caching Strategy Summary
```
First request:  Client → Server: GET /image.png
                Server → Client: 200 OK + Cache-Control: max-age=3600 + ETag: "xyz"

Within 1 hour:  Client uses local cached copy directly — no network request at all.

After 1 hour:   Client → Server: GET /image.png, If-None-Match: "xyz"
                Server → Client: 304 Not Modified (if unchanged — no body sent)
                                  OR 200 OK + new content (if it changed)
```

---

## 11. SSL & TLS — Deep Dive

### What are SSL and TLS?
**SSL (Secure Sockets Layer)** and **TLS (Transport Layer Security)** are cryptographic protocols that provide **encryption, authentication, and data integrity** for data traveling across a network.

- **SSL is obsolete/deprecated** — all versions (SSL 2.0, 3.0) have known vulnerabilities and should never be used.
- **TLS is the modern replacement** — TLS 1.0 and 1.1 are also deprecated; **TLS 1.2** is the current widely-supported baseline, and **TLS 1.3** (faster, more secure, simplified handshake) is the modern standard.
- People still colloquially say "SSL certificate" and "SSL/TLS" out of habit, even though it's actually TLS doing the work today.

### What TLS Actually Provides (The 3 Pillars)
| Property | What it means |
|---|---|
| **Encryption** | Data is scrambled so eavesdroppers on the network can't read it |
| **Authentication** | Verifies you're actually talking to the real server you intended (via certificates), not an impersonator |
| **Integrity** | Guarantees data wasn't altered/tampered with in transit |

### Where TLS Sits
TLS operates **between** the Transport layer (TCP) and the Application layer (HTTP) — this is literally what turns HTTP into HTTPS: the same HTTP protocol, just wrapped in an encrypted TLS tunnel.
```
HTTP  →  (wrapped in)  →  TLS  →  (running over)  →  TCP  →  IP
```

---

## 12. The TLS Handshake (Full Detail)

Before any encrypted application data (like your HTTP request) can be sent, the client and server must perform a **TLS handshake** to agree on encryption parameters and verify identity. This happens **after** the TCP 3-way handshake, and **before** any HTTP data is exchanged.

### TLS 1.2 Handshake (Classic, More Steps)
```
Client                                        Server
  |---- ClientHello ------------------------->|
  |     (supported TLS versions, cipher       |
  |      suites, random number)               |
  |                                            |
  |<--- ServerHello ---------------------------|
  |     (chosen cipher suite, random number)   |
  |<--- Certificate ---------------------------|
  |     (server's SSL/TLS certificate)         |
  |<--- ServerHelloDone ------------------------|
  |                                            |
  |---- Client Key Exchange ------------------->|
  |     (encrypted pre-master secret,          |
  |      encrypted using server's public key)  |
  |---- Change Cipher Spec -------------------->|
  |---- Finished (encrypted) ------------------>|
  |                                            |
  |<--- Change Cipher Spec ---------------------|
  |<--- Finished (encrypted) --------------------|
  |                                            |
  |======== Encrypted Application Data =========|
```

### Simplified Conceptual Steps
1. **ClientHello:** Client says "here's what TLS versions and encryption algorithms (cipher suites) I support, and here's a random number."
2. **ServerHello + Certificate:** Server picks a cipher suite, sends back its own random number, and presents its **digital certificate** (proving its identity, containing its public key).
3. **Certificate Validation:** The client checks the certificate against trusted Certificate Authorities (see Section 14) — verifying it's legitimate, not expired, and matches the domain being visited.
4. **Key Exchange:** Client and server use asymmetric cryptography (the server's public/private key pair) to securely agree on a **shared secret** (session key) — without ever transmitting that secret in a readable form over the network.
5. **Symmetric Encryption Begins:** From this point on, both sides use the shared session key with fast **symmetric encryption** (like AES) to encrypt/decrypt all actual application data (your HTTP requests/responses).

### Why switch to symmetric encryption after the handshake?
- **Asymmetric encryption** (public/private key pairs) is secure but computationally expensive/slow — great for the initial handshake to safely exchange a secret.
- **Symmetric encryption** (same key on both sides) is much faster — ideal for encrypting the actual bulk data flow of a session.
- This "best of both worlds" approach (asymmetric for key exchange, symmetric for bulk data) is standard practice in virtually all modern secure communication.

### TLS 1.3 — What Changed
- Reduces the handshake to essentially **one round trip** (vs two in TLS 1.2), significantly reducing latency before encrypted data can start flowing.
- Removes support for outdated, insecure cipher suites and algorithms entirely (no more downgrade attack surface).
- Supports **"0-RTT" (Zero Round Trip Time) resumption** for returning visitors reconnecting to a previously-visited server, allowing data to be sent immediately, even faster (with some tradeoffs around replay-attack risk for certain use cases).

---

## 13. HTTPS Certificates — How They Actually Work

An **SSL/TLS certificate** is a digital document that cryptographically proves a server's identity and contains the **public key** used in the TLS handshake.

### What's Inside a Certificate
| Field | Purpose |
|---|---|
| Subject | The domain name(s) the certificate is valid for (e.g., `example.com`) |
| Issuer | The Certificate Authority (CA) that issued/signed this certificate |
| Public Key | The server's public key, used during the TLS handshake key exchange |
| Validity Period | Start and expiration dates |
| Digital Signature | The CA's cryptographic signature over the certificate, proving it's genuine and unaltered |

### Public/Private Key Pair (The Foundation)
- The server holds a **private key** — kept absolutely secret, never shared.
- The certificate publishes the corresponding **public key** — freely shared with anyone.
- Data encrypted with the public key can **only** be decrypted with the matching private key (and vice versa for signing) — this asymmetric relationship is the mathematical foundation that makes the whole system work.

### Why You Can Trust a Certificate
Anyone could generate a certificate claiming to be "google.com" — so what stops an attacker from doing that? The answer is the **chain of trust**, covered next.

---

## 14. Certificate Authorities & Chain of Trust

### Certificate Authority (CA)
A trusted third-party organization (e.g., Let's Encrypt, DigiCert, Sectigo) that verifies a domain owner's identity and then digitally **signs** their certificate — vouching "yes, I verified this, this certificate genuinely belongs to this domain."

### The Chain of Trust
```
Root CA Certificate  (self-signed, pre-installed & trusted by OS/browser)
        |
        | signs
        v
Intermediate CA Certificate
        |
        | signs
        v
Your Website's Certificate (example.com)
```

- Your browser/OS comes pre-loaded with a "trust store" of Root CA certificates from well-known, audited authorities.
- When your browser receives a website's certificate, it walks **up the chain**: checks that the site's cert was signed by an Intermediate CA, and that Intermediate CA's cert was signed by a Root CA that the browser already inherently trusts.
- If that unbroken chain checks out — signatures valid, not expired, domain matches, not revoked — the browser trusts the connection and shows the padlock icon.
- Root CAs are kept extremely secure and rarely used directly — they instead delegate signing authority to Intermediate CAs, limiting exposure if an intermediate is ever compromised (it can be revoked without invalidating the entire root of trust).

### Certificate Validation Levels
| Type | Verification Level | What it proves |
|---|---|---|
| **DV (Domain Validated)** | Just proves control of the domain (e.g., via DNS TXT record or email) | Domain ownership only — fast, often free (e.g., Let's Encrypt) |
| **OV (Organization Validated)** | CA verifies the actual business/organization exists | Domain + organization identity |
| **EV (Extended Validation)** | Rigorous legal/business verification process | Highest assurance — used to show a strong verified business identity |

### The Full HTTPS Connection Process (Putting It All Together)
1. Browser initiates TCP connection to server (3-way handshake) on port 443.
2. TLS handshake begins — server presents its certificate.
3. Browser validates the certificate chain up to a trusted Root CA.
4. Browser and server negotiate a shared symmetric session key via the handshake.
5. All subsequent HTTP data (your actual request/response) is encrypted using that session key.
6. Padlock icon appears — connection is authenticated, encrypted, and tamper-proof.

### Certificate Revocation
If a private key is compromised, or a certificate was mis-issued, it needs to be invalidated before its natural expiration:
- **CRL (Certificate Revocation List):** A published list of revoked certificate serial numbers that clients can check.
- **OCSP (Online Certificate Status Protocol):** A real-time query mechanism to check if a specific certificate has been revoked, without downloading an entire list.

---

## 15. Common Web Security Concepts

| Concept | Description |
|---|---|
| **XSS (Cross-Site Scripting)** | Injecting malicious scripts into a trusted website, which then execute in other users' browsers — mitigated by input sanitization, `HttpOnly` cookies, and Content-Security-Policy headers |
| **CSRF (Cross-Site Request Forgery)** | Tricking a logged-in user's browser into making an unwanted request to a site they're authenticated on — mitigated by CSRF tokens and `SameSite` cookie attributes |
| **CORS (Cross-Origin Resource Sharing)** | A browser security mechanism controlling whether a webpage from one origin (domain) can make requests to a different origin — governed by the `Access-Control-Allow-Origin` header |
| **HSTS (HTTP Strict Transport Security)** | A response header telling browsers "always use HTTPS for this site from now on," preventing downgrade attacks |
| **MITM (Man-in-the-Middle)** | An attacker intercepts communication between client and server — this is precisely what TLS/HTTPS is designed to prevent, via encryption + certificate authentication |

---

## 16. Quick-Reference Cheat Sheet

```
HTTP METHODS
  GET     → Read data           (safe, idempotent, no body)
  POST    → Create data         (not idempotent, has body)
  PUT     → Replace entirely    (idempotent, has body)
  PATCH   → Partial update      (has body)
  DELETE  → Remove data         (idempotent)

STATUS CODES
  2xx → Success       (200 OK, 201 Created, 204 No Content)
  3xx → Redirection    (301 Permanent, 302 Temporary, 304 Not Modified)
  4xx → Client Error   (400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found)
  5xx → Server Error   (500 Internal Error, 502 Bad Gateway, 503 Unavailable)

AUTH STORAGE COMPARISON
  Cookie   → Transport mechanism (auto-sent by browser)
  Session  → Server-side state, referenced by a cookie ID
  JWT      → Self-contained signed token, client carries the data

CACHING
  Cache-Control: max-age=3600  → cache for 1 hour
  ETag / If-None-Match          → validate before re-downloading
  304 Not Modified              → cached copy still valid

TLS HANDSHAKE (simplified)
  ClientHello → ServerHello+Certificate → Key Exchange → 
  Symmetric session key established → Encrypted data flows

CERTIFICATE CHAIN OF TRUST
  Root CA (pre-trusted) → Intermediate CA → Your Site's Certificate
```

**Key fact to remember:** SSL is dead — it's all TLS now (currently TLS 1.2/1.3), but the industry still calls certificates "SSL certificates" out of habit. And remember: TLS encrypts the connection, but it's the **certificate + Certificate Authority chain of trust** that proves *who* you're actually encrypting your connection with — encryption alone doesn't stop you from securely talking to an imposter.
