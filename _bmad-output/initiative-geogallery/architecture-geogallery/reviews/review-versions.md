---
review: versions-and-fit
target: architecture-geogallery.md (status draft, updated 2026-11-21)
lens: "Every committed decision web-researched or reality-checked; current versions; technology still exists and fits; starter defaults"
date: 2026-11-21
method: GitHub Releases API (gh), crates.io/npm registry APIs, ConanCenter recipe config.yml, vendor docs (WebFetch), snippet inspection under /Users/zb_bamboo/Documents/DEV/CPP/gallery_snippets
---

# Review — Versions and Fit (Stack, AD-4/12/20/23/25/26/30/32/34/36/37)

**Verdict:** The stack is broadly current and every named technology still exists and fits, but several Stack-table entries are already one release behind, the PostGIS line pinned (3.6) is no longer the maintained container line, and three fit gaps (ConanCenter lag, gRPC socket activation needing libsystemd, RLS owner bypass) would surface only at build time unless written into the spine now.

Legend: **OK** = confirmed current/fits · **UPDATE** = newer release, bump the Stack table · **FIT** = exists but the decision needs a condition or correction · **RESOLVED** = an open checkpoint answered by evidence.

## Findings (prioritised)

### F1 — ConanCenter lags the Stack table for four libraries (FIT, high)
AD-36 commits to Conan 2. ConanCenter recipes (checked 2026-11-21) offer:

| Library | Spine | Upstream current | ConanCenter newest |
| --- | --- | --- | --- |
| libvips | 8.18.x | 8.18.7 (2026-09-26); 8.19.0-rc2 (2026-11-17) | **8.16.0** |
| Exiv2 | 0.28.x | **0.28.9** (2026-08-30, security fixes) | **0.28.3** |
| libavif | 1.4.2 | 1.4.2 (2026-05-26) | 1.4.1 |
| gRPC | ≥ 1.80 | 1.85.4 (2026-11-20) | 1.84.0 |

So 8.18 libvips and a patched Exiv2 are not available from ConanCenter at all. Exiv2 0.28.4–0.28.9 fixed a series of advisories (e.g. CVE-2026-68546 heap OOB write, CVE-2026-49275, CVE-2026-27596); 0.28.3 parses untrusted uploads, so that matters here. `photo_indexer` (candidate for ingest) pins `exiv2/0.28.2`.
**Correction:** Add to AD-36: "Libraries ConanCenter does not have at the required version (libvips, Exiv2, libheif/libavif as needed) come from a project-owned Conan remote/recipes or from distro packages inside the build image; Exiv2 ≥ 0.28.9." Pin libvips 8.18.7 (8.19.0 is at rc2).
Sources: https://github.com/conan-io/conan-center-index/tree/master/recipes (libvips, exiv2, libavif, grpc `config.yml`) · https://github.com/Exiv2/exiv2/releases/tag/v0.28.9 · https://github.com/Exiv2/exiv2/security/advisories · https://github.com/libvips/libvips/releases

### F2 — gRPC C++ socket activation only works when gRPC is built with libsystemd (FIT, high)
AD-23/AD-25 require systemd socket activation for every service. gRPC C++ adopts an inherited systemd UDS in `AddListeningPort("unix:...")` only when compiled with `gRPC_USE_SYSTEMD` (`src/core/lib/iomgr/systemd_utils.cc`, `cmake/systemd.cmake`; upstream example `examples/cpp/systemd_socket_activation`). The ConanCenter `grpc` recipe exposes `with_libsystemd` with default **False**. Without it, the server binds its own socket and activation silently does not happen.
**Correction:** Stack: "gRPC C++ 1.85.x (current 1.85.4), Conan option `grpc/*:with_libsystemd=True`." Add one line to AD-23 that the Rust (tonic) and Go components must take the listener from `LISTEN_FDS` themselves.
Sources: https://github.com/grpc/grpc/releases/tag/v1.85.4 · https://github.com/grpc/grpc/tree/master/examples/cpp/systemd_socket_activation · https://github.com/grpc/grpc/blob/master/src/core/lib/iomgr/systemd_utils.cc · https://github.com/conan-io/conan-center-index/blob/master/recipes/grpc/all/conanfile.py

