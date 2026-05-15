# 📊 Data Inventory — Marketing Dashboard

**Datum:** 12. 5. 2026 (revize: 4 domény + Cloudflare na všech)
**Role:** Klient (Ondřej) připravuje data + spolupracuje s Claude na buildu. **Žádná agentura**.

---

## 🌐 4 trhy = 4 domény (12.5.2026, klientovy odpovědi zaznamenány)

> **Pozor na názvy**: CZ = kli**c**ovecentrum, SK = klu**c**ovecentrum (jiná značka kvůli překladu „klíč" → „kľúč").

| Doména | Trh | Typ | Cloudflare | GA4 property | Google Ads | Sklik | Merchant Center |
|--------|-----|-----|------------|--------------|------------|-------|-----------------|
| **klicovecentrum.cz** | 🇨🇿 CZ | B2C | ✅ Free + 5 rules | **299992437** | ✅ Customer 350-878-7813 | ✅ Drak + Fénix tokeny | ✅ Merchant 130672692 |
| **klucovecentrum.sk** | 🇸🇰 SK | B2C | ✅ NOVĚ | ❓ ID | ✅ separátní účet (klient pošle ID) | ❌ N/A (SK trh) | ⏳ klient zařídí |
| **hbgroup.cz** | 🇨🇿 CZ | B2B | ✅ NOVĚ | ❓ ID | ❌ **NEPOUŽÍVÁ ADS** | ❌ N/A (žádné PPC) | ❌ N/A (žádný Shopping) |
| **hbgroup.sk** | 🇸🇰 SK | B2B | ✅ NOVĚ | ❓ ID | ❌ **NEPOUŽÍVÁ ADS** | ❌ N/A | ❌ N/A |

### 🔑 Klíčové insighty z klientových odpovědí

1. **B2B = žádné PPC** — `hbgroup.cz` + `hbgroup.sk` nepoužívají Google Ads, Sklik ani Merchant Center
   - **Důsledek pro Section 1 (Kampaně)**: ukázat jen B2C trhy (klicovecentrum.cz + klucovecentrum.sk)
   - **B2B akvizice je o čem?** SEO, direct traffic, B2B sales tým, vztahy se zákazníky → zaměřit Section 4 (B2B) na **retention metriky** místo akvizičních

2. **ERP Cézar = PRIMARY zdroj revenue/orders** (NE SQL kosik!)
   - SQL kosik je sekundární / pro detail-level analýzu (per-customer journey, košíky abandoned)
   - Všechny revenue / orders / GMV / margin metriky → vždy z ERP Cézar
   - Cross-reference s SQL kosik jen pro:
     - Customer email + identifikace per zákazník
     - Cart abandonment (košíky bez následného Cézar prodeje)
     - Web traffic linkable to orders

3. **GA4 = 4 separate properties** — klient má per doménu vlastní property
   - Property 299992437 = klicovecentrum.cz (mám)
   - 3 zbývající ID musíme získat (1 dotaz)

4. **Google Ads = 2 účty (jen B2C)**:
   - klicovecentrum.cz: 350-878-7813 (mám)
   - klucovecentrum.sk: ❓ Customer ID (klient pošle)

5. **Merchant Center = jen klicovecentrum.cz dnes**
   - Merchant 130672692 ✅
   - klucovecentrum.sk → klient zařídí MC + user access
   - hbgroup.cz/sk → nikdy (B2B nepoužívá Shopping)

---

---

## 🥇 PRIMARY: ERP Cézar (revenue / orders / margins)

> **DŮLEŽITÉ pravidlo**: Pro VŠECHNY revenue / orders / GMV / margin metriky napříč všemi 4 doménami → **vždy ERP Cézar**, NIKDY SQL kosik. SQL je jen pro detail-level customer journey.

| Co | Zdroj | Frekvence |
|----|-------|-----------|
| Per-doména daily revenue | ERP Cézar Výdejky MO + VO | Měsíční CSV upload klientem |
| Per-zákazník orders | ERP Cézar | Měsíční CSV upload |
| Marže | ERP Cézar (real nákupní ceny) | Měsíční CSV upload |
| Reklamace | ERP Cézar | Měsíční CSV upload |
| Per-pobočka tržby | ERP Cézar Maloobchod pokladna | Klient zařídí export (pending) |

---

## 🟢 MÁM data + API connection (build teď)

