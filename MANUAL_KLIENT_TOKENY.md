# 📋 MANUÁL pro klienta — získání pending tokenů a IDs

**Datum:** 13. 5. 2026
**Pro:** Ondřej Skřebský (klient)
**Cíl:** Step-by-step návod pro získání 9 pending tokenů/IDs/exportů které odblokují další sekce marketing dashboardu.

> ⏱️ **Celkový odhad času: 1.5-3 hodiny** (záleží jak rychle se prokliknu admin rozhraními).
>
> Můžeš to dělat **postupně** — každý zdroj odblokuje 1-2 sekce, dashboard se postupně rozsvítí.

---

## 🚦 Priority + odhad času

| # | Co | Priorita | Čas | Odblokuje |
|---|-----|----------|-----|-----------|
| 1 | Heuréka **Conversion Reports** API klíč | 🔴 P0 | 5 min | Section 5 srovnávače stats |
| 2 | Meta Marketing API token + Ad Account ID | 🔴 P0 | 10 min | Section 1 sub-tab Meta |
| 3 | 3 GA4 property IDs | 🔴 P0 | 5 min | Multi-domain GA4 buildy |
| 4 | klucovecentrum.sk Google Ads Customer ID | 🔴 P0 | 2 min | Section 1 SK kampaně |
| 5 | Search Console SA pro 3 nové properties | 🟡 P1 | 10 min | Section 2 SEO multi-domain |
| 6 | Bing Webmaster 3 nové sites | 🟡 P1 | 15 min | Section 2 SEO Bing |
| 7 | Cézar per-pobočka tržby export | 🟡 P1 | 30-60 min (s IT) | Section 9 pobočky |
| 8 | Master tabulka 17 prodejen | 🟡 P1 | 30 min | Firmy.cz import feed |
| 9 | Alza Partner CSV sample | 🟢 P2 | 5 min | Section 5 Alza tab |

**TL;DR rychlá cesta** (vyřeší 60 % během 30 minut):
1. Heuréka klíč (5 min)
2. Meta token (10 min)
3. 3 GA4 IDs (5 min)
4. klucovecentrum.sk Ads ID (2 min)
5. Alza CSV sample (5 min)

---

## 1️⃣ Heuréka Conversion Reports API klíč (🔴 P0, 5 min)

> **Pozor**: Toto je **JINÝ klíč** než ten co jsi mi poslal pro Overeno zákazníky. Heuréka má dvě nezávislá API.

### Postup

1. Otevři https://sluzby.heureka.cz
2. Login (admin email + heslo)
3. Nahoře vyber obchod **klicovecentrum.cz**
4. V levém menu najdi **„Měření konverzí"** (NE „Ověřeno zákazníky")
5. V této sekci hledej **„API klíč pro reporty"** nebo **„Reporting API"** nebo **„Conversion API key"**
6. Pokud klíč ještě neexistuje → **„Vygenerovat klíč"** / **„Generate key"**
7. **Zkopíruj klíč** (formát: 32 znaků hex string)
8. Pošli mi klíč

### Jak ověřím že funguje

Spustím test:
```bash
curl "https://api.heureka.group/v1/reports/conversions?date=2026-05-08" \
  -H "x-heureka-api-key: TVŮJ_KLÍČ"
```

Očekávám **HTTP 200** s JSON daty (per-produkt klikové, konverze, výnos).

### Pokud klíč neexistuje / nelze vygenerovat

- Možná jsi na **starém Heureka rozhraní** — zkus https://sluzby.heureka.cz nebo https://admin.heureka.cz
- Pokud ani tam → kontaktuj **Heureka support** s žádostí o aktivaci Conversion Reports API
- Alternativa: **manuální export CSV** z Heuréka admin (Reporty → Konverze → Export CSV) měsíčně

---

## 2️⃣ Meta Marketing API token (🔴 P0, 10 min)

### Postup

