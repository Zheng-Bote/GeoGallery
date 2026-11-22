---
type: initiative
title: "FlagMe - The Geo Gallery"
parent: none
sources:
  - _bmad-output/initiative-geogallery/forge-geogallery/forge-geogallery.md
  - /Volumes/1TB_NVME/Documents/obsidian-snippets/GeoGallery.md
companions:
  - _bmad-output/initiative-geogallery/architecture-geogallery/architecture-geogallery.md
after: []
risk: high
---

# FlagMe - The Geo Gallery

## Description

FlagMe - The Geo Gallery ist eine selbst gehostete, geografisch organisierte Bildverwaltung für eine feste Gruppe (Familie/Verein). Sie übernimmt den Bestand von ~200.000 Bildern, wächst bis 1 Mio. und veröffentlicht ausgewählte Alben in einer öffentlichen Galerie. Die Originale bleiben unverändert und sicher; Ort, Metadaten und Sichtbarkeit leben in der Datenbank. Grundlage ist die gehärtete Idee (forge), ergänzt um den Anforderungsentwurf v0.1 (draft). Bei Widersprüchen gilt die Forge-Datei, außer wo die Architektur-Spine sie per Entscheidung vom 2026-11-21 ersetzt (Notes).

## Outcome

Die Gruppe findet ihr gemeinsames Fotoarchiv über Ort, Ordner, Karte und Suche schnell wieder, und Besucher ohne Konto sehen nur bewusst veröffentlichte Alben mit freigegebenen Metadaten. Erfolg heißt: Die Leistungsziele werden bei 1 Mio. Bildern erreicht, und ein Restore klappt in ≤ 24 h ohne Verlust von Originalen.

## Requirements

