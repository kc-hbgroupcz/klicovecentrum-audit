# 📋 Zadání: Marketing Dashboard klicovecentrum.cz + hbgroup.cz
**Klient:** H&B Group s.r.o. (Ondřej Skřebský, oskrebsky@gmail.com)
**Datum zadání:** 12. 5. 2026 (revize: +3 sekce GBP/Firmy.cz/MC)
**Předpokládaný termín dodání:** 5-7 týdnů
**Předpokládaný rozpočet:** dle nabídky agentury (orientačně 95-140 hod)

---

## 1. Účel a rozsah

### Cíl
Vytvořit **live operační marketing dashboard** pro klicovecentrum.cz + hbgroup.cz, který:
- Refreshuje se **každých 24 hodin v 6:00 ráno** z CSV exportů + API
- Pošle **email alert** pokud CSV nebylo aktualizováno > 25h
- Je dostupný pouze **přes Cloudflare Access** (per-user auth)
- Tvoří součást **rozcestníku** na `audit.klicovecentrum.cz` (vedle existujícího audit dashboardu)

### Persona uživatelů
1. **CMO / vedení H&B Group** — daily 30-sec přehled, weekly deep dive
2. **Externí marketingová agentura** — operativní rozhodnutí o spend allocation
3. **CEO / účetní** — měsíční výsledkové reporty (export)

### Klíčový rozdíl proti existujícímu audit dashboardu
- **Audit dashboard** (`/audit/`) = historická retrospektiva 7M auditního období + 70 nálezů s action items
- **Marketing dashboard** (`/marketing/`) = **operativní LIVE provoz** — daily refresh, trend monitoring, alerty na anomálie

---

## 2. Architecture & Tech requirements

### Tech stack (musí být kompatibilní s existujícím)
- **Frontend**: HTML + vanilla JS + Chart.js 4.4.1 (jako audit dashboard) NEBO React/Vue (po dohodě)
- **Backend**: Python 3.11+ build skripty (kompatibilní s existujícími `build_*.py`)
- **Hosting**: GitHub Pages (repo `kc-hbgroupcz/marketing-dashboard` nebo subfolder `kc-hbgroupcz/klicovecentrum-audit/marketing/`)
- **Auth**: Cloudflare Access (existuje pro `audit.klicovecentrum.cz`)
- **Cron**: Windows Task Scheduler na klientově PC (daily 6:00) NEBO GitHub Actions cron NEBO Cloudflare Workers cron
- **Alert delivery**: SMTP email (Gmail App Password) na `oskrebsky@gmail.com`

### Data architecture
```
┌─────────────────────────────────────────────────────────────┐
│ CSV inputs (manuálně klient uploaduje do OneDrive)         │
│  • Helios IQ — VýdejkyMO, VýdejkyVO, Reklamace, PPL        │
│  • Google Ads — Ads Scripts JSON export (denně)            │
│  • Sklik — UI export (1× měsíčně)                          │
│  • Heuréka, Zboží.cz — OCM CSV (1× měsíčně)                │
│  • Alza Partner — orders CSV (1× měsíčně)                  │
└─────────────────┬───────────────────────────────────────────┘
                  │ (OneDrive auto-sync nebo manuální upload)
                  ▼
┌─────────────────────────────────────────────────────────────┐
│ API inputs (auto-pull každý den 6:00)                      │
│  • GA4 Data API (Service Account, sessions/trans)          │
│  • Search Console API (organic clicks/queries)             │
│  • Bing Webmaster API (crawl stats, queries)               │
│  • Sklik Fénix API (kampaně)                               │
│  • MySQL kosik (lokální SQL DB, web_id 1-4)                │
└─────────────────┬───────────────────────────────────────────┘
                  ▼
┌─────────────────────────────────────────────────────────────┐
│ Python build pipeline (denní cron 6:00)                    │
│  1. Pull live data z API                                    │
│  2. Detect new CSVs v OneDrive                             │
│  3. Run build skripty → JSON files do /data/               │
│  4. Update /data/_meta.json s last_update timestamp        │
│  5. Pokud kterýkoli CSV stale > 25h → email alert          │
│  6. Git commit + push                                       │
└─────────────────┬───────────────────────────────────────────┘
                  ▼
┌─────────────────────────────────────────────────────────────┐
│ Frontend (live polling)                                     │
│  • Fetch /data/_meta.json každých 5 min                    │
│  • Pokud last_update > 25h → red warning banner v UI       │
│  • Pokud last_update > 7 dní → critical alert + email      │
└─────────────────────────────────────────────────────────────┘
```

### Source of Truth priority

**KLÍČOVÁ POZNÁMKA**: Klient si výslovně přeje **ERP (Helios IQ) jako primární zdroj** pro revenue/orders metriky, NE SQL kosik tabulku.

| Metrika | Zdroj | Důvod |
|---------|-------|-------|
| Daily revenue, orders | **ERP Helios IQ Výdejky** (MO+VO) | Účetní pravda po refundacích/storno |
| Konverze, ROAS | **ERP** (ne GA4) | GA4 inflated bot daty |
| Marže, zisk | **ERP** | Helios kalkuluje s reálnými nákupními cenami |
| Customer LTV | ERP + SQL kosik cross-ref | SQL pro per-customer historiku |
| Channel attribution | GA4 + Sklik + Google Ads | Live API |
| SEO traffic | Search Console + Bing | Live API |

---

## 3. 18 sekcí dashboardu — KPI specifikace

### 🎯 Section 0 — EXECUTIVE SUMMARY (top, always visible)

