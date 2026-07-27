# Case Study: Migration von S5-/S7-Altstil nach IEC-konformem SCL im TIA-Portal

## Problem

Programme, die von S5 oder S7-300/400 nach TIA überführt wurden, laufen produktiv — sind aber nur migriert, nicht modernisiert. Verändert hat sich die Plattform, nicht die Denkweise: absolute Adressierung, globale Merker als Bit-Bus zwischen Bausteinen, byte-genaue Datenbaustein-Layouts als Kopplung zu Visualisierung und übergeordneten Systemen, Sprungmarken statt Kontrollstrukturen, Umrangier-Bausteine anstelle definierter Schnittstellen.

Das ist kein Versäumnis einzelner Personen, sondern der übliche Zwischenstand. Der Steuerungshersteller sagt es selbst: Mit dem erzeugten TIA-Portal-Projekt sei die vollständige Migration „noch nicht abgeschlossen", eine anschließende Optimierung in den meisten Fällen notwendig (Migrationsleitfaden 109478811, Kap. 5.2.5).

Erschwerend werden zwei sehr unterschiedliche Fälle regelmäßig in einen Topf geworfen: ein Programm, das bereits vollständig in SCL vorliegt, aber einer veralteten herstellereigenen Notation folgt — und ein Programm, dessen Sprache teils schon FUP oder KOP ist, dessen Muster aber unverändert S5 geblieben ist. Im ersten Fall ist die Frage „ist das IEC-61131-3-konform?" nicht beantwortbar, solange Norm und Hausstil nicht getrennt betrachtet werden. Im zweiten wird der Sprachwechsel für die eigentliche Aufgabe gehalten, obwohl er der kleinere Teil ist.

