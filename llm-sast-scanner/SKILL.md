---
name: llm-sast-scanner
description: >
  General-purpose Static Application Security Testing (SAST) skill for code vulnerability analysis.
  Trigger when the user asks to: "analyze code for vulnerabilities", "review code security", "find security bugs",
  "do a SAST scan", "check for [vulnerability type] in code", "audit source code", or requests a security
  code review of any language or framework. Covers 34 vulnerability classes across web, API, auth, mobile, and logic layers.
metadata:
  version: "1.3.1"
  domain: application-security
  references: 34 vulnerability knowledge bases
---

# SAST Vulnerability Analysis

## Purpose

Systematically analyze source code for security vulnerabilities using structured Source→Sink taint tracking,
pattern matching, and vulnerability-class-specific detection heuristics. Produce actionable findings with
severity ratings, affected code locations (file + line number), and remediation guidance.

## Scope

This skill covers the following 34 vulnerability classes. Each has a dedicated reference file loaded on demand:

| Category | Vulnerabilities |
|----------|----------------|
| **Injection** | SQL Injection, XSS, SSTI, NoSQL Injection, GraphQL Injection, XXE, RCE / Command Injection, Expression Language Injection |
| **Access Control & Auth** | IDOR, Privilege Escalation, Authentication/JWT, Default Credentials, Brute Force, Business Logic, HTTP Method Tampering, Verification Code Abuse, Session Fixation |
| **Data Exposure & Crypto** | Weak Crypto/Hash, Information Disclosure, Insecure Cookie, Trust Boundary |
| **Server-Side** | SSRF, Path Traversal/LFI/RFI, Insecure Deserialization, Arbitrary File Upload, JNDI Injection, Race Conditions |
| **Protocol & Infrastructure** | CSRF, Open Redirect, HTTP Request Smuggling/Desync, Denial of Service, CVE Patterns |
| **Language/Platform** | PHP Security, Mobile Security (Android/iOS) |

---

## Workflow

### Step 1: Understand Scope

Determine:
- Target: single file, directory, API endpoint, module, or full repo
- Language(s) and framework(s) in use
- User's goal: quick scan, deep audit, specific vuln class, or full report

### Step 2: Load Relevant References

Based on the code being reviewed, load the appropriate reference files from `references/`:

```
references/sql_injection.md          — SQL / ORM injection
references/xss.md                    — Cross-site scripting
references/ssrf.md                   — Server-side request forgery
references/rce.md                    — Remote code execution
references/idor.md                   — Insecure direct object reference
references/authentication_jwt.md     — Auth flaws, JWT weaknesses
references/csrf.md                   — Cross-site request forgery
references/path_traversal_lfi_rfi.md — Path traversal, LFI/RFI
references/ssti.md                   — Server-side template injection
references/xxe.md                    — XML external entity
references/insecure_deserialization.md    — Insecure deserialization
references/arbitrary_file_upload.md      — Arbitrary file upload
references/privilege_escalation.md       — Privilege escalation
references/nosql_injection.md            — NoSQL injection
references/graphql_injection.md          — GraphQL injection
references/weak_crypto_hash.md           — Weak cryptography / hash
references/information_disclosure.md     — Information disclosure
references/insecure_cookie.md            — Insecure cookie attributes
references/open_redirect.md              — Open redirect
references/trust_boundary.md             — Trust boundary violations
references/race_conditions.md            — Race conditions / TOCTOU
references/brute_force.md                — Brute force / credential stuffing
references/default_credentials.md        — Default / hardcoded credentials
references/verification_code_abuse.md    — Verification code abuse
references/business_logic.md             — Business logic flaws
references/http_method_tamper.md         — HTTP method tampering
references/smuggling_desync.md           — HTTP request smuggling / desync
references/cve_patterns.md               — Known CVE patterns
references/expression_language_injection.md — Expression language injection (SpEL / OGNL)
references/jndi_injection.md             — JNDI injection (Log4Shell class)
references/denial_of_service.md          — Denial of service / resource exhaustion
references/php_security.md               — PHP-specific security issues
references/mobile_security.md            — Mobile security (Android / iOS)
references/session_fixation.md           — Session fixation
```

