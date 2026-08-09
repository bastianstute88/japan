# Japan 2027 — Reise-App

Single-File-PWA für Bastian + Simone, 23 Tage Japan (21.10.–14.11.2027).
Live: <https://bastianstute88.github.io/japan/> · App direkt: `Japan_2027.html`

**`Japan_2027.html` ist die maßgebliche Quelle für die Reise.** Notion und die lokale
`Japan Ablaufplan 2027.xlsx` sind veraltet (Excel hat noch die tote Matsumoto/Hakone-Route).
Datei-Änderungsdatum ist kein Aktualitäts-Indikator.

---

## Dateien

| Datei | Zweck |
|---|---|
| `Japan_2027.html` | Die komplette App — HTML + CSS + JS + alle Bilder als base64. ~48 MB. |
| `index.html` | Startseite mit einem Button „📅 Reiseplan öffnen" |
| `sw.js` | Service Worker: Network-first für HTML, Cache-first für Assets |
| `manifest.json`, `icon.png` | PWA-Metadaten, Icon (⛩️ auf Schwarz, 180×180) |

Alle Pfade sind **relativ** (GitHub Pages serviert unter `/japan/`, nicht unter `/`).

**Keine Backup-Dateien mehr im Ordner.** Es gab mal `*BACKUP*.html` (per `.gitignore`
ausgeschlossen) — die sind gelöscht, weil die App bei jeder Änderung committet wird und damit
**jeder Zustand in der Git-Historie liegt**. Vor riskanten Skript-Läufen trotzdem eine
temporäre Kopie anlegen, aber danach wegräumen.

Wenn `.git` groß wird (war mal bei 3,95 GB mit 520 losen Objekten): `git gc` — das packte
auf 536 MB. Nicht-destruktiv, gefahrlos. GitHub selbst packt serverseitig, dort lag das
Repo die ganze Zeit bei ~180 MB.

---

## Aufbau der App

Zwei Ebenen:

1. **🛠️ Verwaltung** — Startbildschirm mit unterer Nav. 9 Panels:
   `overview · transport · kueche · vorbuchen · dokumente · pack · phrasen · notizen · notfall`
2. **📅 Tage** — Vollbild-Overlay (`#tage.open`), geöffnet über den „📅 Reiseplan"-Button.
   Sticky Tag-Leiste oben, Hotel-Anker mittig, Boxen-Timeline mit Weg-Connectoren.
   Untere Nav ist hier ausgeblendet.

### Datenmodell (alles `const` im zweiten `<script>`-Block)

```js
DAYS = [ { n, date, city, cityKey, lat, lng, title, sub?, stops:[…], endsAtHotel? } ]
```

Ein Stop:

```js
{ t:'09:00',            // Uhrzeit
  i:'🗿',               // Emoji
  name:'Otagi Nenbutsuji',
  desc:'…',             // HTML erlaubt (<em>, <strong>)
  type:'sight',         // sight | food | transport | hotel | warning | takkyubin | birthday
  maps:'Suchbegriff',   // → Google-Maps-Button
  img:'data:…' | ['data:…','data:…'],   // optional: Galerie
  jkey:'otagi',         // optional: Schlüssel in JSTOPS → reiches Foto-Modal
  tags:['GOSHUIN'] }    // GOSHUIN | MATCHA | ND-FILTER
```

```js
WAYS = { 12:{ hop:[…], home:'…' } }
```
`hop[i]` = der Weg **zu** Stop `i`; `hop[0]` ist also Hotel → Stop 0. `home` = letzter Stop → Hotel.
Keine ¥-Preise in Wegen (nur Verkehrsmittel + Zeit).

```js
JSTOPS = { 'key': { name, desc, licht, foto, zeit, tipp, maps } }
```
Das reiche Foto-Modal. **Achtung:** Sobald ein Stop ein `img` hat, short-circuitet `openStop()`
in die Bildergalerie — das JSTOPS-Modal wird dann nicht mehr geöffnet. Kern-Infos deshalb
zusätzlich in den `desc` schreiben.

Weitere: `HOTELS` (city-keyed, mit `img`), `BOOK_LIST` (Vorbuchen), `PACK_LIST`, `PHRASES`,
`DOC_SLOTS` (+ IndexedDB `japan_docs_v4`), `TAKKYUBIN`, `TRANSIT`, `NIJO_HOTEL`.

