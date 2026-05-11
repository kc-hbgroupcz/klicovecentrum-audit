# 🛡️ Cloudflare manuál — Free plan · klicovecentrum.cz
**Datum:** 7. 5. 2026 · **Cíl:** zastavit probíhající 3. vlnu bot útoku (1.-7.5.2026, peak 3.5. = 22 197 sess)

---

## Co máte aktuálně (k 7.5.2026)

| Kategorie | Stav | Slot |
|-----------|------|------|
| Custom Rules | ✅ 4 aktivní (24 469 events blocked/24h) | 4/5 |
| Rate Limiting Rules | ✅ 1 aktivní (Leaked credentials) | **1/1 FULL** |
| Cache Rules | ✅ 1 (Bypass) | 1/10 |
| Page Rules | ✅ 1 (Always HTTPS) | 1/3 |
| Bot Fight Mode | ❌ vypnutý | — |

**Free plan má jen 1 Rate Limiting rule** — už ji používáte. Proto budu řešit přes 5. Custom rule + Bot Fight Mode.

---

## 🔥 KROK 1 — Bot Fight Mode (2 min, ZADARMA)

**Co dělá**: Cloudflare detekuje známé boty (Ahrefs, SEMrush, MJ12, DotBot atd.) a blokuje je. Free plan má základní verzi, plně funkční.

**Postup:**

1. Otevřete Cloudflare Dashboard → vyberte zónu **klicovecentrum.cz**
2. V levém menu klikněte **Security** (štít ikona)
3. Pod ní najděte **Bots**
4. **Bot Fight Mode** → přepínač **ON** ✅
5. ⚠️ V Free planu uvidíte hlášku "Super Bot Fight Mode requires Pro plan" — to je OK, Bot Fight Mode samotný funguje

**Verify**: Po pár minutách → Security → **Events** → měli byste vidět nové events s action "Block — Bot Fight Mode" / "Managed Challenge — Bot Fight Mode"

---

## 🔥 KROK 2 — 5. Custom Rule (10 min)

Použijeme zbývající 5. slot pro **bot scraper block** (rate limit nelze bez Pro planu).

**Postup:**

1. Cloudflare Dashboard → **Security** → **Security rules**
2. V dropdown nahoře vyberte **Custom rules** (ne Rate limiting!)
3. Klikněte **+ Create rule**
4. Vyplňte:

**Rule name:**
```
Block aggressive scrapers + suspicious patterns
```

**Description:**
```
Blokuje SEO scrapery (Ahrefs/SEMrush/DotBot/PetalBot/MJ12), prazdny User-Agent, a utoky z datacenter ASNs (AWS/OVH/Hetzner/GCP). Doplnuje stavajici Bad ASNs rule a Geo block.
```

5. **When incoming requests match...** klikněte **"Edit expression"** (tlačítko vpravo) a vložte:

```
(http.user_agent contains "MJ12bot") or
(http.user_agent contains "DotBot") or
(http.user_agent contains "PetalBot") or
(http.user_agent contains "AhrefsBot" and not cf.client.bot) or
(http.user_agent contains "SemrushBot" and not cf.client.bot) or
(http.user_agent contains "DataForSeoBot") or
(http.user_agent contains "Bytespider") or
(http.user_agent eq "" and not cf.client.bot) or
(ip.geoip.asnum in {16509 14618 16276 14061 51167 8075 396982 60068 45090} and not cf.client.bot)
```

**Vysvětlení ASNs:**
- 16509 = Amazon AWS
- 14618 = Amazon AWS (US-East)
- 16276 = OVH
- 14061 = DigitalOcean
- 51167 = Contabo
- 8075 = Microsoft Azure
- 396982 = Google Cloud
- 60068 = CDN77 datacenter
- 45090 = Tencent Cloud

6. **Then take action...**:
   - Choose action: **Block**
   - (alternativně: **Managed Challenge** pokud chcete dát šanci falešně positivním)

7. **Place at**: Custom (na konec nebo top — doporučuju **top 1** = nejvyšší priorita)

8. Klikněte **Deploy**

**Verify** (po 5 min): Security → Events → filtr Rule = "Block aggressive scrapers" → měli byste vidět blocked traffic

---

## 🔥 KROK 3 — Upravit Geo blokování rule (5 min)

Vaše stávající rule "Geo blokování - bot traffic" blokuje **jen** CN/RU/VN/PK/BD/IQ. Aktivní útok pravděpodobně používá US/EU IPs (proxy). Doplníme rule.

**Postup:**

1. Cloudflare → Security → Security rules → Custom rules
2. Najděte rule **"Geo blokování - bot traffic"** → 3 tečky vpravo → **Edit**
3. **Edit expression**:

**STARÉ** (zachovat ale rozšířit):
```
(ip.geoip.country in {"CN" "RU" "VN" "PK" "BD" "IQ"} and not cf.client.bot) or
(http.user_agent contains "Bytespider") or
(http.user_agent contains "Bytedance")
```

