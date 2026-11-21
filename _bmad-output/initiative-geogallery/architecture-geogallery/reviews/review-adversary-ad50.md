---
type: review
lens: adversary
target: _bmad-output/initiative-geogallery/architecture-geogallery/architecture-geogallery.md
target_scope: AD-50 (Geocoder-Konfiguration) and its edits to AD-1 (geocoder row), AD-26, AD-47, Deferred, Prüfpunkte epic-ort-karte
target_status: final (update run 2026-11-21)
reviewed: '2026-11-21'
units: [epic-ort-karte, epic-datensicherheit-betrieb]
---

# Adversarial Review: AD-50 Geocoder-Konfiguration

**Lens:** Take two units one level down. Each obeys every AD to the letter. Find where they still build incompatibly: clashing shared-data shapes, two owners of one entity, or conflicting state-mutation paths. Only AD-50 and the edits it made are in scope. All other ADs are binding context.

**Verdict:** AD-50 puts configuration, quotas and keys in the right place: the `geocoder` schema, owned by geocoder and reached only through the api. But the call graph forbids the admin RPC it needs. The result shape geocoder reports to catalog has no contract. "Never `failed`" has no terminal path for coordinates that cannot be geocoded. The quota mechanics (time base, supersession, wake-up after reset, restore) are left to each builder. Two of the six holes block the build. The other four lead to quota overruns, endless retries or data leaks once real data flows.

Proposed AD text is written in German so it can go straight into the spine. Explanations are in English.

## Severity summary

| # | Severity | Hole | Units in conflict | Fix |
| --- | --- | --- | --- | --- |
| G1 | Critical | The graph allows api → geocoder only for Health, so the admin RPC from AD-50 cannot be built under AD-9/AD-38 | datensicherheit-betrieb (admin UI, api) × ort-karte (geocoder) | Add graph edge; tighten AD-38/AD-50 |
| G2 | Critical | No contract for the shape of `ReportLocation`, and no owner for mapping provider results onto the canonical place key (AD-6) | ort-karte (geocoder) × ort-karte (catalog places) × raster-suche (filters) | Tighten AD-50 + AD-6 |
| G3 | High | "Never `failed`" has no terminal outcome for "no result" (sea, desert), and invalid coordinates are judged by two validators. Both cause endless retries or silent losses | ort-karte (geocoder) × ort-karte (catalog review list) | Tighten AD-50 |
| G4 | High | Supersession by `geo_rev` across long `retry_wait` periods, attempt counters that grow forever, and a thundering herd at quota reset | ort-karte × datensicherheit-betrieb (job views, initial import) | Tighten AD-50 + AD-15 |
| G5 | High | Quota accounting is ambiguous: reset time zone, which clock counts, count before or after the HTTP call, counter state after PITR | ort-karte × datensicherheit-betrieb (restore) | Tighten AD-50 |
| G6 | Medium | Coordinates at rest and on egress: the cache key collides with AD-5 per-image deletion, job views carry coordinates (AD-44), and key ciphertext has no version for rotation or restore | ort-karte × datensicherheit-betrieb (backup, audit, admin views) | Tighten AD-50, AD-26, AD-44 |

---

## G1 — Admin RPC forbidden by the graph (Critical)

**Construction.**
- The datensicherheit-betrieb builder writes the Angular admin module and the api endpoints, and calls `geocoder.Admin*` with the Principal, as AD-50 requires.
- The ort-karte builder writes geocoder's `access.yaml` from the graph, which AD-9 and AD-38 state is exhaustive. The graph shows `api -.->|nur Health| geocoder`, so the builder adds only `grpc.health.v1.Health/*` for the api uid. The socket group (AD-9) does not even include the api's user for data methods.
- Both builders comply. The admin call fails with `PERMISSION_DENIED`, or one builder quietly widens the ACL, and then the graph no longer means "no other call path".
- A second gap: AD-38 says a Principal is accepted only from the api uid, but it does not say which geocoder methods the api may call. Nothing forbids the api from also being allowed `Submit`/`Delete`. Those must come only from catalog, or the api could queue geocoding of arbitrary coordinates.
- `system.admin` also has to exist in the closed list `contracts/permissions/` (AD-10). AD-50 uses the name, but no unit owns adding it.