- R1 (forge §Nutzer und Sichtbarkeit; draft §10, §13): Der Admin legt Konten an, weist Rollen und Gruppen zu und setzt den Status aktiv/deaktiviert/gesperrt; ohne vom Admin angelegtes, aktives Konto ist keine Anmeldung möglich.
- R2 (forge §Scope; draft §11): Nutzer melden sich passwortlos per WebAuthn/FIDO2 an (Passkey, Windows Hello, YubiKey, Platform Authenticator), mit TOTP nach RFC 6238 als Fallback; eine Anmeldung ohne einen dieser Faktoren wird abgewiesen.
- R3 (forge §Nutzer und Sichtbarkeit; draft §12): Rollen vergeben RBAC-Berechtigungen (u. a. `image.*`, `user.*`, `album.publish`, `system.admin`; Standardrollen Viewer/Editor/Admin); eine Aktion ohne Berechtigung wird abgewiesen, und Rollen ändern nie, was ein Nutzer sieht.
- R4 (forge §Nutzer und Sichtbarkeit; Decision 2026-11-21): Album und Bild tragen die Sichtbarkeit public / alle angemeldeten / bestimmte Gruppen / privat (nur Owner). Die Basis jedes Bildes ist „alle angemeldeten“; Alben erweitern sie nur, über mehrere Alben gilt das weiteste, und einengen kann nur der Owner mit einer eigenen Obergrenze am Bild. Gruppen werden über die Alben vereinigt und mit der Obergrenze geschnitten. Raster, Karte, Suche und API zeigen einem Nutzer nur, was diese Regel ihm erlaubt.
- R5 (forge §Nutzer und Sichtbarkeit; Decision 2026-11-21): Wer `album.publish` hat, setzt ein Album auf public; seine Bilder sind dann ohne Konto sicht- und auffindbar, außer den für dieses Album einzeln ausgeschlossenen. Nie öffentlich erscheint ein Bild ohne public-Album oder mit einer eigenen Obergrenze unter public; ein fremdes Bild öffnet ein Album höchstens bis zu dessen Obergrenze, und ohne Obergrenze ≥ public des Owners macht kein anderer es öffentlich.
- R6 (forge §Metadaten, §Verworfen): Ausgeliefert werden nur Ableitungen (AVIF, Thumbnails), nie Originale; Ableitungen tragen außer Copyright keine Metadaten, und es gibt eine Ableitung pro Bild für intern und öffentlich.
- R7 (forge §Metadaten; Decision 2026-11-21): Öffentlich erscheinen nur Metadatenfelder der Positivliste des Admins, überschreibbar pro Album und Bild; liegt ein Bild in mehreren public-Alben, gilt die Schnittmenge, und andere Felder fehlen in Seite und API. Das Copyright der Ableitung ist immer öffentlich, steht außerhalb der Positivliste und wird beim Veröffentlichen angezeigt.
- R8 (forge §Metadaten): Öffentliches GPS ist pro Album exakt / vergröbert / aus; bei mehreren public-Alben gilt aus vor vergröbert vor exakt.
- R9 (forge §Upload, Import, Ablage; draft §3.1): Der Admin lädt per SCP/SFTP hoch, alle anderen über das Web (Einzel, Mehrfach, Drag & Drop) mit selbst gewähltem Pfad; beides landet unter `/storage/uploads/<username>/<Pfad>` und wird ohne weiteren Schritt verarbeitet.
- R10 (forge §Upload, Import, Ablage): Der Import verschiebt jede Datei nach `/storage/originals/<gelieferte Ordnerstruktur>/<yyyy-mm-dd_hhmmss>.<ext>`; der Name kommt aus der Aufnahmezeit, sonst aus dem Datei-Zeitstempel. Bei einer Kollision zählt er fortlaufend hoch, auch bei parallelen Workern eindeutig. Die Konvention `Kontinent/Land/Provinz/Region/Stadt[/yyyy/mm]` wird vorgeschlagen, nicht erzwungen, und der Import sortiert nicht nach GPS.
- R11 (forge §Upload, Import, Ablage; §Metadaten): Nach dem Import ändern sich Original und physischer Pfad nie mehr, und der SHA256 bleibt gültig; Verschieben und Umbenennen ändern nur die DB.
- R12 (forge §Upload, Import, Ablage; draft §8): Eine Datei mit bereits vorhandenem SHA256 wird geparkt statt importiert und Admin und Owner in der Weboberfläche angezeigt.
- R13 (forge §Architektur, §Scope; draft §3.2, §3.3): Die Pipeline verarbeitet JPEG, PNG, TIFF, HEIC, AVIF und WebP: Hash, Metadaten, Geo-Validierung, AVIF, Vorschaubilder, DB-Eintrag. Jeder Job zeigt seinen Status, wird bei Fehlern wiederholt und endet sonst in einem sichtbaren Fehlerzustand.
- R14 (forge §Metadaten; draft §4, §9): Alle EXIF-, IPTC- und XMP-Metadaten landen beim Import in der DB und werden aus ihr angezeigt; angemeldete Nutzer sehen sie vollständig in der Detailansicht.
- R15 (forge §Metadaten; draft §13): Berechtigte bearbeiten Metadaten nur in der DB; der Originalwert bleibt daneben erhalten und einsehbar, angezeigt wird der bearbeitete.
- R16 (forge §Metadaten; Decision 2026-11-20): Ein Bild hat genau eine Bewertung 0–5 und frei vergebene, global sichtbare Tags; IPTC/XMP-Keywords werden beim Import zu Tags. Autor ist der Fotograf aus EXIF/IPTC, getrennt vom Owner (Uploader).
- R17 (forge §Ortsmodell; draft §5; Decision 2026-11-21): Ein Bild mit GPS erhält per Reverse-Geocoding Kontinent/Land/Provinz/Region/Stadt; dieser Ort ist die Wahrheit für Karte, Geo-Navigation und Geo-Filter. Sind alle Anbieter erschöpft oder nicht erreichbar, wartet das Geocoding bis zum nächsten Kontingent-Reset, statt aufzugeben. Orte von Anbietern mit Pflicht zur Namensnennung (OSM/ODbL) zeigen diese in jeder Oberfläche, auch öffentlich.
- R18 (forge §Ortsmodell; Decision 2026-11-21): Ein Bild ohne GPS, oder solange es noch nicht geocodiert ist, erhält seinen Ort aus dem Ordnerpfad, markiert als „ungefähr“, sonst erscheint es unter „Ort fehlt“; weichen Ordner und GPS voneinander ab, landet es auf einer Prüfliste.
- R19 (forge §Ortsmodell, §Scope): Nutzer navigieren nach Ort (Kontinent bis Stadt) und nach dem Ordnerbaum.
- R20 (forge §Performance; draft §5): Die Karte clustert serverseitig, und Nutzer suchen auf ihr per Radius, Polygon und Bounding Box.
- R21 (draft §9): Das Raster ist responsiv und lädt per Lazy Loading und Infinite Scroll nach; die Detailansicht zeigt Bild, Metadaten, Karte, Tags und Bewertung.
- R22 (draft §7): Die Volltextsuche durchsucht Titel, Beschreibung, Keywords, Kamera und Objektiv; Filter nach Land, Region, Stadt, Kamera, Objektiv, Jahr, Monat, Autor, Bewertung und Tags lassen sich per UND kombinieren.
- R23 (forge §Scope; draft §13, §19): Der Admin sieht Worker, Job-Queue, Datenbank, Speicherverbrauch, geparkte Duplikate und die Prüfliste. Er stößt für ausgewählte oder alle Bilder ein Reindexing an: Metadaten neu aus dem Original (bearbeitete Werte bleiben), Ableitungen und Suchindex neu. Metriken (Uploads/s, Jobs, Queue-Länge, API-Antwortzeiten, DB-Latenz) werden erfasst.
- R24 (forge §Architektur und Betrieb; Decision 2026-11-21): Originale gehen nie verloren (RPO 0), und die DB hat PITR. Ein dokumentierter, getesteter Restore auf einen leeren Host stellt in ≤ 24 h DB, Originale und das Protokoll der Einengungen und Löschungen her; danach bleibt Zurückgenommenes zurückgenommen und Gelöschtes gelöscht, und bis dahin zeigt die öffentliche Galerie die Wartungsseite. Ableitungen werden nicht gesichert, sondern danach aus den Originalen neu erzeugt; bis dahin zeigen Raster und Galerie Platzhalter. Die Admin-Ansicht warnt bei ungesicherten Originalen.
- R25 (forge §Architektur und Betrieb): Ist die Anwendung nicht erreichbar, zeigt die öffentliche Galerie eine statische Wartungsseite.
- R26 (draft §13; Decision 2026-11-20): Wer `image.delete` hat, verschiebt ein Bild in den Papierkorb, wo es überall unsichtbar und wiederherstellbar ist. Nur ein Admin löscht endgültig Original und Ableitungen; Backups laufen nach ihrer Aufbewahrungsfrist aus.
- R27 (Decision 2026-11-20): Ein Bild ohne Aufnahmezeit in den Metadaten trägt das Kennzeichen „Zeit unsicher“ und ist danach filterbar; Berechtigte korrigieren die Zeit in der DB, der Dateiname bleibt.
- R28 (Decision 2026-11-20): Ordnerbaum und Alben sind getrennt: Ein Upload legt nie ein Album an, Alben werden bewusst angelegt und befüllt, und ein Bild kann in mehreren Alben liegen.
- R29 (Decision 2026-11-20): Der Owner setzt jede Sichtbarkeit seiner Alben und Bilder. Ein Admin ändert bei fremden Inhalten Gruppen- und public-Sichtbarkeit, public nur mit `album.publish`; Privates sieht und setzt kein Admin.
- R30 (Decision 2026-11-20): Beim Veröffentlichen bestätigt man, dass die Einwilligungen abgebildeter Personen vorliegen. Ein Bild lässt sich in einem Schritt aus allen public-Alben zurücknehmen, und die öffentliche Galerie verlinkt Impressum, Datenschutz und Kontakt.
- R31 (Decision 2026-11-20): Das Copyright einer Ableitung ist das des Bildes (bearbeiteter Wert vor Dateiwert), sonst „© <Jahr> <Autor>“, sonst ein globaler Default des Admins.
- R32 (Decision 2026-11-21): Der Admin pflegt die Geocoding-Anbieter (Reihenfolge, aktiv, Limits, API-Keys) in der Weboberfläche; Änderungen wirken ohne Neustart, und ein API-Key wird nie angezeigt, nur ob er gesetzt ist.

