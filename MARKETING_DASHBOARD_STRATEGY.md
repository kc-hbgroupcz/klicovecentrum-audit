# 🎯 Marketing Dashboard — Strategy & KPI Design (12. 5. 2026)

**Cíl:** Live operační dashboard pro CMO / marketing manažera klicovecentrum.cz + hbgroup.cz, daily refresh z CSV, alerts.

---

## 1. Strategická úvaha — pro koho a proč?

### Audience
- **Primární**: Klient (CMO / vedení H&B Group) — denní 30-sec přehled, weekly deep dive
- **Sekundární**: Marketingová agentura — operativní rozhodnutí o spend allocation
- **Terciární**: CEO / účetní — měsíční výsledkové reporty

### Účel (z čeho rozhodují)
1. **Allocate budget**: Kam dát další korunu? (kampaň/kanál/produkt)
2. **Detect anomálie**: Bot útok, payment outage, propad ROAS, technical issue
3. **Monitor health**: Jsou KPI v normálu? (alerts = výjimky)
4. **Verify hypotézy**: Funguje SEO Sprint 1? Reaktivace LOCAL kampaní?

### Anti-pattern (čeho se vyvarovat)
- ❌ Vanity metrics (impressions, sessions samotné — bot ukázal že jsou nespolehlivé)
- ❌ Tabulky bez insightů (= klient se musí dohadovat co je dobře/špatně)
- ❌ Real-time všeho (overkill — daily je víc než dost pro většinu rozhodnutí)
- ❌ GA4 jako primární zdroj (kontaminované bot daty 8 týdnů)

---

## 2. Architecture (live + 24h refresh + alerts)

### Tech stack
```
┌─────────────────────────────────────────────────────────────┐
│ KLIENT — OneDrive folder s CSV (manual upload denně)       │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      │ (auto sync via OneDrive)
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ LOKÁLNÍ PC (Windows Task Scheduler — každý den 6:00 ráno)  │
│  • Python build scripts → JSON                              │
│  • Pokud CSV starší než 25h → ALERT                         │
│  • git commit + push                                        │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      │ (GitHub PAT)
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ GITHUB PAGES → Cloudflare Access                            │
│ marketing.klicovecentrum.cz/dashboard.html                  │
│  • Polling /data/_meta.json každých 5 min                   │
│  • Pokud last_update > 25h → red banner v UI                │
└─────────────────────────────────────────────────────────────┘
```

### Alert systém (3 vrstvy)
1. **Klient sám vidí v UI** — červený banner "⚠️ Data stará 30h, klient nevyaktualizoval CSV"
2. **Email alert** — Python script při fail pošle email na `oskrebsky@gmail.com` přes SMTP (Gmail App Password)
3. **GitHub Actions notification** — pokud cron fail, GitHub posílá auto email

### Live data (z API — nečeká na CSV)
- **GA4** — daily sessions/transactions (Service Account, auto)
- **Search Console** — organic clicks/impressions (Service Account, auto)
- **Bing Webmaster** — crawl stats (API key, auto)
- **Sklik Fénix** — kampaně data (refresh token, auto)

### CSV-only data (vyžaduje klient)
- **ERP MO+VO Výdejky** — Cézar export (manuální 1× měsíc)
- **Reklamace** — Cézar export (manuální)
- **PPL zásilky** — PPL export
- **Google Ads** — Ads Scripts (klient spustí v Google Ads UI)

### Update strategie
- **Live data**: Refresh každé ráno (6:00) přes Python pull
- **CSV data**: Detekce nového souboru v OneDrive folder → auto rebuild
- **Alert thresholds**:
  - CSV stale > 25h → ⚠️ warning banner
  - CSV stale > 7 dní → 🔴 critical alert + email
  - Live API fail → 🟡 sekce zobrazí "Data unavailable"

---

## 3. KPI návrh per sekce (senior marketing expert pohled)

### 🎯 SECTION 0 — Executive Summary (top of dashboard)

**6 hero cards** (hlavní KPIs, viditelné na první pohled):

