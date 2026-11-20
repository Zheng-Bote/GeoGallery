---
type: initiative
title: GeoGallery
parent: none
sources:
  - _bmad-output/initiative-geogallery/forge-geogallery/forge-geogallery.md
  - /Volumes/1TB_NVME/Documents/obsidian-snippets/GeoGallery.md
companions: []
after: []
risk: high
---

# GeoGallery

## Description

GeoGallery ist eine selbst gehostete, geografisch organisierte Bildverwaltung für eine feste Gruppe (Familie/Verein). Sie übernimmt den Bestand von ~200.000 Bildern, wächst bis 1 Mio. und veröffentlicht ausgewählte Alben in einer öffentlichen Galerie. Die Originale bleiben unverändert und sicher; Ort, Metadaten und Sichtbarkeit leben in der Datenbank. Grundlage ist die gehärtete Idee (forge), ergänzt um den Anforderungsentwurf v0.1 (draft). Bei Widersprüchen gilt die Forge-Datei.

## Outcome

Die Gruppe findet ihr gemeinsames Fotoarchiv über Ort, Ordner, Karte und Suche schnell wieder, und Besucher ohne Konto sehen nur bewusst veröffentlichte Alben mit freigegebenen Metadaten. Erfolg heißt: Die Leistungsziele werden bei 1 Mio. Bildern erreicht, und ein Restore klappt in ≤ 24 h ohne Verlust von Originalen.

## Requirements