| # | Zdroj | Připojení | Pokrývá které domény | Refresh |
|---|-------|-----------|---------------------|---------|
| 1 | **GA4 API** (klicovecentrum.cz) | Service Account `hbg-101@hbgroup-493608.iam.gserviceaccount.com` | klicovecentrum.cz (property 299992437) | API push, daily |
| 1b | **GA4 API** (3 další properties) | Same SA, čekáme IDs od klienta | klucovecentrum.sk + hbgroup.cz + hbgroup.sk | po obdržení IDs |
| 2 | **Google Search Console** | Same SA | klicovecentrum.cz; ostatní 3 GSC properties → klient přidá SA jako User |
| 3 | **Bing Webmaster** | API key `613abedd85e248d6a26067b14e77deee` | klicovecentrum.cz; ostatní → klient přidá site |
| 4a | **Google Ads** (klicovecentrum.cz) | OAuth refresh token | Customer **350-878-7813** | API push, daily |
| 4b | **Google Ads** (klucovecentrum.sk) | OAuth refresh token (**stejný OAuth, jiný customer**) | ❓ Customer ID — klient pošle | po obdržení ID |
| 5 | **Sklik Fénix API** | JWT refresh token | klicovecentrum.cz only (Sklik je CZ) | API push, daily |
| 6 | **Sklik drak XML-RPC** | Token `0x49d49317...` | klicovecentrum.cz only | API push, daily |
| 7 | **MySQL kosik DB** | XAMPP localhost | 4 weby (web_id 1-4) — **POUZE pro detail/journey, NE revenue** | Local query, daily |
| 8 | **PageSpeed Insights** | API key | Per URL on-demand | API push, on-demand |
| 9 | **Google Business Profile API** | Same SA, manager access | 17 prodejen (CZ) | quota limited 1 req/min |
| 10 | **Google Merchant Center API** | Same SA, Standard user | klicovecentrum.cz (Merchant 130672692) | API push, daily |

---

## 🟡 MÁM data jako CSV (klient ručně exportuje + uploaduje)

| # | Zdroj | Pokrývá domény | Co potřebuji od klienta |
|---|-------|----------------|-------------------------|
| 11 | **ERP Cézar Výdejky MO (B2C)** | klicovecentrum.cz + klucovecentrum.sk | Měsíční CSV `data/raw/erp_vydejky_mo_YYYY_MM.csv` (možná split per doména?) |
| 12 | **ERP Cézar Výdejky VO (B2B)** | hbgroup.cz + hbgroup.sk | Měsíční CSV `data/raw/erp_vydejky_vo_YYYY_MM.csv` |
| 13 | **ERP Cézar Reklamace** | Všechny 4 (sdílený ERP) | Měsíční CSV |
| 14 | **PPL dopravce** | CZ (klicovecentrum.cz + hbgroup.cz) — SK má jiného dopravce? | Měsíční CSV |
| 15 | **Mergado feed URL** | klicovecentrum.cz only zatím | Live URL feed — auto fetch |

---

## 🟢 API access HOTOVÝ (12.5.2026) — připojuju TEĎ

| # | Zdroj | Status | Příští kroky |
|---|-------|--------|--------------|
| 14 | **Google Business Profile** | ✅ Manager access HOTOVO | Enable APIs v Cloud Console + test API call + buildy |
| 15 | **Google Merchant Center** | ✅ Standard user access HOTOVO | Enable Content + Reports API + test API call + buildy |
| 16 | **Meta Marketing API** | ⏳ Čeká token od klienta | Po obdržení tokenu: napojit + test |
| 17 | **Heuréka Reporting** | ⏳ Vyjasnit s klientem | API klíč existuje? Pokud ano → API. Pokud ne → manuální CSV |

---

## 🔴 NEMÁM connection — POTŘEBUJI exporty od klienta

Detaily v `EXPORTY_OD_KLIENTA.md`. Seznam:

| # | Zdroj | Priorita | Pro kterou sekci |
|---|-------|----------|------------------|
| 18 | **Per-pobočka tržby (Cézar Maloobchod pokladna)** | P0 | Section 9 (Pobočky) |
| 19 | **Alza Partner orders** | P1 | Section 5 (Srovnávače) |
| 20 | **Zboží.cz OCM export** | P2 | Section 5 (Srovnávače) |
| 21 | **Glami.cz export** | P3 | Section 5 (Srovnávače) |
| 22 | **Firmy.cz import feed master tabulka** | P1 | Sync s Firmy.cz (jednorázové) |
| 23 | **Firmy.cz stats CSV (manuální)** | P2 | Section 16 (Firmy.cz statistiky) |
| 24 | **Mailchimp/Ecomail** | P3 (volitelné) | Section 1 sub-tab Email |
| 25 | **Heureka Ověřeno NPS** | P3 (volitelné) | Section 12 sub-tab |

### 📤 Firmy.cz — speciální workflow

Firmy.cz nemá statistiky API. Má pouze:
1. **Import feed** (JSON schema v1.7) — my generujeme + posíláme Seznamu, sync naších profilů
2. **Manuální CSV export statistik** z admin.firmy.cz — klient ručně měsíčně

