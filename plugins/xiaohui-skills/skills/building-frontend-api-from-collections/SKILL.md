---
name: building-frontend-api-from-collections
description: Use when generating or updating frontend API integration code from a Reqable collection JSON and optional API Markdown docs, especially for creating request wrappers, TypeScript types, auth handling, env-based base URLs, and page or form integration examples without letting components call raw backend URLs directly.
---

# Building Frontend API From Collections

## Overview

Generate frontend integration code from a Reqable collection JSON, using API Markdown docs only as a secondary source for semantics and response details.

Produce three things together: API layer code, TypeScript request/response types, and page or form integration examples that consume the API layer instead of calling raw URLs directly.

## Source Priority

1. Reqable collection JSON is the primary source of truth for request shape.
   Use it for method, path, query, headers, body mode, and auth expectations.
2. API Markdown docs are secondary.
   Use them to clarify field meaning, response structure, business constraints, and example flows.
3. Route code and schemas override stale docs when available.

If the JSON and Markdown conflict, do not average them. Prefer the collection for transport details and prefer real backend code for current truth.

## Output Contract

When implementing frontend integration from these inputs, generate or update:

- request wrappers such as `src/api/*.ts` or `src/services/*.ts`
- shared API types such as `src/types/api.ts`
- page or form integration examples showing how views call the API layer
- `.env` or `.env.example` entries for API base URL when missing
- optional development proxy examples when useful for local development

## Architecture Rules

- Put the backend base URL in environment variables.
  Example: `VITE_API_BASE_URL=/api` or `VITE_API_BASE_URL=http://127.0.0.1:9889/api`.
- Do not let Vue or React pages hardcode backend endpoints.
  Components should call API helpers, not raw `fetch('/path')` strings scattered in views.
- Do not let the frontend call third-party upstream services directly.
  The frontend may call only the project's own backend.
- A development proxy is optional and mainly for local dev ergonomics and CORS handling.
  It is not required in production if the frontend is already talking to the project's backend origin correctly.
- Keep auth injection centralized.
  Bearer token logic belongs in a shared HTTP client or interceptor, not per-page request code.

## Workflow

1. Read the Reqable collection and group endpoints by domain.
2. Identify auth patterns.
   Example: bearer token, guest token, public endpoints.
3. Read Markdown docs for business semantics and example responses.
4. Inspect actual route/schema code when the project is available.
5. Define or update TypeScript types first.
6. Implement request wrappers around a shared HTTP client.
7. Update pages or forms to consume the wrappers.
8. Ensure base URL configuration comes from `.env`.
9. Add or update a dev proxy only if it helps local development.
10. Verify that pages do not directly call raw backend URLs.

## Mapping Rules

### From Reqable request to frontend API wrapper

- `url.base` becomes the route path relative to the frontend HTTP client's base URL when possible.
- `url.query` maps to typed query params.
- JSON body maps to typed payload interfaces.
- `Authorization: Bearer ...` means the request should rely on centralized auth injection.
- Public requests should not manually add auth headers.
- Guest flows should expose explicit parameters such as `guestToken` instead of hiding them in the HTTP client.

### From docs to types

- Use docs to name fields clearly and capture enums, nullable fields, and response nesting.
- If response shape is unclear, inspect backend code before inventing types.
- Keep request types separate from response types.

## Page Integration Rules

- Pages and forms should import API functions, not recreate request payloads in ad hoc ways across multiple files.
- Form state can live in the component, but submission should go through typed API helpers.
- Navigation, success toasts, and error display stay in the page layer.
- Transport concerns stay in the API layer.

## Dev Proxy Guidance

Use a dev proxy when:

- the frontend runs on a separate dev server
- the backend is on another origin or port
- you want cleaner local URLs such as `/api/...`

Do not treat proxying as a mandatory production requirement. The important boundary is that the frontend talks only to the project's own backend, not directly to upstream third-party services.

## Validation

Before claiming completion:

- verify `.env` or `.env.example` contains the base URL setting
- verify the shared HTTP client reads the base URL from env
- verify auth-required endpoints rely on centralized token injection
- verify public endpoints do not manually attach auth
- verify pages use API wrappers instead of raw URLs
- verify generated types match current backend schemas or responses

## Common Mistakes

- Treating Markdown docs as more authoritative than actual request JSON or route code
- Hardcoding `http://127.0.0.1:9889` inside page components
- Putting token logic in every request function or every page
- Letting pages call third-party upstream services directly
- Mixing guest-flow identity with logged-in auth handling
- Generating only API wrappers without the supporting types and page integration examples
