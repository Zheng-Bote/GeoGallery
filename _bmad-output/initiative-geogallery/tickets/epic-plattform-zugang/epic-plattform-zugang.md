---
type: epic
title: "Plattform steht, die Gruppe meldet sich an"
parent: initiative-geogallery
covers: ["R1", "R2", "R3", "R25"]
sources:
  - _bmad-output/initiative-geogallery/initiative-geogallery.md
companions:
  - _bmad-output/initiative-geogallery/architecture-geogallery/architecture-geogallery.md
after: []
refined: true
risk: high
---

# Plattform steht, die Gruppe meldet sich an

## Description

Plattform-Baseline zuerst: die drei Podman-Container `h2o` (H2O-Master mit mruby, TLS 1.3, HTTP/2/3, CSP, Wartungsseite), `geogallery` (systemd mit den Services `api`, `iam` und `webauthn`, je eigener Unit, Unix-User und Socket mit Socket Activation) und `postgresql` (PG 18 + PostGIS + PGroonga), die gemeinsame Bibliothek `libs/service-core`, die Verträge unter `contracts/`, die Angular-Shell mit Spartan/UI, CI in jedem Repo, ein Release-Manifest und ein Image in GHCR, ausgerollt auf eine dev-Umgebung, die wie prod aufgebaut ist. Darauf legt ein Bootstrap-Kommando den ersten Admin an, der Admin pflegt Konten, Rollen und Gruppen in der Admin-Oberfläche, Mitglieder richten über einen Einmal-Link Passkey und TOTP ein und melden sich damit an, und ist die api nicht erreichbar, zeigt H2O die Wartungsseite.

## Outcome

Die Gruppe hat eine laufende, abgesicherte Plattform, auf der jedes Mitglied mit eigenem Konto per Passkey oder TOTP hineinkommt und nur tun kann, was seine Rolle erlaubt.

## Requirements

- R1 (parent R1; Decision 2026-11-21, 2026-11-22): Ein Bootstrap-Kommando im iam-Container legt den ersten Admin an und gibt einen Einrichtungslink aus; existiert bereits ein Admin mit eingerichtetem Faktor, wird es abgewiesen; hat der Admin noch keinen Faktor, gibt es einen neuen Link aus, und der alte wird ungültig.
- R2 (parent R1): Der Admin legt in der Admin-Oberfläche Konten an, weist Rollen und Gruppen zu und setzt den Status aktiv/deaktiviert/gesperrt; ohne aktives Konto ist keine Anmeldung möglich, und eine Deaktivierung oder Sperre beendet laufende Sessions nach spätestens 60 s.
- R3 (parent R1, R2; Decision 2026-11-21): Für jedes neue Konto erzeugt iam einen Einmal-Einrichtungslink, 24 h gültig, über den das Mitglied einen Passkey und optional TOTP einrichtet; ein abgelaufener oder schon benutzter Link wird abgewiesen.
- R4 (parent R2): Mitglieder melden sich per WebAuthn/FIDO2 (Passkey, Windows Hello, YubiKey, Platform Authenticator) oder per Benutzername und TOTP-Code nach RFC 6238 an; jede andere Anmeldung wird abgewiesen.
- R5 (parent R2; Decision 2026-11-21): Hat ein Mitglied keinen Faktor mehr, setzt der Admin seine Faktoren zurück und erzeugt einen neuen Einrichtungslink; beides steht im Audit-Log.
- R6 (parent R3; Decision 2026-11-21): Rollen vergeben Berechtigungen aus einer geschlossenen Liste, mit den Standardrollen Viewer, Editor und Admin in einer vom Nutzer freigegebenen Zuordnung; eine Aktion ohne Berechtigung wird abgewiesen und im Audit-Log festgehalten, und Rollen ändern nie, was ein Nutzer sieht.
- R7 (parent R25): Ist die api nicht erreichbar, liefert H2O für jede Adresse die statische Wartungsseite.

## Constraints

