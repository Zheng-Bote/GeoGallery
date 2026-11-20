---
type: forged-idea
title: GeoGallery
outcome: hardened
date: 2026-11-20
source: /Volumes/1TB_NVME/Documents/obsidian-snippets/GeoGallery.md (v0.1 Entwurf; diese Datei geht bei Widerspruch vor)
---

# GeoGallery – geschärfte Idee

Selbst gehostete, geografisch organisierte Bildverwaltung für eine feste Gruppe (Familie/Verein) mit ~200.000 Bildern Bestand (wachsend, Ziel 1 Mio.) und öffentlicher Galerie. Linux, C++ (Drogon), PostgreSQL + PostGIS, Angular, H2O.

## Nutzer und Sichtbarkeit
- Feste Gruppe mit Konten, keine offene Registrierung. Gemeinsames Archiv.
- Rollen = RBAC (was man tun darf). Gruppen = vom Admin gepflegte Nutzermengen (was man sehen darf). Getrennt halten.
- Sichtbarkeit pro Album/Bild: public / alle angemeldeten (Default) / bestimmte Gruppen / privat (nur Owner). Bild erbt vom Album. **Restriktivste Einstellung gewinnt.**
- Veröffentlichen = Album auf public setzen (`album.publish`). Bilder darin standardmäßig public, einzeln ausschließbar (gilt nur für dieses Album). Einzelbild nie ohne Album public. Gruppen-/private Bilder nie public.
- Annahme: privat gilt auch gegenüber Admin in der Anwendung.

## Metadaten
- Nur Originale tragen EXIF/IPTC/XMP. Ableitungen (AVIF, Thumbnails) metadatenfrei außer Copyright → eine Ableitung pro Bild für intern und öffentlich.
- Alle Metadaten beim Import in die DB; Anzeige aus der DB. Intern vollständig.
- Öffentlich: Positivliste, global vom Admin, überschreibbar pro Album und Bild; bei mehreren Alben Schnittmenge.
- Öffentliches GPS pro Album: exakt / vergröbert / aus (aus > vergröbert > exakt).
- Originale werden nie verändert (SHA256 dauerhaft gültig). Bearbeitungen nur in DB, Originalwert daneben. XMP-Sidecar-Export auf Wunsch.
- Bewertung 0–5, eine pro Bild. Tags frei, global. Autor = Fotograf aus EXIF/IPTC, getrennt vom Owner (Uploader).

## Upload, Import, Ablage
- Admin: SCP/SFTP. Alle anderen: Web (später Desktop/Mobile-App). Upload nach `/storage/uploads/<username>/<Pfad vom Nutzer>`.
- Import-Service verschiebt (nicht kopiert) nach `/storage/originals/<vom Nutzer gelieferte Ordnerstruktur>`. Konvention `Kontinent/Land/Provinz/Region/Stadt[/yyyy/mm]` wird vorgeschlagen, nicht erzwungen. Import sortiert nicht nach GPS.
- Dateiname `<yyyy-mm-dd_hhmmss>.<ext>` aus Aufnahmezeit (Fallback Datei-Zeitstempel), bei Kollision fortlaufender Zähler (atomar vergeben).
- Physischer Pfad nach Import unveränderlich. Verschieben/Umbenennen ändern nur die DB.
- SHA256-Duplikate werden geparkt und Admin/Owner angezeigt.

## Ortsmodell
- Ordnerpfad = Ablage-/Herkunftsstruktur, als Ordnerbaum in der DB navigierbar.
- Wahrheit für Karte, Geo-Navigation, Geo-Filter: GPS → Reverse-Geocoding → Kontinent..Stadt.
- Ohne GPS: Ort aus Ordnerpfad, markiert „ungefähr“; sonst Ansicht „Ort fehlt“. Ordner ≠ GPS → Prüfliste.

## Architektur und Betrieb
- Ein Host. Komponenten als systemd-Units. Synchron: gRPC über UDS (Protobuf), Socket Activation, gRPC Health. Asynchron: dauerhafte Job-Queue in PostgreSQL (Status, Retry, Fehler).
- 99,5 % Verfügbarkeit ohne Wartungsfenster; öffentliche Galerie zeigt dann Wartungsseite.
- Datensicherheit vor HA: Originale RPO 0, DB mit PITR, getesteter Restore, RTO ≤ 24 h. Ableitungen regenerierbar.

## Performance (Zielhost, 1 Mio. Bilder)
- Karte < 300 ms (serverseitiges Clustering / Vector Tiles). Raster nächste Seite < 150 ms p95, Thumbnails statisch vom Edge. Suche/Filter < 200 ms p95. API-Metadaten < 100 ms p95.
- Import ≥ 10.000 Bilder/h (JPEG/HEIC); UI-Ziele gelten während Import, Worker niedrig priorisiert.
- 10 gleichzeitige Uploads.

## Scope
- **V1:** Admin-Import + Web-Upload, Pipeline, JPEG/PNG/TIFF/HEIC/AVIF/WebP, SHA256-Duplikate, Metadaten + Reverse-Geocoding + Ortsmodell, Raster/Detail/Karte, Navigation Ort + Ordner, Suche/Filter, Passkey + TOTP, Rollen/Gruppen/Sichtbarkeit, öffentliche Galerie, Metadaten-Edit/Bewertung/Tags, Admin-Monitoring, Backup/Restore.
- **V1.x:** Slideshow, PWA, XMP-Sidecar-Export, pHash, RAW.
- **Später:** Desktop-App, Mobile-App, Zukunftserweiterungen des Entwurfs.

## Verworfen
- NATS – zusätzliche Infrastruktur, Pipeline-Status besser transaktional in PostgreSQL.
- FlatBuffers – dritte Serialisierung ohne Mehrwert.
- 99,9 % Verfügbarkeit – mit einem Host nicht verlässlich; Datensicherheit wichtiger.
- 50 gleichzeitige Uploads – kein realer Lastfall; Durchsatz zählt.
- Freigabe-Links – öffentliche Galerie über Alben stattdessen.
- Metadaten in Ableitungen / Rückschreiben in Originale – Datenschutz bzw. Integrität der Originale.
- Feingranulare Rechte pro Person/Bild – Gruppen genügen.

## Offen
- Datenquelle Reverse-Geocoding (Nominatim / GeoNames / Natural Earth + Verwaltungsgrenzen), uneinheitliche Ebene „Region“ → `bmad-deep-recon`.
- Zielhardware; Import-Durchsatz dagegen validieren.
- Datei-Zeitstempel als Fallback unzuverlässig (Kopie/Download) – Kennzeichnung „Zeit unsicher“ erwägen.
- Wird der vom Nutzer mitgegebene Upload-Pfad auch als Album angelegt?
