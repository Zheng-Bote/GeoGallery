---
type: review
lens: reconcile (Ticket ↔ Spine)
target: architecture-geogallery.md (status draft, updated 2026-11-21)
ticket: initiative-geogallery.md
decision-log: .memlog.md
date: 2026-11-21
---

# Reconcile-Review: Initiative ↔ Architektur-Spine

Abgleich jeder Anforderung (R1–R31), jedes Constraint-Punkts, der Done-when-Punkte, Boundaries/Touch points und Notes gegen die Spine. Grundlage für Entscheidungen ist die Memlog-Datei. Die Spine wurde nicht verändert.

Legende: **OK** = gelandet · **TEIL** = teilweise / implizit · **LÜCKE** = fehlt · **ABW** = bewusste Abweichung (Nutzerentscheidung, Ticket muss von der PO nachgezogen werden)

---

## 1. Lücken mit Priorität (empfohlene Korrektur)

| # | Lücke | Ticket-Quelle | Vorschlag |
| --- | --- | --- | --- |
| G1 | **Edge-Sicherheit fehlt fast vollständig:** TLS 1.3 (nur „TLS“), CSP, XSS-Schutz, Session-Cookie-Flags `Secure; HttpOnly; SameSite=Strict` stehen nirgends. Nur CSRF und Rate Limit (AD-10) und Audit (AD-35) sind verankert. Was auditiert wird, ist offen. | Constraints „Sicherheit“; R2 (Cookie-Sessions statt JWT, Source conflict) | Neues AD „Edge- und Session-Sicherheit“: h2o nur TLS 1.3; CSP-Basis (`default-src 'self'`, `img-src` für signierte Ableitungs-URLs und Kachelquelle); api setzt Session-Cookie mit Secure/HttpOnly/SameSite=Strict, kein JWT; Rate-Limit-Schlüssel (IP/Konto/Login). Auditereignisse als Mindestliste (Login, Rollen/Status, Sichtbarkeit/Publish inkl. Einwilligung R30, endgültiges Löschen) oder ausdrücklich nach epic-iam/epic-datensicherheit-betrieb verschieben. |
| G2 | **Öffentliche Projektion gilt nicht ausdrücklich für Karte, FlatBuffers und Cluster.** AD-20 sagt „anonymer Principal liest nur die öffentliche View“. AD-33 (Quadkey je Ort) und AD-30 (`.fbs` mit „denselben Feldern“) sagen nicht, dass bei anonymen Nutzern vergröbertes/ausgeblendetes GPS (R8) und die Positivliste (R7) auch für Cluster, Quadkeys, BBox/Radius-Suche und das Binärformat gelten. Exakte Quadkeys bis Zoom 18 auf public-Clustern verraten exakte Orte. Die Regeln „Schnittmenge der Positivlisten“ (R7) und „aus vor vergröbert vor exakt“ (R8) bei mehreren public-Alben fehlen ebenso wie das Vergröberungsverfahren. | R7, R8, Done when (anonymer Besucher) | AD-20 erweitern: öffentliche View materialisiert *öffentliche* Koordinate + öffentlichen Quadkey (vergröbert/NULL), berechnet mit Schnittmenge/Strengste-Regel; AD-33 clustert anonyme Anfragen nur darüber; AD-30: für anonyme Principals ein eigener `.fbs`-Typ (oder Felder fehlen physisch), nie Nullung im Client. Vergröberungsraster (z. B. Quadkey-Zoom N) als Convention oder Deferred → epic-oeffentliche-galerie. |
| G3 | **Restore ≤ 24 h ist mit „Ableitungen nicht sichern, neu erzeugen“ bei 1 Mio. Bildern nicht vereinbar.** Bei ≥ 10.000/h (Profil M) dauert die Neuerzeugung ~100 h. PITR wird nur im Ticket genannt; die Spine erwähnt PITR nicht, ebenso wenig die Konsistenz zwischen DB-Zeitpunkt und Original-Bestand beim Restore und die 90-Tage-Aufbewahrung nach endgültigem Löschen (Note 2026-11-20). | R24, R26, Done when (Restore), Note Backups 90 Tage | AD-27 ergänzen: (a) „Restore abgeschlossen“ definieren (DB + Originale; Ableitungen priorisiert nachziehen) **oder** Ableitungen mitsichern; (b) PITR der DB als Pflicht des Backup-Adapters; (c) Restore-Reihenfolge und Abgleich DB ↔ `originals/` (SHA256-Prüfung als Werkzeug, wird von Done when gefordert); (d) Aufbewahrung 90 Tage als Vertragsparameter. Bei (a) PO informieren, da Interpretation von R24. |
| G4 | **Upload-Transport und SCP/SFTP sind nicht architektonisch festgelegt.** Touch point OpenSSH (owner epic-import-pipeline) fehlt völlig: sshd läuft auf dem Host, `/storage` ist im geogallery-Container, Schreibrecht für `uploads/` hat nur ingest (AD-13). Offen: wie erkennt ingest fertige Uploads (inotify/Scan, Teildateien), welcher Unix-User schreibt `uploads/<username>/`, wie wird der Username einem Konto zugeordnet? Für Web-Uploads ist offen, wie große Dateien h2o → api → ingest erreichen (Streaming über gRPC? Direkt-Spool?), ebenso das Limit „10 gleichzeitige Uploads“. | R9, Constraint Import (10 gleichzeitige Uploads), Done when (Import per SCP/SFTP), Touch point SFTP | Neues AD oder Erweiterung von AD-13: Upload-Eingang (Spool-Pfad, atomarer Abschluss per Rename aus `.part`, Erkennung), SFTP-Chroot pro Admin auf `uploads/<username>/` mit Gruppenrecht für ingest; Web-Upload-Pfad (Streaming-RPC oder Spool) und Upload-Limit in api/ingest. Alternativ ausdrücklich Deferred → epic-import-pipeline mit diesen Leitplanken. |
| G5 | **Admin-Oberflächen außerhalb der catalog-RLS können Privates zeigen.** R29/Note: „Privates sieht kein Admin“. Der Duplikat-Parkplatz (ingest), Job-Views (Pfade, Fehlertexte), die Prüfliste, der Audit-Log und Reindex „alle Bilder“ laufen nicht über AD-20. Ordnerpfade und Dateinamen privater Bilder können so beim Admin landen. | R12, R23, R29, Note „privat gilt auch gegenüber dem Admin“ | AD-18/AD-20 ergänzen: Jede Admin-Ansicht, die Bild- oder Pfaddaten zeigt, geht über catalog mit Principal; ingest/media/geocoder-Views zeigen nur IDs und Zustände, keine Pfade/Metadaten privater Bilder; geparkte Duplikate eines privaten Bilds sieht nur dessen Owner. |
| G6 | **Verfügbarkeit 99,5 % ohne Wartungsfenster und deren Messung.** AD-28 begründet Expand/Contract mit „rollierendem Neustart“. Ein Update des einen `geogallery`-Containers (AD-25) startet aber alle Services auf einmal neu, und ein Update des `postgresql`-Containers unterbricht die DB. Wie Verfügbarkeit gemessen wird (Done when: „gemessene Verfügbarkeit“), steht nirgends. | Constraint 99,5 %; Done when V1 Produktion | AD-25 oder AD-28 ergänzen: Update-Verfahren (Service-Binaries im Container einzeln per systemd neu starten statt Image-Wechsel, oder bewusst akzeptierte Kurzunterbrechung mit Wartungsseite R25 im Budget). Verfügbarkeitsmessung (externer Probe auf public + /api/v1/health) als Convention oder Deferred → epic-datensicherheit-betrieb. |
| G7 | **HEIC/TIFF-Dekodierung und Pipeline-Schritt Geo-Validierung.** Der Stack nennt libvips und libavif, aber kein libheif (HEIC, das Import-Ziel nennt JPEG/HEIC ausdrücklich) und kein libtiff/libpng/libwebp. Die „Geo-Validierung“ aus R13 und die Ableitung des Orts aus dem Ordnerpfad (R18, Note „bis geocodiert gilt Ordnerort“) haben keinen Besitzer: Wer parst den Pfad zu Kontinent/…/Stadt, wer markiert „ungefähr“, wer füllt die Prüfliste? | R13, R18, Note R18 | Stack: libheif (+ Codecs) festnageln. Capability-Map/AD-15: Ort aus Ordnerpfad und Geo-Validierung (Koordinate plausibel, Abweichung Ordner ↔ GPS → Prüfliste) gehören catalog, ausgelöst bei `Confirm` bzw. `ReportLocation`. |
| G8 | **Entzug und Änderung wirken auf ausgelieferte Ableitungen verzögert bzw. gar nicht.** (a) Signierte URLs gelten bis Fensterende (≤ 1 h, wegen Rundung bis ~2 h), auch nach Papierkorb (R26 „überall unsichtbar“), Zurücknehmen (R30 „in einem Schritt“) und endgültigem Löschen. Kopierte URLs wirken in dieser Zeit wie Freigabe-Links (Boundaries: verworfen). (b) Eine Änderung von Copyright/Autor (R31, override) erzwingt keine neue Ableitung, obwohl das Copyright in der Ableitung steckt. | R26, R30, R31, Boundaries „Freigabe-Links verworfen“ | (a) In AD-12 die maximale Entzugslatenz ausdrücklich nennen und an die PO geben (siehe Abweichung D6); optional kürzeres Fenster für public. (b) AD-15: Änderung an Copyright-Eingaben erhöht die Version und bestellt media neu. |

