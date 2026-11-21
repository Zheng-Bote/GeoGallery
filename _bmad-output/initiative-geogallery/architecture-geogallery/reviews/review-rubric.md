# Rubric Review — Architecture Spine GeoGallery

- Reviewed: `architecture-geogallery.md` (status draft, updated 2026-11-21)
- Against: `initiative-geogallery.md` (R1–R31, Constraints, Done when), 8 epics in `tickets/`, `.memlog.md` for decision status
- Date: 2026-11-21

## Verdict

The spine is strong on the data, contract, visibility and job invariants: those are the real divergence points for the 8 epics, and most Rules can be enforced. It is **not ready to finalize** because the delivery envelope (environments, CI, how a release is assembled from multiple repos, rollout without a maintenance window) is mostly left unstated. There are also three internal contradictions (SFTP vs. `uploads/` write rule, monitoring vs. the call graph, RLS vs. "one role per service"), and several decisions are marked [ADOPTED] even though the memlog records them as assumptions.

## Findings

### F1 — HIGH — Deployment & environments dimension largely silent

The altitude owns the operational envelope. Epic 1 (`epic-plattform-zugang`) must deliver "CI und Deployment auf den Host", and every epic's Done-when says "läuft deployt". The spine fixes the runtime shape (AD-25 Podman/Quadlet, AD-37 PG container). It says nothing on these points:
- **Environments:** no dev / test / benchmark / prod split. The 1 Mio. benchmark (AD-21) and production run on the same host or on different ones? Does local dev use the same Quadlets?
- **CI:** CI is mentioned only as "`buf breaking` in CI" (AD-34). The spine names no CI provider and no pipeline contract per repo (build, test, contract check), and no artifact or image registry.
- **Release composition:** services live in separate repos (AD-34), but all of them ship in one `geogallery` container image (AD-25). Nothing says who builds that image, how service versions are pinned into it (a manifest or lockfile of repo tags), or how the frontend build reaches `h2o`.
- **Rollout:** see F2.

Two epics will make these choices independently: Epic 1 creates the pipeline and later epics add repos to it.
**Fix:** add an AD "Build, Release & Environments". It should fix: environments (dev, bench, prod) and what differs between them (only config/profile, per AD-22); one CI contract per repo (build, unit tests, `buf breaking`, publishing an image or artifact with a version tag); one release manifest in the main repo (`deploy/release.toml` or similar) that pins each service tag, the contracts tag and the web build; and image build from that manifest. If the user won't decide now, record it as an explicit Open Question owned by epic-plattform-zugang, with the manifest invariant fixed in the spine.

### F2 — HIGH — Rollout contradicts the container shape (99,5 % ohne Wartungsfenster)

AD-28 assumes "rollierender Neustart mit gemischten Versionen". AD-25 packs all seven services into a single `geogallery` container image. Replacing that image restarts all services at once, so versions never actually mix, and every release is a full outage of api plus services. Rollout is not decided at all: replace the container in place, swap binaries per service inside the container, or run blue/green with a second container on the api socket. The H2O maintenance page (R25) would cover the gap, but nothing ties the two together.
**Fix:** decide the rollout unit. One option is per-service binary updates inside a long-lived container, via `systemctl restart <service>` with socket activation holding connections. The other is container replacement with a stated outage budget per release that is counted against the 99,5 %. State that the maintenance page is the planned behaviour during that window. Then either keep AD-28 as written or reword it to match.

### F3 — HIGH — AD-13 forbids what R9 requires (SFTP writes into `uploads/`)

AD-13: "`uploads/` und `parked/`: nur ingest schreibt". But under R9 the admin uploads via SCP/SFTP (host OpenSSH, outside the containers) directly into `/storage/uploads/<username>/…`. Web upload goes api → ingest, so that path is fine. Two things are missing: how ingest detects SFTP arrivals "ohne weiteren Schritt" (inotify/fanotify, a polling scan, a quiescence or "file complete" rule), and which Unix user or group owns the files. epic-import-pipeline and epic-datensicherheit-betrieb (bulk import of 200.000 files) will each pick an answer.
**Fix:** amend AD-13: "`uploads/<username>/` is written by ingest (web) and by the host SFTP account of that user (chroot, group `gg-upload`). ingest alone moves or deletes there. A file counts as complete after <rule>." The detection mechanism can be left to the epic, but the completeness rule and permissions must be in the spine.

### F4 — MEDIUM — Call graph (AD-9) blocks the monitoring AD-1 promises