1. Otevři https://business.facebook.com
2. Vlevo dole **„Settings"** (ikona ozubeného kolečka)
3. **„System Users"** (vlevo v menu)
4. **„Add"** tlačítko vpravo nahoře
5. Vyplň:
   - **Name**: `dashboard-readonly`
   - **Role**: `Employee` (NE Admin — Admin má více práv než potřebujeme)
6. **„Create System User"**
7. Klikni na nově vytvořeného uživatele
8. **„Add Assets"** → **„Ad Accounts"** → vyber svůj Ad Account
9. Set role: **„View performance"** (NE „Manage" — read-only stačí)
10. **„Save Changes"**
11. Vrať se na detail uživatele → **„Generate New Token"**
12. **App**: vyber svojí Facebook app (nebo si ji vytvoř na https://developers.facebook.com)
13. **Permissions**: zaškrtni:
    - ✅ `ads_read`
    - ✅ `business_management`
14. **Token Type**: **„Never expires"** (System User token nemá expiraci)
15. **„Generate Token"**
16. **Zkopíruj token** (formát: 200+ znaků, začíná `EAA...`)

### Pošli mi 2 věci

```
Token: EAA...
Ad Account ID: act_XXXXXXXXX (najdeš v Settings → Ad Accounts)
```

### Pokud nemáš Facebook app

1. https://developers.facebook.com → **„My Apps"** → **„Create App"**
2. Type: **„Business"**
3. Name: `KC Dashboard`
4. Po vytvoření vrať se k bodu 12 výše

---

## 3️⃣ 3 GA4 property IDs (🔴 P0, 5 min)

### Postup

1. Otevři https://analytics.google.com
2. Vlevo dole **„Admin"** (ikona ozubeného kolečka)
3. Nahoře vidíš dropdown s aktuální property
4. **Otevři dropdown** → vidíš seznam všech property
5. Pro každou property si všimni **Property ID** (číslo pod jménem)

### Pošli mi 3 IDs

```
klicovecentrum.cz: 299992437 (mám) ✅
klucovecentrum.sk: ?
hbgroup.cz:        ?
hbgroup.sk:        ?
```

### Bonus — přidat SA jako Viewer (1 minuta každá)

V každé property:
1. **Admin** → **Property Access Management**
2. **„+"** vpravo nahoře → **„Add users"**
3. Email: `hbg-101@hbgroup-493608.iam.gserviceaccount.com`
4. Role: **Viewer**
5. **Save**

Po přidání mi řekni → otestuju API call pro každou property + spustím buildy.

---

## 4️⃣ klucovecentrum.sk Google Ads Customer ID (🔴 P0, 2 min)

### Postup

1. Otevři https://ads.google.com
2. Pokud máš více účtů → vlevo nahoře vidíš dropdown s aktuálním účtem
3. Přepni na **klucovecentrum.sk**
4. Vpravo nahoře (vedle ikony notifikací) vidíš **Customer ID** (formát: `XXX-XXX-XXXX`)
5. Zkopíruj ho

### Pošli mi

```
klucovecentrum.sk: XXX-XXX-XXXX
```

### Bonus — verify SA má access

Přepni na klucovecentrum.sk účet → **Settings** (vlevo dole) → **Access and security** → **Users**:
- Mělo by tam být `hbg-101@hbgroup-493608.iam.gserviceaccount.com` (stejné jako u klicovecentrum.cz)
- Pokud NE → **„+"** → Add user → email + Role: **Read only**

---

## 5️⃣ Search Console SA pro 3 nové properties (🟡 P1, 10 min)

Pro **každou** ze 3 nových domén (klucovecentrum.sk, hbgroup.cz, hbgroup.sk):

### Postup

1. Otevři https://search.google.com/search-console
2. Vlevo nahoře **dropdown property** → vyber doménu (např. `klucovecentrum.sk`)
3. Pokud doména **NENÍ v seznamu** → **„Add property"**:
   - Type: **„Domain"** (preferované — pokrývá všechny subdomény)
   - Domain: `klucovecentrum.sk`
   - Verify metoda: **DNS TXT record** (Cloudflare → DNS → Add record)
   - Po verifikaci → property je v Search Console
4. Vlevo dole **„Settings"** (ikona ozubeného kolečka)
5. **„Users and permissions"**
6. **„Add user"**:
   - Email: `hbg-101@hbgroup-493608.iam.gserviceaccount.com`
   - Permission: **„Restricted"** (read-only stačí pro API)
7. **„Add"**

### Pošli mi po dokončení

```
✅ klucovecentrum.sk přidaná do GSC + SA jako Restricted user
✅ hbgroup.cz přidaná do GSC + SA jako Restricted user
✅ hbgroup.sk přidaná do GSC + SA jako Restricted user
```

→ Otestuju API call pro každou + spustím SEO buildy.

---

## 6️⃣ Bing Webmaster 3 nové sites (🟡 P1, 15 min)

Pro **každou** ze 3 nových domén:

### Postup

1. Otevři https://www.bing.com/webmasters
2. Login (Microsoft account)
3. Pokud doména **NENÍ v seznamu**:
   - **„Add a site"** vpravo nahoře
   - Zadej `https://www.klucovecentrum.sk`
   - Verify (HTML tag, DNS, nebo Bing Webmaster file upload)
4. Po přidání použij **stejný API key** který už máme: `613abedd85e248d6a26067b14e77deee`
5. **Settings** → **API access** → ověř že key je active pro tuto site

### Pošli mi

```
✅ klucovecentrum.sk verified v Bing Webmaster
✅ hbgroup.cz verified v Bing Webmaster
✅ hbgroup.sk verified v Bing Webmaster
```

---

## 7️⃣ Cézar per-pobočka tržby export (🟡 P1, 30-60 min s IT)

> Toto je nejnáročnější — vyžaduje konzultaci s IT/účetní co nastavují Cézar reporty.

### Co potřebuji

Měsíční CSV export s tržbami **per provozovna** (17 fyzických prodejen). Idealní formát:

```csv
Datum;Provozovna;PocetUctenek;TrzbyCelkem;TrzbyHotovost;TrzbyKarta;TrzbyDobirka;PrumernaUctenka;Marze
2026-04-01;Praha 4 Modřany;145;87650.00;42100.00;39200.00;6350.00;604.48;26295.00
2026-04-01;Plzeň centrum;98;52340.00;28900.00;20100.00;3340.00;534.08;15702.00
...
```

### Co dělat s IT

1. Cézar má modul **„Maloobchod / Pokladna"** (nebo podobný název)
2. V tomto modulu lze udělat report **per středisko / provozovna** za měsíc
3. Export do **CSV UTF-8** s delimiter `;` (středník)

Otázky pro IT:
- Jak vypadá filtr „per provozovna" v Cézaru?
- Lze export naplánovat (cron) nebo musí klient klikat ručně každý měsíc?
- Jak přidat marži do exportu (pokud chybí, dopočítáme z products)

### Pošli mi

1. **Sample CSV** za 1 měsíc (třeba duben 2026) abychom viděli skutečný formát
2. **Workflow**: bude klient exportovat ručně 1× měsíčně, nebo IT to zautomatizuje?

Pokud sample CSV má jiné sloupce než můj návrh, **přizpůsobím parser**.

---

## 8️⃣ Master tabulka 17 prodejen (🟡 P1, 30 min)

Pro Firmy.cz import feed (a později pro GBP cross-reference) potřebuju master tabulku všech 17 prodejen.

### Co potřebuji

Soubor `data/raw/pobocky_master.csv` se sloupci:

```csv
id;name;ic;city;street;houseNumber;zip;lat;lng;email;phone;url;hours;description;logo_url;photo_url;filters
kc-01;Klíčové centrum Plzeň;26168685;Plzeň;Klatovská;105;30100;49.7361;13.3744;plzen@klicovecentrum.cz;+420377123456;https://klicovecentrum.cz;"Mo-Fr 08:00-18:00; Sa 09:00-12:00";Pobočka Plzeň - klíče cylindrické vložky trezory;https://...;https://...;"s-parkovistem,bezbarierove,platba-kartou"
```

### Postup

1. Otevři **`data/raw/pobocky_master.csv`** v repo (template tam už je se 4 sample řádky)
2. **Doplň zbylých 13 řádků** (kc-05 až kc-17) s reálnými údaji
3. Pro každou pobočku potřebuji:
   - **id** (unikátní, formát `kc-NN`)
   - **name** (oficiální název pobočky)
   - **ic** (26168685 pro celou H&B Group)
   - **city, street, houseNumber, zip** (adresa)
   - **lat, lng** (GPS — vyhledat v https://www.google.com/maps)
   - **email** (např. `plzen@klicovecentrum.cz`)
   - **phone** (formát `+420XXXXXXXXX`)
   - **url** (https://klicovecentrum.cz nebo per-pobočka URL pokud existuje)
   - **hours** (otvírací doba — OSM formát `Mo-Fr 08:00-18:00; Sa 09:00-12:00`)
   - **description** (max 300 znaků)
   - **logo_url** (URL na logo, ideálně CDN/web hosting)
   - **photo_url** (URL na hlavní foto pobočky)
   - **filters** (oddělené čárkou — povolené hodnoty: `s-parkovistem`, `bezbarierove`, `platba-kartou`, `na-splatky`, `klimatizovano`, `s-wifi`, `bezhotovostni-pobocka`, atd.)

### Pošli mi (2 možnosti)

**Možnost A**: Ty doplníš CSV → commit do repo:
```bash
cd C:/Users/PRA19-skrebsky/claude/analytics-audit
git add data/raw/pobocky_master.csv
git commit -m "Add 13 zbývajících prodejen do master tabulky"
git push
```

**Možnost B**: Pošleš mi data v jakémkoli formátu (Excel, Google Sheets link) a já doplním CSV za tebe.

---

## 9️⃣ Alza Partner CSV sample (🟢 P2, 5 min)

### Postup

1. Otevři https://partner.alza.cz
2. **Objednávky** → vyber rozsah (třeba posledních 7 dní)
3. **Export** → CSV
4. Pošli mi sample (může to být jen 5-10 řádků abys neporušoval GDPR)

### Co očekávám ve sloupcích (zjistím podle sample)

- DatumObjednavky
- CisloObjednavky
- ProduktSKU
- ProduktNazev
- Pocet
- CenaProdejni
- Provize (Alza commission)
- StavObjednavky

→ Po obdržení sample připravím parser + frekvenci uploadu (týdně manuálně, nebo Alza API pokud máš access).

---

## 📦 Bonus — Cloudflare leaked credential check (3× ~30s)

Zbývající úkol z Cloudflare replikace:

1. https://dash.cloudflare.com → **klucovecentrum.sk** → **Security** → **Settings**
2. Najdi **„Leaked credential checks"** (nebo „Leaked passwords")
3. Toggle **ON**
4. Opakuj pro **hbgroup.cz** a **hbgroup.sk**

Po dokončení mi řekni → re-audit + dashboard ukáže 8/8 rules na všech 4 doménách.

---

## ✅ Co dostaneš zpět

Pro každý dokončený task → tvůj marketing dashboard se rozsvítí novou sekcí. Aktualizace v real-time:

| Task | Sekce co se rozsvítí |
|------|----------------------|
| 1 (Heuréka API) | Section 5 Heuréka tab |
| 2 (Meta token) | Section 1 Meta sub-tab |
| 3 (GA4 IDs) | Section 0/2/3/4 multi-domain |
| 4 (Ads CZ) | Section 1 SK tab |
| 5 (GSC) | Section 2 SEO multi-domain |
| 6 (Bing) | Section 2 Bing multi-domain |
| 7 (Cézar pobočky) | Section 9 Pobočky |
| 8 (Master tabulka) | Firmy.cz auto-sync feed |
| 9 (Alza CSV) | Section 5 Alza tab |

## 💬 Kontakt

Pokud někde nejde uplatnit postup (rozhraní vypadá jinak, tlačítko chybí, error message), pošli mi screenshot — najdeme alternativu.

**Důležité**: lepší **nepřesný export rychle** než **perfektní export pozdě**. Datasety doplníme iterativně.