**Fix (graph + AD-50):**

> Graph: `api -->|gRPC Admin (Principal)| geocoder` ersetzt `api -.->|nur Health| geocoder`.
>
> AD-50, neuer Punkt: „Die api darf in geocoder ausschließlich die Admin-Methoden (`geogallery.geocoder.v1.Admin/*`) und Health aufrufen; `Submit`/`Delete` nimmt geocoder nur von der uid von catalog an (`access.yaml`). Admin-Methoden verlangen einen Principal mit `system.admin`; die Berechtigung steht in `contracts/permissions/`.“

---

## G2 — `ReportLocation` has no shape, and the canonical key has no producer (Critical)

**Construction.**
- AD-6 says places have a canonical key: ISO 3166 for country and province, a provider-independent ID below that, and display names per language. Catalog owns places (AD-1).
- AD-50 adds failover: the same coordinate can be answered by Nominatim today and by Google or OpenCage tomorrow, after a quota reset or a reindex.
- The geocoder builder (using the `geocoding` snippet) reports what the library returns: a provider name, a provider place ID and a localized name for each level.
- The catalog builder expects a canonical key. Seeing provider IDs, it creates one place per provider. "München" now exists as an OSM place and a Google place, and two filter facets split the same images (R17, R22).
- The reverse is just as compliant: catalog normalizes, but then catalog has to know provider-specific formats, which breaks the service boundary.
- Display names "per language" leave open which languages are queried. If geocoder makes one call per language, every image costs N quota units, and the quota numbers in the Prüfpunkt assume one.
- "Region" is deferred to epic-ort-karte "innerhalb AD-6", but the spine does not say which service maps it.

**Fix (AD-50 + AD-6):**

> AD-50, neuer Punkt: „geocoder normalisiert jedes Anbieterergebnis vor dem Report auf die Vertragsform `geogallery.geocoder.v1.Location` (`contracts/proto/`): je Ebene (Kontinent, Land, Provinz, Region, Stadt) kanonischer Schlüssel (ISO 3166-1/-2, darunter anbieterunabhängiger Schlüssel nach AD-6) und Namen je konfigurierter Sprache; Anbieter-ID und Anbietername nur als Herkunftsinfo, nie als Schlüssel. Die Abbildung „Region“ liegt in geocoder. catalog vergibt Orts-UUIDs ausschließlich über den kanonischen Schlüssel und kennt keine Anbieterformate. Eine Bildanfrage zählt genau ein Kontingent pro Sprache; die Sprachliste ist Teil der Konfiguration.“

---

## G3 — No terminal path for "no result", and two validators (High)

**Construction.**
- AD-50 says a job goes to `retry_wait` when every provider is exhausted or unreachable, never becomes `failed`, and becomes `failed` only for invalid coordinates.
- Case "no result": valid coordinates in the open sea, or a provider that answers with an empty result. The geocoder builder reads this as "provider did not deliver", falls through all providers, and lands in `retry_wait` until reset. That repeats every day, burning quota on hopeless jobs forever.
- Case "invalid coordinates": AD-6 gives geo validation to catalog, and AD-50 has geocoder fail the job. If catalog accepts `(0,0)` or out-of-range EXIF values and submits them, geocoder sets the job to `failed`. But AD-50 sends no report back, so catalog never learns about it. The image shows "ungefähr" forever and never reaches the review list (R18).
- The admin ops view (datensicherheit-betrieb) shows a growing `retry_wait` backlog that never drains.

**Fix (AD-50, replace the last bullet):**

> „Ergebnisse eines Abrufs sind genau: `FOUND` (Report mit Location), `NO_RESULT` (mindestens ein Anbieter hat gültig ohne Treffer geantwortet), `INVALID_INPUT`, `DEFERRED` (alle Anbieter erschöpft oder nicht erreichbar). `FOUND`, `NO_RESULT` und `INVALID_INPUT` sind terminal und werden per `ReportLocation` an catalog gemeldet; catalog setzt das Bild auf „Ort fehlt“ bzw. die Prüfliste. Nur `DEFERRED` geht nach `retry_wait` und wird nie `failed`. catalog validiert Koordinaten vor dem Submit (AD-6); geocoder prüft nur den Wertebereich.“