## Constraints

- Stack: Linux, C++23 mit Drogon (REST/JSON, Sessions, RBAC), PostgreSQL 18 + PostGIS (`GEOGRAPHY(Point,4326)`) + PGroonga, Angular 22 mit Spartan/UI, H2O (TLS, HTTP/2, HTTP/3, Reverse Proxy), libvips mit libheif für AVIF. Einzelne Komponenten dürfen Rust oder Go sein; die WebAuthn-Prüfung ist Rust (forge Einleitung; draft §2, Technologie-Stack; Decision 2026-11-21).
- Ein Host mit Podman (Quadlets): die Container `h2o`, `flagme` und `postgresql` laufen je als systemd-Unit, im `flagme`-Container jeder Service als eigene systemd-Unit. Synchron gRPC über UDS (Protobuf) mit Socket Activation und gRPC Health; asynchron dauerhafte Job-Queues in PostgreSQL. Kein NATS; FlatBuffers nur zwischen api und Frontends, intern nie (forge §Architektur und Betrieb, §Verworfen; Decision 2026-11-21, 2026-11-22).
- Leistungsziele bei 1 Mio. Bildern auf dem Referenzprofil M (8C/16T, 32 GB, NVMe), auch während eines laufenden Imports: Karte < 300 ms, Raster nächste Seite < 150 ms p95, Suche/Filter < 200 ms p95, API-Metadaten < 100 ms p95; Thumbnails statisch vom Edge. Profil S (4 Kerne, 8 GB) ist unterstützter Einstieg mit dokumentiert niedrigeren Zielen; die Architektur ist für alle Profile gleich, und die Profilzahlen gelten bis zum Benchmark in epic-import-pipeline als Schätzung (forge §Performance; Decision 2026-11-21).
- Import ≥ 10.000 Bilder/h (JPEG/HEIC) auf Profil M mit niedrig priorisierten Workern; 10 gleichzeitige Uploads; Bestand ~200.000, Ziel 1 Mio. (forge §Performance; draft §17).
- 99,5 % Verfügbarkeit ohne Wartungsfenster; Datensicherheit geht vor Hochverfügbarkeit (forge §Architektur und Betrieb).
- Sicherheit: TLS 1.3, CSP, XSS- und CSRF-Schutz, Rate Limiting, Audit Logging; Sessions serverseitig ohne JWT, Session-Cookies Secure, HttpOnly, SameSite=Strict (draft §11, §17; Decision 2026-11-21).
- Betrieb: strukturierte JSON-Logs, Metrik-Export per OpenTelemetry (draft §18, §19).
- Originale werden nie verändert; Metadaten werden nie in Dateien zurückgeschrieben (forge §Metadaten, §Verworfen).
- Ablage lokal unter `/storage` auf einem redundanten, lokal eingehängten Block-Volume, keine Netzfreigabe; MinIO/S3 (draft §14) erst mit der Erweiterung „S3-native“ (Decision 2026-11-21).
- Reverse-Geocoding darf externe Online-Anbieter mit Fallback-Kette nutzen; dafür verlassen nur auf ≈ 100 m gerundete Koordinaten den Host, und nur der Geocoder spricht mit Anbietern (Decision 2026-11-21). Der Bestand wird asynchron über Tage geocodiert, und das Import-Ziel von 10.000 Bildern/h gilt ohne Geocoding (Decision 2026-11-20).