### F3 — RLS: the table owner bypasses policies; with the AD-3 role model, catalog's own role is likely the owner (FIT, high)
AD-20 says "Leserolle hat kein BYPASSRLS". PostgreSQL 18 docs: "Table owners normally bypass row security as well, though a table owner can choose to be subject to row security with ALTER TABLE ... FORCE ROW LEVEL SECURITY." AD-3 has each service running migrations with its own role, so catalog's runtime role would own `images` unless the roles are split, and RLS would then not be fail-closed.
**Correction (AD-20):** "Tables are owned by a separate migration role; the runtime role is neither owner nor BYPASSRLS; in addition `FORCE ROW LEVEL SECURITY`. The principal is set with `set_config('geogallery.principal_*', …, true)` (= SET LOCAL, parameterisable); policies read `current_setting(…, true)` and return no rows when it is empty." The pattern itself (SET LOCAL / `set_config(...,true)` per transaction) is current and transaction-pool safe.
Sources: https://www.postgresql.org/docs/18/ddl-rowsecurity.html · https://www.bytebase.com/blog/postgres-row-level-security-footguns/

### F4 — PostGIS 3.6 is no longer the maintained container line; 3.7.0 is final and supports PG 14–19 (UPDATE/FIT, medium)
The memlog `version` entry says "PostGIS 3.7.0 bis rc2 … finales Release unbestätigt". **PostGIS 3.7.0 was released 2026-10-30** ("requires … PostgreSQL 14-19"). The official `postgis/postgis` image says "`*-3.5` and `*-3.6` tags are no longer updated"; its PG18 line is `debian:trixie | 18 | 3.7.0`. The PG18 decision (AD-37) still stands, because **PGroonga has not released PG19 support**: its CI runs PG19 only as `postgresql-unreleased: "yes"` and packages exist only up to `postgresql-18-pgdg-pgroonga`. PostGIS alone no longer argues for PG18.
**PGroonga + PostGIS together:** no published image bundles both. The practical base is `postgis/postgis:18-3.7` (Debian trixie) + `postgresql-18-pgdg-pgroonga` from packages.groonga.org (trixie is the only supported Debian). This fits.
**Correction:** Stack: PostGIS **3.7.0**, PGroonga **4.1.0** (2026-10-09; 4.0.8 is two releases old; 4.0.8 moved the source build to Meson and dropped bookworm). Revise the AD-37 rationale to "PGroonga has no PG19 release yet". Note that the PG18 image's PGDATA mount path is `/var/lib/postgresql`.
Sources: https://postgis.net/news/ · https://github.com/postgis/docker-postgis · https://pgroonga.github.io/news/ · https://pgroonga.github.io/install/debian.html · https://github.com/pgroonga/pgroonga/blob/main/.github/workflows/linux.yml

### F5 — H2O mruby HMAC: answered from the H2O source; the "Offene Prüfpunkt" can be narrowed (RESOLVED, medium)
H2O vendors `deps/mruby-digest` (fork using picohash, no external dependency) and `misc/mruby_config.rb` compiles every `deps/mruby-*` gem into the build. The README says "Message Digest and HMAC classes are available … Digest::SHA256 … Digest::HMAC", and `digest.c` defines `HAVE_SHA256` + `mrb_hmac` on the picohash path. So `Digest::HMAC.hexdigest(msg, key, Digest::SHA256)` is available in a stock H2O build with `WITH_MRUBY=ON` (CMake default ON when Ruby/bison are present). Conditions: (a) the h2o container image must be built with mruby (some distro packages build without it); (b) mruby-digest has no constant-time compare, so write one in Ruby (XOR loop over all bytes); (c) read the key from the Podman secret file at config load (`mruby-env`/File are bundled).
Also confirmed: H2O publishes no more tags; the `tag-no-more-releases` release (2026-01-19) says "Users are advised to use the up-to-date commit of the master branch". Current master: `d3e43fcf` (2026-11-14). Spine wording "2.3.0-beta2 bzw. master-Commit" → **master-Commit only** (2.3.0-beta2 dates from 2019).
**Correction:** Narrow the open checkpoint to "Spike: HMAC-SHA256 + constant-time compare in mruby on H2O master `<sha>` with WITH_MRUBY=ON"; the fallback stays.
Sources: https://github.com/h2o/h2o/tree/master/deps/mruby-digest · https://github.com/h2o/h2o/blob/master/misc/mruby_config.rb · https://github.com/h2o/h2o/releases/tag/tag-no-more-releases · https://h2o.examp1e.net/configure/mruby.html

