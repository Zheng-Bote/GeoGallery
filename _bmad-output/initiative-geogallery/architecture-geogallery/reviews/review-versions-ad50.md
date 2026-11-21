# Review: AD-50 (plus edits to AD-26/AD-47), versions and reality check

Lens: was each committed decision checked against the web, the existing project or the snippet, or is it asserted from training data?
Scope: AD-50, the AD-26 line on geocoding API keys, and the AD-47 audit event "Änderungen der Geocoder-Konfiguration". Provider policies were not researched, because the user declined that.
Reality source: `/Users/zb_bamboo/Documents/DEV/CPP/gallery_snippets/geocoding` (regeocode 0.1.0: `README.md`, `include/regeocode/*.hpp`, `src/re_geocode_core.cpp`, `src/quota_manager.cpp`, `src/http_client.cpp`, `docs/re-geocode_example.ini`, `docs/quota_status_example.json`).

## Verdict

AD-50 adds no new technology or version that lacks evidence. The real gap is the snippet: its quota and fallback model does not match what AD-50 requires. The spine reuses the snippet "as a library in geocoder" (memlog, reuse decision), so this gap is the main risk. Five changes to the library are needed (F1 to F5), and an epic should own them.

## Technology and fact claims in AD-50

| Claim | Evidence | Status |
| --- | --- | --- |
| XChaCha20-Poly1305 (libsodium) for API keys | review-versions.md: `crypto_aead_xchacha20poly1305_ietf_*`, libsodium 1.0.22 | OK. AD-50 reuses AD-26's primitive and changes only where the encrypted data lives (DB column instead of file). |
| Key delivered as a Podman secret | review-versions.md: Quadlet `Secret=` is documented; Podman 6.2.0 | OK |
| `LISTEN/NOTIFY` for reload without restart | Already settled for the queue library (memlog, queue decision) | OK. The mechanism is the same. The open point is how the reload is applied in the library, see F4. |
| Angular admin module, gRPC admin interface | Angular 22 checked (review-versions.md); gRPC 1.85.x checked | OK |
| "Nominatim 1 Abfrage/s" | User constraint (memlog) | Acceptable as a user constraint. It is still listed as open in the spine (Nominatim usage policy for bulk lookups), which is correct. |
| "Tageslimit" + "Reset-Zeitpunkt" as the quota model | User constraint | Acceptable. Note that the model assumes one counter per provider with one daily reset. If a provider has a monthly or rolling quota, that is configuration (the spine already treats provider limits as configuration), but the schema must be able to store it. See F1. |
| "failed nur bei ungültigen Koordinaten" | No library evidence | The library cannot tell this case apart today, see F3. |

No new library version is pinned. Nothing in AD-50 depends on starter defaults.

## Reality check against regeocode

### F1 — Quota storage is a fixed file, not a pluggable store (blocking for AD-50)

- `ReverseGeocoder` holds `mutable QuotaManager quota_manager_` as a concrete member and builds it from `quota_file_path` (`re_geocode_core.hpp`). There is no interface and nothing can be injected.
- `QuotaManager` loads `quota_status.json` once, keeps it in memory behind a process-local `std::mutex`, and rewrites the whole file on every `try_consume` (`quota_manager.cpp`). With two or more workers or processes, each one holds its own copy and the last writer wins. This is exactly the failure AD-50 is meant to prevent.
- The reset uses the date from `std::localtime` (local midnight). There is no configurable reset time and no time zone. AD-50 requires "Reset-Zeitpunkt" per provider.
- The quota is consumed before the HTTP call, so network failures and HTTP errors also use up quota. That may be intended, but AD-50 should state it.

**Change:** add an abstract `QuotaStore` with an operation such as `try_acquire(provider, now) -> {granted | rate_wait(until) | exhausted(reset_at)}`, injected into `ReverseGeocoder`. Keep the file implementation for the CLI. The geocoder service implements the store in Postgres in a single statement (a conditional `UPDATE ... SET count = count + 1 WHERE count < limit RETURNING`, with the reset folded in) so the check is atomic across all workers.

### F2 — No per-second rate limit exists (missing feature)

The library has only `daily_limit`. There is no per-second throttle, and `ApiConfig` has no field for one. `batch_reverse_geocode` starts one `std::async` per coordinate with no limit on concurrency, which would break 1 request/s at once. "Zähler und Takt atomar in der DB" is therefore new code, not reuse.