---

## 2. Weitere Befunde (niedriger)

- **R3 „Rollen ändern nie, was ein Nutzer sieht“** (TEIL): AD-18 „Admin hat keine Ausnahme“ deckt es nur für Admins ab. Regel ergänzen: Die RLS-Policy (AD-20) wertet nur `user_id` und `group_ids` aus, nie `permissions`.
- **R1 Kontostatus aktiv/deaktiviert/gesperrt** (TEIL): Nur die 60-s-Wirkung der Deaktivierung steht im Deferred-Teil. Ob „gesperrt“ (z. B. nach Fehlversuchen) automatisch gesetzt wird, ist offen → epic-iam.
- **R6 „eine Ableitung pro Bild für intern und öffentlich“** (TEIL): Folgt implizit aus dem Pfad `<image_id>_<größe>`, ist aber nicht als Regel formuliert (Verbot getrennter public-Varianten).
- **R16 Keywords → Tags beim Import; Autor ≠ Owner** (TEIL): AD-6 nennt Tags und Autor nicht. Ein Satz in AD-6 genügt.
- **R30 Einwilligungsbestätigung** (LÜCKE): Wo sie gespeichert wird (catalog am Album, Audit-Eintrag), ist offen. Die Links auf Impressum/Datenschutz/Kontakt sind Frontend-Sache → epic-oeffentliche-galerie, sie stehen aber nicht im Deferred-Teil.
- **R23 Speicherverbrauch, DB-Status** (TEIL): Wer misst den Speicherverbrauch von `/storage` (ingest? api?) und den DB-Zustand im fremden Container, ist offen. Die Metrikliste aus R23 (Uploads/s, Queue-Länge, API-Antwortzeit, DB-Latenz) ist nicht als Pflicht-Metrikset in der OTel-Convention festgehalten.
- **R11 SHA256 bleibt gültig** (TEIL): Keine periodische Fixity-Prüfung. Done when verlangt den SHA256-Nachweis nach dem Restore → Werkzeug fehlt (siehe G3).
- **Geocoding-Egress** (TEIL): Nur geocoder soll den Host nach außen verlassen. Weil alle Services im selben Container laufen (AD-25), ist das per Netz nicht trennbar. Als Regel festhalten (z. B. Egress-Proxy nur für den geocoder-User, oder nftables nach UID). Dass „Import ≥ 10.000/h ohne Geocoding“ gilt, steht nicht ausdrücklich da (AD-17 nennt geocoder nicht). Die Datenschutzwirkung „GPS privater Bilder verlässt den Host“ ist vom Ticket akzeptiert.
- **Kartenkachel-Quelle** (TEIL): Deferred ist in Ordnung. Für CSP (G1) und Datenschutz der öffentlichen Galerie (die IP des Besuchers geht an den Kachelanbieter, R30 Datenschutzseite) sollte eine Leitplanke mitgegeben werden (Proxy über h2o oder selbst gehostet).
- **„Thumbnails statisch vom Edge“** (OK mit Vorbehalt): Die mruby-Prüfung je Request ist ein offener Prüfpunkt, und der Rückfall „lokaler Prüfdienst“ gefährdet < 150 ms. Er ist bereits gelistet.
- **Boundaries V1.x/Später/Verworfen** (OK): Werden nicht verletzt. Ausnahmen sind FlatBuffers (D3) und Freigabe-Link-Charakter der signierten URLs (G8/D6). Die Spine nennt die Boundaries nicht, das ist vertretbar.
- **Tracer-Pfad** (nicht in der Spine): Gehört zum Epic-Schnitt, ist hier nicht nötig.
- **Interne Inkonsistenz:** AD-12 sagt „HMAC-Schlüssel ist ein Podman-Secret“, AD-26 sagt „HMAC-Schlüssel in verschlüsselter Datei, Schlüssel als Podman-Secret“. Eine Formulierung wählen (h2o liest die libsodium-Datei vermutlich nicht).
- **Status-Markierungen ≠ Memlog:** Memlog #100 führt als `[ASSUMPTION]` (Bestätigung offen): zweistufiges Reserve/Confirm (AD-2), vollständige Submit-Eingaben (AD-15), Rechteprüfung beim Datenbesitzer (AD-10), HMAC-Key als Podman-Secret (AD-12) und catalog besitzt Katalog-Einstellungen (AD-1). AD-2, AD-12, AD-15 und AD-1 sind in der Spine trotzdem `[ADOPTED]`. AD-28 (Expand/Contract) ist im Memlog ebenfalls nur ASSUMPTION. Für AD-33 (Quadkey-Cluster) gibt es **keinen** Memlog-Eintrag. Vor `status: final` bestätigen lassen.

