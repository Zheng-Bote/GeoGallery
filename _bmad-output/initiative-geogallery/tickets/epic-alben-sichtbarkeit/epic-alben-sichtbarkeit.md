---
type: epic
title: "Alben und Sichtbarkeit bestimmen, wer was sieht"
parent: initiative-geogallery
covers: ["R4", "R28", "R29"]
sources: []
companions: []
after: []
refined: false
risk: medium
---

# Alben und Sichtbarkeit bestimmen, wer was sieht

## Description

Nutzer legen Alben bewusst an und befüllen sie aus dem Raster; Sichtbarkeit pro Album wird vererbt und restriktivste gewinnt, indem dieses Epic den Sichtbarkeitsfilter von epic-import-pipeline um Alben erweitert; Owner und Admin setzen Sichtbarkeit nach R29 in der Oberfläche; die Leistungsziele halten weiter bei 1 Mio. Bildern.

## Outcome

Jedes Mitglied steuert über Alben, wer seine Bilder sieht, und niemand sieht mehr, als die restriktivste Einstellung erlaubt.

## Done when

- Ein Nutzer legt ein Album an und befüllt es aus dem Raster; ein Bild kann in mehreren Alben liegen.
- Ein Bild erbt die Sichtbarkeit seiner Alben, die restriktivste gewinnt, in Raster, Detail, Suche, Karte und API.
- Der Owner setzt jede Sichtbarkeit; ein Admin ändert Gruppen fremder Inhalte, sieht und setzt aber nie privat.
- Die Leistungsziele aus epic-raster-suche und epic-ort-karte halten mit Albenfilter bei 1 Mio.; das Epic läuft deployt.

## Boundaries

- Die Stufe public und alles Öffentliche liefert epic-oeffentliche-galerie.

## References

- parent — _bmad-output/initiative-geogallery/initiative-geogallery.md, section Requirements (R4, R28, R29), Constraints, Notes

## Notes

- Waits on epic-import-pipeline because: zentraler Sichtbarkeitsfilter, den dieses Epic um Alben erweitert.
- Waits on epic-raster-suche because: Raster und Auswahl, aus denen Alben befüllt werden, und 1-Mio.-Testbestand.
- Waits on epic-ort-karte because: Abfragepfade von Karte und Ort-Navigation, die mit dem erweiterten Filter ihre Ziele halten.
