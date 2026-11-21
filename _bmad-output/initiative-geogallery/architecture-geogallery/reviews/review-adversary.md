---
type: review
lens: adversary
target: _bmad-output/initiative-geogallery/architecture-geogallery/architecture-geogallery.md
target_status: draft (paused in topic 1)
reviewed: '2026-11-21'
units: [epic-plattform-zugang, epic-import-pipeline, epic-raster-suche, epic-datensicherheit-betrieb, epic-ort-karte, epic-alben-sichtbarkeit, epic-oeffentliche-galerie, epic-pflege]
---

# Adversarial Review: GeoGallery Architecture Spine

**Lens:** Take two units one level down (the 8 epics). Each obeys every AD to the letter. Find where they still build incompatibly, or where the visibility guarantee leaks anyway. The guarantee is: no private, group-only or excluded image, and no metadata field outside the positive list, ever becomes public. For admins, R29 adds that nothing private is visible to them.

**Verdict:** The spine gets the big structure right: one data owner per entity, one writer for image data, and a materialized RLS filter. Its weak spot is that it protects **rows** and not **columns, aggregates, album context or callers**. As written, a fully compliant build can leak exact public GPS, hidden metadata (through search and cursors), excluded images (through album listings) and private content (to the admin, through operations views). A compromised media worker can also forge any Principal. Seven holes are critical and must close before the spine leaves draft.

Proposed AD text is written in German so it can go straight into the spine. Explanations are in English.

## Severity summary

| # | Severity | Hole | Units in conflict | Fix |
| --- | --- | --- | --- | --- |
| H1 | Critical | Socket ACL is per socket, not per method; any catalog caller can forge a Principal | import-pipeline (media/ingest) × alben-sichtbarkeit / oeffentliche-galerie | New AD-38: method rights by `SO_PEERCRED` |
| H2 | Critical | RLS bypassed by the owner role, missing `FORCE`, definer views, or an unset GUC that fails open | import-pipeline × datensicherheit-betrieb | New AD-39: RLS hardening |
| H3 | Critical | Public map clusters and geo searches run on exact coordinates despite the R8 GPS rule | ort-karte × oeffentliche-galerie | New AD-40: materialized public projection (geo) |
| H4 | Critical | Public search, filters, sort and cursor read fields outside the positive list | raster-suche × oeffentliche-galerie | AD-40 (search/sort part), tighten AD-31/AD-32 |
| H5 | Critical | Per-album exclusion cannot be expressed in the row-level `vis_public`; an excluded image shows in that album's public listing | alben-sichtbarkeit × oeffentliche-galerie | New AD-41: album-context filter |
| H6 | Critical | Recomputing visibility can race or lag; a narrowing change leaves `vis_public = true` | alben-sichtbarkeit × oeffentliche-galerie × pflege | New AD-42: synchronous, row-locked recompute |
| H7 | Critical | Aggregates and non-image entities (counts, covers, folder/place trees, tag facets, album rows) are outside RLS | raster-suche × ort-karte × alben-sichtbarkeit | New AD-43: audience rule for every read path |
| H8 | High | Missing graph edges (catalog → ingest, backup adapter → catalog) mean reindex and permanent delete cannot be built as the spine describes | datensicherheit-betrieb × import-pipeline × pflege | Tighten graph, AD-2, AD-13, AD-27 |
| H9 | High | Signed URLs: HMAC input unspecified, no `Cache-Control` rule, revocation takes up to about 2 h | oeffentliche-galerie × pflege | Tighten AD-12 |
| H10 | High | Derivative files have no version, so a late job overwrites a newer derivative | pflege × datensicherheit-betrieb | Tighten AD-13, AD-15 |
| H11 | High | R30 withdrawal built as exclusions; a later public album publishes the image again | oeffentliche-galerie × alben-sichtbarkeit | Tighten AD-18 |
| H12 | High | "No own cap" is ambiguous; a foreign album can *narrow* someone else's image | alben-sichtbarkeit × raster-suche | Tighten AD-18, AD-19 |
| H13 | High | Admin views (job errors, review list, audit, parked duplicates) show private content to the admin | datensicherheit-betrieb × ort-karte × plattform-zugang | New AD-44: content-free operations data |
| H14 | High | PITR restore revives withdrawn publications and deleted images | datensicherheit-betrieb × oeffentliche-galerie × pflege | New AD-45: privacy ledger and closed start after restore |
| H15 | High | EXIF stripping not specified; copyright/author in the shared derivative goes around the positive list | import-pipeline × oeffentliche-galerie | Tighten AD-36, decide R6/R7/R31 |
| H16 | Medium | Owner derived from the upload directory; path traversal and symlinks; SFTP owner cannot be resolved | plattform-zugang × import-pipeline | New AD-46 |
| H17 | Medium | Reserve rows visible before Confirm; ghost reservations block a SHA256; SHA256 register after permanent delete | import-pipeline × pflege | Tighten AD-2 |
| H18 | Medium | Geocoded place has no metadata layer (reindex wipes it); folder and geocoder places have different shapes | ort-karte × datensicherheit-betrieb | Tighten AD-6 |
| H19 | Medium | One `version` covers every purpose: derivatives regenerate on tag edits, stale `ReportLocation` wins | pflege × import-pipeline × ort-karte | Tighten AD-15 |
| H20 | Medium | Keyword tags vs user edits vs reindex | import-pipeline × pflege × datensicherheit-betrieb | Tighten AD-6 |
| H21 | Medium | "Admin has no exception" vs R29 admin managing foreign group content leads to a second "admin filter" | alben-sichtbarkeit × plattform-zugang | Tighten AD-18/AD-20 |
| H22 | Medium | Trash view needs to see trashed rows; an `include_trashed` flag becomes a bypass | pflege × raster-suche | Tighten AD-18/AD-20 |
| H23 | Medium | Existence oracles: 403 vs 404, duplicate response, error details | raster-suche × oeffentliche-galerie × import-pipeline | Tighten AD-29 |
| H24 | Low | Logs, OTel labels and access logs carry content | all | Tighten conventions |
| H25 | Low | Test data owner conflict (AD-21 vs epic-raster-suche); originals without pixels | import-pipeline × raster-suche | Tighten AD-21 |
| H26 | Low | FlatBuffers defaults vs "fields are absent" (R7) | oeffentliche-galerie × web | Tighten AD-30 |
| H27 | Low | Capture-time fallback shape (file timestamp) differs between filename and filter | import-pipeline × raster-suche | Tighten AD-7 |

