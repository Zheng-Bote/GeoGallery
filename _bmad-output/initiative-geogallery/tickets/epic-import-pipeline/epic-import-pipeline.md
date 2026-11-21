---
type: epic
title: "Bilder kommen per Upload und Import ins Archiv"
parent: initiative-geogallery
covers: ["R4", "R6", "R9", "R10", "R11", "R12", "R13", "R14", "R16", "R27", "R31"]
sources: []
companions: []
after: []
refined: false
risk: high
---

# Bilder kommen per Upload und Import ins Archiv

## Description

Web-Upload (10 gleichzeitig) und Admin-SCP/SFTP (Server auf dem Host konfiguriert dieses Epic) landen in /storage/uploads; der Import verschiebt und benennt unveränderlich nach /storage/originals, parkt SHA256-Duplikate mit Anzeige für Admin/Owner, und die Job-Queue in PostgreSQL legt Metadaten in der DB ab (inkl. Bewertung, Keywords, Autor, Kennzeichen Zeit unsicher) und erzeugt AVIF und Thumbnails mit Copyright nach R31, statisch über H2O ausgeliefert, mit ≥ 10.000 Bildern/h auf Testdaten. Keywords werden zu Tags. Dieses Epic legt den zentralen Sichtbarkeitsfilter (Default alle angemeldeten, Gruppen aus epic-plattform-zugang, privat) pro Bild an, durch den jede spätere Abfrage läuft. Dieses Epic baut den gemeinsamen 1-Mio.-Testbestand-Generator (fester Seed, Bilder nur über den Importpfad des catalog, mit Verteilung von GPS-Clustern, ~15 % ohne GPS, Albengrößen und Sichtbarkeiten inkl. Gruppen, privat und ausgeschlossen; spine AD-21) und misst mit ihm die Zahlen der Hardware-Profile (spine AD-22).

## Outcome

Jedes Mitglied bringt Bilder per Web, der Admin per SCP/SFTP ins Archiv, und jedes Bild liegt danach unveränderlich abgelegt mit Metadaten, Ableitungen und Sichtbarkeit in der DB.

## Done when

- Ein Web-Upload (Einzel, Mehrfach, Drag & Drop, 10 gleichzeitig) und ein SCP/SFTP-Upload des Admins landen in /storage/uploads/<username>/… und werden ohne weiteren Schritt importiert.
- Das Original liegt unter /storage/originals/<Struktur>/<yyyy-mm-dd_hhmmss>[_n].<ext>, SHA256 unverändert; ein Duplikat ist geparkt und für Admin/Owner sichtbar.
- Jeder Job zeigt Status, wiederholt bei Fehlern und endet sonst im Fehlerzustand; alle sechs Formate erzeugen AVIF und Thumbnails ohne Metadaten außer Copyright nach R31.
- EXIF/IPTC/XMP stehen vollständig in der DB, Keywords als Tags, „Zeit unsicher“ gesetzt, wo die Aufnahmezeit fehlt; ein Nutzer bekommt per API nur Bilder, die der Sichtbarkeitsfilter ihm erlaubt.
- Der Generator erzeugt reproduzierbar 1 Mio. Bilder über den Importpfad; der Benchmark bestätigt oder korrigiert die Profilzahlen.
- Auf Testdaten werden auf Profil M ≥ 10.000 JPEG/HEIC pro Stunde importiert; das Epic läuft deployt auf dem Host.

## Boundaries

- Keine Oberfläche zum Durchsehen: Raster und Detail liefert epic-raster-suche.
- Kein Reverse-Geocoding: GPS wird nur gespeichert, den Ort liefert epic-ort-karte.
- Kein echter Bestand: der Import der ~200.000 Bilder folgt in epic-datensicherheit-betrieb.
- Touch point: SCP/SFTP-Server des Hosts (OpenSSH) — konfiguriert hier.

## References

- parent — _bmad-output/initiative-geogallery/initiative-geogallery.md, section Requirements (R4, R6, R9, R10, R11, R12, R13, R14, R16, R27, R31), Constraints, Notes

## Notes

- Waits on epic-plattform-zugang because: Konten und Anmeldung für Web-Upload und Upload-Verzeichnis pro Nutzer; Plattform-Baseline (systemd, UDS, DB, H2O, Deployment).
