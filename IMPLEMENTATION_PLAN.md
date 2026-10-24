# ILLUMATRIX V.2 — Implementation Plan & Phase Roadmap

> **Document Status:** Active Implementation Roadmap  
> **Approach:** Strict risk-ordered, phased execution; one scoped task at a time; verification before progression.

---

## Phase Overview

- **Phase 0 — Baseline Audit & Security Triage (CURRENT)**
- **Phase 1 — Stabilize the Existing Workflow & Unresolved Invariants**
- **Phase 2 — Harden Provider & Model Manager & Credential Isolation**
- **Phase 3 — Establish Shared AI Execution Engine Interface**
- **Phase 4 — Cost-Policy Router & Bounded Failover**
- **Phase 5 — Truthful Telemetry & Jobs Dashboard**
- **Phase 6 — Production Release Readiness & Final Certification**

---

## Phase 0: Baseline Audit & Security Triage Tasks

### Task 0.1: Repository & Baseline State Audit (Completed & Verified)
- **Objective**: Inspect the real repository tree, scripts, dependencies, server setup, and test runner.
- **Scope**: Codebase audit, runtime environment verification.
- **Affected Files**: `README.md`, `package.json`, `server.ts`.
- **Deliverables**: Comprehensive audit report, running dev server, passing tests.
- **Verification**: `npm test` (84 tests passing), `npm run lint` (clean), `npm run build` (clean).
- **Status**: **Verified**.

### Task 0.2: Project Documentation Foundation (Completed & Verified)
- **Objective**: Create and update the seven root documentation files required by the V.2 brief.
- **Scope**: `README.md`, `PROJECT_REQUIREMENTS.md`, `ARCHITECTURE.md`, `FEATURES.md`, `IMPLEMENTATION_PLAN.md`, `DESIGN_SYSTEM.md`, `PROJECT_MEMORY.md`.
- **Deliverables**: 7 synchronized, evidence-based documentation files distinguishing verified facts from unverified/planned items.
- **Verification**: Cross-file consistency review and path verification.
- **Status**: **Verified**.

### Task 0.3: Credential Security & Gitignore Hardening (Completed & Verified)
- **Objective**: Eliminate risk of plaintext secrets being committed or exposed; create `.gitignore`; quarantine existing credentials in `.provider_connections.json`; dynamically inject environment credentials.
- **Scope**: Added root `.gitignore` ignoring `.env*`, `.provider_connections.json`, `dist/`, `node_modules/`, with `!.env.example` exception. Sanitized `.provider_connections.json`. Added dynamic `resolveEffectiveApiKey()` in `providerManager.ts`. Added test coverage for dynamic loading and zero on-disk persistence.
- **Affected Files**: `/.gitignore`, `/.provider_connections.json`, `src/server/providerManager.ts`, `tests/api-provider-manager.test.ts`.
- **Verification**: 90/90 tests passing (`npm test`), clean lint (`tsc --noEmit`), clean build (`vite build`).
- **Status**: **Verified**.

### Task 0.4: Custom Endpoint SSRF Protection (Completed & Verified)
- **Objective**: Protect server against SSRF attacks through user-configured custom OpenAI-compatible `baseUrl` values.
- **Scope**: Centralized URL safety validator (`src/server/urlSafetyValidator.ts`) enforcing `http:`/`https:` protocols, forbidding URL credentials, blocking loopback IPv4/IPv6, cloud metadata (`169.254.169.254`, `metadata.google.internal`), private RFC 1918 and CGNAT ranges, and resolving hostnames against all DNS answers. Enforced in `providerManager.ts` (`testConnection` and `executeTextGeneration`), in `server.ts` (`app.post('/api/providers')`), with manual redirect re-validation (`safeSsrfFetch`).
- **Affected Files**: `src/server/urlSafetyValidator.ts`, `src/server/providerManager.ts`, `server.ts`, `tests/api-provider-manager.test.ts`.
- **Verification**: 103/103 tests passing (`npm test`), clean lint (`tsc --noEmit`), clean build (`vite build`).
- **Status**: **Verified**.

---

## Phase 1: Stabilize Existing Workflow Tasks

### Task 1.1: Long-Form Duration Decision Resolution
- **Objective**: Confirm or formalize the duration options for 16:9 Long-Form (3-minute vs 4-minute minimum).
- **Scope**: Maintain existing `[3, 4, 5, 8, 10]` minutes or update if product decision is provided.
- **Prerequisite**: Product owner decision.
- **Status**: **Blocked on Decision DEC-01**.

