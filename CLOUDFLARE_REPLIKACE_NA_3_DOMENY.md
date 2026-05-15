# 🔁 Cloudflare replikace pravidel na 3 nové domény

**Datum:** 12. 5. 2026
**Cíl:** Replikovat 5 custom rules + Bot Fight Mode + cache rule + page rule z `klicovecentrum.cz` na 3 nové domény (`klucovecentrum.sk`, `hbgroup.cz`, `hbgroup.sk`).

> Klient zařídil přidání domén do Cloudflare účtu (account ID `272b45252de5cb263a7f7151e53f5892`). Teď je potřeba zkopírovat ochrana ze stávající `klicovecentrum.cz` zóny na 3 nové.

---

## ⚡ Quick win — 2 metody

### Metoda A — UI (manuálně, ~30 min na doménu)
Pro každou ze 3 nových domén projít `CLOUDFLARE_MANUAL_FREE.md` → kroky 1-7.

### ✅ Metoda B — API skript (DOPORUČENO, klient poslal token 13.5.2026)

**Skript:** `cloudflare_replicate.py` (HOTOVO, čeká na klientovo spuštění)

```bash
# Dry-run preview (bez změn)
python cloudflare_replicate.py

# Apply replikace
python cloudflare_replicate.py --apply --yes
```

Detaily: viz `CLOUDFLARE_REPLIKACE_HOWTO.md` (kompletní návod krok-by-krok)

Token je uložen v `~/.cloudflare/cf_replicate_token` (mimo repo, .gitignore chrání).

---

## 🔧 Setup API tokenu (pokud klient zvolí Metodu B)

1. Cloudflare Dashboard → **My Profile** (avatar vpravo nahoře) → **API Tokens**
2. **Create Token** → **Custom token**
3. **Permissions**:
   - Zone → Zone Settings → Edit
   - Zone → Page Rules → Edit
   - Zone → Cache Rules → Edit
   - Zone → Firewall Services → Edit
   - Zone → Zone WAF → Edit
   - Zone → Bot Management → Edit (může se jmenovat „Bot Management Read" + Edit)
4. **Zone Resources**: Include → Specific zone → vyber **všechny 4 zóny** (klicovecentrum.cz, klucovecentrum.sk, hbgroup.cz, hbgroup.sk)
5. **TTL**: nastav na 7 dní (jen na replikaci, pak revoke)
6. **Continue to summary** → **Create Token**
7. Pošli token v safe channelu (ne git!)

Token uložím do `~/.cloudflare/cf_replicate_token` → spustím replikaci.

---

## 📋 Co konkrétně replikovat

### 1. Custom Rules (5/5)

| # | Název | Action | Filter expression | Events/24h (klicovecentrum.cz) |
|---|-------|--------|-------------------|--------------------------------|
| 1 | Block/Challenge Bad ASNs | Managed Challenge | `(ip.src.asnum in {16276 14618 16509 396982})` | 23 670 |
| 2 | Geo blokování | Block | `(ip.src.country in {CN RU VN PK BD IQ ID TH PH NG EG TR}) or (http.request.uri.path contains "/wp-admin") or (http.request.uri.path contains "/.env")` | 610 |
| 3 | Geo challenge datacentra | Managed Challenge | `(ip.src.country in {NL US DE}) and not (cf.client.bot)` | 189 |
| 4 | Block uploaded scripts | Block | `(http.request.uri.path contains "/upload") and (http.request.uri.path contains ".php")` | 0 |
| 5 | Block aggressive scrapers | Block | `(http.user_agent contains "MJ12bot") or (http.user_agent contains "AhrefsBot") or (http.user_agent contains "SemrushBot") or (http.user_agent contains "DotBot") or (http.user_agent contains "PetalBot") or (http.user_agent contains "Bytespider") or (http.user_agent eq "")` | (varies) |

### 2. Rate Limiting (1/1 — Free max)

| Název | Action | Match |
|-------|--------|-------|
| Leaked credential check | Block | (Cloudflare Managed) |

### 3. Cache Rules (1/10)

| Název | Action | Match |
|-------|--------|-------|
| Bypass authenticated paths | Bypass cache | `(http.request.uri.path contains "/sign") or (http.request.uri.path contains "/account") or (http.request.uri.path contains "/admin") or (http.request.uri.path contains "/kosik") or (http.request.uri.path contains "/objednavka")` |

### 4. Page Rules (1/3)

| URL | Setting |
|-----|---------|
| `*klicovecentrum.cz/*` (resp. `*hbgroup.cz/*` atd.) | Always Use HTTPS |

### 5. Settings (Security tab)

| Setting | Value |
|---------|-------|
| Security Level | High |
| Browser Integrity Check | ON |
| Challenge Passage | 1 hour |
| Privacy Pass | ON |

### 6. Bots tab

| Setting | Value |
|---------|-------|
| Bot Fight Mode | ON |

---

## 🌍 Důležité specifikum pro SK domény

Pro `klucovecentrum.sk` + `hbgroup.sk`:
- ❌ **NEBLOKOVAT SK!** Tvoji vlastní zákazníci by dostali 403.
- ✅ Block list zemí stejný jako CZ (CN/RU/VN/PK/BD/IQ + ID/TH/PH/NG/EG/TR)
- ⚠️ **NEBLOKOVAT AT/HU/PL** — sousedi SK, mohou být legit zákazníci

---

## ✅ Acceptance test po nasazení

Pro každou ze 3 nových domén:

```bash
# Test 1: bot UA má dostat 403
curl -A "MJ12bot/v1.4.8" -I https://www.klucovecentrum.sk/
# Očekávám: HTTP/2 403, header `cf-mitigated: challenge`

# Test 2: legitimní browser dostane 200
curl -A "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36" -I https://www.klucovecentrum.sk/
# Očekávám: HTTP/2 200

# Test 3: Googlebot whitelist (cf.client.bot)
curl -A "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)" -I https://www.klucovecentrum.sk/
# Očekávám: HTTP/2 200

# Test 4: blokovaná země (musíme přes proxy z CN, nebo simulovat curl --header)
curl -H "CF-IPCountry: CN" -I https://www.klucovecentrum.sk/
# Očekávám: HTTP/2 403 (rule 2)
```

Opakovat pro `hbgroup.cz` a `hbgroup.sk`.

---

## 💸 Náklady (Free vs Pro pro 4 zóny)

| Plan | Cena | Co dostaneš |
|------|------|-------------|
| **Free** (4 zóny) | $0 | 5 custom rules + 1 RL rule + Bot Fight Mode + Cache Rules — stačí pro 99 % klidných období |
| **Pro** (4 zóny) | 4× $20 = **$80/měsíc** | + `matches` regex, + 5 RL rules, + Super Bot Fight Mode, + Polish image opt |

**Doporučení**: Zůstat na Free dokud:
- Organická návštěvnost > 100k/měs na žádné z domén
- Není aktivní bot útok co Free pravidla neuchytí

---

## 🚦 Akční plán pro klienta

**TL;DR — 3 jednoduché kroky**:

1. **Bot Fight Mode + Security High** na 3 nových doménách (5 min total)
   - Pro každou doménu: Settings → Security → Security Level: **High** + Bots → **Bot Fight Mode ON**

2. **Custom Rules replika** — bud:
   - 🅰️ Klikat manuálně dle `CLOUDFLARE_MANUAL_FREE.md` (~30 min/doména)
   - 🅱️ Vygenerovat API token + poslat mi → udělám replikaci (~5 min total)

3. **Acceptance test** — projít curl testy výše

Po dokončení mi dej vědět → spustím další iteraci marketing dashboardu.