---

## Critical

### H1: Per-socket ACL lets any catalog caller forge a Principal

- **Units:** epic-import-pipeline builds `media` and `ingest`, each with legitimate access to `catalog.sock` (for `ReportDerivatives` and `Reserve`/`Confirm`). epic-alben-sichtbarkeit and epic-oeffentliche-galerie build `SetVisibility`, `ListImages` and `PublishAlbum` on the same socket.
- **Compliant divergence:** AD-9 controls who may connect to a socket. AD-8 makes the Principal a field in the request, and AD-10 trusts it. Once `media` can connect, it can call `ListImages(principal={admin, all groups})` or `SetVisibility(... public)`, and nothing in the spine stops it. `media` (libvips on HEIC/TIFF) and `ingest` (Exiv2) parse untrusted files: they are the processes most likely to be compromised. Socket ACLs only work as a security boundary if they apply per method.
- **Fix (new AD-38):**
  > **AD-38 — Methodenrechte per Peer-Credential.** Jeder Service liest pro Verbindung `SO_PEERCRED` (uid) und prüft jede Methode gegen eine Allow-Liste im Vertrag (`contracts/proto/.../access.yaml`). Ein `Principal` wird nur von der uid der api angenommen; jede Methode mit `Principal` lehnt andere Aufrufer mit `PERMISSION_DENIED` ab. Service-Aufrufe (`Reserve`, `Confirm`, `Report*`) tragen keinen Principal, sondern eine Service-Identität aus der uid; ihre Felder (z. B. `owner_id`) sind nur von dieser uid zulässig. media und geocoder dürfen nur `Report*` aufrufen.

### H2: RLS can be bypassed while AD-20 is obeyed

- **Units:** epic-import-pipeline (sets up catalog schema, role and policies) × epic-datensicherheit-betrieb (published views for monitoring, reindex jobs).
- **Compliant divergences:**
  1. AD-3 gives one role per service and has that role run migrations. In PostgreSQL the table owner **bypasses RLS** unless `FORCE ROW LEVEL SECURITY` is set. "The read role has no `BYPASSRLS`" is then true and still useless.
  2. AD-3 allows "published views". By default a PostgreSQL view runs with the owner's rights, so it bypasses RLS. A monitoring view over `images` that the api reads (AD-1) returns every row.
  3. `SET LOCAL` outside an explicit transaction (autocommit) has no effect. A policy like `current_setting('gg.principal', true)` then returns NULL. If NULL is mapped to "anonymous" or "no restriction", the filter fails open.
  4. The materializer (AD-20 upkeep) needs to see all rows. Implementers will give it `BYPASSRLS` on the shared pool.
