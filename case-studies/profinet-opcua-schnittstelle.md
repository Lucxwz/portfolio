# Case Study: Kommunikationsschnittstelle zwischen zwei Steuerungswelten (PROFINET & OPC UA)

## Problem

In der industriellen Automatisierung treffen bei Anlagenintegrationen regelmäßig unterschiedliche Steuerungswelten aufeinander: ein Anlagenlieferant mit einer eigenen, herstellerspezifischen Steuerungsplattform auf der einen Seite, ein Maschinenbauer mit Siemens-Standardsteuerungen auf der anderen. Der Maschinenbauer erhält zwar eine Schnittstellenbeschreibung, hat aber oft keine praktische Erfahrung mit der Integration der fremden Steuerungswelt in sein Projekt. Das führte in der Praxis zu langen Inbetriebnahmezeiten und hohem Klärungsaufwand bei jedem neuen Projekt.

## Lösung

Ich habe ein Referenzprogramm im TIA-Portal entwickelt, das dem Maschinenbauer als direkte Vorlage dient: ein vollständiger Kommunikationsaufbau (Handshake) zwischen einer Siemens S7-1200 und der Fremdsteuerung — einmal über PROFINET, einmal über OPC UA. Der Kunde kann das Programm bzw. einzelne Teilbausteine nahezu unverändert übernehmen und muss im Kern nur noch projektspezifische Adressbereiche und Variablennamen anpassen.

Zentrale Design-Entscheidungen:

- **Bibliotheksfähigkeit**: Alle Bausteine sind bibliothekskonform aufgebaut (keine globalen Datenbausteinzugriffe innerhalb der Bausteine, Konstanten in der Schnittstelle deklariert, Multiinstanzen) und für S7-1200 und S7-1500 gleichermaßen einsetzbar.
- **PLC-Datentypen statt Einzelvariablen**: Für jeden Datenblock wurde ein eigener PLC-Datentyp angelegt, wodurch der Kunde nur Adressen und Namen, nicht aber die Struktur anpassen muss.
- **Zeitmessung des Handshakes**: Die Abarbeitungszeit wurde über Tracemessung im Cyclic Interrupt erfasst und dokumentiert.
- **Zweisprachigkeit intern**: Variablen, Kommentare und Bausteinnamen wurden komplett in Englisch gehalten, um internationale Nutzbarkeit zu gewährleisten.
- **Datenintegrität**: Ein eigener Baustein wandelt vom Feldbus-Gateway gelieferte Byte-Arrays verlustfrei in Strings um, ohne die sonst entstehenden Leerzeichen-Auffüllungen (ASCII „Space") mit zu übertragen.

## Technologien

TIA-Portal V18, Siemens S7-1215C, PROFINET (inkl. GSDML-Hardwarekonfiguration, DPRD_DAT/DPWR_DAT), OPC UA (Server-Konfiguration, Security-Profil Basic256Sha256 mit Signieren & Verschlüsseln, Benutzer-Authentifizierung), CODESYS V3.5, SCL/ST nach IEC 61131-3.

## Ergebnis

Beide Schnittstellenvarianten wurden erfolgreich in Betrieb genommen und funktional getestet:

- PROFINET-Handshake: gemessene Zykluszeit von **108,464 µs** (Messzeitraum 10 Minuten)
- OPC-UA-Handshake: gemessene Zykluszeit von **68,15 µs** (Messzeitraum 5 Minuten)

Bei der Inbetriebnahme wurden zwei reale Fehlerquellen identifiziert und behoben: eine fehlerhafte Speicherbereichszuordnung durch eine unvollständig konfigurierte GSDML-Datei sowie ein Timing-Problem bei einer Einschaltverzögerung innerhalb einer Case-Struktur.

Der direkte Vergleich beider Protokolle zeigte: PROFINET punktet bei Echtzeitfähigkeit, Geschwindigkeit und flexibler Topologie, bringt aber hohen Konfigurationsaufwand und keine eingebauten Sicherheitsmechanismen mit. OPC UA ist herstellerunabhängig, sicher und schnell zu konfigurieren, aber nicht echtzeitfähig. Fazit: Für Industrie-4.0-taugliche Architekturen sind beide Protokolle komplementär einzusetzen — PROFINET auf Feldebene, OPC UA zur Anbindung an übergeordnete Systeme (MES/ERP).
