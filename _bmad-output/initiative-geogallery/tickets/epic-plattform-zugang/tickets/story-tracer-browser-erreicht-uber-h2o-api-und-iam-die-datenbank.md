---
id: 1
type: story
title: "Tracer: Browser erreicht über H2O, api und iam die Datenbank"
parent: epic-plattform-zugang
after: []
hitl: true
risk: high
---

# Tracer: Browser erreicht über H2O, api und iam die Datenbank

## Description

Legt die Repos und Ordner an (contracts/, deploy/, libs/service-core, services/api, services/iam, web/), baut H2O-Master mit mruby und TLS 1.3, das geogallery-Image mit systemd und je einer Unit, einem Unix-User und einem Socket für api und iam (Socket Activation), den postgresql-Container mit PG 18 + PostGIS + PGroonga und dem Schema iam (Owner-Rolle für Migrationen, Laufzeitrolle für iam; DB-Zugang aus der mit XChaCha20-Poly1305 verschlüsselten Datei, Schlüssel als Podman-Secret `config-key`), alles als Podman-Quadlets in der dev-Umgebung. Die Angular-Shell (Spartan/UI) liest `/health` als Envelope; die api fragt dafür per gRPC Health über UDS bei iam an, und iam meldet nur gesund, wenn seine DB-Verbindung mit der Laufzeitrolle steht. Beweist die beiden riskantesten Teile: H2O ohne Releases und mehrere Socket-aktivierte Units im Container.

Eine Person stellt die dev-Umgebung bereit, legt die GitHub-Repos an, setzt die Podman-Secrets und hinterlegt ein TLS-Zertifikat für den dev-host, dem der Browser vertraut.

## Requirements

Verify: Im Browser zeigt https://<dev-host>/ die Angular-Shell mit „api: gesund, iam: gesund, db: gesund“; `/health` antwortet im Envelope (AD-29). Nach `systemctl stop iam.service` im Container (iam.socket bleibt aktiv) startet die nächste Anfrage iam per Socket Activation neu und zeigt wieder „gesund“. Ist die DB gestoppt, zeigt die Shell „db: nicht gesund“. Die Konfigurationsdatei im Volume enthält kein Passwort im Klartext.

## References

- parent — _bmad-output/initiative-geogallery/tickets/epic-plattform-zugang/epic-plattform-zugang.md
- architecture — _bmad-output/initiative-geogallery/architecture-geogallery/architecture-geogallery.md, AD-3, AD-9, AD-23, AD-25, AD-26, AD-29, AD-34, AD-37, Consistency Conventions, Stack, Structural Seed

## Notes

- Person stellt die dev-Umgebung bereit, legt die GitHub-Repos an und setzt die Podman-Secrets.
- Person hinterlegt ein vom Browser vertrautes TLS-Zertifikat für den dev-host; ohne es scheitert WebAuthn in Story 2 (Passkeys brauchen einen sicheren Kontext).
- Nicht hier: service-core enthält in dieser Story nur Socket Activation, gRPC Health und das Entschlüsseln der Konfiguration; `SO_PEERCRED`, JSON-Logs und OTel liefert Story 4, TLS-1.3-only-Prüfung, CSP und Header Story 10, die Wartungsseite Story 11.
- H2O wird vom gepinnten Master-Commit `d3e43fcf` gebaut (`WITH_MRUBY=ON`, mruby-digest), siehe Stack.
- Für PG 18 mit PostGIS und PGroonga gibt es kein fertiges Image: eigenes Containerfile auf Basis von `postgis/postgis:18-3.7` mit PGroonga 4.1.0.
- Die Services binden `contracts/` über einen ersten Git-Tag (AD-34).
