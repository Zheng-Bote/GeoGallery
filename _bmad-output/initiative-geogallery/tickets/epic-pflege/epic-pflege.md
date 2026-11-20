---
type: epic
title: "Berechtigte pflegen Metadaten, Bewertung, Tags und löschen"
parent: initiative-geogallery
covers: ["R15", "R16", "R26", "R27", "R31"]
sources: []
companions: []
after: []
refined: false
risk: medium
---

# Berechtigte pflegen Metadaten, Bewertung, Tags und löschen

## Description

Metadaten werden nur in der DB bearbeitet mit Originalwert daneben, Bewertung und Tags gesetzt, unsichere Zeiten korrigiert; eine Copyright-Änderung erzeugt die Ableitungen über die Pipeline von epic-import-pipeline neu; Löschen verschiebt in den Papierkorb (überall unsichtbar, auch öffentlich, wiederherstellbar), nur der Admin löscht endgültig Original und Ableitungen, Backups laufen nach ihrer Frist aus. Bearbeitete Werte überstehen ein Reindexing aus epic-datensicherheit-betrieb (dieser Test liegt hier). Als letztes Epic misst es die Leistungsziele abschließend mit echtem Bestand plus synthetischer Last bis 1 Mio., vollem Filter (Gruppen, privat, Alben, public, Papierkorb) und laufendem Import.

## Outcome

Die Gruppe hält ihr Archiv aktuell und aufgeräumt, ohne dass ein Original verändert wird, und V1 erfüllt seine Leistungsziele unter Volllast.

## Done when

- Ein Berechtigter bearbeitet Metadaten, Bewertung, Tags und unsichere Zeiten; der Originalwert bleibt sichtbar, und ein Reindexing überschreibt die Bearbeitung nicht.
- Eine Copyright-Änderung erzeugt die Ableitungen neu.
- Ein gelöschtes Bild verschwindet überall, auch öffentlich, und ist wiederherstellbar; endgültig löscht nur der Admin, und nach 90 Tagen ist es aus den Backups entfernt.
- Mit echtem Bestand plus synthetischer Last bis 1 Mio., vollem Filter und laufendem Import halten alle Leistungsziele; V1 läuft in Produktion.

## Boundaries

- Kein Rückschreiben in Originale, kein XMP-Sidecar-Export (V1.x).

## References

- parent — _bmad-output/initiative-geogallery/initiative-geogallery.md, section Requirements (R15, R16, R26, R27, R31), Constraints, Notes

## Notes

- Waits on epic-datensicherheit-betrieb because: Backup mit Aufbewahrungsfrist, das endgültiges Löschen berücksichtigt.
- Waits on epic-raster-suche because: Detailansicht, in der bearbeitet wird.
- Waits on epic-oeffentliche-galerie because: öffentlicher Sichtbarkeitsfilter, den der Papierkorb mit ausblendet.