**NAHRADIT za:**
```
(ip.geoip.country in {"CN" "RU" "VN" "PK" "BD" "IQ" "ID" "TH" "PH" "NG" "EG" "TR"} and not cf.client.bot) or
(http.user_agent contains "Bytespider") or
(http.user_agent contains "Bytedance") or
(http.request.uri.path contains "wp-admin" and not cf.client.bot) or
(http.request.uri.path contains ".env") or
(http.request.uri.path contains "phpmyadmin") or
(http.request.uri.path contains ".git/")
```

**Co přidáno:**
- 6 nových zemí s vysokým bot trafficem (Indonésie, Thajsko, Filipíny, Nigerie, Egypt, Turecko)
- WordPress admin scanning (klicovecentrum.cz není WP → automaticky bot)
- `.env` file scanning (klasické exploit)
- phpMyAdmin scanning
- `.git/` directory leaks

4. **Deploy**

---

## 🔥 KROK 4 — Cache Rule pro statics (5 min, snižuje server zátěž)

Útok zatěžuje origin server. Pokud Cloudflare cachuje co nejvíc, origin dostane méně requestů → útok má menší dopad.

**Postup:**

1. Cloudflare → **Caching** → **Cache Rules**
2. **Create rule**
3. **Rule name**: `Aggressive cache statics`
4. **When incoming requests match...** Edit expression:
```
(http.request.uri.path matches "\.(css|js|jpg|jpeg|png|gif|webp|svg|woff|woff2|ico|ttf|eot|otf|map|pdf)$") or
(http.request.uri.path contains "/css/") or
(http.request.uri.path contains "/js/") or
(http.request.uri.path contains "/images/") or
(http.request.uri.path contains "/fonts/")
```
5. **Then...**:
   - **Cache eligibility**: Eligible for cache ✅
   - **Edge TTL**: Override origin → **1 month**
   - **Browser TTL**: Override origin → **1 week**
6. Deploy

To zajistí, že Cloudflare cachuje statics agresivně → 95 % bot requestů na obrázky/CSS/JS nikdy nedosáhne origin server.

---

## 🔥 KROK 5 — Security Level (1 min)

1. Cloudflare → **Security** → **Settings** (vedle Events)
2. **Security Level**: nastavit na **High** (default je Medium)

V High mode Cloudflare automaticky challenge všechny návštěvníky s threat score > 0. Free plan to umí.

⚠️ Pozor: může dočasně omezit i legitimní traffic. Po opadnutí útoku přepněte zpět na **Medium**.

---

## 🔥 KROK 6 — Browser Integrity Check (1 min)

1. Cloudflare → **Security** → **Settings**
2. **Browser Integrity Check**: ✅ ON
3. **Challenge Passage**: 30 minutes (default)

Bot často nemá funkční browser → Cloudflare ho vyzve challenge, který bot nezvládne → blok.

---

## 🔥 KROK 7 — Challenge Passage + Privacy Pass (1 min)

1. Cloudflare → **Security** → **Settings**
2. **Challenge Passage**: 1 hour (snižuje frikci pro legitimní uživatele po prvním challenge)
3. **Privacy Pass Support**: ✅ ON (umožňuje přátelským botům jako Brave / Tor projít)

---

## 🔥 KROK 8 — GA4 bot filtering + annotate (5 min)

Nezávislé na Cloudflare, ale stejně důležité — očištění analytics dat:

1. https://analytics.google.com → vyberte property **klicovecentrum.cz (299992437)**
2. Admin (ozubené kolečko vlevo dole) → **Property settings** → **Data Settings** → **Data Filters**
3. Pokud "Internal Traffic" filter neexistuje → **Create Filter** → "Developer Traffic" → exclude
4. ⚠️ GA4 už má **automatic bot filtering** zapnuté (default — Google filtruje SPB/IAB known bots automaticky). Nelze ručně přepnout, ale můžete přidat **custom Internal Traffic filter** pro vlastní firemní/agenturní IP.

**Annotate 3 vlny útoku** (visual ve všech reportech):
1. GA4 → Reports → libovolný report → klik na konkrétní datum v grafu
2. **Add annotation**:
   - **6.-18. 3. 2026** = "Bot útok vlna 1 (peak 16.3. = 253k sess)"
   - **27. 3. - 6. 4. 2026** = "Bot útok vlna 2 (peak 29.3. = 103k sess)"
   - **1.-7. 5. 2026** = "Bot útok vlna 3 AKTIVNÍ (peak 3.5. = 22k sess)"

---

## ✅ Souhrn všech 8 kroků

| # | Akce | Místo v Cloudflare | Čas |
|---|------|--------------------|----|
| 1 | Bot Fight Mode ON | Security → Bots | 2 min |
| 2 | 5. Custom rule (scrapery + datacenter ASNs) | Security → Security rules → Custom rules | 10 min |
| 3 | Rozšířit Geo blokování rule (více zemí + exploit paths) | Security → Custom rules (edit existing) | 5 min |
| 4 | Cache Rule pro statics | Caching → Cache Rules | 5 min |
| 5 | Security Level → High | Security → Settings | 1 min |
| 6 | Browser Integrity Check ON | Security → Settings | 1 min |
| 7 | Challenge Passage 1h + Privacy Pass ON | Security → Settings | 1 min |
| 8 | GA4 bot filtering verify + annotate 3 vlny | analytics.google.com → Admin | 5 min |