| KPI | Current | Trend (vs minulý týden) | Alert threshold |
|-----|---------|------------------------|------------------|
| **Daily revenue** (B2C+B2B+Alza) | XX k Kč | ↑/↓ % | < 80 % avg = warning |
| **Marketing spend / day** | XX k Kč | trend | > 120 % budget = warning |
| **Blended ROAS** (real, ne GA4) | X,XX× | trend | < 3,0× = warning |
| **Real Conv Rate** (SQL/sessions) | X,X % | trend | < 0,5 % = warning |
| **Active campaigns count** | XX | new vs paused | žádné LOCAL = warning |
| **Open reklamace** | XX | new this week | > 100 = warning |

---

### 📢 SECTION 1 — KAMPANĚ (PPC — Google Ads + Sklik + Heuréka + Zboží)

**Hero KPIs (5):**
1. **ROAS realný** (na základě SQL ground truth, ne GA4 inflated) — per channel
2. **Spend share** % per channel (Google Ads / Sklik / Heuréka / Zboží)
3. **CPA real** (Cost / Real conversions z SQL)
4. **Impression Share** (kde ztrácíme exposure)
5. **Quality Score avg** per channel

**Per-kategorie split (PRODUCT / BRAND / LOCAL / RTG):**
- Spend %, ROAS, conversions per kategorie
- Identifikace „zlatých žil" (např. Sklik 0 VG Brand 23,89× ROAS)
- LOCAL kampaně per pobočka — ROAS + store visit metrics

**Supporting (8):**
- CTR per kampaň (top 10)
- Avg CPC trend
- Conv rate per kampaň
- Search lost IS budget vs rank
- Negative keywords coverage
- Customer Match audience size + activation
- Top 10 search queries (z SC + Sklik)
- 🆕 **MER (Marketing Efficiency Ratio)** = Total Revenue / Total Marketing Spend

**Alert triggers:**
- Spend > 120 % budget → warning
- ROAS < 2,0 → warning
- IS lost budget > 30 % → optimization opportunity
- Quality Score < 6 → bidding inefficiency

---

### 🔍 SECTION 2 — SEO

**Hero KPIs (5):**
1. **Organic clicks** (90-day rolling, brand vs non-brand)
2. **Organic CTR** (target 1,5 % — current 0,9 %)
3. **Avg position** top 50 queries
4. **Indexed URLs** (GSC + Bing — sledovat URL exploze)
5. **AI Citability score** (GEO measurement — kolik AI Overviews/ChatGPT odpovědí cituje brand)

**Supporting (8):**
- Brand vs non-brand split (40/60 cíl)
- Top 10 stránky CWV (LCP/CLS/INP)
- Crawl budget (Bing/GSC)
- Sitemap warnings count (cíl < 1 200, current 6 119)
- 🆕 **New ranking keywords / week** (z GSC API)
- 🆕 **Lost ranking keywords / week** (= konkurence překonává)
- Backlinks count (Bing inbound links + Common Crawl)
- 🆕 **Branded search trend** (Google Trends API: "klicove centrum" + variants)

**Alert triggers:**
- Daily clicks pokles > 30 % → warning (možný útok / penalty)
- New 4xx pages v sitemap → fix ASAP
- AI Citability score < 30 → AI search nedostupný

---

### 🛒 SECTION 3 — B2C E-shop (klicovecentrum.cz)

**Hero KPIs (5):**
1. **Daily revenue** (SQL ground truth, ne GA4)
2. **Daily orders** (real)
3. **AOV** (avg order value)
4. **Real Conv Rate** = SQL orders / GA4 sessions (target 2-3 %)
5. **Repeat customer rate** (% objednávek od existujících)

**Supporting (8):**
- New vs Returning customer ratio
- 🆕 **Cart Abandonment funnel**: add_to_cart → checkout → purchase (se výškou propad)
- Time-to-Purchase (od first session po conversion)
- Top 20 SKU (rolling 30 days)
- Top 10 kategorií revenue
- Anonymous (guest checkout) vs registered ratio
- 🆕 **Cohort retention table** (kupci z měsíce X, kolik koupilo X+1, X+3)
- 🆕 **Mobile vs Desktop conversion rate split**

**Alert triggers:**
- Daily orders < 50 % avg → 🔴 critical (payment outage signal)
- Cart-to-checkout < 50 % → funnel issue
- AOV pokles > 20 % → product mix changed?

---

### 🏢 SECTION 4 — B2B (hbgroup.cz)

