---
name: redis-session-authority-cache
description: Use when adding Redis-backed token control, device login limits, refresh-token rotation, or user authority caching to a Node.js backend.
---

# Redis Session And Authority Cache

## Scope

Use this pattern for AI/app login systems that need:

- Server-side token validity control.
- Refresh token rotation.
- Max active device limits.
- MySQL-backed session audit records.
- Redis authority cache for course/character access.

Do not replace the database truth with Redis. Redis accelerates and controls runtime state; MySQL remains the durable source.

## Session Keys

Use a dedicated Redis DB if available, so keys do not need product prefixes.

- `session:{sid}` -> JSON string session record.
- `refresh:{refreshJti}` -> string `sid`.
- `user_sessions:{userId}` -> set of active `sid`.
- `device_session:{userId}:{deviceFingerprint}` -> string `sid`.
- `access_blacklist:{accessJti}` -> string `1`, TTL to access token expiry.

Session JSON should contain only login/session decision fields:

```json
{
  "userId": 5,
  "deviceFingerprint": "md5",
  "brand": "Apple",
  "model": "iPhone",
  "accessJti": "uuid",
  "refreshJti": "uuid",
  "allowLegacyAccess": true,
  "defaultSystem": "new",
  "status": "active",
  "createdAt": 1770000000,
  "accessExpiresAt": 1770040000,
  "refreshExpiresAt": 1772600000
}
```

Do not store full user profile data in the session key. Cache profile data separately if needed.

## MySQL Session Table

Keep a durable `ai_user_sessions` table for audit and Redis fallback.

Required fields:

- `ai_user_id`
- `session_id`
- `device_fingerprint`
- `brand`
- `model`
- `device_type`
- `platform`
- `app_version`
- `version_code`
- `device_channel`
- `access_jti`
- `refresh_jti`
- `status`: `active | revoked | expired`
- `last_active_at`
- `created_at`
- `updated_at`

Write MySQL on every login, refresh rotation, and logout. Redis may be unavailable; device limits must still fall back to active rows in MySQL.

## Device Limit

The current temporary device identity is:

```ts
deviceFingerprint = md5(`${brand}::${model}`)
```

Rules:

- Same `userId + deviceFingerprint`: revoke previous session and create a new one.
- New device: count active sessions.
- If active device count is greater than or equal to the configured max, reject login.
- Default max should be configurable, for example `AI_MAX_ACTIVE_DEVICES`, with a sane default such as `3`.

Known limitation: two same-model devices for the same user are treated as one device. Prefer a stable frontend `device-id` later.

## Token Payload

Access and refresh JWTs must keep existing public response field names, such as `accessToken` and `refreshToken`.

JWT payload should include:

- `sub`: user id.
- `scope`: `ai_user`.
- `tokenType`: `access` or `refresh`.
- `sid`: session id.
- `jti`: token id.
- `deviceId`: current `deviceFingerprint`.

## Login Flow

1. Validate login credential.
2. Resolve or create the AI user.
3. Parse device info from headers.
4. Build `deviceFingerprint`.
5. Check same-device existing session.
6. If same device exists, revoke it.
7. If new device, enforce max active device count.
8. Generate `sid`, `accessJti`, `refreshJti`.
9. Write Redis session keys if Redis is available.
10. Insert `ai_user_sessions`.
11. Return existing response fields unchanged.

## Access Validation

For protected AI APIs:

1. Verify access JWT signature and expiry.
2. Require `scope=ai_user` and `tokenType=access`.
3. Require `sid` and `jti`.
4. Reject if `access_blacklist:{jti}` exists.
5. Load `session:{sid}` from Redis, or fall back to MySQL.
6. Require `status=active` and matching `accessJti`.
7. Load current user from MySQL.

If Redis connection fails, do not permanently cache the failed connection. Return `null` and allow MySQL fallback where designed.

## Refresh Rotation

Refresh token reuse must be prevented.

1. Verify refresh JWT.
2. Require `sid` and old `refreshJti`.
3. Resolve `refresh:{refreshJti}` to `sid`, or fall back to MySQL by `refresh_jti`.
4. Load active session.
5. Generate new `accessJti` and `refreshJti`.
6. Delete old `refresh:{oldRefreshJti}`.
7. Update `session:{sid}` and `refresh:{newRefreshJti}`.
8. Update MySQL `access_jti`, `refresh_jti`, `last_active_at`.
9. Return `accessToken` and `refreshToken` with unchanged field names.

## Logout

Current-device logout:

- Delete `session:{sid}`.
- Delete `refresh:{refreshJti}`.
- Delete `device_session:{userId}:{deviceFingerprint}`.
- Remove `sid` from `user_sessions:{userId}`.
- Add `access_blacklist:{accessJti}` with TTL to access expiry.
- Update MySQL session `status=revoked`.

All-device logout:

- Iterate `user_sessions:{userId}` when Redis exists.
- Fall back to active MySQL sessions when Redis is unavailable.
- Revoke each session.

Optionally add an environment switch such as `AI_SESSION_LOGOUT_ENABLED=false` for debugging database records without revoking sessions.

## Authority Cache Keys

Cache by user, not by resource.

- `course_ids:{userId}` -> Redis set of accessible course ids.
- `course_group_ids:{userId}` -> Redis set of accessible course group ids.
- `char_ranges:{userId}` -> JSON string of `{ startCid, endCid }[]`.
- `has_char_module:{userId}` -> string `1` or `0`.

Use these through one service method, for example:

```ts
getUserAuthoritySnapshot(userId)
```

Business code should not assemble Redis keys directly.

## Authority TTL

Use the nearest valid permission expiry as the TTL upper bound.

```ts
ttl = min(maxCacheTtlSeconds, earliestExpiredAt - now)
```

If no permission has `expired_at`, use a short fixed TTL such as 600 seconds.

Never rely only on TTL. Actively clear authority cache on permission changes.

## Cache Invalidation

Clear user authority cache after:

- Login if login may attach or repair identities.
- Redemption success.
- Legacy account claim.
- Legacy authority migration.
- Legacy access status change.
- Admin changes to user authority groups.
- Admin changes to authority group items.
- Admin changes to authority resource mappings.

Prefer a single helper:

```ts
clearUserAuthorityCache(userId)
```

## Logging

Log steps, not secrets.

Useful events:

- `phone_session.login.start`
- `phone_session.login.device_checked`
- `phone_session.login.device_limit_reached`
- `phone_session.login.session_created`
- `refresh_session.start`
- `refresh_session.rotated`
- `logout.session.revoked`
- `auth.session.invalid`
- `authority.cache.hit`
- `authority.cache.miss`
- `authority.cache.cleared`
- `redis.connect.failed`

Never log access tokens, refresh tokens, SMS codes, or full phone numbers.

## Verification

Minimum tests:

- Login creates Redis keys and MySQL session.
- Same-device login revokes old session.
- Different devices are limited by max count.
- Redis unavailable still enforces device limit from MySQL.
- Refresh rotates token IDs and rejects old refresh token.
- Logout revokes current session and blocks old access token.
- Authority cache hit/miss and explicit cache clear.