Pro import feed potřebuju jednorázově master tabulku 17 prodejen → vytvořím auto-generator → daily refresh.

---

## 🌍 Multi-domain implications pro každou sekci

Klientovy odpovědi (12.5.2026) zjednodušily strukturu — **B2B nepoužívá ads/srovnávače/MC**, takže akviziční sekce jsou jen pro B2C (2 trhy).

| Sekce | Pokrývá které domény? | Default view |
|-------|----------------------|--------------|
| 0 — Executive | Všechny 4 (revenue agreg) | Total + 4-card |
| 1 — Kampaně | **Jen B2C** (klicovecentrum.cz + klucovecentrum.sk) | Per-trh tab CZ/SK; B2B disabled |
| 2 — SEO | 4 GSC properties | Per-doména tab |
| 3 — B2C | 2 domény (CZ + SK B2C) | Total + per-trh |
| 4 — B2B | 2 domény (CZ + SK B2B) — **focus na retention, ne akvizici** | Total + per-trh |
| 5 — Srovnávače | **Jen B2C** (Heuréka, Zboží.cz, Alza — žádný B2B Shopping) | Per-trh |
| 6 — Feedy | **Jen B2C** (MC jen pro klicovecentrum.cz, klucovecentrum.sk čeká) | klicovecentrum.cz only zatím |
| 7 — Produkty | Sdílené (jeden katalog v ERP) | Total + per-doména split |
| 8 — Kategorie | Per-doména (jiná navigace) | Per-doména |
| 9 — Pobočky | 17 prodejen (CZ obsluha — SK je e-shop only?) | Total |
| 10 — Kraje | CZ kraje (klicovecentrum.cz + hbgroup.cz) + SK kraje (klucovecentrum.sk + hbgroup.sk) | Per-trh |
| 11 — Doprava | PPL (CZ) + slovenský dopravce (SK?) | Per-trh |
| 12 — Reklamace | Sdílené (jeden ERP) | Total + per-doména |
| 13 — Vratky | Per-doména | Total + per-trh |
| 14 — Holistic | Total + 4-card | Total |
| 15 — GBP | 17 prodejen lokálně (CZ?) | Total |
| 16 — Firmy.cz | Jen CZ (Seznam je CZ) | CZ |
| 17 — MC | Jen klicovecentrum.cz (zatím) | klicovecentrum.cz only |

**Filter design**: Globální dropdown v hlavičce dashboardu:
- **Trh**: Vše / CZ / SK
- **Typ**: Vše / B2C / B2B
- **Doména**: Vše / klicovecentrum.cz / klucovecentrum.sk / hbgroup.cz / hbgroup.sk

**Zámky filtru** (interlocked):
- Vyberu doménu → auto-set market + type
- Vyberu Typ = B2B → Section 1 (Kampaně), 5 (Srovnávače), 6 (Feedy), 17 (MC) zobrazí empty state „B2B nepoužívá tyto kanály"
- Vyberu Trh = SK → Section 16 (Firmy.cz) zobrazí empty state „Firmy.cz je jen pro CZ"

---

## 🗂️ Mapping sekcí → data sources → status

| Sekce | Status | Data sources | Co potřebuju |
|-------|--------|--------------|--------------|
| **0 — Executive Summary** | 🟢 BUILD TEĎ | GA4, GAds, Sklik, ERP, SQL | — |
| **1 — Kampaně PPC** | 🟢 BUILD TEĎ (částečně) | GAds API, Sklik API | + Meta Ads export (P1) |
| **2 — SEO** | 🟢 BUILD TEĎ | GSC, Bing | — |
| **3 — B2C E-shop** | 🟢 BUILD TEĎ | SQL kosik, ERP MO | — |
| **4 — B2B** | 🟢 BUILD TEĎ | SQL kosik (web_id 3), ERP VO | — |
| **5 — Srovnávače** | 🟡 ČÁSTEČNĚ | (žádné connection) | Heuréka, Zboží, Glami, Alza exporty |
| **6 — Feedy** | 🟡 ČÁSTEČNĚ | Mergado URL | + Google Merchant Center API (po setupu) |
| **7 — Produkty** | 🟢 BUILD TEĎ | ERP per-položkový, SQL polozka | — |
| **8 — Kategorie** | 🟢 BUILD TEĎ | SQL polozka + kategorie web | — |
| **9 — Pobočky** | 🟡 GBP READY, ČEKÁ na ERP | GBP API ✅ ready | **Per-pobočka tržby export z Cézar (P0)** |
| **10 — Kraje** | 🟢 BUILD TEĎ | SQL kosik PSČ → kraj | — |
| **11 — Doprava** | 🟢 BUILD TEĎ | PPL CSV | Měsíční PPL update |
| **12 — Reklamace** | 🟢 BUILD TEĎ | ERP Reklamace | Měsíční update |
| **13 — Vratky** | 🟢 BUILD TEĎ | SQL kosik (status `vraceno`) | — |
| **14 — Cross-channel** | 🟢 BUILD TEĎ | Agregát z 0-13 | — |
| **15 — GBP** | 🟢 BUILD TEĎ | GBP API ✅ | Enable APIs v Cloud Console + buildy |
| **16a — Firmy.cz import feed** | 🟡 ČEKÁ master tabulku | Vlastní generator | **Master tabulka 17 prodejen** |
| **16b — Firmy.cz stats** | 🔴 MANUAL | CSV manuální export | **Klient měsíčně z admin.firmy.cz** |
| **17 — Merchant Center** | 🟢 BUILD TEĎ | MC API ✅ | Enable Content + Reports API + buildy |