## Done when

- Der Bestand von ~200.000 Bildern ist per SCP/SFTP importiert, Duplikate sind geparkt, und auf Referenzprofil M wurden ≥ 10.000 Bilder/h gemessen.
- Bei 1 Mio. Bildern (Bestand plus synthetische Last) und laufendem Import erreichen auf Referenzprofil M Karte, Raster, Suche und API die Leistungsziele.
- Ein anonymer Besucher sieht nur Bilder aus public-Alben, mit Positivliste und GPS-Regel angewendet; ein Test mit Gruppen-, privaten und ausgeschlossenen Bildern zeigt keines davon öffentlich.
- Ein Restore aus dem Backup auf einen leeren Host stellt DB, Originale und Protokoll der Einengungen in ≤ 24 h her, alle Originale haben ihren SHA256 behalten, und kein zurückgenommenes oder gelöschtes Bild ist danach öffentlich.
- V1 läuft in Produktion auf dem Host, mit Admin-Monitoring, Wartungsseite und gemessener Verfügbarkeit.

## Boundaries

- Nicht in V1: Slideshow, PWA, XMP-Sidecar-Export, pHash (visuelle Duplikate), RAW-Formate. Sie kommen in V1.x (forge §Scope).
- Später: Desktop- und Mobile-App sowie die Zukunftserweiterungen des Entwurfs (Gesichtserkennung, KI-Klassifikation, Landmarken, Ähnlichkeitssuche, Multi-Tenant, S3-native/MinIO, OpenSearch, Föderation) (forge §Scope; draft §14, §20).
- Verworfen: NATS, FlatBuffers intern (zwischen Services), 99,9 % Verfügbarkeit, 50 gleichzeitige Uploads, Freigabe-Links, Metadaten in Ableitungen, Rückschreiben in Originale, Rechte pro Person/Bild, offene Registrierung (forge §Verworfen, §Nutzer und Sichtbarkeit).
- Kein Multi-Host-Betrieb.
- Touch point: Online-Geocoding-Anbieter (Nominatim, Google, OpenCage, Bing) — per eigener `geocoding`-Bibliothek abgefragt; owner: epic-ort-karte
- Touch point: Kartenkachel-Quelle — für die Kartenansicht eingebunden; owner: epic-ort-karte
- Touch point: SCP/SFTP-Server des Hosts (OpenSSH) — für den Admin-Import konfiguriert; owner: epic-import-pipeline
- Touch point: Backup-Ziel — für Originale (RPO 0) und DB-PITR genutzt; owner: epic-datensicherheit-betrieb
- Tracer-Pfad: Web-Upload → Import und Pipeline → Metadaten und Ortsmodell → Sichtbarkeit → Raster/Karte für Angemeldete → public-Album in der öffentlichen Galerie.

