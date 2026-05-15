# 📥 Exporty od klienta — Marketing Dashboard

**Datum:** 12. 5. 2026 (revize: GBP+MC HOTOVO, Firmy.cz import feed, Meta brandové, Heuréka)
**Pro koho:** Ondřej (klient = data preparer)
**Cíl:** Přesná specifikace všech CSV/dat které potřebuju, abys mi je mohl pravidelně dodávat.

> **Pravidlo**: Každý CSV upload do `analytics-audit/data/raw/` → automaticky vyvolá build skripty + push na GitHub Pages.

---

## 🎉 ČERSTVÉ NEWS (12.5.2026)

### ✅ HOTOVO
- **Google Business Profile** — managers přidaní → **mohu rozjet API**
- **Google Merchant Center** — user access dán → **mohu rozjet API** (LIVE — 2228 produktů, 99.24% approval)
- **Cloudflare na všech 4 doménách** — klicovecentrum.cz/sk + hbgroup.cz/sk za CF Free planem

### 🔄 NOVÉ INFO
- **4 eshopy = 4 trhy**: CZ B2C (klicovecentrum.cz), SK B2C (klicovecentrum.sk?), CZ B2B (hbgroup.cz), SK B2B (hbgroup.sk)
- **ERP je Cézar** (ne Helios) — přepsáno všude v dokumentech
- **Firmy.cz** — má jen IMPORTNÍ feed (JSON schema v1.7), statistiky pouze manuálně CSV
- **Meta** má aktivní brandové kampaně — **mohu napojit Marketing API**
- **Heuréka** — uvedl jsi `github.com/heureka/hcapi/`, ale to je PHP knihovna pro Order/Payment callbacks (NE statistiky)

---

## ❓ OTÁZKY PRO KLIENTA — ASAP odpovědi

Pro sprint dashboardu potřebuju vyjasnit pár věcí o multi-domain setupu:

