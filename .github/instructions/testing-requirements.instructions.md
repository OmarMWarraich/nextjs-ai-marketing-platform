---
description: "Use when changing behavior in Next.js or Python service code. Defines required test updates and preferred test quality standards."
name: "Testing Requirements"
applyTo:
  - "nextjs/**/*.{ts,tsx}"
  - "asset-processing-service/**/*.py"
---
# Testing Requirements

- Hard rule: Any behavior change must include a test update (new test or updated existing test) that proves the new expected behavior.
- Hard rule: Any bug fix must include a regression test that fails before the fix and passes after.
- Hard rule: Avoid shipping untested changes to API routes, auth logic, billing flows, or asset-processing pipelines.
- Preference: Keep tests deterministic by mocking network, OpenAI, Stripe, and clock/time dependencies where applicable.
- Preference: Prefer focused unit/integration tests near the changed behavior over broad, brittle end-to-end additions.
- Preference: Run the smallest relevant test/lint set for touched code and report what was executed.
