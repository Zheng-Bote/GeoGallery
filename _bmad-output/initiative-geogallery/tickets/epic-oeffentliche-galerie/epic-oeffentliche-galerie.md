---
type: epic
title: "Besucher ohne Konto sehen veröffentlichte Alben"
parent: initiative-geogallery
covers: ["R4", "R5", "R7", "R8", "R30"]
sources: []
companions: []
after: []
refined: false
risk: high
---

# Besucher ohne Konto sehen veröffentlichte Alben

## Description

Dieses Epic ergänzt die Stufe public im Sichtbarkeitsfilter: mit album.publish und Einwilligungsbestätigung wird ein Album public; anonyme Besucher sehen nur dessen nicht ausgeschlossene Bilder als Ableitungen, mit Positivliste (Schnittmenge) und GPS-Regel; ein Bild ist in einem Schritt aus allen public-Alben zurücknehmbar; Impressum, Datenschutz, Kontakt sind verlinkt.

## Outcome

Besucher ohne Konto sehen bewusst veröffentlichte Alben, und kein Gruppen-, privates oder ausgeschlossenes Bild und kein nicht freigegebenes Metadatum wird öffentlich.

## Done when

- Mit album.publish und bestätigter Einwilligung wird ein Album public; ein anonymer Besucher sieht dessen nicht ausgeschlossene Bilder, nur als Ableitungen.
- Öffentlich erscheinen nur Felder der Positivliste (Schnittmenge über alle public-Alben), GPS nach der restriktivsten Album-Einstellung.
- Ein Test mit Gruppen-, privaten und ausgeschlossenen Bildern zeigt keines davon öffentlich, weder in Seite noch API.
- Ein Bild ist in einem Schritt aus allen public-Alben zurückgenommen; Impressum, Datenschutz, Kontakt sind verlinkt; das Epic läuft deployt.

## Boundaries

- Keine Freigabe-Links, keine Einzelbilder ohne Album (forge §Verworfen).

## References

- parent — _bmad-output/initiative-geogallery/initiative-geogallery.md, section Requirements (R4, R5, R7, R8, R30), Constraints, Notes

## Notes

- Waits on epic-alben-sichtbarkeit because: Alben und der zentrale Sichtbarkeitsfilter, den dieses Epic um public erweitert.
