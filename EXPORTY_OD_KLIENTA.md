# 📥 Exporty od klienta — Marketing Dashboard

**Datum:** 12. 5. 2026
**Pro koho:** Ondřej (klient = data preparer)
**Cíl:** Přesná specifikace všech CSV/dat které potřebuju, abys mi je mohl pravidelně dodávat.

> **Pravidlo**: Každý CSV upload do `analytics-audit/data/raw/` → automaticky vyvolá build skripty + push na GitHub Pages.

---

## 🚨 P0 — KRITICKÉ (bez nich neexistuje sekce)

### 1. Per-pobočka tržby z Helios IQ (Section 9 — Pobočky)

**Co**: Měsíční tržby + počet účtenek + průměrná hodnota účtenky **PER PROVOZOVNA** (17 prodejen)

**Jak to získat**:
1. Helios IQ → **Maloobchod / Pokladna**
2. Filter: období = poslední uzavřený měsíc
3. **Group by**: provozovna (středisko)
4. Export do CSV (Excel → Save As → CSV UTF-8)

**Formát souboru**:
- Cesta: `analytics-audit/data/raw/erp_pobocky_YYYY_MM.csv`
- Encoding: **UTF-8** (jinak rozbiju diakritiku)
- Delimiter: `;` (středník — Helios default)
- Decimal: `,` (česká lokalizace OK)

**Sloupce (přesný order)**:
```
Datum;Provozovna;PocetUctenek;TrzbyCelkem;TrzbyHotovost;TrzbyKarta;TrzbyDobirka;PrumernaUctenka;Marze
2026-04-01;Praha 4 Modřany;145;87650.00;42100.00;39200.00;6350.00;604.48;26295.00
2026-04-01;Plzeň centrum;98;52340.00;28900.00;20100.00;3340.00;534.08;15702.00
...
```

**Pokud nemůžeš exportovat marži**: ponechej sloupec `Marze` prázdný nebo `0` — dopočítám z products data.

**Frekvence**: Měsíčně (ideálně 1.-3. den měsíce za uzavřený předchozí)

**Co s tím udělám**:
- Section 9 Pobočky → tržby per pobočka, MoM trend, top/bottom 3
- Cross-link na GBP impressions per pobočka (po setupu)
- Heat mapa pobočky × kraj

---

## 🟡 P1 — DŮLEŽITÉ

### 2. Alza Partner orders (Section 5 — Srovnávače)

**Co**: Objednávky přes Alza marketplace per den + revenue + commission

**Jak to získat**:
1. https://partner.alza.cz → **Objednávky**
2. Filter: posledních 7 dní (stačí týdenní upload)
3. Export → CSV

**Formát souboru**:
- Cesta: `analytics-audit/data/raw/alza_orders_YYYY_WW.csv` (WW = ISO week)
- Encoding: **UTF-8**
- Delimiter: `;` nebo `,` (oba podporuju)

**Sloupce (minimum)**:
```
DatumObjednavky;CisloObjednavky;ProduktSKU;ProduktNazev;Pocet;CenaProdejni;Provize;StavObjednavky
2026-04-15;A-2026-04-15-001;HSE-1234;Cylindrická vložka 30+30;1;890.00;142.40;Doručeno
```

**Frekvence**: Týdně (každé pondělí za předchozí týden) — nebo daily pokud dáš

**Co s tím udělám**:
- Section 5 Srovnávače → Alza tab
- Marketplace ROAS (revenue - commission) / total
- Top 20 SKU na Alza

---

### 3. Heuréka OCM export (Section 5 — Srovnávače)

**Co**: Order Conversion Measurement — kolik objednávek přišlo z Heuréky

**Jak to získat (2 varianty)**:

**Varianta A — API push (preferovaná)**:
1. Heuréka admin → **OCM** → vygeneruj API token
2. Pošli mi token → nastavím automatický pull každý den 6:00

**Varianta B — Manuální export**:
1. Heuréka admin → **Reporty → Konverze**
2. Filter: posledních 7 dní
3. Export CSV

**Formát**:
- Cesta: `analytics-audit/data/raw/heureka_ocm_YYYY_WW.csv`
- Sloupce: `Datum;OrderID;Hodnota;Source;Klikové;CTR;Pozice`

**Frekvence**: Týdně (nebo daily přes API)

---

### 4. Zboží.cz OCM export (Section 5 — Srovnávače)