### F6 — libvips `heifsave` already writes AVIF; `img2avif` is probably redundant and does not fit "input from libvips" (FIT, medium)
libvips `heifsave` picks AV1 for `.avif`, with `bitdepth` 8/10/12, `effort` 0–9, `lossless`, and `encoder` selection (aom, SVT-AV1, rav1e) via libheif. Metadata control is through `keep` (`VIPS_FOREIGN_KEEP_NONE|EXIF|XMP|IPTC|ICC|OTHER|GAINMAP|ALL`), so "nur Copyright" works as: keep EXIF (+ICC), remove all other `exif-*` fields from the image before saving (spike needed). The `img2avif` snippet decodes with `stb_image.h`, resizes with `stb_image_resize2.h` and writes metadata via Exiv2 into libavif. Its value is exactly the decode/resize/metadata path the spine replaces with libvips. Handing libvips pixels to libavif is possible (`vips_image_write_to_memory` → `avifRGBImage`) but duplicates `heifsave`. libvips does not use libavif; it uses libheif (current 1.23.7, ConanCenter 1.23.4). libheif is needed anyway to read HEIC input.
**Correction (AD-36, Stack):** "media encodes AVIF via libvips `heifsave` (libheif with AV1 encoder, e.g. aom/SVT-AV1); `img2avif` only as reference/benchmark; libavif drops out of the stack unless the spike shows that `heifsave` cannot hold copyright-only metadata." Add libheif ≥ 1.23 to the stack.
Sources: https://www.libvips.org/API/current/method.Image.heifsave.html · https://www.libvips.org/API/current/flags.ForeignKeep.html · https://github.com/strukturag/libheif/releases · snippet `gallery_snippets/img2avif/CMakeLists.txt` (pkg-config libavif, exiv2; stb includes)

### F7 — webauthn-rs 0.6.2 is now stable on crates.io (UPDATE, low)
The memlog says 0.6.2 is "unbestätigt". crates.io `max_stable_version` = **0.6.2** (published 2026-10-31); 0.5.5 dates from 2026-04-30. (GitHub Releases for kanidm/webauthn-rs only show v0.5.2, so use crates.io as the source.) **Correction:** Stack "webauthn-rs 0.6.2" (still pin in epic-iam; read the 0.5→0.6 migration notes).
Source: https://crates.io/crates/webauthn-rs

### F8 — Minor version bumps (UPDATE, low)
| Item | Spine | Current (2026-11-21) | Source |
| --- | --- | --- | --- |
| Drogon | 1.9.13 | **1.9.14** (2026-10-12; also on ConanCenter) | https://github.com/drogonframework/drogon/releases |
| Podman | 6.1.x | **6.2.0** (2026-11-10); 6.1.4 last 6.1 | https://github.com/containers/podman/releases |
| Protobuf | (with gRPC) | 36.2 / C++ 7.36.2 stable; 37.0 at rc3 | https://github.com/protocolbuffers/protobuf/releases |
| libsodium | (no version) | **1.0.22** (2026-04-09; ConanCenter 1.0.22) | https://github.com/jedisct1/libsodium/releases |
| buf | (no version) | **1.75.0** (2026-10-22) | https://github.com/bufbuild/buf/releases |

## Checked and OK

