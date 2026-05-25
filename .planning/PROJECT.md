# Event Sea (Renginių/Įvykių jūra)

## What This Is

Event Sea yra Lietuvos renginių agregatorius — kaip Skyscanner, bet spektakliams, koncertams ir muzikiniams pasirodymams. Platforma automatiškai surenka visus renginius iš visų šaltinių (bilietai.lt, tiketa.lt, Facebook Events, ir kt.) ir pateikia juos vienoje vietoje su tiesioginėmis bilietų pirkimo nuorodomis. Skirta miestiečiams 25–60 m., kurie nori greitai ir lengvai rasti ką veikti savo mieste.

## Core Value

Atidaro — ir iškart mato visą miestą: visi šios savaitės renginiai, kainos, bilietų nuorodos — be registracijos, be filtravimo, tiesiog viskas vienoje vietoje.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] Automatinis renginių rinkimas iš LT šaltinių (scraping + API)
- [ ] Renginių katalogas su paieška pagal miestą / datą / kategoriją
- [ ] Tiesioginės bilietų pirkimo nuorodos (redirect į originalų pardavėją)
- [ ] Personalizuotas renginių srautas (pagal vartotojo pomėgius ir miestą)
- [ ] Renginių kainų stebėjimas nuo anonso iki pardavimo
- [ ] Early Bird / kainų pokyčio alert'ai (email + Telegram)
- [ ] Kalendoriaus integracija (Google Calendar, Apple, Outlook)
- [ ] Ateinančių/planuojamų renginių skiltis (anonsai prieš bilietų pardavimą)
- [ ] B2B: renginio organizatoriaus skydelis (featured listings, analytics)
- [ ] Vartotojų autentifikacija (registracija, prisijungimas)
- [ ] Wishlist / seklojami renginiai

### Out of Scope

- Bilietų pardavimas tiesiogiai — Event Sea yra agregatorius, ne pardavėjas
- Renginių kūrimas eiliniams vartotojams — tik organizatoriams per B2B skydelį
- Tarptautiniai renginiai v1 — fokusas LT, plėtra vėliau

## Context

**Rinka:** LT renginių rinka fragmentuota — bilietai.lt, tiketa.lt, Eventbrite, Facebook Events, organizatorių svetainės. Nėra vieno šaltinio, kur matytum visus renginius ir galėtum palyginti kainas. Tai sprendžiame.

**Konkurentai:** bilietai.lt ir tiketa.lt yra pardavimo platformos, ne agregatoriai. Facebook Events yra socialiniai, ne komerciniai. Event Sea užima nišą tarp jų — kaip Google Flights tarp avialinijų.

**Verslo modelis:** B2B pirmas — organizatoriai moka už featured listings ir analytics skydelį. Vartotojams platforma nemokama. Tai leidžia greitai augti vartotojų bazę, o monetizacija ateina iš organizatorių, kurie nori matomumo.

**Dizaino kryptis:** 2026 m. tendencijos — dark mode pagal nutylėjimą, fluid layouts, micro-interactions. Inspiracija: Skyscanner + Booking.com UI/UX mišinys. Mobile-first.

## Constraints

- **Tech Stack**: Mobile-first (PWA ar React Native) — dauguma vartotojų naudos telefoną
- **Geografija**: Pradžia — Lietuva (Vilnius, Kaunas, Klaipėda). Plėtra — Baltija vėliau
- **Scraping**: Legalumas ir etika — naudoti viešus duomenis, robots.txt gerbti, API pirmenybė kai įmanoma
- **Realaus laiko**: Renginiai turi būti atnaujinami bent kartą per dieną (idealiai — realiu laiku)

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Agregatorius, ne pardavėjas | Greitesnis MVP, mažiau reguliacinių barjerų, galima integruoti daug šaltinių | — Pending |
| B2B monetizacija pirmas | Vartotojų bazė auga greičiau kai viskas nemokama; organizatoriai turi biudžetą | — Pending |
| Mobile-first | 70%+ LT vartotojų naršo telefonu; PWA leidžia calendar sync ir push notifications | — Pending |
| Scraping + žmogaus priežiūra | Automatika greita, bet žmogus užtikrina kokybę ir teisinius aspektus | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd:complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-05-25 after initialization*