---

## 3. Abgleich je Anforderung (Kurzmatrix)

| R | Status | Spine-Ort / Bemerkung |
| --- | --- | --- |
| R1 | TEIL | iam (AD-1, AD-8); Status-Semantik offen; 60 s Deferred |
| R2 | TEIL | AD-24, AD-36; Cookie-Session statt JWT nicht ausdrücklich → G1 |
| R3 | TEIL | AD-10; „Rollen ändern Sicht nie“ nur über AD-18 → §2 |
| R4 | ABW/OK | AD-18 präzisiert (D4) |
| R5 | OK | AD-18, AD-19, AD-20 |
| R6 | TEIL | AD-12, AD-36; „eine Ableitung“ implizit |
| R7 | TEIL | AD-20; Schnittmenge, FlatBuffers-Pfad → G2 |
| R8 | TEIL | AD-20; Vorrangregel, Vergröberung, Cluster → G2 |
| R9 | LÜCKE | SFTP/Web-Upload-Transport → G4 |
| R10 | OK | AD-7, AD-13 (`renameat2`) |
| R11 | OK | AD-13 (0440), Ableitungspfad nach image_id |
| R12 | TEIL | AD-2; Sichtbarkeit für Admin vs. privat → G5 |
| R13 | TEIL | AD-14–16; HEIC-Lib, Geo-Validierung → G7 |
| R14 | OK | AD-6 raw, AD-36 Exiv2 |
| R15 | OK | AD-6 |
| R16 | TEIL | Tags/Bewertung in catalog; Keywords→Tags, Autor/Owner nicht genannt |
| R17 | OK | geocoder, AD-15; Anbieter Deferred |
| R18 | TEIL | Ordner→Ort-Besitz, Prüfliste-Auslöser → G7 |
| R19 | OK | catalog, Map |
| R20 | TEIL | AD-33; anonym/vergröbert → G2 |
| R21 | OK | AD-30, AD-31 |
| R22 | OK | AD-32, typisierte Spalten |
| R23 | TEIL | AD-15/16, OTel; Pflichtmetriken, Speicher → §2 |
| R24 | TEIL | AD-27; PITR, Restore ≤ 24 h vs. Ableitungen → G3 |
| R25 | OK | h2o (AD-25) |
| R26 | TEIL | AD-5, AD-18; 90 Tage, Entzugslatenz → G3, G8 |
| R27 | OK | AD-7 |
| R28 | OK | AD-1 (Ordnerbaum und Alben getrennt im catalog) |
| R29 | TEIL | AD-10, AD-18, AD-19; Admin-Flächen außerhalb RLS → G5 |
| R30 | TEIL | Einwilligung-Speicherort, Links, Entzugslatenz → §2, G8 |
| R31 | TEIL | AD-6, AD-15; Neuerzeugung bei Copyright-Änderung → G8 |