### Task 1.2: End-to-End Workflow Regression Suite (Completed & Verified)
- **Objective**: Create or improve deterministic regression tests covering all seven workflow stages (Setup & Rules, Source Library, Script Studio, Scene Prompts, SEO Package, Thumbnail Studio, Review & Export).
- **Scope**:
  - Validates stage-to-stage navigation and state persistence across storage reloads.
  - Validates downstream stale marking and invalidation matrix on configuration, visual style, source story, and script edits.
  - Validates single-deliverable regeneration isolation (e.g. single scene prompt, single thumbnail option, title variant selection).
  - Validates missing required inputs and invalid workflow transitions across all stages.
  - Validates Review & Export readiness audit for blockers (missing source, scene count mismatch, unapproved script, incomplete scene packages) and warnings (stale deliverables, unapproved thumbnails).
  - Validates all 6 export formats (Master Markdown, JSON Backup, Google Flow batch, Teleprompter narration script, SEO metadata file, Thumbnail prompts file) and certifies zero secrets exposure.
  - Preserves long-form duration options `[3, 4, 5, 8, 10]` minutes and Devanagari Hindi language fidelity.
- **Affected Files**: `tests/e2e-workflow-regression.test.ts`, `src/services/storage.ts`, `PROJECT_MEMORY.md`, `IMPLEMENTATION_PLAN.md`.
- **Verification**: `npm test` (119/119 passing tests, 0 failures), `npm run lint` (0 diagnostics), `npm run build` (clean Vite production build).
- **Status**: **Verified**.

### Task 1.3A: SSRF DNS Rebinding Risk Assessment (Completed & Verified)
- **Objective**: Conduct formal security audit of custom-provider endpoint handling (`src/server/urlSafetyValidator.ts`, `providerManager.ts`, `server.ts`) for DNS rebinding and TOCTOU vulnerabilities.
- **Scope**: Code trace, analysis of 7 security properties, transport connection evaluation, test plan design.
- **Findings**:
  - Found that `safeSsrfFetch` calls `validateCustomProviderUrl` at $T_0$, but then calls Node global `fetch` at $T_1$, which performs a second unpinned DNS lookup, enabling DNS rebinding bypass.
  - Designed pinned-socket transport using native Node `node:https`/`node:http` maintaining TLS certificate & SNI verification.
- **Status**: **Assessment Completed & Verified**.

### Task 1.3B: SSRF Transport Hardening & Deterministic IP Pinning (Completed & Verified)
- **Objective**: Implement socket-level IP pinning transport for `safeSsrfFetch` using native `node:https`/`node:http`, preserving SNI and certificate validation, adding 10s default timeout and 5 MiB payload cap, and blocking IPv4-compatible IPv6 addresses.
- **Scope**:
  - Implemented `pinnedHttpRequest()` with socket-level IP pinning via custom DNS `lookup` callback on Node's native `node:https`/`node:http` client.
  - Strictly preserved `parsed.hostname` for Host header, SNI (`servername`), and TLS certificate validation (`rejectUnauthorized: true`).
  - Implemented 10-second default overall timeout returning `category: 'TIMEOUT'` (HTTP 504) and 5 MiB maximum response body size returning `category: 'PAYLOAD_TOO_LARGE'` (HTTP 413) with full stream/socket cleanup.
  - Validated and pinned each 3xx redirect hop independently (max 3 hops), stripping `Authorization` on cross-origin redirects.
  - Closed IPv6 classification gap for deprecated IPv4-compatible IPv6 addresses (`::127.0.0.1`, `::169.254.169.254`, `::10.0.0.1`) and IPv4-translated IPv6 addresses (`::ffff:0:a.b.c.d`).
  - Added 14 automated tests in `tests/api-provider-manager.test.ts`.
- **Affected Files**: `src/server/urlSafetyValidator.ts`, `tests/api-provider-manager.test.ts`, `PROJECT_MEMORY.md`, `IMPLEMENTATION_PLAN.md`.
- **Verification**: `npm test` (133/133 tests passing across 10 suites), `npm run lint` (0 diagnostics), `npm run build` (clean Vite production build).
- **Status**: **Completed & Verified**.

