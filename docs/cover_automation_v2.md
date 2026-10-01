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
   - Nachtmodus: ein `input_boolean`, z. B. "Nacht-Modus". Einer für alle Instanzen;
     wie er geschaltet wird (Zeitplan, Guten-Nacht-Szene, von Hand), bleibt dir überlassen.
     Alternativ pro Fenster eine eigene Nacht-Uhrzeit über einen `input_datetime`-Helfer
     (nur Uhrzeit) — immer nur eines von beiden.
   - Sonnenschutz: ein `input_boolean` **pro Fenster** als Status-Speicher,
     Namensvorschlag: "Beschattung <Fenstername>".
   - Sonnenheizen: ein **weiterer** `input_boolean` pro Fenster (nicht denselben wie
     für den Sonnenschutz verwenden!).
4. Für Sonnenschutz/Sonnenheizen die **Fenstergeometrie** eintragen (Ausrichtung in
   Grad, Sichtfeld, Fensterhöhe, Brüstungshöhe; bei Vordach oder Balkon darüber die
   maximale Sonnenhöhe) — Details unten.

Fehlt ein zwingend nötiger Helfer bei aktiviertem Feature, meldet sich die Automation
selbst: Eine dauerhafte Benachrichtigung in Home Assistant benennt das betroffene
Fenster, bis der Helfer gesetzt oder das Feature deaktiviert ist.

## Die Features im Überblick

| Feature             | Was es tut                                                                                                                                                                | Voraussetzung                              |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------ |
| Morgens öffnen      | Fährt zur eingestellten Uhrzeit auf die Zielposition (nur wenn geschlossener)                                                                                             | `input_datetime`-Helfer (nur Uhrzeit)      |
| Fenster-Interaktion | Kippen → Lüftungsposition, Öffnen → ganz auf (optional: wie Kippen behandeln); nach dem Schließen zurück in die Ausgangsposition                                          | Fenstersensor (dann immer aktiv)           |
| Nachtmodus          | Schließt beim Einschalten des Helfers oder zur eigenen Uhrzeit ganz oder auf eine Nachtposition; offene/gekippte Fenster bekommen bis zum Schließen eine Lüftungsposition | `input_boolean` oder Uhrzeit-Helfer        |
| Sturmschutz         | Fährt bei Starkwind hoch (oder im Panzer-Modus herunter)                                                                                                                  | Wetter-Entität oder Wind-Sensor            |
| Sonnenschutz        | Beschattet nach Sonnenstand, sodass die Sonne höchstens X m in den Raum fällt; öffnet danach wieder; optional nur bei Freigabe                                            | Status-Helfer, Geometrie, Temperaturquelle |
| Sonnenheizen        | Öffnet im Winter vergessene Rollos, wenn Sonne ins Fenster scheint und es kalt ist                                                                                        | eigener Status-Helfer, Geometrie           |
| Frostschutz         | Öffnet bei Frost nur bis zu einer Maximalposition (z. B. 90 %), damit ein festgefrorener Panzer nicht reißt                                                               | Temperaturquelle wie beim Sonnenschutz     |
| Moskito-Modus       | Schaltet beim Fensteröffnen nach Sonnenuntergang die Lichter im Raum aus (mit Ausnahmen)                                                                                  | Fenstersensor (liefert auch den Bereich)   |
| Benachrichtigungen  | Meldet zu lange offene/gekippte Fenster aufs Handy, mit "Rollladen schließen"-Button; verschwindet automatisch beim Schließen                                             | Fenstersensor, Companion-App-Geräte        |
| Pausieren           | Hält die komplette Automation an, solange ein Helfer eingeschaltet ist — z.B. während Videoaufnahmen oder wenn Gäste schlafen                                             | `input_boolean`-Helfer (optional)          |
| Notfall-Öffnen      | Fährt bei Rauch-/CO-Alarm oder Hagelwarnung sofort ganz auf und hält den Rollladen oben, bis alle Sensoren wieder aus sind                                                | `binary_sensor`/`input_boolean` (optional) |
| Hindernis-Sperre    | Fährt nicht nach unten, solange ein Sperr-Sensor an ist (z.B. Fliegengittertür offen, jemand auf der Terrasse); mit Wartezeit                                             | Kontakt-/Präsenzsensor (optional)          |