- **Fix (new AD-39):**
  > **AD-39 — RLS-Härtung.** catalog hat drei Rollen: `catalog_owner` (Migrationen, nie zur Laufzeit), `catalog_app` (`NOBYPASSRLS`, nicht Eigentümer), `catalog_maint` (Pflege der Publikumsspalten, eigener Pool, nur RPC-intern, nie für Leseantworten). Alle Bildtabellen haben `ENABLE` und `FORCE ROW LEVEL SECURITY`. Policies scheitern geschlossen: fehlt oder ist leer `gg.principal`, sieht die Abfrage nichts. Jede Leseabfrage läuft in einer expliziten Transaktion mit `SET LOCAL`. Jede View über Bildzeilen ist `security_invoker = true`; Views für andere Services enthalten keine Bildinhalte (AD-44). CI-Test: Abfrage ohne Principal liefert 0 Zeilen.

### H3: Public map clusters and geo searches use exact coordinates

- **Units:** epic-ort-karte (AD-33 quadkey per place, PostGIS radius/polygon/bbox) × epic-oeffentliche-galerie (R8: exact / coarsened / off).
- **Compliant leak:** AD-33 says clustering runs "under RLS". RLS filters rows, not columns. An anonymous cluster query that groups by the stored quadkey (zoom 18) puts a GPS = "off" image into a cluster at its true location. A count of 1 at zoom 18 gives its position to about 100 m. A coarsened image shows its exact point the same way. Radius and bbox search are worse: they let an attacker binary-search the exact point even if returned coordinates are coarsened, because the *filter* runs on the exact geography. The same applies to the Land/Region/Stadt filter when the place is not on the positive list.
- **Fix (new AD-40, geo part; tighten AD-33):**
  > **AD-40 — Öffentliche Projektion materialisiert.** catalog pflegt je Bild neben den Publikumsspalten die öffentliche Projektion: `pub_fields` (Bitmaske der Positivliste nach R7, Schnittmenge über die public-Alben, in denen das Bild nicht ausgeschlossen ist), `pub_gps_mode` (aus/vergröbert/exakt nach R8, gleiche Albenmenge), `pub_point` (exakt, auf das Raster der Vergröberung gerundet, oder NULL), `pub_quadkey` (aus `pub_point`, bei „vergröbert“ auf die Vergröberungsstufe gekürzt), `pub_place_*` (nur wenn in `pub_fields`). Der anonyme Principal clustert, sucht (Radius/Polygon/BBox) und filtert ausschließlich über diese Spalten; die exakten Spalten sind für ihn nicht lesbar (Spaltenrechte oder View ohne sie).
  >
  > AD-33 ergänzen: „Für den anonymen Principal gelten Cluster und Geo-Suche nur über `pub_quadkey`/`pub_point` (AD-40).“

### H4: Public search, filters, sort and cursor read hidden fields

- **Units:** epic-raster-suche (AD-32: one PGroonga column over title, description, keywords, camera, lens; AD-31 cursor = sort key + UUID) × epic-oeffentliche-galerie (R7 positive list).
- **Compliant leaks:**
  - *Search oracle:* Say description is not on the positive list. An anonymous user searches `Klinik` against the single search column and gets the image back. The hidden description has leaked bit by bit. Snippets and highlights taken from the column print it outright.
  - *Filter oracle:* Filtering by camera, author or tags when those fields are hidden.
  - *Cursor:* `nextCursor` = (sort key, UUID). If the public grid sorts by capture time and capture time is hidden, every cursor carries the hidden value of the last image.
- **Fix:** AD-40 additionally:
  > `pub_search` ist eine eigene PGroonga-Spalte nur aus den Feldern in `pub_fields`; Snippets stammen nur aus ihr. Anonyme Filter und Sortierungen sind nur über Felder in `pub_fields` zulässig, sonst `INVALID_ARGUMENT`.
  >
  > AD-31 ergänzen: „Cursor sind für den Client opak (mit dem Schlüssel der api authentifiziert verschlüsselt) und enthalten nur Sortierwerte, die der Principal sehen darf.“
  >
  > AD-32 ergänzen: „Für den anonymen Principal gilt nur `pub_search` (AD-40).“

### H5: Per-album exclusion does not fit a row-level `vis_public`

- **Units:** epic-alben-sichtbarkeit (album membership) × epic-oeffentliche-galerie (exclusion per public album, R5).
- **Compliant leak:** AD-18 says "exclusion per public album applies only to that album's public path". Image X is in public album A (excluded) and public album B (not excluded), so AD-20 correctly sets `vis_public = true`. The anonymous listing of **album A** filters `vis_public AND member_of(A)`, and X appears in A, where the owner explicitly excluded it. Everything else is correct, so the leak is hard to spot.
- **Fix (new AD-41):**
  > **AD-41 — Albumkontext.** Jede Abfrage im Kontext eines Albums filtert zusätzlich über die Mitgliedschaftszeile: für den anonymen Principal `membership.excluded = false AND album.public`. Für andere Principals gilt die Albumsicht nur, wenn das Album selbst für sie sichtbar ist. Die Mitgliedschaftstabelle steht unter RLS wie Bildzeilen.

