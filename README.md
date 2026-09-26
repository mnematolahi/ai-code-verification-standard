# AI-Assisted Code Quality, Testing & Release Verification Standard

**Version:** 1.0 &nbsp;·&nbsp; **Last updated:** 2026-09-25 &nbsp;·&nbsp; **Status:** Active reference standard

> **Purpose:** A practical, language-agnostic checklist for reviewing and validating code written or modified with AI coding agents before merge, deployment, or release.
>
> **Primary languages covered:** Python, PHP, Laravel. The structure is intentionally extensible to JavaScript/TypeScript, Go, Rust, Java, .NET, and others.

---

## Scope

This document defines a practical verification standard for code that is written or modified by AI coding agents in general-purpose backend/web projects. It applies whether the AI wrote the change unassisted, was used for a large refactor, or only touched a small function.

## Non-Goals

This standard intentionally does **not** cover:

- evaluating the quality of an AI/ML model itself (accuracy, bias, hallucination rate of a model you are training or fine-tuning);
- mobile-native (iOS/Android) or embedded/firmware release processes;
- infrastructure-as-code verification (Terraform, Pulumi, Ansible, Kubernetes manifests) beyond the generic CI/CD guidance in [§28](#28-cicd-pipeline-security-github-actions-hardening);
- legal/licensing review of AI-generated code for similarity to training data — a real and growing concern, but outside the scope of a testing/verification checklist.

## How to Use This Standard — Adoption Profiles

Not every project needs every gate in this document on every change. A checklist nobody can realistically follow gets ignored entirely. Pick a profile per project (or per repository) and apply it consistently; moving a project to a stricter profile later is easier than abandoning an all-or-nothing standard.

| Profile | Typical project | Baseline sections |
|---|---|---|
| **Minimal** | Personal tools, internal scripts, early prototypes | [§1](#1-core-principle), [§2](#2-mandatory-rules-for-ai-coding-agents), [§4](#4-formatting-and-linting), [§6](#6-unit-tests), [§23](#23-security-scanning-static), [§24](#24-secrets-detection), [§38](#38-git-diff-review) |
| **Standard** | Most production web/backend projects | Minimal + [§5](#5-static-analysis), [§8](#8-regression-tests), [§11](#11-coverage) (line coverage only), [§12](#12-integration-tests), [§18](#18-authentication-and-authorization-tests), [§19](#19-validation-and-abuse-cases), [§25](#25-dependency-and-supply-chain-checks), [§37](#37-manual-review-of-ai-generated-code), [§51](#51-release-gate) |
| **Critical** | Payments, auth systems, healthcare, anything handling money or personal data | All sections, including [§9](#9-property-based-testing), [§10](#10-mutation-testing), [§22](#22-concurrency-and-race-conditions), [§26](#26-behavioral--runtime-trust-verification), [§28](#28-cicd-pipeline-security-github-actions-hardening), [§49](#49-webhook-and-payment-testing), [§51](#51-backup--recovery-validation) |

A project may adopt individual gates from a higher profile without adopting the whole tier — for example, a Minimal-profile project that happens to process payments should still adopt §49 regardless of its overall tier.

## Normative Language

This document uses **MUST** / **MUST NOT** for non-negotiable rules, **SHOULD** / **SHOULD NOT** for strong defaults that can be consciously overridden with a documented reason, and **MAY** / "consider" for optional practices. Everything in [§2 (Mandatory Rules)](#2-mandatory-rules-for-ai-coding-agents) is MUST-level regardless of the softer phrasing used for readability. Where a section says "should" for a project on the **Critical** profile, read it as MUST.

## Table of Contents

1. Core Principle
2. Mandatory Rules for AI Coding Agents
3. Pre-Change Baseline
4. Formatting and Linting
5. Static Analysis
6. Unit Tests
7. Assertions Must Test the Contract
8. Regression Tests
9. Property-Based Testing
10. Mutation Testing
11. Coverage
12. Integration Tests
13. Database and Migration Testing
14. API Testing
15. Browser / End-to-End Testing
16. Browser Test Depth
17. Browser Test Data Safety
18. Authentication and Authorization Tests
19. Validation and Abuse Cases
20. File Upload Testing
21. Background Jobs / Queues / Scheduled Tasks
22. Concurrency and Race Conditions
23. Security Scanning (Static)
24. Secrets Detection
25. Dependency and Supply-Chain Checks
26. Behavioral / Runtime Trust Verification
27. Docker / Container Checks
28. CI/CD Pipeline Security (GitHub Actions Hardening)
29. Build Verification
30. Reproducibility
31. Logging Verification
32. Error and Observability Testing
33. Performance Smoke Tests
34. Accessibility Checks
35. API / Frontend Contract Verification
36. Golden / Snapshot / Fixture Tests
37. Manual Review of AI-Generated Code
38. Git Diff Review
39. AI Agent Skill Verification
40. Agent Self-Review Prompt
41. Test Pyramid
42. Failure Classification
43. Flaky Test Policy
44. Test Isolation
45. Test Data Integrity
46. Webhook and Payment Testing
47. External Service Failure Testing
48. Backup / Recovery Validation
49. AI Change Provenance & Final Verification Report
50. Definition of Done
51. Release Gate
52. Release Checklist
53. Recommended Repository Structure
54. Suggested Command Matrix
55. CI Recommendation
56. Human + AI Review Model
57. Final Principle

---

## 1. Core Principle

AI-assisted development is treated as **untrusted change until verified**.

No code should be considered release-ready merely because:

- the agent says it works;
- the application starts successfully;
- unit tests pass;
- the code looks correct in review;
- a single happy-path browser test succeeds.

A release candidate should pass multiple independent layers of verification:

```text
Format
  ↓
Lint
  ↓
Type / Static Analysis
  ↓
Unit Tests
  ↓
Regression Tests
  ↓
Integration / Database Tests
  ↓
API / Contract Tests
  ↓
End-to-End / Browser Tests
  ↓
Security Checks (Static + Behavioral)
  ↓
Dependency / Supply-Chain Checks
  ↓
Build / Deployment Validation
  ↓
Logs / Runtime Verification
  ↓
Final Diff + Release Review
```

The objective is not to prove that software is perfect. The objective is to **reduce the probability of shipping broken, insecure, incompatible, or accidentally changed behavior**.

---

## 2. Mandatory Rules for AI Coding Agents

An AI coding agent must follow these rules before declaring a task complete.

### 2.1 Never claim a test passed unless it was actually executed

Forbidden:

```text
Tests should pass.
Everything looks good.
This should work.
```

Preferred:

```text
pytest: 184 passed
mypy: passed
ruff: passed
```

If a command could not be executed:

```text
NOT RUN
Reason: Docker is unavailable.
```

### 2.2 Never weaken tests to make the task pass

The agent must not:

- delete failing tests without justification;
- reduce assertions;
- change expected results simply to match the implementation;
- skip tests without documenting why;
- reduce coverage thresholds;
- disable security rules;
- weaken lint/type-check configuration;
- silently change snapshots/golden files;
- remove validation to avoid an error.

If a test is genuinely outdated, the agent must explain why the behavior changed and update the test intentionally.

### 2.3 Never hide failures

A failed test, warning, crash, timeout, browser error, HTTP 500, console error, migration failure, or flaky test must be reported.

### 2.4 Preserve the project contract

Before changing behavior, verify:

- existing public APIs;
- CLI behavior;
- database schema;
- routes;
- validation rules;
- authentication/authorization;
- response formats;
- error formats;
- file formats;
- existing integrations.

### 2.5 Use the smallest safe change

Do not refactor unrelated code merely because it is nearby.

### 2.6 Diff size and blast-radius must be proportionate to the task

Before declaring a task complete, the agent should be able to state:

- how many files were changed;
- how many lines were changed;
- whether any file was touched that the task description did not call for.

A diff that is significantly larger than the stated task would suggest must come with an explanation, not silence. An unexplained large diff is itself a review signal, independent of whether the individual lines are correct — it is one of the most common ways an AI agent quietly does more than was asked.

---

## 3. Pre-Change Baseline

Before modifying an existing project, the agent should establish a baseline.

Run the project's existing checks when available and record:

- test count;
- passing tests;
- failing tests;
- skipped tests;
- coverage;
- lint status;
- type-check status;
- build status;
- known warnings;
- current dependency status.

Example:

```text
BASELINE
pytest: 241 passed, 2 skipped
coverage: 87%
ruff: PASS
mypy: PASS
PHPUnit: 318 passed
Pint: PASS
PHPStan: PASS
build: PASS
```

A new change should not silently introduce unrelated regressions.

---

## 4. Formatting and Linting

Formatting is not a substitute for correctness, but it is the first mechanical quality gate.

### Python

Recommended:

```bash
ruff format .
ruff check .
```

Where type checking is used:

```bash
mypy .
```

Other project-specific formatters/linters may be used when already part of the repository.

### PHP / Laravel

Recommended:

```bash
vendor/bin/pint --test
```

For static analysis:

```bash
vendor/bin/phpstan analyse
```

Laravel projects may additionally use Larastan where appropriate.

### JavaScript / TypeScript

Typical checks:

```bash
npm run lint
npm run format:check
npx tsc --noEmit
```

Use the repository's own package manager and scripts when present.

### General rule

Do not introduce a second formatting system just because an AI agent prefers it. Prefer the project's existing standard.

---

## 5. Static Analysis

Static analysis should catch defects before runtime.

Typical categories:

- unreachable code;
- undefined variables;
- incorrect types;
- invalid return values;
- incompatible interfaces;
- suspicious control flow;
- unused code;
- dangerous API usage;
- invalid framework usage;
- nullability problems;
- unreachable branches;
- inconsistent exception handling.

Recommended tools include:

| Ecosystem | Tools |
|---|---|
| Python | Ruff, mypy, pyright, Bandit |
| PHP | PHPStan, Larastan, Psalm |
| Laravel | PHPStan/Larastan, Pint |
| JS/TS | ESLint, TypeScript compiler |
| General | Semgrep where appropriate |

Use only tools actually configured and supported by the project.

---

## 6. Unit Tests

Every non-trivial function, class, service, parser, validator, calculation, or business rule should have unit-level coverage where practical.

### Python

```bash
pytest
```

Optional coverage:

```bash
pytest --cov
```

### PHP / Laravel

Use the project's PHPUnit or Pest suite, for example:

```bash
php artisan test
```

or the repository's configured PHPUnit/Pest command.

Unit tests should verify both:

- expected behavior;
- important failure behavior.

Do not test only the happy path.

---

## 7. Assertions Must Test the Contract

Weak test:

```python
assert result is not None
```

Stronger test:

```python
assert result.status == "approved"
assert result.total == Decimal("125.00")
assert result.currency == "IRR"
```

Assertions should verify the actual business contract.

Where possible, test:

- exact output;
- important fields;
- state transitions;
- side effects;
- exceptions;
- database changes;
- emitted events;
- external calls;
- authorization decisions.

Avoid tests that pass even when the implementation is obviously wrong.

---

## 8. Regression Tests

Every significant bug discovered during development should ideally become a regression test.

Pattern:

```text
Bug discovered
     ↓
Minimal reproduction
     ↓
Failing test
     ↓
Fix
     ↓
Test passes
     ↓
Test stays permanently
```

The goal is to prevent the same bug from returning later.

A regression test should reproduce the original failure as closely as practical.

Examples:

- invalid date caused crash;
- duplicate payment caused two orders;
- permission check was bypassed;
- malformed JSON caused 500;
- empty field caused database error;
- race condition created duplicate records.

---

## 9. Property-Based Testing

For algorithms and complex input handling, test properties rather than only examples.

### Python

Consider:

```text
Hypothesis
```

Examples of properties:

- sorting preserves element count;
- parser never crashes on valid generated input;
- normalization is idempotent;
- serialization followed by deserialization preserves data;
- total never becomes negative where the domain forbids it.

### PHP

Use an appropriate property-based testing library only when the project benefits from it.

Do not add a library merely to check a simple function.

---

## 10. Mutation Testing

Mutation testing is useful when the test suite looks large but its assertions may be weak.

The idea:

```text
Original Code
    ↓
Intentional tiny mutation
    ↓
Tests should fail
```

Examples of mutation tools:

- Infection for PHP;
- mutmut or another suitable tool for Python;
- Stryker for supported JS/TS ecosystems.

If an important mutation survives, inspect whether tests actually verify the behavior.

Do not require mutation testing for every small project, but consider it for critical business logic.

---

## 11. Coverage

Coverage is a signal, not proof of correctness.

Track at least:

- line coverage;
- branch coverage where available;
- critical path coverage.

A test suite with 95% line coverage can still miss an important business rule.

Do not increase coverage by writing meaningless assertions.

Critical paths should be tested intentionally:

- authentication;
- authorization;
- payments;
- order creation;
- database migrations;
- permissions;
- data export;
- sensitive workflows.

---

## 12. Integration Tests

Integration tests verify that components work together.

Examples:

```text
API
 ↓
Service
 ↓
Database
```

or:

```text
Controller
 ↓
Validation
 ↓
Service
 ↓
Queue
 ↓
Database
```

Test real integration boundaries where practical.

Examples:

- database transactions;
- cache;
- queues;
- filesystem;
- mail driver;
- storage;
- external-service adapters;
- authentication stack.

Use test infrastructure, not production services.

---

## 13. Database and Migration Testing

For database-backed applications, test:

- fresh installation;
- migrations from a clean database;
- migration from a realistic previous version;
- rollback where supported;
- constraints;
- indexes;
- unique rules;
- foreign keys;
- cascade behavior;
- seed data;
- transaction boundaries.

Laravel examples may include:

```bash
php artisan migrate:fresh --seed
php artisan test
```

Do not run destructive database commands against a real production database.

---

## 14. API Testing

Test:

- status codes;
- response schema;
- validation;
- authentication;
- authorization;
- pagination;
- filtering;
- sorting;
- error format;
- rate limits where relevant;
- idempotency where relevant.

For documented APIs, consider contract/property testing tools such as Schemathesis or equivalent ecosystem-specific tooling.

---

## 15. Browser / End-to-End Testing

Critical user journeys should be tested in a real browser automation environment where appropriate.

Recommended options include:

- Playwright;
- Laravel Dusk;
- Cypress;
- equivalent project-approved browser automation.

Typical flows:

```text
Landing page
→ registration
→ email/verification flow in test environment
→ login
→ dashboard
→ create item
→ edit item
→ save
→ logout
```

For e-commerce:

```text
Browse
→ product page
→ cart
→ checkout
→ fake/test payment
→ payment callback/webhook simulation
→ order creation
→ confirmation
```

IMPORTANT:

All payment tests must use a documented **sandbox/test payment provider, fake gateway, or local mock**.

Never use real money, real bank credentials, or real customer payment credentials during automated verification.

---

## 16. Browser Test Depth

Do not stop at checking whether a button exists.

Verify:

- visible UI state;
- form validation;
- navigation;
- redirects;
- authorization;
- server response;
- database state;
- success state;
- failure state;
- duplicate submission behavior;
- browser console errors;
- failed network requests;
- unexpected 4xx/5xx requests;
- accessibility-critical interactions;
- mobile/responsive behavior where required.

Where useful, collect:

- screenshot;
- video;
- trace;
- network log;
- console log.

---

## 17. Browser Test Data Safety

Use dedicated test identities and synthetic data.

Examples:

```text
user@example.test
qa-user-001
fake card/test token provided by the payment sandbox
```

Do not automate real account registration on third-party production services unless explicitly authorized and legally appropriate.

For applications that require email verification, use a controlled test inbox, local mail catcher, or staging mail service.

---

## 18. Authentication and Authorization Tests

For every protected feature, test at least:

```text
Unauthenticated user → denied
Authenticated normal user → allowed only where permitted
Wrong role → denied
Correct role → allowed
Resource owner → allowed where required
Non-owner → denied
Expired session/token → denied
```

Also test IDOR/BOLA-style cases:

```text
User A requests User B's resource
→ must be denied
```

Do not assume frontend hiding is authorization.

Authorization must be tested server-side.

---

## 19. Validation and Abuse Cases

For every input boundary test:

- empty input;
- null where relevant;
- minimum value;
- maximum value;
- too-long value;
- invalid format;
- Unicode;
- unexpected type;
- duplicate value;
- malformed JSON;
- missing fields;
- extra fields;
- boundary numeric values.

Where applicable, test:

- SQL injection resistance;
- XSS resistance;
- command injection resistance;
- path traversal resistance;
- SSRF protections;
- file upload validation.

Use safe test payloads in isolated test environments.

---

## 20. File Upload Testing

Where uploads exist, test:

- allowed type;
- invalid extension;
- MIME mismatch;
- oversized file;
- empty file;
- malformed file;
- duplicate filename;
- filename traversal;
- dangerous filename;
- archive extraction;
- storage isolation;
- download authorization.

Verify that user-uploaded files cannot unexpectedly execute as server-side code.

---

## 21. Background Jobs / Queues / Scheduled Tasks

Test:

- job dispatch;
- successful execution;
- retries;
- failure state;
- idempotency;
- duplicate execution;
- timeout behavior;
- scheduled task behavior.

A scheduled job should not silently run twice and create duplicate financial or business records.

---

## 22. Concurrency and Race Conditions

Important operations should be tested under concurrent requests where practical.

Examples:

- two simultaneous checkout requests;
- double-click payment;
- concurrent inventory reservation;
- two workers updating one record;
- simultaneous password reset requests.

Verify:

- transactions;
- locking;
- unique constraints;
- idempotency keys;
- retry behavior.

---

## 23. Security Scanning (Static)

Security checks should include the tools appropriate to the project. This section covers **static** scanning only — checks performed by reading the code and its metadata, without executing it in a monitored sandbox. See [§26](#26-behavioral--runtime-trust-verification) for dynamic/behavioral verification.

### Python

Consider:

```text
Bandit
pip-audit
```

### PHP

Consider:

```text
composer audit
```

and suitable static security analysis.

### Repository-wide

Consider:

```text
Gitleaks
Semgrep
OSV-Scanner
Trivy
```

where appropriate.

These tools should not be added automatically if they duplicate existing project controls.

---

## 24. Secrets Detection

Before release, scan for accidental secrets:

- API keys;
- private keys;
- tokens;
- passwords;
- cloud credentials;
- database credentials;
- test credentials accidentally copied from production.

Recommended tool:

```text
Gitleaks
```

False positives should be handled through documented, narrow allowlists—not by disabling the scanner globally.

---

## 25. Dependency and Supply-Chain Checks

Review:

- newly added packages;
- removed packages;
- version changes;
- lockfile changes;
- package source/registry changes;
- install scripts;
- Git dependencies;
- URL dependencies.

Use the ecosystem's native audit mechanisms where appropriate.

Examples:

```text
pip-audit
composer audit
npm audit / pnpm audit / yarn audit
OSV-Scanner
```

Dependency warnings must be evaluated rather than blindly ignored.

### AI-specific supply-chain risks

AI coding agents can suggest a dependency, or an API call within a dependency, that does not actually exist — or that exists under a different, similarly-named package. Attackers are known to register packages under the exact names a given model is statistically likely to hallucinate ("slopsquatting"), so this is not a theoretical concern.

Before accepting a dependency suggested by an AI agent:

- confirm the package genuinely exists on the official registry (PyPI, npm, Packagist, …) under the **exact** name and version referenced;
- treat a brand-new, low-download, low-maintainer package with a name close to a well-known one as a WARNING, not an automatic block — `Unknown ≠ Malicious`, but it deserves a second look;
- confirm that functions, classes, or endpoints the AI cites from that dependency actually exist in the version installed — do not assume an AI-cited API is real without checking the installed library's actual public interface.

---

## 26. Behavioral / Runtime Trust Verification

Static analysis, linting, and functional tests confirm that code behaves correctly for the inputs it was tested against. They do **not** confirm that the code has no *additional*, unintended behavior — an unexpected outbound network call, an attempt to read credentials outside its declared scope, an install script doing something unrelated to the dependency's stated purpose. This gap matters specifically for AI-generated code: an agent's output is rarely adversarial, but it can still introduce unreviewed side effects that no unit test would ever exercise, because no one asked it to write a test for something they didn't know to look for.

For changes on the **Standard** or **Critical** adoption profile, consider adding a dynamic, sandboxed behavioral verification step to the pipeline, in addition to the static and functional checks above. A behavioral verification step typically:

- runs the changed code in an isolated container with no network access by default;
- records the process tree, filesystem writes, and any outbound network attempts;
- plants inert "canary" credentials to detect unexpected credential access or exfiltration attempts;
- reports findings on an evidence-based severity scale (e.g., `PASS / INFO / WARNING / HIGH / CRITICAL`) rather than a binary pass/fail, since most unexpected behavior is not malicious and should not be reported as if it were.

**AgentGuard** is a companion project — currently in development, not yet published — built specifically to provide this kind of behavioral trust testing for AI-generated and AI-modified code: static analysis combined with sandboxed runtime observation, canary-based exfiltration detection, and evidence-based severity reporting. It is meant to complement this standard, not replace it: this document verifies that code does what it is supposed to do; AgentGuard verifies that it does not do anything it was not supposed to do. *(Repository link to be added here once it is published.)*

If no dedicated behavioral-verification tool is available yet, at minimum apply the same principle manually: run the changed code once in a container with no network egress and confirm it does not attempt outbound connections that the task does not call for.

---

## 27. Docker / Container Checks

If the project uses containers, verify:

- Dockerfile builds;
- image starts;
- health checks work;
- environment configuration works;
- no unexpected privileged mode;
- no unnecessary host mounts;
- no secrets baked into image layers;
- non-root operation where practical;
- exposed ports are intentional.

For image vulnerability scanning, consider Trivy or an equivalent approved scanner.

---

## 28. CI/CD Pipeline Security (GitHub Actions Hardening)

Once this standard — and the project it governs — lives on GitHub, the CI pipeline itself becomes part of the attack surface, not just a tool for enforcing the checks above. Verify:

- workflows triggered by `pull_request` do **not** have write access to repository secrets by default; anything that needs secrets and also needs to run against fork PRs should use `pull_request_target` only with extreme care, after explicitly checking out a trusted ref rather than the PR head;
- third-party GitHub Actions are pinned to a full commit SHA, not a mutable tag (`uses: owner/action@<full-sha>` rather than `@v3`, which can be silently repointed by the action's maintainer or an attacker who compromises their account);
- the default `GITHUB_TOKEN` permissions are explicitly scoped to the minimum needed (a top-level `permissions:` block) rather than left at the repository's broad default;
- changes to workflow files (`.github/workflows/**`) require review from a maintainer, since a workflow change is effectively a change to the project's trust boundary, not an ordinary code change;
- self-hosted runners, if used, are never exposed to workflows triggered by untrusted fork PRs.

---

## 29. Build Verification

A release candidate must be buildable from a clean environment.

The clean build should start from:

```text
clean checkout
```

and then perform:

```text
install dependencies
→ build
→ migrate/test database
→ run tests
→ package/deploy artifact
```

This helps detect hidden local-machine dependencies.

---

## 30. Reproducibility

Where practical, verify that the same source revision produces the same or equivalent build artifact.

Check:

- lockfiles;
- pinned tool versions where appropriate;
- generated files;
- build scripts;
- environment assumptions.

The project should not depend on undeclared software installed only on one developer's machine.

---

## 31. Logging Verification

After integration and browser tests, inspect logs.

Look for:

- unexpected exceptions;
- stack traces;
- authentication failures;
- authorization failures;
- database errors;
- queue failures;
- retries;
- timeout spikes;
- unexpected external requests;
- warnings introduced by the new code;
- sensitive information accidentally logged.

A green test suite does not replace log inspection.

---

## 32. Error and Observability Testing

For important failures verify that:

- the user sees a safe message;
- sensitive details are not exposed;
- the server logs sufficient diagnostic context;
- the error has an appropriate status code;
- retries do not create duplicate side effects.

Test representative failure modes, not only success paths.

---

## 33. Performance Smoke Tests

Before release, perform a basic performance sanity check for critical endpoints.

Check:

- response time;
- memory usage;
- CPU usage;
- query count where practical;
- N+1 query problems;
- unnecessary external calls.

For serious systems, use an appropriate load-testing tool such as:

- k6;
- Locust;
- equivalent approved tooling.

Do not run aggressive load tests against production unless explicitly authorized.

---

## 34. Accessibility Checks

For browser-based applications, include accessibility checks where appropriate.

Possible tooling:

- axe-core / axe-playwright;
- Lighthouse;
- framework-specific accessibility tooling.

At minimum inspect:

- labels;
- keyboard navigation;
- focus behavior;
- form errors;
- semantic structure;
- color-independent status communication.

Automated accessibility testing is not a complete accessibility audit.

---

## 35. API / Frontend Contract Verification

When frontend and backend are changed together, verify that:

- field names match;
- types match;
- nullability matches;
- error formats match;
- pagination formats match;
- authentication behavior matches;
- versioned endpoints remain compatible.

Do not assume a TypeScript interface or PHP/Python model automatically proves runtime compatibility.

---

## 36. Golden / Snapshot / Fixture Tests

Use stable fixtures for outputs that should not unexpectedly change.

Examples:

- JSON response;
- generated document;
- parser output;
- report;
- normalized configuration;
- SQL migration output.

Snapshot changes must be reviewed, not blindly accepted.

An AI agent must never update snapshots automatically merely to make tests pass without explaining the behavioral change.

---

## 37. Manual Review of AI-Generated Code

Automated tests are necessary but not sufficient.

A human or senior reviewer should inspect:

- business logic;
- security boundaries;
- error handling;
- concurrency;
- data integrity;
- external integrations;
- dependencies;
- unexpected abstractions;
- generated code quality;
- maintainability.

Especially review code added around:

```text
Authentication
Authorization
Payments
File uploads
Database writes
External APIs
Secrets
Shell commands
Deserialization
Webhooks
Background jobs
```

---

## 38. Git Diff Review

Before merge, inspect:

```bash
git status
git diff
git diff --stat
```

Review:

- unexpected files;
- generated artifacts;
- lockfile changes;
- dependency changes;
- environment/config changes;
- URLs;
- new scripts;
- new shell commands;
- test changes;
- deleted validation;
- weakened security controls.

The final diff should match the stated purpose of the task.

---

## 39. AI Agent Skill Verification

When using AI coding agents such as OpenCode or other agent frameworks:

1. Identify the project's applicable instructions.
2. Read the relevant repository-level instruction files.
3. Read the relevant Skill files before using them.
4. Confirm the requested tool/Skill actually exists.
5. Do not invent Skills, commands, APIs, or configuration keys.
6. Run the project's tests after using a Skill that changes code.
7. Review the final diff.

If the repository defines:

```text
AGENTS.md
CLAUDE.md
CONTRIBUTING.md
.github/
skills/
.skill/
```

or equivalent agent instructions, treat them as part of the project's development contract.

An agent should report which project instructions affected the implementation.

---

## 40. Agent Self-Review Prompt

Before completing a task, the agent should internally perform this checklist:

```text
Did I change anything unrelated?
Did I add a dependency?
Why was that dependency necessary?
Did I change a lockfile?
Did I add an external URL?
Did I introduce a new network call?
Did I introduce shell/process execution?
Did I change authentication or authorization?
Did I change validation?
Did I change database behavior?
Did I change an API contract?
Did I update tests because behavior changed, or only to make them pass?
Did I add regression coverage for the bug/feature?
Did I run the full relevant test suite?
Did I run linting and formatting checks?
Did I run type/static analysis?
Did I inspect logs?
Did I verify the browser flow where applicable?
Did I test important failure paths?
Did I check for secrets?
Did I inspect the final Git diff?
```

---

## 41. Test Pyramid

Prefer a balanced test strategy:

```text
                 E2E
              /       \
         Integration  Browser/API
            /              \
        Unit / Regression / Property
                 |
        Static / Type / Lint
```

Do not try to solve every requirement with E2E tests.

E2E tests are slower and more fragile; use them for important user journeys.

---

## 42. Failure Classification

Every failed verification should be classified.

```text
CODE BUG
TEST BUG
ENVIRONMENT PROBLEM
DEPENDENCY PROBLEM
FLAKY TEST
DOCUMENTATION PROBLEM
CONFIGURATION PROBLEM
EXPECTED BEHAVIOR CHANGE
UNKNOWN
```

Do not automatically rewrite code when the environment is actually broken.

---

## 43. Flaky Test Policy

A flaky test must not simply be retried forever.

If a test is unstable:

1. reproduce it;
2. collect logs;
3. identify whether the problem is timing, isolation, shared state, network, or environment;
4. fix the underlying cause;
5. document unavoidable limitations.

Retries may mask failures and should not replace diagnosis.

---

## 44. Test Isolation

Tests should avoid hidden dependence on:

- developer machine state;
- local files;
- real user accounts;
- production databases;
- production APIs;
- real payment credentials;
- undeclared environment variables.

Use fixtures, factories, mocks, test databases, sandbox APIs, or controlled test services where appropriate.

---

## 45. Test Data Integrity

Every important test should make clear:

- what data it creates;
- what data it expects;
- what cleanup occurs;
- whether the test is isolated;
- whether it can be safely rerun.

Tests involving money, inventory, permissions, or state transitions should explicitly assert the final state.

---

## 46. Webhook and Payment Testing

For payment or webhook integrations, test at least:

```text
payment initiated
→ pending
→ success
→ failure
→ timeout
→ duplicate callback
→ invalid callback
→ replayed callback
→ signature validation
→ order state update
```

Use the provider's test/sandbox mode or a local fake gateway.

A successful browser redirect alone is not proof that the backend payment flow is correct.

---

## 47. External Service Failure Testing

For every important external dependency, test failure behavior.

Examples:

```text
API unavailable
API timeout
HTTP 500
invalid response
slow response
rate limit
connection reset
expired credential
malformed payload
```

The application should fail safely and predictably.

---

## 48. Backup / Recovery Validation

For systems where data persistence matters, verify:

- backup generation;
- restore procedure;
- migration compatibility;
- integrity after restore.

A backup that was never restored in testing should not be assumed to be recoverable.

---

## 49. AI Change Provenance & Final Verification Report

Every substantial AI-assisted coding task should end with a concise verification report. In addition to the test results, the report must record **which agent produced the change**, so that if an issue surfaces later, the team can trace it back to a specific model, version, and (where available) the task or prompt that produced it — a coding agent should not be an anonymous contributor.

Required fields:

```text
TASK

AGENT / MODEL
(name, version, and — where available — a link or reference to the
originating prompt, issue, or task description)

CHANGED FILES

IMPLEMENTED BEHAVIOR

TESTS RUN

TEST RESULTS

SECURITY CHECKS

DEPENDENCY CHANGES

DATABASE CHANGES

BROWSER/E2E RESULTS

LOG REVIEW

KNOWN LIMITATIONS

NOT RUN / BLOCKED CHECKS

FINAL DIFF STATUS

HUMAN REVIEWER
```

Example:

```text
AGENT / MODEL
Claude Sonnet 4.6, task: "add coupon-code validation to checkout" (issue #482)

TESTS RUN
- pytest: PASS (214 passed)
- ruff: PASS
- mypy: PASS
- browser E2E: PASS (18/18)
- composer audit: PASS
- gitleaks: PASS

NOT RUN
- load test: not required for this change

KNOWN LIMITATION
- Windows-native scheduled-task behavior is not covered by the Linux CI environment.
```

---

## 50. Definition of Done

A task is **Done** only when:

1. The requested functionality exists.
2. The relevant tests exist.
3. Regression coverage exists for important bugs.
4. Static checks pass.
5. Security checks pass or documented findings are accepted.
6. Relevant browser/API flows pass.
7. Logs show no unexplained errors.
8. The final diff is reviewed.
9. Dependency/configuration changes are explained.
10. Limitations are documented.

"Code written" is not the same as "feature verified."

---

## 51. Release Gate

A release should not proceed if a mandatory gate fails.

Recommended gate structure:

| Gate | Required |
|---|---:|
| Format check | Yes |
| Lint | Yes |
| Type/static analysis | Yes where applicable |
| Unit tests | Yes |
| Regression tests | Yes for changed behavior |
| Integration tests | Yes for affected integrations |
| Security/dependency checks | Yes |
| Secrets scan | Yes |
| Build | Yes |
| Critical E2E flows | Yes for affected product areas |
| Log/error review | Yes |
| Final diff review | Yes |
| Behavioral/runtime verification (§26) | Recommended / mandatory on the Critical profile |
| Performance smoke test | Recommended / mandatory for critical systems |
| Accessibility smoke test | Recommended for UI |
| Mutation testing | Recommended for critical business logic |

A failed non-required check must still be documented.

---

## 52. Release Checklist

Before publishing, confirm the automated portion above is green, then walk through the items that still need a human eye:

```text
[ ] Clean checkout tested
[ ] All Release Gate checks (§51) passed
[ ] Critical negative paths tested
[ ] Authentication tested
[ ] Authorization tested
[ ] Database migrations tested
[ ] Docker/image checks passed where applicable
[ ] Browser console reviewed
[ ] Network failures reviewed
[ ] Test payment uses sandbox/fake provider only
[ ] Final Git diff reviewed
[ ] No unexplained files changed
[ ] No unexplained dependency added
[ ] No unexplained external URL added
[ ] Documentation updated
[ ] Known limitations documented
```

---

## 53. Recommended Repository Structure

A repository can expose this standard through files such as:

```text
README.md
CONTRIBUTING.md
AGENTS.md
TESTING.md
SECURITY.md
LICENSE

.github/
  workflows/
    ci.yml
    security.yml

scripts/
  test-all.*
  security-check.*

tests/
  unit/
  integration/
  e2e/
  regression/
  fixtures/
```

The exact structure may vary by project.

---

## 54. Suggested Command Matrix

Use only commands appropriate to the project.

### Python

```bash
ruff format --check .
ruff check .
mypy .
pytest
pytest --cov
pip-audit
bandit -r .
```

### PHP / Laravel

```bash
vendor/bin/pint --test
vendor/bin/phpstan analyse
composer audit
php artisan test
```

Project-specific Laravel checks may additionally include:

```bash
php artisan route:list
php artisan config:clear
php artisan config:cache
```

Only run commands that are safe for the current environment; never point destructive database commands at production.

### JavaScript / TypeScript

```bash
npm run lint
npx tsc --noEmit
npm test
npm audit
```

Use the repository's package manager and scripts when they differ.

### Repository-wide / Optional

```text
Gitleaks
Semgrep
OSV-Scanner
Trivy
Playwright
k6
```

---

## 55. CI Recommendation

Whenever practical, run the deterministic checks automatically in CI.

A typical pipeline:

```text
Checkout
  ↓
Install pinned dependencies
  ↓
Format Check
  ↓
Lint
  ↓
Type Check
  ↓
Unit Tests
  ↓
Integration Tests
  ↓
Security / Secret Scan
  ↓
Dependency Audit
  ↓
Build
  ↓
E2E / Browser Tests
  ↓
Artifact Verification
```

The CI pipeline should fail clearly when a mandatory gate fails. See [§28](#28-cicd-pipeline-security-github-actions-hardening) for hardening the pipeline itself, not just what it checks.

---

## 56. Human + AI Review Model

The safest practical model is:

```text
AI writes
   ↓
AI runs tests
   ↓
Automated CI verifies
   ↓
Human reviews important behavior
   ↓
Release
```

Do not make the coding agent the only authority that decides whether its own code is correct.

---

## 57. Final Principle

The standard is simple:

> **Do not trust generated code because it looks correct. Trust it more only after independent checks, tests, runtime verification, and review produce evidence that it behaves as intended.**

---

## License

This document is released under the license in the [LICENSE](LICENSE) file of this repository. It is intended to be freely reusable and adaptable by other teams and projects.

## Contributing

Contributions, disagreements, and corrections are welcome via issues and pull requests. Changes to this standard should go through the same review discipline it asks projects to apply to their own code: a clear diff, a stated reason, and at least one human reviewer.
