---
id: 2
type: story
title: "Erster Admin richtet seinen Passkey ein und meldet sich an"
parent: epic-plattform-zugang
covers: [R1, R3, R4]
after: [1]
hitl: true
risk: high
---

# Erster Admin richtet seinen Passkey ein und meldet sich an

## Description

Bootstrap-Kommando in iam legt den ersten Admin an und gibt auf der Konsole einen Einmal-Einrichtungslink (24 h) aus; hat der Admin noch keinen Faktor, gibt ein erneuter Aufruf einen neuen Link aus und macht den alten ungültig (Epic R1). Über den Link registriert der Admin einen Passkey, geprüft vom neuen Rust-Service webauthn (webauthn-rs, zustandslos, Attestation „none“; Credentials und Challenges besitzt iam und übergibt sie je Aufruf), und meldet sich an. iam führt die Session serverseitig, die api setzt das Cookie Secure/HttpOnly/SameSite=Strict und löst es je Request über iam zu einem Principal auf; `GET /api/v1/me` liefert ihn. Abmelden beendet die Session.

Eine Person registriert den Passkey mit einem echten Authenticator.

## Requirements

Verify: Ein frischer Bootstrap gibt einen Link aus; damit registriert eine Person einen Passkey und landet angemeldet in der Shell. Nach dem Abmelden meldet sie sich mit demselben Passkey erneut an. Abgewiesen werden: ein zweiter Bootstrap, sobald der Admin einen Passkey hat; ein zweites Öffnen des Links; ein Link nach 24 h; eine wiederverwendete Challenge. Ein Bootstrap vor der Registrierung gibt einen neuen Link aus, und der alte wird abgewiesen. Das Cookie trägt Secure, HttpOnly und SameSite=Strict. Ohne Cookie liefert `GET /api/v1/me` 401 im Envelope. Der Link-Token steht weder in der Datenbank im Klartext noch im Log.

## References

- parent — _bmad-output/initiative-geogallery/tickets/epic-plattform-zugang/epic-plattform-zugang.md, Requirements R1, R3, R4
- architecture — _bmad-output/initiative-geogallery/architecture-geogallery/architecture-geogallery.md, AD-8, AD-9, AD-23, AD-24, AD-29, AD-44, AD-47

## Notes

- Eine Person registriert den Passkey mit einem echten Authenticator.
- Person legt das Repo `services/webauthn` an.
- Der Link-Token wird nur gehasht gespeichert (AD-44).
- Die api ruft webauthn nur für Health auf; der Socket von webauthn ist nur für iam zugänglich (AD-9).
- Nicht hier: der Principal trägt vorerst nur die `user_id`, Gruppen und Berechtigungen kommen mit Story 6; CSRF-Token und Rate Limit Story 10; Audit von Bootstrap und An-/Abmeldung Story 5; TOTP Story 8; Links für weitere Konten Story 7.