`FOTO_SPOTS` / `FOTO_CITY_LABELS` sind **tot** — der separate Foto-Tab wurde entfernt,
die Konstanten stehen nur noch ungenutzt drin.

### Querverweise, die bei Routenänderungen mitgezogen werden müssen

Tagesangaben stehen nicht nur in `DAYS`. Wer einen Stop verschiebt, muss auch prüfen:

- **Küche-Tab** — jedes Gericht trägt `📍 Ort · Tag N`. Beim Tausch von Tag 12/13 mussten
  sechs davon mitwandern; zwei weitere (Golden Gai, Omoide Yokocho) standen schon vorher
  auf dem falschen Tag.
- **`TRANSIT`** — Notizen pro Reisetag, teils mit Uhrzeiten und Gepäckhinweisen.
- **`BOOK_LIST`** — jede Vorbuchung nennt Tag und Datum.
- **`TAKKYUBIN`** — die Kofferkette, siehe unten.

---

## Design

Schwarz/Weiß + Flaggenfarben, OLED-optimiert.

- Hintergrund **`#000000`**, Karten `#0C0A09`, Rand `#201917`
- Japan-Rot `#BC002D` nur auf **Rahmen, Linien, Unterstreichungen** — nicht auf Fließtext
- **Aller Fließtext ist weiß** (Titel, Notizen, Uhrzeiten, Links, Nav-Labels)
- Einzige legitime Farb-Ausnahme: **Grün `#34C759` für „erledigt"** (Häkchen, Fortschrittsbalken,
  Pack-Zähler). Erledigt darf nie auf den Akzent gemappt werden.
- Serif-Überschriften (Georgia)

---

## Änderungen vornehmen

**Nie mit dem Edit-Tool an großen Blöcken arbeiten** — die Datei ist 48 MB mit base64.
Immer ein Python-Skript im Scratchpad, mit `assert count == n` pro Anker, damit es laut
bricht statt still Mist zu bauen.

```python
P = "…/Japan_2027.html"
c = open(P, encoding='utf-8').read()
assert c.count(ALT) == 1
c = c.replace(ALT, NEU, 1)
open(P, 'w', encoding='utf-8').write(c)
```

Schreibe erst ganz am Ende des Skripts. Bricht ein `assert` in der Mitte, bleibt die Datei
unberührt — das hat mehrfach einen halbfertigen Zustand verhindert.

### Fallen (alle mindestens einmal teuer gelernt)

- **Byte-Offset ≠ String-Index.** Die Datei ist voll mit Emoji/Umlauten. `grep -oab` liefert
  **Byte**-Positionen, die nicht auf `c[i]` in Python passen. Stops immer im Python-String-Raum
  finden (`c.index("name:'…'")`, dann `c.rfind("{",0,i)` / `c.index("}",i)`).
- **Stops nie per Teilstring suchen.** `[Ee]ikan` trifft „Gekk**eikan**", `[Rr]yoan` trifft
  „Taxi nach Ryoan-ji". Immer den **exakten vollen Namen** als Anker, danach den getroffenen
  Namen ausgeben und gegenprüfen.
- **Maskierte Apostrophe brechen Auslese-Regexe.** `name:'Yoshi\'s Adventure'` ist gültiges JS,
  aber `name:'([^']*)'` bricht am `\'` ab und liefert `Yoshi\`. Immer `name:'((?:[^']|\\')*)'`
  verwenden — sonst hältst du eine korrekte Datei für kaputt (genau das ist einmal passiert).
- **Auch der exakte Name reicht nicht — er kommt an mehreren Tagen vor.** „Frühstück in Kyoto"
  steht an Tag 16 *und* Tag 23, „Frühstück to-go" an sechs Tagen, „Ryokan-Frühstück (La Vista
  Buffet)" an Tag 9 und 10. `c.index(...)` trifft dann stumm den falschen Tag. Jede Stop-Änderung
  **tagesweise** ausführen: erst den Block über `{ n:<N>, date:` bis zum nächsten `\n{ n:` schneiden,
  darin arbeiten, zurückschreiben.
- **Doppelter `img:`-Key.** Vor dem Einfügen prüfen, ob das Objekt schon eins hat
  (`obj.count('img:')`). Bei doppeltem Key gewinnt in JS der **letzte** → Symptom
  „nur 1 Bild, obwohl 4 im Quelltext stehen".
- **CSS-Klammer-Balance.** Eine Ersetzung, die die schließende `}` im Suchmuster mitnimmt,
  im Ersatz aber weglässt, verschluckt das gesamte folgende CSS. Ganze Regeln inkl. `}`
  matchen, danach `{` vs `}` im `<style>`-Block zählen.
- **Emoji-Escapes.** JS-Blöcke nicht als Python-`r"""raw string"""` schreiben — `\U0001…`
  bleibt sonst literal stehen und JS rendert `U0001F3E8` als Text.
- **Typografische Anführungszeichen in Python-Strings.** `„Text"` mit geradem ASCII-`"` als
  Schlusszeichen beendet einen doppelt gequoteten Python-String mitten im Satz →
  `SyntaxError`. Immer `„Text"` mit typografischem Schlusszeichen schreiben.

