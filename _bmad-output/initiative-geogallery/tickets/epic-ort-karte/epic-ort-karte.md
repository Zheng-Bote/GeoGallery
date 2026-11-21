---
type: epic
title: "Bilder sind nach Ort und Ordner navigierbar und auf der Karte"
parent: initiative-geogallery
covers: ["R17", "R18", "R19", "R20", "R22", "R23", "R32"]
sources: []
companions: []
after: []
refined: false
risk: high
---

# Bilder sind nach Ort und Ordner navigierbar und auf der Karte

## Description

GPS wird über die Fallback-Kette der Online-Anbieter reverse-geocodiert (asynchron, für den Bestand über Tage), Bilder ohne GPS bekommen den Ort aus dem Ordner oder landen unter 'Ort fehlt', Abweichungen auf der Prüfliste mit Admin-Ansicht; Navigation nach Ort und Ordnerbaum, Karte mit Kartenkacheln und serverseitigem Clustering < 300 ms, Suche per Radius, Polygon, Bounding Box sowie Filter Land/Region/Stadt bei 1 Mio. Bildern aus dem Testbestand von epic-raster-suche.

## Outcome

Die Gruppe findet ihre Bilder über Ort, Ordner und Karte, auch Bilder ohne GPS.

## Done when

- Ein Bild mit GPS hat Kontinent/Land/Provinz/Region/Stadt aus der Anbieter-Fallback-Kette; der Bestand wird asynchron geocodiert.
- Ein Bild ohne GPS hat den Ort aus dem Ordner („ungefähr“) oder steht unter „Ort fehlt“; Abweichungen Ordner ≠ GPS stehen auf der Prüfliste in der Admin-Oberfläche.
- Nutzer navigieren nach Ort und Ordnerbaum und filtern nach Land, Region, Stadt.
- Die Karte clustert serverseitig in < 300 ms bei 1 Mio. Bildern und sucht per Radius, Polygon, Bounding Box; das Epic läuft deployt.

## Boundaries

- Öffentliches GPS (exakt/vergröbert/aus) liefert epic-oeffentliche-galerie.
- Touch point: Online-Geocoding-Anbieter — abgefragt hier.
- Touch point: Kartenkachel-Quelle — eingebunden hier.

## References

- parent — _bmad-output/initiative-geogallery/initiative-geogallery.md, section Requirements (R17, R18, R19, R20, R22, R23, R32), Constraints, Notes

## Notes

- Waits on epic-import-pipeline because: GPS und Ordnerpfad je Bild in der DB.
- Waits on epic-raster-suche because: Raster, Filtermechanik und 1-Mio.-Testbestand, die Ort-Navigation und Geo-Filter erweitern.
- Waits on epic-datensicherheit-betrieb because: Admin-Oberfläche, in die die Prüfliste-Ansicht kommt.