**Loading strategy:**
- For a targeted review (e.g., "check for SQL injection"), load only the relevant reference(s).
- For a full audit, load all 34 references and scan systematically.
- Always load references for the top OWASP risks even if not explicitly requested.

---

### Step 3: Analyze Code — Source→Sink Taint Tracking

For each loaded vulnerability class, perform taint analysis:

1. **Identify Sources** — User-controlled input entry points:
   - HTTP params, headers, cookies, request body
   - File uploads
   - WebSocket messages
   - Environment variables
   - Database reads of user-supplied data, deserialized objects

2. **Trace Data Flow** — Follow the data through:
   - Variable assignments, function arguments, return values
   - Framework helpers, ORM calls, template rendering
   - Cross-module/service boundaries

3. **Check Sinks** — Dangerous operations receiving tainted data:
   - Query execution (SQL, NoSQL, LDAP, XPath)
   - Shell/OS command execution
   - File system operations
   - HTTP client calls
   - Template rendering / eval / expression parsing
   - Serialization/deserialization

4. **Evaluate Sanitization** — Between source and sink, look for:
   - Input validation (allowlist vs denylist)
   - Context-appropriate encoding/escaping
   - Parameterization (prepared statements)
   - Framework-native protections

5. **Determine Preliminary Verdict**:
   - **VULN**: Taint reaches sink with no effective sanitization
   - **LIKELY VULN**: Sanitization present but bypassable per reference heuristics
   - **SAFE**: Effective sanitization or no taint path

---

### Step 4: Business Logic & Auth Analysis

Beyond taint tracking, check for:
- Missing authentication/authorization on sensitive endpoints
- Insecure state machine transitions
- Race conditions in concurrent operations
- Improper trust boundaries between components
- JWT algorithm confusion, token fixation, session issues
- Default/hardcoded credentials
- Enumeration via timing or response differences

---

### Step 5: Judge — Validity Re-Verification

Before reporting, every preliminary finding (VULN or LIKELY VULN) **must pass a Judge review**. The Judge acts as an adversarial second opinion to eliminate false positives.

For each candidate finding, answer all of the following:

#### Reachability Check
- [ ] Is the source actually user-controlled, or is it internal/trusted data?
- [ ] Is the vulnerable code path reachable from an HTTP endpoint / entry point, or is it dead code / internal-only?
- [ ] Are there upstream guards (auth middleware, input filters) that block the path before it reaches the sink?

#### Sanitization Re-Evaluation
- [ ] Is there sanitization that was missed in Step 3? (Check parent functions, middleware, framework internals)
- [ ] Is the sanitization method sufficient for this specific sink and context?
- [ ] Does the framework provide implicit protection for this pattern?

#### Exploitability Check
- [ ] Can the tainted value actually reach the sink in a form that triggers the vulnerability?
- [ ] Is exploitation conditional on a specific environment, config, or privilege level?
- [ ] For logic bugs: is the business impact real, or hypothetical?
- [ ] Is the chosen tag the most precise valid label for this finding?

#### Judge Verdict

| Verdict | Meaning | Action |
|---------|---------|--------|
| **CONFIRMED** | All reachability/sanitization/exploitability checks pass | Include in report |
| **LIKELY** | Most checks pass; one uncertainty remains | Include in report, flag uncertainty |
| **NEEDS CONTEXT** | Cannot determine without runtime behavior / config / additional files | Note as "unverifiable without X" |
| **FALSE POSITIVE** | A check definitively fails | Drop silently |

**Only CONFIRMED and LIKELY findings are reported.**

#### Judge Output Format (internal, before reporting)

```
Finding: VULN-NNN — <class>
Reachability:   PASS / FAIL / UNCERTAIN — <reason>
Sanitization:   PASS / FAIL / UNCERTAIN — <reason>
Exploitability: PASS / FAIL / UNCERTAIN — <reason>
Judge Verdict:  CONFIRMED / LIKELY / NEEDS CONTEXT / FALSE POSITIVE
```

#### False Positive Guardrails

