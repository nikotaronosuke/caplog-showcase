# Owner Decision Log

[日本語](OWNER_DECISIONS.md) | English

Caplog has both Web and Mobile clients, but many of its important engineering choices came from deciding **what should be shared and what should stay separate**.

This document highlights owner-level decisions about the product experience, safety boundaries, migration strategy, API responsibilities, privacy, and verification.

---

## 1. Move the center of development to Mobile without abandoning the Web product

### Problem

Caplog already had a Web product in production.

At the same time, several core experiences fit Mobile better:

- using the map while moving
- building plans from photos
- using location data
- recording an outing while away from a desktop

A full rewrite that replaced Web with Mobile would throw away an already-running client and duplicate product data.

### Decision

The product moved to:

- **Mobile = primary active development client**
- **Web = still-running browser client**
- **Supabase = shared product data**

Plans, spots, and posts are not duplicated into separate "mobile" and "web" copies.

The goal was not to rebuild everything around the newer client.
It was to move the main experience while preserving the product that was already in use.

**Evidence:** [README — shared data / mobile-first](../README.md)

---

## 2. Separate public URL identity from internal database identity

### Problem

Using the internal UUID directly in every public URL is simple.

But it creates URLs that are:

- long
- hard to read
- disconnected from place context
- harder to evolve later

### Decision

Posts keep the internal UUID for relations and authorization, while public URLs use a separate short `public_id`.

The canonical shape is:

```text
/posts/{prefecture}/{city}/{public_id}
```

The public id is **not** treated as a security mechanism.

Private-data protection remains the responsibility of authentication, ownership checks, and database access control.

Public identity and authorization identity stay separate.

**Evidence:** [Public URL design](public-url-design.md)

---

## 3. Preserve legacy URLs instead of making migration purity a prerequisite

### Problem

When moving to a cleaner URL format, deleting the old UUID routes would simplify the routing system.

It would also break:

- shared links
- search-indexed links
- partially migrated records

### Decision

Legacy UUID URLs continue to resolve.

When a canonical public URL can be built, the old URL redirects with **308 Permanent Redirect**.

If required public-url data is missing, the old UUID form remains a fallback instead of producing a 404.

Redirect loops are explicitly avoided.

The migration was designed around **continuity first**, not a requirement that all existing data become perfectly migrated before the new structure can ship.

**Evidence:** [Public URL design — legacy redirect / fallback](public-url-design.md)

---

## 4. Do not draw a fake straight line when route geometry is unknown

### Problem

Connecting two spots with a straight line makes the map look complete.

But that line may have little relationship to the real route.

For transit such as train or bus, it can be especially misleading.

### Decision

If route data is unavailable, Caplog leaves the segment **without a line**.

The same rule applies when saved polyline data is invalid:

- decode failure
- invalid coordinate range
- fewer than two valid points

A visually complete map is less important than avoiding route geometry that the product cannot justify.

**Evidence:** [Map system — no fake straight lines](map-system.md)

---

## 5. Allow route retrieval to succeed per segment instead of all-or-nothing

### Problem

A plan may contain several adjacent route segments.

If the entire plan depends on one provider call succeeding for every segment,
one upstream failure can remove all route data.

### Decision

Routes are handled per adjacent segment.

If one segment fails:

- successful segments remain available
- the failed segment stays unavailable
- the overall plan remains usable

This is an intentional **partial-success** model.

The system uses the information it actually has without manufacturing missing data.

**Evidence:** [App API — segment-level route success](app-api.md)

---

## 6. Keep server-side provider credentials out of the Mobile bundle

### Problem

Calling server-oriented Places / Routes endpoints directly from the mobile app would reduce backend code.

It would also move provider credentials and billing-sensitive traffic control into a client distributed to users.

### Decision

A dedicated **Cloudflare Workers App API** sits between Mobile and external provider APIs.

Mobile sends a Supabase Bearer token.
The Worker resolves the authenticated user before continuing.

Provider credentials remain on the server side.

The boundary also becomes the place for:

- rate limiting
- response shaping
- field selection
- error sanitization
- caching / provider-specific behavior