**Hero KPIs (5):**
1. **Daily B2B revenue** (web_id=2)
2. **Top 10 partneři % share** (cíl < 60 % = diverzifikace)
3. **Avg B2B order value** (currentl ~5 800 Kč s DPH)
4. **B2B margin** (cíl 40-45 %)
5. **Active partners (last 30 days)** count

**Supporting (8):**
- New B2B accounts / měsíc
- 🆕 **B2B Churn**: partneři kteří nenakoupili 60+ dní (red flag)
- 🆕 **B2B Net Revenue Retention** (NRR) — meziroční změna obratu existujících partnerů
- B2B vs B2C revenue ratio (currentl 9,93M vs 4,11M = 2,4× B2B)
- Top partneři: sales velocity (kolik objednávek/měs)
- 🆕 **Partner Concentration risk**: pokud top 1 partner > 10 % obratu = risk
- 🆕 **B2B repeat order interval** (avg dnů mezi objednávkami per partner)
- Geographic B2B distribution

**Alert triggers:**
- Top 10 partneři % > 70 % → over-concentration
- Active partners pokles > 20 % MoM → churn problem
- 🆕 **Specific partner alert**: pokud TOP 5 partner nenakoupil > 30 dní → kontaktovat

---

### ⚖️ SECTION 5 — SROVNÁVAČE (Heuréka + Zboží.cz)

**Hero KPIs (5):**
1. **Heuréka ROAS** (real, ne reportovaný)
2. **Zboží.cz ROAS**
3. **Cost share** (% z total marketing spend)
4. **Conv rate per srovnávač**
5. **Rating + reviews count** (trust signal)