### Task 1.4A: Provider Execution & Cost Policy Audit (Completed & Verified)
- **Objective**: Audit existing provider execution engine, backend cost policies, Auto Free selection, BYOK behavior, and fallback handling (assessment-only).
- **Scope**:
  - Inspected `providerManager.ts`, `modelCompatibility.ts`, `server.ts`, `WorkflowContext.tsx`, `ProviderManagerModal.tsx`, and test suites.
  - Verified 10 policy areas across Auto Free, BYOK, paid execution, server-side cost enforcement, credentials, retries/fallback, error categories, endpoints, connection testing, and secret leakage.
  - Documented findings: server-side cost policy engine is currently unimplemented; failover does not enforce cost tier parity; connection testing dispatches live generative inference; unrecognized connection ID silently falls back to default.
- **Affected Files**: `PROJECT_MEMORY.md`, `IMPLEMENTATION_PLAN.md`.
- **Verification**: `npm test` (133/133 passing), `npm run lint` (0 diagnostics), `npm run build` (clean Vite build).
- **Status**: **Assessment Completed & Verified**.

### Task 1.4B: Provider Execution & Cost Policy Remediation (Completed & Verified)
- **Objective**: Remediate the 4 cost policy, failover, connection testing, and routing findings identified in Task 1.4A (`COST-01`, `COST-02`, `COST-03`, `ROUTE-01`).
- **Scope & Implementation**:
  - **Strict Connection Resolution (`ROUTE-01`)**: Updated `src/server/providerManager.ts` (`getEffectiveCredentials` and `executeTextGeneration`) to fail closed with `CONNECTION_NOT_FOUND` (HTTP 404) whenever an explicitly requested `connectionId` does not exist. Silent fallback to default connections is permanently eliminated.
  - **Server-Side Cost Policy Enforcement (`COST-01`)**: Implemented and enforced strict backend cost policies across all execution paths (`free_only`, `prefer_free`, `manual`):
    - `free_only`: Positively verifies zero-cost status using trusted metadata. Immediately rejects paid, unverified, missing, or ambiguous pricing models with `PAYWALL_BLOCKED` (HTTP 402).
    - `prefer_free`: Prefers zero-cost models, but rejects paid model execution without explicit authorization (`allowPaidExecution: true`).
    - `manual`: Enforces connection validity and blocks paid execution without explicit authorization.
    - Server routes in `server.ts` (`/api/ai/generate-script`, `/api/ai/regenerate-scene`, `/api/ai/generate-scene-prompts`, `/api/ai/regenerate-style-prompts`, `/api/ai/regenerate-scene-prompt`, `/api/ai/generate-seo`, `/api/ai/generate-thumbnails`, `/api/ai/render-thumbnail-image`) pass cost parameters and handle errors via `handleGenerationError`.
  - **Failover Cost-Tier Isolation (`COST-02`)**: Updated `findEligibleFailoverConnection` in `src/server/providerManager.ts` to strictly prohibit `free_only` from failing over to paid or unverified connections. `prefer_free` failovers cannot cross into paid connections without explicit authorization (`allowPaidExecution: true`). Preserves complete credential and connection isolation.
  - **Safe Connection Testing (`COST-03`)**: Hardened `testConnection` in `src/server/providerManager.ts`:
    - Uses non-generative connectivity/key validation endpoints for supported providers (Google GenAI `/v1beta/models`, OpenRouter `/api/v1/auth/key`, Groq `/v1/models`).
    - For paid models or providers requiring generative tests, requires explicit authorization (`allowBillableTest: true`) or fails closed with `BILLABLE_TEST_REQUIRED` (HTTP 402).
  - **Comprehensive Regression Suite**: Implemented 10 deterministic test suites in `tests/cost-policy-remediation.test.ts`.
- **Affected Files**: `src/server/providerManager.ts`, `src/server/modelCompatibility.ts`, `server.ts`, `tests/cost-policy-remediation.test.ts`, `IMPLEMENTATION_PLAN.md`, `PROJECT_MEMORY.md`.
- **Verification**: `npm test` (144/144 tests passing across 11 suites), `npm run lint` (0 diagnostics), `npm run build` (clean Vite production build).
- **Status**: **Completed & Verified**.

---

## Phase 2: Harden Provider & Model Manager