**Prioritäten:** Ganz oben steht das **Notfall-Öffnen** — meldet ein Notfall-Sensor
Alarm, fährt der Rollladen hoch und bleibt oben, egal was Sturmschutz, Pause oder
Nachtmodus wollen (Fluchtweg bzw. Schutz des Behangs gehen vor). Für Fahrten nach
unten gilt außerdem die **Hindernis-Sperre**: Solange ein Sperr-Sensor an ist, fährt
die Automatik den Rollladen nicht herunter, auch nicht für Nachtmodus, Panzer-Modus
oder den Benachrichtigungs-Knopf. Ansonsten gewinnt der **Sturmschutz** — bei Starkwind bewegen weder Morgens-Öffnen noch Beschattung,
Sonnenheizen oder das Zurückfahren den Rollladen, und auch der Pausier-Helfer hält
ihn nicht auf (Schutz der Hardware geht vor).
Danach kommt die Pause (solange ihr Helfer an ist, passiert sonst gar nichts),
dann der Nachtmodus (nachts wird nicht beschattet, nicht geheizt und beim
Fensteröffnen nur bis zur Lüftungsposition geöffnet), dann erst die Komfort-Features.
Der **Frostschutz** ist keine eigene Fahrt, sondern eine Obergrenze für deren
Öffnungsfahrten — den Sturmschutz begrenzt er bewusst nicht.

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

### Manuelle Eingriffe während der Beschattung

Die Automation weiß nie, _wer_ den Rollladen bewegt hat — sie vergleicht bei jedem
Tick nur die Ist-Position mit ihrem berechneten Sollwert:

- Abweichung **unter 5 %**: nichts zu tun (Motorschonung).
- **5–20 %**: normales Nachführen der Sonne.
- **über 20 %**: Das kann keine Sonnenwanderung sein — ein Mensch war am Werk. Der
  Rollladen wird in Ruhe gelassen.

Diese "Sperre" gilt standardmäßig **bis zum Ende der laufenden Beschattungs-Episode**
(Sonne verlässt das Sichtfeld, es kühlt ab, der Nachtmodus kommt oder der
Freigabe-Helfer geht aus). Das Episoden-Ende
öffnet den Rollladen dann regulär — auch über die manuelle Position hinweg. Am
nächsten Tag beginnt alles bei null; die Anfangsbewegung ist von der Toleranz
ausgenommen. Stellst du den Rollladen manuell ungefähr dorthin, wo die Beschattung
ihn haben will, übernimmt das Nachführen wieder stillschweigend. Sturm, Lüften und
Morgens-Öffnen zählen dagegen nicht als manuelle Eingriffe — nach ihnen darf sofort
wieder beschattet werden.

Soll ein Handgriff nicht die ganze Episode lang gelten, begrenzt die **Dauer für
manuelle Eingriffe** die Sperre zeitlich (z. B. 60 Minuten): Steht der Rollladen
seit mindestens dieser Dauer still, fährt die Beschattung wieder auf ihre
Sollposition und führt danach normal nach. Jede weitere Bewegung — etwa ein
erneuter Handgriff — startet die Wartezeit neu. Das gilt in beide Richtungen: Auch
ein von Hand weiter geschlossener Rollladen fährt danach wieder auf die
Sollposition. Geprüft wird im 5-Minuten-Takt der Beschattung, die Rückkehr erfolgt
also bis zu 5 Minuten nach Ablauf. Mit 0 (Standard) bleibt es beim bisherigen
Verhalten.

Wer die Beschattung dauerhaft nicht will, deaktiviert den Schalter "Sonnenschutz
aktivieren" in der Instanz — der Status-Helfer ist **kein** Ausschalter, er ist das
interne Gedächtnis der Automation und stellt sich bei Handbetätigung einfach zurück.
Zum zeitweisen Sperren (z. B. tageweise) gibt es den optionalen Freigabe-Helfer —
siehe FAQ.

### Nachtmodus per Helfer oder per Uhrzeit

Der Nachtmodus lässt sich pro Fenster auf zwei Arten auslösen — **immer nur eine davon**:

- **Helfer** (`input_boolean`): der gemeinsame Schalter für alle Fenster. Die Nacht
  dauert, solange er eingeschaltet ist.
- **Uhrzeit** (`input_datetime`, nur Uhrzeit): Dieses eine Fenster schließt zu einer
  eigenen Zeit, z. B. das Kinderzimmer früher als der Rest. Die Nacht gilt dann ab
  dieser Uhrzeit bis zur Uhrzeit für morgendliches Hochfahren (sofern dort ein Helfer
  gewählt ist — auch wenn das Öffnen selbst deaktiviert ist), sonst bis Sonnenaufgang.

In beiden Fällen verhält sich die Nacht gleich: keine Beschattung, kein Sonnenheizen,
und ein geöffnetes Fenster bekommt nur die Lüftungsposition. Blueprints können "nur
eines von beiden" im Formular nicht erzwingen. Sind beide Felder gesetzt, gilt der
Helfer, die Uhrzeit wird ignoriert, und eine dauerhafte Benachrichtigung in Home
Assistant benennt das betroffene Fenster.

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
bleibt.

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
(schließen bei Sturm) stellt sich die Frage ohnehin nicht.

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
Nachtmodus nach (bei offenem oder gekipptem Fenster mit Lüftungsposition, bei
Sturm gar nicht, während einer Pause erst zu deren Ende). Die Beschattung bewertet
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

## Bekannte Grenzen

