# APIShield Code Audit & Documentation Report

This document provides a comprehensive audit of the APIShield codebase, covering its architecture, security profile, performance, and recommendations for improvement.

## 1. Architectural Overview

APIShield is a security scanner for APIs that follows a **Pipe and Filter** architectural pattern. It decouples the source of API definitions (OpenAPI, Postman, HAR, or Live URLs) from the security analysis logic.

### Data Flow
1.  **Ingestion:** `index.js` detects the input type and invokes the corresponding parser in `lib/parsers/`.
2.  **Normalization:** Parsers convert the input into a standardized internal representation (based on OpenAPI 3.0).
3.  **Scanning:** `lib/normalizer.js` (`scanSpec`) iterates through the normalized definition to identify vulnerabilities.
4.  **Reporting:** `lib/reporters/threatModel.js` maps findings to the STRIDE threat model and OWASP API Top 10.

---

## 2. Module Documentation

### Core Modules

| Module | Purpose | Key Functions |
| :--- | :--- | :--- |
| `index.js` | CLI Entry point & orchestration. | `main()`, `detectInputType()` |
| `lib/normalizer.js` | Core scanning engine and sensitive data patterns. | `scanSpec()`, `normalizeSpec()`, `findSensitiveFields()` |
| `lib/config.js` | Configuration management and merging. | `loadConfig()` |
| `lib/reporters/threatModel.js` | Risk reporting and STRIDE mapping. | `generateThreatModel()` |

### Parsers (`lib/parsers/`)

*   **`openapi.js`**: Standardizes existing OpenAPI specifications.
*   **`postman.js`**: Extracts endpoints and schemas from Postman Collections.
*   **`har.js`**: Infers API structure and data types from HTTP Archive files.
*   **`live.js`**: Probes live endpoints and performs dynamic schema inference from responses.

---

## 3. Security Audit

### Vulnerabilities

| Risk Level | Issue | Description | Location |
| :--- | :--- | :--- | :--- |
| **Medium** | ReDoS (Regex Denial of Service) | Glob-to-regex conversion for path ignoring can be exploited with malicious patterns. | `lib/config.js` / path matching logic |
| **Low/Info** | SSRF (Server-Side Request Forgery) | `scanLiveURL` fetches arbitrary user-provided URLs. | `lib/parsers/live.js` |
| **Low** | Incomplete Auth Detection | Static parsers only look for standard headers like `Authorization` or `X-API-Key`. | `lib/parsers/har.js`, `postman.js` |

---

## 4. Performance & Complexity

### Bottlenecks
*   **Sequential Live Probing**: `lib/parsers/live.js` uses a hardcoded 800ms sleep between requests. This makes scanning large APIs unnecessarily slow.
*   **Deep Recursion**: Schema traversal for sensitive field detection lacks a depth limit, which could cause stack exhaustion on highly nested structures.

### Complexity
*   **`lib/normalizer.js`**: The `scanSpec` function acts as a "God Function," containing all validation logic. This increases the cognitive load for maintainers and makes unit testing individual rules difficult.

---

## 5. Actionable Recommendations

### Short Term (Quick Wins)
1.  **Consolidate Sensitive Data Detection**: Move the logic from `lib/parsers/live.js` into `lib/normalizer.js` to use the more comprehensive regex set (~150 patterns).
2.  **Parallelize Live Scanning**: Use a concurrency-limited worker pattern in `live.js` to speed up probing while avoiding rate limiting.

### Long Term (Architectural)
1.  **Refactor Scanners**: Decompose `scanSpec` into smaller, rule-specific modules located in the `lib/scanners/` directory.
2.  **Improve Input Matching**: Replace manual regex generation with a robust library like `minimatch` for path ignoring.
3.  **Schema Depth Limits**: Introduce a configurable maximum depth for recursive schema analysis.