### H6: Recomputing visibility races or lags

- **Units:** epic-alben-sichtbarkeit (album and membership changes) × epic-oeffentliche-galerie (unpublish, withdraw) × epic-pflege (trash, edits).
- **Compliant leaks:**
  - *Lag:* Publishing or unpublishing an album with 100,000 images updates 100,000 rows. To protect the performance targets (AD-17, AD-22), a unit makes this asynchronous through its outbox. AD-20 only says "on every change" and sets no time bound. Unpublishing leaves the images public for minutes.
  - *Lost update:* Two transactions in parallel remove image X from public album A and from public album B. Each recomputes X from a snapshot that still contains the other album, so `vis_public` stays `true` with no public album left.
- **Fix (new AD-42):**
  > **AD-42 — Neuberechnung der Sichtbarkeit.** Publikums- und Projektionsspalten (AD-20, AD-40) werden aus dem Gesamtzustand des Bildes neu berechnet, nie inkrementell. Die Neuberechnung sperrt die betroffenen Bildzeilen (`FOR UPDATE`, sortiert nach `image_id`) in derselben Transaktion wie die auslösende Änderung. Einengungen (Unpublish, Ausschluss, Zurücknehmen, Grenze senken, Papierkorb, Gruppe entfernen) sind synchron: die RPC antwortet erst nach Commit. Nur Erweiterungen dürfen asynchron nachlaufen. Große Einengungen setzen zuerst ein Album-Gate (`album.public = false`), das AD-41 sofort wirksam macht, und berechnen danach in Batches. Ein Prüfjob vergleicht regelmäßig materialisiert mit neu berechnet und meldet Abweichungen.

### H7: Aggregates and non-image entities are outside RLS

- **Units:** epic-raster-suche (counts, tag facets), epic-ort-karte (place tree and folder tree with counts, R19), epic-alben-sichtbarkeit (album cards: title, count, cover).
- **Compliant leaks:** AD-20 covers **image rows**. Folders, places, tags, albums and every count are separate entities or aggregates. To keep within 150/300 ms at 1 million images, units will keep global counters (`folder.image_count`, `tag.usage_count`, `place.count`) or show `meta.count` from estimates. The results:
  - a folder named "Privat/Klinik" appears in everyone's folder tree;
  - a tag created only on private images ("Scheidung") appears in autocomplete for everyone, including anonymous users, because R16 makes tags "globally visible";
  - album counts include foreign images that viewers cannot see;
  - an album cover is chosen from the album row, not from an image visible to the viewer, so a private image becomes the public cover.
- **Fix (new AD-43):**
  > **AD-43 — Jeder Lesepfad hat ein Publikum.** Ordner, Orte, Tags und Alben erscheinen einem Principal nur, wenn mindestens ein für ihn sichtbares Bild sie trägt (Alben zusätzlich nach eigener Albumsichtbarkeit). Zähler, Facetten, Cover, Cluster und `meta.count` werden unter RLS des anfragenden Principal berechnet oder aus Aggregaten je Publikum (`public`, `authenticated`, je `group_id`, je `owner_id`), die AD-42 mitpflegt; globale Zähler sind verboten. Ein Cover ist immer ein Bild, das der Principal sehen darf; für den anonymen Principal gilt AD-40. Tag-Namen sind global eindeutig, aber nur über sichtbare Bilder auffindbar.

---

## High

### H8: Missing graph edges for reindex, permanent delete and backup status

- **Units:** epic-datensicherheit-betrieb (reindex R23, backup status AD-27) × epic-import-pipeline (ingest owns Exiv2 extraction and `originals/`) × epic-pflege (permanent delete).
- **Compliant divergence:** The graph has only ingest → catalog, never catalog → ingest. So:
  - *Reindex:* "Metadata re-read from the original" cannot reach ingest. datensicherheit-betrieb will build a second Exiv2 extractor inside catalog. Two extractors produce different `raw` shapes and different capture times, so filename (AD-7/R10) and DB drift apart.
  - *Permanent delete:* AD-13 says removal is "triggered by catalog", but only ingest writes `originals/` (files `0440`, owned by the ingest user). catalog cannot instruct ingest. pflege will either delete as catalog (breaking AD-13) or never delete the original (breaking R26).
  - *Backup adapter:* AD-27 says an adapter "reports" status to catalog, but the adapter is not in the graph and has no socket. epic-datensicherheit-betrieb will write directly into `catalog`, breaking AD-2/AD-3.