### Constraints

| Constraint | Status | Bemerkung |
| --- | --- | --- |
| Stack Linux/C++/Drogon/PostGIS GEOGRAPHY/Angular 22/H2O/libvips+libavif | TEIL/ABW | PG18 statt 19 (D1); Rust/Go zusätzlich (D7); PGroonga neu; libheif fehlt (G7) |
| Ein Host, systemd-Units, gRPC/UDS, Socket Activation, gRPC Health, PG-Queue | ABW/OK | Podman-Container (D2); sonst OK (AD-9, AD-14–16, AD-23, AD-25) |
| Kein NATS | OK | |
| Keine FlatBuffers | ABW | D3 |
| Leistungsziele bei 1 Mio. auch während Import | TEIL/ABW | AD-21, AD-22; Profil S lockert (D5) |
| Import ≥ 10.000/h, 10 gleichzeitige Uploads | TEIL | ≥ 10.000/h nur Profil M; Upload-Limit fehlt (G4) |
| 99,5 % ohne Wartungsfenster | TEIL | AD-28; Update-Verfahren, Messung → G6 |
| TLS 1.3, CSP, XSS, CSRF, Rate Limit, Audit, Cookie-Flags | LÜCKE (teilweise) | → G1 |
| JSON-Logs, OTel | OK | Conventions, AD-23 |
| Originale unveränderlich, kein Rückschreiben | OK | AD-6, AD-13 |
| Geocoding online, Egress, asynchron, Import-Ziel ohne Geocoding | TEIL | Egress-Begrenzung, AD-17 ohne geocoder → §2 |

