# 🔐 Cloudflare Access — plná ochrana auditního dashboardu (15-30 min setup)

**Cíl**: Nahradit client-side password gate skutečnou enterprise-grade autentizací (Google login, email PIN, MFA).

**Free pro 50 uživatelů.** Plná ochrana proti přímému přístupu (nelze obejít disabling JS).

---

## Aktuální stav (12.5.2026)

- ✅ Client-side password gate **odstraněn** (klient používá Cloudflare Access)
- ✅ Logout tlačítko v dashboardu i rozcestníku → `/cdn-cgi/access/logout`
- ✅ Rozcestník struktura připravena:
  - `audit.klicovecentrum.cz/` — landing page (index.html)
  - `audit.klicovecentrum.cz/dashboard.html` — Audit dashboard (live)
  - `audit.klicovecentrum.cz/marketing/` — Marketing dashboard placeholder (Q3 2026)
  - `audit.klicovecentrum.cz/MARKETING_DASHBOARD_BRIEF_PRO_AGENTURU.md` — brief pro agenturu
- 🎯 Cloudflare Access chrání **celou doménu** `audit.klicovecentrum.cz/*` — 1 application policy stačí

---

## Předpoklady

- ✅ Cloudflare účet (klient má — pro klicovecentrum.cz)
- ✅ Doména pod Cloudflare DNS (klicovecentrum.cz nebo libovolná, kterou klient vlastní)
- ✅ Cloudflare Zero Trust dashboard přístup (free tier — 50 users)

---

## Krok 1 — Aktivovat Cloudflare Zero Trust (5 min)

1. Cloudflare Dashboard → vlevo dole **„Zero Trust"** (modré tlačítko)
2. Pokud first-time: **Choose plan** → **Free** (50 users) → Subscribe
3. **Set team name**: `hbgroup` (nebo libovolný — bude součástí URL)
4. **Authentication method** → **One-time PIN** (default, email-based) nebo **Google** OAuth pokud chcete

---

## Krok 2 — Vytvořit subdomain `audit.klicovecentrum.cz` (5 min)

GitHub Pages nedovoluje custom domain s password protection přímo. Musíme použít **Cloudflare Workers** nebo **CNAME → GitHub Pages** + **Cloudflare Access** mezi.

### Variant A — CNAME na GitHub Pages (nejjednodušší)

1. Cloudflare Dashboard → klicovecentrum.cz zóna → **DNS** → **+ Add record**
2. **Type**: `CNAME`
3. **Name**: `audit`
4. **Target**: `kc-hbgroupcz.github.io`
5. **Proxy status**: ✅ **Proxied** (oranžový mrak — důležité!)
6. **TTL**: Auto
7. **Save**