- Stack, Laufzeit (Podman/Quadlets, systemd-Units im `geogallery`-Container, gRPC über UDS mit Socket Activation und Health), Sicherheit (TLS 1.3, CSP, XSS-/CSRF-Schutz, Rate Limiting, Audit, Sessions serverseitig ohne JWT, Cookies Secure/HttpOnly/SameSite=Strict), JSON-Logs und OTel, 99,5 % ohne Wartungsfenster: wie in der Initiative (parent Constraints).
- Bereitstellung auf eine dev-Umgebung, aufgebaut wie prod; CI über GitHub Actions in jedem Repo, das Image liegt privat in GHCR (Decision 2026-11-21).
- `libs/service-core` ist die gemeinsame Grundlage aller C++-Services: gRPC über UDS mit Socket Activation, Methodenrechte per `SO_PEERCRED`, JSON-Logs, OTel, Health und Geheimnisse (Decision 2026-11-21).

## Done when

- Ein Commit auf main baut in CI (mit `buf breaking` für `contracts/`); ein Release-Manifest pinnt die Stände, das `geogallery`-Image liegt in GHCR und läuft mit `h2o` und `postgresql` als Podman-Quadlets in der dev-Umgebung; gRPC Health über UDS meldet api, iam und webauthn gesund.
- Das Bootstrap-Kommando legt den ersten Admin an; der Admin legt ein Konto mit Rolle und Gruppe an; das Mitglied richtet über den Einmal-Link Passkey und TOTP ein und meldet sich mit jedem der beiden an; ein deaktiviertes oder gesperrtes Konto wird abgewiesen, und seine Session endet nach spätestens 60 s.
- Eine Aktion ohne Berechtigung wird abgewiesen und im Audit-Log festgehalten; ein Service-Aufruf außerhalb des erlaubten Graphen wird abgewiesen; Cookies tragen Secure/HttpOnly/SameSite=Strict, und TLS 1.3, CSP, CSRF-Schutz und Rate Limit sind aktiv.
- Ist die api gestoppt, liefert H2O die Wartungsseite; ein Neustart eines einzelnen Service im laufenden Container führt zu keiner Fehlerantwort.

## Boundaries

- Kein Upload, kein Bild: das liefert epic-import-pipeline, ebenso die Job-Queue-Bibliothek und die Services catalog, ingest, media und geocoder.
- Gruppen werden hier nur verwaltet; ihre Wirkung auf Sichtbarkeit liefern epic-import-pipeline und epic-alben-sichtbarkeit.
- Kein Produktivhost: prod folgt in gleicher Form, sobald die Hardware steht.
- Keine Anbindung des Desktop-Clients (AD-49).

## References

- parent — _bmad-output/initiative-geogallery/initiative-geogallery.md, section Requirements (R1, R2, R3, R25), Constraints, Notes
- architecture — _bmad-output/initiative-geogallery/architecture-geogallery/architecture-geogallery.md, AD-8, AD-9, AD-10, AD-11, AD-23, AD-24, AD-25, AD-26, AD-29, AD-34, AD-35, AD-36, AD-38, AD-47, AD-48, AD-49, Consistency Conventions, Stack, Structural Seed

## Notes

- Decision: 2026-11-21, Nutzer — der Prüfpunkt „Socket Activation mehrerer Sockets in systemd im Container“ (spine, Deferred → epic-import-pipeline) wird in den Tracer dieses Epics vorgezogen, zusammen mit dem Build von H2O-Master mit mruby.
- Decision: 2026-11-21, Nutzer — das Epic rollt auf eine dev-Umgebung aus statt auf prod, weil die Zielhardware noch nicht feststeht.
- Decision: 2026-11-21, Sofia (Entwurf) — Tracer ist Story 1 (Browser → H2O → api → iam → PostgreSQL mit Health, Socket Activation, verschlüsselte DB-Zugangsdaten); danach Story 2, der Passkey-Pfad mit webauthn, als unsicherster Teil. CI und Release-Manifest (Story 3) warten auf Story 2, weil das Manifest auch webauthn pinnt; Refactor sweep schließt das Epic, keine eigene End-to-End-Suite.