- R1 (forge §Nutzer und Sichtbarkeit; draft §10, §13): Der Admin legt Konten an, weist Rollen und Gruppen zu und setzt den Status aktiv/deaktiviert/gesperrt; ohne vom Admin angelegtes, aktives Konto ist keine Anmeldung möglich.
- R2 (forge §Scope; draft §11): Nutzer melden sich passwortlos per WebAuthn/FIDO2 an (Passkey, Windows Hello, YubiKey, Platform Authenticator), mit TOTP nach RFC 6238 als Fallback; eine Anmeldung ohne einen dieser Faktoren wird abgewiesen.
- R3 (forge §Nutzer und Sichtbarkeit; draft §12): Rollen vergeben RBAC-Berechtigungen (u. a. `image.*`, `user.*`, `album.publish`, `system.admin`; Standardrollen Viewer/Editor/Admin); eine Aktion ohne Berechtigung wird abgewiesen, und Rollen ändern nie, was ein Nutzer sieht.
- R4 (forge §Nutzer und Sichtbarkeit): Album und Bild tragen die Sichtbarkeit public / alle angemeldeten (Default) / bestimmte Gruppen / privat (nur Owner); ein Bild erbt vom Album, die restriktivste Einstellung gewinnt, und Raster, Karte, Suche und API zeigen einem Nutzer nur, was diese Regel ihm erlaubt.
- R5 (forge §Nutzer und Sichtbarkeit): Wer `album.publish` hat, setzt ein Album auf public; seine Bilder sind dann ohne Konto sicht- und auffindbar, außer den für dieses Album einzeln ausgeschlossenen. Ein Bild ohne public-Album, ein Gruppen- oder ein privates Bild erscheint nie öffentlich.
- R6 (forge §Metadaten, §Verworfen): Ausgeliefert werden nur Ableitungen (AVIF, Thumbnails), nie Originale; Ableitungen tragen außer Copyright keine Metadaten, und es gibt eine Ableitung pro Bild für intern und öffentlich.
- R7 (forge §Metadaten): Öffentlich erscheinen nur Metadatenfelder der Positivliste des Admins, überschreibbar pro Album und Bild; liegt ein Bild in mehreren public-Alben, gilt die Schnittmenge, und andere Felder fehlen in Seite und API.
- R8 (forge §Metadaten): Öffentliches GPS ist pro Album exakt / vergröbert / aus; bei mehreren public-Alben gilt aus vor vergröbert vor exakt.
- R9 (forge §Upload, Import, Ablage; draft §3.1): Der Admin lädt per SCP/SFTP hoch, alle anderen über das Web (Einzel, Mehrfach, Drag & Drop) mit selbst gewähltem Pfad; beides landet unter `/storage/uploads/<username>/<Pfad>` und wird ohne weiteren Schritt verarbeitet.
- R10 (forge §Upload, Import, Ablage): Der Import verschiebt jede Datei nach `/storage/originals/<gelieferte Ordnerstruktur>/<yyyy-mm-dd_hhmmss>.<ext>`; der Name kommt aus der Aufnahmezeit, sonst aus dem Datei-Zeitstempel. Bei einer Kollision zählt er fortlaufend hoch, auch bei parallelen Workern eindeutig. Die Konvention `Kontinent/Land/Provinz/Region/Stadt[/yyyy/mm]` wird vorgeschlagen, nicht erzwungen, und der Import sortiert nicht nach GPS.
- R11 (forge §Upload, Import, Ablage; §Metadaten): Nach dem Import ändern sich Original und physischer Pfad nie mehr, und der SHA256 bleibt gültig; Verschieben und Umbenennen ändern nur die DB.
- R12 (forge §Upload, Import, Ablage; draft §8): Eine Datei mit bereits vorhandenem SHA256 wird geparkt statt importiert und Admin und Owner in der Weboberfläche angezeigt.
- R13 (forge §Architektur, §Scope; draft §3.2, §3.3): Die Pipeline verarbeitet JPEG, PNG, TIFF, HEIC, AVIF und WebP: Hash, Metadaten, Geo-Validierung, AVIF, Vorschaubilder, DB-Eintrag. Jeder Job zeigt seinen Status, wird bei Fehlern wiederholt und endet sonst in einem sichtbaren Fehlerzustand.
- R14 (forge §Metadaten; draft §4, §9): Alle EXIF-, IPTC- und XMP-Metadaten landen beim Import in der DB und werden aus ihr angezeigt; angemeldete Nutzer sehen sie vollständig in der Detailansicht.
- R15 (forge §Metadaten; draft §13): Berechtigte bearbeiten Metadaten nur in der DB; der Originalwert bleibt daneben erhalten und einsehbar, angezeigt wird der bearbeitete.
- R16 (forge §Metadaten): Ein Bild hat genau eine Bewertung 0–5 und frei vergebene, global sichtbare Tags; Autor ist der Fotograf aus EXIF/IPTC, getrennt vom Owner (Uploader).
- R17 (forge §Ortsmodell; draft §5): Ein Bild mit GPS erhält per Reverse-Geocoding Kontinent/Land/Provinz/Region/Stadt; dieser Ort ist die Wahrheit für Karte, Geo-Navigation und Geo-Filter.
- R18 (forge §Ortsmodell): Ein Bild ohne GPS erhält seinen Ort aus dem Ordnerpfad, markiert als „ungefähr“, sonst erscheint es unter „Ort fehlt“; weichen Ordner und GPS voneinander ab, landet es auf einer Prüfliste.
- R19 (forge §Ortsmodell, §Scope): Nutzer navigieren nach Ort (Kontinent bis Stadt) und nach dem Ordnerbaum.
- R20 (forge §Performance; draft §5): Die Karte clustert serverseitig, und Nutzer suchen auf ihr per Radius, Polygon und Bounding Box.
- R21 (draft §9): Das Raster ist responsiv und lädt per Lazy Loading und Infinite Scroll nach; die Detailansicht zeigt Bild, Metadaten, Karte, Tags und Bewertung.
- R22 (draft §7): Die Volltextsuche durchsucht Titel, Beschreibung, Keywords, Kamera und Objektiv; Filter nach Land, Region, Stadt, Kamera, Objektiv, Jahr, Monat, Autor, Bewertung und Tags lassen sich per UND kombinieren.
- R23 (forge §Scope; draft §13, §19): Der Admin sieht Worker, Job-Queue, Datenbank, Speicherverbrauch, geparkte Duplikate und die Prüfliste und stößt ein Reindexing an; Metriken (Uploads/s, Jobs, Queue-Länge, API-Antwortzeiten, DB-Latenz) werden erfasst.
- R24 (forge §Architektur und Betrieb): Originale gehen nie verloren (RPO 0), die DB hat PITR, ein dokumentierter Restore ist getestet und dauert ≤ 24 h; Ableitungen werden aus Originalen neu erzeugt.
- R25 (forge §Architektur und Betrieb): Ist die Anwendung nicht erreichbar, zeigt die öffentliche Galerie eine statische Wartungsseite.
- R26 (draft §13): Wer `image.delete` hat, löscht Bilder; die Bedeutung von Löschen ist offen (siehe Notes).

## Constraints

- Stack: Linux, C++ mit Drogon (REST/JSON, Sessions, RBAC), PostgreSQL 19 + PostGIS (`GEOGRAPHY(Point,4326)`), Angular 22, H2O (TLS, HTTP/2, HTTP/3, Reverse Proxy), libvips + libavif (forge Einleitung; draft §2, Technologie-Stack).
- Ein Host, Komponenten als systemd-Units; synchron gRPC über UDS (Protobuf) mit Socket Activation und gRPC Health; asynchron eine dauerhafte Job-Queue in PostgreSQL. Kein NATS, keine FlatBuffers (forge §Architektur und Betrieb, §Verworfen).
- Leistungsziele bei 1 Mio. Bildern auf dem Zielhost, auch während eines laufenden Imports: Karte < 300 ms, Raster nächste Seite < 150 ms p95, Suche/Filter < 200 ms p95, API-Metadaten < 100 ms p95; Thumbnails statisch vom Edge (forge §Performance).
- Import ≥ 10.000 Bilder/h (JPEG/HEIC) mit niedrig priorisierten Workern; 10 gleichzeitige Uploads; Bestand ~200.000, Ziel 1 Mio. (forge §Performance; draft §17).
- 99,5 % Verfügbarkeit ohne Wartungsfenster; Datensicherheit geht vor Hochverfügbarkeit (forge §Architektur und Betrieb).
- Sicherheit: TLS 1.3, CSP, XSS- und CSRF-Schutz, Rate Limiting, Audit Logging; Session-Cookies Secure, HttpOnly, SameSite=Strict (draft §11, §17).
- Betrieb: strukturierte JSON-Logs, Metrik-Export per OpenTelemetry (draft §18, §19).
- Originale werden nie verändert; Metadaten werden nie in Dateien zurückgeschrieben (forge §Metadaten, §Verworfen).

