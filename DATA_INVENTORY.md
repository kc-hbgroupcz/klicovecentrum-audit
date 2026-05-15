# 📊 Data Inventory — Marketing Dashboard

**Datum:** 12. 5. 2026 (revize: 4 domény + Cloudflare na všech)
**Role:** Klient (Ondřej) připravuje data + spolupracuje s Claude na buildu. **Žádná agentura**.

---

## 🌐 4 trhy = 4 domény (12.5.2026 všechny za Cloudflare)

| Doména | Trh | Typ | Cloudflare | DB web_id | GA4 property | Sklik |
|--------|-----|-----|------------|-----------|--------------|-------|
| **klicovecentrum.cz** | 🇨🇿 CZ | B2C | ✅ Free + 5 rules | 1 *(?)* | **299992437** | ✅ aktivní |
| **klicovecentrum.sk** | 🇸🇰 SK | B2C | ✅ NOVĚ | 2 *(?)* | ❓ ověřit | ❌ (Sklik je jen CZ) |
| **hbgroup.cz** | 🇨🇿 CZ | B2B | ✅ NOVĚ | 3 *(?)* | ❓ ověřit | ❓ ověřit |
| **hbgroup.sk** | 🇸🇰 SK | B2B | ✅ NOVĚ | 4 *(?)* | ❓ ověřit | ❌ (Sklik je jen CZ) |

> **Otevřené otázky pro klienta** (zaeviduji do EXPORTY):
> 1. SQL `kosik.web_id` 1-4 → mapping na domény (asi nahoře, ale ověřit query)
> 2. Existují separátní GA4 properties pro `klicovecentrum.sk`, `hbgroup.cz`, `hbgroup.sk`? Nebo jeden GA4 sleduje vše?
> 3. Jakou strukturu má Google Ads (350-878-7813) pro 4 domény — 4 separate accounts pod manager? 4 kampaně? 4 conversion goals?
> 4. **„kucovecentrum.sk" v zadání = překlep?** Předpokládám `klicovecentrum.sk` (s „L" navíc) — POTVRDIT.

---

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
| 9 | **ERP Cézar Výdejky MO** | 10/2025 - 5/2026 (8 měsíců) | Měsíční export do `data/raw/erp_vydejky_MM_YYYY.csv` |
| 10 | **ERP Cézar Výdejky VO** | 10/2025 - 5/2026 | Měsíční export do `data/raw/erp_vydejky_vo_MM_YYYY.csv` |
| 11 | **ERP Cézar Reklamace** | 10/2025 - 5/2026 | Měsíční export do `data/raw/erp_reklamace_MM_YYYY.csv` |
| 12 | **PPL dopravce** | 12/2024 - 5/2026 (Cézar export) | Měsíční export do `data/raw/ppl_zasilky_MM_YYYY.csv` |
| 13 | **Mergado feed URLs** | Live (URL feed) | URL feedu (existuje) — fetch automatically |

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

Pro každou sekci dashboardu zvážit jak **rozdělit / agregovat** data napříč 4 doménami:

| Sekce | Per-doména breakdown? | Default view |
|-------|----------------------|--------------|
| 0 — Executive | ✅ ANO + total | Total + 4-card breakdown |
| 1 — Kampaně | ✅ ANO (per Google Ads sub-account) | Per-trh tab (CZ/SK) |
| 2 — SEO | ✅ ANO (4 GSC properties) | Per-trh tab |
| 3 — B2C | 2 domény (CZ + SK) | Total + per-trh |
| 4 — B2B | 2 domény (CZ + SK) | Total + per-trh |
| 5 — Srovnávače | Heuréka.cz/sk, Zboží jen CZ | Per-trh |
| 6 — Feedy | Per-doména MC accounts (4 separate?) | Per-trh tab |
| 7 — Produkty | Sdílené (jeden katalog) | Total |
| 8 — Kategorie | Per-trh (jiná struktura?) | Per-trh |
| 9 — Pobočky | 17 prodejen v CZ + SK? | Total (pobočky obsluhují oba trhy) |
| 10 — Kraje | CZ + SK kraje | Per-trh |
| 11 — Doprava | CZ (PPL) + SK (jiný dopravce?) | Per-trh |
| 12 — Reklamace | Sdílené (jeden ERP) | Total + per-doména |
| 13 — Vratky | Per-doména | Total + per-trh |
| 14 — Holistic | Total + 4-card | Total |
| 15 — GBP | 17 prodejen lokálně | Total |
| 16 — Firmy.cz | Jen CZ (Seznam je CZ) | CZ |
| 17 — MC | Per-account (4 separate?) | Per-trh tab |

**Filter design**: Globální dropdown v hlavičce dashboardu vedle date pickeru:
- **Trh**: Vše / CZ / SK
- **Typ**: Vše / B2C / B2B
- **Doména**: Vše / klicovecentrum.cz / klicovecentrum.sk / hbgroup.cz / hbgroup.sk

Filtr se aplikuje na celý dashboard současně (jako date picker). URL state.

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
