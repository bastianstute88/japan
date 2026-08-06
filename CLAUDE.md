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
| `Japan_2027.html` | Die komplette App — HTML + CSS + JS + alle Bilder als base64. ~45 MB. |
| `index.html` | Startseite mit einem Button „📅 Reiseplan öffnen" |
| `sw.js` | Service Worker: Network-first für HTML, Cache-first für Assets |
| `manifest.json`, `icon.png` | PWA-Metadaten, Icon (⛩️ auf Schwarz, 180×180) |
| `*BACKUP*.html` | Sicherungen vor großen Umbauten — per `.gitignore` ausgeschlossen |

Alle Pfade sind **relativ** (GitHub Pages serviert unter `/japan/`, nicht unter `/`).

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

### localStorage / IndexedDB

- `japan_done_v1` — zweistufiges Abhaken (1. Tipp = vorgemerkt/hellgrün, 2. Tipp binnen 2 s = erledigt)
- Dokumente liegen in **IndexedDB** (`japan_docs_v4`), nicht in localStorage (5-MB-Limit).
  `DOC_EMBEDDED = {}` bleibt leer — das Repo ist öffentlich.

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

**Nie mit dem Edit-Tool an großen Blöcken arbeiten** — die Datei ist 45 MB mit base64.
Immer ein Python-Skript im Scratchpad, mit `assert count == n` pro Anker, damit es laut
bricht statt still Mist zu bauen.

```python
P = "…/Japan_2027.html"
c = open(P, encoding='utf-8').read()
assert c.count(ALT) == 1
c = c.replace(ALT, NEU, 1)
open(P, 'w', encoding='utf-8').write(c)
```

### Fallen (alle mindestens einmal teuer gelernt)

- **Byte-Offset ≠ String-Index.** Die Datei ist voll mit Emoji/Umlauten. `grep -oab` liefert
  **Byte**-Positionen, die nicht auf `c[i]` in Python passen. Stops immer im Python-String-Raum
  finden (`c.index("name:'…'")`, dann `c.rfind("{",0,i)` / `c.index("}",i)`).
- **Stops nie per Teilstring suchen.** `[Ee]ikan` trifft „Gekk**eikan**", `[Rr]yoan` trifft
  „Taxi nach Ryoan-ji". Immer den **exakten vollen Namen** als Anker, danach den getroffenen
  Namen ausgeben und gegenprüfen.
- **Maskierte Apostrophe brechen Auslese-Regexe.** `name:'Yoshi\\'s Adventure'` ist gültiges JS,
  aber `name:'([^']*)'` bricht am `\\'` ab und liefert `Yoshi\\`. Immer `name:'((?:[^']|\\\\')*)'`
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

### Bilder

Offline-Pflicht → alles als base64-Data-URI direkt am Stop. Nie externe Dateien.

1. Quelle holen (direkte URL · `imgurl=` aus einem Google-`imgres`-Link · `og:image` ·
   Wikimedia-API-`thumburl`). Getty/Dreamstime sind unbrauchbar (lizenziert, Wasserzeichen).
2. **KI-/Fake-Prüfung — Pflicht.** Jedes Bild vor dem Einbetten ansehen (Kontaktbogen per
   `montage`). Raus mit: KI-generierten Composites (`getyourguide.com/…/tour_img/…` ist
   praktisch immer KI), falschem Ort, falschem Gericht (westliche Foodblogs zeigen selten
   das echte japanische Gericht). Und immer sagen, **warum** ein Bild rausflog.
3. `sips -Z 800 -s format jpeg -s formatOptions 50` → die **kleinere** von Original/Reencode
   nehmen (nie hochskalieren) → `base64`.
4. Scratchpad-`.txt` pro Bild aufheben — Nachjustieren („Bild 3 raus") geht dann ohne Re-Download.

Wiederholte Motive (Kaiseki-Dinner, Onsen, Okunoin, Ryokan-Frühstück) **spiegeln**
statt neu laden.

Wenn die Datei gegen GitHubs 50-MB-Soft-Limit läuft: Kompressions-Pass über alle Data-URIs
(800px/q50, kleinere behalten). Vorher Backup.

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
- **Tab/Panel-Parität** — jedes `switchV('x')` braucht genau ein `id="panel-x"`. Die Nav nutzt
  `switchV`, **nicht** `switchTab`.
- **`{` vs `}` im `<style>`-Block**
- **Frühester `t:` pro Tag vs. Sonnenaufgang**

### Inhaltlicher Check — bei jeder Routenänderung neu laufen lassen

Diese drei sind einmal durchgerutscht und haben echte Fehler im Plan hinterlassen:

1. **Ruhetage gegen den echten Wochentag 2027.** Nicht „ist meist offen" annehmen, sondern
   Wochentag ausrechnen und die Ruhetage nachschlagen. Gefunden: Otagi Nenbutsuji hat
   **Mi + Sa** zu und lag auf einem Mittwoch; Yusai-tei hat **Do** zu. Auch japanische
   Feiertage prüfen (3.11. = Kulturtag → Arashiyama am Feiertag ist unbrauchbar).
2. **Geografie über den ganzen Tag,** nicht nur von Stop zu Stop. Ein einzelner Stop mit
   „20 Min Taxi" sieht harmlos aus — hin und zurück in ein anderes Stadtviertel sind 1,5 Std.
   Gefunden: Ryoan-ji hing mitten im Arashiyama-Tag, obwohl es 15 Gehminuten von Kinkaku-ji
   an Tag 23 liegt.
3. **Lücken zwischen Stops.** `t[i+1] - t[i] - Wegzeit` ausrechnen; alles über ~45 Min ist
   Leerlauf und gehört gefüllt oder zusammengezogen.

---

## Deployment

GitHub Pages, Repo `bastianstute88/japan` (public). Push = live nach ~1 Min.

```bash
git add -A && git commit -m "…" && git push origin main
```

**Nach jeder App-Änderung selbstständig committen und pushen** — Bastian soll das nie
anfordern müssen. Ein Stop-Hook in `.claude/settings.local.json` schiebt Vergessenes nach.

Live-Verifikation **nicht** in einer `until`-Schleife: jede Runde lädt 45 MB neu und läuft in
den Timeout, obwohl der Push längst durch ist. Stattdessen einmal
`git rev-list --count origin/main..HEAD` (muss 0 sein), dann ein einzelnes `curl … | grep`.

Netlify ist abgeschafft (Build-Minuten-Limit). Bei URL-Wechsel: alte Homescreen-Kachel
löschen und neu anlegen, sonst zeigt sie weiter auf die tote Adresse.
