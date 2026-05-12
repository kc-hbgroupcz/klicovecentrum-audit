# 🔐 Cloudflare Access — plná ochrana auditního dashboardu (15-30 min setup)

**Cíl**: Nahradit client-side password gate skutečnou enterprise-grade autentizací (Google login, email PIN, MFA).

**Free pro 50 uživatelů.** Plná ochrana proti přímému přístupu (nelze obejít disabling JS).

---

## Aktuální stav

- ✅ Dashboard má **client-side password gate** (heslo `KC-Audit-2026-HBG`, hash v dashboard.html)
- ❌ Stále public URL `https://kc-hbgroupcz.github.io/klicovecentrum-audit/dashboard.html` — kdokoliv se zná URL si může obejit JS prompt
- 🎯 Cloudflare Access nahradí tento gate **server-side ochrannou** s reálnou autentizací

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

## TL;DR — 3 kroky:

1. **DNS**: CNAME `audit.klicovecentrum.cz` → `kc-hbgroupcz.github.io` (Proxied, oranžový mrak)
2. **GitHub Pages**: Custom domain `audit.klicovecentrum.cz` v repo settings
3. **Cloudflare Access**: Application `audit.klicovecentrum.cz` + policy s allow listem emailů

Trvá ~15-30 min. Po propagaci DNS funguje login screen.

**Pak**: Odstranit `kc-hbgroupcz.github.io/klicovecentrum-audit/dashboard.html` z GitHub Pages? Ne — necháme ho jako fallback, ale s client-side password (`KC-Audit-2026-HBG`).

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