- Do not emit `default_credentials` unless there is a reachable authentication path that accepts a hardcoded, documented, factory, or seeded credential pair.
- Do not emit `weak_crypto_hash` when the evidence only shows a vulnerable third-party component or a generic cryptography import. Require direct use of weak hashes, broken password storage, or unsafe signing.
- **Tag clarification**: `weak_crypto_hash` is the canonical tag for both weak cryptographic algorithms (DES, RC4, ECB mode) and weak hash functions (MD5, SHA-1 for passwords). Do not use `weak_crypto` as a separate tag unless the benchmark ground truth explicitly requires it; prefer `weak_crypto_hash` to avoid tag fragmentation.
- Do not emit generic `rce` when the code shows direct shell/process execution — prefer `command_injection`.
- Do not replace `spel_injection` with `command_injection` or `rce` when the exploit primitive is Spring EL expression parsing.
- Do not emit `jndi_injection` for component demos unless the JNDI sink itself is the primary exploit path.
- Do not emit broad tags (`trust_boundary`, `authentication`, `privilege_escalation`) when a narrower precise tag is supported (`xff_spoofing`, `session_fixation`, `verification_code`).
- Do not emit `open_redirect` for infrastructure/parser misconfiguration unless attacker-controlled redirect response is the primary exploit.
- Do not emit `denial_of_service` merely because a path lacks size/rate limits — require explicit evidence that resource exhaustion is the primary impact.
- Do not emit `brute_force` merely because a login endpoint lacks visible rate limiting — rate limiting may exist at infrastructure level. Require explicit evidence of unlimited attempt processing.
- Do not emit `csrf` for stateless APIs using Bearer-token-only authentication with `SessionCreationPolicy.STATELESS`.
- Do not emit `insecure_deserialization` when the same sink is already covered by `component_vulnerability` (e.g., Fastjson autoType) unless a separate deserialization path exists.
- Do not emit `arbitrary_file_upload` for profile/avatar image upload with type restrictions and non-webroot storage.
- Do not emit `session_fixation` when Spring Security default session management is active (migrateSession is the default).
- Do not emit `information_disclosure` for database credentials in application config files — these are deployment issues, not application-level disclosure.

---

### Step 6: Report Findings

#### Severity Classification

| Severity | Criteria |
|----------|----------|
| **Critical** | Direct RCE, authentication bypass, unauthenticated data exposure |
| **High** | SQLi, SSRF, IDOR with sensitive data, stored XSS, privilege escalation |
| **Medium** | Reflected XSS, CSRF, path traversal, insecure deserialization |
| **Low** | Information disclosure, open redirect, weak crypto, insecure cookie |
| **Info** | Missing security headers, verbose errors, defense-in-depth gaps |

**Severity Downgrade Rule:** When exploitation requires authentication, specific non-default configuration, chained prerequisites, or is only reachable through an internal/admin-only path, downgrade severity by one level from the class default; LIKELY-verdict findings whose exploitability is marked UNCERTAIN must be capped at one level below the class default regardless of vulnerability type.

#### Finding Format

```
[SEVERITY] VULN-NNN — <Vulnerability Class>  [CONFIRMED | LIKELY]
File: <path>:<line_number>
Description: <one sentence — what the vulnerability is>
Impact: <what an attacker can achieve>
Evidence:
  <relevant code snippet>
Judge: <one sentence — why this passed re-verification>
Remediation: <specific fix — not generic advice>
Reference: references/<vuln>.md
```

For NEEDS CONTEXT findings:

```
[UNVERIFIABLE] VULN-NNN — <Vulnerability Class>
File: <path>:<line_number>
Blocked by: <what additional context is needed>
```

#### Report Structure

When producing a full report, write to `sast_report.md` (or user-specified path):

```markdown
# SAST Security Report — <target>
Date: <date>
Analyzer: llm-sast-scanner v1.3

## Executive Summary
<2-3 sentences: total findings by severity, most critical issue>

## Critical Findings
## High Findings
## Medium Findings
## Low Findings
## Informational
## Unverifiable Findings

## Remediation Priority
<ordered fix list>
```

---

## Key Principles

- **Evidence over assertion**: always show the vulnerable code path, not just the pattern name
- **Context matters**: a finding is only valid if the sink is reachable with user-controlled data
- **Avoid false positives**: if sanitization exists, verify it is bypassable before marking VULN
- **Be precise**: include exact file paths and line numbers — never approximate
- **Fix > flag**: always provide a concrete remediation, not just a problem statement
- **Language-aware**: adapt sink/source patterns to the specific language and framework in use
