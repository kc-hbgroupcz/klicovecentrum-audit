# 🚨 Cloudflare účinnost — 24h analýza (17.-18. 5. 2026)

**Vstup:** 4 firewall_events JSON (sample 500/zóna × 4 = 2 000 events) + all_sites country breakdown + 2× GSC Crawl Stats xlsx
**Závěr:** Současné nastavení **DETEKUJE správně, ale BLOKUJE málo** — managed_challenge se ukázalo jako nedostatečné proti perzistentnímu scraperu.

---

## 📊 1. Country breakdown napříč 4 doménami (24h)

| Země | Requests | % celku | Hodnocení |
|------|----------|---------|-----------|
| 🇺🇸 **US** | **3 365 147** | **53.9 %** | 🚨 Bot dominance (datacentra) |
| 🇨🇿 CZ | 1 296 273 | 20.7 % | ✅ Náš trh |
| 🇯🇵 **JP** | **1 155 197** | **18.5 %** | 🚨 **NENÍ v block listě!** |
| 🇸🇰 SK | 168 321 | 2.7 % | ✅ Náš trh |
| BE | 28 018 | 0.5 % | OK |
| CN | 22 879 | 0.4 % | Blokované ✅ |
| CA | 20 395 | 0.3 % | OK (managed_challenge přes rule 3) |
| GB | 19 253 | 0.3 % | OK |
| HK | 18 407 | 0.3 % | Suspicious |
| DE | 18 233 | 0.3 % | OK (managed_challenge přes rule 3) |
| LV | 16 928 | 0.3 % | OK |
| ostatní (180+ zemí) | < 14 000 | < 0.2 % každá | OK |

**Total 24h: 6.25M requests**
- **CZ+SK (náš trh): 23.4 %** (1.46M)
- **US+JP (suspicious): 72.4 %** (4.52M)

→ **72 % traffic není z našich trhů**. Tohle je masivní bot scraping.

---

## 🛡️ 2. Firewall events 24h — co se reálně mitiguje

**Top 15 ASNs napříč všemi 4 doménami (sample 2 000 events):**

| ASN | Description | Events | % | Hodnocení |
|-----|-------------|--------|---|-----------|
| **32934** | **Facebook, Inc. (Meta)** | **1 484** | **74.2 %** | 🚨 **DOMINANTNÍ scraper** |
| 62044 | Zscaler Switzerland GmbH | 179 | 8.9 % | ⚠️ Enterprise proxy — review |
| 16509 | Amazon AWS | 75 | 3.8 % | ✅ Blokován |
| 24940 | Hetzner Online | 52 | 2.6 % | ✅ Blokován |
| 8075 | Microsoft | 25 | 1.2 % | ✅ Blokován |
| 15169 | Google LLC | 24 | 1.2 % | ✅ Blokován (non-bot) |
| 14618 | AWS | 21 | 1.1 % | ✅ Blokován |
| 45899 | VNPT Corp (VN) | 21 | 1.1 % | ✅ Geo block |
| 396982 | Google LLC | 16 | 0.8 % | ✅ Blokován |

**Akce breakdown:**
- `managed_challenge`: **1 938/2 000 = 96.9 %** ⚠️
- `block`: 48/2 000 = 2.4 %
- `link_maze_injected`: 14/2 000 = 0.7 % (CF AI Labyrinth feature)

**🚨 PROBLÉM**: 97 % požadavků dostává **managed_challenge** (= výzva, ne blok). Pokud bot challenge projde nebo ji ignoruje, vrátí se znova. Facebook ASN 32934 dostává challenge **1 484× za 24h** = bot pořád zkouší.

**🚨 PROBLÉM 2**: 74.8 % požadavků obsahuje `%7C` (pipe) v cestě = **URL exploration attack** (kombinace brand/category filterů). Bot zkouší `/eshop/dino%7Cschuring%7Csobinco%7Copeners-closers...` = pravděpodobně 7+ kombinatorické filtrace produktů.

---

## 🔍 3. Per-zone breakdown

### klicovecentrum.cz (500 events sample)
- Top ASN: **Zscaler 62044** (179 = 36%), AWS 16509 (15%)
- Top akce: managed_challenge (91%), block (7%)
- Top rule: Geo challenge datacentra (188), Bad ASNs (114), Scrapers (102)
- Top paths: `/favicon.ico` (148×!), `/vyroba-klice-podle-karty/` (53×)
- **Pozn**: zóna ma nejvíce variety = vícero útočníků různého původu

### klucovecentrum.sk (500 events sample)
- Top ASN: **Facebook 32934 (492 = 98.4%)**
- Top akce: managed_challenge (99.8%)
- Top paths: pipe-separated brand filtry (`%7C`)