**6 hero cards:**
| KPI | Source | Refresh | Alert threshold |
|-----|--------|---------|------------------|
| Daily revenue (B2C+B2B+Alza+SK) | ERP CSV | daily | < 80 % weekly avg |
| Marketing spend / day | Google Ads + Sklik + Heuréka + Zboží + Alza Ads | daily | > 120 % budget |
| **Blended ROAS** (ERP-based) | ERP revenue / Total spend | daily | < 3,0× |
| Real Conversion Rate | ERP orders / GA4 sessions | daily | < 0,5 % |
| Active campaigns count | Google Ads + Sklik | daily | 0 LOCAL = warning |
| Open reklamace count | Helios CSV | daily | > 100 |

**Status bar**: ⚠️ Data last updated 2h ago | Next refresh 6:00 | 3 alerts active

---

### 📢 Section 1 — KAMPANĚ (PPC)

**Zdroj revenue: ERP Helios IQ** (ne SQL kosik!)

**Hero KPIs (5):**
1. **ERP-based ROAS** = ERP revenue attribuovaný kampani / spend (per kanál)
2. **Spend share %** per kanál (Google Ads / Sklik / Heuréka / Zboží / Alza Ads)
3. **CPA real** = Cost / ERP orders
4. **Impression Share** (Google Ads + Sklik) — kde ztrácíme exposure
5. **Quality Score avg** per kanál

**Per-kategorie split** (PRODUCT / BRAND / LOCAL / RTG):
- Spend %, ROAS, conversions per kategorie
- LOCAL kampaně per pobočka — ROAS + store visit conversions (GBP linked)
- Identifikace „zlatých žil" jako Sklik 0 VG Brand 23,89× ROAS

**Supporting (8):**
- CTR per kampaň (top 10)
- Avg CPC trend (sledovat 2× nárůst od 2023)
- Conv rate per kampaň
- Search lost IS budget vs rank
- Negative keywords coverage %
- Customer Match audience size
- Top 10 search queries
- **MER** = Total ERP revenue / Total marketing spend

**Alerts:**
- Spend > 120 % budget → warning
- ROAS < 2,0 → warning
- IS lost budget > 30 % → optimization opportunity

---

### 🔍 Section 2 — SEO

**Hero KPIs (5):**
1. Organic clicks (90-day rolling)
2. Organic CTR (target 1,5 % — current 0,90 %)
3. Avg position top 50 queries
4. Indexed URLs (GSC + Bing)
5. AI Citability score (GEO measurement)

**Supporting (8):**
- Brand vs non-brand split (target 40/60)
- Top 10 stránky CWV (LCP/CLS/INP z PageSpeed API)
- Crawl budget Bing/GSC
- Sitemap warnings count
- **New ranking keywords / week**
- **Lost ranking keywords / week**
- Backlinks count
- **Branded search trend** (Google Trends)

**Alerts:**
- Daily clicks pokles > 30 % → útok/penalty
- AI Citability < 30 → AI search nedostupný

---

### 🛒 Section 3 — B2C E-shop (klicovecentrum.cz)

**Zdroj revenue: ERP Helios IQ Výdejky MO**

**Hero KPIs (5):**
1. Daily revenue (ERP MO eshop)
2. Daily orders (ERP)
3. AOV
4. Real Conv Rate = ERP orders / GA4 sessions
5. Repeat customer rate

**Supporting (8):**
- New vs Returning ratio
- **Cart Abandonment funnel**: add_to_cart → checkout → purchase
- Time-to-Purchase
- Top 20 SKU rolling 30d
- Top 10 kategorií
- Anonymous vs registered ratio
- **Cohort retention table** (kupci z měsíce X, kolik koupilo X+1, X+3, X+6)
- **Mobile vs Desktop conv rate**

**Alerts:**
- Daily orders < 50 % avg → critical (payment outage signal)
- Cart-to-checkout < 50 % → funnel issue

---

### 🏢 Section 4 — B2B (hbgroup.cz)

**Zdroj revenue: ERP Helios IQ Výdejky VO** (ne SQL kosik!)

**Hero KPIs (5):**
1. Daily B2B revenue (ERP VO)
2. Top 10 partneři % share (target < 60 % = diverzifikace)
3. Avg B2B AOV
4. B2B margin (ERP)
5. Active partners (last 30 days)

**Supporting (8):**
- New B2B accounts/měsíc
- **B2B Churn alerts** — partneři > 60 dnů offline (red flag, kontaktovat)
- **B2B Net Revenue Retention** (NRR) meziroční existujících
- B2B vs B2C ratio
- Top partneři: sales velocity
- **Partner Concentration risk** — top 1 partner > 10 % = risk
- **Repeat order interval** per partner
- Geographic B2B distribution

**Alerts:**
- Top 10 partneři > 70 % → over-concentration
- TOP 5 partner offline > 30 dnů → 🔴 kontaktovat ihned

---

### ⚖️ Section 5 — SROVNÁVAČE + ALZA

**Zahrnout 4 marketplaces:**

**Hero KPIs (5):**
1. **Heuréka ROAS** (ERP-attributed)
2. **Zboží.cz ROAS** (ERP-attributed)
3. **Alza GMV** (Alza Partner API exports, ne PPC ale revenue z marketplace)
4. Cost share % (mezi 4 platformami)
5. Conv rate per platforma

**Supporting (6):**
- Rating + reviews count per platforma
- Top performing SKU per platforma
- Heuréka + Zboží bidding diagnostika
- **Alza katalog health**: 50 % „Prodej skončil" produkty
- **Compare price competitiveness** (kde jsme drazí vs trh)
- **Review velocity** (new reviews / měsíc)

**Alerts:**
- ROAS < 2,0 → unprofitable
- Alza „Prodej skončil" > 30 % → reaktivace potřeba

---

### 📦 Section 6 — FEEDY (Mergado + Google Merchant Center)

