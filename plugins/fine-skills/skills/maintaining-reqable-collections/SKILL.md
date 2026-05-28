---
name: maintaining-reqable-collections
description: Use when creating, updating, normalizing, or validating a Reqable collection JSON file, especially when aligning one collection to a reference example, syncing requests to current backend APIs, fixing broken import structure, or converting between pretty-printed and single-line JSON.
---

# Maintaining Reqable Collections

## Overview

Maintain Reqable collection JSON files so they stay importable, structurally consistent, and aligned with the current API surface.

Prefer preserving existing business requests while normalizing the collection's container structure, request metadata, and serialization style.

Use the bundled script when the task is mostly structural normalization. The script is intentionally conservative: it normalizes Reqable metadata, common headers, and output style, but it does not infer missing business parameters from your backend.

## Workflow

1. Identify the target collection file and whether there is a reference file such as `example.json`.
2. Inspect the current API source of truth before editing.
   Typical sources: route files, request schemas, API docs, existing curl examples.
3. Decide the scope of normalization.
   Common scopes: request grouping, request names, headers/body format, top-level metadata, single-line output.
4. Update the collection without changing working business semantics unless the user explicitly asks.
   For mainly structural cleanup, run `scripts/normalize_reqable_collection.js`.
5. Validate the final JSON by parsing it.
6. Spot-check a few representative requests to confirm URLs, methods, headers, query params, and body mode match the current API.

## Rules

- If the user gives a reference Reqable JSON, match its structural style first.
  Focus on top-level keys, folder/api item shapes, header conventions, and metadata layout.
- Preserve real request semantics from the current project.
  Do not blindly copy outdated example parameters such as legacy `uid`, old auth schemes, or deprecated fields.
- Keep request names readable and action-oriented.
  Prefer names like `用户登录`, `查询课程`, `下单-游客`, `订单日志-登录用户`.
- For authenticated requests, keep the auth header style consistent with the current backend contract.
- If a request does not require authentication, set its Reqable authorization mode explicitly to `none`. Do not leave public requests on `inherit`, because inherited auth can misrepresent the real contract.
- The bundled normalization script follows the same rule: requests without an `Authorization` header are normalized to `authorization.mode = "none"`.
- For guest/public requests, put parameters in the correct place for the HTTP method.
  Example: `GET` uses query string, not JSON body.
- Only minify to a single line when the user explicitly asks, or when the existing file is already maintained that way.
- After minifying, always parse the file again to verify it was not corrupted.

## Common Operations

### Align to a reference file

- Compare the target collection against the reference file structure.
- Reuse the reference style for `info`, `items`, `properties`, `revision`, `headers`, `body`, `script`, `authorization`, and `settings`.
- Keep the target project's actual endpoints and payloads.
- If the work is mostly structural, run:

```bash
node /Users/fine/.codex/skills/maintaining-reqable-collections/scripts/normalize_reqable_collection.js \
  --target path/to/collection.json \
  --reference path/to/example.json
```

### Sync to current APIs

- Read route definitions and request validation schemas.
- Update URLs, methods, query params, JSON bodies, and auth requirements.
- Remove fields no longer accepted by the backend.
- Add separate requests when logged-in and guest flows differ.

### Minify the collection

- Parse JSON first.
- Rewrite with `JSON.stringify(data)`.
- Parse again after writing.
- Or use the bundled script:

```bash
node /Users/fine/.codex/skills/maintaining-reqable-collections/scripts/normalize_reqable_collection.js \
  --target path/to/collection.json \
  --minify
```

## Validation

Use a real JSON parse check before claiming completion.

Recommended command pattern:

```bash
node -e "JSON.parse(require('fs').readFileSync('path/to/file.json','utf8')); console.log('ok')"
```

If a reference file was involved, also inspect a few representative requests to ensure the result matches the intended style and still reflects the current API behavior.

## Resources

### scripts/

- `scripts/normalize_reqable_collection.js`
  Normalize Reqable collection structure, common internal headers, top-level metadata, and output serialization.