## Done when

- Der Bestand von ~200.000 Bildern ist per SCP/SFTP importiert, Duplikate sind geparkt, und auf dem Zielhost wurden ≥ 10.000 Bilder/h gemessen.
- Bei 1 Mio. Bildern (Bestand plus synthetische Last) und laufendem Import erreichen Karte, Raster, Suche und API die Leistungsziele.
- Ein anonymer Besucher sieht nur Bilder aus public-Alben, mit Positivliste und GPS-Regel angewendet; ein Test mit Gruppen-, privaten und ausgeschlossenen Bildern zeigt keines davon öffentlich.
- Ein Restore aus dem Backup auf einen leeren Host ist in ≤ 24 h abgeschlossen, und alle Originale haben ihren SHA256 behalten.
- V1 läuft in Produktion auf dem Host, mit Admin-Monitoring, Wartungsseite und gemessener Verfügbarkeit.

## Boundaries

- Nicht in V1: Slideshow, PWA, XMP-Sidecar-Export, pHash (visuelle Duplikate), RAW-Formate. Sie kommen in V1.x (forge §Scope).
- Später: Desktop- und Mobile-App sowie die Zukunftserweiterungen des Entwurfs (Gesichtserkennung, KI-Klassifikation, Landmarken, Ähnlichkeitssuche, Multi-Tenant, S3-native/MinIO, OpenSearch, Föderation) (forge §Scope; draft §14, §20).
- Verworfen: NATS, FlatBuffers, 99,9 % Verfügbarkeit, 50 gleichzeitige Uploads, Freigabe-Links, Metadaten in Ableitungen, Rückschreiben in Originale, Rechte pro Person/Bild, offene Registrierung (forge §Verworfen, §Nutzer und Sichtbarkeit).
- Kein Multi-Host-Betrieb.
- Tracer-Pfad: Web-Upload → Import und Pipeline → Metadaten und Ortsmodell → Sichtbarkeit → Raster/Karte für Angemeldete → public-Album in der öffentlichen Galerie.

## References

- forge — _bmad-output/initiative-geogallery/forge-geogallery/forge-geogallery.md, alle Abschnitte (geht bei Widerspruch vor)
- draft — /Volumes/1TB_NVME/Documents/obsidian-snippets/GeoGallery.md, §2–§19 und Technologie-Stack
- forge-memlog — _bmad-output/initiative-geogallery/forge-geogallery/.memlog.md, Herleitung der Forge-Entscheidungen

## Notes

- Decision: 2026-11-20, Nutzer — bei Widerspruch gilt forge vor draft. Danach entschieden: kein NATS, keine FlatBuffers, 99,5 % statt 99,9 %, 10 statt 50 Uploads, `/storage/uploads` statt `/storage/temp`, Slideshow/PWA/pHash/RAW in V1.x.
- Assumption: „privat“ gilt auch gegenüber dem Admin in der Anwendung; technischer Dateizugriff auf dem Host bleibt möglich (forge §Nutzer und Sichtbarkeit).
- Assumption: Ablage auf dem lokalen Dateisystem unter `/storage`; MinIO/S3 (draft §14) erst mit der Erweiterung „S3-native“.
- Open question: Datenquelle für Reverse-Geocoding (Nominatim / GeoNames / Natural Earth + Verwaltungsgrenzen) und Umgang mit der uneinheitlichen Ebene „Region“. R17 wartet darauf; Weg: `bmad-deep-recon`.
- Open question: Zielhardware unbekannt; Import-Durchsatz und Leistungsziele müssen dagegen validiert werden.
- Open question: Der Datei-Zeitstempel als Fallback ist unzuverlässig (Kopie/Download). Soll ein Bild als „Zeit unsicher“ gekennzeichnet werden?
- Open question: Wird der beim Upload mitgegebene Pfad auch als Album angelegt?
- Open question: Was bedeutet Löschen (R26): Original physisch entfernen, inkl. Backups, oder nur logisch? Und wer darf löschen?
- Open question: Datenschutz/DSGVO der öffentlichen Galerie (erkennbare Personen, Einwilligung, Löschwunsch). Beide Quellen schweigen dazu.
- Open question: Wer setzt Gruppen- oder public-Sichtbarkeit auf fremden Alben/Bildern: nur der Owner oder auch Editor/Admin?
- Open question: Woher kommt der Copyright-Wert in den Ableitungen (Feld, Fallback)?