- **Fix:** add graph edges `catalog -->|Outbox → gRPC| ingest` and `backup-adapter -->|gRPC ReportBackup| catalog`.
  > AD-2 ergänzen: „ingest ist der einzige Extraktor. `Confirm` trägt das vollständige `RawMetadata` (Vertrag in `contracts/proto/geogallery/ingest/v1/`). Reindex bestellt catalog per Outbox `Reextract(image_id, path, meta_rev)` bei ingest; ingest antwortet mit `ReportRawMetadata`.“
  >
  > AD-13 ergänzen: „Endgültiges Löschen eines Originals bestellt catalog per Outbox `DeleteOriginal(image_id, path, sha256)` bei ingest; ingest prüft den sha256 vor dem Entfernen.“
  >
  > AD-27 ergänzen: „Der Backup-Adapter ist eine eigene Unit mit eigenem Unix-User und ruft nur `catalog.ReportBackup`.“

### H9: Signed URLs: what is signed, caching, revocation

- **Units:** epic-oeffentliche-galerie (withdraw, R30) × epic-pflege (trash, R26) × epic-plattform-zugang (h2o configuration).
- **Compliant leaks:**
  - AD-12 does not say what the HMAC covers. A unit signs only `image_id + exp`; another lets the size or path suffix be chosen freely, or the h2o rule only checks the prefix.
  - Expiry is rounded to a fixed window, so a URL is valid for up to 2 windows (about 2 h). After trash, unpublish or withdrawal, the image can still be fetched publicly. Because the window is aligned, every viewer gets the same URL.
  - No `Cache-Control` rule exists. A unit sets `public, max-age=31536000` for the browser cache, so any proxy or CDN keeps private derivatives indefinitely.
  - Any authenticated user can hand a URL to someone without an account: bearer semantics, which R5 does not say is allowed.
- **Fix (tighten AD-12):**
  > „HMAC über den exakten Pfad inkl. Größe und `derivative_rev` (AD-13) und `exp`; h2o vergleicht den vollen Pfad. Ableitungen ohne gültige Signatur liefern 404; kein anderes Mapping auf `/storage/derivatives`. Antworten tragen `Cache-Control: private, max-age=<Restlaufzeit>`; nur Bilder mit `vis_public` dürfen `public` tragen. Kein geteilter Cache/CDN in V1. Maximale Wirkverzögerung eines Entzugs = 2 × Fenster; Papierkorb, Zurücknehmen und endgültiges Löschen bestellen zusätzlich bei media eine Umbenennung auf neues `derivative_rev`, damit alte URLs sofort 404 liefern. Weitergabe einer URL durch Angemeldete bis Ablauf ist bewusst akzeptiert.“

### H10: Derivative files have no version; a late job overwrites a newer derivative

- **Units:** epic-pflege (copyright change → regenerate) × epic-datensicherheit-betrieb (reindex all → regenerate). Both bump `version` (AD-15). Several media workers run in parallel (AD-17).
- **Compliant divergence:** AD-13 sets the path `derivatives/<2>/<2>/<image_id>_<größe>.avif`, without a version. Job v2 (old copyright) finishes after v3 (new copyright) and overwrites the file. `ReportDerivatives(v2)` is idempotent and catalog ignores it, but the file on disk now carries the old copyright. Browsers also keep serving the old copyright for the cache window.
- **Fix:**
  > AD-13 ändern: „`derivatives/<2>/<2>/<image_id>_<derivative_rev>_<größe>.avif`. media schreibt in eine temporäre Datei und benennt nur um, wenn `derivative_rev` ≥ dem im Ableitungsregister (unter Zeilensperre); ältere Revisionen werden nach Quittung gelöscht.“
  >
  > AD-12 signiert nur die aktuelle Revision.

### H11: R30 withdrawal as exclusions publishes again later

- **Units:** epic-oeffentliche-galerie ("withdraw from all public albums in one step") × epic-alben-sichtbarkeit (adding to albums).
- **Compliant divergence:** oeffentliche-galerie builds the withdrawal as "set exclusion in every current public album". That is the obvious reading of R30 and fits AD-18. Next week someone adds the image to a new public album, or publishes an existing album that contains it, and the image is public again, although the owner withdrew it on purpose.
- **Fix (tighten AD-18):**
  > „Zurücknehmen (R30) setzt die eigene Obergrenze des Bildes auf höchstens ‚alle angemeldeten‘ und bleibt bestehen, bis der Owner sie wieder anhebt; Ausschlüsse sind albumbezogen und kein Ersatz.“

### H12: The "no own cap" sentinel is ambiguous; foreign albums narrow visibility