**Hero KPIs (5):**
1. Total active products in feeds (Mergado output count)
2. **Disapproved count v Google Merchant Center** (per reason: image too small, missing GTIN, price mismatch, …)
3. Last sync timestamp per feed (Google MC / Heuréka / Zboží / Alza / Glami)
4. **Coverage %**: bestsellers ve feedu vs ERP TOP 50
5. Feed quality score (vážený index issues)

**Supporting (8):**
- Per-feed health (per channel — kolik produktů, kolik issues)
- Variant coverage % (target 100 %) — současný stav 0 %, viz finding #75
- Bestsellers chybí ve feedu count
- **Feed-to-ERP price diff alerts** (cena ve feedu ≠ aktuální)
- **Stock-out coverage** (out-of-stock items pořád ve feedu)
- Custom_label_1 distribution (PMax labely 0/1/2/3 — profitable, low-margin, etc.)
- **Account Suspension status** (Google MC / Heuréka / Alza)
- **Click-through rate per feed channel** (z MC Performance API)

**Google Merchant Center detail (sub-tab):**
- Account-level: Suspended / Active, dataset count, last refresh
- **Issues breakdown**: per-attribute (image, GTIN, brand, GPC category, shipping, tax)
- **Per-product disapproval list** (top 50 nejvýdělečnějších produktů s issue)
- **Best Sellers report** (Google MC → market share top kategorií)
- **Price competitiveness report** (jak jsme proti konkurenci v Shopping)
- **Promotions feed status** (slevy / akce)
- Shipping settings audit (správné sazby per kraj)

**Alerts:**
- Disapproved > 5 % všech produktů → warning, > 20 % → critical
- Account suspended → P0 incident (= Shopping ads zastaveny)
- Last sync > 24h → feed broken
- Top 10 bestsellers s disapproval → P1 (přímý dopad na výnosy)
- Variant coverage drop > 5 % WoW → upozornění

---

### 🔧 Section 7 — PRODUKTY

**Zdroj: ERP Helios IQ per-položkový obrat + Mergado feed cross-ref**

**Hero KPIs (5):**
1. Top 20 SKU revenue (ERP rolling 30d)
2. Bestsellers count (>10 prodejů/měs)
3. New SKU added / měsíc
4. Discontinued SKU count
5. Per-vendor revenue (top 10 dodavatelů)

**Supporting (6):**
- **Stock-out frequency** per top SKU
- **Margin distribution** (ERP-aligned)
- PMax label split (profitable / zombie / costly)
- **Cross-sell affinity** (co se kupuje s top SKU)
- **Product velocity score** (pageviews × add_to_cart × purchase)
- **Product return rate** per SKU (Reklamace cross-ref)

**Alerts:**
- Top 5 SKU pokles > 30 % MoM → product issue
- Stock-out > 7 dní u TOP 50 SKU → revenue lost

---

### 📂 Section 8 — KATEGORIE

**Hero KPIs (5):**
1. Top 10 kategorií revenue (ERP rolling 30d)
2. Per-kategorie marže
3. MoM trend per kategorie
4. Conv rate per kategorie
5. **Cross-sell ratio** (objednávky s 2+ kategorie)

**Supporting (5):**
- New entries per kategorie / měsíc
- Per-kategorie organic traffic (Search Console)
- Per-kategorie PPC CPA
- **Category lifecycle stage** (growth / mature / declining)
- **Seasonality index** per kategorie

---

### 📍 Section 9 — POBOČKY (17 fyzických)

**Hero KPIs (5):**
1. **Per-pobočka tržby** (ERP — vyžaduje per-provozovna export z Helios, klient zařídí)
2. GBP zobrazení / volání / navigace per pobočka
3. Firmy.cz CTR per pobočka
4. Local Google Ads ROAS + store visit conversions
5. Reviews + rating per pobočka

