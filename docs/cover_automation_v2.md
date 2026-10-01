# Intelligente Rollladensteuerung — Dokumentation

**Blueprint:** `automations/cover_automation_v2.yaml` · Mindestversion: Home Assistant 2024.10

[![Import Blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https://github.com/TheRealSimon42/ha-blueprints/blob/main/automations/cover_automation_v2.yaml)

## Konzept

**Eine Automation pro Fenster/Rollladen-Paar.** Du legst für jedes Fenster eine eigene
Instanz aus diesem Blueprint an und wählst dort genau einen Rollladen und — sofern
vorhanden — genau einen Fensterkontakt aus. Als Fensterkontakt funktionieren
klassische binäre Sensoren (offen/geschlossen) genauso wie Drei-Zustands-Sensoren
(offen/gekippt/geschlossen) — auch solche, die ihre Zustände großgeschrieben melden
(`Open`/`Tilted`/`Closed`, z. B. Senoro). Gemeinsame Einstellungen — die Uhrzeit fürs
morgendliche Öffnen, der Nachtmodus-Schalter, die Wetter-Entität — sind Helfer, die du
einfach in allen Instanzen identisch auswählst.

Warum so? Weil jedes Fenster eigene Eigenschaften hat (Ausrichtung, Größe, Balkontür
oder nicht) und weil damit jede Instanz für sich verständlich, testbar und abschaltbar
bleibt. Pflicht ist nur der Rollladen. Der Fenstersensor gehört zu jedem Fenster,
das sich öffnen lässt; bei Festverglasung bleibt das Feld leer, und das Fenster gilt
als immer geschlossen (siehe FAQ). Jedes Feature darüber hinaus ist per Schalter
zuschaltbar.

## Einrichtung

1. **Blueprint importieren** (Button oben) und unter _Einstellungen → Automatisierungen
   & Szenen → Blueprints_ eine Instanz pro Fenster anlegen.
2. **Rollladen + Fenstersensor** zuordnen — mehr braucht es für den Start nicht. Bei
   Festverglasung ohne Kontakt bleibt der Fenstersensor leer.
3. **Je nach gewünschten Features Helfer anlegen** (_Einstellungen → Geräte & Dienste →
   Helfer_):
   - Morgens öffnen: ein `input_datetime`-Helfer, **nur mit Uhrzeit, ohne Datum**
     (ein Datum+Zeit-Helfer feuert nur ein einziges Mal!). Einer für alle Instanzen.
     Wer nach Sonnenstand öffnen will, schaltet "Morgens nach Sonnenstand öffnen" ein —
     der Helfer ist dann optional und gilt als "frühestens".
   - Nachtmodus: ein `input_boolean`, z. B. "Nacht-Modus". Einer für alle Instanzen;
     wie er geschaltet wird (Zeitplan, Guten-Nacht-Szene, von Hand), bleibt dir überlassen.
     Alternativ pro Fenster eine eigene Nacht-Uhrzeit über einen `input_datetime`-Helfer
     (nur Uhrzeit) oder der Nachtmodus nach Sonnenstand (ganz ohne Helfer) — immer nur
     eines davon.
   - Sonnenschutz: ein `input_boolean` **pro Fenster** als Status-Speicher,
     Namensvorschlag: "Beschattung <Fenstername>".
   - Sonnenheizen: ein **weiterer** `input_boolean` pro Fenster (nicht denselben wie
     für den Sonnenschutz verwenden!).
   - Abwesenheit: kein eigener Helfer nötig — Personen, Geräte-Tracker, eine
     Personen-Gruppe oder ein vorhandener Anwesenheits-Helfer genügen. In allen
     Instanzen dieselben auswählen.
   - Diagnose (optional): ein Text-Helfer (`input_text`) **pro Fenster**, maximale
     Länge am besten 255, Namensvorschlag: "Rollladen-Status <Fenstername>".
4. Für Sonnenschutz/Sonnenheizen die **Fenstergeometrie** eintragen (Ausrichtung in
   Grad, Sichtfeld, Fensterhöhe, Brüstungshöhe; bei Vordach oder Balkon darüber die
   maximale Sonnenhöhe) — Details unten.

Fehlt ein zwingend nötiger Helfer bei aktiviertem Feature, meldet sich die Automation
selbst: Eine dauerhafte Benachrichtigung in Home Assistant benennt das betroffene
Fenster, bis der Helfer gesetzt oder das Feature deaktiviert ist.

## Die Features im Überblick

| Feature                 | Was es tut                                                                                                                                                                                  | Voraussetzung                                                    |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------- |
| Morgens öffnen          | Fährt zur eingestellten Uhrzeit oder nach Sonnenstand (frühestens/spätestens) auf die Zielposition (nur wenn geschlossener) — auf Wunsch sanft in Schritten (Sanftes Wecken)                | `input_datetime`-Helfer (nur Uhrzeit); nach Sonnenstand optional |
| Fenster-Interaktion     | Kippen → Lüftungsposition, Öffnen → ganz auf/Wunschposition (optional: wie Kippen); beim Schließen zurück; Reaktionszeit wählbar                                                            | Fenstersensor (dann immer aktiv)                                 |
| Regenschutz beim Lüften | Fährt bei Regen und gekipptem Fenster auf eine Schutzposition (nur abwärts) und nach dem Regen zurück                                                                                       | Regensensor (`binary_sensor` oder `input_boolean`)               |
| Nachtmodus              | Schließt beim Einschalten des Helfers, zur eigenen Uhrzeit oder nach Sonnenstand ganz oder auf eine Nachtposition; offene/gekippte Fenster bekommen bis zum Schließen eine Lüftungsposition | `input_boolean`, Uhrzeit-Helfer oder — (Sonnenstand)             |
| Zufallsversatz          | Morgens-Öffnen und Nachtmodus fahren zufällig bis zu X Minuten später — wirkt bei Abwesenheit bewohnt                                                                                       | — (je ein Regler, Standard 0 = aus)                              |
| Sturmschutz             | Fährt bei Starkwind hoch (oder im Panzer-Modus herunter); optional mit Auslöseverzögerung und Entwarnung nach dem Sturm                                                                     | Wetter-Entität oder Wind-Sensor                                  |
| Sonnenschutz            | Beschattet nach Sonnenstand, sodass die Sonne höchstens X m in den Raum fällt; öffnet danach wieder; optional nur bei Freigabe oder im Zeitfenster, Blendschutz auch an kühlen Tagen        | Status-Helfer, Geometrie, Temperaturquelle                       |
| Sonnenheizen            | Öffnet im Winter vergessene Rollos, wenn Sonne ins Fenster scheint und es kalt ist                                                                                                          | eigener Status-Helfer, Geometrie                                 |
| Frostschutz             | Öffnet bei Frost nur bis zu einer Maximalposition (z. B. 90 %), damit ein festgefrorener Panzer nicht reißt                                                                                 | Temperaturquelle wie beim Sonnenschutz                           |
| Moskito-Modus           | Schaltet beim Fensteröffnen nach Sonnenuntergang die Lichter im Raum aus (mit Ausnahmen)                                                                                                    | Fenstersensor (liefert auch den Bereich)                         |
| Benachrichtigungen      | Meldet zu lange offene/gekippte Fenster aufs Handy (Schließen-Button, räumt sich selbst ab); optional Erinnerungen, Alexa & Co.                                                             | Fenstersensor, Companion-App-Geräte oder Notify-Dienste          |
| Pausieren               | Hält die komplette Automation an, solange ein Helfer eingeschaltet ist — z.B. während Videoaufnahmen oder wenn Gäste schlafen                                                               | `input_boolean`-Helfer (optional)                                |
| Notfall-Öffnen          | Fährt bei Rauch-/CO-Alarm oder Hagelwarnung sofort ganz auf und hält den Rollladen oben, bis alle Sensoren wieder aus sind                                                                  | `binary_sensor`/`input_boolean` (optional)                       |
| Hindernis-Sperre        | Fährt nicht nach unten, solange ein Sperr-Sensor an ist (z.B. Fliegengittertür offen, jemand auf der Terrasse); mit Wartezeit                                                               | Kontakt-/Präsenzsensor (optional)                                |
| Nach Neustart           | Holt nach einem HA-Neustart ein verpasstes Morgens-Öffnen (bis 2 h danach) nach bzw. stellt einen aktiven Nachtmodus wieder her                                                             | — (Schalter, standardmäßig aus)                                  |
| Abwesenheit             | Schließt, wenn alle weg sind (nur bei geschlossenem Fenster); öffnet beim Heimkommen tagsüber; optional strenger beschatten                                                                 | Personen, Tracker o. Ä. (Anwesenheit)                            |
| Diagnose                | Zeigt die letzte Aktion samt Grund, z. B. "21:30 Nachtmodus → 15 % (Fenster offen)"; optional zusätzlich im Logbuch                                                                         | `input_text`-Helfer pro Fenster (optional)                       |

**Prioritäten:** Ganz oben steht das **Notfall-Öffnen** — meldet ein Notfall-Sensor
Alarm, fährt der Rollladen hoch und bleibt oben, egal was Sturmschutz, Pause oder
Nachtmodus wollen (Fluchtweg bzw. Schutz des Behangs gehen vor). Für Fahrten nach
unten gilt außerdem die **Hindernis-Sperre**: Solange ein Sperr-Sensor an ist, fährt
die Automatik den Rollladen nicht herunter, auch nicht für Nachtmodus, Panzer-Modus
oder den Benachrichtigungs-Knopf. Ansonsten gewinnt der **Sturmschutz** — bei Starkwind bewegen weder Morgens-Öffnen noch Beschattung,
Sonnenheizen oder das Zurückfahren den Rollladen, und auch der Pausier-Helfer hält
ihn nicht auf (Schutz der Hardware geht vor).
Danach kommt die Pause (solange ihr Helfer an ist, passiert sonst gar nichts),
dann der Regenschutz (bei Regen und gekipptem Fenster fährt nichts über die
Regenposition), dann der Nachtmodus (nachts wird nicht beschattet, nicht geheizt und
beim Fensteröffnen nur bis zur Lüftungsposition geöffnet), dann erst die
Komfort-Features.
Der **Frostschutz** ist keine eigene Fahrt, sondern eine Obergrenze für deren
Öffnungsfahrten — den Sturmschutz und das Notfall-Öffnen begrenzt er bewusst nicht.

Die optionale **Sturm-Entwarnung** gehört nicht zum Hardware-Schutz: Während einer
Pause entfällt sie, und sie stellt nur den Zustand her, den Nachtmodus bzw.
Morgens-Öffnen ohnehin vorgeben.
Die Abwesenheit reiht sich dort ein: Sturm und Pause gehen vor, ein offenes Fenster
verhindert das Schließen, und ein bei Abwesenheit unten stehender Rollladen wird von
Sonnenschutz und Sonnenheizen nicht wieder geöffnet.

## Verhalten verstehen

### Sichtfeld und Geometrie

![Sichtfeld-Erklärung](https://cdn.jsdelivr.net/gh/TheRealSimon42/ha-blueprints@main/images/sichtfeld_beschattung.svg)

Die Ausrichtung ist die Himmelsrichtung des Fensters in Grad (180 = Süd). Das
Sichtfeld (Standard 90° je Seite) beschreibt das geometrische Maximum: Bei 90°
Abweichung steht die Sonne in der Fassadenebene und kann das Fenster gerade noch
treffen. Verkleinere es nur, wenn real etwas im Weg steht (Laibung, Balkon,
Nachbarhaus). Sehr flach einfallende Sonne wird automatisch milder behandelt — je
schräger der Winkel, desto weiter oben bleibt der Rollladen. Links/rechts gilt von
innen am Fenster stehend: Beim Südfenster ist links die Vormittagsseite (Osten).

Nach oben begrenzt die **maximale Sonnenhöhe** (Standard 90° = keine Einschränkung)
das Sichtfeld — gedacht für Fenster mit Vordach, Balkon oder Dachüberstand darüber.
Steht die Sonne höher, gilt sie als nicht "im Fenster": Das Vordach schattet die
steile Mittagssonne ohnehin ab, Verdunkeln wäre unnötig. Steigt die Sonne mittags
über die Grenze, endet eine laufende Beschattung regulär (Fahrt auf die Position nach
der Beschattung, eine Eingriffs-Sperre fällt weg) und beginnt am Nachmittag neu,
sobald die Sonne wieder darunter sinkt. Für Sonnenheizen gilt die Grenze genauso:
Der Nachmittag zählt dort als neuer Durchgang, ein zwischendurch wieder
heruntergelassener Rollladen wird dann erneut geöffnet.

Den Wert ermittelst du am einfachsten durch Beobachten: Ab welcher Sonnenhöhe
(Attribut `elevation` von `sun.sun`) liegt die Glasfläche komplett im Schatten? Als
Faustformel: arctan(Höhe der Vordachkante über der Glas-Unterkante ÷ Vordach-Tiefe)
— ein 1,5 m tiefer Balkon, dessen Unterkante 1,6 m über der Glas-Unterkante liegt,
schattet das Glas ab etwa 47° komplett ab. Die maximale muss über der minimalen
Sonnenhöhe liegen.

Eine Hysterese gibt es an dieser Grenze bewusst nicht: Die Sonnenhöhe wird berechnet,
nicht gemessen, und überschreitet die Grenze höchstens einmal pro Tag in jede
Richtung — ein Pendeln ist ausgeschlossen. An Tagen, an denen die Sonne mittags nur
knapp über die Grenze steigt, ist die Mittagspause entsprechend kurz (in Deutschland
bei 0,1° über der Grenze rund eine halbe Stunde).

Aus Fensterhöhe, Brüstungshöhe und der maximal erlaubten Sonneneinfall-Tiefe berechnet
die Automation alle 5 Minuten die Position, bei der die Sonne höchstens bis zur
eingestellten Tiefe auf den Boden fällt, und führt den Rollladen der Sonne nach.

Standardmäßig nimmt die Berechnung an, dass die Motor-Prozente des Rollladens linear
der freien Glasfläche entsprechen. Real hat der Laufweg aber Totzonen: Unten sitzt der
Panzer irgendwann auf und nur noch die Lamellen schließen sich, oben wird nur noch in
den Kasten eingezogen. Mit der **Glas-Kalibrierung** (Aufsetz-Punkt und oberer
Glas-Endpunkt im Sonnenschutz-Abschnitt) rechnet die Beschattung in echter Glasfläche:
Aufsetz-Punkt ermitteln = Rollladen langsam herunterfahren, bis der Lichtspalt unten
gerade verschwindet. Netter Nebeneffekt: Die Beschattung fährt dann nie unter den
Aufsetz-Punkt — die Lamellen bleiben immer offen.

### Zeitfenster für die Beschattung

Mit **"Beschattung frühestens ab"** und **"Beschattung spätestens bis"** (Abschnitt
Sonnenschutz) lässt sich die Beschattung auf eine Tageszeit begrenzen. Nur innerhalb
dieses Zeitfensters startet sie und wird der Sonne nachgeführt. Endet das Zeitfenster
während einer laufenden Beschattung, beendet der nächste Tick sie regulär — genau wie
beim Verlassen des Sichtfelds: Der Status-Helfer geht aus, und der Rollladen fährt auf
die "Position nach der Beschattung". Typische Anwendungen:

- **Morgens nicht wecken:** "frühestens ab" auf 09:00 — das Ostfenster im
  Schlafzimmer fährt nicht schon beim ersten Sonnenstrahl um 6 Uhr herunter.
- **Abendsonne genießen:** "spätestens bis" auf 18:00 — danach wird nicht mehr
  beschattet, auch wenn die tiefe Westsonne noch ins Fenster scheint.

Stehen beide Felder auf derselben Uhrzeit (Standard: beide 00:00), gibt es keine
Einschränkung. Es genügt, nur eines der Felder zu setzen: "bis" auf 00:00 bedeutet
"bis Mitternacht". Die "bis"-Uhrzeit selbst gehört nicht mehr zum Zeitfenster (bei
18:00 endet die Beschattung mit dem Tick um 18:00). Liegt "ab" später als "bis",
reicht das Zeitfenster über Mitternacht. Sonnenheizen ignoriert das Zeitfenster.

### Sanftes Wecken

Gedacht fürs Schlafzimmer: Statt dass es zur Weckzeit schlagartig hell wird, fährt der
Rollladen über die eingestellte Dauer ("Sanftes Öffnen über" im Abschnitt _Morgens
öffnen_) in Schritten nach oben — ein Lichtwecker mit echtem Tageslicht. Der erste
Schritt kommt zur Weckzeit, die Zielposition ist nach Ablauf der Dauer erreicht. Damit
der Motor nicht im Sekundentakt anläuft, ist jeder Schritt mindestens 5 % groß und es
gibt höchstens einen pro Minute: Bei 30 Minuten von 0 auf 100 % sind das 20 Schritte zu
5 %, etwa alle eineinhalb Minuten. 0 Minuten (Standard) öffnet wie bisher in einem Zug.

Gefahren wird von der aktuellen Position bis zur **Zielposition für morgens** — beide
Einstellungen ergänzen sich. Wer nur bis 60 % wecken möchte, stellt die Zielposition
auf 60 %. Wie beim normalen Öffnen passiert nichts, wenn der Rollladen schon mindestens
so weit offen steht.

Das sanfte Öffnen bricht ab, sobald etwas anderes übernimmt — der Rest wird dann nicht
mehr gefahren:

- Der Rollladen wird zwischen zwei Schritten bewegt (Wandtaster, App, andere
  Automatik): Die Ist-Position weicht um 5 % oder mehr — also mindestens eine
  Schrittweite — vom zuletzt befohlenen Schritt ab. Geprüft wird erst, wenn der
  Rollladen wieder steht.
- Sturm kommt auf — der Sturmschutz hat Vorrang.
- Die Pause wird aktiv (nach ihrem Ende wird nicht weitergeweckt).
- Der Nachtmodus wird wieder eingeschaltet.
- Das Fenster wird geöffnet oder gekippt — dann übernimmt die Fenster-Interaktion.
- Die Beschattung setzt ein (heißer Sommermorgen) — sie übernimmt und läuft ganz normal
  weiter.

War der Nachtmodus beim Start noch an oder das Fenster über Nacht gekippt, stört das
nicht: Es zählt nur, was sich während des Weckens ändert. Öffnet Sonnenheizen den
Rollladen, endet das Wecken ebenfalls (die Fahrt zählt als Eingriff) — der Rollladen
steht dann auf der Sonnenheizen-Position.

Nach einem Abbruch bleibt der Rollladen dort, wo ihn der Eingriff hingefahren hat. Beim
Fenster heißt das: Wird es während des Weckens gekippt oder geöffnet und später wieder
geschlossen, fährt die Fenster-Interaktion wie gewohnt auf ihre gemerkte
Ausgangsposition zurück — typischerweise den zuletzt erreichten Schritt. Der Rest des
Weckens wird nicht nachgeholt.

Grenzen:

- Ein **Neustart** von Home Assistant oder ein **Neuladen der Automation** (z. B. nach dem
  Speichern) während des Weckens bricht es ab. Der Rollladen bleibt auf dem zuletzt
  erreichten Schritt stehen, nachgeholt wird nicht.
- **Mehrere Instanzen laufen unabhängig:** Wählen mehrere Fenster denselben
  Uhrzeit-Helfer, starten alle gleichzeitig, jede mit ihrer eigenen Dauer und
  Zielposition. Ein Eingriff an einem Rollladen bricht nur dessen Wecken ab.
- Nötig ist ein Rollladen, der seine Position und seinen Fahrzustand verlässlich meldet.
  Ohne Positionsangabe wird wie bisher in einem Zug geöffnet. Meldet ein Rollladen nach
  der Fahrt eine deutlich andere Position als befohlen oder bleibt er auf "öffnet"
  hängen, bricht das Wecken nach dem ersten Schritt ab.

### Blendschutz

Im Frühjahr, Herbst und Winter steht die Sonne tief und scheint weit in den Raum —
seitlich auf den Monitor im Arbeitszimmer oder morgens ins Ost-Badezimmer. Die
Außentemperatur liegt dann aber meist unter der Beschattungs-Schwelle, der normale
Sonnenschutz (Hitzeschutz) bleibt aus. Dafür gibt es im Sonnenschutz-Abschnitt den
Schalter **Blendschutz** (Standard: aus; wirkt nur bei aktiviertem Sonnenschutz):

- Scheint die Sonne laut Geometrie ins Fenster, wird auch **unterhalb der
  Temperatur-Schwelle** beschattet — sogar ganz ohne Temperaturquelle. Alle übrigen
  Bedingungen gelten wie beim Sonnenschutz: Wetterlagen-Filter, Sturm, Nachtmodus,
  offenes Fenster, Pausieren. Es wird derselbe Status-Helfer verwendet.
- Ziel ist die berechnete Beschattungsposition, aber **nie tiefer als die
  Blendschutz-Position** (Standard 50 %). Blendschutz ist damit immer "leichter" als
  Hitzeschutz: Bei tief stehender Sonne würde die Geometrie den Rollladen fast ganz
  schließen — der Blendschutz bleibt auf der Blendschutz-Position stehen. Liegt die
  berechnete Position höher, gilt diese.
- Der Blendschutz fährt **nur herunter**: Steht der Rollladen beim Start schon tiefer
  als das Blendschutz-Ziel (z. B. morgens noch geschlossen oder von Hand zugemacht),
  bleibt er dort. Erst wenn er geöffnet wird (morgens, von Hand), übernimmt der
  Blendschutz.
- Ist es warm genug (Temperatur über der Schwelle), gilt automatisch die normale
  Beschattung. Kühlt es um die Hysterese ab, endet die Episode **nicht**, sondern
  wechselt auf das Blendschutz-Ziel. Dazwischen (im Hysterese-Band) behält die
  Automation das Ziel bei, auf dem der Rollladen gerade steht — kein Pendeln um die
  Schwelle.
- Beendet wird die Episode wie gewohnt, wenn die Sonne das Fenster verlässt (auch
  durch Sonnenuntergang) oder die Wetterlage stabil nicht mehr passt; dann fährt der
  Rollladen auf die "Position nach der Beschattung".
- "Beschattung erzwingen" gilt auch für den Blendschutz: Der Rollladen bleibt dann
  auch bei geöffnetem Fenster unten (mindestens auf der Kipp-Position).
- Blendschutz-Position 0 % macht die Beschattung komplett temperaturunabhängig: Es
  gilt dann immer die volle berechnete Beschattung.

**Tipp:** Unbedingt mit den Wetterlagen für Beschattung kombinieren (z. B. nur
"sonnig" und "teilweise bewölkt"). Die Automation kennt nur den Sonnenstand — ohne
Filter fährt der Blendschutz auch an trüben Tagen herunter. Ein Helligkeitssensor
wird nicht ausgewertet.

### Manuelle Eingriffe während der Beschattung

Die Automation weiß nie, _wer_ den Rollladen bewegt hat — sie vergleicht bei jedem
Tick nur die Ist-Position mit ihrem berechneten Sollwert:

- Abweichung **unter 5 %**: nichts zu tun (Motorschonung).
- **5–20 %**: normales Nachführen der Sonne.
- **über 20 %**: Das kann keine Sonnenwanderung sein — ein Mensch war am Werk. Der
  Rollladen wird in Ruhe gelassen.

Diese "Sperre" gilt standardmäßig **bis zum Ende der laufenden Beschattungs-Episode**
(Sonne verlässt das Sichtfeld, es kühlt ab — nur ohne Blendschutz —, das Zeitfenster
endet, der Nachtmodus
kommt oder der Freigabe-Helfer geht aus). Das Episoden-Ende
öffnet den Rollladen dann regulär — auch über die manuelle Position hinweg. Am
nächsten Tag beginnt alles bei null; die Anfangsbewegung ist von der Toleranz
ausgenommen. Stellst du den Rollladen manuell ungefähr dorthin, wo die Beschattung
ihn haben will, übernimmt das Nachführen wieder stillschweigend. Sturm, Lüften,
Regenschutz, Morgens-Öffnen sowie Schließen und Wiederöffnen durch die Abwesenheit
zählen dagegen nicht als manuelle Eingriffe — nach ihnen darf sofort wieder beschattet werden.

Soll ein Handgriff nicht die ganze Episode lang gelten, begrenzt die **Dauer für
manuelle Eingriffe** die Sperre zeitlich (z. B. 60 Minuten): Steht der Rollladen
seit mindestens dieser Dauer still, fährt die Beschattung wieder auf ihre
Sollposition und führt danach normal nach. Jede weitere Bewegung — etwa ein
erneuter Handgriff — startet die Wartezeit neu. Das gilt in beide Richtungen: Auch
ein von Hand weiter geschlossener Rollladen fährt danach wieder auf die
Sollposition. Geprüft wird im 5-Minuten-Takt der Beschattung, die Rückkehr erfolgt
also bis zu 5 Minuten nach Ablauf. Mit 0 (Standard) bleibt es beim bisherigen
Verhalten.

Mit **Blendschutz** gibt es zwei mögliche Ziele (Hitze- und Blendschutz-Ziel). Wechselt
das gültige Ziel, weil die Temperatur die Schwelle erreicht oder um die Hysterese
darunter fällt, ist der Sprung oft größer als die Toleranz. Ein solcher Zielwechsel
zählt nicht als Handeingriff, wenn der Rollladen noch **genau** (innerhalb der
minimalen Positionsänderung) auf dem bisherigen Ziel steht und die Temperatur den
Schaltpunkt **gerade erst** überschritten hat: bis 2 °C über der Schwelle bzw. bis
1 °C unter "Schwelle minus Hysterese". Sonst zählt nur das gültige Ziel — wer im
Winter während des Blendschutzes von Hand ganz schließt, wird bis zum Episoden-Ende
in Ruhe gelassen. Die Episode umfasst dabei Blend- und Hitzeschutz gemeinsam: Ein
Handeingriff am kühlen Morgen gilt auch noch, wenn es mittags heiß wird.

Wer die Beschattung dauerhaft nicht will, deaktiviert den Schalter "Sonnenschutz
aktivieren" in der Instanz — der Status-Helfer ist **kein** Ausschalter, er ist das
interne Gedächtnis der Automation und stellt sich bei Handbetätigung einfach zurück.
Zum zeitweisen Sperren (z. B. tageweise) gibt es den optionalen Freigabe-Helfer —
siehe FAQ.

### Nachtmodus per Helfer, Uhrzeit oder Sonnenstand

Der Nachtmodus lässt sich pro Fenster auf drei Arten auslösen — **immer nur eine davon**:

- **Helfer** (`input_boolean`): der gemeinsame Schalter für alle Fenster. Die Nacht
  dauert, solange er eingeschaltet ist.
- **Uhrzeit** (`input_datetime`, nur Uhrzeit): Dieses eine Fenster schließt zu einer
  eigenen Zeit, z. B. das Kinderzimmer früher als der Rest. Die Nacht gilt dann ab
  dieser Uhrzeit bis zur Uhrzeit für morgendliches Hochfahren (sofern dort ein Helfer
  gewählt ist — auch wenn das Öffnen selbst deaktiviert ist), sonst bis Sonnenaufgang
  bzw. bis zum früheren Öffnen nach Sonnenstand.
- **Sonnenstand** (Schalter "Nachtmodus nach Sonnenstand", kein Helfer nötig): Die
  Nacht beginnt, sobald die Sonne abends unter die eingestellte Höhe sinkt — nie vor
  "frühestens" und spätestens zu "spätestens". Sie endet wie bei der Uhrzeit. Details
  im nächsten Abschnitt.

In allen Fällen verhält sich die Nacht gleich: keine Beschattung, kein Sonnenheizen,
und ein geöffnetes Fenster bekommt nur die Lüftungsposition. Uhrzeit und Sonnenstand
sind dafür als **Zustand** umgesetzt, nicht nur als Auslöser — sonst würde die
Beschattung den gerade geschlossenen Rollladen an einem heißen Abend wieder
hochfahren. Blueprints können "nur eine Quelle" im Formular nicht erzwingen. Sind
mehrere gesetzt, gilt **Helfer vor Uhrzeit vor Sonnenstand**, die Auslöser der
nachrangigen Quellen bewegen nichts, und eine dauerhafte Benachrichtigung in Home
Assistant benennt das betroffene Fenster und die gesetzten Quellen.

### Öffnen und Schließen nach Sonnenstand

Statt fester Uhrzeiten kann sich die Automation nach der **Sonnenhöhe** (Elevation,
aus der Standard-Integration "Sonne" `sun.sun`) richten — morgens im Abschnitt
"Morgens öffnen" ("Morgens nach Sonnenstand öffnen"), abends im Abschnitt "Nachtmodus"
("Nachtmodus nach Sonnenstand"). Beides ist unabhängig voneinander zuschaltbar.

Die Sonnenhöhe ist bewusst eine Gradzahl statt "Sonnenaufgang ± Minuten": Die
Dämmerung dauert im Sommer und Winter unterschiedlich lange, ein fester Versatz passt
deshalb nie das ganze Jahr. Eine Gradzahl entspricht dagegen immer etwa derselben
Helligkeit. Anhaltswerte:

| Sonnenhöhe | Bedeutung                                                          |
| ---------- | ------------------------------------------------------------------ |
| +5° … +10° | Sonne schon ein Stück über dem Horizont, es ist richtig hell       |
| 0°         | etwa Sonnenauf- bzw. -untergang (offiziell: −0,83°)                |
| −3°        | mitten in der bürgerlichen Dämmerung (Standard für den Nachtmodus) |
| −6°        | Ende der bürgerlichen Dämmerung — ohne Licht wird es dunkel        |
| −12°       | Ende der nautischen Dämmerung, praktisch Nacht                     |

**Morgens** öffnet der Rollladen genau einmal pro Tag, sobald die Sonne über die
eingestellte Höhe steigt — aber nie vor "frühestens" (die vorhandene _Uhrzeit für
morgendliches Hochfahren_, optional) und spätestens zu "Spätestens öffnen um".
Beispiel mit frühestens 07:00, spätestens 09:00 und 0°: Im Juni geht die Sonne um 5
Uhr auf, der Rollladen öffnet um 07:00. Im Dezember erreicht sie 0° erst gegen 08:15
— dann öffnet er um 08:15. Steht die Sonne bei einer höheren Schwelle (z. B. +10°) um
09:00 noch zu tief, öffnet er um 09:00. Ohne "frühestens" gibt es keine Untergrenze;
im Hochsommer öffnet der Rollladen dann entsprechend früh.

**Abends** beginnt die Nacht genau einmal pro Abend, sobald die Sonne unter die
eingestellte Höhe sinkt — nie vor "Nachtmodus frühestens um" und spätestens zu
"Nachtmodus spätestens um". Beispiel mit 18:00, 22:30 und −3°: Im Winter steht die
Sonne um 18:00 längst tief genug, der Rollladen schließt um 18:00. Im Juni sinkt sie
gegen 22:00 unter −3°, dann schließt er um 22:00. Wo es im Hochsommer kaum dunkel wird
(z. B. in Skandinavien), schließt er spätestens um 22:30. "Spätestens" darf auch kurz
nach Mitternacht liegen (z. B. 00:30), aber nicht erst in der Morgendämmerung: Steigt
die Sonne bis dahin schon wieder über die Schwelle, schließt der Rollladen ein
zweites Mal.

Liegt "frühestens" nicht vor "spätestens", gilt "frühestens": Der Rollladen fährt dann
genau zur frühestens-Uhrzeit. Home Assistant aktualisiert die Sonnenhöhe je nach
Tageszeit nur alle zwei bis acht Minuten — der Rollladen fährt also ein paar Minuten
nach dem rechnerischen Zeitpunkt.

### Mindestabstand zwischen Nachführ-Fahrten (Motorschonung)

Beim Nachführen fährt der Rollladen vor allem an Ost- und Westfenstern gern alle 5 bis
10 Minuten ein Stück. Wer das seltener möchte, hat im Sonnenschutz-Abschnitt zwei
Stellschrauben: Die **Minimale Positionsänderung** legt fest, wie groß ein
Nachführ-Schritt mindestens sein muss, der **Mindestabstand zwischen
Nachführ-Fahrten** (Standard 0 = aus), wie lange der Rollladen vorher stillgestanden
haben muss.

Gemessen wird die Zeit seit der letzten Bewegung — egal, wer sie ausgelöst hat
(Automatik, Wandtaster, App). Grundlage ist der Zeitstempel `last_updated` des
Rollladens, weil Teilfahrten bei vielen Aktoren nur das Attribut `current_position`
ändern und den Zustand ("offen") stehen lassen. Fällt eine Fahrt in den Abstand, wird
sie übersprungen; die erste Prüfung nach Ablauf holt sie nach. Geprüft wird im
5-Minuten-Takt, deshalb lässt sich der Abstand in 5er-Schritten einstellen.

Der Abstand gilt bewusst nur für die beiden periodischen Komfort-Fahrten: das
**Nachführen** einer laufenden Beschattung und das **Öffnen durch Sonnenheizen**.
Alles, was schützt oder auf ein Ereignis reagiert, fährt weiterhin sofort:
Sturmschutz, Nachtmodus, Morgens öffnen, Fenster-Interaktion, der "Rollladen
schließen"-Knopf — und **Beginn und Ende einer Beschattung**. Diese beiden sind
keine Feinkorrektur, sondern der eigentliche Zustandswechsel und schalten den
Status-Helfer um: Verzögert hieße das Sonne im Raum bzw. unnötig lange Dunkelheit,
und eine ausgelassene Fahrt bei schon umgeschaltetem Helfer sähe beim nächsten Tick
wie ein manueller Eingriff aus (Beginn) bzw. ließe den Rollladen unten (Ende).

**Wechselwirkung mit der Eingriffs-Erkennung:** Während des Abstands wandert die
Sonne weiter, der nachgeholte Schritt fällt also größer aus als sonst. Überschreitet
er die _Toleranz für manuelle Eingriffe_ (Standard 20 %), wertet die Automation ihn
als Handbetätigung und lässt den Rollladen bis zum Beschattungs-Ende stehen. Als
Richtwert (Deutschland, Standard-Geometrie): Bei Ost- und Westfenstern verschiebt
sich die Sollposition morgens bzw. abends um bis zu etwa 25 % in 10 Minuten und 35 %
in 15 Minuten, bei Südfenstern deutlich weniger. Eine kleinere Sonneneinfall-Tiefe
und sehr schräg einfallende Sonne beschleunigen das. Bei längeren Abständen also die
Toleranz anheben (bis 60 %) oder den Abstand kürzer wählen.

### Zufallsversatz und Anwesenheitssimulation (Urlaub)

"Morgens öffnen" und "Nachtmodus" haben je einen Regler **Zufällige Verzögerung
(max.)**. Steht er z. B. auf 20 Minuten, würfelt die Automation bei jedem Auslösen
neu eine Wartezeit zwischen 0 und 20 Minuten (sekundengenau) und fährt erst danach —
nur später, nie früher. Das hat zwei Effekte: Bei Abwesenheit wirkt das Haus bewohnt,
weil die Rollläden nicht jeden Tag zur exakt gleichen Minute fahren, und mehrere
Rollläden fahren nicht im Gleichschritt, weil jede Instanz für sich würfelt.
Standard ist 0 = sofort, also das bisherige Verhalten.

Nach der Wartezeit bewertet die Automation die Lage neu, statt blind zu fahren:

- **Pause:** Ist die Automatik inzwischen pausiert, entfällt die Fahrt. Morgens wird
  sie wie jedes verpasste Einzelereignis nicht nachgeholt; einen noch aktiven
  Nachtmodus holt das Pause-Ende wie gewohnt nach.
- **Nachtmodus wieder aus:** Die Nachtfahrt entfällt. Wurde der Helfer zwischendurch
  aus- und wieder eingeschaltet, fährt nur der neuere Lauf. Den Status-Helfer des
  Sonnenschutzes setzt der Nachtmodus erst nach der Wartezeit zurück — beschattet
  wird währenddessen trotzdem nicht mehr (der Nachtmodus ist ja schon an), und
  entfällt die Fahrt, endet eine laufende Beschattung regulär, statt in
  Beschattungsposition hängen zu bleiben.
- **Sturm:** Der Wind wird erst nach der Wartezeit geprüft. Morgens wird bei
  Starkwind wie bisher nicht geöffnet. Die verzögerte Nachtfahrt entfällt bei
  Starkwind ebenfalls, der Rollladen wird dann nicht bewegt — außer im Panzer-Modus,
  denn dort ist Schließen ohnehin die Schutzrichtung. Ohne Verzögerung schließt der
  Nachtmodus wie bisher auch bei Wind.
- **Beschattung am Morgen:** Hat die Beschattung erst während der Wartezeit
  begonnen, entfällt das Öffnen — sonst führe der Rollladen hoch und beim nächsten
  Tick gleich wieder herunter.
- Ein während der Nacht-Wartezeit geöffnetes Fenster wird bereits nach den
  Nacht-Regeln behandelt (nur Lüftungsposition statt ganz auf). Wird ein zum Lüften
  geöffnetes Fenster in dieser Zeit innerhalb des Zurückfahr-Zeitfensters
  geschlossen, fährt der Rollladen sofort zu, ohne die Wartezeit abzuwarten.

**Urlaub mit eigenen Zeiten:** Für einen echten Urlaubsmodus — z. B. später öffnen
und früher schließen — legst du für denselben Rollladen eine zweite Instanz an und
schaltest beide über einen gemeinsamen "Urlaub"-Helfer im Abschnitt "Pausieren"
gegeneinander: in der Alltags-Instanz "AN pausiert", in der Urlaubs-Instanz
"AUS pausiert". So ist genau eine der beiden aktiv — vorausgesetzt, der
Urlaub-Helfer ist in beiden der einzige Pausier-Helfer (bei mehreren pausiert eine
Instanz nur, wenn alle im Pausier-Zustand sind) und verfügbar (ein fehlender Helfer
pausiert nie, dann laufen beide). In der Urlaubs-Instanz
eigene Helfer (Uhrzeit, ggf. Nachtmodus) wählen und die Zufallsverzögerung setzen;
Sonnenschutz und Sonnenheizen dort entweder aus lassen oder mit eigenen
Status-Helfern betreiben. Den Sturmschutz in beiden gleich einstellen — er greift
auch in der pausierten Instanz. Den Urlaub-Helfer schaltest du selbst oder per
Automation (z. B. wenn morgens niemand zu Hause ist).

### Abwesenheit & Urlaub

Im Abschnitt "Abwesenheit" wählst du aus, woran die Automation erkennt, ob jemand
zuhause ist: Personen, Geräte-Tracker, eine Personen-Gruppe, die Zone "Zuhause"
(zählt die Personen darin) oder einen eigenen Helfer bzw. Binärsensor wie "Jemand
zuhause". **Abwesend** ist das Haus erst, wenn _alle_ ausgewählten Entitäten "weg"
melden — und zwar ununterbrochen für die Wartezeit (Standard 10 Minuten). Ein
Zonenwechsel unterwegs ("Arbeit" → unterwegs) startet die Wartezeit nicht neu. Eine
nicht verfügbare oder unbekannte Entität zählt als anwesend: lieber einmal nicht
schließen, als dass der Rollladen wegen eines Tracker-Aussetzers fährt.

Was dann passiert, ist einzeln zuschaltbar:

- **Bei Abwesenheit schließen:** Der Rollladen fährt einmal auf die
  Abwesenheits-Position (Standard 0 %) — nur abwärts und nur, wenn das Fenster sicher
  geschlossen ist. Ein offenes oder gekipptes Fenster bzw. eine offene Balkontür
  verhindert das Schließen (Aussperr-Schutz: vielleicht steht doch jemand ohne Handy
  draußen). Bei Sturm hat der Sturmschutz Vorrang, während einer Pause passiert
  nichts. Solange niemand da ist und der Rollladen auf oder unter der
  Abwesenheits-Position steht, öffnen ihn auch Sonnenschutz und Sonnenheizen nicht.
- **Heimkommen:** Sobald wieder jemand sicher zuhause ist, öffnet der Rollladen auf
  die Zielposition für morgens — aber nur, wenn er noch auf oder unter der
  Abwesenheits-Position steht, der Nachtmodus aus ist, die Sonne über dem Horizont
  steht und kein Sturm herrscht. Abends und nachts bleibt er also zu. Die Beschattung
  bewertet danach der nächste Durchlauf neu. Wieder geöffnet wird nur nach einer
  echten Abwesenheit (Wartezeit abgelaufen) — kurz zum Briefkasten und zurück bewegt
  nichts. Maßgeblich ist die Position, nicht wer ihn heruntergefahren hat: Auch ein
  schon vor dem Verlassen von Hand geschlossener Rollladen öffnet beim Heimkommen.
  Wer das für ein Fenster nicht möchte, lässt dort "Bei Abwesenheit schließen" aus —
  das Wiederöffnen gehört zu diesem Schalter.
- **Strenger beschatten:** Während der Abwesenheit gilt für den Sonnenschutz eine
  eigene, kleinere maximale Sonneneinfall-Tiefe (Standard 0,3 m) — mehr Hitzeschutz,
  wenn ein dunklerer Raum niemanden stört. Der Wechsel beim Verlassen und beim
  Heimkommen zählt nicht als manueller Eingriff. Steht der Rollladen bei Abwesenheit
  ohnehin unten, gibt es nichts zu beschatten; spürbar wird die strenge Tiefe, wenn
  das Schließen aus ist oder der Rollladen inzwischen wieder offen ist (z. B. nach
  dem morgendlichen Öffnen im Urlaub).

**Urlaub:** Geschlossen wird einmal beim Verlassen. Morgens-Öffnen und Nachtmodus
laufen während der Abwesenheit normal weiter — die Rollläden fahren also auch im
Urlaub morgens hoch und abends herunter, statt tagelang unten zu bleiben, und das Haus
wirkt bewohnt. Einen eingebauten Zufallsversatz der Fahrzeiten gibt es nicht; wer
ihn möchte, kann die gemeinsamen Helfer (Uhrzeit für morgens, Nachtmodus) während
des Urlaubs mit einer eigenen Automation zu leicht wechselnden Zeiten schalten.

Damit das Heimkommen nur nach einer echten Abwesenheit öffnet, merkt sich die
Automation die laufende Abwesenheit in einer temporären Szene
`scene.away_<rollladen>`. Sie wird beim Heimkommen automatisch wieder entfernt und
sollte nicht von Hand gelöscht oder aktiviert werden.

### Regenschutz beim Lüften

Mit einem Regensensor (binärer Sensor oder `input_boolean`, "an" = Regen) schützt die
Automation gekippte Fenster vor hereinlaufendem Regen — praktisch, wenn im Sommer die
ganze Nacht über gelüftet wird. Ohne Regensensor ist das Feature aus.

- **Regen beginnt:** Meldet der Sensor mindestens die eingestellte Verzögerung lang
  Regen (Standard 2 Minuten) und ist das Fenster gekippt, merkt sich die Automation
  die aktuelle Position und fährt auf die Regenposition (Standard 10 %). Es wird nur
  heruntergefahren — steht der Rollladen schon tiefer, bleibt er, wo er ist.
- **Fenster wird bei Regen gekippt:** Der Rollladen fährt nicht zum Lüften hoch;
  steht er über der Regenposition, fährt er auf die Regenposition herunter (gemerkt
  wird die Position vor dem Kippen). Eine so ausgelassene Kipp-Position wird nach dem
  Regen nicht nachgeholt.
- **Regen vorbei:** Ist es lange genug trocken (Standard 15 Minuten) und das Fenster
  noch gekippt, fährt der Rollladen auf die gemerkte Position zurück. Bei aktivem
  Nachtmodus, Sturm oder Pause bleibt er in der Regenposition. Kurze Regenpausen
  innerhalb der Trocken-Dauer überschreiben die gemerkte Position nicht.
- **Fenster wird vorher geschlossen:** Die gemerkte Position wird verworfen, das
  Zurückfahren übernimmt die normale Fenster-Interaktion (innerhalb ihres
  Zeitfensters, siehe Bekannte Grenzen).

Standardmäßig gilt das nur für gekippte Fenster (bzw. offene mit "Öffnen wie Kippen
behandeln"). Mit "Auch bei ganz geöffnetem Fenster" greift der Regenschutz auch bei
komplett geöffnetem Fenster — für Balkon- und Terrassentüren nicht empfohlen,
Aussperr-Gefahr!

Solange der Regenschutz greift, fährt kein anderes Feature den Rollladen darüber
hinaus: Beschattung und Sonnenheizen ruhen (ihr Sonnenstand ist reine Geometrie und
sagt nichts über echten Sonnenschein), der Nachtmodus fährt höchstens auf die
Regenposition, und das Morgens-Öffnen wird vorgemerkt — beim Zurückfahren nach dem
Regen geht es dann direkt auf die Morgen-Position. Die Regenfahrt zählt nicht als
manueller Eingriff in die Beschattung. Der Sturmschutz hat Vorrang: Bei Starkwind
fährt der Regenschutz nicht herunter (außer im Panzer-Modus, dort ist Herunterfahren
ohnehin die Schutzrichtung). Regnet es beim Ende einer Pause, wird der Regenschutz
nachgeholt.

**Kein Regensensor, aber eine Wetter-Entität?** Ein Template-Binärsensor
(_Einstellungen → Geräte & Dienste → Helfer → Helfer erstellen → Template →
Binärsensor_) macht daraus einen Regensensor, z. B. mit dem Zustands-Template:

```jinja
{{ states('weather.home') in ['rainy', 'pouring', 'lightning-rainy', 'snowy-rainy', 'hail'] }}
```

Genauso lässt sich ein Regenmengen-Sensor einbinden, z. B.
`{{ states('sensor.regenrate') | float(0) > 0 }}`.

### Benachrichtigungen, Erinnerungen und Zusatz-Dienste

Die erste Meldung kommt, wenn das Fenster länger als die eingestellte Zeit offen
bzw. gekippt ist (beide Zeiten getrennt einstellbar). Auf dem Handy trägt sie
einen "Rollladen schließen"-Knopf und verschwindet von selbst, sobald das Fenster
geschlossen wird.

**Erinnerungen:** Ist "Erinnerung alle X Min. wiederholen" größer als 0, kommt die
Meldung erneut, solange das Fenster in derselben Stellung bleibt — höchstens so oft
wie unter "Maximale Anzahl Erinnerungen" eingestellt. Beispiel: offen nach 45
Minuten, Abstand 15, maximal 3 → Meldungen nach 45, 60, 75 und 90 Minuten. Der Text
nennt jeweils die aktuelle Dauer ("seit über 75 Minuten offen"). Auf dem Handy
ersetzt jede Erinnerung die vorherige, der Knopf bleibt. Die Erinnerungen enden:

- **sofort**, wenn das Fenster geschlossen wird,
- wenn das Fenster von offen auf gekippt wechselt (oder umgekehrt) oder der Sensor
  kurz aussetzt — für die neue Stellung kommt nach deren eigener Wartezeit wieder
  eine erste Meldung mit eigenen Erinnerungen,
- nach der letzten erlaubten Erinnerung.

Der "Rollladen schließen"-Knopf schließt nur den Rollladen — solange das Fenster
offen bleibt, kommen die Erinnerungen weiter.

Fällt eine Erinnerung in den Schlafmodus oder eine Pause, entfällt sie, zählt aber
mit; ist danach noch eine übrig, kommt sie ganz normal. Ist der Schlafmodus (oder
die Pause) schon aktiv, wenn die erste Meldung fällig wird, entfallen die Meldung
und ihre Erinnerungen ganz — wie bisher.

**Zusatz-Dienste:** Unter "Zusätzliche Benachrichtigungs-Dienste" lassen sich
weitere Notify-Dienste eintragen, z. B. `notify.alexa_media_kueche`,
`notify.persistent_notification` oder der Dienst eines Telegram-Bots — mit oder ohne
`notify.`-Präfix. Integrationen, die statt eines Dienstes eine
Benachrichtigungs-Entität (`notify.…`) anlegen, funktionieren ebenfalls; die
Automation erkennt das selbst. Zusatz-Dienste bekommen Titel und Text, bei der
ersten Meldung und bei jeder Erinnerung, aber keinen Knopf und keine Tag-Daten
(fremde Dienste kennen sie nicht). Schlafmodus und Pause gelten auch für sie.

Ein falscher Eintrag stört die übrigen Ziele nicht: Einträge im falschen Format
(Leerzeichen, Umlaute, anderer Bereich als `notify`) werden übersprungen und im
Home-Assistant-Log als Warnung genannt. Gibt es einen Dienst nicht (Tippfehler),
steht der Fehler im Log und im Trace — die übrigen Ziele und die Erinnerungen
laufen weiter. Dafür wird jeder Zusatz-Dienst in einem eigenen, kurzen Lauf der
Automation bedient; diese Läufe tauchen deshalb zusätzlich in den Traces auf.

Hängen mehrere Rollläden an einem Fenster (mehrere Instanzen mit demselben
Fenstersensor), meldet jede Instanz für sich. Auf dem Handy ersetzt die zweite
Meldung dank gleichem Tag die erste, Zusatz-Dienste bekämen die Nachricht aber
mehrfach — trage sie dann nur in einer der Instanzen ein.

### Ein- und Ausschaltverzögerung (Quellwolken)

An Tagen mit Quellwolken wechseln Sonne und Schatten im Minutentakt. Ohne Verzögerung
entscheidet die Automation bei jedem 5-Minuten-Takt neu — springen Temperatur oder
Wetterlage dabei hin und her, fährt der Rollladen ständig auf und zu. Dagegen helfen
zwei Einstellungen im Sonnenschutz-Abschnitt:

- **Einschaltverzögerung:** Die Beschattung startet erst, wenn Sonnenstand
  (Sonne im Fenster), Temperatur (über der Schwelle) und Wetterlage (erlaubt) sie so
  viele Minuten **ununterbrochen** erlauben. Jede Unterbrechung startet die Wartezeit
  neu.
- **Ausschaltverzögerung:** Die Beschattung endet erst, wenn ein Ende-Grund (Sonne
  nicht mehr im Fenster, Temperatur unter Schwelle minus Hysterese, nicht erlaubte
  Wetterlage) so viele Minuten ununterbrochen besteht. Bis dahin bleibt der Rollladen
  stehen.

Bewährt hat sich ein asymmetrisches Paar wie **5 Minuten Ein, 20 Minuten Aus**:
schnell schützen, zögerlich wieder öffnen. 0 (Standard) heißt keine Verzögerung —
die Entscheidung fällt wie bisher beim nächsten 5-Minuten-Takt.

Gut zu wissen:

- Die Wartezeit bezieht sich nur auf Sonne, Temperatur und Wetterlage. Spricht nach
  ihrem Ablauf noch etwas anderes gegen die Beschattung (Fenster offen, Nachtmodus,
  Sturm, Pause, Notfall, Freigabe aus, außerhalb des Zeitfensters, Regenschutz,
  Abwesenheit, Hindernis), merkt sich die Automation den Ablauf und beschattet beim
  nächsten 5-Minuten-Takt, sobald der Hinderungsgrund weg ist — ohne erneute
  Wartezeit, solange Sonne, Temperatur und Wetterlage durchgehend gepasst haben.
- Die Ausschaltverzögerung gilt für das reguläre Ende (Sonne, Temperatur,
  Wetterlage) — auch wenn die Sonne abends planmäßig das Sichtfeld verlässt; der
  Rollladen öffnet dann entsprechend später. Nachtmodus, Sturm, Lüften,
  Morgens-Öffnen, das Ende einer Pause, eine ausgeschaltete Freigabe und das Ende des
  Zeitfensters greifen dagegen wie bisher sofort.
- Den Ablauf der Wartezeit merkt sich die Automation in einer temporären Szene
  (`scene.shading_start_…` bzw. `scene.shading_end_…`); sie verschwindet von selbst,
  sobald Sonne, Temperatur oder Wetterlage nicht mehr passen.
- Das Nachführen einer laufenden Beschattung bleibt beim 5-Minuten-Takt.
- Ist die Temperaturquelle kurz nicht verfügbar, zählt das weder als Start- noch als
  Ende-Grund; eine laufende Wartezeit beginnt danach von vorn.
- Mit gesetzter Ausschaltverzögerung ersetzt diese die feste 10-Minuten-Trägheit des
  Wetterlagen-Filters.
- Nach einem Neustart beginnen die Wartezeiten von vorn (siehe Bekannte Grenzen).

### Warum die Status-Helfer nötig sind

Blueprints haben keinen eigenen Speicher, und bei Funk-Rollläden lässt sich aus den
Zustandsdaten nicht ablesen, ob die letzte Bewegung von der Automation oder vom
Wandtaster kam (die Positions-Rückmeldung kommt immer vom Gerät selbst). Ein
`input_boolean` pro Fenster ist der einzige Weg, "die Beschattung läuft gerade"
neustartfest zu speichern — und genau darauf bauen das automatische Wiederöffnen,
die Einmal-Logik des Sonnenheizens und die Eingriffs-Erkennung auf.

### Benachrichtigungen nur an Anwesende

Standardmäßig geht die Fenster-offen-Meldung an alle ausgewählten Geräte. Mit dem
Schalter "Nur anwesend" im Abschnitt Benachrichtigungen bekommt ein Gerät sie nur
noch, wenn seine Person gerade zuhause ist — wer unterwegs ist, kann am Fenster
ohnehin nichts ändern. Ein zusätzlicher Helfer ist dafür nicht nötig: Die Automation
sucht zu jedem ausgewählten Gerät den Geräte-Tracker der Companion App. Ist dieser
Tracker einer Person zugeordnet, zählt der Status der Person (`person.*`), sonst der
des Trackers selbst — gesendet wird nur bei `home`. Geräte ohne Tracker bekommen die
Meldung weiterhin immer. Das automatische Entfernen der Meldung beim Schließen des
Fensters geht unabhängig davon an alle Geräte, damit nichts auf einem Handy hängen
bleibt. Bei Erinnerungen wird vor jeder Runde neu geprüft — wer inzwischen
heimgekommen ist, bekommt die nächste Erinnerung; ist gerade niemand zuhause, entfällt
nur diese Runde. Zusatz-Dienste (Alexa, Telegram …) haben keinen Tracker und werden
nicht nach Anwesenheit gefiltert.

### Nachtmodus und Nachtposition

Beginnt der Nachtmodus (per Helfer oder Uhrzeit), richtet sich der Rollladen nach dem Fenster:
gekippt → Kipp-Position, offen → "Nachtposition bei offenem Fenster", geschlossen →
"Nachtposition bei geschlossenem Fenster". Die steht standardmäßig auf 0 % — der
Rollladen schließt dann wie bisher ganz. Mit einem höheren Wert bleibt nachts ein Spalt
offen, etwa als Lüftungsschlitz oder weil der Rollladen gar nicht ganz zufahren soll.

Die Nachtposition wird nur **abwärts** angefahren: Steht der Rollladen schon tiefer
(z. B. abends von Hand ganz geschlossen), bleibt er dort — nachts wird nie wieder
teilweise geöffnet. Dieselbe Regel gilt, wenn ein gelüftetes Fenster geschlossen wird
und inzwischen der Nachtmodus aktiv ist, und wenn das Ende einer Pause den Nachtmodus
nachholt. Ganz geschlossen wird dagegen weiterhin beim Sturmschutz im Panzer-Modus,
bei "Schließen erzwingen" und über den "Rollladen schließen"-Knopf einer
Benachrichtigung. Rollläden ohne Positions-Angabe schließen nachts immer ganz.

### Frostschutz

Bei Frost kann der Panzer im Kasten oder in den Führungsschienen festfrieren — fährt
der Motor dann ganz hoch, reißt er daran. Mit aktiviertem Frostschutz öffnet die
Automatik deshalb höchstens bis zur Frost-Position (Standard 90 %), solange die
Außentemperatur bei oder unter der Schwelle liegt (Standard 0 °C). Das betrifft:

- **Morgens öffnen:** Ziel ist der kleinere Wert aus Zielposition und Frost-Position;
  gefahren wird weiterhin nur, wenn der Rollladen tiefer steht.
- **Fenster öffnen:** Der Rollladen fährt auf die Frost-Position statt ganz auf —
  steht er schon höher, bleibt er stehen.
- **Ende der Beschattung** und **Sonnenheizen:** Die jeweilige Zielposition wird auf
  die Frost-Position begrenzt.

Die Temperatur kommt aus derselben Quelle wie bei Sonnenschutz und Sonnenheizen: dem
Außentemperatur-Sensor im Sonnenschutz-Abschnitt, sonst dem `temperature`-Attribut der
Wetter-Entität im Sturmschutz-Abschnitt. Beide Felder wirken auch, wenn Sonnenschutz
bzw. Sturmschutz selbst ausgeschaltet sind. Ist keine Quelle gesetzt oder gerade nicht
verfügbar, greift der Frostschutz nicht.

Eine Hysterese braucht es nicht: Die Temperatur wird nur im Moment einer Öffnungsfahrt
geprüft, es gibt keinen Dauerzustand, der beim Über- oder Unterschreiten der Schwelle
nachgefahren würde. Pendelt die Temperatur um die Schwelle, entscheidet sie nur, ob die
nächste Öffnung an der Frost-Position oder an der normalen Zielposition endet — ein
Auf und Ab entsteht dadurch nicht. Umgekehrt holt die Automation nach dem Frost nichts
nach: Der Rollladen bleibt auf der Frost-Position, bis ihn das nächste reguläre
Ereignis bewegt.

Der **Sturmschutz** fährt auch bei Frost ganz hoch. Ein teilweise heruntergelassener
Panzer bietet dem Wind Angriffsfläche und schlägt in den Schienen, der eingefahrene
Panzer ist im Kasten geschützt — Schutz vor Wind hat Vorrang. Im Panzer-Modus
(schließen bei Sturm) stellt sich die Frage ohnehin nicht. Ebenso fährt das
**Notfall-Öffnen** immer ganz auf — Fluchtweg und Zugang für die Feuerwehr gehen vor.

### Notfall-Öffnen (Rauchmelder, Hagelwarnung)

Im Abschnitt "Notfall-Öffnen" wählst du einen oder mehrere binäre Sensoren oder
`input_boolean`-Helfer aus — typischerweise Rauch- und CO-Melder, aber auch eine
Hagelwarnung (z. B. von einem Hagelschutz-Dienst). Sobald einer davon `on` meldet,
fährt der Rollladen ganz auf: als Fluchtweg und Zugang für die Feuerwehr bzw.
damit der Behang nicht vom Hagel beschädigt wird. Das passiert ohne Rücksicht auf
Fenster, Nachtmodus, Pause und Sturmschutz — auch im Panzer-Modus. Liegt beim
Start von Home Assistant bereits ein Alarm an, wird ebenfalls geöffnet. Auch ein
Probealarm (Testknopf am Rauchmelder) öffnet den Rollladen.

Solange mindestens ein Sensor `on` ist, bewegt die Automation den Rollladen nicht:
kein Nachtmodus, keine Beschattung, kein Sturmschutz, kein Morgens-Öffnen, und
auch der "Rollladen schließen"-Knopf der Benachrichtigung bleibt wirkungslos. Der
Moskito-Modus schaltet keine Lichter aus; Fenster-Benachrichtigungen laufen
weiter. Ein gerade laufendes Lüften wird durch den Alarm beendet — die
Ausgangsposition wird danach nicht wiederhergestellt. Den Wandtaster blockiert
die Automation nicht: Wer den Rollladen von Hand herunterfährt, wird erst durch
einen erneuten Alarm wieder übersteuert.

Melden alle Sensoren wieder `off`, holt die Automation einen inzwischen aktiven
Nachtmodus nach (bei offenem oder gekipptem Fenster mit Lüftungsposition, während
einer Pause erst zu deren Ende). Hält ein Sturm an, hat er Vorrang: Im Panzer-Modus
wird das vom Notfall verhinderte Schließen nachgeholt, sonst bleibt der Rollladen
oben. Die Beschattung bewertet
der nächste Tick frisch. Die Position vor dem Alarm wird nicht wiederhergestellt,
weitere verpasste Ereignisse werden nicht nachgeholt.

Licht einschalten, Türen entriegeln oder Durchsagen gehören bewusst nicht in
dieses Blueprint — es steuert nur seinen eigenen Rollladen. Dafür eine eigene
Automation auf dieselben Sensoren anlegen; dort lässt sich auch entscheiden, ob
im Brandfall überhaupt Strom geschaltet werden soll.

### Hindernis-Sperre (Fliegengittertür, Terrasse)

Manche Hindernisse sieht der Fensterkontakt nicht: eine offene Fliegengittertür vor
der Balkontür (der Panzer läuft auf und hakt im Kasten aus) oder jemand, der abends
auf der Terrasse sitzt, während die Tür längst zu ist. Dafür gibt es die
Sperr-Sensoren — ein oder mehrere `binary_sensor`- oder `input_boolean`-Entitäten
(Türkontakt, Präsenzmelder, ein Schalter "Terrasse besetzt"). Solange einer davon
**an** ist, fährt die Automatik den Rollladen **nicht nach unten**:

- **Gesperrt:** Nachtmodus (auch Lüftungs- und Kipp-Position), das Nachholen nach dem
  Pause-Ende, Beschattung (Start, Nachführen und ein Ende auf eine tiefere Position),
  Sturmschutz im Panzer-Modus, "Schließen erzwingen", das Zurückfahren nach dem
  Lüften und der "Rollladen schließen"-Knopf einer Benachrichtigung — Sicherheit vor
  Komfort, vom Handy aus sieht man die offene Fliegengittertür nicht.
- **Weiter erlaubt:** alle Fahrten nach oben — morgens öffnen, Fenster öffnen/kippen,
  Sonnenheizen, Sturm ohne Panzer-Modus, Beschattungs-Ende nach oben.

Die **Wartezeit bis zur Freigabe** verlängert die Sperre: Erst wenn alle Sperr-Sensoren
so lange aus sind, gilt sie als aufgehoben — praktisch für Präsenzmelder, die
zwischendurch kurz "frei" melden. Danach stellt die Automation einen eingeschalteten
Nachtmodus her (fensterabhängig wie beim Pause-Ende; bei anhaltendem Sturm hat der
Sturmschutz Vorrang, im Panzer-Modus wird dann geschlossen) und gibt den
Beschattungs-Status frei, damit der nächste Tick frisch beschattet. Endet die Sperre
während einer Pause, holt erst das Pause-Ende den Nachtmodus nach — nur das
Panzer-Schließen bei Sturm kommt sofort, denn der Sturmschutz durchbricht auch die
Pause.

**Nicht verfügbar heißt gesperrt:** Meldet ein Sperr-Sensor `unavailable` oder
`unknown` (leere Batterie, Funkproblem, gelöschte Entität), gilt das als Hindernis.
"Unbekannt" ist bei einer Fliegengittertür kein Beweis für "zu", und ein ausgehakter
Panzer ist teurer als ein Rollladen, der eine Nacht oben bleibt.

### Sturmschutz: Auslöseverzögerung und Entwarnung

Mit der **Auslöseverzögerung** muss der Wind eine einstellbare Zeit ununterbrochen über
dem Grenzwert liegen, bevor der Sturmschutz den Rollladen bewegt — einzelne Böen lösen
dann nicht mehr sofort aus. Morgens-Öffnen, Beschattung und Sonnenheizen setzen
trotzdem sofort aus, solange der Wind über dem Grenzwert liegt.

Die **Entwarnung** ist standardmäßig aus. Mit einer Zeit größer 0 stellt die
Automation den Normalzustand her, sobald der Wind nach einem Sturm so lange
ununterbrochen unter dem Grenzwert lag:

- **Nachtmodus aktiv:** Der Nachtzustand wird nachgeholt — wie beim Ende einer Pause
  je nach Fenster geschlossen, Kipp- oder Lüftungsposition.
- **Tagsüber im Panzer-Modus:** Ein noch geschlossener Rollladen öffnet auf die
  Zielposition für morgens (nur aufwärts; bei offenem Fenster nur mit "Aktion bei
  Sturm erzwingen", wie beim Sturmschutz selbst). Hat ihn inzwischen die Beschattung
  übernommen oder jemand von Hand bewegt, bleibt er, wo er ist.
- **Tagsüber ohne Panzer-Modus:** Der Rollladen ist schon oben, es wird nichts gefahren.

Den Beschattungs-Status hat schon der Sturmschutz freigegeben, der nächste Tick
beschattet also bei Bedarf. "Tagsüber" heißt: Nachtmodus aus, Sonne über dem Horizont
und die Uhrzeit fürs Morgens-Öffnen (falls eingerichtet) vorbei. Sonst bleibt der
Rollladen in der Sturmposition, bis das nächste reguläre Ereignis ihn übernimmt.
Entwarnt wird nur nach einem echten Unterschreiten des Grenzwerts — meldet die Quelle
nach einem Neustart oder Aussetzer erstmals einen niedrigen Wert, ist das keine
Entwarnung. Ist ein Wind-Sensor gesetzt, entscheidet nur er über die Entwarnung.

### Nach einem Neustart von Home Assistant

Zeitpunkt-Ereignisse gibt es nur, solange Home Assistant läuft: Fällt die
Morgens-Uhrzeit in einen Neustart (typisch: ein Update am frühen Morgen), bleibt der
Rollladen an diesem Tag zu. Ebenso kann ein kurz vor dem Herunterfahren
eingeschalteter Nachtmodus seinen Fahrbefehl verlieren. Mit dem Schalter **"Nach
Neustart nachholen"** (Abschnitt "Nach HA-Neustart", standardmäßig aus) bewertet die
Automation den Zustand 60 Sekunden nach dem Start neu — dann sind Integrationen,
Rollladen und Sensoren in der Regel wieder verfügbar:

1. **Pause aktiv oder Sturm?** Dann passiert nichts.
2. **Morgens-Uhrzeit höchstens 2 Stunden her?** Dann fährt der Rollladen wie beim
   Morgens-Öffnen auf die Zielposition, sofern er geschlossener ist. Ein noch
   eingeschalteter Nachtmodus spielt dabei keine Rolle — der reguläre Morgens-Trigger
   öffnet ebenfalls unabhängig davon, und ein Neustart kurz nach dem Öffnen soll den
   Rollladen nicht wieder schließen. Läuft gerade eine Beschattung oder hat
   Sonnenheizen geöffnet (Status-Helfer an), bleibt der Rollladen, wie er ist.
3. **Nachtmodus an und es ist Nacht?** "Nacht" heißt hier: Die Sonne steht unter dem
   Horizont, und es ist Abend oder die Morgens-Uhrzeit steht noch bevor (ohne
   Morgens-Öffnen gilt die Nacht bis Sonnenaufgang). Dann wird der Nachtzustand
   hergestellt wie beim Einschalten des Nachtmodus — bei geschlossenem Fenster zu, bei
   gekipptem oder offenem Fenster die Lüftungsposition.
   Anders als beim Einschalten fährt der Rollladen bei gekipptem oder offenem Fenster
   dabei nur hoch, nie herunter: Der Neustart kommt zu einem beliebigen Zeitpunkt, und
   wer gerade auf dem Balkon steht, soll nicht ausgesperrt werden. Ohne verfügbaren
   Fensterkontakt wird nichts bewegt.

Tagsüber wird ein noch eingeschalteter Nachtmodus bewusst **nicht** wiederhergestellt:
Der Morgens-Trigger hat den Rollladen dann meist schon geöffnet, obwohl der Helfer
noch an ist — ein Neustart soll ihn nicht mitten am Tag schließen.

**Warum nur 2 Stunden?** Das Nachholen soll einen Neustart _über_ die Morgens-Uhrzeit
hinweg abfangen. Je später der Neustart, desto wahrscheinlicher ist ein geschlossener
Rollladen Absicht — Mittagsschlaf, Kinderzimmer, Hitze. Ein Öffnen am Nachmittag wäre
dann kein Nachholen, sondern ein Fehler.

Beschattung und Sonnenheizen brauchen kein Nachholen: Ihre 5-Minuten-Durchläufe
übernehmen nach dem Start von selbst.

### Diagnose: Warum steht der Rollladen so?

Bei so vielen Features ist das die häufigste Frage. Die Antwort liefert ein
optionaler Text-Helfer im Abschnitt "Diagnose": Nach jeder Fahrt, die die
Automation auslöst, steht darin die letzte Aktion mit Uhrzeit und Grund — nach dem
Muster `Uhrzeit Aktion → Ziel (Grund)`, zum Beispiel:

- `07:00 Morgens → 100 %` (bei Frost `Morgens → 90 % (Frost-Grenze)`)
- `21:30 Nachtmodus → zu` bzw. `21:30 Nachtmodus → 15 % (Fenster offen)`
- `13:05 Beschattung beginnt → 35 %`, `13:30 Beschattung → 32 %` und später
  `16:40 Beschattungs-Ende → 100 %` (mit Blendschutz `Blendschutz → 50 %`)
- `09:12 Fenster gekippt → 20 %`, nach dem Schließen `09:40 Zurückfahren → Ausgangsposition`
- `14:02 Sturm → auf` (im Panzer-Modus `Sturm → zu (Panzer-Modus)`), danach
  `Sturm vorbei → 100 %`
- `Regen → 10 % (Fenster gekippt)` und `Regen vorbei → Position von vorher`
- `Abwesenheit → 0 %`, `Heimkommen → 100 %`, `Notfall → auf`, `Hindernis frei → zu
(Nachtmodus)`, `Nach Neustart → 100 % (Morgens nachgeholt)`
- `Sonnenheizen → 100 %`, `Schließen erzwingen → zu (nach 30 Min.)`,
  `Knopf "Rollladen schließen" → zu`, `Pause beendet → zu (Nachtmodus)`,
  `Moskito: 2 Lichter aus`

Zusätzlich werden die Fälle vermerkt, in denen eine erwartete Fahrt bewusst
ausgelassen oder ersetzt wurde — sie geben sonst am meisten Rätsel auf:

- `07:00 Morgens öffnen übersprungen (Sturm)` bzw. `(Regenschutz)`
- `14:02 Sturmschutz übersprungen (Fenster offen)` — das Fenster war offen und
  "Aktion bei Sturm erzwingen" ist aus.
- `09:40 Zurückfahren → zu (Nachtmodus)` — während des Lüftens kam der Nachtmodus,
  statt der Ausgangsposition wird geschlossen.
- `09:40 Zurückfahren übersprungen (Sturm)` bzw. `(Pause)`, `(Hindernis)` oder
  `(Ausgangsposition unbekannt)` — Letzteres, wenn die beim Öffnen gemerkte
  Position fehlt (gemerkte Positionen überleben keinen Neustart von Home Assistant).

Meldet der Fenstersensor beim Nachtmodus oder Sturm gerade keinen gültigen Zustand
(z. B. `unavailable`), steht als Grund `(Fensterstatus unbekannt)`.

**Einrichten:** Unter _Einstellungen → Geräte & Dienste → Helfer_ einen Helfer vom
Typ "Text" anlegen — einen **pro Fenster**, sonst überschreiben sich die Instanzen
gegenseitig. Die maximale Länge am besten auf 255 setzen (Standard: 100 Zeichen) —
die Texte sind zwar meist deutlich kürzer, zu lange würden aber abgeschnitten.
Den Helfer dann in der Instanz unter "Diagnose" auswählen und z. B. als
Entitäts-Karte neben den Rollladen ins Dashboard legen. Den Verlauf der letzten
Aktionen zeigt schon der Verlauf des Helfers selbst.

**Logbuch:** Mit "Zusätzlich ins Logbuch schreiben" erscheint jede Aktion außerdem
als Eintrag im Logbuch des Rollladens (Name = Name des Rollladens) — so steht der
Grund direkt neben den Zustandswechseln. Das funktioniert auch ohne Text-Helfer,
setzt aber die Logbuch-Integration voraus (bei Standard-Installationen über
`default_config` immer vorhanden). Die Diagnose ist rein informativ: Ein fehlender
oder falsch konfigurierter Text-Helfer bringt die Steuerung nie aus dem Tritt.

Technisch meldet jede Fahrt ihren Grund per Event an die eigene Instanz; geschrieben
wird in einem eigenen, kurzen Lauf der Automation. Diese Läufe tauchen deshalb
zusätzlich in den Traces auf. Ohne Text-Helfer und ohne Logbuch-Option entfallen sie
ganz.

## Bekannte Grenzen

- **Wind-Sensor kurz nicht verfügbar** zählt als "windstill". Bewusste Entscheidung:
  Ein dauerhaft toter Sensor soll nicht sämtliche Komfort-Funktionen lahmlegen. Der
  Sturmschutz selbst hat beim Überschreiten des Grenzwerts längst ausgelöst.
- **Wetterlagen-Filter:** Flattert das Wetter zwischen zwei _nicht_ erlaubten Lagen
  (z. B. Regen ↔ Starkregen), beendet erst Sonnenstand oder Temperatur die
  Beschattung (mit Blendschutz nur der Sonnenstand). Der Filter beendet nur bei
  mindestens 10 Minuten stabil schlechter Lage.
  Mit Ausschaltverzögerung entfällt diese Einschränkung: Dann beendet jede nicht
  erlaubte Lage, die für die eingestellte Zeit ununterbrochen anliegt — auch im
  Wechsel zwischen zwei nicht erlaubten Lagen.
- **Verzögerungen nach einem Neustart:** Startet Home Assistant neu (oder wird die
  Automation neu geladen bzw. wieder eingeschaltet), beginnen laufende Ein- und
  Ausschaltverzögerungen von vorn; nach einem Neustart ist auch eine schon
  abgelaufene, aber noch nicht genutzte Verzögerung vergessen. Sind die Bedingungen
  dabei schon erfüllt, entscheidet stattdessen der 5-Minuten-Takt in den ersten zehn
  Minuten nach Ablauf der Verzögerung (gezählt ab dem Neustart) anhand des Zustands in
  diesem Moment — eine kurze Unterbrechung in der Zwischenzeit fällt dann nicht auf.
- **Cover ohne Positions-Angabe** (nur auf/zu): Morgens-Öffnen funktioniert, der
  Nachtmodus schließt ganz (statt auf die Nachtposition), Kipp-Position, Beschattung
  und Regenschutz werden übersprungen — sie brauchen Positionsdaten. Beim Öffnen des Fensters fährt
  der Rollladen immer ganz auf; weder "Position bei geöffnetem Fenster" noch
  Frostschutz können solche Cover begrenzen. Nach einem Neustart wird das Morgens-Öffnen
  bei ihnen nur nachgeholt, wenn sie als geschlossen gemeldet werden (nicht bei
  "unbekannt").
- **Frostschutz:** Kipp- und Nachtlüftungs-Position werden nicht begrenzt (sie liegen
  normalerweise weit unter der Frost-Position), ebenso wenig das Zurückfahren nach dem
  Lüften (es stellt nur die Position von vorher wieder her). Auch die Nachführung der
  Beschattung bleibt unbegrenzt — sie startet erst ab der Beschattungs-Schwelle
  (mindestens 10 °C). Nur der Blendschutz kann auch bei Frost laufen; sein Ziel wird
  dann auf die Frost-Position begrenzt. Ohne verfügbare Temperaturquelle greift der Frostschutz nicht.
- **Windgeschwindigkeit** wird roh mit dem Grenzwert verglichen — liefert deine Quelle
  m/s statt km/h, muss der Grenzwert entsprechend gesetzt werden.
- **Auslöseverzögerung:** Ein Neustart oder Neuladen der Automationen während der
  Wartezeit verwirft sie; der Sturmschutz löst dann erst beim nächsten Überschreiten
  des Grenzwerts aus.
- **Entwarnung ohne Sturmfahrt:** Die Entwarnung weiß nicht, ob der Sturmschutz den
  Rollladen wirklich bewegt hat — sie folgt jedem Wert über dem Grenzwert, auch einer
  einzelnen Böe, die wegen der Auslöseverzögerung gar keinen Sturmschutz ausgelöst hat.
  Nachts stellt sie dann den Nachtzustand her, im Panzer-Modus öffnet sie tagsüber
  jeden noch geschlossenen Rollladen, auch einen, der schon vor dem Sturm zu war.
- **Entwarnung mit zwei Windquellen:** Sind Wind-Sensor und Wetter-Entität gesetzt,
  kann der Sturmschutz über beide auslösen, die Entwarnung kommt aber nur vom
  Wind-Sensor. Hat allein die Wetter-Entität den Sturm gemeldet, gibt es keine
  Entwarnung.
- **Sturm-Ende:** Ohne Entwarnung (Standard) bleibt der Rollladen nach dem Sturm in der
  Schutzposition. Die Entwarnung entfällt außerdem, wenn währenddessen eine Pause läuft,
  Home Assistant neu startet bzw. die Automationen neu geladen werden oder die
  Windquelle zwischendurch kurz nicht verfügbar ist (dann ist kein echtes Unterschreiten
  belegt). In all diesen Fällen bleibt der Rollladen in der Schutzposition, bis das
  nächste reguläre Ereignis (Nachtmodus, Morgens, Beschattung) ihn übernimmt.
- **"Nur anwesend" ohne Erinnerungen holt nichts nach:** Kommt jemand erst nach
  Ablauf des Timeouts nach Hause, während das Fenster noch offen ist, gibt es keine
  nachträgliche Meldung — der Trigger feuert nur einmal. Mit Erinnerungen bekommt,
  wer inzwischen heimgekommen ist, die nächste Erinnerung.
- **Dauer für manuelle Eingriffe:** Gemessen wird ab der letzten Änderung von Zustand
  oder Attributen des Rollladens (`last_updated`). `last_changed` wäre ungeeignet: Bei
  vielen Covern ändert eine Teilfahrt nur das Attribut `current_position`, der
  Zustand bleibt "offen". Meldet ein Cover laufend weitere veränderliche Attribute
  (z. B. Funkqualität oder "zuletzt gesehen" bei manchen MQTT-/Zigbee-Integrationen)
  oder schwankt die Positionsmeldung ständig um ein Prozent, läuft die Wartezeit nie
  ab — der Eingriff gilt dann wie bisher bis zum Beschattungs-Ende. Ein HA-Neustart
  oder ein kurzes "nicht verfügbar" startet die Wartezeit ebenfalls neu.
- **Nachtposition ist keine Untergrenze:** Sturmschutz im Panzer-Modus, "Schließen
  erzwingen" und der Benachrichtigungs-Knopf schließen weiterhin ganz. Bei gekipptem
  oder offenem Fenster fährt der Nachtmodus auf die Kipp-Position bzw. die
  Nachtposition bei offenem Fenster, auch wenn diese tiefer liegen — sie sollten daher
  nicht unter der Nachtposition bei geschlossenem Fenster liegen. Darf ein Rollladen nie
  ganz zufahren (z. B. wegen eines Klimaschlauchs im Fenster), diese Funktionen
  entsprechend einstellen bzw. nicht nutzen.
- **Nachtmodus-Helfer morgens noch an:** Solange der Helfer an ist, gilt Nacht — wird
  ein Fenster geschlossen, fährt der Rollladen zu, auch wenn Morgens öffnen ihn schon
  geöffnet hat. Den Helfer daher spätestens zur Öffnungszeit ausschalten.
- **Notfall-Öffnen ist kein Sicherheitssystem:** Ist der Rollladen selbst stromlos
  (Sicherung ausgelöst) oder sein Funknetz ausgefallen, kann die Automation ihn
  nicht öffnen. Geöffnet werden nur Rollläden, die eine Instanz dieses Blueprints
  haben — Markisen oder andere Rollläden brauchen eine eigene Automation.
- **Notfall-Sensor fällt im Alarm aus:** `unavailable`/`unknown` zählt nie als
  Alarm. Wird ein Sensor während des Alarms nicht verfügbar (z. B. ein Rauchmelder
  im Brand), läuft die Automatik wieder normal: Der Nachtmodus wird nicht
  nachgeholt (auch nicht, wenn die übrigen Sensoren sauber `off` melden), aber
  spätere Ereignisse (z. B. Beschattung) bewegen den Rollladen wieder. Wer das
  ausschließen will, nimmt einen `input_boolean` "Feueralarm", den eine eigene
  Automation beim Alarm einschaltet und der erst von Hand zurückgesetzt wird.
  Endet der Alarm während eines Neustarts, wird der Nachtmodus ebenfalls nicht
  nachgeholt.
- **Hindernis-Sperre gilt nur für diese Automation:** Wandtaster, Szenen und andere
  Automationen fahren den Rollladen weiterhin herunter. Fahrten nach oben laufen auch
  während der Sperre. Die Sperre verhindert nur den _Start_ einer Abwärtsfahrt — wird
  die Fliegengittertür während einer laufenden Fahrt geöffnet, stoppt sie nicht.
- **Nach der Sperre** werden nur Nachtmodus und Panzer-Schließen bei Sturm nachgeholt.
  Ausgelassene Einzelfahrten (Zurückfahren nach dem Lüften, "Schließen erzwingen",
  Knopf-Druck) entfallen — tagsüber bleibt der Rollladen dann oben, bis das nächste
  Ereignis ihn übernimmt. Umgekehrt stellt jede Freigabe einen eingeschalteten
  Nachtmodus her, auch wenn der Rollladen zwischendurch von Hand geöffnet wurde (wie
  beim Pause-Ende). Außerdem setzt jede Freigabe den Beschattungs-Status zurück: Ein
  manueller Eingriff während der Beschattung ist danach vergessen.
- **Dauerhaft toter Sperr-Sensor** verhindert dauerhaft jedes automatische
  Herunterfahren (siehe oben).
- **Neustart während der Wartezeit:** Ein Neustart von Home Assistant oder ein
  Neuladen der Automation kann das Nachholen verhindern — laufende Wartezeiten gehen
  verloren, und weil die Zeitstempel der Sensoren beim Start neu gesetzt werden, gilt
  die Sperre danach noch einmal für die volle Wartezeit. Ein in diesem Fenster
  eingeschalteter Nachtmodus wird dann erst beim nächsten regulären Ereignis
  hergestellt.
- **Mindestabstand und `last_updated`:** Für den Mindestabstand zählt jede
  Aktualisierung des Rollladens als Bewegung — auch ein kurzes `unavailable`
  (Funk-Aussetzer, Reconnect), ein Neustart von Home Assistant oder Attribute, die
  sich ohne Fahrt ändern (manche Integrationen hängen z. B. Signalstärke oder "zuletzt
  gesehen" an). Ändern sich solche Attribute ständig, bleibt `last_updated` dauerhaft
  jung, und Nachführen bzw. Sonnenheizen-Öffnen kommen nie zum Zug — dann den
  Mindestabstand auf 0 lassen. Prüfen lässt sich das unter _Entwicklerwerkzeuge →
  Template_ mit `{{ states['cover.dein_rollladen'].last_updated }}`.
- **Mindestabstand und Eingriffs-Erkennung:** Ein großer Abstand bei kleiner
  _Toleranz für manuelle Eingriffe_ lässt den nachgeholten Schritt wie eine
  Handbetätigung aussehen — das Nachführen ruht dann bis zum Beschattungs-Ende
  (Richtwerte siehe "Mindestabstand zwischen Nachführ-Fahrten").
- **Reaktionszeit und Benachrichtigungen:** Die Reaktionszeit verzögert nur das
  Abräumen der Meldung nach dem Schließen. Wann eine "Fenster zu lange
  offen/gekippt"-Meldung verschickt wird, bestimmen weiterhin allein deren eigene
  Wartezeiten (in Minuten ab dem Öffnen).
- **Kurz zu, gleich wieder auf:** Wird das Fenster kürzer als die Reaktionszeit
  geschlossen und dann wieder geöffnet, wertet die Automation das als neues
  Öffnen und merkt sich die gerade aktuelle (schon geöffnete) Position als
  Ausgangsposition. Nach dem endgültigen Schließen bleibt der Rollladen dann
  oben, statt zurückzufahren. Je größer die Reaktionszeit, desto eher passiert das.
- **Neustart während der Zufallsverzögerung:** Wird Home Assistant neu gestartet oder
  die Automation neu geladen bzw. gespeichert, während die Wartezeit läuft, entfällt
  die Fahrt — sie wird nicht nachgeholt: Morgens bleibt der Rollladen zu, nachts offen,
  bis das nächste Ereignis ihn übernimmt. Nachts bleibt dann auch der Status-Helfer
  einer abends noch laufenden Beschattung gesetzt; nach dem Ausschalten des
  Nachtmodus behandelt der nächste Tick sie regulär weiter (meist: Ende mit Fahrt auf
  die Position nach der Beschattung), ggf. schon vor der Morgens-Uhrzeit.
- **Mehrere Instanzen würfeln unabhängig:** Jede Instanz zieht ihre eigene Wartezeit.
  Rollläden, die gemeinsam fahren sollen (z. B. eine Fensterfront), lassen sich nicht
  synchron verzögern — dort den Regler auf 0 lassen.
- **Sanftes Wecken** übersteht keinen Neustart und kein Neuladen der Automation — der
  Rollladen bleibt auf dem erreichten Zwischenstand (Details unter
  [Sanftes Wecken](#sanftes-wecken)).
- **Neustart während des Lüftens:** Das Zurückfahren nach dem Schließen des Fensters
  wird nicht nachgeholt — die gemerkte Ausgangsposition und die Wartezeit gehen beim
  Neustart verloren. Der Rollladen bleibt in der Lüftungsposition, bis das nächste
  reguläre Ereignis ihn übernimmt.
- **Nachholen nach Neustart** kann nicht unterscheiden, ob ein Ereignis verpasst,
  bewusst ausgelassen oder danach von Hand rückgängig gemacht wurde: Innerhalb der
  2 Stunden nach der Morgens-Uhrzeit öffnet ein Neustart auch einen Rollladen, den
  jemand nach dem Morgens-Öffnen wieder geschlossen hat oder dessen Morgens-Öffnen in
  eine Pause fiel. Bei aktivem Nachtmodus fährt ein nachts von Hand geöffneter
  Rollladen bei geschlossenem Fenster wieder zu.
- **Veralteter Fensterkontakt nach Neustart:** Manche Integrationen stellen nach dem
  Start den letzten bekannten Zustand wieder her, und batteriebetriebene Kontakte
  melden sich erst bei der nächsten Änderung. Wurde die Balkontür während des
  Neustarts geöffnet und meldet der Kontakt noch "zu", schließt das Nachholen bei
  aktivem Nachtmodus den Rollladen — ⚠️ Aussperr-Gefahr.
- **Grenzen des Nachholens:** Wird der Nachtmodus-Helfer selbst zeitgesteuert
  geschaltet und fiel dieser Zeitpunkt in den Neustart, ist der Helfer noch aus — dann
  gibt es nichts nachzuholen. Steht die Sonne über dem Horizont oder ist die
  Morgens-Uhrzeit (nach Mitternacht) schon vorbei, wird ein aktiver Nachtmodus nicht
  wiederhergestellt — ohne die Integration "Sonne" (`sun.sun`) also nie. Ist der
  Rollladen nach 60 Sekunden noch nicht verfügbar, unterbleibt das Nachholen; ohne
  verfügbaren Fensterkontakt wird der Nachtzustand nicht hergestellt. Einen zweiten
  Versuch gibt es nicht. Ein bloßes Neuladen der Automationen zählt nicht als Neustart.
- **Abwesenheit — HA-Neustart während der Wartezeit:** Die laufende Wartezeit geht
  verloren. Sind nach dem Neustart weiterhin alle weg, kann das Schließen für diesen
  Weggang entfallen — der Rollladen bleibt dann, wie er ist.
- **Abwesenheit — HA-Neustart während der Abwesenheit:** Der Merker (temporäre
  Szene) ist danach weg — vermutlich auch nach einem Neuladen der Szenen, wie es
  beim Speichern einer Szene im Editor passiert. Beim Heimkommen wird dann nicht
  wieder geöffnet, und die strenge Beschattung fällt bis zur nächsten Abwesenheit
  auf die normale Tiefe zurück. Dass Sonnenschutz und Sonnenheizen einen unten
  stehenden Rollladen nicht öffnen, solange niemand da ist, gilt weiterhin.
- **Abwesenheit — einmaliges Schließen:** Wird der Rollladen während der Abwesenheit
  wieder geöffnet (von Hand, per App oder durch das Morgens-Öffnen), schließt die
  Abwesenheit ihn nicht erneut.
- **Abwesenheit und Pause:** Beginnt die Abwesenheit während einer Pause, wird das
  Schließen nicht nachgeholt, und die strenge Beschattung entfällt für diese
  Abwesenheit. Kommt jemand während einer Pause heim, wird nicht wieder geöffnet —
  auch nicht nach dem Ende der Pause.
- **Anwesenheits-Entität nicht verfügbar oder unbekannt:** zählt als anwesend — eine
  Person ohne Tracker (dauerhaft "unbekannt") verhindert die Abwesenheit also
  komplett. Nur Personen und Geräte auswählen, die ihren Standort wirklich melden.
- **Regenschutz und Neustart:** Die Position von vor dem Regen wird in einer
  dynamischen Szene gemerkt und überlebt keinen Home-Assistant-Neustart. Nach einem
  Neustart während des Regens bleibt der Rollladen nach dem Regen in der
  Regenposition.
- **Fenster während des Regens geschlossen:** Die gemerkte Position wird verworfen.
  Liegt das Kippen schon länger zurück als das Zeitfenster fürs Zurückfahren, bleibt
  der Rollladen in der Regenposition, bis das nächste reguläre Ereignis ihn
  übernimmt. Dasselbe gilt, wenn der Regen nachts oder während Sturm/Pause endet.
- **Morgens-Öffnen bei Regen** wird nur nachgeholt, wenn "Nach dem Regen
  zurückfahren" aktiv ist; sonst entfällt es, solange der Regenschutz greift.
- **Regensensor-Typen:** Ausgewertet wird nur an/aus eines `binary_sensor` oder
  `input_boolean`. Regenmengen-Sensoren (mm/h) oder die Wetter-Entität (Zustand
  `rainy`, `pouring` usw.) über einen Template-Binärsensor einbinden (siehe oben).
  Wetterdienst-Zustände sind oft verzögert und nicht ortsgenau — ein echter
  Regensensor am Haus reagiert deutlich zuverlässiger. Windrichtung und
  Schlagregen werden nicht berücksichtigt.
- **Zusatz-Dienste werden nicht abgeräumt:** Beim Schließen des Fensters verschwindet
  nur die Meldung auf den Companion-App-Geräten. Was an Zusatz-Dienste ging
  (Telegram-Nachricht, dauerhafte Benachrichtigung …), bleibt stehen, und jede
  Erinnerung kommt dort als eigene Nachricht an.
- **Erinnerungen überleben keinen Neustart:** Ein Neustart von Home Assistant (ebenso
  das Speichern dieser Automation) beendet laufende Erinnerungen; sie werden danach
  nicht fortgesetzt.
- **Blendschutz kennt keine Bewölkung:** Ausgelöst wird rein über den Sonnenstand.
  Ohne Wetterlagen-Filter fährt der Rollladen auch an trüben Tagen auf die
  Blendschutz-Position; ein Helligkeitssensor wird nicht ausgewertet.
- **Blendschutz-Zielwechsel** (siehe "Manuelle Eingriffe"): Stellst du den Rollladen
  kurz nach dem Überschreiten eines Schaltpunkts von Hand genau auf das andere Ziel
  (z. B. bei 22,5 °C ganz zu, während der Hitzeschutz ohnehin 0 % wollte), sieht das
  wie ein Zielwechsel aus und wird nachgeführt. Umgekehrt: Springt die Temperatur
  zwischen zwei Ticks weit über einen Schaltpunkt (z. B. stündlich aktualisierte
  Wetter-Entität) oder steht der Rollladen nicht genau auf dem bisherigen Ziel, wirkt
  der Wechsel wie ein Handeingriff — der Rollladen bleibt dann bis zum Episoden-Ende
  auf dem bisherigen Ziel. Perfekte Erkennung bräuchte einen zweiten Helfer.
- **Blendschutz am Westfenster:** Die Episode endet mit dem Sonnenuntergang; der
  Rollladen fährt dann auf die "Position nach der Beschattung" — wer abends ohnehin
  schließt, sieht diese Fahrt kurz vor dem Nachtmodus.
- **Blendschutz und Sonnenheizen** verfolgen am selben Fenster gegensätzliche Ziele.
  Sonnenheizen öffnet nur unterhalb seines Grenzwerts "heruntergelassen" (Standard
  50 %) — die Blendschutz-Position also mindestens so hoch wählen, sonst öffnet
  Sonnenheizen den Blendschutz, und der wertet das als Handeingriff.
- **Sonnenstand — verpasste Momente:** Läuft Home Assistant genau dann nicht (Neustart,
  Update), wenn die Sonne die Schwelle überschreitet, wird das Öffnen bzw. Schließen
  nicht nachgeholt — wie bei einer verpassten Uhrzeit. Der Nacht-_Zustand_ stimmt
  trotzdem (keine Beschattung, Lüftungsposition), nur die Fahrt fehlt.
- **Sonnenstand — Ende der Nacht:** Wie bei der Nachtmodus-Uhrzeit endet die Nacht
  zur Uhrzeit für morgendliches Hochfahren (bei "Morgens nach Sonnenstand öffnen" also
  zu "frühestens"), ohne Helfer bei Sonnenaufgang — oder schon beim Öffnen nach
  Sonnenstand, falls das früher kommt (negative Sonnenhöhe oder "spätestens" vor
  Sonnenaufgang). Öffnet der Rollladen erst nach diesem Nacht-Ende, gilt bis dahin
  schon Tag: Ein jetzt geöffnetes Fenster fährt ihn ganz hoch statt nur auf die
  Lüftungsposition, und Sonnenheizen darf ihn bereits öffnen.
- **Helle Nächte ohne "frühestens":** Bleibt die Sonne nachts über der Morgen-Schwelle
  (Hochsommer in Skandinavien), gibt es morgens keinen Übergang. Ohne
  frühestens-Helfer öffnet der Rollladen dann an diesem Tag nicht automatisch — mit
  Helfer öffnet er zu dieser Uhrzeit.
- **Sonnenstand und Nachholen nach einem Neustart:** Wann die Sonne morgens die
  Schwelle überschritten hat, weiß die Automation nach einem Neustart nicht. Ist das
  Öffnen fällig (Sonne hoch genug oder "spätestens" erreicht, "frühestens" vorbei),
  holt "Nach Neustart" es bis 2 Stunden nach "spätestens" nach.

- **Diagnose zeigt nur, was diese Automation tut:** Fahrten per Wandtaster, App oder
  anderer Automation erscheinen nicht — der Text bleibt dann auf der letzten
  Automatik-Aktion stehen. Auch nicht jede ausgelassene Fahrt wird vermerkt, sondern
  nur die oben genannten. Unverändert bleibt der Text z. B., wenn der Rollladen schon
  passend steht, wenn die Beschattung nach einem manuellen Eingriff ruht (sonst würde
  er alle 5 Minuten überschrieben) und bei Ereignissen während einer Pause —
  Ausnahmen sind, was auch in der Pause wirkt, z. B. Sturmschutz, Notfall-Öffnen,
  der "Rollladen schließen"-Knopf und das ausgelassene Zurückfahren nach dem Lüften. Das Nachführen
  der Beschattung schreibt dagegen bei jeder Bewegung — mit Logbuch-Option also
  entsprechend viele Einträge.
- **Moskito-Modus und Fensteröffnen gleichzeitig:** Öffnest du das Fenster nach
  Sonnenuntergang, laufen die Rollladen-Fahrt und das Ausschalten der Lichter
  parallel. Im Text-Helfer steht danach der Eintrag, der zuletzt fertig war — das
  Logbuch zeigt beide.

## FAQ

**Warum steht der Rollladen so?** Am schnellsten beantwortet das der optionale
Status-Text-Helfer im Abschnitt "Diagnose" (siehe
[oben](#diagnose-warum-steht-der-rollladen-so)): Er zeigt die letzte Aktion der
Automation samt Uhrzeit und Grund, auch wenn sie eine Fahrt bewusst ausgelassen
hat. Passt die aktuelle Position nicht zu diesem Text, wurde der Rollladen danach
von Hand oder von etwas anderem bewegt (Ausnahme: ein Moskito-Eintrag, siehe
Bekannte Grenzen).

**Kann der Rollladen mit Sonnenauf- und -untergang fahren statt zu einer festen
Uhrzeit?** Ja — "Morgens nach Sonnenstand öffnen" im Abschnitt "Morgens öffnen" und
"Nachtmodus nach Sonnenstand" im Abschnitt "Nachtmodus" einschalten. Statt eines
Versatzes in Minuten stellst du die Sonnenhöhe ein (0° ≈ Sonnenauf-/-untergang,
negative Werte = in der Dämmerung, siehe Tabelle oben), dazu "frühestens" und
"spätestens", damit der Rollladen im Sommer nicht um 5 Uhr öffnet und im Winter nicht
schon um 16 Uhr schließt. Pro Tag wird genau einmal geöffnet und pro Abend genau
einmal geschlossen; die Uhrzeit-Helfer (morgens als "frühestens") kannst du weiter in
allen Instanzen gemeinsam nutzen. Eine eigene Uhrzeit fürs Wochenende gibt es bisher
nicht.

**Was bedeuten 0 % und 100 % bei den Positionen?** Das Blueprint folgt der
Home-Assistant-Konvention: 100 % = ganz offen, 0 % = ganz geschlossen. Die Prozente
sind dabei die Werte deines Cover-Aktors, also Motor-Laufweg — nicht zwingend
Glasfläche. Für die Beschattung lässt sich dieser Unterschied über die
Glas-Kalibrierung (siehe oben) ausgleichen; alle anderen Positions-Eingaben
(morgens, Kipp-Position, geöffnetes Fenster, Nachtpositionen, Frost-Position,
Blendschutz-Position) sind bewusst direkte
Aktor-Werte.

**Kann der Rollladen nachts auf einer Position statt ganz zu stehen?** Ja — im
Abschnitt "Nachtmodus" die "Nachtposition bei geschlossenem Fenster" auf den
gewünschten Wert stellen (0 % = ganz zu, wie bisher). Für einen Lüftungsschlitz reicht
eine kleine Position, z. B. derselbe Wert wie die "Nachtposition bei offenem Fenster".
Der Rollladen fährt dabei nur abwärts: Hast du ihn abends schon von Hand weiter
geschlossen, bleibt er dort.

**Die Beschattung tut nichts — warum?** Prüfe in dieser Reihenfolge: Gibt es eine
Benachrichtigung wegen fehlendem Status-Helfer? Ist eine Temperaturquelle gesetzt
(eigener Sensor oder Wetter-Entität im Sturmschutz-Abschnitt)? Liegt die
Außentemperatur über der Schwelle (oder ist der Blendschutz aktiviert), steht die
Sonne im Sichtfeld (Ausrichtung
korrekt?) und zwischen minimaler und maximaler Sonnenhöhe, liegt die Uhrzeit im
Zeitfenster ("frühestens ab"/"spätestens bis"), und ist das Fenster nicht komplett offen? Ist ein Freigabe-Helfer gesetzt, muss er eingeschaltet sein. Mit Einschaltverzögerung
müssen Sonnenstand, Temperatur und Wetterlage zusätzlich so lange ununterbrochen passen.

**Über dem Fenster ist ein Balkon oder Vordach — mittags wird trotzdem verdunkelt?**
Die maximale Sonnenhöhe im Abschnitt "Fenstergeometrie & Sonnenausrichtung" auf den
Wert setzen, ab dem das Vordach die Glasfläche abschattet (siehe "Sichtfeld und
Geometrie"). Darüber öffnet der Rollladen auf die Position nach der Beschattung;
am Nachmittag beschattet die Automation bei Bedarf wieder.

**Der Rollladen fährt morgens zur Beschattung herunter und weckt mich — oder abends,
obwohl ich die Abendsonne genießen will?** Dafür gibt es im Sonnenschutz-Abschnitt
das Zeitfenster: "Beschattung frühestens ab" (z. B. 09:00) verhindert den frühen Start,
"Beschattung spätestens bis" (z. B. 18:00) beendet eine laufende Beschattung zur
eingestellten Uhrzeit und fährt auf die "Position nach der Beschattung". Details
unter [Zeitfenster für die Beschattung](#zeitfenster-für-die-beschattung).

**Die Sonne blendet im Winter am Monitor (oder morgens im Bad), aber die Beschattung
tut nichts?** Der Sonnenschutz ist ein Hitzeschutz und greift erst über der
Temperatur-Schwelle. Für tief stehende Wintersonne im Sonnenschutz-Abschnitt den
**Blendschutz** aktivieren und die Blendschutz-Position wählen (z. B. 50 %): Dann
fährt der Rollladen, solange die Sonne ins Fenster scheint, auf diese Position — und
an heißen Tagen wie gewohnt auf die volle Beschattung. Am besten zusätzlich die
Wetterlagen für Beschattung auf "sonnig"/"teilweise bewölkt" beschränken, sonst
fährt er auch an trüben Tagen herunter.

**Warum fährt der Rollladen nach dem Lüften zurück?** Beim Öffnen des Fensters merkt
sich die Automation die Ausgangsposition und stellt sie nach dem Schließen wieder her
(innerhalb des einstellbaren Zeitfensters). Kam inzwischen Nachtmodus oder Sturm,
wird stattdessen deren Zustand hergestellt. Hat während des Lüftens eine
Automatik-Fahrt ein neues Ziel gesetzt — Morgens öffnen, Sonnenheizen oder das Ende
der Beschattung —, verwirft die Automation die gemerkte Position: Der Rollladen
bleibt nach dem Schließen, wo er ist. Das gilt auch, wenn Morgens öffnen gar nicht
fahren musste, weil der Rollladen schon oben war. Bei aktivem Nachtmodus fährt der
Rollladen beim Schließen des Fensters zu — auch wenn das Fenster länger offen war als
das Zeitfenster (nicht während einer Pause). Beginnt die Beschattung erst während des
Lüftens, fährt der Rollladen beim Schließen zunächst zurück und wird beim nächsten
Takt (spätestens nach 5 Minuten) neu beschattet. Eine während des Lüftens von Hand
gewählte Position (Taster, App) wird beim Zurückfahren dagegen überschrieben. Meldet ein Sperr-Sensor der Hindernis-Sperre noch ein
Hindernis, bleibt der Rollladen stehen.

**Der Rollladen fährt beim Nachführen ständig ein kleines Stück — geht das
seltener?** Ja, mit zwei Einstellungen im Sonnenschutz-Abschnitt: "Minimale
Positionsänderung" (größere, dafür seltenere Schritte) und "Mindestabstand zwischen
Nachführ-Fahrten" (Ruhezeit seit der letzten Bewegung, gilt auch fürs
Sonnenheizen-Öffnen). Beim Mindestabstand die Toleranz für manuelle Eingriffe im
Blick behalten — Details unter "Mindestabstand zwischen Nachführ-Fahrten".

**Kann ich denselben Status-Helfer für mehrere Fenster verwenden?** Nein — er
speichert den Zustand genau eines Fensters. Ein geteilter Helfer führt zu falschem
Öffnen/Schließen.

**Sonnenschutz und Sonnenheizen gleichzeitig aktiv — geht das?** Ja, das ist der
Normalfall. Die Temperatur-Schwellen trennen sie (Standard: beschatten über 25 °C,
heizen unter 12 °C); die Schwellen sollten sich nicht überlappen. Mit Blendschutz
beschattet der Sonnenschutz allerdings auch bei Kälte — dann die Blendschutz-Position
mindestens auf den Grenzwert "heruntergelassen" des Sonnenheizens setzen (siehe
Bekannte Grenzen).

**Home Assistant hat genau zur Morgens-Uhrzeit neu gestartet — warum blieb der
Rollladen zu?** Ein verpasster Zeitpunkt wird standardmäßig nicht nachgeholt. Mit dem
Schalter "Nach Neustart nachholen" (Abschnitt "Nach HA-Neustart") öffnet die
Automation nach dem Start nachträglich, sofern der Neustart höchstens 2 Stunden nach
der Morgens-Uhrzeit liegt — Details unter
[Nach einem Neustart von Home Assistant](#nach-einem-neustart-von-home-assistant).

**Die Fenster-offen-Meldung bleibt auf dem Handy stehen?** Sie verschwindet
automatisch, sobald das Fenster geschlossen wird — vorausgesetzt, die Companion-App
ist aktuell (das Aufräumen nutzt `clear_notification` mit Tags). Für Zusatz-Dienste
gilt das nicht, siehe Bekannte Grenzen.

**Kann ich zusätzlich per Alexa oder Telegram gewarnt werden?** Ja, über
"Zusätzliche Benachrichtigungs-Dienste" im Abschnitt Benachrichtigungen. Alexa
(Integration Alexa Media Player): `notify.alexa_media_<gerät>` eintragen, z. B.
`notify.alexa_media_kueche` — Alexa liest den Text vor, der Titel entfällt dabei.
Telegram: den Notify-Dienst deines Bots eintragen (z. B. `notify.telegram_familie`)
oder, falls deine Telegram-Integration eine Benachrichtigungs-Entität pro Chat
anlegt, deren Entitäts-ID. Den genauen Namen findest du unter _Entwicklerwerkzeuge →
Aktionen_ (nach "notify." suchen) bzw. bei den Entitäten. Beide bekommen die erste
Meldung und jede Erinnerung, aber keinen "Rollladen schließen"-Knopf.

**Mein Kontakt kennt nur offen/geschlossen, das Fenster wird aber eigentlich nur
gekippt?** Dafür gibt es in der Fenster-Interaktion den Schalter "Öffnen wie Kippen
behandeln": Jedes "offen" gilt dann als "gekippt" — der Rollladen fährt auf die
Kipp-Position statt komplett auf, und die Beschattung läuft weiter, statt zu
pausieren. Typischer Fall: das Badfenster mit einfachem binärem Kontakt.

**Ich habe ein festes Fenster ohne Kontakt — geht das?** Ja — das Feld
"Fenstersensor" einfach leer lassen; ein Dummy-Sensor ist nicht nötig. Das Fenster
gilt dann als immer geschlossen: Morgens öffnen, Nachtmodus, Sturmschutz,
Sonnenschutz und Sonnenheizen arbeiten ganz normal. Fenster-Interaktion
(Kippen/Öffnen/Zurückfahren), Benachrichtigungen und Moskito-Modus entfallen, weil
sie vom Fensterzustand leben. ⚠️ Bei Balkon- oder Terrassentüren ohne Kontakt fahren
Nachtmodus, Beschattung und Panzer-Modus auch bei offener Tür herunter —
Aussperr-Gefahr.
**Der Rollladen soll beim Öffnen nicht ganz hochfahren — z. B. als Durchgang für
die Katze?** Dafür gibt es in der Fenster-Interaktion die "Position bei geöffnetem
Fenster" (Standard 100 % = ganz auf). Stell sie z. B. auf 25 %: Wird das Fenster
komplett geöffnet, fährt der Rollladen nur bis dorthin — und nur nach oben; steht
er schon höher, bleibt er stehen. Nachts (Nachtmodus an) gilt stattdessen die
"Nachtposition bei offenem Fenster" im Abschnitt Nachtmodus — für die Katze dort
ebenfalls eine passende Höhe wählen. Ohne Nachtmodus-Helfer gilt die Position bei
geöffnetem Fenster rund um die Uhr. ⚠️ Bei Balkon- und Terrassentüren muss die
Position hoch genug zum Durchgehen sein.

**Ich öffne das Fenster nur kurz (Blumen gießen) — muss der Rollladen jedes Mal
hochfahren?** Nein: Die "Reaktionszeit" in der Fenster-Interaktion legt fest, wie
lange das Fenster unverändert offen bzw. gekippt sein muss, bevor der Rollladen
reagiert — und wie lange es nach dem Lüften geschlossen sein muss, bevor er
zurückfährt. Standard sind 2 Sekunden, das filtert nur prellende Kontakte. Mit
z. B. 30 Sekunden bleibt der Rollladen beim kurzen Öffnen einfach stehen. Die
Kehrseite: Bei Balkon- und Terrassentüren dauert es entsprechend länger, bis der
Rollladen hochfährt. Die Reaktionszeit gilt auch für den Moskito-Modus und das
Abräumen der Benachrichtigung, nicht aber für die Wartezeit, nach der eine
Benachrichtigung verschickt wird.

**Alle Rollläden fahren auf die Minute gleichzeitig — geht das unauffälliger?** Ja:
Mit dem Regler "Zufällige Verzögerung (max.)" bei "Morgens öffnen" und im Nachtmodus
fährt jeder Rollladen zufällig bis zu X Minuten später. Details und ein Rezept für
den Urlaub stehen oben unter "Zufallsversatz und Anwesenheitssimulation".

**Kann ich die Automation zeitweise anhalten?** Ja — im Abschnitt "Pausieren" einen
einen oder mehrere `input_boolean`-Helfer auswählen. Die Logik ist wählbar: "AN
pausiert" für Helfer wie "Aufnahme läuft", oder "AUS pausiert" für Aktiv-Schalter
fürs Dashboard ("Rollladensteuerung aktiv" — an heißt: die Automatik läuft). Bei
mehreren Helfern pausiert die Automatik nur, wenn alle gleichzeitig im
Pausier-Zustand stehen — bei "AUS pausiert" läuft sie also, sobald einer an ist. Während
der Pause macht die Automatik nichts:
keine Fahrten, keine Lichter, keine Meldungen. Ausnahmen: Notfall-Öffnen und
Sturmschutz greifen weiterhin, und der "Rollladen schließen"-Knopf einer Benachrichtigung
funktioniert wie der Wandtaster — bewusste Befehle werden nicht blockiert.
Praktisch für Videoaufnahmen (konstantes Licht!), schlafende Gäste oder den
Fensterputzer. Beim Ausschalten holt die Automation einen inzwischen aktiven
Nachtmodus nach, stellt bei Regen den Regenschutz her und bewertet die Beschattung
neu; verpasste Einzelereignisse (morgendliches Öffnen, Zurückfahren nach dem Lüften,
Sturm-Entwarnung, Schließen bei Abwesenheit, Wiederöffnen beim Heimkommen) werden
nicht nachgeholt.

**Kann ich die Beschattung tageweise freigeben oder sperren?** Ja — im Abschnitt
"Sonnenschutz" einen Freigabe-Helfer auswählen: ein `input_boolean`, einen
`binary_sensor` oder einen Zeitplan-Helfer (`schedule`). Beschattet wird dann nur,
solange er eingeschaltet ist; das Einschalten wirkt beim nächsten 5-Minuten-Takt.
Wird er ausgeschaltet, endet eine laufende Beschattung sofort regulär — der
Rollladen fährt auf die "Position nach der Beschattung", auch wenn er zwischendurch
manuell verstellt wurde. Ist der Helfer nicht verfügbar, startet keine neue
Beschattung, eine laufende wird deswegen aber auch nicht beendet, sondern bleibt
stehen (wie bei einer kurz ausgefallenen Temperaturquelle). Typische Steuerungen:
eine eigene Automation, die morgens anhand der Wetterprognose entscheidet, ob heute
beschattet wird; ein Dashboard-Schalter; ein Zeitplan, der die Beschattung z. B. nur
werktags erlaubt. Anders als der Pausier-Helfer betrifft die Freigabe nur die
Beschattung — Fenster-Interaktion, Nachtmodus, Sturmschutz, Sonnenheizen und
Benachrichtigungen laufen normal weiter. Und anders als der Status-Helfer darf ein
Freigabe-Helfer in mehreren Instanzen gemeinsam genutzt werden (z. B. einer pro
Fassade).

**Gibt es einen Urlaubsmodus?** Ja, über den Abschnitt "Abwesenheit": Personen bzw.
Tracker auswählen und "Bei Abwesenheit schließen" aktivieren. Beim Verlassen schließt
der Rollladen, Morgens-Öffnen und Nachtmodus laufen im Urlaub normal weiter (siehe
[Abwesenheit & Urlaub](#abwesenheit--urlaub)). Ein Helfer oder Binärsensor zählt als
anwesend, wenn er "an" ist — ein "Urlaub"-Schalter (an = weg) passt also nicht
direkt; dafür einen "Jemand zuhause"-Helfer verwenden oder den Schalter über einen
Template-Binärsensor umdrehen.

**Alle sind weg, aber der Rollladen schließt nicht — warum?** Prüfe in dieser
Reihenfolge: Meldet wirklich jede ausgewählte Entität "weg" (eine Person ohne
Tracker steht dauerhaft auf "unbekannt" und zählt als anwesend)? Ist die Wartezeit
abgelaufen? Ist das Fenster offen oder gekippt (Aussperr-Schutz)? Herrscht Sturm,
ist die Automation pausiert, oder steht der Rollladen schon auf bzw. unter der
Abwesenheits-Position?