---

## Bilder

Offline-Pflicht → alles als base64-Data-URI direkt am Stop. Nie externe Dateien.
Aktueller Stand: **180 Stops mit Galerie**, ~435 eindeutige Bilder.

### Ablauf

1. **Quelle holen.** Bastian schickt Stop für Stop eine Handvoll Links.
2. **Ansehen — Pflicht, ausnahmslos.** Kontaktbogen per `montage`, dann jedes Bild einzeln
   beurteilen. Siehe Prüfliste unten.
3. `sips -Z 800 -s format jpeg -s formatOptions 50` → die **kleinere** von Original/Reencode
   nehmen (nie hochskalieren) → `base64`.
4. Scratchpad-Ordner pro Stop behalten — Nachjustieren („Bild 3 raus", „tausch das erste")
   geht dann ohne Re-Download.

Ein wiederverwendbares Einbettungs-Skript hat sich bewährt:
`embed.py <ordner> <exakter Stopname> <datei1> <datei2> …` — es prüft, dass der Anker genau
einmal vorkommt und dass der Stop noch kein `img:` hat, und bricht sonst ab.

### Prüfliste vor dem Einbetten

| Prüfung | Woran man es merkt |
|---|---|
| **KI-generiert** | `getyourguide.com/…/tour_img/…` ist praktisch immer KI. Sonst: zu perfekte Spiegelungen, matschige Baum-/Laubtexturen, geografisch unmögliche Kombis, krakeliger Schild-Text. |
| **Falscher Ort** | Mehrdeutige Namen sind die Hauptfalle. „Goryo-jinja" gibt es in Kamakura, Osaka **und** Kyoto — zwei gelieferte Links zeigten die falschen. „Daimon" heißt nur „großes Tor". Immer Quelle und Bildinhalt gegen den gemeinten Ort prüfen. |
| **Falsches Gericht** | Westliche Foodblogs zeigen selten das echte japanische Gericht (Okonomiyaki statt Monjayaki, Matcha-Panna-Cotta statt Parfait). |
| **Wasserzeichen** | Klook brennt es per Bildserver ein (`l_Klook_water…` in der Cloudinary-URL). Auch „Discover Kyoto", „©Kanpai-japan.com", „JAKYO" gesehen. |
| **Veraltet** | Motiv gegen den heutigen Zustand prüfen (Miyashita Park vor dem Umbau 2020 kursiert noch). |

**Wasserzeichen entfernen ist tabu** — auch nicht dadurch, dass man den Overlay-Layer aus einer
Cloudinary-URL herausnimmt. Das ist dasselbe Umgehen. Stattdessen freien Ersatz holen und
Bastian sagen, warum.

Bei **kleinen Copyright-Zeilen in der Ecke** (nicht Overlay-Logos) und wenn es nachweislich
keine freie Alternative mit dem Motiv gibt: einbauen, aber es benennen. Bastian entscheidet.

**Und immer sagen, warum ein Bild rausflog.** Bastian hat sich für die Prüfung ausdrücklich
bedankt — sie ist Teil der Arbeit, nicht Overhead.

### Quellen, die funktionieren

