# Owner Decision Log

[日本語](OWNER_DECISIONS.md) | English

Caplog's production source is private. This document is not presented as public-code verification; it summarizes four decisions documented in the public showcase.

## 1. Moved the center of development to Mobile without abandoning Web

The Web product was already in production, while location, photos, and map-heavy flows fit Mobile better.

Instead of replacing Web entirely:

- Mobile became the primary development client
- Web stayed in production
- both share Supabase data

**Evidence:** [README](../README.md)

## 2. Changed public URLs without breaking legacy links

Public posts moved from raw UUID routes to `/posts/{prefecture}/{city}/{public_id}`.

Legacy UUID URLs remain supported. When the canonical URL can be built they 308-redirect; partially migrated data can fall back instead of becoming a 404.

Continuity was prioritized over a perfectly clean migration.

**Evidence:** [Public URL design](public-url-design.md)

## 3. Did not fill missing route segments with fake straight lines

Drawing a straight line would make the map look complete while misrepresenting the real route.

Missing route data stays line-less. In multi-segment plans, successful segments remain even if one segment fails.

**Evidence:** [Map system](map-system.md) / [App API](app-api.md)

## 4. Separated the Mobile API from Web and made paid endpoints fail closed

Server-side Google Places / Routes credentials stay behind a dedicated Cloudflare Worker rather than inside the Mobile bundle.

The App API is deployed separately from Web. If the rate-limit guard cannot be verified, paid endpoints do not continue to the provider.

Diagnostic logs are also limited to status / counts rather than coordinates, Place IDs, search text, or raw provider responses.

**Evidence:** [App API](app-api.md)
