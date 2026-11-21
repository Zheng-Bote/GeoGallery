---
type: epic
title: "Angemeldete finden Bilder in Raster, Detail und Suche"
parent: initiative-geogallery
covers: ["R14", "R21", "R22", "R27"]
sources: []
companions: []
after: []
refined: false
risk: high
---

# Angemeldete finden Bilder in Raster, Detail und Suche

## Description

Raster mit Lazy Loading und Infinite Scroll, Detailansicht mit allen Metadaten, Volltextsuche und Filter nach Kamera, Objektiv, Jahr, Monat, Autor, Bewertung, Tags und Zeit unsicher erreichen bei 1 Mio. Bildern (synthetischer Testbestand aus epic-import-pipeline) ihre Ziele: Raster < 150 ms p95, Suche/Filter < 200 ms p95, API-Metadaten < 100 ms p95, auch während eines Imports, gemessen durch den Sichtbarkeitsfilter von epic-import-pipeline.

## Outcome

Angemeldete finden jedes für sie sichtbare Bild in Raster, Detail und Suche schnell wieder, auch bei 1 Mio. Bildern und laufendem Import.

## Done when

- Ein angemeldeter Nutzer scrollt durch das Raster und öffnet die Detailansicht mit allen Metadaten, Tags und Bewertung.
- Volltext und Filter (Kamera, Objektiv, Jahr, Monat, Autor, Bewertung, Tags, Zeit unsicher) lassen sich per UND kombinieren.
- Auf dem synthetischen 1-Mio.-Testbestand mit verteilter Sichtbarkeit, Gruppen und GPS und laufendem Import gilt: Raster < 150 ms p95, Suche/Filter < 200 ms p95, API-Metadaten < 100 ms p95; Thumbnails kommen statisch von H2O.
- Das Epic läuft deployt auf dem Host.

## Boundaries

- Keine Geo-Filter und keine Karte: das liefert epic-ort-karte.
- Kein Bearbeiten: das liefert epic-pflege.

## References

- parent — _bmad-output/initiative-geogallery/initiative-geogallery.md, section Requirements (R14, R21, R22, R27), Constraints, Notes

## Notes

- Waits on epic-import-pipeline because: Bilder mit Metadaten, Thumbnails und AVIF in DB und Ablage.