| Item | Result | Source |
| --- | --- | --- |
| PG 18 `uuidv7()` (AD-4) | Built in: `uuidv7([shift interval]) → uuid`, ms + sub-ms timestamp, monotonic per session. OK. | https://www.postgresql.org/docs/18/functions-uuid.html · https://www.postgresql.org/about/news/postgresql-18-released-3142/ |
| libsodium XChaCha20-Poly1305 (AD-26) | `crypto_aead_xchacha20poly1305_ietf_*` since 1.0.12, 192-bit nonce, "the recommended AEAD construction". OK (pin 1.0.22). | https://doc.libsodium.org/secret-key_cryptography/aead/chacha20-poly1305/xchacha20-poly1305_construction |
| `buf breaking` (AD-34) | Current; categories FILE (default) / PACKAGE / WIRE_JSON / WIRE; `--against` accepts git inputs (`.git#tag=vX` via inputs reference). For C++ consumers, FILE is the right default. OK. | https://buf.build/docs/breaking/ · https://buf.build/docs/reference/inputs/ |
| Podman Quadlet socket activation (AD-25) | Documented: a `.socket` unit next to the `.container` (`Requires=echo.socket`), cgroup v2 required; `Secret=` supported in `[Container]`. Not documented: passing several sockets into a container that runs systemd as init and redistributing them to per-service units. **Spike needed** for the "systemd inside geogallery" form (AD-25). The host-level activation pattern itself is current. | https://github.com/containers/podman/blob/main/docs/tutorials/socket_activation.md · https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html |
| Angular 22 (AD-29/30) | 22.3.0 latest on npm; v22 released 2026-06-03, active until 2027-06, LTS until 2028-06; v23 planned ~June 2027. OK. | https://angular.dev/reference/releases · https://registry.npmjs.org/@angular/core |
| FlatBuffers + TypeScript (AD-30) | `flatc --ts` (+ `--gen-object-api`) with npm runtime `flatbuffers`. **Caveat:** GitHub/ConanCenter newest is 25.12.19, but npm `latest` is still **25.9.23**. Pin `flatc` and the C++ headers to the same version as the npm runtime (25.9.23, which `photo_indexer` already uses), or check TS output from 25.12.x against the 25.9.23 runtime. Fit OK. | https://github.com/google/flatbuffers/releases · https://registry.npmjs.org/flatbuffers · https://flatbuffers.dev/languages/typescript/ |
| libavif 1.4.2 | Current upstream (see F1/F6 for fit). | https://github.com/AOMediaCodec/libavif/releases |
| PGroonga with PG18 | Supported since 4.0.4, packages for trixie/noble/resolute. OK (see F4 for version). | https://pgroonga.github.io/news/ |

## Snippet fit (AD-36), quick check

| Snippet | Observation | Fit |
| --- | --- | --- |
| `drogon_auth` | `CMAKE_CXX_STANDARD 26`, contradicting AD-36 "C++23"; requires `grpc/[>=1.54.0 <2]`, `libsodium`, `jwt-cpp`, `openssl`. | Set to 23 when adopting; decide whether JWT also goes (the spine only drops password login + SQLite). |
| `drogon_audit_plugin` | C++23, drogon `>=1.9.12 <2`. | Fits. |
| `geocoding` | C++23, nlohmann_json 3.11.3, libcurl. | Fits. |
| `photo_indexer` | C++23, Conan `exiv2/0.28.2`, `flatbuffers/25.9.23`, xxhash. | Exiv2 must be raised to ≥ 0.28.9 (F1); drop FlatBuffers as planned. |
| `img2avif` | C++23, pkg-config libavif + exiv2, stb decode/resize. | Probably redundant next to libvips `heifsave` (F6). |

## Suggested Stack table (replacement values)

| Name | Version |
| --- | --- |
| Drogon | 1.9.14 |
| gRPC C++ / Protobuf | 1.85.x (1.85.4) / protobuf 36.x; Conan `with_libsystemd=True` |
| H2O | master commit (pin SHA; currently d3e43fcf, 2026-11-14), `WITH_MRUBY=ON` |
| PostgreSQL | 18 |
| PostGIS | 3.7.0 |
| PGroonga | 4.1.0 |
| libvips | 8.18.7 (own recipe/distro; ConanCenter only 8.16.0) |
| libheif | ≥ 1.23 (AV1 encoder aom or SVT-AV1) |
| libavif | 1.4.2, only if img2avif stays |
| Exiv2 | ≥ 0.28.9 (own recipe; ConanCenter only 0.28.3) |
| libsodium | 1.0.22 |
| webauthn-rs | 0.6.2 |
| Angular | 22 (22.3.x) |
| FlatBuffers | 25.9.23 (npm runtime = flatc = C++ headers) |
| buf | 1.75.x |
| Podman / Quadlet | 6.2.x |
