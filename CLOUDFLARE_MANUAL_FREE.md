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

**⚠️ Free plan upozornění**: operátor `matches` (regex) je **Pro+ only**. Použijeme jen `contains` / `is in` (Free-compatible).

**Postup — varianta A (UI bez expression — nejjednodušší):**

1. Cloudflare → **Caching** → **Cache Rules**
2. **Create rule**
3. **Rule name**: `Aggressive cache statics`
4. **When incoming requests match...** v UI vyplňte (žádný "Edit expression"):
   - **Field**: vyberte `File extension`  (z dropdownu, ne URI Path)
   - **Operator**: `is in`
   - **Value**: napište nebo vložte (jeden po druhém, Cloudflare je převede do listu):
     ```
     css, js, jpg, jpeg, png, gif, webp, svg, woff, woff2, ico, ttf, eot, otf, map, pdf
     ```
5. **Then...**:
   - **Cache eligibility**: Eligible for cache ✅
   - **Edge TTL**: Override origin → **1 month**
   - **Browser TTL**: Override origin → **1 week**
6. **Deploy**

**Postup — varianta B (Edit expression, pokud Field „File extension" není dostupné):**

```
(http.request.uri.path.extension in {"css" "js" "jpg" "jpeg" "png" "gif" "webp" "svg" "woff" "woff2" "ico" "ttf" "eot" "otf" "map" "pdf"})
```

(`http.request.uri.path.extension` automaticky extrahuje příponu URL — žádný regex potřeba, funguje ve Free planu.)

**Postup — varianta C (fallback s `contains` pokud ani extension field nefunguje):**

```
(http.request.uri.path contains "/css/") or
(http.request.uri.path contains "/js/") or
(http.request.uri.path contains "/images/") or
(http.request.uri.path contains "/fonts/") or
(http.request.uri.path contains "/static/") or
(http.request.uri.path contains "/uploads/")
```

Tím cachujete celé adresářové prefixy. Méně přesné než file extension match, ale 100 % Free-compatible.

To zajistí, že Cloudflare cachuje statics agresivně → 95 % bot requestů na obrázky/CSS/JS nikdy nedosáhne origin server.

**Verify cache je aktivní (po 5 min):**
1. Otevřete v prohlížeči `https://www.klicovecentrum.cz/` (homepage)
2. F12 → Network tab → reload (Ctrl+R)
3. Klikněte na libovolný `.css` nebo `.jpg` request → Headers tab
4. Hledejte response header `cf-cache-status: HIT` (nebo `MISS` u prvního requestu, pak `HIT`)
5. Pokud vidíte `cf-cache-status: BYPASS` nebo `DYNAMIC` → cache rule nefunguje, zkontrolujte expression

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

## 🔥 KROK 8 — GA4 ochrana dat (5-10 min)

⚠️ **Důležité chápat**: GA4 **NEMÁ annotations** (na rozdíl od starého Universal Analytics). Místo toho má **automatic bot filtering** + **comparisons** + **Audiences**. Ukazuji co reálně **funguje**.

### 8a) Ověřit automatic bot filtering (= ON by default)

1. https://analytics.google.com → vyberte property **klicovecentrum.cz (299992437)**
2. Vlevo dole klik **Admin** (ozubené kolečko)
3. V „Property settings" sekci klik **Data settings → Data Streams**
4. Klik na váš Web stream `https://www.klicovecentrum.cz/`
5. Sekce "Tagging settings" → klik **Configure tag settings**
6. Klik **Internal traffic** (nebo „Show all" → Internal traffic)

Co byste tam měli vidět:
- ✅ **„Bot filtering"** = automaticky ON (Google filtruje IAB Bot/Spider list — nelze vypnout, ne všechny boty)

⚠️ **GA4 automatic bot filter chytí jen ~40-60 % botů** (známí crawlery v IAB seznamu). **Aggressive scrapery v útoku 1.-7.5.2026 NEFILTRUJE** — ty se v datech objeví. Cloudflare (kroky 1-7) je hlavní obrana.

### 8b) Přidat „Internal Traffic" filter pro vás/agenturu (volitelné)

Pokud vaše IP nebo IP agentury vidí GA4 data z testů → exkluduje:

1. Admin → **Data Settings** → **Data filters** (vlevo)
2. Pokud `Internal Traffic` filter neexistuje:
   - **Create Filter** → název `Internal Traffic`
   - Filter type: `Internal Traffic`
   - Filter operation: `Exclude`
   - Parameter value: `internal` (ukáže se výchozí)
3. Aby to fungovalo, musíte ještě v **Data Streams → Tagging Settings → Define internal traffic** přidat IP rules:
   - **Create rule** → Match type: `IP address equals` → vaše veřejná IP (zjistíte na https://whatismyip.com)
   - Stejně přidejte IP agentury

### 8c) Vytvořit „Bot Sessions" Audience (= označit historická + budoucí bot data)

GA4 nedává annotations, ale **Audience** funguje jako label. Vytvoříte audience „Bot Sessions" pro charakteristiky útoku → můžete ji **použít jako Comparison** v jakémkoli reportu.

**Postup:**

1. GA4 → vlevo dole **Admin** → v sekci „Data display" → **Audiences**
2. **New audience** → **Create custom audience**
3. Název: `🚨 Bot Sessions (filter z reports)`
4. Description: `Sessions s bot signaturou: bounce 90%+ a session duration < 30s a Direct channel. Útok 6.3.-7.5.2026`
5. **Add condition** → klik **+ Add new condition**:
   - Event: `session_start` (nebo session-scoped metric)
   - Add filter: `Bounce rate >= 0.9` AND `Average engagement time per session < 30`
   - + Filter: `Session source/medium contains "(direct)"`
6. **Membership duration**: 90 days (default)
7. **Save**

⚠️ Nejde retroaktivně exkluvovat z dat (data jsou už v BigQuery). Audience slouží jen pro **identifikaci** v Comparisons.

### 8d) Comparisons v reports → vidíte „čistá vs zabugovaná" data

V každém GA4 reportu (Reports → Engagement → Pages, atd.):

1. Nahoře vpravo klik **„Add comparison"** (📊 ikona)
2. **+ Add new comparison**:
   - Dimension: `Default channel group`
   - Match type: `does not exactly match`
   - Value: `Direct`
3. Apply → uvidíte 2 sloupce: „All users" vs „Bez Direct" (= bez bot sessions)

To je nejbližší ekvivalent k UA annotations v GA4 — místo poznámky vidíte rovnou čistá vs hrubá data.

### 8e) Audit dashboard slouží jako persistent annotation

Dashboard https://kc-hbgroupcz.github.io/klicovecentrum-audit/dashboard.html → tab **📅 3-letý přehled** → sekce **„NÁLEZ #76 — PROBÍHAJÍCÍ BOT ÚTOK"** zobrazuje:
- 3 wave cards s daty (vlna 1: 6.-18.3., vlna 2: 27.3.-6.4., vlna 3: 1.-7.5.)
- Daily timeline chart s víkendovými flagy
- Tabulku posledních 14 dnů (dnes vidíte sessions/bounce/direct přesné hodnoty)

To je vaše **persistent annotation systém** — dashboard si pamatuje vlny útoku navždy, GA4 je zapomene.

### Co dělat s historickými daty (březen-květen 2026)

**Nelze** retroaktivně očistit GA4 data za 6.-18. 3. + 27. 3. - 6. 4. + 1.-7. 5. 2026. Bot sessions tam zůstanou navždy.

**Co dělat:**
1. Při reportingu klientovi nebo agentuře **vždy uveďte**: „GA4 sessions za tato období jsou kontaminovány bot trafficem (vlny 1-3, viz audit dashboard #76)"
2. Pro ROAS / konverze používejte **ERP / SQL ground truth** (1 261 245 Kč MO, 4 351 779 Kč VO 8M) místo GA4
3. Po-attack (od 8. 5. 2026, pokud Cloudflare zafunguje) máte čistá GA4 data — ta sledujte trend

### Souhrn kroku 8

| Akce | Čas | Funguje? |
|------|-----|----------|
| 8a | Ověřit auto bot filtering | 1 min | ✅ default ON, nic neměnit |
| 8b | Internal Traffic filter (vaše IP) | 3 min | ✅ vyloučí váš testing |
| 8c | „Bot Sessions" Audience | 3 min | ✅ identifikace v Comparisons |
| 8d | Comparison „bez Direct" v reportech | 1 min | ✅ čistá vs hrubá data |
| 8e | Audit dashboard = persistent annotation | 0 min | ✅ už máte hotové |

**Annotations v GA4 NEEXISTUJÍ jako v Universal Analytics. Proto outsourcujeme do audit dashboardu.**

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
| 8 | GA4: bot filter ověřit + Internal Traffic filter + „Bot Sessions" Audience + Comparison „bez Direct" v reports | analytics.google.com → Admin + Reports | 5-10 min |

**Total: ~30-35 min**

**Pozn.**: GA4 nemá funkci „annotations" (Universal Analytics měl). Proto v kroku 8 vytváříme **Audience + Comparisons** = nejbližší ekvivalent. Persistentní notes o vlnách útoku máte v audit dashboardu (sekce „NÁLEZ #76").

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
