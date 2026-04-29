# Audit klicovecentrum.cz — Interaktivní dashboard

Analytický audit a nápravný plán pro klicovecentrum.cz (období: říjen 2025 – duben 2026 + 3-letý ERP přehled 2023-2025).

🔗 **Live**: https://kc-hbgroupcz.github.io/klicovecentrum-audit/dashboard.html
🏢 **Firemní repo**: https://github.com/kc-hbgroupcz/klicovecentrum-audit
📦 Vlastník: H&B Group s.r.o.

## Obsah

- `dashboard.html` — interaktivní dashboard s 40 nálezy + 9 datovými views
- `data/*.json` — 22 agregovaných JSON datových souborů (PPC, ERP, GA4, GSC, SQL, …)
- `index.html` — landing redirect na dashboard

## Funkce dashboardu

### 🏠 Kategorie nálezů (8)
| Kategorie | Počet | Klíčové |
|---|---|---|
| 📊 Měření & analytika | 5 | bot útok, micro-events balast, attribution chyby |
| 🎯 Google Ads | 5 | Smart Bidding na falešné konverze, Meta Ads brand-only |
| 🔵 Sklik | 5 | Zboží.cz 0 conv, budget exhaustion 92 % |
| 📦 Feedy & srovnávače | 5 | OCM tracking rozbitý, feed bez variant |
| 📍 Lokální & pobočky | 5 | 17 prodejen vs lokální PPC efektivita |
| 🛒 Alza marketplace | 5 | 129 k Kč mimo ERP, paralelní kanál |
| 🔍 SEO & Technické | 5 | sitemap kolaps, 91 % bot trafficu |
| 🛡️ Data quality | 4 | CAC neměřen, LTV nereportován, brand split chybí |

### 📊 Data views (9)
1. **📊 Kampaně** — všechny kampaně Google Ads + Sklik + Heuréka detail
2. **📦 Produkty** — 3 197 SKU napříč 5 kanály
3. **🎯 KPI rozpad** — channel × category matice + konverze cross-check + per-kanál ekonomika
4. **🗺️ Regiony** — 13 krajů × ERP obrat + zisk + AOV + cost/order + TOP 3 kategorie
5. **🔀 Konverzní trasy** — funnel (boti odečteni) + Real Purchase vs micro-events
6. **🥊 Konkurence** — 6 tier-1 battlecards + tier 2-3 + smart locks displacement
7. **📅 3-letý přehled** — yearly trend, Páreto Top SKU, segment porovnání, sezónnost forecast
8. **👥 Top zákazníci** — segment-aware (B2C/B2B), Páreto, RFM segmentace
9. **📦 Vratky 3Y** — vratky MO/VO, poštovné per dopravce, reklamace + dopravné odhad
10. **📈 Looker Studio** — agency reports embed

### 🎛️ Globální filtry (header)
- **Období**: 2023 / 2024 / 2025 / Audit 7M / 3-letý přehled
- **Segment**: B2C E-shop MO / B2B Velkoobchod / Oba

## Auditované kanály

GA4 · Search Console · Google Ads · Sklik (Fenix API live) · Zboží.cz · Heuréka · Meta Ads (FB/IG) · Merchant Center · Google Business Profile · Firmy.cz · Mergado · ERP Helios IQ · MySQL kosik · Alza Marketplace

## Klíčové metriky (audit period 7M)

- **Skutečný obrat** (ERP + Alza): 1 300 172 Kč
- **ERP-aligned LTV avg**: 945 Kč
- **Blended LTV:CAC**: 1.22 (hraniční, max safe CAC 315 Kč)
- **Bot traffic**: 90,7 % sessions (1,8 M z 2 M)
- **Konverzní balast**: 99,82 % GA4 conversions = micro-events (time_on_web, view_item)
- **22 kritických nálezů (P0)** ze 40 celkem

## Klíčové metriky (3-letý ERP)

- **2023**: 553 M Kč obrat / 254 M zisk (45,9 % marže) / 16 954 SKU
- **2024**: 543 M Kč / 247 M (45,4 %) / 16 847 SKU · YoY -1,9 %
- **2025**: 535 M Kč / 247 M (46,2 %) / 16 401 SKU · YoY -1,5 %
- **Eshop MO 2025**: 2,76 M Kč (B2C, AOV 1 206)
- **Eshop VO 2025**: 8,31 M Kč (B2B, AOV 4 653, top 20 = 70,7 % obratu)

## Bezpečnost

🔒 **Cloudflare Access** (TODO): Před produkčním nasazením přidat email-based auth gate na audit subdoméně.

## Zdroje dat (live API)

| Zdroj | Auth | Refresh |
|---|---|---|
| GA4 | Service Account | live |
| Search Console | Service Account | live |
| Sklik Fenix | refresh_token | live (1× měsíčně klient vytvoří report) |
| Meta Ads | env token | live |
| MySQL kosik | localhost | live |
| Google Ads | čeká na Sheet ID po Ads Script | manual export → live |
| ERP Helios IQ | CSV export | manual 1× měsíčně |
| Alza Partner | CSV export | manual |
| Heuréka OCM | CSV report | manual |

## Spuštění lokálně

```bash
python -m http.server 8080
# Otevři http://localhost:8080/
```

## Refresh dat (build pipeline)

V parent složce (mimo gitu) je `refresh_all.py`:

```bash
python refresh_all.py --check    # status check (16 live builderů)
python refresh_all.py --live     # jen API zdroje (~30s)
python refresh_all.py            # vše vč. aggregátorů (~3 min)
```

## Historie

- **v1.0** (4/2026): Audit 7M, 6 kategorií, 30 nálezů
- **v1.5** (4/2026): + ERP alignment, Top customers, Konkurence
- **v2.0** (4/2026): + 3-letý ERP přehled, Sezónnost forecast, Sklik+Zboží live
- **v2.1** (4/2026): Přesun na github.com/kc-hbgroupcz (firemní)

---

🤖 Generated with [Claude Code](https://claude.com/claude-code)
