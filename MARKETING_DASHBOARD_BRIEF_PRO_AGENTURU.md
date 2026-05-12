# 📋 Zadání: Marketing Dashboard klicovecentrum.cz + hbgroup.cz
**Klient:** H&B Group s.r.o. (Ondřej Skřebský, oskrebsky@gmail.com)
**Datum zadání:** 12. 5. 2026
**Předpokládaný termín dodání:** 4-6 týdnů
**Předpokládaný rozpočet:** dle nabídky agentury (orientačně 80-120 hod)

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

## 3. 15 sekcí dashboardu — KPI specifikace

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

### 📦 Section 6 — FEEDY (Mergado)

**Hero KPIs (5):**
1. Total active products in feeds
2. Disapproved count (Merchant Center)
3. Last sync timestamp per feed (Google MC / Heuréka / Zboží / Alza)
4. **Coverage %**: bestsellers ve feedu vs ERP TOP 50
5. Feed quality score

**Supporting (6):**
- Per-feed health
- Variant coverage % (target 100 %)
- Bestsellers chybí ve feedu count
- **Feed-to-ERP price diff alerts** (cena ve feedu ≠ aktuální)
- **Stock-out coverage** (out-of-stock items pořád ve feedu)
- Custom_label_1 distribution (PMax labely)

**Alerts:**
- Disapproved > 50 % → critical
- Last sync > 24h → feed broken

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

## 4. UI/UX requirements

### Layout pattern
```
┌────────────────────────────────────────────────────────────────┐
│ HEADER: 🎯 Marketing Dashboard · ⚠️ Last update 6:00          │
│                                              🔓 Logout         │
├────────────────────────────────────────────────────────────────┤
│ EXECUTIVE SUMMARY (6 hero cards)                              │
├────────────────────────────────────────────────────────────────┤
│ ALERTS BAR (red banner pokud kritické)                        │
├────────────────────────────────────────────────────────────────┤
│ TABS: 15 sekcí (kampaně, SEO, B2C, B2B, srovnávače, ...)      │
├────────────────────────────────────────────────────────────────┤
│ ACTIVE SECTION (hero KPIs + supporting + charts + tabulky)    │
└────────────────────────────────────────────────────────────────┘
```

### UI principles
- ✅ Trafik signal everywhere: 🟢 / 🟡 / 🔴
- ✅ MoM/WoW comparison vedle každé KPI
- ✅ Drill-down na klik (KPI → detail tabulka)
- ✅ Timeframe selector: Today / 7d / 30d / 90d / YoY
- ✅ Export button: CSV / PNG snapshot
- ✅ Mobile responsive
- ✅ Dark theme (jako existující audit dashboard, pro brand consistency)

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
| **Alza Partner orders denní** | Alza Seller Portal → CSV export (nebo API integrace) | P1 (pro Section 5) |
| **Heuréka OCM denní** | Klient nastaví OCM API push (existuje) | P2 |
| **Zboží.cz OCM denní** | Klient nastaví OCM API push | P2 |
| **Mailchimp/Ecomail API** | Pokud klient používá email kampaně | P2 (volitelné) |
| **Meta Ads API** | Brand kampaně Plzeň/Brno/Chomutov | P1 (pro Section 1) |
| **NPS data** | Heureka Ověřeno + post-purchase survey | P3 (nice-to-have) |

---

## 6. Implementation phases & timeline

### Phase 1 — Foundation (Week 1)
- [ ] Setup repo + GitHub Pages + Cloudflare Access
- [ ] Backend cron infrastructure + alert email SMTP
- [ ] Frontend scaffold (header + tabs + executive summary)
- [ ] Acceptance: dashboard otevíratelný na `audit.klicovecentrum.cz/marketing/` s prázdnými sekcemi

### Phase 2 — Top 3 sekce (Week 2)
- [ ] Section 0 (Executive Summary) — fully funkční
- [ ] Section 1 (Kampaně) — fully funkční
- [ ] Section 3 (B2C) — fully funkční
- [ ] Acceptance: 3 sekce s real ERP data, alerts fungují, refresh cron běží

### Phase 3 — Zbývajících 12 sekcí (Week 3-4)
- [ ] Section 2, 4-14 — postupně dle priority
- [ ] Drill-down + filters + export
- [ ] Mobile responsive testing

### Phase 4 — Production polish + handoff (Week 5-6)
- [ ] Windows Task Scheduler nastavení / GitHub Actions cron
- [ ] Email alert SMTP fully configured
- [ ] Per-user Cloudflare Access policy
- [ ] Documentation pro klienta jak interpretovat KPIs
- [ ] Training session (1h call)

---

## 7. Acceptance criteria

✅ Marketing dashboard funkční na `audit.klicovecentrum.cz/marketing/` s Cloudflare Access auth
✅ Všech 15 sekcí implementováno s minimálně hero KPIs (supporting může být fáze 2)
✅ Daily 6:00 refresh probíhá automatically, last_update timestamp viditelný v UI
✅ Email alert na `oskrebsky@gmail.com` při CSV stale > 25h FUNGUJE (test scenario)
✅ Mobile responsive (testováno na iPhone + Android)
✅ Page load < 3 sekundy na 4G
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

- **Estimovaná pracnost**: 80-120 hodin
  - Foundation + auth: 12-16h
  - 15 sekcí × ~5h: 75h
  - Backend cron + alerts: 8-12h
  - Production setup + testing: 10-15h
  - Documentation + training: 5-8h
- **Timeline**: 4-6 týdnů od podpisu zadání
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