- **Wind-Sensor kurz nicht verfügbar** zählt als "windstill". Bewusste Entscheidung:
  Ein dauerhaft toter Sensor soll nicht sämtliche Komfort-Funktionen lahmlegen. Der
  Sturmschutz selbst hat beim Überschreiten des Grenzwerts längst ausgelöst.
- **Wetterlagen-Filter:** Flattert das Wetter zwischen zwei _nicht_ erlaubten Lagen
  (z. B. Regen ↔ Starkregen), beendet erst Sonnenstand oder Temperatur die
  Beschattung. Der Filter beendet nur bei mindestens 10 Minuten stabil schlechter Lage.
- **Cover ohne Positions-Angabe** (nur auf/zu): Morgens-Öffnen funktioniert, der
  Nachtmodus schließt ganz (statt auf die Nachtposition), Kipp-Position und Beschattung
  werden übersprungen — sie brauchen Positionsdaten. Auch der Frostschutz kann solche
  Cover nicht begrenzen: Öffnen heißt hier immer ganz auf.
- **Frostschutz:** Kipp- und Nachtlüftungs-Position werden nicht begrenzt (sie liegen
  normalerweise weit unter der Frost-Position), ebenso wenig das Zurückfahren nach dem
  Lüften (es stellt nur die Position von vorher wieder her). Auch die Nachführung der
  Beschattung bleibt unbegrenzt — sie startet erst ab der Beschattungs-Schwelle
  (mindestens 10 °C). Ohne verfügbare Temperaturquelle greift der Frostschutz nicht.
- **Windgeschwindigkeit** wird roh mit dem Grenzwert verglichen — liefert deine Quelle
  m/s statt km/h, muss der Grenzwert entsprechend gesetzt werden.
- **Sturm-Ende:** Nach dem Sturm bleibt der Rollladen in der Schutzposition, bis das
  nächste reguläre Ereignis (Nachtmodus, Morgens, Beschattung) ihn übernimmt.
- **"Nur anwesend" holt nichts nach:** Kommt jemand erst nach Ablauf des Timeouts nach
  Hause, während das Fenster noch offen ist, gibt es keine nachträgliche Meldung — der
  Trigger feuert nur einmal.
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
- **Sturm nach dem Notfall:** Hält der Sturm nach dem Notfall an, wird der
  Sturmschutz nicht nachgeholt — im Panzer-Modus bleibt der Rollladen oben, bis der
  Wind die Schwelle erneut überschreitet oder ein reguläres Ereignis ihn übernimmt.
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

## FAQ

**Was bedeuten 0 % und 100 % bei den Positionen?** Das Blueprint folgt der
Home-Assistant-Konvention: 100 % = ganz offen, 0 % = ganz geschlossen. Die Prozente
sind dabei die Werte deines Cover-Aktors, also Motor-Laufweg — nicht zwingend
Glasfläche. Für die Beschattung lässt sich dieser Unterschied über die
Glas-Kalibrierung (siehe oben) ausgleichen; alle anderen Positions-Eingaben
(morgens, Kipp-Position, Nachtpositionen, Frost-Position) sind bewusst direkte
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
Außentemperatur über der Schwelle, steht die Sonne im Sichtfeld (Ausrichtung
korrekt?) und zwischen minimaler und maximaler Sonnenhöhe, und ist das Fenster nicht
komplett offen? Ist ein Freigabe-Helfer gesetzt, muss er eingeschaltet sein.

**Über dem Fenster ist ein Balkon oder Vordach — mittags wird trotzdem verdunkelt?**
Die maximale Sonnenhöhe im Abschnitt "Fenstergeometrie & Sonnenausrichtung" auf den
Wert setzen, ab dem das Vordach die Glasfläche abschattet (siehe "Sichtfeld und
Geometrie"). Darüber öffnet der Rollladen auf die Position nach der Beschattung;
am Nachmittag beschattet die Automation bei Bedarf wieder.

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

**Kann ich denselben Status-Helfer für mehrere Fenster verwenden?** Nein — er
speichert den Zustand genau eines Fensters. Ein geteilter Helfer führt zu falschem
Öffnen/Schließen.

**Sonnenschutz und Sonnenheizen gleichzeitig aktiv — geht das?** Ja, das ist der
Normalfall. Die Temperatur-Schwellen trennen sie (Standard: beschatten über 25 °C,
heizen unter 12 °C); die Schwellen sollten sich nicht überlappen.

**Die Fenster-offen-Meldung bleibt auf dem Handy stehen?** Sie verschwindet
automatisch, sobald das Fenster geschlossen wird — vorausgesetzt, die Companion-App
ist aktuell (das Aufräumen nutzt `clear_notification` mit Tags).

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
Nachtmodus nach und bewertet die Beschattung neu; verpasste Einzelereignisse
(morgendliches Öffnen, Zurückfahren nach dem Lüften) werden nicht nachgeholt.

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