### Q1. „kucovecentrum.sk" v zadání = překlep? (rychle ANO/NE)
Předpokládám `klicovecentrum.sk` (s „L" navíc — stejná značka, jiný trh). Pokud je to jiná doména, napiš mi přesné jméno.

### Q2. SQL `kosik.web_id` mapping na 4 domény
Můžeš mi spustit query:
```sql
SELECT web_id, COUNT(*) AS orders, MIN(datum) AS first_order, MAX(datum) AS last_order
FROM kosik
GROUP BY web_id
ORDER BY web_id;
```
+ pokud máš tabulku `weby` nebo podobnou s názvem domény, pošli SELECT všechno z ní.

### Q3. GA4 properties pro 4 trhy
Login do https://analytics.google.com → vlevo nahoře přepínač property → kolik máš properties?

Pokud existují separate GA4 pro každou doménu, pošli mi:
- klicovecentrum.cz: property ID 299992437 (mám)
- klicovecentrum.sk: ❓
- hbgroup.cz: ❓
- hbgroup.sk: ❓

Pokud máš všechno v jedné GA4 property → potřebuju zjistit jak je rozeznat (custom dimension `domain`? URL parametr? hostname filter?).

### Q4. Google Ads struktura (account 350-878-7813)
- 1 account pro všechny 4 trhy?
- 4 sub-accounts pod manager account?
- Pošli mi screenshot z Google Ads (vlevo nahoře vidíš strukturu)

### Q5. Sklik účty
- Sklik podporuje **jen CZ trh** (Seznam je český)
- Pro CZ B2C (klicovecentrum.cz) máme Sklik účet ✅
- Pro CZ B2B (hbgroup.cz) — máš Sklik účet? Token mi pošli pokud ano.

### Q6. Merchant Center accounts
- Aktuálně mám access k Merchant ID 130672692 (asi klicovecentrum.cz)
- Pro 3 zbývající domény jsou separate MC accounty?
- Login do https://merchants.google.com vlevo nahoře přepínač → kolik MC?

---

---

## 🔧 K rozjetí TEĎ (mám API access nebo HCAPI / brzy doděláme)

### A. Google Business Profile (Section 15) — JEDU
- Service Account `hbg-101@hbgroup-493608.iam.gserviceaccount.com` přidán jako Manager u 17 GBP profilů ✅
- Příští krok: enable My Business APIs v Cloud Console + test API call
- Build skript: `build_gbp_performance.py` (impressions, calls, directions, reviews per pobočka)

### B. Google Merchant Center (Section 17 + Section 6 deep) — JEDU
- Service Account jako Standard user v MC ✅
- Příští krok: enable Content API + Reports API v Cloud Console + test API call
- Build skript: `build_mc_issues.py` + `build_mc_performance.py`

### C. Meta Marketing API (Section 1 sub-tab Meta) — POTŘEBUJU TOKEN
- **Co**: brandové kampaně Plzeň/Brno/Chomutov/atd.
- **Co potřebuju od tebe**:
  1. https://business.facebook.com → **Settings** → **System Users**
  2. **Add → System User** → name: `dashboard-readonly`, role: Employee
  3. **Add Assets** → vyber Ad Account(s) → role: View performance
  4. **Generate New Token** → app: vyber svojí (nebo si vyrob app na developers.facebook.com), permissions: `ads_read`, `business_management`
  5. **Token Type**: **Never expires** (System User token nemá expiraci)
  6. Pošli mi token (např. v ChatGPT/Claude chat — token NIKAM nedávat do GitHubu)
- Po získání: vytvořím `build_meta_ads.py` + uložení tokenu mimo repo

### D. Heuréka (Section 5) — VYJASNIT
HCAPI z `github.com/heureka/hcapi/` je PHP knihovna pro **callback processing** (Order/Cancel, Payment/Status). To NENÍ to co potřebujeme — to je pro stranu **shopu** který přijímá objednávky z Heuréky.

**Pro statistiky** (kolik kliků, conv rate, pozice produktu, výnos z Heuréky) jsou jiné cesty:

**Možnost 1 — Heuréka Reporting API** (nejlepší):
- https://sluzby.heureka.cz → admin login → API klíč
- Endpoint: stats per den (klikové, conv, výnos)
- Pošli mi: API klíč (z Heuréka admin)

**Možnost 2 — Heuréka manuální CSV export**:
- Heuréka admin → **Reporty → Konverze**
- Export CSV → upload do `data/raw/heureka_stats_YYYY_MM.csv`
- Sloupce viz níže (sekce 3)

**Možnost 3 — Heuréka OCM iframe** (jen tracking nových konverzí):
- Pokud máte iframe pixel na thank-you page → Heuréka loguje conversions, my data nečteme

→ **Pošli mi**: jestli máš API klíč v Heuréka admin, nebo budeme dělat manuální CSV.

---

---

## 🚨 P0 — KRITICKÉ (bez nich neexistuje sekce)

### 1. Per-pobočka tržby z Cézar (Section 9 — Pobočky)

**Co**: Měsíční tržby + počet účtenek + průměrná hodnota účtenky **PER PROVOZOVNA** (17 prodejen)

**Jak to získat**:
1. Cézar → **Maloobchod / Pokladna**
2. Filter: období = poslední uzavřený měsíc
3. **Group by**: provozovna (středisko)
4. Export do CSV (Excel → Save As → CSV UTF-8)

**Formát souboru**:
- Cesta: `analytics-audit/data/raw/erp_pobocky_YYYY_MM.csv`
- Encoding: **UTF-8** (jinak rozbiju diakritiku)
- Delimiter: `;` (středník — Cézar default)
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

### 5. Meta Ads (Facebook Business) — Section 1 sub-tab — **AKTIVNÍ KAMPANĚ ✅**

**Status**: Klient potvrdil že má **aktivní brandové kampaně** → priorita P0, jdu napojit přes API.

**Detaily viz „K rozjetí TEĎ → C. Meta Marketing API"** nahoře v dokumentu.

**TL;DR co potřebuju**:
1. Z `business.facebook.com` → Settings → System Users → vytvoř `dashboard-readonly`
2. Add Asset → tvoje Ad Account → role View performance
3. Generate Token (ads_read + business_management, **Never expires**)
4. Pošli mi Ad Account ID + token

Po obdržení tokenu:
- `build_meta_ads.py` — campaigns + ad sets + ads daily KPIs
- `data/marketing/meta_ads.json`
- Section 1 sub-tab „Meta" se rozsvítí (LIVE badge)

**Backup pokud token nefunguje** — manuální CSV:
- Facebook Ads Manager → Reporting → posledních 30 dní → Columns: Campaign, Spend, Impressions, Clicks, CTR, CPC, Conversions, ROAS → CSV
- Cesta: `analytics-audit/data/raw/meta_ads_YYYY_MM.csv`

---

### 6. Firmy.cz — 2 NEZÁVISLÉ věci (Section 16)

**🔴 Pozor**: Firmy.cz nemá statistiky API. Má pouze **importní feed** (zápis dat o naších prodejnách na Seznam) a **manuální CSV export** pro statistiky.

#### 6a. Firmy.cz IMPORT FEED (my generujeme + posíláme Seznamu)

**Co**: Strukturovaný JSON feed (schema v1.7, klient mi poslal `firmy-cz-v1.7.json`) s informacemi o všech 17 prodejnách — Seznam ho čte 1× denně a aktualizuje naše profily na Firmy.cz.

**Co dělá**: Garantuje že **adresa, telefony, otvírací doba, fotky, hlavní akční tlačítko** jsou na Firmy.cz **vždy synchronizovány s naším ERP** (žádný drift, žádné staré údaje).

**Schema v1.7** požaduje per pobočka:
- `id` (unikátní neměnný — můžeme použít Cézar provozovna ID nebo náš vlastní `kc-prodejna-01` až `kc-prodejna-17`)
- `ic` (IČO — `26168685`?)
- `name` (např. „Klíčové centrum Plzeň")
- `description` (max 300 znaků — popis pobočky pro Firmy.cz)
- `address` (city, street, houseNumber, zip, gps lat/lng)
- `emails` (max 5 — email pobočky)
- `phones` (max 5 — telefon pobočky, fax, recepce)
- `socialNetworks` (max 6 — FB, IG, LinkedIn URL)
- `url` (homepage)
- `openingHours` (OSM format — např. `Mo-Fr 08:00-18:00; Sa 09:00-12:00; PH off`)
- `mainActionButton` (URL + custom text — např. „Otevřít e-shop")
- `filters` (štítky — `s-parkovistem`, `bezbarierove`, `platba-kartou`, `na-splatky`, atd.)
- `graphics.logo` (URL na logo)
- `graphics.mainPhoto` (hlavní foto pobočky)
- `graphics.photos` (až 20 fotek galerie)

**Co potřebuju od tebe (jednorázově)**:

1. **Master tabulka 17 prodejen** ve formátu (CSV nebo JSON):
   ```csv
   id;name;ic;city;street;houseNumber;zip;lat;lng;email;phone;url;hours;description
   kc-01;Klíčové centrum Plzeň;26168685;Plzeň;Klatovská;105;30100;49.7361;13.3744;plzen@klicovecentrum.cz;+420377123456;https://klicovecentrum.cz;Mo-Fr 08:00-18:00; Sa 09:00-12:00;Pobočka v Plzni - klíče, vložky, trezory, kování;
   kc-02;...
   ```
   Cesta: `analytics-audit/data/raw/pobočky_master.csv`

2. **URL na logo + hlavní foto** každé pobočky (může být shared link na Google Drive, Dropbox, vlastní hosting)

3. **Štítky per pobočka** (jaké filtry chceme zobrazit):
   - Co máme: parkování, bezbariérový vstup, platba kartou, splátky, klimatizace, …

**Co s tím udělám**:
- Vytvořím `build_firmy_import_feed.py` který:
  1. Načte master tabulku
  2. Validuje proti schema v1.7
  3. Vygeneruje `firmy_feed.json` na URL `https://kc-hbgroupcz.github.io/klicovecentrum-audit/feeds/firmy.json`
  4. Refreshuje denně (pokud změníš data v ERP)
- **Pošleš Seznamu** URL feedu → jednorázové nastavení v Firmy.cz admin

**Výhody automatizace**:
- ✅ Nikdy nemusíš ručně upravovat 17 profilů
- ✅ Změna otvírací doby → 1 update v ERP/master tabulce → druhý den všechno aktualizované
- ✅ Konsistence napříč všemi 17 prodejnami

#### 6b. Firmy.cz MANUÁLNÍ STATS EXPORT (klient ručně, do CSV)

**Co**: Statistiky views/calls/clicks z Firmy.cz admin

**Jak**:
1. https://admin.firmy.cz → login
2. **Statistiky** → **Pobočky** → vyber rozsah (poslední měsíc)
3. **Export → CSV**
4. Upload do `analytics-audit/data/raw/firmy_stats_YYYY_MM.csv`

**Frekvence**: Měsíčně

**Co očekávám ve sloupcích** (přesné jména si zjistím až mi pošleš sample):
```
Pobocka;Datum;Zobrazeni;Kliky_Telefon;Kliky_Web;Kliky_Mapa;Pozice_Search;Recenze_Pocet;Recenze_Rating
```

**Pošli mi sample CSV** ať vidím přesné sloupce z Firmy.cz exportu.

---

## ✅ SETUP API — HOTOVO

### 7. Google Business Profile — Manager access ✅ HOTOVO

Klient přidal `hbg-101@hbgroup-493608.iam.gserviceaccount.com` jako Manager u všech 17 GBP profilů.
**Příští kroky (já)**: Enable My Business APIs v Cloud Console → otestovat API call → zapnout Section 15 buildy.

### 8. Google Merchant Center — User access ✅ HOTOVO

Klient přidal Service Account jako Standard user v MC.
**Příští kroky (já)**: Enable Content API + Reports API v Cloud Console → otestovat API call → zapnout Section 17 + Section 6 deep buildy.

---

## 📦 EXISTUJÍCÍ exporty — pokračovat (P0 ongoing)

Tyto už dodáváš, ale potřebuju je **každý měsíc** pro live dashboard:

### 9. ERP Cézar Výdejky MO (B2C) — měsíčně

**Cesta**: `analytics-audit/data/raw/erp_vydejky_mo_YYYY_MM.csv`
**Sloupce**: `Datum;CisloDokladu;ZakaznikID;ZakaznikNazev;ProduktSKU;ProduktNazev;Pocet;CenaJednotkova;CenaCelkem;DPH;CelkemSDPH;Provozovna`
**Frekvence**: 1. den měsíce za předchozí měsíc

### 10. ERP Cézar Výdejky VO (B2B) — měsíčně

**Cesta**: `analytics-audit/data/raw/erp_vydejky_vo_YYYY_MM.csv`
**Stejné sloupce + ICO + DIC + Firma**

### 11. ERP Cézar Reklamace — měsíčně

**Cesta**: `analytics-audit/data/raw/erp_reklamace_YYYY_MM.csv`
**Sloupce**: `Datum;CisloReklamace;OriginalDoklad;ProduktSKU;Duvod;Stav;CenaReklamace;DenuVyrizeni`

### 12. PPL dopravce export — měsíčně

**Cesta**: `analytics-audit/data/raw/ppl_zasilky_YYYY_MM.csv`
**Source**: Cézar → modul **Doprava → PPL**
**Encoding**: **cp1250** (Cézar default — neměň, parser umí oboje)
**Delimiter**: `;`
**Datum format**: MM/DD/YYYY (Cézar default)

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

## 📋 CHECKLIST pro klienta — aktualizováno 12.5.2026

### ✅ Hotovo
- [x] **GBP managers** přidaní ✅
- [x] **MC user access** přidaný ✅

### 🚨 Hned (do 24h, blokuje API setupy)

- [ ] **Meta Marketing API token**:
  1. business.facebook.com → Settings → System Users → vytvoř `dashboard-readonly` (Employee role)
  2. Add Asset → Ad Account → role View performance
  3. Generate Token → permissions `ads_read` + `business_management`, **Never expires**
  4. Pošli mi: token + Ad Account ID

- [ ] **Heuréka API klíč** (pokud existuje):
  1. https://sluzby.heureka.cz → admin → Settings → API
  2. Pokud existuje: pošli klíč
  3. Pokud neexistuje: napiš mi → uděláme manuální CSV workflow

### 🟡 Tento týden

- [ ] **Cézar per-provozovna export**: Kontaktovat IT/účetní → zjistit jak vyrobit měsíční CSV s tržbami per provozovna z modulu Maloobchod/Pokladna
- [ ] **Master tabulka 17 prodejen** pro Firmy.cz import feed:
  - Cesta: `data/raw/pobočky_master.csv`
  - Sloupce: `id;name;ic;city;street;houseNumber;zip;lat;lng;email;phone;url;hours;description;logo_url;photo_url;filters`
  - Můžeš začít s 1-2 prodejnami → vyladíme formát → doplníš zbytek
- [ ] **Firmy.cz stats CSV sample**: Login do `admin.firmy.cz` → Statistiky → Pobočky → Export CSV → pošli sample (ať vidím sloupce)
- [ ] **Alza Partner CSV sample**: Pošli sample export z Partner portálu

### 🟢 Měsíčně (existující workflow, jen pokračovat)

- [ ] ERP Cézar Výdejky MO + VO
- [ ] ERP Cézar Reklamace
- [ ] PPL dopravce
- [ ] Firmy.cz stats (manuální export)
- [ ] Alza Partner orders

---

## 🚀 Co můžu rozjet TEĎ (bez čekání na klienta)

12 z 18 sekcí má všechna data:
- Sekce 0, 1, 2, 3, 4, 7, 8, 10, 11, 12, 13, 14

→ Začínám stavět scaffold + buildy pro tyto sekce.

Sekce čekající na klienta:
- Sekce 5 (Srovnávače) — Alza/Heuréka/Zboží
- Sekce 9 (Pobočky) — Cézar per-provozovna
- Sekce 15 (GBP), 17 (MC) — API setup
- Sekce 16 (Firmy.cz) — Partner API
- Sekce 6 (Feedy) — Merchant Center deep-dive

Tyto sekce budou v dashboardu **viditelné s placeholder „Čekáme na data od klienta — viz EXPORTY_OD_KLIENTA.md"**.

---

## 💬 Kontakt

Pokud máš dotaz k formátu nebo se něco nedá exportovat ve formátu výše — napiš mi a najdeme alternativu (jiné sloupce, jiný delimiter, JSON místo CSV, atd.).

**Důležité**: Lepší **nepřesný export rychle** než **perfektní export pozdě**. Datasety doplníme iterativně.
