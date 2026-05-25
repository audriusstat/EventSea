# Event Sea — v1 Requirements

**Project:** Event Sea (Renginių/Įvykių jūra)
**Version:** v1 (MVP)
**Generated:** 2026-05-25
**Status:** Active — not yet validated

---

## v1 Requirements

### DATA — Duomenų rinkimas ir katalogas

- [ ] **DATA-01**: Sistema automatiškai renka renginius iš bilietai.lt kas 1-2 val.
- [ ] **DATA-02**: Sistema automatiškai renka renginius iš tiketa.lt kas 1-2 val.
- [ ] **DATA-03**: Scraper'iai veikia nepriklausomai — vieno gedimas neblokuoja kitų
- [ ] **DATA-04**: Kiekvienas renginys turi unikalų ID (generuojamas iš šaltinio URL hash)
- [ ] **DATA-05**: Kainų pokyčiai registruojami append-only price_history lentelėje
- [ ] **DATA-06**: Scraper'iai stebi robots.txt ir gerbia greičio ribas
- [ ] **DATA-07**: Kiekvienas scraper'is išsaugo `last_successfully_scraped_at` — gedimo atveju source pažymimas kaip degraded
- [ ] **DATA-08**: Pirminio anonso renginiai ("bilietai dar neparduodami") sekiami ir pažymimi pre-sale statusu

### CATALOG — Viešas renginių katalogas

- [ ] **CAT-01**: Vartotojas gali naršyti renginius be registracijos
- [ ] **CAT-02**: Vartotojas gali filtruoti renginius pagal miestą (Vilnius, Kaunas, Klaipėda)
- [ ] **CAT-03**: Vartotojas gali filtruoti renginius pagal datą (dzień / savaitė / mėnuo / data range)
- [ ] **CAT-04**: Vartotojas gali filtruoti renginius pagal kategoriją (koncertas, spektaklis, kino, sportas, kt.)
- [ ] **CAT-05**: Kiekvienas renginys turi atskirą puslapį su pilna informacija (pavadinimas, data, vieta, kaina, aprašymas)
- [ ] **CAT-06**: Renginio puslapyje yra tiesioginė nuoroda "Pirkti bilietus" su UTM sekimu
- [ ] **CAT-07**: Renginių kainos rodomos kaip "nuo €X" su šviežumo žymeniu ("prieš 2 val.")
- [ ] **CAT-08**: Išparduoti renginiai pažymimi "Išparduota" — nevaidinami kaip prieinami
- [ ] **CAT-09**: Vartotojas gali ieškoti renginių pagal raktažodžius su autocomplete (Meilisearch)
- [ ] **CAT-10**: Renginių puslapiai turi schema.org Event struktūrizuotus duomenis (Google Events rich snippets)
- [ ] **CAT-11**: Kiekvieno renginio peržiūros, išsaugojimo ir paspaudimo įvykiai instrumentuojami analitikai

### AUTH — Autentifikacija

- [ ] **AUTH-01**: Vartotojas gali prisijungti per Google OAuth
- [ ] **AUTH-02**: Vartotojas gali prisijungti per magic link (el. paštu gauna prisijungimo nuorodą — be slaptažodžio)
- [ ] **AUTH-03**: Vartotojas išlieka prisijungęs per sesiją (iki 30 dienų)
- [ ] **AUTH-04**: Vartotojas gali išsijungti iš bet kurio puslapio
- [ ] **AUTH-05**: Vartotojas gali ištrinti savo paskyrą ir visus duomenis (GDPR teisė būti pamirštam)

### WATCH — Sekimas ir wishlist

- [ ] **WATCH-01**: Prisijungęs vartotojas gali išsaugoti renginį į wishlist'ą
- [ ] **WATCH-02**: Prisijungęs vartotojas gali peržiūrėti savo išsaugotus renginius
- [ ] **WATCH-03**: Vartotojas gali sekti pre-sale renginį — gauna pranešimą kai bilietai pasirodo pardavime
- [ ] **WATCH-04**: Vartotojas gali sekti kainų pokyčius — gauna pranešimą kai kaina nukrenta (Early Bird pabaiga, kainų kritimas)

### PRICE — Kainų stebėjimas

- [ ] **PRICE-01**: Kainų istorija renkama nuo pirmojo renginio pastebėjimo (nuo DATA-05)
- [ ] **PRICE-02**: Renginio puslapyje rodoma kainų istorija kaip grafikas arba tekstinė tendencija
- [ ] **PRICE-03**: Vartotojas gali nustatyti kainų kritimo slenkstį (pvz., -10%) ir gauti alert'ą
- [ ] **PRICE-04**: Kainų alert'ai siunčiami ne dažniau kaip 1x/dieną vienam vartotojui vienam renginiui

### NOTIFY — Pranešimų sistema

