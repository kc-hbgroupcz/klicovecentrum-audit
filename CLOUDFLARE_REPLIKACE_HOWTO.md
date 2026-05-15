# 🔁 Cloudflare replikace — instrukce pro spuštění

**Datum:** 13. 5. 2026
**Cíl:** Nasadit 5 custom rules + Bot Fight Mode + security settings z `klicovecentrum.cz` na 3 nové domény (`klucovecentrum.sk`, `hbgroup.cz`, `hbgroup.sk`).

> Token klient poslal, je uložen lokálně v `~/.cloudflare/cf_replicate_token` (mimo repo, NIKDY nebude v gitu).

---

## ⚡ TL;DR — 3 příkazy

```bash
# 1. DRY-RUN — preview co se replikuje, ŽÁDNÉ změny
cd /Users/PRA19-skrebsky/claude/analytics-audit
python cloudflare_replicate.py

# 2. APPLY s prompty (každá akce vyžaduje y/N)
python cloudflare_replicate.py --apply

# 3. APPLY bez promptů (auto-confirm všeho)
python cloudflare_replicate.py --apply --yes
```

---

## 🔐 Bezpečnostní setup (jednou)

Token už je uložen v `~/.cloudflare/cf_replicate_token`. Pokud bys ho potřeboval znovu nastavit:

```bash
mkdir -p ~/.cloudflare
echo "<TVŮJ_CF_TOKEN>" > ~/.cloudflare/cf_replicate_token   # nikdy necommitovat!
chmod 600 ~/.cloudflare/cf_replicate_token
```

`.gitignore` už chrání `*.token`, `*_token`, `.cloudflare/`, `secrets/`.

---

## 📋 Co skript dělá

### Fáze 1 — Verifikace (read-only, bezpečné)
1. Verifikuje token (volá `/user/tokens/verify`)
2. Listuje 4 zóny v účtu (klicovecentrum.cz, klucovecentrum.sk, hbgroup.cz, hbgroup.sk)
3. Načte z `klicovecentrum.cz`:
   - Custom firewall rules (5 ks)
   - Settings: security_level, browser_check, challenge_ttl, always_use_https
   - Bot Fight Mode

### Fáze 2 — DRY-RUN preview
- Pro každou ze 3 cílových zón **vypíše** co by se replikovalo
- **ŽÁDNÉ změny** se neprovedou
- Idempotency check: rule s identickým `description` se přeskočí

### Fáze 3 — APPLY (jen s `--apply` flagem)
- Pro každou cílovou zónu:
  1. Settings (security_level=high, BFM=on, …) — `PATCH /zones/{id}/settings/...`
  2. Custom rules — `POST /zones/{id}/rulesets/{id}/rules`
- Mezi requesty 0.5s pause (rate limit safety)

---

## 🔍 Co token dovoluje

Klient vytvořil token s těmito permissions (předpoklad — verifikuje to skript):
- ✅ Zone Settings: Read + Edit
- ✅ Firewall Services: Read + Edit
- ✅ Cache Rules: Read + Edit
- ✅ Bot Management: Read + Edit
- ✅ Page Rules: Read + Edit

Pokud token nemá některou z permissions, dostaneš v dry-run **403** u dané akce — buď klient přidá permission, nebo přeskočíme.

---

## ✅ Acceptance test (po `--apply`)

Skript na konci vypíše curl příkazy. Spustit pro každou ze 3 nových domén:

```bash
# Test 1: bot UA má dostat 403 (rule 5 — Block aggressive scrapers)
curl -A "MJ12bot/v1.4.8" -I https://www.klucovecentrum.sk/
# Očekávám: HTTP/2 403

# Test 2: legit browser → 200
curl -A "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36" -I https://www.klucovecentrum.sk/
# Očekávám: HTTP/2 200

# Test 3: Googlebot whitelist (rule expression má `not cf.client.bot`)
curl -A "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)" -I https://www.klucovecentrum.sk/
# Očekávám: HTTP/2 200

# Opakovat pro hbgroup.cz a hbgroup.sk
```

---

## 🌍 Důležité specifikum pro SK domény

**Geo block list klicovecentrum.cz**:
```
CN, RU, VN, PK, BD, IQ, ID, TH, PH, NG, EG, TR
```

✅ **SK NENÍ v listu** — slovenští zákazníci nebudou zablokováni.
✅ **AT/HU/PL/CZ NEJSOU v listu** — sousedi SK fungují.

Pokud by se v budoucnu objevily fake nákupy z SK, můžeme přidat **rate limiting per IP** (Free plán umí 1 RL rule).

---

## 🔄 Rollback (pokud něco selže)

Custom rules se ukládají s `description` markerem — můžeš je smazat manuálně v UI:

1. Cloudflare Dashboard → vyber zónu → Security → WAF → Custom rules
2. Najdi rule podle description (např. „Block aggressive scrapers + datacenter ASNs")
3. Delete

Settings (security_level, BFM) se vrátí na default přes UI:
- Settings → Security → Security Level: dropdown
- Bots → Bot Fight Mode: toggle

---

## 🚦 Po skončení

1. Pošli mi výstup `python cloudflare_replicate.py` (dry-run) — uvidíme co plánuje
2. Pak `python cloudflare_replicate.py --apply --yes` — replikace
3. Acceptance curl testy
4. **Smaž token** (jakmile skončím): `rm ~/.cloudflare/cf_replicate_token`
   - Nebo nech token až vyprší (klient nastavil 7-day TTL)
5. **Revoke token** v Cloudflare admin (Profile → API Tokens → Roll/Delete)

---

## ❓ Otázky / problémy

Pokud:
- **403** v dry-run → token nemá permission, doplnit
- **404** zone not found → klient ji ještě nepřidal do CF
- **429** rate limited → skript má built-in 0.5s sleep, ale Cloudflare API limity jsou ~1200 req/5min — pokud bys to spustil mnohokrát, počkej

Napiš mi co viděl, vyřešíme.