### hbgroup.cz (500 events sample)
- Top ASN: **Facebook 32934 (492 = 98.4%)**
- Top akce: managed_challenge (99%)
- Top paths: pipe-separated filtry

### hbgroup.sk (500 events sample)
- Top ASN: **Facebook 32934 (472 = 94.4%)**
- Top akce: managed_challenge (98%), block (2%)
- Top paths: pipe-separated filtry

---

## 📈 4. GSC Crawl Stats (klicovecentrum.cz + klucovecentrum.sk)

### klicovecentrum.cz — last 10 dní
| Datum | Reqs | MB |
|-------|------|-----|
| 7.5. | 3 477 | 97 |
| 8.5. | 3 130 | 99 |
| **9.5.** | 3 757 | **876** ⚠️ |
| 10.5. | 3 241 | 80 |
| 11.5. | 3 588 | 111 |
| 12.5. | 3 572 | 145 |
| 13.5. | 2 996 | 179 |
| 14.5. | 2 860 | 62 |
| 15.5. | 2 973 | 102 |
| **16.5.** | 3 253 | **870** ⚠️ |

Pattern: ~3 000 reqs/den, 60-180 MB normal, ale 9. + 16.5. **>800 MB** (probably PDF / image batch crawl). Cloudflare neblokuje Googlebot.

### klucovecentrum.sk — last 10 dní
- Stabilně 1 800-2 200 reqs/den, 40-55 MB
- Response time 640-3 200 ms (vysoká variance = server pod load nebo komplexní queries)

### Bot type breakdown (oba sites)
- Smartphone Googlebot: 56-60 %
- Desktop: 29-33 %
- Image: 6-7 %
- AdsBot: 2 %
- Účel: **96 % Aktualizace** (refresh known URLs) vs 4 % Discovery → SEO stable

---

## 🎯 5. Doporučení — POSÍLIT současné nastavení

### 🔴 P0 — Hned (big wins)

#### 5.1 Upgrade managed_challenge → **BLOCK** pro Facebook ASN 32934
**Proč:** 74 % všech firewall events = Facebook ASN, dostává jen výzvu → vrací se. **Block ho zastaví na úrovni edge** (žádný CPU na origin).
**Riziko:** Žádné. Facebook ASN 32934 NENÍ legit user traffic (to je Meta interní cloud). Legit `facebookexternalhit` (link preview) má `cf.client.bot=true` → CF whitelist platí.
**Implementace:** Nová custom rule:
```
(ip.src.asnum eq 32934 and not cf.client.bot)
→ action: block
```

#### 5.2 Přidat **JP do geo block listu** (rule 4 "Geo blokování - bot traffic")
**Proč:** 1.15M requestů z JP za 24h = 18.5 % celkového traffic. **Žádný legit JP zákazník** — exportujeme zámky? Spíš ne. JP je vstupní brána Asia datacenter (AWS Tokyo, GCP asia-northeast, Tencent JP).
**Riziko:** Pokud máte japonské zákazníky → flagujete je. **Klient potvrdí**.
**Implementace:** Update rule 4 expression:
```
(ip.geoip.country in {"CN" "RU" "VN" "PK" "BD" "IQ" "ID" "TH" "PH" "NG" "EG" "TR" "JP"} and not cf.client.bot)
```

#### 5.3 **Block pipe-separated URL paths** (URL exploration attack)
**Proč:** 74.8 % events obsahuje `%7C` = bot zkouší kombinatorické brand/category filtry (`/eshop/dino%7Cschuring%7Csobinco%7C...`).
**Riziko:** Pokud frontend legitimně používá pipe-separated brand filtry → ztratíte funkce. **Klient potvrdí**.
**Implementace:** Nová custom rule:
```
(http.request.uri.path contains "%7C" or http.request.uri.path contains "|")
and not cf.client.bot
→ action: block
```

### 🟡 P1 — Tento týden

#### 5.4 Upgrade managed_challenge → **js_challenge** pro Bad ASN rule
**Proč:** js_challenge je harder bar než managed_challenge (vyžaduje plný JS engine s navigation events). Sofistikované scrapery (playwright, puppeteer) projdou managed, ne js_challenge.
**Riziko:** Pomalejší page load pro legit users co prošli managed_challenge cookies. Většinou žádný impact (cookies platí 30 min).

#### 5.5 Whitelist legit Facebook user-agent **PŘED** Facebook ASN block
**Pořadí pravidel:** Whitelist je první (rule 1), block druhý (rule 5). CF respektuje order.
```
Rule 1: (http.user_agent contains "facebookexternalhit" or http.user_agent contains "FacebookBot")
       → action: skip (allow)

Rule 5: (ip.src.asnum eq 32934 and not cf.client.bot)
       → action: block
```