- [ ] **NOTIFY-01**: Pranešimai siunčiami el. paštu (Resend) — kainų kritimai, pre-sale atidarymai, priminimai
- [ ] **NOTIFY-02**: Pranešimai siunčiami per Telegram Bot (grammY) — tas pats turinys
- [ ] **NOTIFY-03**: Vartotojas gali valdyti prenumeratą — atskirai el. paštas / Telegram per pranešimų tipą
- [ ] **NOTIFY-04**: Kiekvienas el. paštas turi atsisakymo nuorodą (CAN-SPAM / GDPR)
- [ ] **NOTIFY-05**: Pranešimų deduplication — tas pats alert'as nesiunčiamas 2x per 24 val.
- [ ] **NOTIFY-06**: Rate limiting — max 3 pranešimai/dieną vartotojui (viso visų tipų)

### CALENDAR — Kalendoriaus integracija

- [ ] **CAL-01**: Vartotojas gali atsisiųsti renginio .ics failą (vienas paspaudimas)
- [ ] **CAL-02**: Prisijungęs vartotojas gauna asmeninį .ics subscription URL (auto-sync wishlist'as į kalendorių)
- [ ] **CAL-03**: Vartotojas gali gauti el. pašto priminimą prieš renginį (24h ir 1h prieš)
- [ ] **CAL-04**: iCal įrašai naudoja Europe/Vilnius IANA timezone

### B2B — Organizatorių funkcijos

- [ ] **B2B-01**: Organizatorius gali susikurti atskirą organizatoriaus paskyrą
- [ ] **B2B-02**: Organizatorius gali "prisisieti" savo renginius iš katalogo
- [ ] **B2B-03**: Organizatorius gali pridėti/redaguoti savo renginių informaciją (nuotraukos, aprašymas)
- [ ] **B2B-04**: Organizatorius gali mokėti už featured listing (rodymasis viršuje) — Stripe + SEPA
- [ ] **B2B-05**: Organizatorius mato savo renginių peržiūrų ir paspaudimų statistiką

### GDPR — Privatumas ir atitikimas

- [ ] **GDPR-01**: Privatumo politika prieinama visose puslapiuose
- [ ] **GDPR-02**: Cookie sutikimo banner'is prieš bet kokį sekimą
- [ ] **GDPR-03**: Duomenų tvarkymo registras (Article 30) sukurtas ir palaikomas
- [ ] **GDPR-04**: Kaskadinis duomenų ištrynimas — ištrynus paskyrą ištrinami visi su vartotoju susiję duomenys

---

## v2 Requirements (deferred)

- Personalizuotas AI srautas (ML rekomendacijos) — reikia pakankamos vartotojų bazės
- Google Calendar / Apple Calendar tiesioginė OAuth sinchronizacija — po iCal validation
- Kainų palyginimas tarp kelių pardavėjų (tas pats renginys keliose platformose)
- tiketo.lt, Eventbrite, Facebook Events scraperiai — po pirmų dviejų šaltinių stabilizavimo
- B2B analytics skydelis (pilnas) — peržiūros grafikai, konversijų statistika
- Scarcity signals ("X bilietų liko šia kaina")
- Artist/atlikėjo sekimas (cross-source entity matching)
- Kainų palyginimo grafikas (duomenys renkami nuo v1)
- Push notifikacijos (PWA) — papildymas prie Telegram/email

---

## Out of Scope

- **Bilietų pardavimas tiesiogiai** — Event Sea yra agregatorius, ne pardavėjas; teisinė atsakomybė per didelė v1
- **End-user event creation** — tik organizatoriams per B2B; netvirto turinio rizika
- **Reviews / komentarai** — moderavimo našta; taip pat konkuruotų su organizatoriais
- **Vartotojų paywall** — B2B yra monetizacijos modelis; vartotojams viskas nemokama
- **In-app social network** — šalta pradžia + moderavimas; focus = discovery, ne socialinis
- **Native iOS/Apple apps** — PWA pakanka v1; app store review laikas per ilgas
- **Secondary market / perparduodami bilietai** — teisinė rizika
- **Tarptautiniai renginiai** — fokusas LT rinkoje; plėtra po v1 validacijos

---

## Traceability (to be filled by roadmap)

| REQ-ID | Phase | Plan |
|--------|-------|------|
| DATA-01..08 | — | — |
| CAT-01..11 | — | — |
| AUTH-01..05 | — | — |
| WATCH-01..04 | — | — |
| PRICE-01..04 | — | — |
| NOTIFY-01..06 | — | — |
| CAL-01..04 | — | — |
| B2B-01..05 | — | — |
| GDPR-01..04 | — | — |

---

**Total v1 requirements:** 50
**Categories:** DATA (8), CAT (11), AUTH (5), WATCH (4), PRICE (4), NOTIFY (6), CAL (4), B2B (5), GDPR (4)
