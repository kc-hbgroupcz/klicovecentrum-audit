# 🚨 Cloudflare runbook — okamžitá náprava útoku (1.-7.5.2026)

**Klient má již nakonfigurováno** (z screenshotů 7.5.2026):
- ✅ Cache Rules 1/10: Bypass `/sign|/account|/admin|/kosik|/objednavka`
- ✅ Custom rules 4/5: Block/Challenge Bad ASNs (23 670 events/24h!), Geo challenge datacentra, Geo blokování bot traffic, Block uploaded scripts
- ✅ Rate limiting 1/1: Leaked credential check
- ✅ Page Rules 1/3: Always Use HTTPS

**Chybí (kritické pro zastavení 3. vlny útoku):**

---

## 🔥 P0-1: Per-IP Rate Limiting rule (NEJDŮLEŽITĚJŠÍ)

**Problém**: Aktuální rate limiting jen na leaked credentials. Bot může poslat 1000 req/min z jedné IP a Cloudflare ho neomezí.

**Akce**: Cloudflare Dashboard → Security → Security Rules → **Rate limiting rules** → **Create rule**

**Konfigurace:**
- **Rule name**: `Per-IP rate limit (bot protection)`
- **If incoming requests match...**:
  - Field: `User Agent`
  - Operator: `does not contain`
  - Value: `Googlebot`
  - AND Field: `User Agent` `does not contain` `Bingbot`
  - AND Field: `User Agent` `does not contain` `DuckDuckBot`
- **Then take action...**:
  - **When rate exceeds**: `30 requests per 1 minute`
  - **Counting characteristics**: IP source address
  - **Action**: `Managed Challenge` (nebo `Block` pokud chcete tvrdší)
  - **Duration**: 10 minutes

**API equivalent** (pokud máte CF_AUTH_TOKEN):
```bash
ZONE_ID=$(curl -s -X GET "https://api.cloudflare.com/client/v4/zones?name=klicovecentrum.cz" \
  -H "Authorization: Bearer $CF_AUTH_TOKEN" -H "Content-Type: application/json" \
  | python -c "import json,sys; print(json.load(sys.stdin)['result'][0]['id'])")

curl -X POST "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/rulesets/phases/http_ratelimit/entrypoint/rules" \
  -H "Authorization: Bearer $CF_AUTH_TOKEN" \
  -H "Content-Type: application/json" \
  --data '{
    "description": "Per-IP rate limit (bot protection) — 30 req/min",
    "expression": "not (http.user_agent contains \"Googlebot\" or http.user_agent contains \"Bingbot\" or http.user_agent contains \"DuckDuckBot\")",
    "action": "managed_challenge",
    "ratelimit": {
      "characteristics": ["ip.src"],
      "period": 60,
      "requests_per_period": 30,
      "mitigation_timeout": 600
    }
  }'
```

---

## 🔥 P0-2: Bot Fight Mode + Super Bot Fight Mode

**Akce**: Cloudflare Dashboard → Security → **Bots**

**Konfigurace:**
- **Bot Fight Mode**: ✅ ON (zdarma plan, blokuje known bots)
- **Super Bot Fight Mode** (vyžaduje Pro/Business plan, doporučeno):
  - Definitely automated: **Block**
  - Likely automated: **Managed Challenge**
  - Verified bots (Google, Bing): **Allow**
  - JavaScript Detections: **ON**

---

## 🔥 P0-3: 5. Custom rule — Block podle bot signature útoku

**Slot 5/5 zbývá**. Návrh rule na **specifické signaturee aktivního útoku** (1.-7.5.2026):

**Akce**: Security → Security Rules → Custom rules → Create rule

**Rule name**: `Block weekend bot pattern (3rd wave 5/2026)`
**Expression** (Edit expression mode):
```
(http.user_agent contains "MJ12bot") or
(http.user_agent contains "AhrefsBot" and not cf.client.bot) or
(http.user_agent contains "SemrushBot" and not cf.client.bot) or
(http.user_agent contains "DotBot") or
(http.user_agent contains "PetalBot") or
(http.user_agent eq "") or
(ip.geoip.asnum in {16509 14618 16276 51167 8075 396982} and not cf.client.bot)
```
**Action**: `Block`

(První 5 = aggressive SEO scrapers; ASNs = AWS, OVH, Hetzner, Google Cloud datacentra)

---

## 🔥 P0-4: Monitor + Alert setup

**API monitoring command** (pošlete to každých 5 minut přes cron):
```bash
# Get blocked count last hour
curl "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/security/events?since=$(date -u -d '1 hour ago' '+%Y-%m-%dT%H:%M:%SZ')" \
  -H "Authorization: Bearer $CF_AUTH_TOKEN" \
  | python -c "import json,sys; d=json.load(sys.stdin); print(len(d.get('result',[])))"
```

Pokud > 1000 events/h → alert (email/Slack).

---

## 🎯 Po nastavení (1 hodina monitoring)

1. Sledujte Cloudflare → Security → Events — měli byste vidět tisíce blocked
2. GA4 → Real-time — sessions/min by mělo klesnout < 10
3. Pokud útok pokračuje, zvyšte rate limit na `15 req/min` + přidejte specifické IP blocky

---

## 📊 Current state (7.5.2026)

| Rule | Action | 24h events |
|------|--------|------------|
| Block/Challenge Bad ASNs | Managed Challenge | **23 670** (0.12 % CSR) |
| Geo challenge - datacentra | Managed Challenge | 189 (0.53 % CSR) |
| Geo blokování - bot traffic | Block | **610** |
| Block Uploaded Scripts | Block | 0 |
| Leaked credential check | Block | 0 |

Celkem 24 469 blocked = **Cloudflare už spoustu útoku blokuje**, ale GA4 vidí 9-22k sessions/den = je tu **víc útočníků než CF chytá**. Per-IP rate limit doplní obranu pro útoky které procházejí (různé IPs, různé ASNs, valid user agent).