---

## G4 — Supersession, attempt counters and the herd after reset (High)

**Construction.**
- Supersession: a Bestand job for `geo_rev = 3` waits in `retry_wait` for days. The user corrects the GPS (epic-pflege), and catalog submits `geo_rev = 4`. AD-15 makes receivers idempotent over `image_id`+revision, so the two revisions are different jobs and both exist. Whichever finishes first spends quota. The stale result is dropped by catalog, but only after the quota is gone. If permanent deletion happens in between, `Delete` must find jobs in every state, including `retry_wait`.
- Attempt counter: AD-16 requires an attempt counter and backoff. A job deferred for 30 days counts 30 or more attempts. A builder who caps attempts (the usual pattern in the shared C++ job library) pushes it to `failed`, which AD-50 forbids. A builder who does not cap it ends up with meaningless counters in the ops view.
- Herd: AD-50 says "`retry_wait` until the earliest quota reset". At reset, 200,000 or more Bestand jobs wake together. `LISTEN/NOTIFY` wakes every worker, and each worker claims a batch with `SKIP LOCKED`. The rate limit (1/s) then lets one through per second, and the rest go straight back to `retry_wait`. This churns the queue and hits the DB while catalog import runs (AD-17 lowers priority only for ingest and media). New uploads queue behind the Bestand backlog.
- Outage versus quota: "unreachable" is also sent to `retry_wait` until the quota reset. A 5-minute Nominatim outage then delays work by up to 24 h, while a different builder applies normal backoff. The two ops views give different numbers for the same state.

**Fix (AD-50, new bullets):**

> „Je `image_id` existiert höchstens ein offener Geocoding-Job; `Submit` mit höherer `geo_rev` ersetzt Koordinaten und Revision eines Jobs in `queued`/`retry_wait`, niedrigere oder gleiche `geo_rev` werden ignoriert; `Delete` entfernt Jobs in jedem Zustand.“
>
> „`DEFERRED` wegen Kontingent erhöht den Versuchszähler nicht und trägt den Code `QUOTA_EXHAUSTED`, `next_run_at` = frühester Reset; `DEFERRED` wegen Nichterreichbarkeit nutzt den normalen Backoff (AD-16) mit Code `PROVIDER_UNAVAILABLE`, ebenfalls ohne Obergrenze.“
>
> „Ein Worker holt einen Job erst, nachdem er einen Takt-Slot reserviert hat (Pull nach Kontingent, nicht Push nach Reset); Reihenfolge nach Priorität (Neuimport vor Bestand), dann `next_run_at`. Ein Reset weckt höchstens so viele Worker, wie Slots frei sind.“

---

## G5 — Quota accounting is ambiguous (High)

**Construction.**
- Time zone: "Reset-Zeitpunkt" has no time base. Google resets at midnight Pacific time, and other providers reset at midnight UTC. The admin UI builder sends `00:00` in the browser's local zone. The geocoder builder stores `00:00` as UTC (Konvention: system times UTC). Twice a year, at the DST switch, the counters reset an hour early or late, and on the early side the provider limit is exceeded. That is exactly what AD-50's "Prevents" line is meant to rule out.
- Clock: "atomic in the DB" does not say whose clock counts. One worker checks `last_call_at + 1 s <= now()` against its own system clock, while another uses `clock_timestamp()` in PG. `now()` is the transaction start time, so a long transaction lets two calls through within the same second.
- Order: one builder counts after a successful HTTP call. A second worker then sees free quota while the first call is still in flight. Failed calls (HTTP 429/5xx) are often billed or counted by the provider anyway.
- Restore: counters are part of the DB, so PITR (AD-27) rolls them back. A restore at noon to the 08:00 state believes half of today's quota is still free and exceeds the limit right away. datensicherheit-betrieb tests the restore and sees nothing wrong.