Stejné jako Heuréka — Zboží.cz admin má podobné OCM rozhraní.

**Cesta**: `analytics-audit/data/raw/zbozi_ocm_YYYY_WW.csv`

---

### 5. Meta Ads (Facebook Business) — Section 1 sub-tab

**Co**: Facebook + Instagram kampaně (brand kampaně Plzeň/Brno/Chomutov dle auditu)

**Jak to získat (2 varianty)**:

**Varianta A — Facebook Marketing API (preferovaná)**:
1. https://business.facebook.com → Settings → System Users
2. Vytvoř System User „Dashboard"
3. Generate Access Token s permissions: `ads_read, business_management`
4. Pošli mi token → nastavím automatický pull

**Varianta B — Manuální export**:
1. Facebook Ads Manager → Reporting
2. Date range: posledních 30 dní
3. Columns: Campaign Name, Spend, Impressions, Clicks, CTR, CPC, Conversions, ROAS
4. Export CSV

**Cesta**: `analytics-audit/data/raw/meta_ads_YYYY_MM.csv`

**Frekvence**: Měsíčně (manuální) nebo daily (API)

---

### 6. Firmy.cz Partner API access (Section 16 — Firmy.cz)

**Co**: Listing performance pro 17 prodejen na Firmy.cz

**Jak to získat**:
1. Pošli email na **partner@seznam.cz**:
   ```
   Předmět: Žádost o API access pro analytický dashboard

   Dobrý den,
   chtěl bych požádat o API access pro náš firemní účet (klicovecentrum.cz / 17 prodejen).
   Účel: Interní analytický dashboard pro vyhodnocení viditelnosti našich prodejen.

   Nepotřebujeme write/edit funkce, pouze read-only pro:
   - Profile views statistics
   - Search position v results
   - Reviews count + rating
   - Click-to-call / direction events

   Pokud API není dostupné, prosím o souhlas s jemným scrapingem našich vlastních profile URL
   (1× denně, respekt k robots.txt).

   Děkuji,
   Ondřej Skřebský
   H&B Group s.r.o.
   ```

2. Po obdržení odpovědi mi pošli buď:
   - **API credentials** (pokud Seznam povolí)
   - **Souhlas s scrapingem vlastních profilů** (zbuduju Python scraper)

**Frekvence**: Daily (po setupu)

---

## 🔧 SETUP API (jen jednorázové akce, žádné CSV)

### 7. Google Business Profile — Manager access (Section 15 — GBP)

**Co**: Přidat agenturní email (= můj Service Account) jako Manager u všech 17 GBP profilů

**Jak**:
1. https://business.google.com → vyber pobočku
2. **Settings** → **Managers** → **Add manager**
3. Email: `hbg-101@hbgroup-493608.iam.gserviceaccount.com`
4. Role: **Manager** (ne Owner)
5. Opakuj pro všech 17 prodejen

**Tip pro úsporu času**: Pokud máš **Business Group** v GBP, přidej Manager na úrovni groupu = 1 klik místo 17.

**Po dokončení mi řekni** → udělám test API call + zapnu Section 15 buildy.

---

### 8. Google Merchant Center — User access (Section 17 — MC)

**Co**: Přidat můj Service Account jako Standard user v Merchant Center

**Jak**:
1. https://merchants.google.com → **Settings** → **Users**
2. **Add user**
3. Email: `hbg-101@hbgroup-493608.iam.gserviceaccount.com`
4. Access: **Standard** (ne Admin — Admin nepovoluje API access)
5. Save

**Po dokončení mi řekni** → udělám test API call + zapnu Section 17 buildy.

---

## 📦 EXISTUJÍCÍ exporty — pokračovat (P0 ongoing)

Tyto už dodáváš, ale potřebuju je **každý měsíc** pro live dashboard:

### 9. ERP Helios Výdejky MO (B2C) — měsíčně

**Cesta**: `analytics-audit/data/raw/erp_vydejky_mo_YYYY_MM.csv`
**Sloupce**: `Datum;CisloDokladu;ZakaznikID;ZakaznikNazev;ProduktSKU;ProduktNazev;Pocet;CenaJednotkova;CenaCelkem;DPH;CelkemSDPH;Provozovna`
**Frekvence**: 1. den měsíce za předchozí měsíc

### 10. ERP Helios Výdejky VO (B2B) — měsíčně