- **Units:** epic-alben-sichtbarkeit (computes effective visibility) × epic-raster-suche (default visibility from import-pipeline).
- **Compliant divergences:**
  - AD-18: default "no own limit". AD-19: "without an own limit ≥ public, no foreigner makes an image public". One unit reads "no limit" as unlimited, so a foreign public album publishes my image. The other reads it as "authenticated", so for *own* albums the owner must raise the cap explicitly before publishing. The rule cannot be computed without knowing **whose** album it is, and AD-20's columns do not say so.
  - *Narrowing (griefing):* "Without album and without cap: all authenticated." "Over several albums the widest album wins." User B adds my image (default visibility) to B's group album G. Widest album = G, so my image is no longer visible to everyone else. With a private album it is invisible to everyone except me. Who does "privat (nur Owner)" mean on an album: the album owner or the image owner?
- **Fix (tighten AD-18/AD-19):**
  > „Die Obergrenze hat die Werte `unset | private | groups(…) | authenticated | public`. `unset` gilt für Alben des Bild-Owners als `public`, für fremde Alben als `authenticated`. Die Basis jedes Bildes ohne eigene Grenze ist ‚alle angemeldeten‘; Alben können nur erweitern (bis zur Grenze), nie unter die Basis einengen. Nur die eigene Grenze des Owners engt ein. ‚Privat‘ an einem Album heißt: nur der Album-Owner sieht das Album; ein fremdes Bild darin sieht er nur, soweit er es ohnehin sehen darf.“

### H13: Admin and operations views show private content

- **Units:** epic-datensicherheit-betrieb (job queues, parked duplicates, metrics; AD-1 "api aggregates published views") × epic-ort-karte (review list R18 in the admin UI) × epic-plattform-zugang (audit plugin).
- **Compliant leaks:** R29 says "an admin never sees anything private". Yet all of the following show private data to the admin:
  - AD-16 job views carry "last error": free text such as `failed to decode /storage/originals/Privat/Arzt/2024-03-01_101500.heic` or `geocoding 48.137,11.575 failed`.
  - The R18 review list shows image, folder and GPS for private images too.
  - `drogon_audit_plugin` logs request bodies, for example the private title in a metadata edit.
  - The parked-duplicates view shows the duplicate's target image (title, path, owner) without a visibility check, because ingest has no filter.
  - Outbox rows and `media`/`geocoder` jobs keep coordinates, paths and copyright indefinitely, even after permanent delete (AD-5 only covers "dependent entities").
- **Fix (new AD-44):**
  > **AD-44 — Betriebsdaten sind inhaltsfrei.** Job-Views (AD-16), Outbox, Audit und Fehlerfelder enthalten nur IDs, Codes und Zähler, keine Pfadbestandteile unterhalb von `originals/`, keine Koordinaten, keine Metadatenwerte; Fehlertext stammt aus einer geschlossenen Code-Liste. Admin-Ansichten lösen Bilddetails per catalog mit dem Admin-Principal auf (RLS); nicht sichtbare Bilder erscheinen als ‚privates Bild‘ mit ID, ohne Vorschau. Die Prüfliste (R18) und Duplikatansicht liefert catalog unter dem Principal des Betrachters; `Reserve` → `DUPLICATE` liefert nur die bestehende `image_id`. Erledigte Jobs und quittierte Outbox-Einträge werden nach N Tagen gelöscht; endgültiges Löschen entfernt sie sofort in allen Services (AD-5).

### H14: Restore and PITR revive withdrawn publications and deleted images

- **Units:** epic-datensicherheit-betrieb (PITR restore R24) × epic-oeffentliche-galerie (withdraw R30) × epic-pflege (permanent delete R26).
- **Compliant leak:** The DB is restored to time T. Everything withdrawn, unpublished or narrowed after T is public again. Images permanently deleted within 90 days come back from the backup. R30 (the consent case) is broken by the recovery process. Also: originals have RPO 0 but the DB has PITR, so images imported after T exist only as files. That leaves orphan originals, and their SHA256s are not in the register.
- **Fix (new AD-45):**
  > **AD-45 — Restore wahrt Einengungen.** catalog schreibt jede Einengung und jedes endgültige Löschen zusätzlich in ein append-only Privacy-Ledger, das wie Originale mit RPO 0 gesichert wird. Nach einem Restore startet die öffentliche Galerie geschlossen (Wartungsseite, R25), bis das Ledger nach T erneut angewendet ist. Ein Abgleich `originals/` ↔ SHA256-Register importiert verwaiste Originale über ingest neu und meldet sie dem Admin.

### H15: EXIF in derivatives; copyright/author bypass the positive list

