---
type: epic
title: "Plattform steht, die Gruppe meldet sich an"
parent: initiative-geogallery
covers: ["R1", "R2", "R3", "R25"]
sources: []
companions: []
after: []
refined: false
risk: medium
---

# Plattform steht, die Gruppe meldet sich an

## Description

Plattform-Baseline zuerst: H2O (TLS 1.3, HTTP/2/3, CSP, Rate Limiting), XSS-Schutz durch Ausgabe-Kodierung in Angular und Drogon, Drogon-API, Angular-Shell, PostgreSQL/PostGIS als systemd-Units mit gRPC über UDS und Health, JSON-Logs, Audit-Log, CI und Deployment auf den Host. Darauf legt der Admin Konten, Rollen und Gruppen an, Nutzer melden sich per Passkey oder TOTP an (Session-Cookies Secure/HttpOnly/SameSite=Strict, CSRF-Schutz), und bei Ausfall zeigt die öffentliche Seite die Wartungsseite.

## Outcome

Die Gruppe hat eine laufende, abgesicherte Plattform auf dem Host, auf der jedes Mitglied mit eigenem Konto per Passkey oder TOTP hineinkommt und nur tun kann, was seine Rolle erlaubt.

## Done when

- Ein Commit auf main baut in CI und wird per Deployment als systemd-Units (H2O, Drogon-API, PostgreSQL/PostGIS) auf den Host ausgerollt; gRPC-Health über UDS meldet alle Dienste gesund.
- Der Admin legt ein Konto mit Rolle und Gruppe an; der Nutzer registriert einen Passkey und meldet sich an, TOTP funktioniert als Fallback; ein deaktiviertes oder gesperrtes Konto wird abgewiesen.
- Eine Aktion ohne Berechtigung wird abgewiesen und im Audit-Log festgehalten; Cookies tragen Secure/HttpOnly/SameSite=Strict, TLS 1.3, CSP und Rate Limiting sind aktiv.
- Ist die API gestoppt, liefert H2O für die öffentliche Adresse die statische Wartungsseite.

## Boundaries

- Kein Upload, kein Bild: das liefert epic-import-pipeline.
- Gruppen werden hier nur verwaltet; ihre Wirkung auf Sichtbarkeit liefern epic-import-pipeline und epic-alben-sichtbarkeit.

## References

- parent — _bmad-output/initiative-geogallery/initiative-geogallery.md, section Requirements (R1, R2, R3, R25), Constraints, Notes
