---
description: "Use when editing the Python asset-processing service. Enforces hard validation/runtime safety rules and soft consistency preferences."
name: "Python Service Guidelines"
applyTo: "asset-processing-service/**/*.py"
---
# Python Service Guidelines

- Hard rule: Validate external payloads with pydantic models (or equivalent typed validation) before processing.
- Hard rule: Keep async code non-blocking; avoid introducing blocking network or file operations on async paths.
- Hard rule: Do not log secrets, API keys, or sensitive payload content.
- Preference: Reuse existing service modules and patterns before adding new abstractions.
- Preference: Keep functions focused and side effects explicit at call boundaries.
- Preference: Match existing project style and typing conventions used in the service.