- **Units:** epic-import-pipeline (media, libvips → img2avif) × epic-oeffentliche-galerie (R7).
- **Compliant leaks:**
  - By default libvips keeps EXIF, XMP and ICC when saving, and AVIF supports EXIF/XMP boxes. AD-36 says "only copyright as metadata" but does not say how stripping is guaranteed or tested. Thumbnails made by a second code path (for example `vips_thumbnail`) keep GPS.
  - R6 requires one derivative for internal and public use. R31 puts "© <Jahr> <Autor>" into it. If "author" is not on the positive list, the public derivative still exposes the author name. This is a requirement conflict that the spine does not decide.
- **Fix:**
  > AD-36 ergänzen: „media speichert mit `keep=none` (nur ICC zulässig), wendet die EXIF-Orientierung vorher an und schreibt danach ausschließlich das Copyright-Feld. Ein CI-Test prüft jede Größe aller sechs Formate auf Abwesenheit von EXIF/XMP/IPTC außer Copyright.“
  >
  > Neue Festlegung (Entscheidung nötig): „Das Copyright der Ableitung ist immer öffentlich und steht außerhalb der Positivliste; die Veröffentlichungsbestätigung (R30) zeigt es an.“
  >
  > Alternative: eine Fallback-Kette ohne Autor bei öffentlichen Bildern, was eine zweite Ableitung bräuchte und damit R6 widerspricht.

---

## Medium

### H16: Upload ownership, path and symlinks

- **Units:** epic-plattform-zugang (accounts, usernames) × epic-import-pipeline (`uploads/<username>/<Pfad>`, SFTP watcher).
- **Divergence/leak:** The SFTP watcher has no Principal and no edge to iam, so it derives the owner from the directory name. A path the user chooses (`../B/...`) or a renamed username puts images under someone else's account, which gives them that owner's visibility defaults. Symlinks in the upload let ingest read host files.
- **Fix (new AD-46):**
  > **AD-46 — Upload-Herkunft.** Der Owner eines Imports stammt aus dem Principal des Web-Uploads bzw. aus einer von api provisionierten Zuordnung `upload_dir → user_id` in ingest, nie aus dem Verzeichnisnamen allein. Pfade werden mit `openat2(RESOLVE_BENEATH | RESOLVE_NO_SYMLINKS)` geöffnet; `..`, absolute Pfade und Symlinks werden abgelehnt.

### H17: Reserve and Confirm lifecycle

- **Units:** epic-import-pipeline × epic-pflege.
- **Divergence/leak:**
  - `Reserve` creates a row with the default visibility. Before `Confirm` and before the owner can set it to private, the row is visible to everyone, without derivatives (broken URLs).
  - A reservation without `Confirm` (file gone, crash) blocks that SHA256 forever: later uploads are parked as duplicates of a ghost.
  - Nothing says whether a permanently deleted image releases its SHA256.
- **Fix (tighten AD-2):**
  > „Bildzeilen haben `state = reserved | active`; RLS zeigt nur `active`. `Reserve` trägt die beim Upload gewählte Anfangsgrenze. Reservierungen ohne `Confirm` verfallen nach Frist; ingest bekommt dann erneut `ACCEPTED`. Endgültiges Löschen gibt den sha256 frei; geparkte Duplikate darauf werden dem Owner zum Import angeboten.“

### H18: Geocoded place has no metadata layer and no shared shape

- **Units:** epic-ort-karte (geocoder → `ReportLocation`, places from folders) × epic-datensicherheit-betrieb (reindex replaces `raw`).
- **Divergence:**
  - AD-6 has only raw/override/effective. ort-karte stores the geocoded place in `raw`, and reindex ("only reindex replaces raw") then wipes it until the image is geocoded again, which costs provider quota.
  - The geocoder returns "Germany/Bavaria" and the folder says "Deutschland/Bayern". Two places appear in the R19 tree, and the R18 mismatch check fires on every image.
- **Fix (tighten AD-6):**
  > „Ort hat die Ebenen `override > geocoded > folder (ungefähr)`. `geocoded` gehört dem Ortsmodell, nicht `raw`; Reindex bestellt Geocoding nur bei geänderten Koordinaten. Orte haben einen kanonischen Schlüssel (ISO-3166-Codes für Land/Provinz, Anbieter-unabhängige ID darunter) und Anzeigenamen je Sprache; Ordnernamen werden auf diesen Schlüssel abgebildet, bevor verglichen wird.“

### H19: One `version` covers every purpose