`upload.wikimedia.org` (frei, HD) · `japan-guide.com` · `res*.cloudinary.com/jnto` (JNTO) ·
`gltjp.com` · `kanpai-japan.com` · offizielle Tempel-/Schrein-Seiten (`yusai.kyoto`,
`todaiji.or.jp`, `hongwanji.kyoto`, `uenotoshogu.com`) · `osaka-info.jp` · `trvl-media.com` ·
`dynamic-media-cdn.tripadvisor.com` · `images.unsplash.com` · Google-Places
(`lh3.googleusercontent.com/gps-cs-s/…`)

Blocken oder liefern HTML: `web-japan.org` (hart), `cdn.jocjapantravel.com`, teils
`gotokyo.org`. Immer `curl -sL -A "<UA>" -e "<Herkunftsseite>"` verwenden.

**Wikimedia-Suche:** `action=query&generator=search&gsrnamespace=6&prop=imageinfo&iiurlwidth=1400`
→ `thumburl` nehmen. Thumb-Pfade **nie raten** (`Access Denied`). UA mit Mail setzen.
Die Suche liefert bei mehrdeutigen Namen zuverlässig die falschen Orte — Titel lesen.

**Zusammengeklebte URLs:** Bastian pastet gelegentlich mehrere Links ohne Trenner. An
`(?=https://)` splitten.

**Wiederholte Motive spiegeln** statt neu laden (Kaiseki-Dinner, Onsen, Okunoin,
Ryokan-Frühstück): `img`-Code aus dem Quell-Stop ausschneiden und im Ziel-Stop einsetzen.

### Dateigröße

Aktuell **48,3 MB** bei einem GitHub-Soft-Limit von 50 MB.

**Ein Kompressions-Durchlauf bringt nichts mehr** — gemessen an einer 40er-Stichprobe liegen
39 Bilder bereits exakt bei 800 px längster Kante, die Ersparnis wäre 0,1 % bei zusätzlichem
Generationsverlust. Der nächste echte Hebel wäre **700 px / q45**; vor dem Einsatz an einer
Stichprobe messen statt blind laufen zu lassen.

---

## QA vor „fertig"

```bash
# 1. JS parst?  (kein node nötig)
python3 -c "import re;c=open('Japan_2027.html',encoding='utf-8').read();\
open('/tmp/app.js','w').write('\n'.join(re.findall(r'<script>([\s\S]*?)</script>',c)))"
/System/Library/Frameworks/JavaScriptCore.framework/Versions/A/Helpers/jsc \
  -e 'try{new Function(readFile("/tmp/app.js"));print("JS OK")}catch(e){print("FEHLER "+e)}'

# 2. Headless-Screenshot (Tag-Overlay per openTage(n) injizieren)
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless \
  --virtual-time-budget=6000 --screenshot=out.png --window-size=430,2400 shot.html
```

Zusätzlich per Skript prüfen:

- **hop/stop-Parität pro Tag** — `WAYS[n].hop.length` muss exakt der Stop-Zahl entsprechen.
  Sonst stehen alle Wege ab der Lücke am falschen Stop. (Tag 0 hat bewusst keinen WAYS-Eintrag.)
- **Keine doppelten `img:`-Keys** pro Stop-Objekt
- **`jkey` ohne JSTOPS-Eintrag** und **verwaiste JSTOPS-Einträge**
- **Tab/Panel-Parität** — jedes `switchV('x')` braucht genau ein `id="panel-x"`. Die Nav nutzt
  `switchV`, **nicht** `switchTab`.
- **`{` vs `}` im `<style>`-Block**
- **Frühester `t:` pro Tag vs. Sonnenaufgang** — mit den drei bekannten Ausnahmen unten

### Inhaltlicher Check — bei jeder Routenänderung neu laufen lassen

Diese drei sind einmal durchgerutscht und haben echte Fehler im Plan hinterlassen:

1. **Ruhetage gegen den echten Wochentag 2027.** Nicht „ist meist offen" annehmen, sondern
   Wochentag ausrechnen und die Ruhetage nachschlagen. Gefunden: Otagi Nenbutsuji hat
   **Mi + Sa** zu und lag auf einem Mittwoch; Yusai-tei hat **Do** zu; Kyoto Gosho **Mo**;
   Kuromon-Markt hat **Sa/So/Feiertag** offiziell Ruhetag. Auch japanische Feiertage prüfen
   (3.11. = Kulturtag).