#### 5.6 Review **Zscaler ASN 62044** (179 events na klicovecentrum.cz)
**Question pro klienta:** Máš enterprise B2B zákazníky co používají Zscaler proxy?
- **Pokud ano** → keep managed_challenge (zaktualizovat naše rule 2 → přidat Zscaler do allow list pokud user-agent vypadá legit)
- **Pokud ne** → block ASN 62044

### 🟢 P2 — Review

#### 5.7 `/favicon.ico` napadán 148× za 24h
**Proč:** Bot fingerprinting nebo error retry loop. Cache miss pravděpodobně.
**Akce:** Verify že favicon.ico má `Cache-Control: public, max-age=31536000` (1 rok) — eliminuje 99 % requests na origin.

#### 5.8 Per-IP rate limit (Free plan = 1 RL rule)
**Aktuálně:** Máme "Leaked credential check" jako jedinou RL rule (Free max 1).
**Trade-off:** Pokud upgradneme na Pro plan ($20/zóna = $80/měs pro 4 zóny), dostaneme 5 RL rules → můžeme přidat `> 100 reqs / 60s per IP → block`.

---

## 📥 6. Nový API token — co potřebuju pro real-time monitoring

Klient nabídl víc CF access ("nezdá se mi že je současné nastavení účinné. můžeme nastavit přístup"). Pojďme:

### Doporučený token scope

V Cloudflare → My Profile → API Tokens → Create Token → **Custom token**:

**Permissions:**
- Account → Account Analytics → **Read**
- Account → Cloudflare One Settings → **Read**
- Account → Logs → **Read** (real-time tail firewall events přes API)
- Zone → Zone WAF → **Edit** (rule changes)
- Zone → Bot Management → **Edit**
- Zone → Logs → **Read**
- Zone → Analytics → **Read**

**Zone Resources:** Include → All zones from account (nebo Specific zone × 4)
**TTL:** 30 dní (nebo Never expires)
**IP filtering:** ponechat prázdné

Po obdržení tokenu (pošli mi):
- Nastavím `cloudflare_firewall_pull.py` — automatický pull firewall events každých 6h
- Vytvořím `cloudflare_strengthen.py` který nasadí 3 P0 doporučení (s tvým schválením)
- Přidám Section 18 v marketing dashboardu o real-time CF stats (events/24h, top ASNs, top blocked countries)

---

## 📊 7. Souhrn — co se opravdu dělá vs co bychom měli

| Co | Současný stav | Doporučené nastavení |
|----|---------------|---------------------|
| Facebook ASN 32934 (Meta scraper) | `managed_challenge` 1 484× | **block** (1 rule) |
| JP geo (1.15M reqs/24h) | **prochází** (no rule) | **block** (přidat JP do rule 4) |
| Pipe-separated URL (URL exploration) | `managed_challenge` 1 495× | **block** (1 rule) |
| Zscaler ASN 62044 | `managed_challenge` 179× | Review (může být enterprise) |
| Bad ASN action | `managed_challenge` | Upgrade na `js_challenge` |
| /favicon.ico (148×/24h) | `managed_challenge` | Cache header fix (1-rok max-age) |
| Per-IP rate limit | Žádný (Free max 1 RL = leaked cred) | Pro plan ($80/měs) → 5 RL rules |

**Net effect doporučených změn**: ~80 % aktuálního traffic z bot ASNs by se blokoval na edge místo CF challenge (= žádný CPU origin load, žádné scraping úspěšné prochody).

---

## 🚀 8. Akční plán pro tebe

1. **Potvrdit JP block** (máš japonské zákazníky?) — ANO/NE
2. **Potvrdit pipe-separated path block** (legitimní pipe v URL frontendu?) — ANO/NE
3. **Posoudit Zscaler** (máš enterprise zákazníky přes Zscaler proxy?) — ANO/NE
4. **Vygenerovat nový API token** s permissions výše → pošli
5. Po obdržení tokenu já:
   - Nasadím 3 P0 doporučení (po tvém OK)
   - Spustím re-audit po 24h pro verify (acceptance test)
   - Přidám real-time CF monitoring sekci do marketing dashboardu
   - Setup automatický pull firewall events každých 6h → trend chart

**Očekávaný výsledek:**
- Bot traffic 4.5M → < 1M / 24h (eliminujeme Facebook + JP)
- managed_challenge ratio 97% → < 30% (block většina bad traffic na edge)
- Server load: ~3× nižší (méně challenge JS render na origin)
- SEO: žádný impact (Googlebot/Bingbot whitelisted přes `cf.client.bot`)
