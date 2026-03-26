---
description: "Use when editing the Next.js app in this repository. Enforces hard safety rules and soft consistency preferences for App Router, validation, and project structure."
name: "Next.js App Guidelines"
applyTo: "nextjs/**/*.{ts,tsx}"
---
# Next.js App Guidelines

- Hard rule: Keep server-only logic in server contexts (route handlers, server actions, or server modules). Do not import server-only modules into client-executed files.
- Hard rule: Validate external input (request bodies, query params, webhook payloads, and AI tool inputs) with zod before use.
- Hard rule: Do not expose secrets or raw server environment values to client components.
- Preference: Reuse shared types from interfaces and existing helpers from lib/utils before introducing new abstractions.
- Preference: Keep edits minimal and scoped, preserving existing public API and file shape unless refactoring is explicitly requested.
- Preference: For UI work, follow existing component and Tailwind patterns already used in the Next.js app.
