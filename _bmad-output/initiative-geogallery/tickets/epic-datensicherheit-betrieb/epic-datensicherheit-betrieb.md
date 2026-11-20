---
type: epic
title: "Originale sind gesichert, der Bestand ist importiert"
parent: initiative-geogallery
covers: ["R23", "R24"]
sources: []
companions: []
after: []
refined: false
risk: medium
---

# Originale sind gesichert, der Bestand ist importiert

## Description

Originale haben RPO 0 auf ein Backup-Ziel, die DB PITR, ein dokumentierter Restore ist auf einem leeren Host in ≤ 24 h getestet; der Admin sieht Worker, Queue, DB und Speicher, stößt Reindexing an (Pipeline erneut, inkl. Suchindex aus epic-raster-suche; dass bearbeitete Werte es überstehen, prüft epic-pflege), Metriken gehen per OpenTelemetry raus, und die Verfügbarkeit wird gemessen. Danach wird der echte Bestand (~200.000) importiert und der Durchsatz auf der Mindestausstattung gemessen. Die Prüfliste-Ansicht liefert epic-ort-karte.

## Outcome

Die Originale der Gruppe sind gegen Verlust gesichert, der Admin sieht den Zustand des Systems, und der echte Bestand ist im Archiv.

## Done when

- Jedes neu importierte Original ist sofort auf dem Backup-Ziel (RPO 0); die DB lässt sich per PITR auf einen Zeitpunkt zurücksetzen.
- Ein dokumentierter Restore auf einen leeren Host ist in ≤ 24 h abgeschlossen, und alle Originale haben ihren SHA256 behalten.
- Der Admin sieht Worker, Queue, DB, Speicher und Duplikate und stößt ein Reindexing für ausgewählte oder alle Bilder an; Metriken gehen per OpenTelemetry raus, und die Verfügbarkeit wird gemessen.
- Der Bestand von ~200.000 Bildern ist importiert, und der Durchsatz auf der Mindestausstattung ist gemessen.

## Boundaries

- Die Prüfliste-Ansicht liefert epic-ort-karte.
- Der Test, dass bearbeitete Werte ein Reindexing überstehen, liegt in epic-pflege.
- Touch point: Backup-Ziel — genutzt hier.

## References

- parent — _bmad-output/initiative-geogallery/initiative-geogallery.md, section Requirements (R23, R24), Constraints, Notes

## Notes

- Waits on epic-import-pipeline because: Ablage /storage/originals, Job-Queue, Worker und Import, die gesichert, überwacht und für den Bestand genutzt werden.
- Waits on epic-raster-suche because: Suchindex, den das Reindexing neu aufbaut.
