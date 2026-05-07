# Závěrečná zpráva auditu analytiky a PPC kampaní
**Klient:** klicovecentrum.cz  
**Zpracoval:** Ondřej Skřebský (vlastník)  
**Datum zprávy:** 21. dubna 2026  
**Auditované období:** říjen 2025 – duben 2026  
**Auditované kanály:** GA4 · Search Console · Google Ads · Sklik (drak + Fénix API) · Merchant Center · Google Business Profile · Heuréka · Zboží.cz · Firmy.cz (14 poboček) · Mergado (produktový feed) · ERP data

---

## EXECUTIVE SUMMARY

Audit odhalil **systémové selhání** v práci agentury na čtyřech úrovních:

1. **Měření je nefunkční.** Konverze jsou **265× nafouklé** — agentura reportuje 370 870 konverzí, skutečných nákupů je **1 398** (SQL DB ground truth; GA4 zachytil pouze 641, tj. **46 % reality** — gap 54 % způsobený mj. payment outage #48). Veškeré optimalizace PPC kampaní probíhají na základě falešných dat. Cross-reference s ERP prokázala, že v březnu 2026 GA4 nafoukl revenue o 54 % oproti skutečnosti (bot traffic spouštěl falešné `purchase` eventy).

2. **Rozpočet teče mimo cíl.** 32 % Google Ads rozpočtu (~74 200 Kč ze 232 632 Kč) míří do krajů bez jediné pobočky klicovecentrum.cz. **Sklik účet celkově ztrátový (ROAS 0,92×, PNO 109 %)** — obsahová síť spálila 12 099 Kč čisté ztráty, Zboží.cz 25 886 Kč na produkty bez konverze. Paradoxně vyhledávací síť Sklik je mírně profitabilní (ROAS 1,24×) ale **92,5 % zobrazení ztraceno kvůli nedostatečnému rozpočtu** — peníze leží na stole. Heuréka ROAS 1,44× — ztrátová v 5 ze 7 měsíců.

3. **Produktový feed je odpojený od reality.** 80 % produktů ve feedu (1 746 z 2 175) za 7 měsíců neprodalo ani kus. Zároveň **45 % obratu e-shopu (527 118 Kč) generují produkty, které v placených kanálech vůbec nejsou** — PPC k těmto zákazníkům nemá přístup.

4. **Bezpečnostní incident nebyl řešen.** Bot útok v březnu 2026 (~1,5 milionu falešných sessions ze zahraničí) nebyl agenturou detekován ani mitigován. Data za celý měsíc jsou nepoužitelná. Cross-reference s Firmy.cz (+37 % v březnu) potvrzuje, že reálný provoz naopak rostl — bot inflace postihla jen Direct v GA4.

5. **Agenturní reporty jsou z 75 % prázdné.** Přes Sklik Fénix API jsou k dispozici měsíční reporty agentury za únor a březen 2026. Ze čtyř typů (produktové inzeráty, vyhledávací síť, RTG, obsahová síť) jsou **tři úplně prázdné** — 0 zobrazení, 0 kliků. Agentura reportuje „sledování všech Sklik kanálů", fakticky sleduje jen obsahovou síť.

**Celkové hodnocení práce agentury: nedostatečné.** Základní úkol — správné měření výsledků, kurátorstvo produktového feedu a efektivní nasazení rozpočtu — není plněn. **E-shop jako byznys funguje dobře** (ERP obrat 1 170 742 Kč, marže 59,1 %); problém je v PPC a měření.

### Strategická poznámka — per-kampaň analýza ukázala zlaté žíly

Per-kampaň rozpad (viz sekce 12) odhalil **tři masivní ztracené příležitosti**:

1. **Sklik „0 VG Brand" search: ROAS 23,89× při 3,4 % rozpočtu** — agentura dává 1 694 Kč do kampaně, která vrací 40 468 Kč. Kdyby tuto kampaň 10× navýšili, je to čistý zisk.
2. **Google Ads LOCAL PMax kampaně: ROAS 7,58×** (Plzeň-Lochotín dokonce 13,22×), ale chybí pro 13+ poboček — v minulosti (2018) existovaly, agentura je smazala.
3. **Sklik PRODUCT + LOCAL kampaně mají 0 konverzí** kvůli nefunkčnímu OCM trackingu — 44 k Kč (88 % Sklik rozpočtu) se nedá vyhodnotit.

Agentura hodnotí kampaně jako jeden celek přes ROAS. **Bez rozdělení na 3 kategorie (PRODUCT / BRAND / LOCAL) a opravy trackingu nelze optimalizovat** — ale data z Google Ads ukazují jasně, že LOCAL a BRAND jsou nejziskovější kanály a jsou podfinancované.

### Vlastnictví nápravy — kdo má co opravit (rollup 40 nálezů)

Každý nález má označen primárního vlastníka nápravy (badge v hlavičce). Souhrn:

| Vlastník | Počet nálezů | Co dělá | Priorita akcí |
|----------|--------------|---------|---------------|
| 🎯 **Agentura** (marketing) | **13** | Admin v Google Ads / Sklik / GA4 / GTM, optimalizace kampaní, nastavení bot filtru, UTM audit | Většina P0 nálezů (#1, #2, #5-#10c) — bez agentury se hnout nelze |
| 🛠️ **Vývojář eshopu** | **7** | Code change: payment 500 fix (#42, #48), URL canonical/301 (#45), feed varianty (#11, #15), Alza→ERP API (#26), schema/title/meta (#4) | P0: #42 (~261 k Kč/rok ztráta), #45 (URL exploze), #44 (PHPMailer CVE) |
| ⚙️ **DevOps / správce serveru** | **2** | Server config: zombie subdomain odebrat (#43), PHPMailer smaz + audit (#44) | P0: oba — security + crawl budget |
| 👤 **Klient sám** | **8** | Manuální admin: Firmy.cz duplikát (#18), GBP profily (#12, #19), Alza katalog (#27-30), GTM access (#5) | Většinou P1 — drobné akce, rychlá realizace |
| 🔀 **Mixed** (koordinace) | **5** | Vyžadují agenturu + vývojáře/devops: OCM tracking (#10c, #13, #17), feed strukturálně odpojen (#14, #23) | P0: zejména #14 (45 % obratu mimo PPC) |
| ℹ️ **Informativní** | **5** | Pozitivní důkazy / observační nálezy bez okamžité akce: #20 Firmy.cz neulízaná botem, #29 Alza ceny OK, #46-49 Bing vlny + Q4 chronologie | Žádná akce — důkazní materiál pro #2, #14, atd. |

**Pro klienta — interpretace tabulky:** Před začátkem realizace vyber **nálezy podle vlastníka, ne podle priority** — tím poznáš, co máš v rukou ty (8 klientských úkonů, převážně P1) vs. co musíš delegovat na agenturu / vývojáře. **Nejvíc bottlenecků je u vývojáře** (7 nálezů, z toho 3 KRITICKÉ s přímým finančním dopadem 261 k Kč/rok).

V interaktivním dashboardu (https://kc-hbgroupcz.github.io/klicovecentrum-audit/dashboard.html) lze filtrovat nálezy podle vlastníka (`Kdo to opraví:` filtr v horní liště).

---

## 1. MĚŘENÍ A ANALYTIKA (GA4)

### 🔴 #1 — Konverze nafouklé 265× (KRITICKÉ)

**Vlastník:** 🎯 Agentura

Agentura označila jako primární konverze micro-eventy, které nepředstavují žádný obchodní výsledek:

| Event označen jako „konverze" | Počet | Realita |
|-------------------------------|-------|---------|
| time_on_web-180seconds | 144 832 | Uživatel strávil 3 min na webu |
| view_item | 126 437 | Zobrazení produktové stránky |
| time_on_web-360seconds | 33 187 | 6 minut na webu |
| time_on_web-540seconds | 27 248 | 9 minut na webu |
| time_on_web-720seconds | 23 952 | 12 minut na webu |
| begin_checkout | 6 698 | Zahájení objednávky (ne transakce) |
| add_to_cart | 5 095 | Přidání do košíku |
| file_download | 2 780 | Stažení souboru |
| **purchase (GA4 zachycených)** | **641** | GA4 capture rate jen 46 % reality |
| **purchase (SQL ground truth)** | **1 398** | ✅ Skutečné nákupy z DB B2C eshopu |
| **CELKEM agenturou hlášeno** | **370 870** | 265× nafouklé vs SQL (578× vs GA4) |

**Dopad na PPC kampaně:**
- Google Ads vidí 71 128 konverzí z 32 439 kliků → „konverzní poměr" 219 % (fyzicky nemožné)
- Smart Bidding se učí optimalizovat na uživatele, kteří „strávili 3 minuty na webu" — ne na ty, kteří nakoupili
- Agentura může klientovi ukazovat jakékoli číslo a vydávat ho za úspěch

**Skutečná čísla (purchase):**

| Měsíc | Revenue | Počet objednávek |
|-------|---------|-----------------|
| Říjen 2025 | 259 305 Kč | — |
| Listopad 2025 | 106 866 Kč | — |
| Prosinec 2025 | 103 118 Kč | — |
| Leden 2026 | 118 836 Kč | — |
| Únor 2026 | 150 036 Kč | — |
| Celkem (6M) | **984 387 Kč** | **641 objednávek** |

Průměrná hodnota objednávky: **1 535 Kč**

---

### 🔴 #2 — Bot útok v březnu 2026 nebyl detekován ani řešen (KRITICKÉ)

**Vlastník:** 🎯 Agentura

V období 7.–21. března 2026 přišlo ~1,5 milionu falešných sessions. Agentura na útok nereagovala — data nebyla filtrována.

**Charakteristika bot provozu:**

| Země | Sessions | Bounce rate | Průměrná doba |
|------|----------|-------------|---------------|
| USA | 476 888 | 92,4 % | 5,2 s |
| Německo | 209 676 | 92,0 % | 4,7 s |
| Hong Kong | 198 205 | 91,4 % | 4,4 s |
| Singapur | 192 912 | 91,3 % | 4,3 s |
| Japonsko | 184 681 | 92,0 % | 4,5 s |
| Vietnam | 154 629 | 93,3 % | 4,0 s |
| **Česká republika** | **111 398** | **2,5 %** | **303,6 s** |

Česká republika je jediná země s normálním chováním (bounce 2,5 %, 5 minut průměrně). Veškerý zahraniční provoz jsou boti.

Špičkové dny: 16. 3. 2026 — **253 711 sessions za jediný den** (normální den: ~2 000–3 000 sessions).

**Co nebylo nastaveno:**
- Bot filtering v GA4 (Admin → Data Filters)
- IP filter pro interní provoz
- Upozornění na traffic anomálie

---

### 🔴 #3 — Direct traffic 90 % z celku (KRITICKÉ)

**Vlastník:** 🎯 Agentura

GA4 ukazuje 1 796 846 sessions jako „Direct" — 90 % veškerého provozu. Norma pro srovnatelný web je 20–35 %.

**Příčiny:**
1. Bot provoz přichází bez referer → GA4 klasifikuje jako Direct
2. Chybějící nebo špatné UTM parametry na PPC kampaních
3. Pravděpodobné self-referral problémy (interní přesměrování bez konfigurace)
4. Interní provoz zaměstnanců a agentury není filtrován

**Dopad:** Atribuční data jsou bezcenná. Nelze zjistit, který kanál generuje skutečné zákazníky.

---

## 2. VYHLEDÁVÁNÍ (Google Search Console)

### 🟡 #4 — Nízké CTR na klíčových stránkách

**Vlastník:** 🛠️ Vývojář

**Celkový výkon (říjen 2025 – duben 2026):**
- 2 866 943 impresí → 53 790 kliků → **CTR 1,88 %**
- Průměrná pozice: 7,9

**Stránky s největší ztrátou kliků:**

| URL | Imprese | CTR | Pozice | Odhadovaná ztráta |
|-----|---------|-----|--------|-------------------|
| /cylindricke-vlozky/ | 167 151 | **0,6 %** | 7,1 | ~2 500 kliků/měs. |
| /trezory/ | 98 135 | **0,4 %** | 11,9 | ~1 200 kliků/měs. |
| /dverni-kovani/ | 81 849 | **0,5 %** | 10,4 | ~900 kliků/měs. |
| /navod-jak-urcit-orientaci-dveri/ | 67 884 | **0,5 %** | **4,2** | ~800 kliků/měs. |
| /zadlabaci-zamky/ | 77 090 | 0,9 % | 7,7 | ~700 kliků/měs. |

Stránka `/navod-jak-urcit-orientaci-dveri/` je na **4. místě** ve výsledcích, ale CTR je pouze 0,5 % — to je abnormální a signalizuje špatný title tag nebo meta description.

**Příčina:** Generické nebo duplicitní title tagy, chybějící schema.org rich snippets.

---

## 3. GOOGLE ADS — KAMPANĚ

### 🔴 #5 — Kampaně optimalizují na falešné konverze (KRITICKÉ)

**Vlastník:** 🎯 Agentura

Smart Bidding v Google Ads pracuje s importovanými konverzemi z GA4 — tedy s nafouklými čísly z nálezu #1. Systém se naučil cílit na uživatele, kteří „zobrazili produkt" nebo „strávili 3 minuty na webu".

**Implikace:** Reálná efektivita kampaní je neznámá. Jakákoli data o ROAS nebo CPA z Google Ads jsou pro toto období nepoužitelná.

### 🔴 #6 — 32 % rozpočtu v krajích bez poboček (KRITICKÉ)

**Vlastník:** 🎯 Agentura

> **Pozn. k revizi 6. 5. 2026:** Původní hodnota uváděla 61 % / 141 008 Kč na základě chybného mapování ID krajů (skript `geo_verify.py` opravil mapping; data z `data/regions.json`, period 2025-10-01 → 2026-04-21). Revidovaná čísla níže.

Klíčový nález z analýzy nastavení kampaní: **74 208 Kč** (31,9 % celkového Google Ads rozpočtu **232 632 Kč**) bylo utraceno v regionech, kde klicovecentrum.cz nemá žádnou kamennou prodejnu.

**Geografické rozložení výdajů (revidováno):**

| Kraj / Region | Útrata (Kč) | Podíl | Pobočka KC? |
|---------------|------------|-------|-------------|
| Středočeský | 51 627 | 22,2 % | ❌ **Žádná** |
| Ústecký | 42 914 | 18,4 % | ✅ Chomutov |
| Jihomoravský | 40 882 | 17,6 % | ✅ Brno |
| Plzeňský | 36 134 | 15,5 % | ✅ Plzeň (Gerská, Žatecká, Lochotín) |
| Ostatní | ~61 075 | ~26,3 % | mix |

**Spádové vs. mimo spádové (revize z `regions.json`):**

| Kategorie | Útrata (Kč) | Podíl rozpočtu | Konverze | CPA |
|-----------|------------|----------------|----------|-----|
| **S pobočkou** | 158 424 | **68,1 %** | 1 563 | 101 Kč |
| **Bez pobočky** | 74 208 | **31,9 %** | 867 | 86 Kč |
| Celkem | 232 632 | 100 % | 2 431 | 96 Kč |

Kampaně nejsou omezeny na spádové oblasti poboček. Společnost s 22 kamennými prodejnami platí za kliknutí z regionů, kam zákazník nemůže přijet (~74 200 Kč za 7 měsíců). Paradoxně mimo-spádové regiony mají **lepší CPA** (86 Kč vs 101 Kč) — naznačuje, že online-only kupující generují více konverzí na klik než lokální zákazníci, kteří preferují fyzickou návštěvu prodejny.

### 🟡 #7 — Connected TV: 2 662 Kč za 0 kliků

**Vlastník:** 🎯 Agentura

Performance Max automaticky rozšířil kampaně na Connected TV (SmartTV, Chromecast, atd.). Výsledek za sledované období:
- Útrata: **2 662 Kč**
- Kliků: **0**
- Konverzí: **0**

Zařízení nebylo vyloučeno, ačkoli pro e-shop s fyzickými produkty nemá SmartTV žádný smysl.

### 🟡 #8 — Negative keywords: 3 059 termínů, téměř žádné aktivní

**Vlastník:** 🎯 Agentura

Z analýzy negativních klíčových slov:
- Celkem 3 059 negative KW v účtu
- Drtivá většina je přiřazena k **pozastaveným nebo odstraněným kampaním**
- Performance Max kampaně nemají **žádné efektivní negative KW** — standardní sdílené listy nelze přímo aplikovat na PMax

**Dopad:** PMax kampaně pravděpodobně zobrazuje reklamy na irelevantní dotazy bez jakéhokoli filtrování.

### 🟡 #9 — Produktové Search kampaně chybí od roku 2018

**Vlastník:** 🎯 Agentura

Analýza historických kampaní ukazuje, že specializované search kampaně na produktové kategorie (cylindrické vložky, trezory, zámky) nejsou aktivní. Veškerý placený search provoz jde přes obecné kampaně nebo PMax.

---

## 4. SKLIK

### 🔴 #10 — Sklik celkově ztrátový, ROAS 0,92× (KRITICKÉ)

**Vlastník:** 🎯 Agentura

**Přes Sklik Fénix API stažen úplný přehled účtu za 6 měsíců (účet hbgroup.ppc@seznam.cz):**

| Síť | Prokliky | Náklady | Konverze | Hodnota konverzí | **ROAS** | Kvalita |
|-----|----------|---------|----------|------------------|----------|---------|
| Obsahová (display) | 5 761 | 17 238 Kč | 27 | 5 139 Kč | **0,30×** 🔴 | — |
| Vyhledávací (search) | 7 084 | 32 596 Kč | 43 | 40 468 Kč | **1,24×** 🟢 | 9/10 |
| **CELKEM** | **12 845** | **49 834 Kč** | **70** | **45 607 Kč** | **0,92×** | — |

**PNO celkem: 109,3 %** — náklady převyšují přinesený obrat. Ztráta čistě z PPC ~4 227 Kč, plus náklady obsluhy agentury.

### 🔴 #10a — 92,5 % zobrazení ztraceno kvůli rozpočtu

**Vlastník:** 🎯 Agentura

**Impression share: 7,4 %**  
**Ztráta IS kvůli rozpočtu: 92,5 %**

Vyhledávací síť má kvalitu reklamy 9/10 (výborné), ale agentura nenastavila dostatečný rozpočet → reklama se zobrazuje jen v 7,4 % relevantních vyhledávání. **92,5 % potenciálních prokliků končí u konkurence.**

Při aktuálním ROAS 1,24× ve vyhledávací síti by navýšení rozpočtu znamenalo přímý zisk. Agentura nechává peníze na stole.

### 🔴 #10b — Obsahová síť je černá díra

**Vlastník:** 🎯 Agentura

| Metrika | Hodnota |
|---------|---------|
| Náklady | 17 238 Kč |
| Hodnota konverzí | 5 139 Kč |
| ROAS | 0,30× |
| **Ztráta** | **−12 099 Kč** |
| CTR | 0,67 % |

Obsahová síť spálila přes 12 tisíc Kč za 6 měsíců se zanedbatelnou návratností. Obsahová síť na B2B+B2C zámky/bezpečnost prakticky nefunguje — zákazníci s nákupním záměrem jsou na search. Doporučení: pozastavit obsahovou síť a rozpočet přesunout do vyhledávací.

### 🔴 #10c — Zboží.cz (Sklik Nákupy): 82 % rozpočtu bez konverze

**Vlastník:** 🔀 Mixed (agentura + vývojář OCM)

Přes Sklik Fénix API detail per produkt (2 071 položek):
- Náklady: 31 616 Kč
- **Konverze: 0** (potvrzuje nález #17 — OCM tracking rozbitý)
- 410 produktů se spotřebovaným rozpočtem **bez jediné konverze** = **25 886 Kč plýtvání (82 % Zboží.cz rozpočtu)**

Top 5 „ztrátových" produktů:

| Produkt | Náklady | Kliků |
|---------|---------|-------|
| Zámek STAR Europortal do plast. dveří | 745 Kč | 143 |
| Škrabka lota s vroubkovaným ostřím | 570 Kč | 139 |
| Bezpečnostní vložka FAB 3*** | 549 Kč | 82 |
| Přídavný zámek S1570 bez vložky | 525 Kč | 94 |
| Jmenovka na schránky DOLS | 416 Kč | 83 |

Škrabka lota (kuchyňský doplněk) je 2. nejdražší produkt na Zboží.cz — stejný problém jako u Heuréky (Victorinox škrabka na rajčata) a v hlavní Zboží.cz analýze. **Feed nekurátovaný — nerelevantní produkty spalují rozpočet napříč srovnávači.**

---

## 5. MERCHANT CENTER

### 🟡 #11 — 1 338 produktů s chybnou dostupností

**Vlastník:** 🛠️ Vývojář (feed varianty)

**Přehled katalogu (Merchant Center ID: 130672692):**

| Stav | Počet produktů |
|------|----------------|
| Aktivní (approved) | ~900 |
| Disapproved | ~400+ |
| Celkem v katalogu | **2 239** |

**Nejčastější problémy:**

| Problém | Počet produktů |
|---------|----------------|
| Dostupnost neshodná s webem (in_stock vs. out_of_stock) | **1 338** |
| Chybějící nebo neplatná cena | ~200 |
| Chybějící GTIN / MPN | ~150 |
| Neplatný obrázek | ~80 |

1 338 produktů má v feedu uvedenou dostupnost, která neodpovídá reálnému stavu na webu. To způsobuje schvalování/zamítání produktů v Shopping kampaních a zkresluje výkon.

**Výkon Shopping reklam (říjen 2025 – duben 2026):**
- CTR: **1,46 %**
- Trendově klesající imprese (data k dispozici po měsících)

---

## 6. GOOGLE BUSINESS PROFILE (POBOČKY)

### 🟡 #12 — Kritické pobočky s nízkým call rate

**Vlastník:** 👤 Klient (GBP profily)

Analýza 16 poboček (říjen 2025 – duben 2026) z exportu business.google.com:

**Výkon dle zobrazení a hovorů:**

| Pobočka | Imp. Mapy | Imp. Search | Volání | Call rate | Stav |
|---------|-----------|-------------|--------|-----------|------|
| Praha (Nové Město) | nejvyšší | nejvyšší | — | — | OK |
| Chomutov | — | — | — | — | ⚠️ Sledovat |
| Brno | střední | střední | nízký | **kritický** | 🔴 |
| Plzeň OC | nízký | nízký | 0 | **0 %** | 🔴 |
| Košice | — | — | — | — | 🔴 Broken odkaz |

> Poznámka: Přesná čísla jsou v exportovaném CSV (GMB insights Performance Report). Hodnocení vychází z relativního porovnání 16 poboček.

**Zjištění:**
- Pobočky Brno a Plzeň OC mají výrazně nižší call rate než srovnatelné pobočky
- Pobočka Košice má nefunkční odkaz na webové stránky v GBP profilu
- GBP profily nejsou propojeny s Google Ads (store visits konverze nejsou sledovány)

---

## 7. SROVNÁVAČE ZBOŽÍ (Heuréka + Zboží.cz)

### 🔴 #13 — Heuréka prakticky nefunguje (KRITICKÉ)

**Vlastník:** 🔀 Mixed (agentura + vývojář OCM)

Analýza detailního OCM reportu z Heuréky za období 1. 10. 2025 – 20. 4. 2026:

| Metrika | Hodnota |
|---------|---------|
| Celkové návštěvy | 1 156 |
| z toho placené (bidded) | 25 (2,2 %) |
| z toho organické (free) | 21 (1,8 %) |
| Náklady celkem (bez DPH) | 7 037 Kč |
| Objednávky | 12 |
| Revenue | 10 130 Kč |
| **ROAS celkem** | **1,44×** |
| PNO | 69,5 % |

**Měsíční ROAS — ztrátové v 5 ze 7 měsíců:**

| Měsíc | ROAS | Stav |
|-------|------|------|
| Říjen 2025 | 4,57× | ✅ |
| Listopad 2025 | 0,00× | 🔴 0 objednávek |
| Prosinec 2025 | 0,68× | 🔴 |
| Leden 2026 | 1,64× | 🟡 |
| Únor 2026 | 0,44× | 🔴 |
| Březen 2026 | 2,46× | ✅ |
| Duben 2026 | 2,49× | ✅ |

**Na úrovni produktu:**
- **59 % nákladů (4 136 Kč)** utraceno za produkty bez jediné objednávky
- Nejdražší produkt: **Victorinox Škrabka na rajčata** (732 Kč, ROAS 0,41×) — kuchyňský doplněk v eshopu zámky/bezpečnost
- Bestsellery klicovecentrum.cz (smart Q20 zámek, ISEO panik.hrazda, trezory) mají minimální expozici

**Tracking objednávek nefunguje správně** — 12 objednávek se rozpadá na 0 placených + 0 organických; kanálová atribuce se nezaznamenává, ačkoli OCM klíč je nasazen.

---

### 🔴 #17 — Zboží.cz: 31 655 Kč, 0 konverzí za 7 měsíců (KRITICKÉ)

**Vlastník:** 🔀 Mixed (vývojář OCM + agentura)

| Metrika | Hodnota |
|---------|---------|
| Zobrazení | 243 740 |
| Prokliky | 5 772 |
| Náklady | **31 655 Kč** |
| **Konverze** | **0** |
| CTR | 2,37 % |
| Průměrné CPC | 5,48 Kč |

**ROAS nelze vypočítat** — za 7 měsíců (1. 10. 2025 – 21. 4. 2026) nedorazila jediná konverze přes OCM API. Konverzní klíč je nasazen (`qipmUyJGswP4u2sJgmlz3B0_i6MY2hPb`), ale eshop konverze nezapisuje.

**Objem klesá o 75 % během 7 měsíců:**

| Měsíc | Zobrazení | Trend |
|-------|-----------|-------|
| Říjen 2025 | 47 101 | — |
| Listopad 2025 | 44 107 | −6 % |
| Prosinec 2025 | 41 447 | −12 % |
| Leden 2026 | 38 613 | −18 % |
| Únor 2026 | 35 008 | −26 % |
| Březen 2026 | 25 801 | −45 % |
| Duben 2026 (21 dní) | 11 663 | **−75 %** |

To není sezónnost. Agentura ztrácí 11 000 zobrazení měsíčně — pravděpodobně kombinace klesajícího quality score (špatný feed → nižší priorita v rankování) a klesajícího biddingu.

**Rozložení útraty dle typu umístění:**

| Umístění | Útrata | Podíl |
|----------|--------|-------|
| Hledání | 27 259 Kč | 86 % |
| Produkt — doporučené | 2 396 Kč | 8 % |
| Produkt — dle ceny | 2 000 Kč | 6 % |
| Výpis/hledání v kategorii | 0 Kč | 0 % |

**Stejné produkty out-of-scope jako Heuréka:**
- Škrabka lota zelená — 496 Kč
- Knoflíková baterie LR44 — 304 Kč
- Baterie GP 9V — 152 Kč

Opět kuchyňské a drobné spotřební zboží v eshopu zámky/bezpečnost.

### Souhrn srovnávačů

| Kanál | Kliky | Náklady | Konverze | ROAS |
|-------|-------|---------|----------|------|
| Heuréka | 1 156 | 7 037 Kč | 12 | 1,44× |
| Zboží.cz | 5 772 | 31 655 Kč | 0 | 0× |
| **Celkem (7M)** | **6 928** | **38 692 Kč** | **12** | **~0,26×** |

**38 692 Kč bez DPH utraceno ve srovnávačích s měřitelnou návratností 1,05× (jen Heuréka).** U Zboží.cz agentura nemá žádnou metriku pro vyhodnocení výkonu ani pro optimalizaci biddingu.

---

## 8. PRODUKTOVÝ FEED (Mergado)

### 🔴 #14 — Feed je strukturálně odpojený od reality e-shopu (KRITICKÉ)

**Vlastník:** 🔀 Mixed (vývojář feed + agentura kurátorství)

Analýza 4 feedů: zdrojový `klicovecentrum.cz/nakupy.xml` a 3 výstupní přes Mergado (Google Shopping, Heuréka, Zboží.cz). Všechny obsahují **2 175 produktů**.

**Cross-reference s ERP prodejními daty (říjen 2025 – duben 2026):**

| Kategorie | Počet | Obrat bez DPH | Podíl |
|-----------|-------|---------------|-------|
| Produkty ve feedu co se prodaly | 429 (19,7 %) | 643 624 Kč | 55 % |
| **Produkty ve feedu BEZ jediného prodeje za 7 měsíců** | **1 746 (80,3 %)** | **0 Kč** | **0 %** |
| **Produkty MIMO feed co se prodaly** | **450** | **527 118 Kč** | **45 %** |

**Interpretace:**
- **80 % feedu je mrtvý** — 1 746 produktů neprodalo ani kus za 7 měsíců. Zatěžují audit kvality feedu, rozředují signál pro Google/Heuréka algoritmy a ve Sklik/Shopping kampaních za ně agentura platí prokliky bez konverzí.
- **45 % obratu e-shopu (527 118 Kč) je mimo feed** — bestsellery jako Smart nábytkový zámek Q20 (29 707 Kč), ISEO Palmo panik.hrazda (22 664 Kč), Mazadlo Super Lube (12 850 Kč) vůbec nejsou v placených kanálech. Najde je jen SEO nebo přímý zákazník — PPC k nim nemá přístup.

### 🟡 #15 — Feed neobsahuje varianty produktů

**Vlastník:** 🛠️ Vývojář (feed varianty)

Feed exportuje pouze rodičovské produkty; availability je agregovaná za všechny varianty („alespoň 1 skladem → in stock"). Google Shopping očekává availability na úrovni SKU variant.

**Důsledek:** 1 338 produktů v Merchant Center disapproved kvůli availability mismatch (nález #11). Problém se opraví až když se do feedu přidají varianty s `item_group_id` a per-variant availability.

**Další strukturální nedostatky:**
- **100 % produktů bez MPN** (Manufacturer Part Number) — 2 175 z 2 175
- 33 produktů bez GTIN
- 5 produktů bez description
- Minimum cena 0 Kč — produkty s nulovou cenou ve feedu (Google je zamítne)

---

## 9. CROSS-REFERENCE GA4 vs ERP

### 🔴 #16 — Březen 2026: ERP potvrdil, že bot útok nafoukl revenue v GA4

**Vlastník:** 🎯 Agentura

Porovnání měsíčního obratu z GA4 (`purchase` event) a z ERP (interní účetnictví):

| Měsíc | ERP obrat (bez DPH) | GA4 revenue | Rozdíl | Diagnostika |
|-------|---------------------|-------------|--------|-------------|
| 2025-10 | 277 707 Kč | 259 305 Kč | −6,6 % | ✅ Normální |
| 2025-11 | 143 464 Kč | 106 866 Kč | −34 % | 🟡 Tracking gap |
| 2025-12 | 156 718 Kč | 103 118 Kč | −51 % | 🟡 Tracking gap |
| 2026-01 | 179 847 Kč | 118 836 Kč | −51 % | 🟡 Tracking gap |
| 2026-02 | 175 098 Kč | 150 036 Kč | −14 % | 🟢 OK |
| **2026-03** | **130 138 Kč** | **200 206 Kč** | **+54 %** | **🔴 BOT INFLACE** |
| 2026-04 | 107 769 Kč | — | — | — |

**Důkaz:** Březen je jediný měsíc, kde GA4 reportuje **vyšší revenue než ERP** (+54 %). Všechny ostatní měsíce jsou GA4 ≤ ERP (normální tracking gap). V březnu se do GA4 dostaly falešné `purchase` eventy s falešnou hodnotou od bot trafficu (viz nález #2).

Agentura tato nafouklá čísla za březen reportovala jako dosažený výsledek, ačkoli skutečný obrat e-shopu byl o 70 068 Kč nižší.

**🔴 ROOT CAUSE pro listopad-leden tracking gap (−34 % až −51 %):** Pozdější analýza (květen 2026) přes Bing Webmaster + GA4 funnel identifikovala **rozbitý payment flow** (nálezy **#42 + #48**). Stránky `/page/default/dekujeme-za-vasi-objednavku` a `/platebni-stranka/` vrací HTTP 500. Pokud zákazník dokončí platbu, ale thank-you stránka vrátí 500, GA4 `purchase` event se nikdy neodešle — objednávka existuje v ERP, ale chybí v GA4. **45 z 212 dnů audit period** mělo broken payment (21,2 %), což přesně koresponduje s 30-50 % gap v listopad-leden a 14 % gap v únoru (kdy se payment mezi outage period spravil).

**Celkové porovnání (7M):**

| Zdroj | Obrat | Počet objednávek |
|-------|-------|------------------|
| ERP (skutečné prodeje) | 1 170 742 Kč | 2 567 prodejů |
| GA4 (purchase) | 984 387 Kč | 641 nákupů |
| Rozdíl | −186 355 Kč | — |

Průměrná marže e-shopu z ERP: **59,1 %** (velmi zdravá). Tato čísla ukazují, že podnikání jako takové funguje — problém je v měření a placených kanálech.

---

## 10. FIRMY.CZ A SEZNAM SÍŤ

### 🔴 #18 — Duplicitní profil Plzeň Žatecká (KRITICKÉ)

**Vlastník:** 👤 Klient (Firmy.cz)

Ve Firmy.cz existují **dva aktivní profily** pro pobočku Plzeň, Žatecká:

| ID profilu | Zobrazení | Prokliky | CTR |
|------------|-----------|----------|-----|
| 598704 | 10 506 | 2 644 | 25,2 % |
| 2662342 | 2 610 | 479 | 18,4 % |

Žatecká slouží jako pobočka i výdejní místo eshopu. Dva profily na stejné adrese znamenají:
- Rozdělenou autoritu (SEO váha, recenze, zobrazení)
- Matou zákazníka — může najít zastaralou verzi
- **Doporučení:** jeden smazat / sloučit, zachovat 598704 (vyšší engagement)

### 🟡 #19 — Tři pobočky s kriticky nízkým call rate

**Vlastník:** 👤 Klient (GBP + Firmy.cz)

Analýza 14 poboček za 7 měsíců:

| Pobočka | Zobrazení | Telefony | Navigace | Call rate |
|---------|-----------|----------|----------|-----------|
| Chomutov (Žižkovo) | 11 510 | 8 | 4 | 0,07 % 🔴 |
| Plzeň (Gerská) | 5 415 | 9 | 17 | 0,17 % 🔴 |
| Praha (Václavské) | 2 657 | 1 | 10 | 0,04 % 🔴 |

Chomutov je 3. nejviditelnější pobočka, ale druhá nejnižší engagement. Stejný vzorec jsme identifikovali v GBP (nález #12). Profily potřebují:
- Kvalitní fotky
- Aktualizovaný popis činnosti
- Ověřit otevírací dobu (14,6 % kliků míří na otevírací dobu)

### ✅ #20 — Firmy.cz nebyly zasaženy bot útokem (POZITIVNÍ DŮKAZ PRO #2)

**Vlastník:** ℹ️ Informativní

| Měsíc | Zobrazení | Trend |
|-------|-----------|-------|
| Říjen 2025 | 11 322 | — |
| Listopad | 11 355 | ±0 |
| Prosinec | 12 031 | +6 % |
| Leden 2026 | 9 164 | −24 % |
| Únor | 10 771 | +18 % |
| **Březen** | **14 785** | **+37 %** |
| Duben (22 dní) | 7 512 | na tempu říjen |

**Firmy.cz/Seznam v březnu 2026 ROSTLY o 37 %**, zatímco GA4 zaznamenalo bot inflaci +54 %. To dokazuje, že bot útok byl cílený na přímý provoz (Direct channel v GA4) a neovlivnil real-user chování na Seznamu. Firmy.cz výkon koresponduje s ERP obratem, ne s nafouklými GA4 čísly.

### 🟡 #21 — Celkový výstup Firmy.cz je silný, ale nesleduje se

**Vlastník:** 👤 Klient (Firmy.cz UTM)

- **1 845 prokliků na web** z Firmy.cz za 7 měsíců (26 % všech akcí)
- **710 naplánovaných tras** (reálný záměr návštěvy)
- **256 volání** (ale rozložené na 14 poboček = ~37 volání/měsíc)

Tento provoz z Firmy.cz (~263 webových návštěv/měsíc) pravděpodobně přichází do GA4 jako Direct (nebo s utm_source=seznam), ale agentura ho samostatně nevyhodnocuje ani nepropojuje s konverzemi.

---

## 11. SKLIK — NÁVAZNOST NA AGENTURNÍ REPORTY

### 🔴 #22 — Agenturní měsíční reporty nezachycují vyhledávací síť (KRITICKÉ)

**Vlastník:** 🎯 Agentura

Přes Sklik Fénix API (účet hbgroup.ppc@seznam.cz) jsou dostupné agenturní měsíční reporty za únor a březen 2026. Existují 4 typy reportů, ale vrací data jen jeden:

| Typ reportu | Únor 2026 | Březen 2026 |
|-------------|-----------|-------------|
| Produktové inzeráty | **0 záznamů** | **0 záznamů** |
| Vyhledávací síť | **0 záznamů** | **0 záznamů** |
| RTG (retargeting) | **0 záznamů** | **0 záznamů** |
| Obsahová síť (display) | ~145 000 zobrazení | ~143 000 zobrazení |

**Přitom reálně účet MÁ aktivitu ve vyhledávací síti** (viz nález #10 — 239 891 zobrazení, 7 084 kliků, 32 596 Kč, ROAS 1,24×). Agenturní reporty vyhledávací síť nezachycují.

**Dvě možné příčiny:**
1. Šablony reportů byly vytvořené bez filtru konkrétní kampaně — generátor neví, co reportovat
2. Agentura reporty nastavila pro kampaně, které již byly přesunuté nebo smazané

**Důsledek:** Klient dostává měsíční reporty, kde 3 ze 4 sekcí jsou prázdné, ačkoli účet aktivitu má. Je těžké vyhodnotit, co reporty klientovi fakticky prezentují — pokud jsou stejně prázdné jako ty API-verze, agentura reportuje fikci.

### 🟡 #23 — 78 % produktů ve feedu Zboží.cz je „možno vylepšit"

**Vlastník:** 🔀 Mixed (vývojář feed + agentura)

Z Sklik Fénix API (diagnostika feedu pro Zboží.cz nákupy):

| Stav produktu | Počet | % |
|---------------|-------|---|
| OK | 465 | 21 % |
| **Možno vylepšit** | **1 694** | **78 %** |
| Chyba | 16 | 1 % |
| Nezobrazené | 53 | 2 % |
| **Bez kategorie** | **355** | **17 %** |
| Přiřaditelné do kategorie | 2 122 | — |

**Pouze 21 % produktů je ve stavu OK.** Zbylých 78 % má kvalitativní problémy — to přímo koresponduje s naším nálezem #14 (80 % feedu = mrtvé produkty). 355 produktů nemá přiřazenou kategorii → nezobrazují se v relevantních výpisech na Zboží.cz.

### 🟡 #24 — Eshop nemá na Zboží.cz žádné recenze

**Vlastník:** 👤 Klient (Zboží.cz recenze)

```
"name": "Klicovecentrum.cz",
"rating": null,
"premiseId": 61575
```

Pole `rating` je **null** — eshop nikdy nedostal recenzi na Zboží.cz za celou dobu fungování. Na B2C trhu s elektronikou/bezpečností je počet a kvalita recenzí důležitý signál pro zákazníka i pro algoritmus Zboží.cz. Chybějící recenze:
- Znevýhodňuje v rankování oproti konkurenci s recenzemi
- Uživatelé preferují eshopy s viditelným hodnocením
- Agentura nikdy nerozjela kampaň na získání recenzí

---

## 11b. ALZA MARKETPLACE — SKRYTÝ PARALELNÍ KANÁL

### 🔴 #26 — Alza sold 112 350 Kč mimo ERP (KRITICKÉ pro obchodní přehled)

**Vlastník:** 🛠️ Vývojář (Alza API → ERP)

Analýza 7 měsíců objednávek z Alza Marketplace (říjen 2025 – duben 2026):

| Metrika | Hodnota |
|---------|---------|
| Produkty v katalogu | 189 |
| Objednávky za 7M | 602 |
| Prodaných kusů | 703 |
| **Obrat (bez DPH)** | **129 431 Kč** |
| Obrat (s DPH) | 162 778 Kč |
| Průměrná objednávka | 271 Kč |

**Cross-reference s ERP: pouze 1 z 31 prodávaných SKU má záznam v ERP** — zbytek (514 ks, 112 350 Kč) není v interních prodejních datech. Alza funguje jako paralelní fulfillment, pravděpodobně FBA (Fulfilled By Alza) ze zvláštního skladu.

**Důsledek:** Skutečný celkový obrat e-shopu + Alza = **~1 300 000 Kč** (ne jen 1 170 000 Kč z ERP). **Alza přidává ~10 % obratu** o kterém agentura ani analytika nemají přehled.

### 🔴 #27 — 50 % Alza katalogu „Prodej skončil"

**Vlastník:** 👤 Klient (Alza katalog)

| Stav prodeje | Počet | % |
|--------------|-------|---|
| V prodeji | 33 | 17,5 % |
| **Prodej skončil** | **95** | **50,3 %** |
| Neaktivní | 52 | 27,5 % |
| Není v prodeji | 5 | 2,6 % |
| Prodává Alza | 3 | 1,6 % |
| Dražší než ostatní | 1 | 0,5 % |

**95 produktů v kategorii BURG-WÄCHTER trezory, STAR visací zámky a mySafe** má status „Prodej skončil" ale produkty jsou v katalogu aktivní. Klasický problém chybějícího kurátorstva — když Alza produkt automaticky stáhne (nedostupný, vyprodáno, změna ceny u konkurence), agentura ani interní tým ho nereaktivuje.

### 🟡 #28 — Struktura prodejů na Alze je diametrálně jiná než e-shop

**Vlastník:** 👤 Klient (Alza strategie)

Top 5 Alza bestsellerů (7 měsíců):

| Produkt | Kategorie | Ks | Obrat |
|---------|-----------|-----|-------|
| STAR Mazadlo 150 ml | Údržba | 189 | 23 521 Kč |
| STAR Mazadlo 30 ml | Údržba | 189 | 17 081 Kč |
| Star balkonové madlo | Dveře | 143 | 7 529 Kč |
| BW Trezor DUAL SAFE 415 E FP | Trezory | 2 | 24 369 Kč |
| Visací zámek 20HS 20mm | Visací | 51 | 1 691 Kč |

**Alza prodává drobný konzumní sortiment** (mazadla 378 ks, madla 143 ks, visací zámky série 20HS). E-shop v ERP prodává high-value smart locks, panic hrazdy, motorické zámky (AOV 1 535 Kč). **Alza AOV: 271 Kč — 5,6× nižší.** Jiné publikum, jiná proposition, jiný optimalizační cíl.

### ✅ #29 — Ceny na Alze jsou správně nastavené

**Vlastník:** ℹ️ Informativní

Jen 1 produkt (M.A.T. Group schránka Radim M) má flag „Dražší než ostatní" z 189. Cenový monitoring na Alze je dobře vyřešený.

### 🟡 #30 — Agentura nepracuje s Alza kanálem

**Vlastník:** 👤 Klient (Alza správa)

Alza má vlastní reklamní platformu (Alza Ads — kampaně v Alza výpisech a newsletterech), interní feed pro produkty „Prodává Alza" atd. Z auditu nevyplývá, že by agentura:
- Kontrolovala, zda 95 produktů „Prodej skončil" jde reaktivovat
- Nastavovala dynamické ceny přes Alza Marketplace API
- Koordinovala reklamní kampaně (Google Ads PMax pro e-shop vs. Alza Ads na marketplace)
- Reportovala Alza obrat klientovi

**Alza je dnes na autopilotu** — generuje 129 k Kč / 7M bez jakéhokoli řízení.

---

## 12. STRATEGICKÝ RÁMEC: PRODUKTOVÉ vs BRANDOVÉ / LOKÁLNÍ KAMPANĚ

### Reálná data — per kampaň z Google Ads (říjen 2025 – duben 2026)

**10 aktivních kampaní, rozpočet 231 000 Kč:**

| Kategorie | Kampaní | Náklady | Konv. | Hodnota | **ROAS** |
|-----------|---------|---------|-------|---------|----------|
| LOCAL (PMax pobočky) | 5 | 88 000 | 799 | 667 000 | **7,58×** 🟢 |
| BRAND (BR_SR_All) | 1 | 29 000 | 61 | 170 000 | **5,86×** 🟢 |
| PRODUCT (PMax_All + SEA + PLA) | 4 | 114 000 | 448 | 344 000 | **3,02×** 🟡 |
| **CELKEM** | **10** | **231 000** | **1 308** | **1 181 000** | **5,11×** |

**Klíčové zjištění Google Ads:**
- **LOCAL kampaně jsou zlaté** — 7,58× ROAS, PMax_Local_Plzeň-Lochotín dokonce **13,22×**
- Přesto mají LOCAL jen 38 % rozpočtu (88 k ze 231 k)
- PRODUCT kampaň PMX_All generuje 45 % rozpočtu při podprůměrném ROAS 3,13× (benchmark B2C 4–6×)
- SEA_Gravírování: ROAS 1,00× (hraniční)

### 🔴 Historická destrukce LOCAL infrastruktury

Od roku 2018 agentura **odstranila 42 lokálních kampaní**. Šlo o celou síť search kampaní per pobočka + remarketing:
- Vyhledávání Cheb / Ostrava / Plzeň / Praha × (Autoklíče / Bezpečnostní cylindry / Elektronické přístupové systémy / Trezory / Zabezpečení)
- Remarketing Sokolov / Ostrava / Plzeň / Praha / Č. Budějovice

**Nahrazeny** 5 PMax kampaněmi pro jen 4 pobočky (Chomutov, Brno, Plzeň-Lochotín, Hrdějovice). **Zbylých 13+ poboček nemá lokální PPC kampaň** — přitom ROAS fungujících LOCAL je 7,58×.

### Reálná data Sklik per kampaň (říjen 2025 – duben 2026)

**15 kampaní, rozpočet 49 834 Kč (přes Fénix API):**

| Kampaň | Kategorie | Náklady | Konv. | Hodnota | ROAS |
|--------|-----------|---------|-------|---------|------|
| **0 VG Brand** | BRAND | 1 694 | 43 | 40 468 | **23,89×** 🟢 |
| 5 VG DRMKT (D/M) | RTG | 3 129 | 27 | 5 139 | 1,64× |
| Zboží.cz: Klicovecentrum.cz | PRODUCT | 30 542 | 0 | 0 | 🔴 tracking |
| LOCAL kampaně (8× JOK/SEA/P4M pobočka) | LOCAL | 13 446 | 0 | 0 | 🔴 tracking |
| Ostatní | — | 1 023 | — | — | — |

**Sklik klíčové zjištění:**
1. **Brand search ROAS 23,89× při jen 3,4 % rozpočtu.** Agentura ignoruje nejziskovější kanál.
2. **88 % rozpočtu (43 988 Kč) v kampaních s nefunkčním OCM trackingem** — PRODUCT (Zboží.cz) a LOCAL nemají konverze.
3. Pokud LOCAL kampaně na Skliku mají stejnou účinnost jako na Google Ads (ROAS 7–8×), agentura nevidí ~100 000 Kč skrytého obratu.

### Klasifikace po kanálech — porovnání

| Kanál | PRODUCT | LOCAL | BRAND | RTG |
|-------|---------|-------|-------|-----|
| Google Ads ROAS | 3,02× | **7,58×** | 5,86× | n/a |
| Sklik ROAS | ? (broken) | ? (broken) | **23,89×** | 1,28× |
| Sklik % rozpočtu | 61,5 % | 27 % | **3,4 %** | 8,1 % |
| Google Ads % rozpočtu | 49 % | **38 %** | 13 % | — |

**Asymetrie**: Google Ads má tracking OK a strukturu přiměřenou (i když ne optimální). Sklik má tracking rozbitý na 88 % rozpočtu a **brutálně podfinancovaný brand search**.

---

### Obecný rámec 3 kategorií

Většina nálezů v tomto auditu hodnotí PPC kampaně přes **ROAS** (návratnost investice do reklamy). To je správná metrika pro **produktové/performance kampaně**, ale **nevhodná pro brandové a lokální kampaně**, které sledují jiné cíle.

### Dvě samostatné kategorie kampaní

| Kategorie | Cíl | Správné KPI | Aktuální kampaně v účtu |
|-----------|-----|-------------|--------------------------|
| **Produktové / Performance** | Prodej konkrétního zboží | ROAS, CPA, konverzní poměr | Google Shopping / PMax, Sklik Nákupy (Zboží.cz), Heuréka, Glami |
| **Brandové / Lokální** | Návštěva pobočky, povědomí, ochrana značky | Store visits, volání, navigace, brand search growth, CTR, impression share | Brand search kampaně, GBP, Firmy.cz, obsahová síť remarketingu, lokální Sklik |

### Proč to v auditu řešit

1. **Nálezy #10, #10a, #10b, #13, #17 hodnotí ROAS na Sklik obsahové síti a Heuréce.** Pokud obsahová síť funguje jako **remarketing** nebo **brand awareness**, ROAS 0,30× není automaticky selhání — remarketing má dlouhé atribuční okno a funguje na asistované konverze. Otázka zní: **Co konkrétně je cílem obsahové sítě?** Agentura to v reportech klientovi nekomunikuje.

2. **Nálezy #6 (61 % Google Ads mimo spádové oblasti) a #12 (GBP profily) jsou lokální.** U nich není cílem online nákup, ale návštěva kamenné prodejny. KPI by měly být: store visit konverze, „volání", „plánování trasy", nárůst brand search pro „klíče [město]".

3. **Sklik search kampaň "P4M_SZ_ACQ_SR_All_pobočka Chomutov"** (identifikovaná v dřívějším auditu) je podle názvu **lokální pobočková kampaň**, ne produktová. ROAS 1,24× na vyhledávací síti je sice dobrý, ale zahrnuje i brand dotazy (přirozeně vysoký konverzní poměr). **Rozklad brand vs non-brand search neexistuje** — agentura je nesegmentuje.

### Jak to přelasifikuje stávající nálezy

| Nález | Původní hodnocení | Správný kontext |
|-------|-------------------|------------------|
| #5 Smart Bidding na falešných konverzích | Platí univerzálně — falešné konverze kazí oboje | — |
| #6 61 % Google Ads do krajů bez poboček | Jako performance = plýtvání | **Jako lokální = klíčová chyba**: kampaň měla cílit pouze na spádové oblasti |
| #7 CTV 2 662 Kč, 0 kliků | Performance ztráta | CTV je brand awareness — ale bez jakéhokoli měřitelného impaktu nemá smysl |
| #10 Sklik ROAS 0,92× | Ztrátové celkově | **Je nutné rozdělit:** search pro produkty vs search pro pobočky vs obsahová pro remarketing |
| #10b Obsahová síť ROAS 0,30× | Ztráta 12 099 Kč | Pokud je to **remarketing**, ROAS isn't kompletní příběh. Ale agentura nemá stanovené KPI. |
| #12 GBP — nízký call rate | — | **Lokální cíl** — správné metriky jsou volání, navigace, store visit. GBP nepropojen s Google Ads pro store visit conversions. |
| #13 Heuréka ROAS 1,44× | Ztrátové v 5/7 měs. | **Čistě produktové** — hodnocení ROAS správné |
| #17 Zboží.cz 0 konverzí | Tracking rozbitý | **Produktové** — tracking je základ |
| #18–21 Firmy.cz | — | **Lokální/brandové** — 1 845 kliků na web, 710 navigací, 256 volání za 7M; chybí propojení s GA4 pro atribuci |

### Co z toho plyne pro nápravu

**Než začne optimalizace, agentura musí:**

1. **Oddělit v Google Ads produktové a brand kampaně** jako samostatné kampaně (nejen ad groups):
   - `KC_Brand_SRC` (ochrana značky, ROAS ~10×)
   - `KC_Product_PMax` (produktový prodej, ROAS cíl 4-6×)
   - `KC_Local_[pobočka]` (cíl: store visits, volání)

2. **V Sklik stejně** — dnes je 1 kampaň pro Chomutov na search síti míchaná s PMax Nákupy a obsahovou sítí.

3. **Pro každou kampaň stanovit primární KPI a reportovat v této struktuře klientovi.**

4. **GBP propojit s Google Ads** → store visit conversions budou primární metrikou pro lokální kampaně.

5. **Firmy.cz traffic označit UTM** → atribuovat Firmy.cz jako samostatný kanál v GA4.

---

## 13. SOUHRN NÁLEZŮ — MATICE ZÁVAŽNOSTI

| # | Nález | Závažnost | Oblast | Finanční dopad |
|---|-------|-----------|--------|----------------|
| 1 | Konverze nafouklé 265× | 🔴 Kritické | GA4 / Google Ads / Sklik | Veškerý PPC rozpočet |
| 2 | Bot útok 1,5M sessions, neřešen | 🔴 Kritické | GA4 | Data celého března |
| 3 | Direct 90 % — atribuce nepoužitelná | 🔴 Kritické | GA4 | Nelze měřit žádný kanál |
| 4 | CTR 0,4–0,6 % na top stránkách | 🟡 Závažné | Search Console | ~6 100 kliků/měs. ztráta |
| 5 | Smart Bidding na falešných datech | 🔴 Kritické | Google Ads | Celý Google Ads rozpočet |
| 6 | 32 % rozpočtu mimo spádové oblasti | 🔴 Kritické | Google Ads | ~74 200 Kč |
| 7 | Connected TV: 2 662 Kč, 0 kliků | 🟡 Závažné | Google Ads | 2 662 Kč přímá ztráta |
| 8 | PMax bez efektivních negative KW | 🟡 Závažné | Google Ads | Neměřitelný únik |
| 9 | Žádné produktové search kampaně | 🟡 Závažné | Google Ads | Ztráta intent provozu |
| 10 | Sklik celkově ROAS 0,92× (ztrátové), PNO 109 % | 🔴 Kritické | Sklik | −4 227 Kč + agenturní fee |
| 10a | Sklik impression share 7,4 % (92,5 % ztraceno budgetem) | 🔴 Kritické | Sklik | Ušlý zisk z ROAS 1,24× search |
| 10b | Sklik obsahová síť ROAS 0,30× (ztráta 12 099 Kč) | 🔴 Kritické | Sklik | −12 099 Kč |
| 10c | Zboží.cz: 82 % rozpočtu (25 886 Kč) bez konverze | 🔴 Kritické | Sklik / Zboží.cz | 25 886 Kč plýtvání |
| 10d | Sklik BRAND (0 VG Brand) ROAS 23,89× ale jen 3,4 % rozpočtu | 🔴 Kritické | Sklik | Obrovský ušlý zisk |
| 10e | 88 % Sklik rozpočtu (44 k Kč) v kampaních s rozbitým OCM trackingem | 🔴 Kritické | Sklik | Neměřitelný výkon |
| 10f | Google Ads LOCAL ROAS 7,58× ale jen 38 % rozpočtu, chybí 13+ poboček | 🔴 Kritické | Google Ads | Ušlý zisk ~100 k+ |
| 10g | 42 historických LOCAL kampaní smazaných 2018 (search + RTG) | 🟡 Závažné | Google Ads historie | Strukturální regres |
| 11 | 1 338 produktů s chybnou dostupností | 🟡 Závažné | Merchant Center | Shopping výkon |
| 12 | GBP pobočky — chybné info, nízký call rate | 🟡 Závažné | GBP | Ztráta lokálních zákazníků |
| 13 | Heuréka ROAS 1,44× — ztrátové 5 ze 7 měs. | 🔴 Kritické | Heuréka | 4 136 Kč bez konverze |
| 14 | 80 % feedu mrtvé, 45 % obratu MIMO feed | 🔴 Kritické | Mergado feed | 527 118 Kč mimo PPC |
| 15 | Feed bez variant (MPN chybí 100 %) | 🟡 Závažné | Mergado feed | Přímá vazba na #11 |
| 16 | Březen 2026: GA4 +54 % vs ERP (bot inflace) | 🔴 Kritické | GA4 / ERP | Důkaz pro #2 |
| 17 | Zboží.cz: 31 655 Kč, 0 konverzí, traffic −75 % | 🔴 Kritické | Zboží.cz | 31 655 Kč neměřitelné |
| 18 | Firmy.cz: duplicitní profil Plzeň Žatecká | 🔴 Kritické | Firmy.cz | Rozdělená autorita |
| 19 | Pobočky Chomutov/Plzeň/Praha Václavské — 0 engagement | 🟡 Závažné | Firmy.cz / GBP | Ztráta zákazníků |
| 20 | Firmy.cz +37 % v březnu — další důkaz bot inflace #2 | ✅ Důkaz | Firmy.cz / GA4 | Validace #16 |
| 21 | Firmy.cz provoz (263 návštěv/měs.) nesledován | 🟡 Závažné | Firmy.cz / GA4 | Skrytý kanál |
| 22 | Agentura: 3 ze 4 Sklik reportů prázdné | 🔴 Kritické | Sklik / agentura | Fiktivní reporting |
| 23 | 78 % produktů v Zboží.cz feedu „možno vylepšit" | 🟡 Závažné | Mergado / Zboží.cz | Nízká viditelnost |
| 24 | Zboží.cz rating = null (0 recenzí) | 🟡 Závažné | Zboží.cz | SEO + konverze |

---

## 14. DOPORUČENÉ KROKY

### KROK 0 — Strategický rámec (před jakoukoli optimalizací)

**Rozdělit všechny PPC kampaně do 3 kategorií s oddělenými cíli a KPI:**

| Kategorie | Cíl | Primární KPI | Příklady |
|-----------|-----|--------------|----------|
| **PRODUCT** | Přímý prodej | ROAS ≥ 4× | Google Shopping, PMax product, Sklik Nákupy, Heuréka, Zboží.cz |
| **BRAND** | Ochrana značky, povědomí | Impression share 95 %, CTR | Brand search „klíče centrum", „klicovecentrum" |
| **LOCAL** | Návštěva pobočky | Store visits, volání, navigace | Lokální search per město, GBP, Firmy.cz, místní lokality v PMax |

Tento rámec je **prerekvizita pro všechny následující kroky.** Dokud se kampaně míchají, nelze správně vyhodnotit výkon ani optimalizovat.

### Okamžitě — tento týden

**GA4:**
1. **Odebrat micro-eventy z primárních konverzí** — ponechat pouze `purchase` jako Primary. Všechny ostatní přesunout na Secondary nebo zcela odebrat.
2. **Zapnout Bot Filtering** — GA4 Admin → Data Settings → Data Filters → aktivovat bot filter.
3. **Nastavit IP filter** pro interní provoz (kanceláře klienta, agentury).
4. **Data retention** nastavit na 14 měsíců (výchozí 2 měsíce způsobují ztrátu historických dat).

**Google Ads:**

5. **Přenastavit konverze** — po nápravě GA4 reimportovat konverze nebo vytvořit novou konverzi pouze na `purchase`.
6. **Vyloučit Connected TV** ze všech kampaní (Device → CTV → Excluded).
7. **Geotargeting audit** — omezit kampaně na spádové oblasti skutečných poboček (okruh ~50 km od každé prodejny).
8. **Vyžádat GTM přístup** — je to majetek klienta, agentura nemá právo přístup odmítat.

**Sklik:**

9. **Pozastavit obsahovou síť** — ROAS 0,30× za 6 měsíců znamená čistou ztrátu 12 099 Kč. Rozpočet přesunout do vyhledávací sítě.
9a. **Navýšit rozpočet vyhledávací sítě** — aktuální impression share 7,4 % (ROAS 1,24×, kvalita 9/10). Každá navýšená koruna přinese zisk, pokud ROAS drží nad 1×.
9b. **Zboží.cz produktové vyloučení** — 410 nerelevantních produktů (škrabka lota, baterie LR44) je 82 % rozpočtu bez konverze. Odstranit z biddingu přes Mergado custom label.
9c. **Opravit šablony agenturních reportů** — momentálně 3 ze 4 reportů prázdných; přidat filtry pro existující kampaně.
9d. **Dramaticky navýšit Sklik brand search** — „0 VG Brand" ROAS 23,89× při 3,4 % rozpočtu. Zvýšit rozpočet 10× a sledovat ROAS. Pokud drží nad 10×, pokračovat v navyšování.
9e. **Opravit OCM tracking pro PRODUCT a LOCAL kampaně** — Zboží.cz + 8 LOCAL Sklik kampaní má 0 konverzí kvůli nefunkční atribuci. Zkontrolovat, že eshop posílá `orderId`, `itemId` a `unitPrice` pro všechny objednávky.

**Google Ads — LOCAL kampaně:**

10. **Obnovit LOCAL infrastrukturu** — historicky existovalo 42 kampaní per pobočka, agentura je 2018 smazala. Dnes pokrývá 4 z 17 poboček.
    - Vytvořit PMax Local pro všech 13 chybějících poboček (Praha, Sokolov, Ostrava, Cheb, Vodňany, Č. Budějovice centrum, Mladá Boleslav, atd.)
    - LOCAL PMax mají ROAS 5–13× → každá nová pobočka by měla generovat kladný ROAS
11. **Přesunout rozpočet z PMax_All (3,13× ROAS) do LOCAL PMax (7,58× ROAS)** — jednoduchá optimalizace, okamžitý dopad.

**Heuréka:**

10. **Opravit OCM tracking** — kanálová atribuce (bidded/free) se neukládá; bez toho nelze měřit skutečný přínos placené expozice.
11. **Pozastavit bidding na nerelevantních produktech** (Victorinox kuchyňské doplňky atd.) — 59 % nákladů bez konverze.

### Krátkodobě — 1–2 týdny

12. **Title tagy a meta descriptions** pro stránky s nízkým CTR (priorita: /cylindricke-vlozky/, /navod-jak-urcit-orientaci-dveri/, /trezory/).
13. **Schema.org implementace** — Product schema pro produktové kategorie, LocalBusiness schema pro všech 16 poboček.
14. **UTM audit** — projít všechny aktivní kampaně (Google Ads, Sklik, emaily, Heuréka) a ověřit konzistenci UTM parametrů.
15. **Opravit GBP profil Košice** — nefunkční webový odkaz.

**Produktový feed (kritický blok):**

16. **Přidat chybějících ~450 bestsellerů do feedu** — produkty z ERP, které se prodávají, ale ve feedu nejsou (ztracený obrat 527 118 Kč). Seznam dle ERP analýzy.
17. **Vyřadit 1 746 mrtvých produktů** z placených kanálů — v Mergadu nastavit custom label `no_sales_7m` a v Google Shopping / Heuréka / Zboží kampaních je exclude z biddingu (nemazat z feedu — nechat pro organickou viditelnost).
18. **Opravit export variant ve feedu** — přidat `item_group_id` a per-variant availability. To vyřeší nález #11 (1 338 Merchant Center disapproved) i nález #15.
19. **Doplnit MPN** do feedu pro všech 2 175 produktů (dnes chybí u 100 %).

### Střednědobě — 1 měsíc

20. **Přenastavit Smart Bidding** po opravě konverzí (nutná relearning fáze 2–4 týdny — počítat s dočasným poklesem výkonu).
21. **Přidat negative keywords do PMax** — přes brand exclusions nebo signal nastavení.
22. **Spustit produktové search kampaně** pro klíčové kategorie (cylindrické vložky, trezory, zámky).
23. **Nastavit GA4 upozornění** na traffic anomálie (threshold: +300 % denních sessions).
24. **Propojit GBP s Google Ads** pro měření store visits konverzí.
25. **Označit březnová data** v GA4 jako anomálii (Data annotation) — s odkazem na důkaz z ERP (nález #16).

---

## 15. FINANČNÍ SHRNUTÍ

### Skutečný výkon e-shopu (ERP, říjen 2025 – duben 2026)

| Položka | Hodnota |
|---------|---------|
| **Celkový obrat bez DPH** | **1 170 742 Kč** |
| z toho poštovné | 52 241 Kč |
| z toho zboží | 1 118 501 Kč |
| **Hrubý zisk** | **692 431 Kč** |
| **Průměrná marže** | **59,1 %** |
| Počet prodejů | 2 567 |
| Unikátních prodaných SKU | 879 |

### Ztráty a rizika identifikovaná v auditu

| Položka | Hodnota |
|---------|---------|
| Google Ads: útrata mimo spádové oblasti | ~74 200 Kč (revidováno z původních ~141 000 Kč po opravě geo mappingu) |
| Google Ads: Connected TV bez kliknutí | 2 662 Kč |
| Heuréka: náklady na produkty bez konverze | 4 136 Kč |
| **Sklik: celkový 6měs. ROAS** | **0,92× (ztrátové)** |
| Sklik: obsahová síť čistá ztráta | −12 099 Kč |
| Sklik: vyhledávací síť ROAS | 1,24× (profitabilní, ale IS 7,4 %) |
| Sklik Zboží.cz: plýtvání | 25 886 Kč (82 % rozpočtu) |
| **Sklik: ztracený potenciál — 92,5 % IS vyhledávací** | **obrovský** |
| GA4 tracking gap vs ERP (mimo březen) | −186 355 Kč obratu neviděn |
| **Obrat mimo placené kanály (produkty nejsou ve feedu)** | **527 118 Kč** |

### Co nelze vyčíslit (ale je reálné)

- **Ztráta z neefektivního Smart Bidding** — kampaně se 6 měsíců učily na špatných datech. Dopad na CPA a ROAS je neznámý, ale jistě negativní.
- **Ztráta organických kliků** z nízkého CTR na top stránkách — odhadovaně 6 000+ kliků/měsíc, které šly ke konkurenci.
- **Ztráta z jediné aktivní Sklik kampaně** — 29 pozastavených kampaní = 29 kategorií, kde klicovecentrum.cz není vidět na Seznamu.
- **Propadlý potenciál 450 bestsellerů bez placené expozice** — pokud se dostanou do feedu, PPC kampaně mohou konverze násobit.

---

## 16. PŘÍLOHY

| Příloha | Obsah |
|---------|-------|
| `audit-findings.md` | Původní nález #1–#6 s daty z GA4 |
| `google_ads_script_v2.js` | Script pro export Google Ads dat |
| `google_ads_script_settings_v2.js` | Script pro export nastavení kampaní |
| `Audit Google Ads v2.xlsx` | Export dat kampaní (Kampaně, Sestavy, KW, Měsíce) |
| `Audit Google Ads — Nastaveni v2.xlsx` | Export nastavení (Budgety, Device split, Geo split, Negative KW) |
| `merchant_center_v2.py` | Analýza Merchant Center katalogu |
| `merchant_perf.py` | Výkon Shopping reklam po měsících |
| `gbp_analysis.py` | Analýza Google Business Profile |
| `GMB insights Performance Report.csv` | Export výkonu poboček z business.google.com |
| `heureka_analysis.py` | Analýza Heuréka OCM reportu (1 156 návštěv, ROAS) |
| `report-2025-10-to-2026-04-20.csv` | Zdrojový Heuréka OCM export |
| `erp_analysis.py` | Analýza 7 měsíců ERP prodejních dat |
| `prodeje zboží MO e-shop (*.csv)` | ERP exporty po měsících (říjen 2025 – duben 2026) |
| `mergado_feed_analysis.py` | Parsování 4 produktových feedů |
| `feed_vs_erp.py` | Cross-reference feed × ERP prodeje (nález #14) |
| `zbozi_analysis.py` | Analýza Zboží.cz statistik (nález #17) |
| `statisticsReport*.csv` | Denní + produktové + kategorie exporty Zboží.cz |
| `firmy_analysis.py` | Analýza 14 Firmy.cz profilů (nálezy #18–#21) |
| `klicove-ce_prem*.csv` | 14 CSV exportů z Firmy.cz (per pobočka) |
| `sklik_fenix_pull.py` + `sklik_fenix_download.py` | Sklik Fénix API pull (nálezy #22–#24) |
| `sklik_exports/*.json` | Sklik měsíční reporty (únor + březen 2026) |
| `sklik_full_analysis.py` | Kompletní analýza Sklik účtu 6 měsíců (nálezy #10, #10a–c) |
| `sklik_campaigns_classify.py` | Per-kampaň Sklik + klasifikace PRODUCT/BRAND/LOCAL (nálezy #10d–e, sekce 12) |
| `gads_classify.py` | Per-kampaň Google Ads + klasifikace (nálezy #10f–g, sekce 12) |
| `hbgroup.ppc@seznam.cz - Kampane po dnech minulych 6 mesicu.csv` | Sklik per kampaň po dnech |
| `hbgroup.ppc@seznam.cz - Ucet po dnech minulych 6 mesicu (1).csv` | CSV účtu po sítích |
| `hbgroup.ppc@seznam.cz - Nabidky z feedu 1.10.2025 - 21.4.2026.json` | Zboží.cz per produkt |
| `credentials.json` | Všechny API tokeny a přístupy (lokálně, nesdílet) |

---

## 15. BING WEBMASTER + PAYMENT OUTAGE ANALÝZA (květen 2026)

Doplňková analýza po prvním zveřejnění zprávy: napojen Bing Webmaster Tools API (klient získal API key 5/2026), který odhalil vrstvu kritických technických problémů, jež vysvětlují root cause několika dříve nálezů.

### 🔴 #42 — Payment / thank-you stránka vrací HTTP 500 (KRITICKÉ)

**Vlastník:** 🛠️ Vývojář (payment 500 fix)

Bing Webmaster Tools `GetCrawlIssues` endpoint vrátil **2 242 URLs s problémy**, mezi nimi:

- `https://klicovecentrum.cz/page/default/dekujeme-za-vasi-objednavku` → **HTTP 500**
- `https://www.klicovecentrum.cz/platebni-stranka/` → **HTTP 500**

Bing crawl history ukazuje **konstantních 45 HTTP 5xx errorů každý den** od listopadu 2025 do února 2026 = stejných ~45 URLs vrací 500 každý den, Bingbot opakovaně testuje.

**Dopad:** Pokud zákazník dokončí platbu, ale thank-you stránka 500uje, GA4 `purchase` event se nikdy neodešle. To je přímá příčina nálezu #16 (tracking gap −34 % až −51 %).

**Akce klienta:** Server-side debug `/page/default/dekujeme-za-vasi-objednavku` a `/platebni-stranka/`. Po opravě **měřitelné zlepšení GA4 conversion tracking** i bez dalších úprav.

### 🔴 #43 — old.klicovecentrum.cz zombie subdomain (KRITICKÉ)

**Vlastník:** ⚙️ DevOps (DNS / redirect)

16+ URLs na subdoméně `old.klicovecentrum.cz` vrací HTTP 500. Subdoména existuje, Bingbot ji aktivně crawluje, ale aplikace je rozbitá. Sample 500 URLs:

- `/index.php/cs/component/contact/58-ostrava.html`
- `/index.php/cs/klicarsky-zpravodaj/oteviraci-technika.html`
- `/index.php/cs/kontakty/plzen/vedouci-skladu-thanak.html`

**Akce klienta:** Buď opravit subdoménu, nebo úplně odebrat z DNS / 301 redirect na hlavní doménu. Plýtvání crawl budgetem.

### 🔴 #44 — SECURITY: PHPMailer 5.2.1 directory exposed (KRITICKÉ — bezpečnostní)

**Vlastník:** ⚙️ DevOps (file delete + audit)

18 URLs v `/phpmailer_5.2.1/` na produkčním webu. PHPMailer < 5.2.18 má **CVE-2016-10033 (Remote Code Execution)**. Bing tyto URLs aktivně indexuje.

**Akce klienta:** Okamžitě smazat z webroot, prověřit jestli adresář nebyl zneužit (logy POST requestů na exploit endpointy). Penetrační audit.

### 🔴 #45 — URL exploze + 3 vrstvy legacy debt (KRITICKÉ)

**Vlastník:** 🛠️ Vývojář (canonical / 301 redirect cleanup)

Bing index obsahuje **2 318 650 URLs** pro eshop s ~2 000 produkty. Robots.txt blokuje **~150 000 requests/den** (26 064 378 za 174 dnů). Tři vrstvy legacy:

| Vrstva | Příklad | Počet URLs |
|--------|---------|------------|
| Joomla migration debt | `/index.php/cs/`, `/index.php/sk/`, `/index.php/eshop/` | 307 |
| Duplicate produkty | `/produkt/`, `/product/default/`, `/page/default/` | 1 280 |
| HTML encoding bug | `?&amp;amp;amp;itemid=`, `&amp;amp;amp;option=` | 281 |
| HTTP (ne HTTPS) | celé staré linky | 711 (32 %) |
| Filter/sort URLs | `?itemscontrol-limit`, `?itemscontrol-short` | 94 |

**Akce klienta:** Implementovat canonical URLs, 301 redirecty Joomla legacy → nové URLs, opravit HTML escaping bug, dokončit HTTP→HTTPS migraci.

### 🟡 #46 — Druhá vlna útoku duben 2026 + mini-vlna prosinec 2025

**Vlastník:** ℹ️ Informativní

Z-score analýza Bing crawl errors (174 dnů) ukázala další 2 vlny mimo březen 2026:

| Vlna | Datum | Signature | Z-score |
|------|-------|-----------|---------|
| Mini-vlna 12/2025 | 7., 11., 12. 12. | crawl_errors ~5 900/den | z+2.6 |
| Hlavní vlna 03/2026 (#2) | 23.-25. 3. | 5xx peak 791, 4xx 4 026 | z+5 až +6.9 |
| Druhá vlna 04/2026 | 15.-16. 4. | 4xx peak 7 840, 5 115 | z+9.9, z+6.4 |

Aktualizovat finding #2 — útok není izolovaný incident března 2026, ale kontinuální problém se 3 hlavními vlnami od prosince 2025 do dubna 2026.

### 🟡 #47 — Q4 2025 chronologie — pomalý nárůst Direct share

**Vlastník:** ℹ️ Informativní

GA4 měsíční Direct share ukázal **postupný nárůst před hlavním útokem**:

| Měsíc | Direct share % | Komentář |
|-------|---------------|----------|
| Září 2025 | 38,8 % | baseline |
| Říjen 2025 | 41,8 % | +3 pp |
| Listopad 2025 | 48,0 % | +9 pp od září |
| Březen 2026 | ~90 % | hlavní útok kulminoval |

Útok **fáze přípravy** od listopadu 2025, **plný impact** v březnu 2026. Bot kampaně začaly subtilně.

### 🔴 #48 — Payment outage 24 období, 45 dnů audit period, ~261 k Kč ztráta (KRITICKÉ)

**Vlastník:** 🛠️ Vývojář (souvisí s #42)

Plný audit period 1. 10. 2025 → 30. 4. 2026 (212 dnů) — GA4 funnel `add_to_cart` → `checkout` → `transactions`:

**Detection algoritmus:** Den je broken pokud (a) sessions > 500 + checkout > 5 + 0 transactions, nebo (b) checkout > 15 + ≤ 1 transaction, nebo (c) sessions > 1000 + revenue < 200 Kč.

**Výsledek: 45 z 212 dnů (21,2 %) má rozbitý payment.**

Top 10 outage období:

| # | Období | Kal. dnů | Checkouts | Transactions | Conv % |
|---|--------|----------|-----------|--------------|--------|
| 1 | 21.-25. 11. 2025 | 5 | 74 | 2 | 2,7 % |
| 2 | 29. 11. - 2. 12. 2025 | 4 | 80 | 2 | 2,5 % |
| 3 | 29. 12. 2025 - 3. 1. 2026 | 6 | 117 | 3 | 2,6 % |
| 4 | 29. 1. - 2. 2. 2026 | 5 | 138 | 5 | 3,6 % |
| 5 | 11.-16. 2. 2026 | 6 | 105 | 2 | 1,9 % |
| 6 | 26.-28. 3. 2026 | 3 | 59 | 1 | 1,7 % |
| 7 | 2.-6. 4. 2026 | 5 | 73 | 3 | 4,1 % |
| 8 | 11.-14. 4. 2026 | 4 | 101 | 1 | **1,0 %** |
| 9 | 25.-26. 4. 2026 | 2 | 89 | 1 | 1,1 % |
| 10 | 22.-23. 10. 2025 | 2 | 59 | 1 | 1,7 % |

**Baseline (167 zdravých dnů):** 36,7 checkouts/den × **12,5 % conversion** = 4,6 transactions/den.
**Reálná ztráta:** 45 broken dnů × 4,6 = **~207 očekávaných transakcí**, dostali ~30. **177 ztracených × 1 472 Kč/trans = ~261 000 Kč přímé ztráty revenue za 7 měsíců**.

**3 typy outage identifikovány:**

| Typ | Signature | Příklad | Kořenová příčina |
|-----|-----------|---------|------------------|
| **A) Hosting / feed outage** | Sessions ↓ napříč všemi kanály | 20.-21. 9. 2025 (sessions −42 %) | Server / CDN down |
| **B) Payment-only outage** | Sessions OK, checkout OK, trans ≈ 0 | 23. 11. → 12. 12. 2025 (20 dní) | Payment 500 (#42) |
| **C) Bot inflated + payment broken** | Sessions explode (10k-175k), trans propad | 11. 3., 28. 3., 2.-6. 4. 2026 | Combined útok + #42 |

Klient potvrdil "od 26. 11. nefungoval web" — ve skutečnosti byl front-end OK, jen platba.

**Akce klienta (priorita #1):** Opravit #42 (payment 500). Přímý dopad ~37 000 Kč/měsíc zachráněného revenue + zlepšení GA4 trackingu.

### 🟡 #49 — Bing crawl errors korelují s payment outage podle období

**Vlastník:** ℹ️ Informativní

Cross-validation 39 broken payment dnů v Bing overlap windowu (4. 11. 2025 → 30. 4. 2026):

| Období | Bing signature | Multiplier vs healthy |
|--------|----------------|----------------------|
| Listopad-prosinec 2025 | 5xx **fixní 45/den** + crawled ≈ 0 | 5xx +20 %, crawled −95 % |
| Leden-únor 2026 | blocked_by_robots **15 → 555 714/den** | 37 000× nárůst |
| Březen-duben 2026 | 5xx **45 → 232/den** + blocked stále vysoký | 5xx 5×, errors 10× |

**Interpretace:** Tři typy outage (A/B/C z #48) korespondují se třemi vrstvami Bing signature. Listopad-prosinec = pure payment 500 outage. Leden-únor = URL exploration attack (bot objevuje masivní URL space). Březen-duben = combined (server overload + URL exploze pokračuje).

**Důkaz:** Kombinací GA4 funnel + Bing crawl + GSC crawl stats máme 3 nezávislé zdroje, které potvrzují stejný timeline kontinuálního problému payment + bot útoku od listopadu 2025 do dubna 2026.

---

## 16. SOUHRN AKČNÍCH KROKŮ (priorita)

### 🔴 P0 — Okamžitě (≤ 7 dní)
1. **Opravit payment 500** (#42, #48) — přímý dopad 37 k Kč/měs. Server-side debug `/page/default/dekujeme-za-vasi-objednavku` + `/platebni-stranka/`
2. **Smazat PHPMailer 5.2.1** (#44) — CVE-2016-10033 RCE riziko
3. **Vyřešit konverze v GA4 + Google Ads** (#1, #5) — opravit primární conversion event

### 🔴 P1 — Krátkodobě (≤ 30 dní)
4. **old.klicovecentrum.cz subdomain** (#43) — opravit nebo smazat
5. **Bot/crawl ochrana** (#2, #46, #49) — Cloudflare WAF rules, rate limiting
6. **Heuréka revize** (#13) — pause nebo restart kampaní
7. **Sklik LOCAL kampaně + brand budget** (#10, #10a-c)

### 🟡 P2 — Střednědobě (≤ 90 dní)
8. **URL exploze cleanup** (#45) — canonical, 301 redirecty Joomla legacy, HTTP→HTTPS finishing
9. **Feed varianty** (#15) — generovat všechny SKU
10. **Schema.org implementace** + title tagy / meta descriptions (#4)

---

*Zpráva je založena na přímém přístupu k datům přes API (GA4 Data API, Search Console API, Google Ads Scripts, Merchant Center Content API, Sklik API, Bing Webmaster API) a ručním exportu z Google Business Profile. Agentura neměla možnost ovlivnit exportovaná data.*