AD-1 says monitoring is aggregated by api "aus gRPC Health, OTel und veröffentlichten Views". AD-9 restricts socket access to exactly the arrows in the graph. That graph has no api → media, api → geocoder or api → webauthn edge, so api can't reach their gRPC Health. The same applies to admin worker status (R23) when it isn't served purely from DB views. The graph also has no DB edges, so it never says which roles may read which published views (api has only the `audit` schema).
**Fix:** either add health-only edges (api → all services, `grpc.health.v1` only) or state that health is read via systemd/OTel and not over gRPC. Also add a small table "published views → allowed reader role" (e.g. `*.jobs_v` → api).

### F5 — MEDIUM — RLS rule cannot be enforced as written with AD-3's single role

AD-3 says each service gets one schema and one role. AD-20 says "die Leserolle hat kein `BYPASSRLS`". PostgreSQL table owners bypass RLS unless the table has `FORCE ROW LEVEL SECURITY`. If catalog has only one role and that role owns the tables (it also runs migrations, AD-3), RLS is silently off and the fail-closed rule fails open.
**Fix:** in AD-3/AD-20, require per service a migration/owner role separate from the runtime role. catalog's runtime role is non-owner and has no `BYPASSRLS`; also require `FORCE ROW LEVEL SECURITY` on all image-bearing tables. Add a CI or startup check that asserts this.

### F6 — MEDIUM — Decision status inflated (assumptions marked [ADOPTED])

The memlog marks several items as `[ASSUMPTION]`, but the spine shows them as `[ADOPTED]`:
- AD-23 (language-neutral contracts): "Bestätigung ausstehend".
- AD-2 Reserve/Confirm, AD-15 "Submit trägt alle Eingaben", the HMAC-secret placement in AD-12, and the catalog-owned settings in AD-1: all from memlog item 100, "dem Nutzer zur Bestätigung vorzulegen".

AD-10, AD-28 and AD-33 correctly carry no marker. Epics will treat [ADOPTED] as settled.
**Fix:** drop the [ADOPTED] tag on AD-23 and on the sub-rules from item 100 until the user confirms them, or get confirmation and log it in the memlog.

### F7 — MEDIUM — Ticket and epic conflicts are not recorded in the spine

The user decided three deviations from the ticket, and the memlog flags each one "→ Sofia":
- FlatBuffers api ↔ frontend (AD-30) vs. ticket Constraint and Boundary "keine FlatBuffers / Verworfen: FlatBuffers".
- PostgreSQL 18 (AD-37, Stack) vs. Constraint "PostgreSQL 19".
- Podman containers (AD-25) vs. Constraint "Komponenten als systemd-Units". epic-plattform-zugang's Done-when literally says "als systemd-Units (H2O, Drogon-API, PostgreSQL/PostGIS)".

The spine reads as if no conflict exists, so a reader holding the ticket sees a contradiction.
**Fix:** add a short "Abweichungen von der Initiative" list (deviation, decision date, pending ticket update), or make sure the ticket and epic-plattform-zugang are updated before the spine leaves draft.

### F8 — MEDIUM — AD-19 is ambiguous for the default "keine eigene Grenze"

AD-18 sets the default image cap to "keine eigene Grenze" (unbounded). AD-19 says a foreign album opens an image only up to its cap. It also says that without an owner cap ≥ public no foreigner makes the image public. Read literally, "no cap" means unbounded, so a foreign public album *would* publish it. The second sentence implies the opposite. epic-alben-sichtbarkeit and epic-oeffentliche-galerie will each pick one reading, and that difference is a privacy leak.
**Fix:** state it explicitly. For example: "For albums not owned by the image owner, 'keine eigene Grenze' counts as cap = alle angemeldeten. Only an explicit owner cap of public lets a foreign album publish the image."

### F9 — MEDIUM — Backup/restore invariants that cross epics are missing

- AD-27 lets the hosting profile decide "ob der Import auf die Sicherung wartet". If it doesn't wait, RPO 0 (R24) is not guaranteed, and the spine doesn't tie this to the redundant volume (AD-13) as the RPO-0 argument. Pick one: RPO 0 = mirrored volume plus asynchronous backup, or RPO 0 = import waits for backup.
- The 90-day backup retention after final delete (ticket Notes; epic-pflege Done-when) is not a requirement on the backup adapter. epic-pflege tests it and epic-datensicherheit-betrieb builds it.
- Restore onto an empty host (≤ 24 h) needs the Podman-secret key from AD-26 and the HMAC key. The spine never says that this key material must be escrowed outside the host, so a restore could find the secrets file unreadable.
- `parked/` and the `audit` schema are not stated as in or out of backup scope.

**Fix:** extend AD-27 with: the RPO-0 mechanism, retention ≤ 90 d after final delete as an adapter obligation, key escrow as a restore precondition, and backup scope (originals, parked, PG incl. audit; not derivatives).

### F10 — MEDIUM — Test-dataset rule (AD-21) does not match the epic order