**Cesta**: `analytics-audit/data/raw/erp_vydejky_vo_YYYY_MM.csv`
**Stejné sloupce + ICO + DIC + Firma**

### 11. ERP Helios Reklamace — měsíčně

**Cesta**: `analytics-audit/data/raw/erp_reklamace_YYYY_MM.csv`
**Sloupce**: `Datum;CisloReklamace;OriginalDoklad;ProduktSKU;Duvod;Stav;CenaReklamace;DenuVyrizeni`

### 12. PPL dopravce export — měsíčně

**Cesta**: `analytics-audit/data/raw/ppl_zasilky_YYYY_MM.csv`
**Source**: Helios IQ → modul **Doprava → PPL**
**Encoding**: **cp1250** (Helios default — neměň, parser umí oboje)
**Delimiter**: `;`
**Datum format**: MM/DD/YYYY (Helios default)

---

## 🟢 P2/P3 — VOLITELNÉ (nice-to-have, nebrání launchi)

### 13. Mailchimp/Ecomail — Email kampaně (Section 1 sub-tab)

Pouze pokud máš email marketing aktivní.

**Cesta**: `analytics-audit/data/raw/email_campaigns_YYYY_MM.csv`
**Sloupce**: `Datum;CampaignNazev;Recipients;OpenRate;CTR;Conversions;Revenue`

### 14. Heureka Ověřeno NPS (Section 12 sub-tab)

Pokud máte „Ověřeno zákazníky" widget.

**Cesta**: `analytics-audit/data/raw/heureka_nps_YYYY_MM.csv`
**Sloupce**: `Datum;OrderID;Rating;Komentar;ResponseStatus`

### 15. Glami.cz (pokud používáme módní cross-sell)

Pravděpodobně nepoužíváme (jsme zámky/kování), ale pokud ano:
**Cesta**: `analytics-audit/data/raw/glami_orders_YYYY_MM.csv`

---

## 📋 CHECKLIST pro klienta — co udělat hned

**Hned (15-30 min):**
- [ ] **GBP**: Přidat `hbg-101@hbgroup-493608.iam.gserviceaccount.com` jako Manager u všech 17 profilů (nebo group level)
- [ ] **MC**: Přidat stejný email jako Standard user
- [ ] **Firmy.cz**: Email na `partner@seznam.cz`

**Tento týden:**
- [ ] **Helios**: Zjistit jak udělat per-provozovna export z modulu Maloobchod/Pokladna (kontaktovat IT/účetní)
- [ ] **Alza**: Otestovat manuální export z Partner portálu — pošli mi sample CSV ať vidím sloupce
- [ ] **Meta Ads**: Vyhodnotit jestli máme aktivní kampaně (jinak skip)
- [ ] **Heuréka OCM**: Login do Heuréka admin → zjisti jestli máš API access (jinak manuální)

**Po setupu API:**
- [ ] **GBP**: Po přidání managerů mi napiš → otestuju API + spustím buildy
- [ ] **MC**: Po user access → otestuju + spustím buildy

**Měsíčně (existující workflow, jen pokračovat):**
- [ ] ERP Výdejky MO + VO
- [ ] ERP Reklamace
- [ ] PPL dopravce

---

## 🚀 Co můžu rozjet TEĎ (bez čekání na klienta)

12 z 18 sekcí má všechna data:
- Sekce 0, 1, 2, 3, 4, 7, 8, 10, 11, 12, 13, 14

→ Začínám stavět scaffold + buildy pro tyto sekce.

Sekce čekající na klienta:
- Sekce 5 (Srovnávače) — Alza/Heuréka/Zboží
- Sekce 9 (Pobočky) — Helios per-provozovna
- Sekce 15 (GBP), 17 (MC) — API setup
- Sekce 16 (Firmy.cz) — Partner API
- Sekce 6 (Feedy) — Merchant Center deep-dive

Tyto sekce budou v dashboardu **viditelné s placeholder „Čekáme na data od klienta — viz EXPORTY_OD_KLIENTA.md"**.

---

## 💬 Kontakt

Pokud máš dotaz k formátu nebo se něco nedá exportovat ve formátu výše — napiš mi a najdeme alternativu (jiné sloupce, jiný delimiter, JSON místo CSV, atd.).

**Důležité**: Lepší **nepřesný export rychle** než **perfektní export pozdě**. Datasety doplníme iterativně.