### Done when

| Punkt | Status |
| --- | --- |
| 200.000 per SCP/SFTP, Duplikate geparkt, ≥ 10.000/h gemessen | TEIL: SFTP-Pfad fehlt (G4); Benchmark gegen Profil M (AD-22) |
| 1 Mio. mit laufendem Import erreicht Ziele | OK: AD-21, AD-22 (Profil M) |
| Anonymer Besucher, Test mit Gruppen/privat/ausgeschlossen | TEIL: AD-21 deckt die Daten ab; Projektion für Karte/FlatBuffers → G2 |
| Restore ≤ 24 h, SHA256 erhalten | LÜCKE → G3 |
| Produktion mit Admin-Monitoring, Wartungsseite, gemessener Verfügbarkeit | TEIL → G6 |

### Boundaries / Touch points

| Punkt | Status |
| --- | --- |
| Nicht in V1, Später, Kein Multi-Host | OK (nicht verletzt) |
| Verworfen: FlatBuffers | ABW (D3) |
| Verworfen: Freigabe-Links | TEIL: Restrisiko durch gültige signierte URLs (G8, D6) |
| Touch point Geocoding-Anbieter | OK, Deferred |
| Touch point Kartenkachel-Quelle | OK, Deferred (CSP-/Datenschutz-Leitplanke fehlt) |
| Touch point SCP/SFTP (OpenSSH) | LÜCKE → G4 |
| Touch point Backup-Ziel | OK, AD-27 hosting-spezifisch (PITR → G3) |

### Notes

| Note | Status |
| --- | --- |
| privat gilt auch gegenüber Admin | TEIL: AD-18, Lücke G5 |
| Ablage lokal `/storage` | OK: AD-13 (präzisiert: Block-Volume, redundant) |
| Ort bis Geocoding aus Ordner | TEIL → G7 |
| Snippets = Kandidaten | OK: AD-36 |
| Source conflicts (drogon_auth, img2avif, photo_indexer, rz-gallery, web-gallery_worker) | OK: AD-36, AD-18 (Default angemeldet); JWT nicht ausdrücklich verboten (G1) |
| Zielhardware/Mindestausstattung | OK: AD-22 |
| Geteilte Entscheidungen in der Spine | OK: Schema, Verträge, Storage, Queue, Filter, Testbestand, Snippets, Hardware |
| Backups 90 Tage nach endgültigem Löschen | LÜCKE → G3 |
| Open question Geocoding-Reihenfolge/Region | OK, Deferred (deep-recon) |