**Change:** fold the per-second throttle into the same `QuotaStore` call (store the last-allowed time or a token bucket per provider). geocoder must not use `batch_reverse_geocode`. The job queue (AD-16) controls concurrency.

### F3 — The library returns no machine-readable failure classes (blocks the retry rule)

Every failure is a `std::runtime_error` with a text message: "Daily limit exceeded", "HTTP error: N", 599 for a curl error, and "Unknown API". `reverse_geocode_fallback` catches all of them, writes them to `std::cerr` and returns `{"error":"All providers failed"}`. AD-50's rule ("all exhausted or unreachable -> `retry_wait` until the earliest reset; `failed` only for invalid coordinates") needs three things the library cannot provide:

- the class of each failure (limit, rate, unreachable, HTTP 429/5xx, invalid input or no result),
- the earliest reset time across all providers,
- a check for invalid coordinates. There is no range check, and it is unclear whether an empty or "no result" response (for example coordinates over the ocean) counts as "invalid" or as "Ort fehlt". AD-50 should state which.

**Change:** add a typed result (`std::expected<AddressResult, GeoError>` with an error kind and `retry_after`), and have the fallback return one error per provider plus the minimum `reset_at`. Replace `std::cerr` with the service's JSON logger (Conventions).

### F4 — Configuration source can already be swapped; three fields are missing and reload needs a design

- The good part: the `ReverseGeocoder` constructor takes `unordered_map<string, ApiConfig>` plus adapters. `ConfigLoader` (INI) is separate, and the priority list is passed on each call. A config loader that reads from the DB fits without changing the core.
- `ApiConfig` has no `active`, `rate_per_sec` or `reset_at` field, and the order lives in INI `[strategies]` instead of on the provider. Add these fields, or map them in geocoder.
- `configs_` cannot change after construction. Reloading on `NOTIFY` means building a new `ReverseGeocoder` and swapping it atomically (for example with `std::atomic<std::shared_ptr<...>>`) while running calls finish on the old instance. Write this down as a rule so that no one changes the map in place while it is being used.
- `ConfigLoader` throws if `API-Key` is missing (`re_geocode_core.cpp:128`). That conflicts with AD-50's "Admin-Schnittstelle meldet nur, ob ein Key gesetzt ist" for providers that need no key, such as Nominatim. The DB loader must allow an empty key.

### F5 — API keys end up in URLs, which puts the "never in logs" rule at risk

Keys are written into the query string through `uri_template` (`key={{ apikey }}`, `subscription-key=`, `token=`, `username=`) and passed to curl as the URL. The library does not log the URL itself. However, any OTel or curl tracing of HTTP clients (Conventions: OTel) that records `url.full`, a curl verbose log, or an error message that contains the URL would leak the key. AD-50 states "never in logs/views/answers" but does not say how this is enforced.

**Change (small):** add to AD-50 or to the Conventions that outgoing geocoder URLs are redacted before they are logged or traced. Where a provider accepts a header, send the key in the header instead.

Side note, not researched (provider policy): `HttpClient` sends the generic User-Agent `regeocode-cpp/1.0`. Add this to the existing open Nominatim check before the first import.

## Edits to AD-26 and AD-47

- AD-26: this is consistent with AD-50. It now says "Die Schlüssel dafür" (plural), which leaves open whether the file and the DB column share one Podman secret or use two. Name it, for example `geogallery-secrets-key` and `geocoder-apikey-key`, so that backup or restore (key stored apart from the host) covers both. Also note that a DB restore without this key makes the API keys unreadable. That is acceptable, because they can be entered again, but write it down.
- AD-47: the audit event was added. Nothing in it can go out of date.

## Summary of required library work

1. Add an injectable `QuotaStore` with a Postgres implementation that is atomic and handles both per-second and daily limits with a configurable reset (F1, F2).
2. Return typed errors plus the earliest `reset_at` from the fallback, and define what "invalid coordinates" means (F3).
3. Add the `ApiConfig` fields `active`, `rate_per_sec` and `reset_at`, allow empty keys, and swap the configuration atomically on reload (F4).
4. Redact URLs in logs and traces (F5).