8. **GitHub repo settings** (https://github.com/kc-hbgroupcz/klicovecentrum-audit/settings/pages):
   - Custom domain: `audit.klicovecentrum.cz`
   - Save → vygeneruje se `CNAME` soubor v repo
   - **Enforce HTTPS**: ✅ ON (po DNS propagaci 5-30 min)

---

## Krok 3 — Nakonfigurovat Cloudflare Access policy (10 min)

1. Cloudflare → **Zero Trust** → **Access** → **Applications** → **+ Add an application**
2. **Self-hosted** application
3. **Application name**: `Audit klicovecentrum.cz`
4. **Session Duration**: `24 hours` (default, klient se nemusí znova logovat 24h)
5. **Application domain**: `audit.klicovecentrum.cz` (přesně jak jste nastavil v DNS)
6. **Identity providers**: vybrat **One-time PIN** (default) nebo **Google** pokud nakonfigurováno
7. **Next**

8. **Configure policies** → **+ Add a policy**:
   - **Policy name**: `Allowed users`
   - **Action**: `Allow`
   - **Configure rules**:
     - **Include** → **Emails** → list emailů, kteří mají přístup, např.:
       ```
       oskrebsky@gmail.com
       jan.novak@hbgroup.cz
       agentura.ppc@example.cz
       ```
     - Nebo **Include** → **Emails ending in** → `@hbgroup.cz` (pokud chcete pustit celou doménu)
9. **Next** → **Add application**

---

## Krok 4 — Test (2 min)

1. Otevřete https://audit.klicovecentrum.cz/dashboard.html v inkognito
2. Měli byste vidět **Cloudflare login screen**:
   - „Sign in with PIN" → zadat email → email s 6-digit kódem
3. Po validaci kódu → přesměrování na dashboard

---

## Krok 5 — Odstranit client-side password gate (volitelné, po verify Access funguje)

Pokud Cloudflare Access funguje, můžete odebrat JS gate v dashboard.html (sekce mezi `<!-- PASSWORD GATE -->` markers). Není to nutné — jen ušetříte ~3 KB JS.

Postup:
1. V dashboard.html odstranit script blok mezi `// ═══ PASSWORD GATE` a `})();`
2. Commit + push

Nebo nechat jako **2 vrstvy ochrany** (Cloudflare Access + client-side gate) — bezpečnější ale více klikání.

---

## Co Cloudflare Access dělá (vs client-side gate)

| Feature | Client-side (současné) | Cloudflare Access |
|---------|------------------------|-------------------|
| Bezpečnost | ⚠️ Lze obejít disabling JS / view-source | ✅ Server-side ochrana, nelze obejít |
| Login UX | Single password všichni sdílí | Per-user email/Google login |
| Audit log | ❌ žádný | ✅ Cloudflare loguje kdo a kdy přistoupil |
| MFA | ❌ ne | ✅ Lze přidat (Google MFA, FIDO2) |
| Revoke přístup | Změnit password + redeploy všech | ✅ Smazat email z policy (instant) |
| Cena | Free | Free (50 users), pak $7/user/měs |

---

## Cloudflare Access — alternativy autentizace

V kroku 3 → **Identity providers** vyberete:

### One-time PIN (default, email)
- Klient napíše email → dostane 6-digit PIN → vloží
- ✅ Žádný setup, funguje hned
- ❌ Vyžaduje přístup k emailu při každém loginu (ne při 24h session)

### Google OAuth
- Klient klikne „Sign in with Google" → Google login flow
- ✅ Zero friction (1 klik)
- ❌ Vyžaduje Google Workspace nebo Gmail
- Setup: Cloudflare Zero Trust → Settings → Authentication → + Add → Google

### GitHub
- Stejný principle, Sign in with GitHub
- Hodí se pro tech tým

### SAML / OIDC
- Pokud klient používá Microsoft Entra ID (Azure AD), Okta, atd.

---

## Backup plán: Cloudflare Worker reverse proxy (pokud GitHub Pages CNAME nefunguje)

Pokud GitHub Pages odmítne `audit.klicovecentrum.cz` (např. kvůli chybějícímu CNAME souboru), použít **Cloudflare Worker** jako proxy:

```javascript
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request));
});

async function handleRequest(request) {
  const url = new URL(request.url);
  url.hostname = 'kc-hbgroupcz.github.io';
  url.pathname = '/klicovecentrum-audit' + url.pathname;
  return fetch(url.toString(), request);
}
```

Worker přidat na route `audit.klicovecentrum.cz/*` + Cloudflare Access policy se použije **před** Workerem.

---

## 🔓 Logout (odhlášení z Cloudflare Access)

Cloudflare Access má vestavěné 2 logout endpointy:

### Per-application logout (doporučeno)
```
https://audit.klicovecentrum.cz/cdn-cgi/access/logout
```
- Smaže `CF_Authorization` cookie pro tuto konkrétní aplikaci
- Po logout redirect na login screen
- ✅ **Logout button v dashboardu** (vpravo nahoře v hlavičce, „🔓 Odhlásit") posílá uživatele právě na tento URL

### Team-wide logout (odhlásí se ze VŠECH Access aplikací)
```
https://hbgroup.cloudflareaccess.com/cdn-cgi/access/logout
```
(nahraď `hbgroup` skutečným team name z Cloudflare Zero Trust)

### Session timeout (auto-logout)
- Default: **24 hodin** session duration
- Lze změnit v Cloudflare Zero Trust → Access → Applications → tvoje aplikace → **Session Duration**: 1h / 6h / 24h / 7 days / 30 days / Never

### Force logout všech uživatelů (revoke session)
Cloudflare Zero Trust → Logs → Access → vyber session → **Revoke**.
Nebo bulk: Settings → **Authentication** → **Revoke all sessions**.

---

## TL;DR — 3 kroky:

1. **DNS**: CNAME `audit.klicovecentrum.cz` → `kc-hbgroupcz.github.io` (Proxied, oranžový mrak)
2. **GitHub Pages**: Custom domain `audit.klicovecentrum.cz` v repo settings
3. **Cloudflare Access**: Application `audit.klicovecentrum.cz` (wildcard `*` v Path) + policy s allow listem emailů

Trvá ~15-30 min. Po propagaci DNS funguje login screen.

### Application path coverage

V Cloudflare Access → Application configuration → **Application domain**:
- **Subdomain**: `audit`
- **Domain**: `klicovecentrum.cz`
- **Path**: ponechat **prázdné** = chrání všechno (root + /dashboard.html + /marketing/ + /data/*)

Pokud byste chtěl rozdělit ochranu (např. jiná policy pro `/marketing/` než pro audit), vytvořit 2 applications:
1. App `audit.klicovecentrum.cz` (path: `/marketing/*`) → policy pro marketing tým
2. App `audit.klicovecentrum.cz` (path: ostatní) → policy pro audit tým

### Struktura rozcestníku (od 12.5.2026)

```
audit.klicovecentrum.cz/
├── index.html                                    ← rozcestník (3 karty)
├── dashboard.html                                ← Audit dashboard (LIVE)
├── marketing/
│   └── index.html                                ← placeholder „V přípravě"
├── MARKETING_DASHBOARD_BRIEF_PRO_AGENTURU.md     ← brief pro agenturu
├── CLOUDFLARE_ACCESS.md                          ← tento dokument
├── data/                                         ← JSON datasety
└── CNAME                                         ← audit.klicovecentrum.cz
```

**Logout tlačítko** je v rozcestníku (vpravo nahoře) i v dashboard.html → posílá na `/cdn-cgi/access/logout`.

**Fallback**: `kc-hbgroupcz.github.io/klicovecentrum-audit/dashboard.html` — ponecháno bez ochrany pro případ, že Cloudflare Access selže (klient by neměl tuto URL sdílet).

---

## Změna client-side hesla (pokud chcete dočasně udržet)

V `dashboard.html` najít:
```javascript
const _PWD_HASH = '87a86706be2be5477a77392f44376413f5b379f93f8266fc7529f3f229e796a2';
```

Vygenerovat nový hash:

**Linux/Mac:**
```bash
echo -n "NoveHeslo123" | sha256sum
```

**PowerShell:**
```powershell
$bytes = [System.Text.Encoding]::UTF8.GetBytes("NoveHeslo123")
$hash = [System.Security.Cryptography.SHA256]::Create().ComputeHash($bytes)
[System.BitConverter]::ToString($hash).Replace("-","").ToLower()
```

**Online** (méně bezpečné — nezadávat citlivá hesla): https://emn178.github.io/online-tools/sha256.html

Nahradit hash v dashboard.html → commit + push.