**Supporting (6):**
- Mystery shopping score (pokud zavedeno)
- **Per-pobočka Maps Pack ranking** (sledovat „klíče Plzeň" SERP)
- **Hours utilization** (peak vs idle)
- **Walk-in vs phone call ratio**
- Per-pobočka local keyword rankings
- Status: LOCAL kampaň aktivní/chybí (13 chybí dle auditu)

**Alerts:**
- Pobočka bez LOCAL kampaně → opportunity
- Call rate < 0,2 % → engagement issue
- Negative review (1-2★) → 24h response window

---

### 🗺️ Section 10 — KRAJE

**Zdroj: ERP + SQL PSČ → kraj mapping**

**Hero KPIs (5):**
1. Top 5 krajů revenue (ERP-aligned)
2. Cost per kraj (Google Ads geo split)
3. CPA per kraj
4. Conv rate per kraj
5. **Pokrytí kraj × pobočka × revenue** matrix

**Supporting (5):**
- New customers per kraj
- Repeat rate per kraj
- AOV per kraj
- **Penetration rate** (% population s nákupem, dat z ČSÚ)
- **Underserved regions** (high traffic + low conv = chybí pobočka)

---

### 🚚 Section 11 — DOPRAVA (PPL + ostatní)

**Hero KPIs (5):**
1. Daily zásilky count (PPL export)
2. Avg dobírka % (current 15 %)
3. **On-time delivery rate** (pokud PPL data poskytuje)
4. Top 10 destinace měst
5. Interní vs externí ratio (currentl 23/77)

**Supporting (5):**
- **Delivery time SLA** (ordered → delivered avg dnů)
- **Lost packages count** (claims rate)
- Per-recipient PPL frequency
- Per-kraj PPL share
- **PPL cost per package** (negotiation lever)

**Alerts:**
- PPL volume = 0 v pracovní den → carrier outage
- On-time < 90 % → SLA breach

---

### ⚠️ Section 12 — REKLAMACE

**Hero KPIs (5):**
1. Reklamační poměr (current month %, target < 3 %)
2. Total open reklamací (currentl 86)
3. Avg doba vyřízení (target < 30 dní)
4. **% > 30 dnů** (ObčZ § 2172 violation risk!)
5. Top 5 problematických produktů

**Supporting (5):**
- Top reklamující zákazníci (B2B partneři risk!)
- **Reasons breakdown** (Pareto chart)
- **Resolution rate per týden**
- **Cost of reklamací** (return shipping + product replacement)
- **Repeat reklamace per zákazník** (chronic returners)

**Alerts:**
- Reklamace > 30 dnů (B2C) → 🔴 IHNED vyřídit (legal risk)
- Top SKU 5+ reklamací/měs → defektní šarže

---

### 🔄 Section 13 — VRATKY

**Hero KPIs (5):**
1. Vratky rate (% objednávek)
2. Hodnota vratek Kč (last 30 days)
3. Avg time to refund (dnů)
4. Top 5 důvody vratek
5. **Per-kategorie return rate** (Pareto chart)

**Supporting (5):**
- Per-pobočka return rate
- **Return cost per kategorie** (logistics + restock)
- **Net Revenue After Returns**
- **Customer lifetime returns** (chronic returners flag)
- **Return reason → product description gap** (zlepšení popisů)

---

### 📊 Section 14 — CROSS-CHANNEL HOLISTIC

**Top-level metrics agregující vše:**

1. **MER (Marketing Efficiency Ratio)** = Total ERP revenue / Total Marketing Spend (cíl > 5×)
2. **CAC blended** = Total Spend / New Customers
3. **LTV blended** (predicted, ne historický)
4. **LTV : CAC ratio** (cíl ≥ 3 : 1)
5. **Payback Period** (kolik měsíců než CAC zaplatí)
6. **Channel Mix %**: Direct/Organic / PPC Google / PPC Sklik / Srovnávače / Alza
7. **Net Revenue Retention** (existing customers YoY)
8. **First-touch attribution** vs **Last-click** comparison

---

### 🏪 Section 15 — GOOGLE BUSINESS PROFILE (17 prodejen)

**Zdroj: Google Business Profile API (Performance + Reviews + Posts)**
**Auth: stejný OAuth2 jako Google Ads (existující projekt hbgroup-493608) + GBP scope**

**Hero KPIs (6):**
1. **Total impressions** (search + maps) — všechny pobočky agregovaně
2. **Direct vs Discovery searches** ratio (brand vs „klíče Plzeň")
3. **Actions count** (volání + navigace + návštěvy webu) per den
4. **Avg rating** napříč všemi pobočkami (cíl ≥ 4,5★)
5. **Reviews count last 30d** (nové) + response rate (cíl > 90 %)
6. **Photos count + views** (uploaded vs viewed)

**Supporting per-pobočka (10) — tabulka 17 řádků:**
- Pobočka name + adresa
- Impressions (Maps + Search) per týden
- **Searches breakdown**: brand search vs categorical search vs near-me
- Calls (clicked phone)
- Direction requests (clicked navigation)
- Website clicks (z GBP profile na web)
- **Conversion rate** = actions / impressions
- Avg rating + reviews count + last review date
- **Q&A pending** (otázky bez odpovědi > 7d)
- **Posts last 30d** (akce, novinky, COVID, …)

**Photos & Content (4):**
- Photos uploaded last 30d per pobočka
- Photo views vs uploaded ratio (engagement)
- **Customer photos** vs business-owned (důvěra)
- Cover photo update date (cíl < 6 měsíců)

**Reviews intelligence (5):**
- Sentiment distribution (5★ / 4★ / 3★ / 2★ / 1★)
- **Top keywords v reviews** (NLP — co zákazníci zmiňují: „rychlost", „personál", „cena")
- **Negative review response time** (P50 a P90)
- Reviewer types (locals vs visitors)
- **Competitor benchmark** (avg rating konkurence v 5km radiusu)

**Local SERP (3):**
- **Maps Pack ranking** per pobočka pro top 5 keywords (např. „klíče Praha 4", „cylindrické vložky Brno")
- **Share of Local Voice** vs konkurence
- **Local Service Ads visibility** (pokud aktivováno)

**Alerts:**
- Negative review (1-2★) → 24h SLA pro response
- Q&A pending > 7d → notif majiteli
- Pobočka mimo Maps Pack TOP 3 pro home keyword → opportunity
- Hours diff (GBP vs realita) → flagovat
- Pobočka bez nových fotek > 90d → reminder
- Avg rating drop > 0,2★ MoM → varování

**Cross-section linking:**
- → Section 9 (Pobočky): per-pobočka tržby vs GBP impressions = lokální MER
- → Section 1 (Kampaně): LOCAL kampaň aktivní pro pobočku?
- → Section 12 (Reklamace): negative review correlation s reklamacemi

---

### 🏢 Section 16 — FIRMY.CZ (Seznam local listings)

**Zdroj: Firmy.cz Partner API (Seznam) NEBO scraping vlastního profilu**
**Auth: Seznam Partner login — klient zařídí; pokud API není dostupné, agentura připraví scraper s respektem k robots.txt**

**Hero KPIs (5):**
1. **Total profile views** per den (všechny pobočky)
2. **Profile rank** v top 5 výsledcích Seznam.cz pro „klíče" + 16 lokalit
3. **Click-to-call rate** (volání z Firmy.cz profilu)
4. **Avg rating** (Seznam reviews jsou separátní od Google!)
5. **Reviews count + response rate**

**Supporting per-pobočka (8) — tabulka 17 řádků:**
- Pobočka URL na Firmy.cz
- **Pozice v search results** pro „klíče {město}" / „cylindrická vložka {město}"
- Profile views per týden
- Click-to-call count
- Click-to-website count
- Direction requests (mapy.cz redirect)
- **Sponsored listing status** (Premium / Plus / Free)
- Avg rating + count

**Listing quality (5):**
- **Profile completeness %** (foto, popis, otevírací doba, akce, video, ceník)
- **Categories tagged** (kolik kategorií = víc viditelnosti)
- **Last update date** (Seznam preferuje čerstvé profily)
- **Akce/slevy publish count** last 30d
- **Q&A** count + response rate

**Reviews & Sentiment (4):**
- Reviews count split (5★ / 4★ / 3★ / 2★ / 1★)
- **Negative review response time**
- **Top mentioned keywords** v Seznam reviews (NLP)
- **Cross-platform consistency** (rating Firmy.cz vs Google — divergence = bot/fake review signal)

**Sponsored vs Organic (3):**
- Pokud klient platí Premium/Plus listing → cost vs profile views
- **Effective CPV** (Cost per profile View)
- **Sponsored conv rate** vs organic

**Alerts:**
- Profile rank drop > 3 pozice → opportunity
- Negative review (1-2★) → 24h SLA
- Profile completeness < 70 % → action item
- Sponsored listing expirace < 14d → renewal reminder

**Cross-section linking:**
- → Section 9 (Pobočky): Firmy.cz views vs GBP impressions vs walk-in (kompletní lokální funnel)
- → Section 2 (SEO): Seznam search vs Google search performance gap

---

### 🛍️ Section 17 — GOOGLE MERCHANT CENTER DEEP-DIVE

> **Pozn.:** Section 6 obsahuje high-level přehled feedů včetně MC. Tato sekce je **deep-dive** dedicated pro MC — pokud agentura uzná, může být součástí Section 6 jako sub-tab místo samostatné sekce.

**Zdroj: Google Content API for Shopping + Merchant Center Reports API**
**Auth: stejný OAuth2 + projekt hbgroup-493608 + Merchant Center scope**

**Hero KPIs (5):**
1. **Account status** (Active / Warning / Suspended)
2. **Total products submitted** vs **active in Shopping**
3. **Disapproval rate %** + change WoW
4. **Click share** (jaký % Shopping ad clicks v ČR pro naše kategorie máme)
5. **Price competitiveness benchmark** (jsme dražší / stejní / levnější vs trh)

**Issues management (8):**
- **Issues by severity**: critical (blocking) / warning / suggestion
- **Top 10 issues by affected products** (např. „missing GTIN" 320 produktů, „image too small" 156)
- **Issues by category** (cylindrické vložky, dveřní kování, trezory, …)
- **Resolution time** (avg hours od detection → fix)
- **Recurring issues** (stejné issue 2× měsíčně = systémový problém v Mergado/feed)
- **Account-level warnings** (shipping, tax, return policy missing)
- **Policy violations history** (pro detekci risk of suspension)
- **Crowd-out report** (které produkty jsou v Shopping přebité konkurencí)

**Performance per product (8):**
- **Top 50 SKU by Shopping clicks** (přímo z Performance Report)
- **Top 50 SKU by Shopping conversions**
- **Worst 50 SKU by impression-share lost** (kvůli rozpočtu / rank)
- **Disapproved bestsellers** (top 20 ERP × disapproved = high-impact fix)
- **CTR per category** (které kategorie performují v Shopping)
- **Avg CPC per category** (kde Shopping leverage)
- **ROAS per SKU** (Shopping campaign attribution)
- **Custom_label_1 (PMax) profitability cohort** — high/medium/low margin breakdown

**Best Sellers & Trends (5):**
- **Google MC Best Sellers report** (top kategorie v ČR — kde můžeme zaútočit)
- **Trending products** (rychle rostoucí queries v naší kategorii)
- **Product gap analysis** (top sellers v kategorii, které my nemáme)
- **Brand performance** (jaké brandy v Shopping vyhráváme)
- **Promotions performance** (slevy/akce attached k produktům)

**Price Competitiveness (4):**
- **Price benchmark per SKU** (naše cena vs avg trhu)
- **Price drop opportunities** (kde malou změnou ceny získáme rank)
- **Above-market products** (>15 % nad trhem = nikdo nekoupí)
- **Below-market unjustified** (jsme zbytečně levní = ztráta marže)

**Alerts:**
- Account suspension → P0 (Shopping ads zastaveny okamžitě)
- Disapproval rate > 5 % → P1
- Bestseller (TOP 20 ERP) disapproved → P0
- New issue type detected → notif (může být systémový bug)
- Price > 20 % nad market → margin opportunity warning
- Issues unresolved > 7d → escalation

**Cross-section linking:**
- → Section 1 (Kampaně): Shopping campaign performance odráží MC health
- → Section 6 (Feedy): vstupní data před MC validation
- → Section 7 (Produkty): per-SKU performance konsolidace

---

## 4. UI/UX requirements

### Layout pattern
```
┌────────────────────────────────────────────────────────────────┐
│ HEADER: 🎯 Marketing Dashboard · ⚠️ Last update 6:00          │
│                                              🔓 Logout         │
├────────────────────────────────────────────────────────────────┤
│ 📅 DATE RANGE PICKER (globální) | ⚖️ Compare to: Previous | YoY │
├────────────────────────────────────────────────────────────────┤
│ EXECUTIVE SUMMARY (6 hero cards) — reaguje na date filter     │
├────────────────────────────────────────────────────────────────┤
│ ALERTS BAR (red banner pokud kritické)                        │
├────────────────────────────────────────────────────────────────┤
│ TABS: 18 sekcí (kampaně, SEO, B2C, B2B, srovnávače, ..., GBP, Firmy, MC) │
├────────────────────────────────────────────────────────────────┤
│ ACTIVE SECTION (hero KPIs + supporting + charts + tabulky)    │
└────────────────────────────────────────────────────────────────┘
```

### 📅 Date range filter — POVINNÁ funkcionalita

**Globální date picker** musí být **vždy viditelný v hlavičce** dashboardu a aplikovat se na **VŠECHNY sekce + KPIs + grafy + tabulky najednou**.

**1. Quick presets** (1-click buttons):
- **Dnes** (jen pro úplnost — většina dat má 1 den delay)
- **Včera**
- **Posledních 7 dní** (default při otevření)
- **Posledních 14 dní**
- **Posledních 28 dní** (matchuje GA4 default)
- **Posledních 30 dní**
- **Posledních 90 dní**
- **Tento měsíc (MTD)** — Month-to-Date od 1. dne
- **Minulý měsíc** (celý uzavřený měsíc)
- **Tento kvartál (QTD)**
- **Tento rok (YTD)**
- **Posledních 12 měsíců**

**2. Custom date range picker**:
- Dva date inputs: **od** / **do**
- Kalendář popup s českou lokalizací (PO-NE, leden-prosinec)
- **Min datum**: 1.10.2025 (start ERP dat — Helios Výdejky)
- **Max datum**: včerejšek (data nejsou k dispozici real-time)
- Validace: do ≥ od, max 2 roky rozsah

**3. Comparison mode** (vedle date pickeru):
- **Žádné srovnání** (default)
- **vs Předchozí období** — automaticky stejně dlouhé období hned před (např. 7d → předchozí 7d)
- **vs Stejné období minulý rok** (YoY)
- **vs Custom** — uživatel vybere srovnávací rozsah ručně
- Comparison se projeví jako:
  - U KPI: současná hodnota + delta % vs comparison + 🟢/🔴 indikátor
  - U grafů: dvě překryvné křivky (current solid, comparison dashed)
  - U tabulek: sloupec „vs předchozí" s % změnou

**4. Per-section overrides** (advanced):
Některé sekce mají vlastní logiku — agentura může (po dohodě) udělat **section-level override**:
- **Section 5 (Srovnávače)**: může potřebovat denní granularitu i pro 90d view
- **Section 15 (GBP)**: GBP Performance API vrací data pouze **last 6 months max** — UI musí to indikovat („Data dostupná od {date}")
- **Section 17 (MC)**: Merchant Reports API max **3 měsíce** historie
- **Section 13 (Cohorts)**: měsíční granularita (ne denní), zobrazuje cohorts od měsíce nákupu

Pokud zvolený date range obsahuje **období bez dostupných dat**, sekce zobrazí **prázdný state s vysvětlením** (např. „GBP data dostupná od 12.11.2025 — vybraný rozsah obsahuje 60d bez dat").

**5. Granularity selector** (denní / týdenní / měsíční):
- Auto-detect podle date rangu:
  - ≤ 31 dní → **denní** granularita (default)
  - 32-180 dní → **týdenní** (sloupcové grafy = celý týden)
  - > 180 dní → **měsíční**
- Uživatel může override (přepnout manuálně)
- Týdenní view začíná **pondělím** (ISO week)

**6. URL state (shareable links)**:
Date filter musí být **uložen v URL query parameters** aby šel poslat odkaz kolegovi:
```
audit.klicovecentrum.cz/marketing/?from=2026-04-01&to=2026-04-30&compare=previous&section=campaigns
```
Při otevření této URL se dashboard otevře s přednastaveným filterem + sekcí.

**7. Sticky behavior**:
- Datum se **persistuje napříč tabs** (přepnu Kampaně → Pobočky a datum zůstane)
- Datum se **persistuje v localStorage** (refresh stránky zachová stav)
- Při novém openu dashboardu (nový login session) se resetuje na **Posledních 7 dní**

**8. Loading states**:
- Při změně datumu: skeleton placeholdery v každé sekci po dobu načítání nových dat
- Spinner v header („Načítám data pro 1.4.2026 — 30.4.2026…")
- Pokud data fetching > 5s: progress indicator
- Pokud API selže: error state s retry buttonem (data zůstanou cached)

**9. Data freshness indicator**:
V hlavičce vedle date pickeru:
- 🟢 **„Data čerstvá (last refresh 6:00, před 4h)"**
- 🟡 **„Data starší (last refresh 18h)"** — pokud cron neproběhl
- 🔴 **„Data zastaralá > 25h"** — kritický alert
- ⏰ **„Next refresh: zítra 6:00"**

**10. Empty states**:
- Pokud vybraný range nemá data: „Žádná data pro toto období"
- Pokud sekce vůbec data nemá (např. Section 15 před GBP setupem): „GBP integrace probíhá — data budou k dispozici po dokončení auth setupu"

### UI principles
- ✅ Trafik signal everywhere: 🟢 / 🟡 / 🔴
- ✅ MoM/WoW comparison vedle každé KPI
- ✅ Drill-down na klik (KPI → detail tabulka)
- ✅ **Date range picker globální + per-section overrides** (viz výše)
- ✅ **Granularity auto-switching** (denní / týdenní / měsíční)
- ✅ **Comparison mode** (vs předchozí / YoY / custom)
- ✅ **URL state sharable** (date + section v query params)
- ✅ Export button: CSV / PNG snapshot (zachovává aktuální date filter)
- ✅ Mobile responsive (date picker funguje na mobilu — native HTML5 inputs)
- ✅ Dark theme (jako existující audit dashboard, pro brand consistency)
- ✅ Keyboard shortcuts: `T` = dnes, `W` = 7d, `M` = 30d, `Y` = YTD

### Branding
- Existing CSS palette dashboardu: `#0f172a` background, `#3b82f6` primary, `#10b981` positive, `#fbbf24` warning, `#ef4444` critical
- Logo H&B Group / klicovecentrum.cz (klient pošle SVG)

---

## 5. Data sources mapping — co existuje vs co klient musí připravit

### ✅ Existující data (audit dashboardu, dostupné teď)
| Zdroj | Formát | Refresh |
|-------|--------|---------|
| GA4 funnel, sessions | API | Live (Service Account) |
| Search Console clicks/queries | API | Live |
| Bing Webmaster crawl + queries | API | Live |
| Sklik Fénix kampaně | API | Live (refresh token) |
| MySQL kosik (4 weby, web_id 1-4) | DB | Live |
| ERP Výdejky MO + VO 10/2025-5/2026 | CSV | Manuální upload |
| Helios Reklamace | CSV | Manuální upload |
| PPL dopravce | CSV | Manuální upload |
| Top customers SQL | derived JSON | Daily rebuild |
| Mergado feed | URL feed | Live |

### ❌ Chybí — klient musí připravit pro plnou funkčnost
| Zdroj | Co potřebujete od klienta | Priorita |
|-------|---------------------------|----------|
| **Per-pobočka tržby z Helios** | Helios IQ → modul „Maloobchod pokladna" → per-provozovna export | P1 (pro Section 9) |
| **Google Ads denní export** | Klient nastaví Google Ads Scripts (denně push do GitHub) | P0 (pro Section 0, 1) |
| **Google Business Profile API access** | Cloud Console → Performance API + Business Information API + Reviews API enable + OAuth scope `https://www.googleapis.com/auth/business.manage` na existující projekt hbgroup-493608. Klient přidá agenturní email jako manager všech 17 GBP profilů. | **P0 (pro Section 15)** |
| **Google Merchant Center API access** | Content API for Shopping + Merchant Reports API enable na hbgroup-493608. OAuth scope `https://www.googleapis.com/auth/content`. Klient přidá agenturu jako Standard user v MC. | **P0 (pro Section 17 + rozšíření Section 6)** |
| **Firmy.cz Partner API access** | Klient kontaktuje partner@seznam.cz a požádá o API access (může vyžadovat smlouvu). Pokud API odmítnuto, agentura připraví scraper s respektem k robots.txt. | **P1 (pro Section 16)** |
| **Seznam.cz CHKAR / Sklik partner login** | Pro Sponsored Firmy.cz listings cost data — login do Sklik nebo separátně. | P2 (pro Section 16 Sponsored) |
| **Alza Partner orders denní** | Alza Seller Portal → CSV export (nebo API integrace) | P1 (pro Section 5) |
| **Heuréka OCM denní** | Klient nastaví OCM API push (existuje) | P2 |
| **Zboží.cz OCM denní** | Klient nastaví OCM API push | P2 |
| **Mailchimp/Ecomail API** | Pokud klient používá email kampaně | P2 (volitelné) |
| **Meta Ads API** | Brand kampaně Plzeň/Brno/Chomutov | P1 (pro Section 1) |
| **NPS data** | Heureka Ověřeno + post-purchase survey | P3 (nice-to-have) |

### 🔑 Auth setup pro 3 nové zdroje (Google Business Profile / Google Merchant Center / Firmy.cz)

**Společný OAuth projekt**: `hbgroup-493608` (existuje, používá ho Google Ads + GA4)

**Krok 1 — Google Cloud Console enable APIs:**
```
1. Console.cloud.google.com → projekt hbgroup-493608
2. APIs & Services → Library
3. Enable:
   - "My Business Account Management API"
   - "My Business Business Information API"
   - "My Business Performance API"
   - "My Business Q&A API"
   - "My Business Reviews API"
   - "Content API for Shopping"
   - "Merchant Reports API"
4. OAuth consent screen → add scopes:
   - https://www.googleapis.com/auth/business.manage
   - https://www.googleapis.com/auth/content
```

**Krok 2 — GBP access:**
```
1. business.google.com → Settings → Managers
2. Add agentura email jako "Manager" pro každou ze 17 prodejen
   (NEBO: pokud má klient Business Group, přidat na úrovni groupu = 1 kliknutí)
```

**Krok 3 — Merchant Center access:**
```
1. merchants.google.com → Users
2. Add agentura email jako "Standard" user
   (musí být ne "Admin" — to nepovoluje audit dashboardu API access)
```

**Krok 4 — Firmy.cz:**
```
1. Klient pošle email na partner@seznam.cz s žádostí o API access
2. Pokud API odmítnuto:
   - Agentura připraví web scraper pro public profile URL
   - Frekvence: 1× denně (respekt k Seznam serverům)
   - Cíl: extract impressions, position, reviews, photos count
```

---

## 6. Implementation phases & timeline

### Phase 1 — Foundation (Week 1)
- [ ] Setup repo + GitHub Pages + Cloudflare Access
- [ ] Backend cron infrastructure + alert email SMTP
- [ ] Frontend scaffold (header + tabs + executive summary)
- [ ] **Global date range picker komponenta** (12 presets + custom range + comparison mode + URL state + localStorage persist)
- [ ] **Granularity auto-switching logic** (denní/týdenní/měsíční podle date rangu)
- [ ] **Data layer wrapper** který každé sekci poskytne data filtrovaná dle aktuálního date rangu (single source of truth pro datum)
- [ ] Acceptance: dashboard otevíratelný na `audit.klicovecentrum.cz/marketing/` s prázdnými sekcemi + funkčním date pickerem (placeholder data zachovává date range při refreshi)

### Phase 2 — Top 3 sekce (Week 2)
- [ ] Section 0 (Executive Summary) — fully funkční **s reakcí na date filter**
- [ ] Section 1 (Kampaně) — fully funkční **s reakcí na date filter + comparison mode**
- [ ] Section 3 (B2C) — fully funkční **s reakcí na date filter**
- [ ] Acceptance: 3 sekce s real ERP data, date filter funguje napříč všemi, alerts fungují, refresh cron běží

### Phase 3 — Zbývajících 15 sekcí (Week 3-5)
- [ ] Section 2, 4-14 — postupně dle priority, **každá reaguje na date filter**
- [ ] Section 15 (GBP), 16 (Firmy.cz), 17 (MC) — 3 nové sekce + per-section override (data dostupnost indikace)
- [ ] Drill-down + filters + export (export respektuje aktuální date range)
- [ ] Mobile responsive testing (vč. date pickeru na mobilu)

### Phase 4 — Production polish + handoff (Week 6-7)
- [ ] Windows Task Scheduler nastavení / GitHub Actions cron
- [ ] Email alert SMTP fully configured
- [ ] Per-user Cloudflare Access policy
- [ ] **Performance tuning**: date filter change re-render < 1s (cached aggregations)
- [ ] **URL state testing**: shareable links fungují
- [ ] Documentation pro klienta jak interpretovat KPIs + jak používat date filter
- [ ] Training session (1h call)

---

## 7. Acceptance criteria

✅ Marketing dashboard funkční na `audit.klicovecentrum.cz/marketing/` s Cloudflare Access auth
✅ Všech 18 sekcí implementováno s minimálně hero KPIs (supporting může být fáze 2)
✅ **Date range picker funkční** — 12 quick presets + custom range + comparison mode + URL state
✅ **Date filter aplikuje se na všech 18 sekcí konsistentně** (test scenario: vyber „minulý měsíc" → všechny KPIs reagují)
✅ **Comparison mode**: každé KPI ukazuje delta % vs předchozí období (nebo YoY)
✅ **Granularity auto-switching**: denní (≤31d) → týdenní (32-180d) → měsíční (>180d)
✅ **URL sharable links** fungují (test: zkopírovat URL s ?from=...&to=... → otevřít v jiném browseru → stejný state)
✅ Daily 6:00 refresh probíhá automatically, last_update timestamp viditelný v UI
✅ Email alert na `oskrebsky@gmail.com` při CSV stale > 25h FUNGUJE (test scenario)
✅ Mobile responsive (testováno na iPhone + Android)
✅ **Date picker funguje na mobilu** (native HTML5 inputs nebo touch-optimized library)
✅ Page load < 3 sekundy na 4G
✅ **Date filter change re-render < 1 sekunda** (cached JSON, ne re-fetch)
✅ Loaded JSON files celkem < 5 MB (lazy load podle sekce)
✅ Cross-browser: Chrome, Firefox, Safari, Edge
✅ Dokumentace: README.md s setup + údržba postupy

---

## 8. Out of scope (NEZAHRNUTO v této fázi)

- Real-time data (refresh < 1h) — daily je dostatečné
- Machine learning predictions (LTV forecast, churn prediction) — fáze 2
- A/B testing platform — fáze 2
- Custom report builder (klient si tvoří vlastní views) — fáze 2
- Multi-language UI (pouze čeština)
- Slack / Teams integrace (jen email alerts)

---

## 9. Předpokládaný rozpočet a timeline

- **Estimovaná pracnost**: 95-140 hodin (původně 80-120, +15-20h za 3 nové sekce)
  - Foundation + auth: 12-16h
  - 18 sekcí × ~5h: 90h (15 puvodnich + 3 nove: GBP, Firmy.cz, Google Merchant Center deep-dive)
  - Backend cron + alerts: 10-14h (více API integrací)
  - Production setup + testing: 10-15h
  - Documentation + training: 5-8h
- **Timeline**: 5-7 týdnů od podpisu zadání (původně 4-6, +1 týden za 3 nové data zdroje)
- **Platba**: dle dohody (1/3 záloha, 1/3 mid-project, 1/3 po acceptance)

---

## 10. Klient poskytne

- ✅ Cloudflare Access (existuje)
- ✅ GitHub repo access (kc-hbgroupcz organization)
- ✅ Google Service Account JSON (hbg-101@hbgroup-493608.iam.gserviceaccount.com)
- ✅ Sklik Fénix refresh token
- ✅ Bing Webmaster API key
- ✅ MySQL kosik DB access (XAMPP credentials)
- ⏳ Helios IQ per-provozovna export (klient nastaví — viz sekce 5)
- ⏳ Google Ads Scripts nastavený (klient zajistí denní export)
- ⏳ Alza Partner API credentials (klient generuje)
- ⏳ Logo SVG + brand kit

---

## 11. Kontakt

**Klient (vedení + technical owner):**
Ondřej Skřebský
oskrebsky@gmail.com

**Příklad existujícího dashboardu (pro reference designu):**
https://audit.klicovecentrum.cz (po Cloudflare Access login)
Heslo přístupu: na vyžádání

**Repository:**
https://github.com/kc-hbgroupcz/klicovecentrum-audit (existující audit + bude submodul marketing-dashboard nebo subfolder)

**Memory + technical reference:**
`memory/audit_key_numbers.md` — všechny audit metriky
`memory/hbgroup_sql_db.md` — SQL DB schema
`memory/bot_attack_2026_3_waves.md` — bot detekce kontext
`memory/cloudflare_klicovecentrum_setup.md` — auth setup

---

## 12. Otevřené otázky pro agenturu

Před tím než přijmete projekt, prosím odpovězte:

1. Máte zkušenosti s **GitHub Pages + Cloudflare Access** auth flow?
2. Preferujete **React/Vue** nebo **vanilla JS** (kvůli consistency s existujícím audit dashboardem)?
3. Jak řešíte **mobile responsive** dashboardy se 15+ sekcemi? (sidebar / drawer / tabs?)
4. **Real-time** vs **daily** preferujete pro váš tech stack?
5. Máte preferred SMTP service pro alerts (Gmail / SendGrid / Mailgun)?
6. Můžete poskytnout **2-3 reference** podobných marketing dashboardů?

---

**Tento brief je závazný od podpisu obou stran. Změny rozsahu v rámci projektu budou účtovány zvlášť dle hodinové sazby.**