**Total: ~30 min**

---

## 📊 Co očekávat (1-24h po nasazení)

### Hodina 1
- Cloudflare → Security → Events: + tisíce challenged / blocked
- GA4 → Real-time: sessions/min klesá

### 24h
- GA4 daily sessions klesnou z 22k → 2-3k (normal)
- Bing crawl 5xx klesnou z 200 → < 20
- Bing crawled pages stoupnou zpět na ~28 000/den
- ROAS Smart Bidding začne dostávat čistá data

### 7 dní
- Trans/den vzroste z 1 → 4-6 (baseline)
- Bot useless sessions excluded → Real Direct % klesne na 30-40 % (z 96 %)
- Bounce rate klesne z 90+ % na ~12-15 %

---

## 🚨 Pokud útok pokračuje i po nasazení

### Option A — IP-level block (manual)
1. Cloudflare → Security → **Events** → filter Action = "Block" za posledních 24h
2. Top 20 IP s nejvyšším počtem requestů → klik na IP → **Block this IP**
3. Můžete blocknout celé /24 nebo /16 rozsahy (klikněte na IP → "Block /24" v menu)

### Option B — Snížit Custom rule expression na měkčí Managed Challenge
Pokud blokujete moc legitimního trafficu, změňte:
- Custom rule #5 Action: **Block** → **Managed Challenge**
- Geo block rule Action: **Block** → **Managed Challenge**

### Option C — Zapnout "I'm Under Attack" mode (nuclear)
1. Security → Settings → **Security Level** → **I'm Under Attack!**
2. To zapojí JS challenge na **každého** návštěvníka 5 sekund před přístupem
3. ⚠️ Zhorší UX, ale bot útok zastaví okamžitě
4. Vypnout po opadnutí útoku (vrátit na Medium/High)

---

## 🛠️ Po-útoku — re-baseline (po 7 dnech čistého trafficu)

1. **Smart Bidding relearning**: Google Ads → každá kampaň s Smart Bidding → manual review (relearning fáze 2-4 týdny, ROAS dočasně poklesne)
2. **GA4 audience rebuild**: stávající audiences (lookalike, remarketing) jsou kontaminovány bot daty → exclude period 6.3.-7.5.2026 z audience definitions
3. **Re-export reports**: leden-květen 2026 = export "skutečných" dat pro reporting, ne ta nafouklá z GA4
4. **Update audit report**: nálezy #1 (konverze 265× nafouklé) + #41 (ROAS pád 4,78→1,96×) získají kontext **bot inflated 8 týdnů**

---

## ❓ FAQ

**Q: Co když Cloudflare custom rule v kroku 2 začne blokovat legitimní Googlebot/Bingbot?**
A: Expression má `and not cf.client.bot` ochranu = verified bots Cloudflarem (Googlebot, Bingbot, DuckDuckBot, Yandex…) projdou.

**Q: Proč jsem nemohli použít skutečný Rate Limiting?**
A: Free plan má **1 slot** Rate Limiting rules. Klient už ji používá pro Leaked Credentials. Buď přesunout Leaked Credentials do Custom rule (chybí slot 6/5), nebo upgrade na Pro plan ($20/měs = 5 RL rules).

**Q: Můžu dostat Pro plan free trial?**
A: Cloudflare občas dává 14-30 dní trial. Zkuste Settings → Plans → Pro → "Try free for 14 days" (pokud nabízí).

**Q: Co když útok bude pokračovat příští víkend?**
A: Bot Fight Mode + Custom rules zachytí 90 %+ z útoku. Pokud útok eskaluje, zapněte "I'm Under Attack" mode přes víkend, vypněte v pondělí ráno.

**Q: Mohu API místo UI?**
A: Ano, viz `cloudflare_runbook.md` pro curl commands. Potřebujete CF_AUTH_TOKEN s Zone:Edit permissions.

**Q: Jak otestuju že Custom rule v kroku 2 funguje?**
A: Otevřete v jiném prohlížeči `https://www.bing.com/webmasters` (legitimní bot) → projít. Pak otevřete `curl -A "MJ12bot/1.0" https://www.klicovecentrum.cz` → měli byste dostat 403 Block. Cloudflare → Security → Events → poslední záznam = váš MJ12bot test = blocked.

---

Plný runbook (s curl API equivalenty): `analytics-audit/cloudflare_runbook.md`
Dashboard live: https://kc-hbgroupcz.github.io/klicovecentrum-audit/dashboard.html (sekce 📅 3-letý přehled → "NÁLEZ #76 — PROBÍHAJÍCÍ BOT ÚTOK")
