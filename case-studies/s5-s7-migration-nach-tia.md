# Case Study: Migration von S5-/S7-Altstil nach IEC-konformem SCL im TIA-Portal

## Problem

Programme, die von S5 oder S7-300/400 nach TIA überführt wurden, laufen produktiv — sind aber nur *migriert*, nicht *modernisiert*. Verändert hat sich die Plattform, nicht die Denkweise: absolute Adressierung, globale Merker als Bit-Bus zwischen Bausteinen, byte-genaue Datenbaustein-Layouts als Kopplung zu Visualisierung und übergeordneten Systemen, Sprungmarken statt Kontrollstrukturen, Umrangier-Bausteine anstelle definierter Schnittstellen.

Zwei sehr unterschiedliche Probleme werden dabei regelmäßig in einen Topf geworfen — beide unter der Sammelüberschrift „wir müssen von AWL weg":

**Fall 1 — funktional modern, stilistisch veraltet.** Ein Programm ist bereits vollständig in SCL geschrieben, folgt aber einer älteren herstellereigenen Notation mit Datentyp-Präfixen. Die naheliegende Frage „ist das überhaupt IEC-61131-3-konform?" lässt sich nicht beantworten, solange Norm und Hausstil nicht getrennt betrachtet werden. Ohne diese Trennung wird jede Stildiskussion zur Geschmacksfrage — und damit unentscheidbar.

**Fall 2 — Altstil unter neuer Plattform.** Die Sprache ist teils schon FUP oder KOP, das Programmiermuster aber unverändert S5. Hier wird der Sprachwechsel als *die* Aufgabe gesehen, obwohl er der kleinere Teil ist.