2. **Geografie über den ganzen Tag,** nicht nur von Stop zu Stop. Ein einzelner Stop mit
   „20 Min Taxi" sieht harmlos aus — hin und zurück in ein anderes Stadtviertel sind 1,5 Std.
   Und: liegt ein Stop vielleicht an einem **anderen Tag** besser? Kennin-ji löste durch einen
   Tageswechsel gleich zwei Löcher.
3. **Lücken zwischen Stops.** `t[i+1] - t[i] - Wegzeit` ausrechnen; alles über ~45 Min ist
   Leerlauf. Achtung: die reine Differenz ist kein Leerlauf-Detektor, sie enthält die
   Aufenthaltsdauer. Gegen realistische Besuchsdauern abgleichen.

---

## Die Route: Entscheidungen, die absichtlich so sind

Damit eine spätere Sitzung sie nicht für Fehler hält und „repariert":

- **Drei Tage starten vor Sonnenaufgang.** Kiyomizu-dera 06:00 an Tag 14 (öffnet 06:00, blaue
  Stunde plus leerer Tempel), das Mönchsritual 06:00 an Tag 20 (Fixtermin im Shukubo), die
  erste Miyajima-Fähre 05:15 an Tag 21 (blaue Stunde am Torii). Alle drei sind fotografisch
  gewollt.
- **Sakasa-Fuji 06:00 an Tag 10** liegt fünf Minuten vor Sonnenaufgang — der Gipfel fängt das
  Licht rund 15 Min vor dem Tal.
- **Tag 12 = Arashiyama, Tag 13 = Fushimi/Uji.** Getauscht, weil Otagi Nenbutsuji mittwochs zu
  hat und der 3.11. Kulturtag ist. Nicht zurücktauschen.
- **Ryoan-ji liegt auf Tag 23**, nicht im Arashiyama-Tag — es ist 15 Gehminuten von Kinkaku-ji.
- **Der Nozomi an Tag 23 fährt bewusst spät (17:30).** Ankunft HND 20:20, knapp vier Stunden
  vor dem Flug. Vorher waren es fünfeinhalb Stunden Wartehalle.
- **Kyoto trifft das Herbstlaub nicht.** Peak ist dort die dritte Novemberwoche, die Reise ist
  vom 1.–5. und 12./13.11. dort. Tofuku-ji und Eikan-do stehen deshalb als „optional — nur wenn
  es färbt". Getroffen werden dafür Nikko/Chuzenji (28.10.), Fuji/Chureito (30.10.) und
  Momijidani auf Miyajima (11.11.) — Höhenlagen färben früher.
- **Tag 18 ist ein kompletter USJ-Tag.** Passt nicht zu Bastians Profil, ist ihm bewusst, er
  will Super Nintendo World trotzdem. Minoo Park wurde dafür gestrichen und passt an diesen
  Daten auch nicht (Peak dort Mitte/Ende November). Nicht wieder aufmachen.
- **Tag 1 ist absichtlich locker** — Ankunft im Hotel um 00:15. Es ist Bastians Geburtstag;
  Namiki Yabu Soba ist als `type:'birthday'` markiert.
- **Tag 15 hat einen freien Nachmittag** — Simones Geburtstag nach dem Kaiseki bei Junsei.
- **Die Hotels bleiben wie sie sind — Preis/Leistung ist geprüft und abgehakt (Aug 2026).**
  Bastian hat Billig-Alternativen (APA-Kette) durchgerechnet: ~2.350 € Ersparnis möglich, aber
  11–13 m² für zwei Personen plus Fotoausrüstung, und der APA-Gründer legte in den Zimmern
  Bücher aus, die das Nanjing-Massaker leugnen. Verworfen. Entscheidend war die Einsicht, dass
  **La Vista, der Osaka Marriott und Ekoin keine Betten, sondern Programmpunkte sind**
  (Balkon-Onsen + Kaiseki / Sonnenaufgang aus 252 m + Harukas im Haus / Feuerzeremonie +
  Mönchsritual) — die stehen als eigene Stops im Plan. Nicht wieder aufmachen.
