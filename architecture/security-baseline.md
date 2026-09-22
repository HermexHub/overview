<div align="center">

# 🛡️ Hermex Security Baseline
### Application Security, OWASP Mitigation & PCI-DSS Compliance

[ **English** ] &nbsp;•&nbsp; [ [Українська](security-baseline.ua.md) ] &nbsp;•&nbsp; [ [System Overview](../README.md) ]

</div>

This document details the application security (AppSec) standards, data protection compliance (PCI-DSS), and vulnerability mitigation practices (OWASP Top-10) implemented across the Hermex platform.

---

## 🔒 1. User Anti-Enumeration (OWASP / CWE-204)

- **Uniform Authentication Responses:**  
  All login and registration failures return a generic `401 Unauthorized` response with the message `Invalid email or password`.
- **Account State Confidentiality:**  
  The system never leaks whether an account exists or is disabled, preventing attackers from building dictionaries of valid customer email addresses.

---

## ⏱️ 2. Anti-Timing Attacks

- Server response latency must not reveal whether a user email exists in the database.
- **DUMMY_HASH:** If an email is not located in `gateway_db`, the authentication service executes a real `bcrypt.compare` against a precomputed valid dummy hash:
  ```typescript
  const DUMMY_HASH = '$2b$10$abcdefghijklmnopqrstuvABCDEFGHIJKLMNOPQRSTUV0123456789';
  await bcrypt.compare(password, user ? user.passwordHash : DUMMY_HASH);
  ```
  This guarantees constant-time cryptographic execution regardless of whether the user exists in the database.

---

## 🎭 3. Error Masking (CWE-209)

- **Global Exception Filter (`GlobalExceptionFilter`):**  
  In production environments, clients are strictly prevented from receiving database error messages, table schemas, SQL queries, or RPC trace dumps.
- On unhandled 5xx errors, the client receives a sanitized payload:
  ```json
  {
    "statusCode": 500,
    "message": "Internal server error",
    "timestamp": "2026-09-22T12:00:00.000Z",
    "traceId": "c3b95a8e-5b12-421b-bf8d-d3c26725ea94"
  }
  ```
- Detailed stack traces and diagnostic logs are persisted exclusively to internal server stdout with correlation via `traceId`.

---

## 💳 4. PCI-DSS Compliance (Zero Raw Card Storage)

1. **Strict CVV Exclusion:** Card verification values (`CVV` / `CVC`) are never written to disk or stored in any database.
2. **Log Redaction:** `HermexLogger` (Pino) contains an automated redaction interceptor targeting sensitive keys:
   - `password`, `refreshToken`, `cvv`, `cardNumber`, `pan`, `pin`.
   Values matching these keys are automatically replaced with `[REDACTED]`.
3. **Isolated Microfrontend (`payment-portal`):** Card data entry is physically separated from the storefront application (`website`), reducing the PCI-DSS audit boundary.

---

## 🔄 5. Stateless Token Revocation & Versioning

- **Dual-Token Scheme:** Short-lived `accessToken` (JWT, 15 minutes) paired with a secure HttpOnly Cookie `refreshToken` (7 days).
- **`token_version` Column in `UserEntity`:**  
  Rather than managing bloated database tables of active refresh tokens, the user record stores an integer `token_version`.
- During `/logout`, password reset, or token rotation, `token_version` is atomically incremented, instantly invalidating all outstanding tokens without external cache lookups.

---

## 🧱 6. API Gateway Edge Hardening

- **Helmet HTTP Headers:** Configured against MIME-sniffing (`X-Content-Type-Options: nosniff`), clickjacking (`X-Frame-Options: SAMEORIGIN`), and server identification leaks (`X-Powered-By`).
- **Distributed Correlation Tracing:** Generates a unique UUIDv4 `x-correlation-id` for every incoming HTTP request, propagating it through all downstream gRPC metadata headers and AMQP message envelopes.