**Supporting (6):**
- Top performing SKU per srovnávač
- Per-srovnávač product visibility (impression share)
- Bidding diagnostika (Sklik „možno vylepšit" %)
- 🆕 **Review velocity** (new reviews / měsíc)
- 🆕 **Compare price competitiveness** (jsme nejlevnější / průměr / nejdražší?)
- Zboží.cz traffic trend (current −75 % YoY = krise)

**Alert triggers:**
- ROAS < 2,0 → unprofitable spend
- Reviews growth = 0 → reputation stagnation
- Impression share < 5 % → bidding issue

---

### 📦 SECTION 6 — FEEDY (Mergado)

**Hero KPIs (5):**
1. **Total active products** in feeds
2. **Disapproved count** (Merchant Center, current 1 338)
3. **Last sync timestamp** per feed
4. **Coverage %**: bestsellers ve feedu vs ERP TOP 50
5. **Feed quality score** (Sklik diagnostika, MC issues)

**Supporting (6):**
- Per-feed health (Google MC, Heuréka, Zboží)
- Variant coverage (současné 0 % → cíl 100 %)
- Bestsellers chybí ve feedu (současně 450)
- 🆕 **Feed-to-ERP price diff alerts** (cena ve feedu ≠ aktuální cena)
- 🆕 **Stock-out coverage** (kolik out-of-stock items pořád ve feedu = wasted spend)
- Custom_label_1 distribution (PMax labely)

**Alert triggers:**
- Disapproved > 50 % → critical
- Last sync > 24h → feed broken
- Bestsellers chybí ve feedu > 100 → revenue leak

---

### 🔧 SECTION 7 — PRODUKTY

**Hero KPIs (5):**
1. **Top 20 SKU revenue** (rolling 30 days)
2. **Bestsellers** (count > 10 prodejů/měs)
3. **New SKU added** / měsíc
4. **Discontinued SKU** count
5. **Per-vendor revenue** (top 10 dodavatelů)

**Supporting (6):**
- 🆕 **Stock-out frequency** per top SKU
- 🆕 **Margin distribution** (histogram 0-100 % per SKU)
- PMax label split (profitable / zombie / costly)
- 🆕 **Cross-sell affinity** (co se kupuje s top SKU)
- 🆕 **Product velocity score** (pageviews × add_to_cart × purchase rate)
- Product return rate per SKU (z reklamace data)

**Alert triggers:**
- Top 5 SKU pokles > 30 % MoM → product issue
- Stock-out > 7 dní u TOP 50 SKU → revenue lost
- New SKU = 0 / měsíc → catalog stagnation

---

### 📂 SECTION 8 — KATEGORIE

**Hero KPIs (5):**
1. **Top 10 kategorií revenue** (rolling 30 days)
2. **Per-kategorie marže** (margin %)
3. **MoM trend** per kategorie (growth / decline)
4. **Conv rate per kategorie**
5. **Cross-sell ratio** (kolik objednávek má 2+ kategorie)

**Supporting (5):**
- New entries per kategorie / měsíc
- Per-kategorie organic traffic (z SC)
- Per-kategorie PPC CPA
- 🆕 **Category lifecycle stage** (growth / mature / declining)
- 🆕 **Seasonality index** per kategorie

---

### 📍 SECTION 9 — POBOČKY (17 fyzických)

**Hero KPIs (5):**
1. **Per-pobočka tržby** (pokud Cézar export bude obsahovat per-provozovna data)
2. **GBP zobrazení / volání / navigace** per pobočka
3. **Firmy.cz CTR** per pobočka
4. **Local Google Ads ROAS** + store visit conversions
5. **Reviews + rating** per pobočka

**Supporting (6):**
- Mystery shopping score (pokud máte)
- 🆕 **Per-pobočka Maps Pack ranking** (sledovat „klíče Plzeň" SERP)
- 🆕 **Hours utilization** (peak hours vs idle)
- 🆕 **Walk-in vs phone call ratio**
- Per-pobočka local keyword rankings
- Reaktivace 13 missing LOCAL kampaní (nález #6)

**Alert triggers:**
- Pobočka bez LOCAL kampaně → opportunity miss
- Per-pobočka call rate < 0,2 % → engagement issue (nález #22)
- Negative review (1-2★) → response within 24h

---

### 🗺️ SECTION 10 — KRAJE

**Hero KPIs (5):**
1. **Top 5 krajů revenue** (ERP-aligned z PSČ mapping)
2. **Cost per kraj** (Google Ads geo split)
3. **CPA per kraj** (Cost / SQL orders)
4. **Conversion rate per kraj**
5. **Pokrytí**: kraj × pobočka × revenue

**Supporting (5):**
- New customers per kraj / měsíc
- Repeat rate per kraj
- AOV per kraj
- 🆕 **Penetration rate** (% population s nákupem v kraji — populace dat z ČSÚ)
- 🆕 **Underserved regions** (vysoký traffic + nízká conversion = chybí pobočka?)

---

### 🚚 SECTION 11 — DOPRAVA (PPL)

**Hero KPIs (5):**
1. **Daily zásilky** count (PPL volume)
2. **Avg dobírka %** (current 15 %)
3. **On-time delivery rate** (pokud PPL data poskytuje)
4. **Top 10 destinace měst**
5. **Interní vs externí ratio** (current 23/77)

**Supporting (5):**
- 🆕 **Delivery time SLA** (ordered → delivered avg dnů)
- 🆕 **Lost packages count** (claims rate)
- 🆕 **Per-recipient PPL frequency** (kdo objednává nejčastěji)
- Per-kraj PPL share
- 🆕 **PPL cost per package** (negotiation lever)

**Alert triggers:**
- PPL volume = 0 v pracovní den → carrier outage (single point of failure!)
- Dobírka > 25 % → trust issue
- On-time < 90 % → SLA breach

---

### ⚠️ SECTION 12 — REKLAMACE

**Hero KPIs (5):**
1. **Reklamační poměr** (current month %, target < 3 %)
2. **Total open reklamací** (aktuálně 86 = tikající bomba)
3. **Avg doba vyřízení** (target < 30 dní)
4. **% > 30 dnů** (regulatory risk!)
5. **Top 5 problematické produkty**

**Supporting (5):**
- Top reklamující zákazníci (B2B partneři risk!)
- 🆕 **Reasons breakdown** (Pareto chart)
- 🆕 **Resolution rate** per týden
- 🆕 **Cost of reklamací** (return shipping + product replacement)
- 🆕 **Repeat reklamace per zákazník** (chronic returners flag)

**Alert triggers:**
- Reklamace > 30 dnů (B2C) → 🔴 ObčZ § 2172 violation, ihned vyřídit
- Top SKU s 5+ reklamací/měs → defektní šarže investigation
- B2B top 10 partner reklamace > 10 % objednávek → vztah risk

---

### 🔄 SECTION 13 — VRATKY

**Hero KPIs (5):**
1. **Vratky rate** (% objednávek)
2. **Hodnota vratek Kč** (last 30 days)
3. **Avg time to refund** dní
4. **Top 5 důvody vratek**
5. **Per-kategorie return rate** (Pareto chart)

**Supporting (5):**
- Per-pobočka return rate
- 🆕 **Return cost per kategorie** (logistics + restock)
- 🆕 **Net Revenue After Returns** (= Gross - Returns)
- 🆕 **Customer lifetime returns** (chronic returners)
- 🆕 **Return reason → product description gap** (možnost zlepšit popis)

---

### 📊 SECTION 14 — CROSS-CHANNEL HOLISTIC (NEW)

**Top-level metrics agregující vše:**

1. **MER (Marketing Efficiency Ratio)** = Total Revenue / Total Marketing Spend (cíl > 5×)
2. **CAC blended** = Total Spend / New Customers
3. **LTV blended** (predicted, ne historický)
4. **LTV : CAC ratio** (cíl ≥ 3 : 1)
5. **Payback Period** (kolik měsíců než CAC se zaplatí)
6. **Channel Mix %**:
   - Direct/Organic: X %
   - PPC Google: X %
   - PPC Sklik: X %
   - Srovnávače: X %
   - Marketplaces (Alza): X %
7. **Net Revenue Retention** (existing customers YoY)
8. **First-touch attribution** vs **Last-click** comparison

---

## 4. 🆕 Doplňkové metriky které doporučuju (top 15)

| # | Metric | Proč důležité | Data source |
|---|--------|---------------|-------------|
| 1 | **MER** (Marketing Efficiency Ratio) | Single number summing all marketing | Existing data |
| 2 | **Cohort retention table** | Vidět skutečný repeat business | SQL kosik |
| 3 | **LTV : CAC ratio per kanál** | Které kanály platí, které ne | Cross |
| 4 | **B2B Churn alerts** | Top partneři offline > 60 dnů = lose | SQL kosik |
| 5 | **Stockout lost revenue** | Kolik tržeb teče vodou kvůli skladu | ERP + GA4 |
| 6 | **Mobile vs Desktop conv rate** | Mobile UX optimization priority | GA4 |
| 7 | **Site search analysis** | Co lidi hledají = chybějící kategorie | GA4 events |
| 8 | **Cart-to-checkout-to-purchase funnel** | Detect payment outages early | GA4 funnel |
| 9 | **Brand search trend** | SOV (share of voice) proxy | Google Trends + GSC |
| 10 | **Direct traffic legitimacy %** | How much direct = bot | GA4 + Bing cross |
| 11 | **Time-to-First-Purchase** | Acquisition velocity | SQL + GA4 |
| 12 | **Per-pobočka Maps Pack ranking** | Local SEO health | SerpApi/manual |
| 13 | **Email/Newsletter open + CTR** | Owned channel performance | Mailchimp/Ecomail (pokud máte) |
| 14 | **Social engagement rate** | Brand awareness | Meta API |
| 15 | **Net Promoter Score (NPS)** | Customer loyalty | Survey (Heureka Ověřeno?) |

---

## 5. Information architecture — jak to navrhnout vizuálně

### Layout pattern
```
┌────────────────────────────────────────────────────────────────┐
│ HEADER: 🎯 Marketing Dashboard · ⚠️ Data updated 2h ago       │
│                                              🔓 Logout         │
├────────────────────────────────────────────────────────────────┤
│ SECTION 0 — EXECUTIVE SUMMARY (6 hero cards, always visible)  │
│ [Daily Revenue] [Spend] [ROAS] [Conv Rate] [Active] [Open Rek]│
├────────────────────────────────────────────────────────────────┤
│ ⚠️ ALERTS BAR (red banner pokud něco kritického)              │
├────────────────────────────────────────────────────────────────┤
│ TABS:                                                           │
│ 📢 Kampaně | 🔍 SEO | 🛒 B2C | 🏢 B2B | ⚖️ Srovnávače |      │
│ 📦 Feedy | 🔧 Produkty | 📂 Kategorie | 📍 Pobočky | 🗺️ Kraje │
│ 🚚 Doprava | ⚠️ Reklamace | 🔄 Vratky | 📊 Cross-channel    │
├────────────────────────────────────────────────────────────────┤
│ ACTIVE TAB CONTENT (5 hero KPIs + 6-8 supporting + charts)    │
└────────────────────────────────────────────────────────────────┘
```

### UI principles
- **Trafik signal everywhere**: 🟢 dobré / 🟡 sledovat / 🔴 akce
- **MoM/WoW comparison** vedle každé KPI
- **Drill-down na klik**: KPI číslo → detail tabulka
- **Timeframe selector** (top right): Today / 7d / 30d / 90d / YoY
- **Export button**: CSV / PNG snapshot for reports

---

## 6. Implementační plán (po vašem schválení)

### Phase 1 — Foundation (1-2 dny)
1. Setup nový repo `kc-hbgroupcz/marketing-dashboard` (nebo subfolder existing)
2. Cloudflare Access policy pro `marketing.klicovecentrum.cz`
3. Backend cron skripty + alert email
4. Static UI scaffold (header + tabs + executive summary)

### Phase 2 — Sections (3-5 dnů)
5. Section 0 (Executive) + 12 sekcí postupně
6. Per sekce: data fetch + JSON build + chart rendering
7. Drill-down + filters + export

### Phase 3 — Production (1 den)
8. Windows Task Scheduler nastavení (denní 6:00)
9. Email alert SMTP setup (Gmail App Password)
10. Documentation pro klienta jak interpretovat KPIs

### Phase 4 — Iterace (continuous)
11. Týdenní review s klientem — co používá, co chybí
12. Add new sections / metrics dle feedbacku

---

## 7. Co od vás potřebuju (decisions before implementation)

| Otázka | Možnosti | Default |
|--------|----------|---------|
| **Subdomain** | `marketing.klicovecentrum.cz` / `dashboard.hbgroup.cz` / jiný? | marketing.klicovecentrum.cz |
| **Refresh frequency** | Real-time (5 min) / Hourly / Daily 6:00 | Daily 6:00 |
| **Alert channel** | Email / Slack / Discord / SMS | Email (oskrebsky@gmail.com) |
| **CSV upload location** | OneDrive folder / FTP / API endpoint | OneDrive (jako dnes) |
| **Existující audit dashboard** | Zachovat / merge / nahradit | **Zachovat separately** (audit = analytical retrospective, marketing = operational live) |
| **Owner per sekce** | Vybrat které sekce klient sám řídí, které agentura | Doporučuju oboje vidí, ale TODO list per role |
| **Historical data** | Last 90 days / 12M / Lifetime | 90d default + 12M comparison |
| **Mobile view** | Optimalizovat / desktop only | Mobile responsive |
| **Branding** | Vlastní logo / barvy klienta | klient pošle logo + brand kit |

---

## 8. Odhad nákladů + ROI

### Implementation
- **Setup time**: 5-8 dnů (je to 14 sekcí × cca 5 KPI = 70 metrik + alerts + chart rendering)
- **Cost**: pokud já = mé hodiny; pokud agentura = 50-80 hod × hodinová sazba
- **Hosting**: zdarma (GitHub Pages + Cloudflare Free)
- **Email alerts**: zdarma (Gmail SMTP)
- **Maintenance**: 1-2h/měs (přidat new metrics dle feedbacku)

### ROI projekce
- **Detekce 1 dalšího payment outage** (20 dnů × 4 trans/den × 1 472 Kč = ~118 k Kč) → návratnost setupu
- **Detekce 1 ROAS pádu** (před 8 týdny bot útok = stovky tisíc utracených na bot data)
- **B2B churn alert** — 1 zachráněný TOP 10 partner = 100-300 k Kč/rok obratu
- **Stock-out lost revenue** — 5-10 % obratu typicky teče vodou nedostatkem; dashboard by chytl 80 % case

**Konzervativně**: dashboard hodnota pro klienta = 200-500 k Kč/rok zachráněný revenue + lepší rozhodování.

---

## ❓ Vaše rozhodnutí

Schvalte / upravte:

**Schvalujete strategii?** (Y/N)
- Pokud Y → začnu Phase 1 (Foundation) hned
- Pokud N → řekněte co měnit (méně sekcí? jiné KPIs? jiná architecture?)

**Které sekce mají highest priority?** (top 3 implementovat nejdřív)
- Default doporučuji: Executive Summary + Kampaně + B2C E-shop (= daily operations)

**Budget on tohle:**
- (A) Implementuji v rámci současné spolupráce (žádný extra náklad)
- (B) Strategy doc + outline → klient/agentura realizuje (pošlu detail zadání)
- (C) Pouze část — vyberte které sekce

Pak jdeme na to.