**Evidence:** [App API](app-api.md)

---

## 7. Deploy the Mobile App API separately from the Web application

### Problem

Adding Mobile API endpoints to the existing Web Worker would be operationally simpler.

But then mobile autocomplete / routing traffic and normal Web delivery would share:

- request budget
- CPU budget
- deployment
- incident scope

### Decision

The App API is deployed separately from the Web client.

Google / Supabase remain common dependencies, but the application-specific traffic and failure surface are separated.

The goal was not the fewest deployments.
It was a clearer **failure and cost boundary**.

**Evidence:** [App API — separate failure boundary](app-api.md)

---

## 8. Fail closed when a paid endpoint loses its rate-limit protection

### Problem

If the rate-limit service fails, a fail-open design keeps the endpoint available.

For external paid APIs, that can also remove the control preventing unbounded provider calls.

### Decision

Paid endpoints such as Places / Routes fail closed.

If the application cannot verify the rate-limit guard, it does not continue to the provider.

Non-billable health / diagnostic endpoints can use a different policy.

The failure policy is chosen based on **what the failure can cost**, rather than maximizing availability uniformly.

**Evidence:** [App API — fail-closed rate limits](app-api.md)

---

## 9. Do not proxy raw external-provider responses to Mobile

### Problem

A thin proxy could forward the raw Google response directly to the app.

That would expose provider response structure throughout the client and send many fields the product does not need.

### Decision

The Worker extracts the required values and returns a smaller app-specific response.

Requests also use Field Masks so only needed provider fields are requested.

Examples:

- Place Details → fields needed by the screen
- Routes → encoded polyline needed for map rendering

This reduces payload, provider coupling, accidental field exposure, and unnecessary API / SKU cost.

**Evidence:** [App API — response shaping / Field Masks](app-api.md)

---

## 10. Keep enough logging to diagnose problems without making logs a location-history store

### Problem

External API and map failures need diagnostics.

But logging values such as:

- latitude / longitude
- Place ID
- search text
- encoded route polyline
- user id

can turn operational logs into a record of user activity.

### Decision

The log shape is deliberately limited.

Useful operational values such as request id, status, counts, and short error codes can be kept.

Sensitive location / identity / raw provider data is not logged.

The design is neither "log nothing" nor "log everything."
It keeps the minimum operational evidence needed for debugging.

**Evidence:** [App API — counts-only logging](app-api.md) / [Map system — location logging boundary](map-system.md)

---

## 11. Make map focus depend on real spot spacing instead of one fixed zoom

### Problem

A single zoom level behaves badly across different plans:

- nearby spots overlap
- distant spots are framed too tightly

The bottom card carousel also covers part of the map.

### Decision

Map focus uses the distance from the selected spot to its nearest other spot,
then clamps the resulting viewport within product-defined bounds.

The focus center is also offset to account for the card carousel.

The map and the plan-card interaction are treated as one experience instead of two unrelated controls.

**Evidence:** [Map system — carousel sync / dynamic zoom](map-system.md)

---

## 12. Do not treat emulator / static-mock success as final verification for device behavior

Maps, photos, location, Android markers, and other device-specific behavior can look correct in code and still behave differently on a real phone.

### Decision

Mobile development uses an Expo dev client and real Android verification.

Real-device behavior is part of acceptance for device-dependent features.

This is not simply a platform feature.
It is a decision about **what counts as done**.

**Evidence:** [README — real-device verification](../README.md)

---

## What this project prioritizes

Caplog prioritizes:

- moving to mobile-first development without breaking the running Web product
- separating public identity from authorization identity
- preserving old links during URL migration
- not drawing route data the product does not actually have
- partial success for independent external route segments
- keeping provider credentials and billing controls off the Mobile client
- separating Mobile API and Web failure domains
- fail-closed behavior where provider cost is at risk
- shaped responses instead of raw upstream payloads
- diagnostic logs that avoid user location details
- map interaction based on real layout and spot spacing
- real-device verification before final acceptance

AI-assisted development is used in the project.

The part I wanted this repository to preserve is **how convenience, compatibility, cost, privacy, and truthfulness were balanced at the boundaries**.
