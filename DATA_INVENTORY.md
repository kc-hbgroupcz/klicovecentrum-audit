# 📊 Data Inventory — Marketing Dashboard

**Datum:** 12. 5. 2026
**Role:** Klient (Ondřej) připravuje data + spolupracuje s Claude na buildu. **Žádná agentura**.

---

## 🟢 MÁM data + API connection (build teď)

| # | Zdroj | Připojení | Aktuální data | Refresh |
|---|-------|-----------|---------------|---------|
| 1 | **GA4 API** | Service Account `hbg-101@hbgroup-493608.iam.gserviceaccount.com` | Property 299992437, live | API push, daily |
| 2 | **Google Search Console** | Same SA | klicovecentrum.cz property | API push, daily |
| 3 | **Bing Webmaster** | API key `613abedd85e248d6a26067b14e77deee` | Live | API push, daily |
| 4 | **Google Ads API** | OAuth refresh token | Customer 350-878-7813 admin | API push, daily |
| 5 | **Sklik Fénix API** | JWT refresh token | Live | API push, daily |
| 6 | **Sklik drak XML-RPC** | Token `0x49d49317...` | Live (starší) | API push, daily |
| 7 | **MySQL kosik DB** | XAMPP localhost | 4 weby (web_id 1-4) — kosik, polozka, klient | Local query, daily |
| 8 | **PageSpeed Insights** | API key `AIzaSyBPQHAf1g7vWPPyBt4ufHzKnYm7l-YhsAM` | Live | API push, on-demand |

---

## 🟡 MÁM data jako CSV (klient ručně exportuje + uploaduje)

| # | Zdroj | Aktuální data | Co potřebuji od klienta |
|---|-------|---------------|-------------------------|
| 9 | **ERP Helios Výdejky MO** | 10/2025 - 5/2026 (8 měsíců) | Měsíční export do `data/raw/erp_vydejky_MM_YYYY.csv` |
| 10 | **ERP Helios Výdejky VO** | 10/2025 - 5/2026 | Měsíční export do `data/raw/erp_vydejky_vo_MM_YYYY.csv` |
| 11 | **ERP Helios Reklamace** | 10/2025 - 5/2026 | Měsíční export do `data/raw/erp_reklamace_MM_YYYY.csv` |
| 12 | **PPL dopravce** | 12/2024 - 5/2026 (Helios IQ export) | Měsíční export do `data/raw/ppl_zasilky_MM_YYYY.csv` |
| 13 | **Mergado feed URLs** | Live (URL feed) | URL feedu (existuje) — fetch automatically |

---

## 🟠 MOHU NASTAVIT API (existující OAuth, jen enable APIs + scope)

| # | Zdroj | Co je potřeba | Status |
|---|-------|---------------|--------|
| 14 | **Google Business Profile** | Enable My Business APIs v hbgroup-493608 + scope `business.manage` + manager access na 17 GBP profilech | **Klient zařídí managery v GBP** |
| 15 | **Google Merchant Center** | Enable Content API + Reports API v hbgroup-493608 + scope `content` + Standard user v MC | **Klient zařídí user access v MC** |

---

## 🔴 NEMÁM connection — POTŘEBUJI exporty od klienta

Detaily v `EXPORTY_OD_KLIENTA.md`. Seznam:

| # | Zdroj | Priorita | Pro kterou sekci |
|---|-------|----------|------------------|
| 16 | **Per-pobočka tržby (Helios Maloobchod pokladna)** | P0 | Section 9 (Pobočky) |
| 17 | **Alza Partner orders** | P1 | Section 5 (Srovnávače) |
| 18 | **Heuréka OCM export** | P2 | Section 5 (Srovnávače) |
| 19 | **Zboží.cz OCM export** | P2 | Section 5 (Srovnávače) |
| 20 | **Glami.cz export** | P3 | Section 5 (Srovnávače) |
| 21 | **Meta Ads (Facebook Business)** | P1 | Section 1 (Kampaně) |
| 22 | **Firmy.cz export/scrape** | P1 | Section 16 (Firmy.cz) |
| 23 | **Mailchimp/Ecomail** | P3 (volitelné) | Section 1 sub-tab Email |
| 24 | **Heureka Ověřeno NPS** | P3 (volitelné) | Section 12 sub-tab |

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
| **9 — Pobočky** | 🔴 ČEKÁ | GBP API (po setupu) | **Per-pobočka tržby export (P0)** |
| **10 — Kraje** | 🟢 BUILD TEĎ | SQL kosik PSČ → kraj | — |
| **11 — Doprava** | 🟢 BUILD TEĎ | PPL CSV | Měsíční PPL update |
| **12 — Reklamace** | 🟢 BUILD TEĎ | ERP Reklamace | Měsíční update |
| **13 — Vratky** | 🟢 BUILD TEĎ | SQL kosik (status `vraceno`) | — |
| **14 — Cross-channel** | 🟢 BUILD TEĎ | Agregát z 0-13 | — |
| **15 — GBP** | 🟠 PO SETUPU | GBP API | **Klient zařídí GBP managers** |
| **16 — Firmy.cz** | 🔴 ČEKÁ | (žádné connection) | **Klient zařídí Partner API nebo scraping povolení** |
| **17 — Merchant Center** | 🟠 PO SETUPU | MC API | **Klient zařídí MC user access** |

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
6. Section 9 Pobočky (po Helios Maloobchod pokladna)

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
| ERP Helios Výdejky | Měsíčně (1. den) | **Klient** | CSV upload do `data/raw/` |
| ERP Helios Reklamace | Měsíčně | **Klient** | CSV upload |
| PPL dopravce | Měsíčně | **Klient** | CSV upload |
| Per-pobočka tržby | Měsíčně (po setupu Helios) | **Klient** | CSV upload |
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