---

## 📊 Co můžu postavit HNED (12 z 18 sekcí)

**Sekce ready pro Phase 1 buildu** (data mám, API funguje):

✅ Section 0 — Executive Summary (agregát z existujících)
✅ Section 1 — Kampaně (Google Ads + Sklik, bez Meta zatím)
✅ Section 2 — SEO (GSC + Bing)
✅ Section 3 — B2C E-shop (SQL + ERP MO)
✅ Section 4 — B2B (SQL + ERP VO)
✅ Section 7 — Produkty (ERP per-položkový)
✅ Section 8 — Kategorie (SQL polozka)
✅ Section 10 — Kraje (SQL PSČ → kraj mapping)
✅ Section 11 — Doprava (PPL CSV)
✅ Section 12 — Reklamace (ERP)
✅ Section 13 — Vratky (SQL status)
✅ Section 14 — Cross-channel (agregát)

**Sekce čekající na klienta:**
⏳ Section 5 — Srovnávače (Heuréka/Zboží/Glami/Alza exporty)
⏳ Section 6 — Feedy deep-dive (Merchant Center API setup)
⏳ Section 9 — Pobočky (per-pobočka tržby export)
⏳ Section 15 — GBP (managers access)
⏳ Section 16 — Firmy.cz (Partner API)
⏳ Section 17 — Merchant Center (user access)

---

## 🎯 Plán implementace

**Fáze 1 — TEĎ (1-2 týdny)**:
1. Marketing dashboard scaffold (header, date picker, tabs)
2. Build 12 sekcí které mám data
3. Daily refresh cron (Windows Task Scheduler nebo GitHub Actions)
4. Alert email pokud refresh selže

**Fáze 2 — Po dodání exportů od klienta**:
5. Section 5 Srovnávače (po exportech)
6. Section 9 Pobočky (po Cézar Maloobchod pokladna)

**Fáze 3 — Po API setupech**:
7. Section 15 GBP (po managers access)
8. Section 16 Firmy.cz (po Partner API rozhodnutí)
9. Section 17 Merchant Center (po user access)

**Fáze 4 — Polish**:
10. Mobile responsive
11. Acceptance testing
12. Documentation

---

## 📅 Aktualizační workflow (po launchi)

| Data | Frekvence | Kdo | Jak |
|------|-----------|-----|-----|
| GA4 / GSC / Bing / GAds / Sklik | Daily 6:00 | **Automatický cron** | API push |
| MySQL kosik | Daily 6:00 | **Automatický cron** | Local query |
| ERP Cézar Výdejky | Měsíčně (1. den) | **Klient** | CSV upload do `data/raw/` |
| ERP Cézar Reklamace | Měsíčně | **Klient** | CSV upload |
| PPL dopravce | Měsíčně | **Klient** | CSV upload |
| Per-pobočka tržby | Měsíčně (po setupu Cézar) | **Klient** | CSV upload |
| Alza Partner | Týdně | **Klient** | CSV upload |
| Heuréka/Zboží OCM | Daily (po API setupu) | **Automatický cron** | API push |
| GBP / MC | Daily (po API setupu) | **Automatický cron** | API push |
| Firmy.cz | Weekly (po setupu) | **Klient nebo scraper** | CSV nebo automated |

---

## 🚨 Co se stane když klient něco nedodá

**Pro každou sekci**:
- ✅ Pokud data přijdou → sekce live + alerts
- ⏳ Pokud data NE → sekce zobrazí *„Čeká na data od klienta — viz EXPORTY_OD_KLIENTA.md"*
- 🚧 Žádná sekce nezablokuje ostatní

**Pro daily refresh**:
- Pokud API selže (GBP token expired) → dashboard ukáže poslední úspěšný refresh + alert email
- Pokud CSV stale > 30 dní → sekce flaguje „Data zastaralá — klient nedodal měsíční update"