---

## 4. Bewusste Abweichungen vom Ticket (PO muss das Ticket nachziehen)

Alle Punkte sind im Memlog als Nutzerentscheidung `[ADOPTED]` belegt, wenn nicht anders vermerkt.

| # | Abweichung | Ticket-Stelle | Spine / Memlog |
| --- | --- | --- | --- |
| D1 | **PostgreSQL 18 statt 19**, weil PostGIS/PGroonga für PG19 unbestätigt sind; Major-Upgrade ist Betriebssache | Constraints Stack | AD-37; Memlog #97 |
| D2 | **Podman + Quadlet, drei Container** (h2o, geogallery mit systemd innen, postgresql) statt „Komponenten als systemd-Units“ direkt auf dem Host; widerspricht auch dem Note-Satz „Docker widerspricht Ein-Host/systemd“ | Constraints Ein Host; Notes Source conflict | AD-25; Memlog #76–78 |
| D3 | **FlatBuffers zwischen api und Frontend** (per `Accept`), intern weiterhin keine | Constraints „keine FlatBuffers“; Boundaries Verworfen; Note Decision forge | AD-30, AD-28; Memlog #90 |
| D4 | **Präzisierte Sichtbarkeitsregel:** Bildeinstellung = Obergrenze; über mehrere Alben gilt die *weiteste* Albumeinstellung (nicht „restriktivste gewinnt“ über alles); Gruppen vereinigt; Ausschluss nur auf dem public-Weg; fremde Bilder höchstens bis zu ihrer Obergrenze | R4, R5 (auch R29) | AD-18, AD-19; Memlog #50–53 (ausdrücklich: Sofia soll R4/R5 nachziehen) |
| D5 | **Leistungsziele und Import-Durchsatz gelten pro Hardware-Profil;** Profil S ist offiziell unterstützt und darf p95 während des Imports reißen (~3.000–4.000/h). Das Ticket verlangt die Ziele „auf dem Zielhost, auch während eines Imports“ | Constraints Leistung/Import; Note Zielhardware | AD-22; Memlog #56, #58 |
| D6 | **Entzug von Sichtbarkeit wirkt auf Ableitungen erst nach Ablauf des URL-Fensters** (Default 1 h); bis dahin funktioniert eine kopierte URL wie ein befristeter Link | R26 „überall unsichtbar“, R30 „in einem Schritt zurücknehmen“, Boundaries „Freigabe-Links verworfen“ | AD-12; Memlog #37 („Entzug wirkt spätestens nach Fensterende“) |
| D7 | **Sprachoffenheit:** Komponenten dürfen auch Rust oder Go sein; `webauthn` ist Rust (webauthn-rs); Stack-Zusatz PGroonga, libsodium | Constraints Stack („C++ mit Drogon“) | AD-23, AD-24, AD-32, AD-26; Memlog #64, #65, #96 |
| D8 | **Microservice-Schnitt mit sieben Services und Database-per-Service** (Schema-Variante); das Ticket nennt nur „Komponenten“. Eher Ergänzung als Widerspruch, sollte aber in Constraints/Notes stehen, weil die Epics darauf verweisen | Constraints Ein Host; Note geteilte Entscheidungen | AD-1, AD-3; Memlog #19–21 |
| D9 | **Backup ist hosting-spezifisch** (Adapter + Sicherungsstatus); „ob Import auf Sicherung wartet“ ist Profileinstellung. Das berührt RPO 0 (R24), wenn ein Profil nicht wartet | R24, Touch point Backup-Ziel | AD-27; Memlog #70, #83 |
| D10 | **`/storage` muss lokales, redundantes Block-Volume sein** (keine Netzfreigabe); verschärft die Note „lokales Dateisystem“ | Notes Assumption Ablage | AD-13; Memlog #59 |

Der Zeitpunkt der Restore-Definition (G3, Variante a) wäre bei Annahme eine weitere Abweichung von R24 / Done when.