## References

- forge — _bmad-output/initiative-geogallery/forge-geogallery/forge-geogallery.md, alle Abschnitte (geht bei Widerspruch vor)
- draft — /Volumes/1TB_NVME/Documents/obsidian-snippets/GeoGallery.md, §2–§19 und Technologie-Stack
- spine — _bmad-output/initiative-geogallery/architecture-geogallery/architecture-geogallery.md, AD-1..AD-51: Service-Schnitt, Verträge, Sichtbarkeitsfilter, Storage, Job-Queues, Hardware-Profile, Backup-Vertrag; bindet alle Epics (Herleitung in `.memlog.md` daneben, Reviews in `reviews/`)
- forge-memlog — _bmad-output/initiative-geogallery/forge-geogallery/.memlog.md, Herleitung der Forge-Entscheidungen
- prior-art — /Users/zb_bamboo/Documents/DEV/CPP/gallery_snippets: Kandidaten zur Wiederverwendung, entschieden in der Architektur. Am nächsten dran: `drogon_auth` (TOTP, RBAC, gRPC-Health), `drogon_audit_plugin` (Audit-Log), `geocoding` (Multi-Provider-Reverse-Geocoding mit Fallback und Kontingenten), `img2avif` (AVIF in mehreren Größen), `web-gallery_backend_h2o-drogon` (H2O+Drogon, Ort aus Ordnern, ER-Modell), `photo_indexer` (Metadaten-Benchmark)

## Notes