**Fix (AD-50, replace the first bullet's second sentence):**

> „Der Reset ist als IANA-Zeitzone + lokale Uhrzeit (bzw. gleitendes Fenster) je Provider konfiguriert; geocoder speichert daraus `next_reset_at timestamptz`. Alle Zeitvergleiche nutzen `clock_timestamp()` der DB, nie die Worker-Uhr. Ein Abruf reserviert Slot und Kontingent vor dem HTTP-Aufruf in einer einzigen Anweisung (`UPDATE … WHERE used < limit AND next_slot_at <= clock_timestamp() RETURNING`); auch fehlgeschlagene Aufrufe zählen. Nach einem Restore gelten alle Kontingente bis zum nächsten Reset als verbraucht.“

---

## G6 — Coordinates at rest and on egress; key ciphertext (Medium)

**Construction.**
- Cache key versus AD-5: AD-5 has geocoder "also remove cache entries for the image". One builder keys the cache by `image_id`, which gives no reuse, so the Bestand hits providers once per image even for 500 photos at the same spot. Another keys it by coordinate, which gives reuse but cannot delete by image. A third keys it by exact coordinate plus `image_id`. After permanent deletion that row still holds the exact coordinate, and that cannot be revoked.
- Egress: the initiative accepts that coordinates leave the host, but the spine says nothing about precision. One builder sends full EXIF precision, and another rounds to the cache grid. AD-50 also does not say that geocoder is the only component that may make outbound calls to providers.
- AD-44: AD-15 makes jobs carry coordinates, and AD-16 makes every queue publish a view. A builder who publishes the job table one to one puts coordinates in the admin ops view (datensicherheit-betrieb), which breaks AD-44.
- Keys: AD-26 encrypts API keys in the DB, with the key as a Podman secret. Rotating that key without a key ID in the ciphertext either breaks every old row or requires an atomic re-encryption. A PITR restore to a point before rotation, combined with the current secret, leaves keys that cannot be decrypted, so every provider fails and every job sits in `DEFERRED` with no visible reason. Without AAD, a ciphertext can also be copied from provider A's row to provider B's.
- Set-key path: the plaintext key crosses the api. The audit plugin (`drogon_audit_plugin`) and OTel tracing must not record the request body. AD-47 now names "Änderungen der Geocoder-Konfiguration" as an audit event, but nothing says the event carries only provider ID and field names.

**Fix (AD-50 new bullet + AD-26 + AD-44):**

> AD-50: „Cache-Schlüssel ist die auf ein festes Raster gerundete Koordinate (Konfiguration, Default ≈ 100 m), ohne `image_id`; an Anbieter geht nur diese gerundete Koordinate. Nur geocoder darf ausgehend Anbieter erreichen. Exakte Koordinaten liegen nur in offenen Jobs und werden mit dem Job gelöscht; `Delete` (AD-5) entfernt Jobs, der Cache hat keine Bildbezüge.“
>
> AD-26: „Verschlüsselte Werte tragen `key_id`; AAD = Tabelle + Zeilen-ID. Das Secret enthält aktiven und vorherige Schlüssel; alte Schlüssel bleiben mindestens über das PITR-Fenster erhalten und werden mit gesichert.“
>
> AD-44 / AD-47: „Job-Views von geocoder enthalten keine Koordinaten. Audit-Ereignisse der Geocoder-Konfiguration enthalten nur Provider-ID und geänderte Feldnamen, nie Werte von Keys; Request-Bodies dieser Endpunkte werden weder auditiert noch getraced.“

---

## Checked and not a hole

- **Two owners of the configuration:** AD-1, AD-14 and AD-50 give geocoder sole ownership, and the api has no schema of its own beyond `audit`. No conflict.
- **Second HTTP surface:** AD-50 rules it out explicitly, which matches AD-8 and AD-47.
- **Rate limit across workers:** AD-50 requires atomic DB accounting. Only the time base and the order were open, and G5 closes those.
- **Folder location before geocoding:** AD-6 and AD-50 agree (`override > geocoded > folder`). Catalog drops stale reports through `geo_rev` (AD-15).