- **Tokio bleibt Asakusa, nicht Shinjuku.** Der gängige „Shinjuku als Basis"-Rat wurde geprüft
  und ist bei dieser Route ein Unentschieden: Shinjuku gewinnt Tag 2, 5, 6, 8 (~3 Std),
  Asakusa gewinnt Tag 0, 3, 7 (~2,5 Std). Ausschlaggebend gegen den Wechsel: **Senso-ji um
  07:00 leer** funktioniert nur mit Wohnsitz Asakusa, der **Tobu-Spacia nach Nikko** fährt ab
  der Hoteltür (ab Shinjuku nur 1–2 Abfahrten/Tag), und vier Abende sind auf den Stadtteil
  gebaut (Geburtstags-Soba, Goshuin-Buch, Asakusa bei Nacht, Tempura nach Kamakura).
  Merksatz: **Shinjukus Vorteil ist der Abend, Bastians Vorteil ist der Morgen.**
- **Preisstand aller Hotel-/Flugzahlen im Notizen-Tab ist 20.07.2026.** Für Okt/Nov 2027
  realistisch 10–15 % aufschlagen (~6.880 € → ~7.500–7.900 €). Beim Aktualisieren die Zeile
  „Stand …" mitziehen, sonst wirken alte Zahlen wie frische.

### Kofferkette

Muss lückenlos bleiben; jeder Check-out-Tag braucht eine Antwort.

| Tag | Lösung |
|---|---|
| 7 · 9 · 15 · 18 | Takkyubin zum jeweils nächsten Hotel (`TAKKYUBIN`-Konstante) |
| 16 | nur Tagesgepäck nach Nara — die Koffer sind seit Tag 15 in Osaka |
| 19–20 | Kōyasan nur mit Übernachtungstasche (ergibt sich aus Tag 18) |
| 22 | Check-out The Knot, Koffer an der Rezeption, nachmittags auf dem Weg zum Bahnhof abholen |
| 23 | Tages-Lieferdienst Hotel → Kyoto Bahnhof: Abgabe bis 10:00, Abholung Crosta 14:00–20:00, ~1.000¥/Stück. Steht im Vorbuchen-Tab, weil nicht jedes Haus mitmacht. Fallback: JR Nijo → Kyoto Bahnhof sind 6 Minuten. |

---

## Deployment

GitHub Pages, Repo `bastianstute88/japan` (public).

```bash
git add -A && git commit -m "…" && git push origin main
```

**Nach jeder App-Änderung selbstständig committen und pushen** — Bastian soll das nie
anfordern müssen.

### ⚠️ Jeder Push bricht den vorherigen Deploy-Lauf ab

Das ist normalerweise egal (der letzte gewinnt), wird aber zur Falle, wenn GitHubs Runner
gerade überlastet sind: dann wartet jeder Lauf minutenlang in der Warteschlange, und der
nächste Push killt genau den, der vielleicht kurz vorm Start war. In einer Sitzung wurde so
ein Dutzend Läufe hintereinander abgebrochen — **die Seite blieb stundenlang auf einem alten
Stand, obwohl jeder Push „erfolgreich" war.**

Deshalb:

- **Bei mehreren kleinen Änderungen hintereinander lokal committen und am Ende einmal pushen.**
- Nach dem Push das Repo in Ruhe lassen, bis der Lauf durch ist.
- Fehlermeldung `The job was not acquired by Runner of type hosted` = GitHub-Ausfall,
  nicht unser Fehler. Abwarten oder `gh api -X POST repos/OWNER/REPO/pages/builds`.

### Verifikation

`git rev-list --count origin/main..HEAD` == 0 beweist nur, dass der **Push** durch ist —
nicht, dass die Seite aktuell ist. Zusätzlich prüfen:

```bash
gh run list --limit 3 --json headSha,status,conclusion   # muss completed/success zeigen
curl -s --max-time 220 https://bastianstute88.github.io/japan/Japan_2027.html -o /tmp/l.html
ls -la /tmp/l.html          # Größe gegen die lokale Datei halten
grep -c "<neues Stichwort>" /tmp/l.html
```

Die ausgelieferte Datei ist etwas **größer** als die lokale (Zeilenumbruch-Overhead) — das ist
kein anderer Inhalt.

**Nicht in einer `until`-Schleife verifizieren:** jede Runde lädt 48 MB und läuft in den
Timeout, obwohl der Push längst durch ist. Stattdessen einen Hintergrund-Wächter mit
großzügigen Intervallen laufen lassen.

Netlify ist abgeschafft (Build-Minuten-Limit). Bei URL-Wechsel: alte Homescreen-Kachel
löschen und neu anlegen, sonst zeigt sie weiter auf die tote Adresse.
