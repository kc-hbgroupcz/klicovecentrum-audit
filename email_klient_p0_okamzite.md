# Email pro klienta — okamžitá akce (předpis 7.5.2026)

---

**Komu:** [klient hbgroup.cz / vývojář / agentura]
**Předmět:** 🚨 URGENT — Probíhající bot útok 3. vlna · 4 P0 akce do 30 min

---

Ahoj,

Audit analytics-audit dashboardu (https://kc-hbgroupcz.github.io/klicovecentrum-audit/dashboard.html) identifikoval **aktivní probíhající bot útok** (3. vlna v 2026, peak víkend 2.-3.5., dnes 7.5. pořád 5× normal traffic).

**Data:**
- GA4: 22 197 sessions/den (vs 1500-2500 normal), bounce 94 %, direct 96 %, jen 1 transakce
- Bing: 200 5xx + 49 4xx + 155 646 robots.txt blocked v jeden den (3.5.)
- Cloudflare už blokuje 23 670/24h na Bad ASNs custom rule — část útoku prochází

**4 akce do 30 min (návod níže):**

## 1️⃣ Cloudflare Rate limiting per-IP (5 min)
Dashboard → Security → Security Rules → Rate limiting rules → **Create rule**
- Name: `Per-IP rate limit (bot protection)`
- If: User Agent does not contain `Googlebot` AND not contain `Bingbot`
- When rate exceeds: **30 requests / 1 minute** per IP source
- Action: **Managed Challenge** (10 min duration)

## 2️⃣ Bot Fight Mode (2 min)
Dashboard → Security → **Bots** → Bot Fight Mode **ON** + Super Bot Fight Mode (Pro plan)

## 3️⃣ 5. Custom rule — blokovat scrapery (10 min)
Dashboard → Security → Custom rules → Create rule
- Name: `Block weekend bot pattern`
- Expression: `(http.user_agent contains "MJ12bot") or (http.user_agent contains "DotBot") or (http.user_agent contains "PetalBot") or (http.user_agent eq "") or (ip.geoip.asnum in {16509 14618 16276 51167 8075 396982} and not cf.client.bot)`
- Action: **Block**

## 4️⃣ GA4 bot filtering + annotate útoky (10 min)
analytics.google.com → Admin → Property → Data Settings → Data Filters → **Bot filtering ON**
+ Annotate (pravidelné notes):
  - 6.-18.3.2026 = bot vlna 1 (peak 16.3. = 253k sess)
  - 27.3.-6.4.2026 = bot vlna 2 (peak 29.3. = 103k sess)
  - 1.-7.5.2026 = bot vlna 3 AKTIVNÍ (peak 3.5. = 22k sess)

---

**Po nastavení (1 hodina monitoring):**
1. Cloudflare → Security → Events → měli byste vidět tisíce challenged/blocked
2. GA4 → Reports → Realtime → sessions/min by mělo klesnout pod 10
3. Pokud útok stále vidíte → zvyšte rate limit na **15 req/min** + manuální IP block top útočníků z CF logs

**Co očekávat za 24-48h:**
- 90 % redukce bot trafficu v GA4
- Bing Webmaster 5xx errors klesnou na 0
- Smart Bidding získá čistá data (relearning 2-4 týdny)
- Recovery PPC ROAS z 1,96× → 4× (cca 1,2-1,6 M Kč/rok dopad)

Plný runbook (s curl API commands) v repo: `analytics-audit/cloudflare_runbook.md`

Dej vědět jakmile bude nasazeno — budu monitorovat dashboard a potvrdit pokles.

Díky,
Ondra

---