- Decision: 2026-11-20, Nutzer — bei Widerspruch gilt forge vor draft. Danach entschieden: kein NATS, keine FlatBuffers, 99,5 % statt 99,9 %, 10 statt 50 Uploads, `/storage/uploads` statt `/storage/temp`, Slideshow/PWA/pHash/RAW in V1.x.
- Decision: 2026-11-20, Nutzer — die Snippets in `gallery_snippets` sind Kandidaten zur Wiederverwendung (prior-art), keine Quelle für Anforderungen; die Wiederverwendung entscheidet die Architektur.
- Decision: 2026-11-20, Nutzer — Reverse-Geocoding über Online-Anbieter mit Fallback ist erlaubt (siehe Constraints).
- Decision: 2026-11-20, Nutzer — die Zielhardware steht noch nicht fest. Die Architektur nennt eine Mindestausstattung, und erst gegen diese werden Import-Durchsatz und Leistungsziele verbindlich gemessen.
- Decision: 2026-11-20, Nutzer — Zeit unsicher (R27), Pfad ≠ Album (R28), Papierkorb (R26), Sichtbarkeitsrechte (R29), Datenschutz beim Veröffentlichen ohne Gesichtserkennung (R30), Copyright-Kette (R31).
- Decision: 2026-11-20, Nutzer — Epic-Schnitt nach der Regel „ein Epic = eine Fähigkeit der Quelle, gemeinsam geliefert“; 8 Epics in `tickets/tickets.toml`. Der Sichtbarkeitsfilter (angemeldet/Gruppen/privat) entsteht früh in epic-import-pipeline, damit Raster, Suche und Karte von Anfang an mit ihm messen; Alben, Vererbung und Bedienung kommen in epic-alben-sichtbarkeit. Der echte Bestand wird erst nach epic-datensicherheit-betrieb importiert.
- Decision: 2026-11-20, Nutzer — geteilte Entscheidungen (DB-Schema, gRPC/Protobuf-Verträge, Storage-Layout, Job-Queue, Sichtbarkeitsfilter, 1-Mio.-Testbestand, Wiederverwendung der Snippets, Mindesthardware) legt eine Architektur-Spine per `bmad-architecture` fest; die Epics verweisen darauf.
- Decision: 2026-11-20, Nutzer — Backups werden nach endgültigem Löschen 90 Tage aufbewahrt; danach ist das Bild auch aus den Backups entfernt (R26, R30).
- Decision: 2026-11-21, Nutzer — die Architektur-Spine ist Companion der Initiative; wo sie bewusst abweicht, gilt sie vor forge: PostgreSQL 18 statt 19, Podman-Container mit systemd-Units darin, FlatBuffers zwischen api und Frontends, Sichtbarkeitsbasis „alle angemeldeten“ mit Obergrenze des Owners (R4, R5), Leistungsziele je Hardware-Profil (M Referenz, S Einstieg), Restore = DB + Originale + Protokoll (R24), AVIF über libvips/libheif, Copyright immer öffentlich (R7), Rust für WebAuthn.
- Decision: 2026-11-21, Nutzer — durch die Spine geklärt: Geocoding-Anbieter und Kontingente (Konfiguration in der DB, Start mit Nominatim; die Ebene „Region“ bildet epic-ort-karte ab), Zielhardware (Profile, verbindlich nach dem Benchmark in epic-import-pipeline), „privat“ auch gegenüber dem Admin (R29), Wiederverwendung der Snippets samt ihrer Konflikte (spine AD-36).
- Decision: 2026-11-22, Nutzer — das Projekt heißt „FlagMe - The Geo Gallery“; der technische Kurzname ist `flagme` (Container, Image, Pfade, Proto-Pakete, Secrets), Service-, Bibliotheks- und Web-Repos heißen `flagme-<name>` (z. B. `flagme-api`), das Haupt-Repo `flag-me_gallery` hält weiter `contracts/`. Die Planungsordner (`initiative-geogallery` usw.) behalten ihren Namen. Die Spine nennt noch `geogallery` und wird per `bmad-architecture` nachgezogen; bis dahin gilt diese Entscheidung vor der Spine.