AD-21 puts the generator in the repo "ab dem ersten Epic", running "über den catalog-Importpfad". But epic 1 (plattform-zugang) has no catalog, and epic-raster-suche says it produces the dataset. "Über den Importpfad" at ≥ 10.000/h means ~100 h for 1 Mio. "Pixel nur für eine Stichprobe" together with AD-2's Reserve/Confirm (path to a real original) leaves open whether the generator goes through ingest or calls catalog RPCs with synthetic paths and placeholder derivatives. The memlog says "Platzhalter-Ableitungen", but the spine doesn't.
**Fix:** say "ab epic-import-pipeline". State that the generator calls catalog's Reserve/Confirm (bypassing ingest file moves) with synthetic paths and that media registers placeholder derivatives. Say who owns the generator (epic-raster-suche, per the tickets).

### F11 — MEDIUM — Permission vocabulary has no contract home

AD-10 makes each data owner check RBAC permissions (`image.*`, `album.publish`, `image.delete`, `system.admin`, …). iam owns roles and permissions. Several epics will introduce permission strings: pflege (`image.delete`), oeffentliche-galerie (`album.publish`), datensicherheit-betrieb (admin). There is no single registry, so names and granularity can drift.
**Fix:** add `contracts/permissions/` (or a section of the OpenAPI/proto contracts) as the closed list of permission identifiers and default-role mapping, owned by iam. Services may only check listed identifiers.

### F12 — LOW — Minor items

- AD-12: the revocation window (up to ~1 h, plus up to one rounding window) is a deliberate trade-off against R30's "in einem Schritt zurücknehmen", but it is only in the memlog. State "Entzug wirkt spätestens nach Fensterende" in the Rule so epic-oeffentliche-galerie doesn't try to "fix" it.
- Rate limiting is placed in api (AD-8/AD-10). epic-plattform-zugang lists it under H2O. Say which one (or both, with which limits where).
- The Deferred item "Session-Cache … muss spätestens nach 60 s wirken" is a binding rule (R1). Move it into AD-8.
- Deferred "Größen und Qualität der Ableitungen: epic-import-pipeline" is safe only because sizes are referenced via catalog-issued URLs. Add "size names are part of the api `.fbs`/OpenAPI contract" so raster-suche and oeffentliche-galerie don't hard-code sizes.
- OTel collector location (which container) is unstated. Cheap to fix in AD-25.

## Checklist results

| Criterion | Result |
| --- | --- |
| Fixes real divergence points for the 8 epics | Mostly yes: data ownership, single writer, IDs, metadata layers, visibility (rule + single filter), job form, outbox, signed URLs, storage layout, API envelope, pagination, search, map keys, contracts location. Missing: permissions registry (F11), SFTP intake (F3), release composition (F1). |
| Every Rule enforceable and prevents its divergence | Mostly. Exceptions: AD-20 with AD-3 (F5), AD-9 with AD-1 (F4), AD-19 ambiguity (F8), AD-28 with AD-25 (F2). |
| Deferred items cannot cause divergence | Mostly safe. Derivative sizes need the contract note (F12). Session-cache 60 s is a rule, not deferred. Backup target is fine only once F9 obligations are in AD-27. |
| Named tech verified-current | Plausible per memlog stack check of 2026-11-21: Drogon 1.9.13, PostGIS 3.6.3, PGroonga 4.0.8, libvips 8.18, libavif 1.4.2, Angular 22, Podman 6.1. Flagged: H2O has no stable release since 2.2.6 (beta/master pin, an operational risk that should be accepted explicitly); gRPC and FlatBuffers are unpinned ("beim Bau festnageln", acceptable as first-epic checkpoint); webauthn-rs 0.5.5 vs 0.6.x open; PG 18 chosen over the ticket's PG 19 (F7). |
| Covers every ticket requirement | R1–R31 all appear in the Capability map. Partial: R9 SFTP path (F3), R24 RPO 0 / retention / restore keys (F9), R30 revocation window (F12), R25 tied to rollout (F2). Constraints: TLS 1.3, CSP, XSS are not placed in any AD (presumably h2o/web; one line in AD-25 or Conventions would do). |
| Every owned dimension decided/deferred/open | Missing: environments, CI, release, rollout (F1, F2). Infra/provider strategy is covered (single host, Podman, hosting-specific backup adapter, profiles AD-22). Operations are partly covered (logs, metrics, secrets, PG upgrade); availability measurement has no owner or mechanism. |
| Valid mermaid | All three diagrams (flowchart LR, flowchart TB with subgraphs, erDiagram) look syntactically valid on manual check (not rendered; no mmdc available). |
| No template comments/placeholders | None found (no TODO/TBD/`{{`/`<!--`). `…` in AD-14 is a deliberate ellipsis. |
| Decisions not rationale | Good. Rules are imperative and rationale stays in the memlog. AD-22's estimate caveat and the profile table are acceptable as configuration facts. |