- **Units:** epic-pflege (tags, rating, title) × epic-import-pipeline (media) × epic-ort-karte (geocoder).
- **Divergence:** AD-15 has `image_id + version`. pflege increments it on every edit, so media re-encodes on every tag change: CPU load during imports. A late `ReportLocation` for old coordinates is accepted because the version also moved for unrelated reasons.
- **Fix (tighten AD-15):**
  > „Getrennte Revisionen: `derivative_rev` (effektives Copyright, Reindex, Orientierung), `geo_rev` (Koordinaten), `meta_rev` (Reextraktion). Empfänger quittieren je Revision; catalog verwirft Reports mit veralteter Revision.“

### H20: Keyword tags vs user edits vs reindex

- **Units:** epic-import-pipeline (keywords → tags) × epic-pflege (user removes a tag) × epic-datensicherheit-betrieb (reindex).
- **Divergence:** AD-6 covers typed fields, not tag sets. The user removes the keyword tag "Ex", and reindex adds it again.
- **Fix (tighten AD-6):**
  > „Tags je Bild: `raw` (aus Keywords), `added`, `removed` (Grabsteine); effektiv = (raw − removed) ∪ added. Reindex ersetzt nur `raw`.“

### H21: Admin "no exception" vs R29 admin management of foreign content

- **Units:** epic-alben-sichtbarkeit × epic-plattform-zugang (roles).
- **Divergence:** R29 lets the admin change group and public visibility of foreign content. AD-18 says "Admin has no exception", so an admin who is not in group G cannot see G-images and cannot manage them. The unit builds an "admin mode" with a second filter, which goes against AD-20.
- **Fix (tighten AD-20):**
  > „Für Principals mit `system.admin` gibt es genau eine zweite Policy `admin_manage`: Zeilen, die nicht privat sind (`vis_authenticated OR vis_public OR cardinality(vis_group_ids) > 0`), nur für Verwaltungs-RPCs, nie für Raster/Suche/Karte. Privat bleibt für Admins unsichtbar.“

### H22: Trash view becomes a bypass

- **Units:** epic-pflege (trash, restore) × epic-raster-suche (shared list RPC).
- **Divergence:** AD-18 says the trash hides images from everyone. A restore view needs to see them, so pflege adds `include_trashed = true` to the list RPC and forgets the remaining vis predicate.
- **Fix (tighten AD-18):**
  > „Die Papierkorbansicht zeigt Zeilen, die der Principal ohne `trashed` sehen dürfte und für die er `image.delete` hat; eigene Policy, kein Flag am allgemeinen List-RPC.“

### H23: Existence oracles

- **Units:** epic-raster-suche × epic-oeffentliche-galerie × epic-import-pipeline.
- **Leak:** `GET /api/v1/images/{id}` returns 403 for private images and 404 for unknown ones. gRPC `detail` texts pass through. The duplicate response reveals that someone already owns this exact file. UUIDv7 reveals the import time.
- **Fix (tighten AD-29):**
  > „Nicht sichtbar und nicht vorhanden liefern identisch 404 mit gleichem `code`; gRPC-Details werden nie durchgereicht.“

  Also document as accepted: duplicate detection reveals that a file exists (global SHA256, R12), and the UUIDv7 timestamp is the import time.

---

## Low

- **H24 Logs and telemetry.** Logs, OTel labels and h2o access logs contain search terms, paths and coordinates, and the OTel backend is deferred (it could be external). Add to the conventions: "Logs and metric labels follow AD-44 (IDs and codes only; no `image_id` as a metric label). Access logs drop the query string and the signature. The OTel backend runs locally or receives scrubbed data only."
- **H25 Test data owner.** AD-21 says "from the first epic", but epic-raster-suche says it creates the generator. 1 million `Confirm` calls without pixels produce 1 million failed media jobs and broken URLs. Fix: "The generator belongs to epic-import-pipeline. Images without pixels point to a shared placeholder original and carry the flag `synthetic`, and media renders them as a placeholder."
- **H26 FlatBuffers defaults.** A hidden scalar field (rating, lat/lon) reads as 0 instead of "absent". Fix in AD-30: "Fields outside the public projection are optional (`optional` scalars / missing tables), and the JSON omits the key."
- **H27 Capture-time fallback.** R10 takes the filename from the file timestamp. One unit also writes that value into the capture-time column and the other leaves it NULL, so year filters disagree. Fix in AD-7: "The file timestamp is used only for the filename. `effective.capture_time` stays NULL and `zeit_unsicher = true` until it is corrected."

## Pairs checked and found sound

- Sessions and the Principal (AD-8): one resolver. Group membership travels in the Principal, so catalog needs no copy of iam groups.
- SHA256 uniqueness (AD-2): one authority with a unique constraint. Parallel `Reserve` calls cannot both win.
- Name collisions in `originals/` (AD-13): `RENAME_NOREPLACE` is race-free.
- Error shapes (AD-11/AD-29) and ID types (AD-4): no room for divergence.