Das eigentliche Problem in beiden Fällen ist dasselbe: **Migrationsentscheidungen werden ohne Zahlen getroffen.** Aufwand, Blast-Radius und Risiko sind unbekannt. Deshalb wird entweder gar nicht migriert („zu riskant") oder in einem Big-Bang-Ansatz alles gleichzeitig angefasst — Umbenennung, Umstrukturierung und Sprachwechsel in einem Schritt, mit einem Diff, das kein Review mehr zulässt. Bei einer Anlage in laufender Produktion ist beides teuer.

## Lösung

Ich habe die beiden Fälle getrennt bearbeitet und für jeden eine belegte, rückbaubare Vorgehensweise erstellt — kein Vorschlag auf Zuruf, sondern jeweils mit quantifizierter Auswirkung. Ergebnis ist ein Vorgehensmodell, das auf beliebige Bestandsprogramme anwendbar ist.

**Teil A — Stilnormalisierung mit zwei vollständigen Varianten.** Statt eine Namenskonvention zu behaupten, habe ich das Programm in zwei kompletten Stilvarianten erzeugt und deren Auswirkung auf die Umgebung ausgezählt: eine streng nach aktuellem Hersteller-Styleguide (Datentyp-Präfixe entfernt, Objekte in `UpperCamelCase`, Bausteine verb-first) und eine Kompromissvariante (Datentyp-Präfixe beibehalten und vereinheitlicht, nur Baustein- und Datenbaustein-Präfixe entfernt). Beide Varianten liegen als importfähige Quellen vor, inklusive Umbenennungs-Mapping und dokumentierter Import-Reihenfolge.

**Teil B — Ist-Analyse und Phasenplan für ein Bestandsprogramm.** Aus den Bausteinexporten eines produktiv laufenden Programms habe ich eine Kennzahlenbasis erhoben und daraus einen Sieben-Phasen-Plan mit Risikoklassen abgeleitet.

Zentrale Design-Entscheidungen:

- **Safety-Gate vorgeschaltet, nicht überspringbar**: Vor jeder Bausteinbetrachtung wird geprüft, ob fehlersichere Logik betroffen ist. Fehlersichere Bausteine dürfen nicht nach SCL überführt werden — dort sind ausschließlich F-FUP und F-KOP zulässig. Standard- und Safety-Anteile werden sauber getrennt statt gemeinsam migriert.
- **Norm und Hausstil strikt getrennt**: IEC 61131-3 regelt Sprachen, Bezeichnerregeln und POU-Modell — „richtig oder falsch". Der Hersteller-Styleguide regelt Hausstil — „sauber oder unsauber". SCL erfüllt die Norm praktisch bereits; die eigentliche Arbeit steckt in Konsistenz und Hausstil. Diese Trennung macht die Diskussion überhaupt erst entscheidbar.
- **Verhaltensneutrale Phasen zuerst**: Symbolisierung der absoluten Zugriffe und Einführung von PLC-Datentypen ändern das Laufzeitverhalten nicht, liefern aber den Großteil der Wartbarkeit. Wird das Vorhaben nach diesen Phasen gestoppt, ist der Zwischenstand trotzdem ein Gewinn.
- **Eingefrorene Schnittstellen („Frozen Interfaces")**: Datenbausteine mit externer Byte-Offset-Kopplung — Visualisierung, übergeordnete Systeme, Peripheriebaugruppen — bleiben unverändert im Standard-Zugriff. Der Zugriff erfolgt ausschließlich über einen Mapping-Baustein, der zwischen PLC-Datentyp und Byte-Layout umsetzt. Der Umbau der Visualisierung ist damit explizit ein eigenes Projekt und kein Nebeneffekt.
- **Risikoklassen statt Reihenfolge nach Gefühl**: Jeder Baustein wird in eine von vier Klassen einsortiert — von prozessfrei bis qualitäts- und rechtsrelevant. Migriert wird klassenweise, maximal ein Baustein pro Umschaltung, dazwischen produktiver Betrieb. Der alte Baustein bleibt im Projekt, bis die Abnahme durch ist.
- **Bewusstes Nicht-Migrieren als Teil des Plans**: Ablaufketten in der grafischen Schrittkettensprache bleiben, wo sie sind — das ist die IEC-konforme Lösung (SFC), und eine Übersetzung nach `CASE` würde die Kettendiagnose in der Visualisierung zerstören. Bibliotheksbausteine des Baugruppenherstellers bleiben unangetastet, nur ihre Aufrufer werden migriert. Der Hauptorganisationsbaustein wird zuletzt und nur als reine Aufrufliste angefasst.
- **Originale grundsätzlich unangetastet**: Es wird ausschließlich auf Kopien gearbeitet, jede Variante liegt in einem eigenen Ordner. Vor jeder Umschaltung Projektarchiv plus Online-Sicherung der Aktualwerte.
- **Automatisierung realistisch eingeordnet**: Die Openness-Schnittstelle des Engineering-Werkzeugs kann exportieren, importieren, symbolisieren, Kreuzreferenzen ziehen und massenhaft umbenennen. Sie kann **nicht** FUP oder KOP nach SCL übersetzen. Von sieben Phasen ist genau eine wirklich skriptbar — diese Erwartung vorab zu korrigieren verhindert eine Fehlplanung.

## Technologien

TIA-Portal V18–V20, Siemens S7-1500, SCL/ST nach IEC 61131-3, PLC-Datentypen (UDT/STRUCT) und ARRAY, optimierter Bausteinzugriff, IEC-Timer (TON/TOF/TP) als Multiinstanz, `MOVE_BLK` statt untypisierter Byte-Kopie (`BLKMOV` mit `P#`-Pointern), GRAPH/SFC-Ablaufketten, TIA-Portal-Openness-API (XML-Export/Import, Massen-Umbenennung, Kreuzreferenz), PLCSIM Advanced, Trace-Aufzeichnung, PROFINET, Kopplung an Visualisierung über absolute Datenbaustein-Offsets, Siemens Programmier-Styleguide GL002 (Beitrags-ID 81318674, V2.1.0).

## Ergebnis

**Teil A — Stilnormalisierung: aus einer Geschmacksfrage wurde eine Rechnung.**

Der IEC-Befund ist eindeutig: Das Programm ist im Kern normkonform. POU-Modell, Datentypen, `TYPE…END_TYPE`, Kontrollstrukturen, Deklarationsabschnitte und Zeitliterale entsprechen der Norm. Die einzigen echten Normverstöße waren **8 Bezeichner mit Punkt oder Bindestrich**, die nur durch Quoting funktionierten — nach IEC 61131-3 sind ausschließlich Buchstaben, Ziffern und Unterstrich zulässig.

Beide Stilvarianten wurden vollständig erzeugt: je **14 Bausteinquellen und 4 globale Datenbausteine**, jeweils mit Umbenennungs-Mapping und dokumentierter Import-Reihenfolge. Der entscheidende Unterschied ist messbar:

| | Streng nach Styleguide | Kompromiss |
|---|---|---|
| Extern betroffene Namen | praktisch alle Bausteine, Parameter, Datentypen, Datenbausteine, Konstanten | **9 Objektnamen + 8 Parameter-Elemente** |
| Auswirkung auf Aufrufer, Visualisierung, Partner-Steuerung | komplette Neuverdrahtung inklusive Re-Test | 9 Namen bei Aufrufern, 8 Tags in der Visualisierung |
| Eignung | Neuentwicklung / Greenfield | Anlage in Produktion |

Damit war die Entscheidung begründbar statt geschmacklich: **Kompromissvariante für die laufende Anlage, strenge Variante nur für Neuentwicklung** oder wenn ein vollständiger Re-Test ohnehin eingeplant ist.

Zwei Erkenntnisse aus der Umsetzung, die sich verallgemeinern lassen: Reines Umbenennen ist **verhaltensneutral** — der Datenaustausch läuft über Offsets, nicht über Namen, das Byte-Layout bleibt identisch. Das Risiko liegt nicht in der Funktion, sondern in der **Integration**. Und: „alles konvertieren" heißt wirklich alles — auch die Datenbaustein-Quellen sind Textdateien und trugen dieselben Altnamen samt eingebetteter Alt-Datentypen. Mehrere Quellen definierten denselben Datentyp eingebettet, was eine feste Import-Reihenfolge (Datentypen zuerst) und „Typ überschreiben" erzwingt.

**Teil B — Ist-Analyse: der Befund, der die Aufwandsschätzung umdreht.**

Die konkreten Stückzahlen sind hier bewusst nicht angegeben. Entscheidend ist nicht ihre absolute Höhe, sondern ihr **Verhältnis zueinander** — und genau das ist auf andere Programme übertragbar.

| Erhobene Kennzahl | Befund |
|---|---|
| Bausteine nach Typ (OB / FB / FC / DB) | XXX |
| Netzwerke gesamt | XXX |
| Sprachverteilung | überwiegend FUP, einzelne GRAPH- und SCL-Bausteine, wenig KOP |
| reine **AWL**-Netzwerke | XXX — nur ein kleiner einstelliger Prozentanteil des Gesamtprogramms |
| absolute Datenbaustein-Zugriffe | XXX — **um mehr als eine Größenordnung häufiger** als AWL-Netzwerke |
| Merker-Zugriffe | XXX Zugriffsstellen auf XXX unterschiedliche Adressen |
| Sprunganweisungen | XXX |
| untypisierte Byte-Kopien mit Pointer-Konstrukten | XXX, verteilt über mehrere Bausteine |
| S5-Timer | XXX Instanzen, zusätzlich XXX S5-Zeitliterale |
| bereits vorhandene IEC-Timer | deutlich mehr als S5-Timer — die Modernisierung war begonnen, aber nicht abgeschlossen |
| Bausteinzugriff | durchgängig Standard (nicht optimiert) — Interfaces mit Byte-Offsets |

**Der Kernbefund:** Das Programm ist kein AWL-Programm. Es ist ein FUP-Programm im S5-Denkmuster. Die **überwiegende Mehrheit der Netzwerke ist gar nicht AWL** — der Migrationsaufwand steckt nicht in den AWL-Netzwerken, sondern in den absoluten Datenbaustein-Zugriffen und der Byte-Layout-Kopplung nach außen. Die Zahl der zu symbolisierenden Zugriffsstellen übersteigt die Zahl der zu übersetzenden AWL-Netzwerke um mehr als eine Größenordnung.

Das ist das eigentliche Ergebnis der Analyse: Die verbreitete Annahme „AWL → SCL ist eine Übersetzungsaufgabe" ist für Programme dieses Typs falsch. Eine darauf beruhende Aufwandsschätzung verfehlt den Bedarf erheblich — und zwar in die Richtung, die im Projekt am teuersten ist: zu optimistisch.

Weitere Ergebnisse:

- **Kritische Programmpunkte und Projektrisiken systematisch erfasst**, jeder mit konkreter Gegenmaßnahme. Die gefährlichsten sind die **stummen** Fehler: eine Layout-Änderung an einem visualisierungsgekoppelten Datenbaustein zeigt falsche Werte, statt eine Störung auszulösen; eine untypisierte Byte-Kopie mit nicht mehr passender Quell-/Ziellänge erzeugt Datenmüll ohne CPU-Fehler. Beides fällt im Test nur auf, wenn gezielt danach gesucht wird.
- **Zeitverhalten als eigene Risikoklasse erkannt**: S5-Timer arbeiten mit einem gestuften Zeitraster (10 ms / 100 ms / 1 s / 10 s) und runden anders als IEC-Timer. Ein nominal identischer Zeitwert kann nach der Umstellung real anders auslösen. Wo eine Zeit qualitätsrelevant ist, ist das kein Schönheitsfehler, sondern ein Produktfehler.
- **Ein bis dahin unbekannter Anlagenzustand aufgedeckt**: Zwei Versionen desselben Funktionsbausteins laufen parallel produktiv, ohne dokumentierten Unterschied und ohne klare Zuordnung, welche Instanz welche Version nutzt. Das muss **vor** der Migration vereinheitlicht werden — andernfalls wird ein unbekannter Fehler mitmigriert und ist danach nicht mehr vom Migrationsfehler unterscheidbar.
- **Empfohlener Zuschnitt**: Die Migration der Risikoklassen A bis C erfasst deutlich weniger als das Gesamtprogramm, adressiert aber den überwiegenden Teil des Wartungsaufwands. Ausgenommen bleiben Bibliotheksbausteine des Baugruppenherstellers, die grafischen Ablaufketten und die Byte-Layouts zu Visualisierung und übergeordneten Systemen. Qualitäts- und rechtsrelevante Pfade werden separat entschieden — nur mit Testsystem und geklärter rechtlicher Lage.
- **Verhaltensgleichheit operationalisiert** statt behauptet: Ausgangs-Bitmuster über mindestens 1.000 Zyklen bei identischem Eingangsvektor, Zeitverhalten aller konvertierten Timer auf ±1 Zyklus (bei qualitätsrelevanten Zeiten ±0), Prozess-Endwerte über 100 simulierte Vorgänge inklusive Grenzfälle (Sollwert 0, Sollwert unterhalb der Regelreserve, Toleranzfehler, Ausgangszustand nicht neutral, Abbruch mitten im Vorgang), Anlaufverhalten nach Netz-Aus und nach Urlöschen, visueller Vergleich aller betroffenen Visualisierungsbilder, Zykluszeit nicht schlechter als vorher.
- **Rollback in unter 15 Minuten**: Der alte Baustein bleibt bis zur Abnahme im Projekt aufrufbar, Aktualwerte werden vor jeder Umschaltung gesichert. Gelöscht wird erst nach einer vollen Produktionswoche ohne Auffälligkeit. Hartes Abbruchkriterium vorab definiert: ein Produktionslos außerhalb der Toleranz durch Softwarefehler führt zum sofortigen Rollback mit Ursachenanalyse vor Wiederanlauf.

**Übertragbarer Ertrag.** Aus beiden Teilen ist ein wiederverwendbares Vorgehensmodell entstanden — Safety-Gate, Klassifizierung, verhaltensneutrale Phasen zuerst, eingefrorene Schnittstellen, Test-Gates, Rollback —, das auf jedes weitere Bestandsprogramm anwendbar ist, unabhängig von Branche und Prozess.

Der teuerste Fehler wäre gewesen, die Sprache zu migrieren und die Adressierung zu behalten: kosmetisch modern, im Kern unverändert, und mit einem Testaufwand, dem kein Nutzen gegenübersteht.

Ein Nebenergebnis ist für die Organisation wichtiger als der Code: Die prozesskritische Parametrier- und Toleranzlogik war nirgends dokumentiert und lag ausschließlich im Kopf einer Person. Die Analysephase hat sie erstmals schriftlich festgehalten. Das ist unabhängig vom Sprachwechsel bereits der belastbarste Teil des Business-Case — und der Grund, warum eine Migration bei ausscheidendem Know-how nicht beliebig aufschiebbar ist.