Das gemeinsame Kernproblem: **Migrationsentscheidungen werden ohne Zahlen getroffen.** Aufwand, Blast-Radius und Risiko sind unbekannt. Deshalb wird entweder gar nicht migriert („zu riskant") oder alles gleichzeitig angefasst — Umbenennung, Umstrukturierung und Sprachwechsel in einem Schritt, mit einem Diff, das kein Review mehr zulässt. Bei einer Anlage in laufender Produktion ist beides teuer.

---

## Lösung

Ich bin in drei Schritten vorgegangen: auszählen, zwei Stilvarianten vollständig bauen und rechnen — und anschließend jede tragende Aussage gegen die Originaldokumentation des Herstellers prüfen.

Der dritte Schritt hat sich als der wichtigste erwiesen, weil er meine eigene Arbeit korrigiert hat:

> **Was ich zurückziehen musste.** Mein Vorgehensmodell enthielt die Erwartung, SCL sei auf der neuen
> Steuerungsgeneration „in der Regel gleich schnell oder schneller". Der Hersteller sagt für migrierte
> Programme das Gegenteil: Weil sich Befehlssätze und -strukturen unterscheiden, könne ein migriertes
> Programm auf der S7-1500 **langsamer** laufen als auf der S7-300/400, obwohl die technischen Daten
> deutlich für die neue Steuerung sprechen (109478811, Kap. 5.2.5).
>
> Der Leistungsgewinn kommt nicht aus der Sprache, sondern aus optimierten Bausteinen und symbolischem
> Zugriff. Konsequenz für den Plan: Zykluszeit wird vor und nach jedem Schritt **gemessen**, nicht
> versprochen. Das ist kein Argument gegen die Modernisierung — es ist eines gegen unbelegte Zusagen,
> die beim ersten Gegenmessergebnis das ganze Vorhaben unglaubwürdig machen.

Ein zweiter Befund aus derselben Prüfung war gravierender, weil er eine Lücke im Safety-Gate aufdeckte: Ein Baustein muss gar nicht fehlersicher sein, um blockiert zu sein. Greift das Sicherheitsprogramm lesend oder schreibend auf einen **Standard**-Baustein zu, verliert es bei dessen Änderung seine Konsistenz — ein Download ist dann nur im Steuerungs-Stopp möglich (Safety-Leitfaden 109750255, Kap. 3.7.1). Solche Bausteine sind bei laufender Anlage nicht migrierbar, unabhängig davon, wie unkritisch ihre Logik ist. Sie haben eine eigene Sperrklasse und ein Stillstandsfenster bekommen. Ohne diese Prüfung wäre der Fehler erst beim ersten ungeplanten Anlagenstopp aufgefallen.

**Zentrale Design-Entscheidungen:**

- **Safety-Gate vorgeschaltet, nicht überspringbar.** Fehlersichere Bausteine werden nicht nach SCL überführt — im Safety-Werkzeug stehen ausschließlich die grafischen Sprachen FUP und KOP zur Verfügung (109750255, Kap. 3.1.6). Standard- und Safety-Anteile werden getrennt statt gemeinsam migriert; für die Kopplung sieht der Hersteller drei Koppel-Datenbausteine vom Typ eines F-UDT vor, befüllt in der Vorverarbeitung der F-Ablaufgruppe (Kap. 3.7.1/3.7.2).
- **Norm und Hausstil strikt getrennt.** IEC 61131-3 regelt Sprachen, Bezeichnerregeln und POU-Modell — „richtig oder falsch". Der Hersteller-Styleguide regelt Hausstil — „sauber oder unsauber". Bei SCL ist die Norm praktisch erfüllt; die Arbeit steckt in Konsistenz und Hausstil. Erst diese Trennung macht die Diskussion entscheidbar.
- **Verhaltensneutrale Phasen zuerst.** Symbolisierung der absoluten Zugriffe und Einführung von PLC-Datentypen ändern das Laufzeitverhalten nicht, liefern aber den Großteil der Wartbarkeit. Wird das Vorhaben danach gestoppt, ist der Zwischenstand trotzdem werthaltig.
- **Eingefrorene Schnittstellen mit genau einem Konvertierungspunkt.** Datenbausteine mit externer Byte-Offset-Kopplung bleiben im Standard-Zugriff, der Zugriff läuft über einen Mapping-Baustein. Entscheidend ist, *wo* konvertiert wird: Der Hersteller warnt ausdrücklich vor der Mischung optimierter und nicht optimierter Bausteine — wegen impliziter Konvertierung bei jeder Zuweisung und Parameterübergabe habe sie „in der Regel den gegenteiligen Effekt" auf die Performance (Programmierleitfaden 81318674, Kap. 3.9). Der Mapping-Baustein ist deshalb als **Systemgrenze** konzipiert: ein Konvertierungspunkt je Fremdsystem, ausgelöst nur bei neuen Daten.
- **Risikoklassen statt Reihenfolge nach Gefühl.** Migriert wird klassenweise, maximal ein Baustein pro Umschaltung, dazwischen produktiver Betrieb. Der alte Baustein bleibt bis zur Abnahme aufrufbar.
- **Bewusstes Nicht-Migrieren als Teil des Plans.** Grafische Ablaufketten bleiben — das ist bereits die IEC-konforme Lösung (SFC), und eine Übersetzung nach `CASE` zerstört die Kettendiagnose. Der Hersteller liefert zusätzlich das Laufzeitargument: Programmiert man die Kettenfunktionalität selbst nach, führt das „zu ähnlichen Laufzeiten" bei zusätzlichem Aufwand (Kap. 3.9). Ein Nachbau kauft also nichts und kostet Diagnose.
- **Änderbarkeit im laufenden Betrieb als Planungsgröße.** Optimierte Bausteine erlauben das Erweitern ihrer Schnittstelle im Betriebszustand RUN ohne Verlust der Aktualwerte — vorausgesetzt, vorher wurde eine Speicherreserve definiert (Default 100 Byte, maximal 2 MB, pro Baustein einzeln; Kap. 3.2.8). Nachträglich ist das nicht aktivierbar, deshalb wird die Reserve in der Datentypen-Phase gesetzt, nicht bei Bedarf.
- **Automatisierung realistisch eingeordnet.** Die Openness-Schnittstelle kann exportieren, importieren, symbolisieren, Kreuzreferenzen ziehen und massenhaft umbenennen — aber nicht FUP oder KOP nach SCL übersetzen. Von sieben Phasen ist genau eine wirklich skriptbar.

---

## Der Code: was „nicht IEC-konform" konkret bedeutet

Der Kern der Stilfrage lässt sich an einer Struktur zeigen. **Original** — die Bezeichner enthalten Punkte und einen Bindestrich und funktionieren nur, weil sie durchgängig gequotet sind:

```pascal
TYPE "CarrierDataParameter"
VERSION : 0.1
   STRUCT
      "rPressForce-inDim." { S7_SetPoint := 'True'} : Real;
      "aComponentA.Mat.Num"  : Array[0..19] of USInt;   // Bytes = 20
      "aComponentB.Seg.Mat.Num" : Array[0..19] of USInt;   // Bytes = 20
      aMarkingIns : Array[0..19] of USInt;   // Bytes = 20
      aReserve : Array[0..89] of USInt;
   END_STRUCT;
END_TYPE
```

Nach IEC 61131-3 sind in Bezeichnern ausschließlich Buchstaben, Ziffern und Unterstrich zulässig — Punkt und Bindestrich sind unzulässig, ein Punkt am Wortende ohnehin. Das sind **echte Normverstöße**, nicht Geschmack. Sichtbar wird die Absurdität an dieser Zuweisung aus dem Originalprogramm, in der derselbe Wert zweimal unterschiedlich heißt:

```pascal
#stCarrierDataParameter."rPressForce-inDim." := #tyParameter.rPressForceInDim;
```

**Variante 1 — streng nach Hersteller-Styleguide V2.1.0:** Datentyp-Präfixe entfallen, Codeelemente in `lowerCamelCase`, Typen mit `type`-Präfix.

```pascal
TYPE "typeCarrierDataParameter"
VERSION : 0.1
   STRUCT
      pressForceInDim { S7_SetPoint := 'True'} : Real;
      componentAMatNum    : Array[0..19] of USInt;   // Bytes = 20
      componentBSegMatNum : Array[0..19] of USInt;   // Bytes = 20
      markingIns : Array[0..19] of USInt;   // Bytes = 20
      reserve : Array[0..89] of USInt;
   END_STRUCT;
END_TYPE
```

**Variante 2 — Kompromiss:** Normverstöße behoben, Datentyp-Präfixe bewusst **beibehalten** und vereinheitlicht (`r` = Real, `a` = Array, `i` = Int). Grund: Die Anlage läuft. Jeder geänderte Elementname ist ein Tag in der Visualisierung und ein Datenpunkt in der Partner-Steuerung.

```pascal
TYPE "CarrierDataParameter"
VERSION : 0.1
   STRUCT
      rPressForceInDim { S7_SetPoint := 'True'} : Real;
      aComponentAMatNum    : Array[0..19] of USInt;   // Bytes = 20
      aComponentBSegMatNum : Array[0..19] of USInt;   // Bytes = 20
      aMarkingIns : Array[0..19] of USInt;   // Bytes = 20
      aReserve : Array[0..89] of USInt;
   END_STRUCT;
END_TYPE
```

Beide Varianten sind normkonform. Der Unterschied ist keine Qualitätsfrage, sondern eine Frage des Blast-Radius — und genau der wurde ausgezählt statt diskutiert (siehe Ergebnis).

Dasselbe Prinzip auf Bausteinebene, hier an einer Funktion mit unveränderter Logik. **Original:**

```pascal
FUNCTION "FcStationCondition" : Void
{ S7_Optimized_Access := 'TRUE' }
   VAR_INPUT
      xBusConnectionPartner : Bool;
      xTransportSystemOn  : Bool;
      xHomePosition       : Bool;
   END_VAR
   VAR_TEMP
      xBusTRUE : Bool;
   END_VAR
BEGIN
    REGION Bus connection is okay
        #xBusTRUE := TRUE;
        IF #xBusTRUE AND #xBusConnectionPartner THEN
            #xBusConnectionStation := TRUE;
        ELSE
            #xBusConnectionStation := FALSE;
        END_IF;
    END_REGION
END_FUNCTION
```

**Streng nach Styleguide:** `Fc`-Präfix entfällt (der Bausteintyp steht ohnehin im Projektbaum), Funktionen verb-first, Bool-Zustände mit `is`, temporäre Variablen mit `temp`-Präfix statt Datentyp-Präfix:

```pascal
FUNCTION "GetStationCondition" : Void
{ S7_Optimized_Access := 'TRUE' }
   VAR_INPUT
      busConnectionPartner : Bool;
      isTransportSystemOn : Bool;
      homePosition        : Bool;
   END_VAR
   VAR_TEMP
      tempBusTrue : Bool;
   END_VAR
BEGIN
    REGION Bus connection is okay
        #tempBusTrue := TRUE;
        IF #tempBusTrue AND #busConnectionPartner THEN
            #busConnectionStation := TRUE;
        ELSE
            #busConnectionStation := FALSE;
        END_IF;
    END_REGION
END_FUNCTION
```

Die Logik ist bitgleich. Das ist der Punkt: **Umbenennen ist verhaltensneutral** — der Datenaustausch läuft über Offsets, nicht über Namen, das Byte-Layout bleibt identisch. Das Risiko liegt nicht in der Funktion, sondern in der Integration.

---

## Technologien

TIA-Portal V18–V20, Siemens S7-1500, SCL/ST nach IEC 61131-3, PLC-Datentypen (UDT/STRUCT) und ARRAY, optimierter Bausteinzugriff, Laden ohne Reinitialisierung mit definierter Speicherreserve, IEC-Timer (TON/TOF/TP) als Multiinstanz, `MOVE_BLK` statt untypisierter Byte-Kopie (`BLKMOV` mit `P#`-Pointern), GRAPH/SFC-Ablaufketten, F-UDT-basierte Koppel-Datenbausteine zwischen Standard- und Sicherheitsprogramm, TIA-Bibliothekskonzept (Typen vs. Kopiervorlagen, dreistellige Versionierung), TIA-Portal-Openness-API, PLCSIM Advanced, Trace-Aufzeichnung, PROFINET.

Herstellerdokumentation als Referenz: Programmierleitfaden S7-1200/1500 (81318674, V1.6.1), Programmier-Styleguide (81318674, V2.1.0), Programmierleitfaden Safety (109750255, V1.6), Migrationsleitfaden S7-300/400 → S7-1500 (109478811, V1.1), Leitfaden Bibliothekshandhabung (109747503, V1.3).

---

## Ergebnis

### Der Befund, der die Aufwandsschätzung umdreht

| Kennzahl | Gemessen |
|---|---|
| Netzwerke gesamt | ca. **1.203** |
| davon reine AWL-Netzwerke | ca. **109** — rund **9 %** |
| absolute Datenbaustein-Zugriffe | ca. **4.518** |
| Merker-Zugriffe | **667** Stellen auf **138** Adressen |
| Sprunganweisungen (`SPB`/`SPA`) | **230** |
| untypisierte Byte-Kopien (`BLKMOV` mit `P#`) | **158** |
| S5-Timer | **20** Instanzen + 47 `S5T#`-Literale |
| bereits vorhandene IEC-Timer | deutlich mehr als S5-Timer |

**Das Programm ist kein AWL-Programm, sondern ein FUP-Programm im S5-Denkmuster.** Rund 91 % der Netzwerke sind gar nicht AWL. Der Migrationsaufwand steckt nicht in den 109 AWL-Netzwerken, sondern in den ca. 4.518 absoluten Zugriffen und der Byte-Layout-Kopplung nach außen — die Zahl der zu symbolisierenden Zugriffsstellen übersteigt die der zu übersetzenden AWL-Netzwerke um mehr als eine Größenordnung.

Eine Aufwandsschätzung nach der Annahme „AWL → SCL ist eine Übersetzungsaufgabe" verfehlt den Bedarf damit erheblich — und zwar zu optimistisch, also in die im Projekt teuerste Richtung.

Der Befund lässt sich in der Systematik des Herstellers verorten: Dessen Rangfolge der Zugriffsgeschwindigkeiten führt optimierte Bausteine mit symbolischem Zugriff an der Spitze; Zugriffe auf nicht optimierte Bausteine, indizierte Zugriffe mit Laufzeitindex und Zugriffe mit Laufzeitprüfung (Register-, indirekte Zugriffe) bilden das Ende (Programmierleitfaden, Kap. 3.4.4). Der Bestand liegt systematisch in den hinteren Rängen. Zum Merker-Bus ist die Position eindeutig: „Verwenden Sie keine Merker und nutzen Sie stattdessen Global-DBs" — begründet damit, dass der Merkerbereich aus Kompatibilitätsgründen nicht optimiert ist (Kap. 3.4.2). Aus einer Stilkritik wird damit eine dokumentierte Abweichung von der Herstellerempfehlung.

### Die Stilentscheidung — gerechnet statt diskutiert

Beide Varianten wurden vollständig erzeugt: je **14 Bausteinquellen und 4 globale Datenbausteine**, mit Umbenennungs-Mapping und dokumentierter Import-Reihenfolge. Der IEC-Befund war eindeutig: Das Programm ist im Kern normkonform, die einzigen echten Verstöße waren **8 Bezeichner** mit Punkt oder Bindestrich.

| | Streng nach Styleguide | Kompromiss |
|---|---|---|
| Extern betroffene Namen | praktisch alle Bausteine, Parameter, Datentypen, DBs, Konstanten | **9 Objektnamen + 8 Parameter-Elemente** |
| Auswirkung | komplette Neuverdrahtung aller Aufrufer inkl. Re-Test | 9 Namen bei Aufrufern, 8 Tags in der Visualisierung |
| Eignung | Neuentwicklung / Greenfield | **Anlage in Produktion** |

Entscheidung: Kompromiss für die laufende Anlage, streng nur für Neuentwicklung. Begründbar statt geschmacklich.

Eine Erkenntnis aus der Umsetzung, die sich verallgemeinern lässt: „alles konvertieren" heißt wirklich alles — auch die Datenbaustein-Quellen sind Textdateien und trugen dieselben Altnamen samt eingebetteter Alt-Datentypen. Mehrere Quellen definierten denselben Datentyp eingebettet, was eine feste Import-Reihenfolge (Datentypen zuerst) und „Typ überschreiben" erzwingt.

### Die stummen Fehler

Dokumentiert wurden 15 kritische Programmpunkte und 11 Projektrisiken, jeweils mit Gegenmaßnahme. Die gefährlichsten melden sich nicht — sie zeigen falsche Werte, statt zu stören:

- Eine Layout-Änderung an einem visualisierungsgekoppelten Datenbaustein zeigt falsche Werte, statt eine Störung auszulösen: Optimierte Bausteine kennen ausschließlich symbolischen Zugriff (Kap. 2.6.1).
- Eine untypisierte Byte-Kopie mit nicht mehr passender Quell-/Ziellänge erzeugt Datenmüll ohne CPU-Fehler.
- Temporäre Variablen sind in nicht optimierten Bausteinen bei jedem Aufruf **undefiniert**, in optimierten dagegen mit dem Defaultwert initialisiert (Kap. 3.4.3). Code, der unbeabsichtigt auf Restwerte baute, verhält sich nach der Umstellung anders.
- S5-Timer arbeiten mit einem gestuften Zeitraster und runden anders als IEC-Timer — ein nominal identischer Zeitwert kann real anders auslösen. Wo eine Zeit qualitätsrelevant ist, ist das kein Schönheitsfehler, sondern ein Produktfehler.

### Ein Anlagenzustand, der vorher unbekannt war

Die Analyse legte offen, dass **zwei Versionen desselben Funktionsbausteins parallel produktiv liefen** — ohne dokumentierten Unterschied und ohne klare Zuordnung, welche Instanz welche Version nutzt. Das muss vor der Migration vereinheitlicht werden, sonst wird ein unbekannter Fehler mitmigriert und ist danach nicht mehr vom Migrationsfehler unterscheidbar.

Der Bibliotheksleitfaden benennt den Mechanismus dahinter: Bausteine, die als *Kopiervorlage* statt als *Typ* weitergegeben werden, haben „keine Verbindung zu Ihren Verwendungsstellen im Projekt und keine Möglichkeit zur systemunterstützten Versionierung oder zentralen Änderbarkeit" (109747503, Kap. 1.3). Der Zustand ist damit vorhersehbar statt zufällig — und das Gegenmittel ist kein Appell an Disziplin, sondern ein Typkonzept mit dreistelliger Versionierung, in der die Hauptversionsnummer für nicht kompatible Schnittstellenänderungen reserviert ist (Kap. 8.6).

### Verhaltensgleichheit — operationalisiert statt behauptet

Bitmuster-Vergleich über mindestens 1.000 Zyklen bei identischem Eingangsvektor · Timer-Toleranz ±1 Zyklus, bei qualitätsrelevanten Zeiten ±0 · 100 simulierte Vorgänge inklusive Grenzfälle · Anlaufverhalten nach Netz-Aus und Urlöschen · gemessene Zykluszeit vor und nach jedem Schritt.

Rollback unter 15 Minuten, mit vorab definiertem hartem Abbruchkriterium — mit einer explizit dokumentierten Ausnahme: Bei Bausteinen der Sperrklasse erfordert auch der Rollback ein erneutes Übersetzen des Sicherheitsprogramms und einen Download im Steuerungs-Stopp. Dort ist das Rollback-Fenster kein Viertelstundenthema, sondern ein Stillstand. Das gehört vorab in den Plan, nicht in die Fehleranalyse.

### Was der Abgleich mit der Herstellerdokumentation bestätigt hat

Als Optimierungspunkte nach einer Werkzeugmigration nennt der Hersteller: optimierte Bausteine, Bausteingrößen, neue Datentypen, neue Anweisungen, Symbolik, Bibliothekskonzept und integrierte Bausteine (109478811, Kap. 5.2.5) — im Kern die Phasen 2 bis 6 meines Plans. Und für genau die Bausteinklasse, die in S5-Altprogrammen als Umrangier-Baustein auftritt, existiert eine ausdrückliche Empfehlung: Datenhandling, Suchalgorithmen, Kopier- und Vergleichsfunktionen sollen bei der Migration auf SCL umgestellt werden (Kap. 5.1.1).

Zur Vollständigkeit gehört, was die Dokumentation **nicht** hergibt: AWL bleibt für die S7-1500 im Engineering-Werkzeug verfügbar (Kap. 5.1.1). Die belastbare Aussage lautet nicht „AWL geht nicht mehr", sondern „für diese Bausteinklasse empfiehlt der Hersteller SCL, und der bestehende Stil hat dokumentierte Nachteile". Wer mehr behauptet, verliert die Diskussion bei der ersten Nachprüfung.

---

## Fazit

Der teuerste Fehler wäre gewesen, die Sprache zu migrieren und die Adressierung zu behalten — kosmetisch modern, im Kern unverändert, mit einem Testaufwand, dem kein Nutzen gegenübersteht. Der zweitteuerste wäre gewesen, einen Performance-Gewinn zuzusagen, den die Sprache nicht liefert.

Der empfohlene Zuschnitt migriert ca. 60 % der Netzwerke und adressiert damit ca. 90 % des Wartungsaufwands. Entstanden ist daraus ein wiederverwendbares Vorgehensmodell — Safety-Gate mit Kopplungsprüfung, Klassifizierung, verhaltensneutrale Phasen zuerst, eingefrorene Schnittstellen mit einem Konvertierungspunkt, Test-Gates, Rollback, abschließende Typisierung mit Versionierung —, das auf jedes weitere Bestandsprogramm anwendbar ist, unabhängig von Branche und Prozess.

Ein Nebenergebnis wog für die Organisation schwerer als der Code: Die prozesskritische Parametrier- und Toleranzlogik war nirgends dokumentiert und lag ausschließlich im Kopf einer Person, die absehbar aus dem Betrieb ausscheidet. Die Analysephase hat sie erstmals schriftlich festgehalten. Das ist unabhängig vom Sprachwechsel der belastbarste Teil des Business-Case — und der Grund, warum eine solche Migration bei ausscheidendem Know-how nicht beliebig aufschiebbar ist.