### Task 2.1: Secure Key Storage Migration (Completed & Verified)
- **Objective**: Audit and securely migrate provider API-key storage. Eliminate plaintext credentials at rest, enforce cryptographic encryption at rest, centralize secret redaction, and implement origin protection for provider management APIs.
- **Scope & Implementation**:
  - **Comprehensive Storage Audit**: Audited all 6 credential pathways (UI input, in-memory runtime, persistent file storage, outbound API responses, logging/errors, and backup exports). Identified at-rest plaintext risk in `.provider_connections.json` for BYOK keys.
  - **AES-256-GCM AEAD Encryption at Rest**: Implemented cryptographic storage engine (`src/server/credentialSecurity.ts`) using standard Node.js `node:crypto`.
    - Key derivation via RFC 5869 HKDF-SHA256.
    - 96-bit (12-byte) unique cryptographically secure random IV per encryption.
    - 128-bit (16-byte) authentication tag guaranteeing ciphertext authenticity and integrity.
    - Format: `enc:v1:<base64-iv>:<base64-tag>:<base64-ciphertext>`.
    - Tampering with ciphertext or auth tag fails closed with `DecryptionFailedError`.
  - **Server-Side Storage Boundary & Fail-Closed Behavior**:
    - Precedence: `ILLUMATRIX_ENCRYPTION_KEY` / `KEY_ENCRYPTION_SECRET` environment variables -> dedicated local single-user master key file (`.encryption_master_key` mode `0o600`).
    - If master key is disabled/missing (`ILLUMATRIX_DISABLE_FILE_KEY=true`), persistent disk encryption fails closed with `MasterKeyMissingError` (HTTP 500); BYOK keys are held strictly in memory for the active process without leaking plaintext to disk (`apiKey: ""` on disk).
    - Legacy plaintext auto-migration: When `.provider_connections.json` is loaded with legacy plaintext keys, it automatically encrypts them to `enc:v1:...` and overwrites disk with plaintext erased.
  - **Centralized Secret Redaction**: Centralized `redactSecretString`, `redactHeaders`, `redactObject`, and `sanitizeError` covering all provider prefixes (Google AI Studio, OpenAI, OpenRouter, Groq, Cerebras, Hugging Face, SambaNova), auth headers, and query parameters.
  - **Cross-Site & Origin Protection**: Implemented `validateProviderAccessOrigin` on `/api/providers` rejecting foreign web origins with HTTP 403 `FORBIDDEN`.
  - **Environment & Repository Hygiene**: Hardened `/.gitignore` to ignore `.provider_connections.json*`, `*.key`, `*.secret`, `.env*`. Updated `/.env.example` with placeholders for `ILLUMATRIX_ENCRYPTION_KEY` and optional provider keys.
  - **Automated Regression Suite**: Added 10 deterministic tests in `tests/secure-key-storage.test.ts`.
- **Affected Files**: `src/server/credentialSecurity.ts`, `src/server/providerManager.ts`, `server.ts`, `tests/secure-key-storage.test.ts`, `package.json`, `/.gitignore`, `/.env.example`, `IMPLEMENTATION_PLAN.md`, `PROJECT_MEMORY.md`.
- **Verification**: `npm test` (155/155 tests passing across 11 suites), `npm run lint` (0 diagnostics), `npm run build` (clean Vite production build).
- **Status**: **Completed & Verified**.

### Task 2.2: Live Catalog Discovery Caching & Cooldowns
- **Objective**: Cache model discovery responses with configurable TTL to prevent rate-limiting third-party provider endpoints.
- **Status**: **Planned**.

---

## Phase 3: Establish Shared AI Execution Interface

### Task 3.1: Typed AI Execution Request/Response Contracts
- **Objective**: Introduce unified types for AI tasks (`script_generation`, `scene_prompt_generation`, `style_regeneration`, `thumbnail_prompt_generation`, `seo_generation`).
- **Status**: **Planned**.

### Task 3.2: Incrementally Route AI Endpoints Through Execution Engine
- **Objective**: Route Stage 03 (`generate-script`) through the shared execution engine with validation and structured error classification.
- **Status**: **Planned**.

---

## Phase 4: Cost-Policy Router & Bounded Failover

### Task 4.1: Cost Policy Engine (`Free Only`, `Prefer Free`, `Cost Controlled`, `Manual`)
- **Objective**: Guarantee server-side enforcement that `Free Only` rejects any model whose pricing is paid or unknown.
- **Status**: **Planned**.

### Task 4.2: Two-Pool Model Management (Verified Free vs BYOK)
- **Objective**: Keep user BYOK pool cleanly isolated from auto-verified free routes.
- **Status**: **Planned**.

---

## Phase 5: Truthful Telemetry & Jobs Dashboard

### Task 5.1: Observability & Jobs Console
- **Objective**: Provide detailed job history showing task type, status, duration, retries, and honest data source attribution.
- **Status**: **Planned**.

---

## Phase 6: Production Release Readiness

### Task 6.1: Full Release Checklist & Master Audit Certification
- **Objective**: Execute end-to-end certification, secret audit, bundle size checks, and build readiness.
- **Status**: **Planned**.
