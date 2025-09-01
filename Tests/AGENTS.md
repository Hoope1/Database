Die Datenbank wird als **SQLite-3** Datenbank umgesetzt und liegt als einzelne Datei im Projekt, damit sie mit Python und GitHub problemlos versioniert und geprüft werden kann.
Sie speichert **Aufgaben als abstrakte Vorlagen** in mathematischer Allgemeinform, zum Beispiel `"{a} + {b} - {c} ="`, ohne konkrete Zahlen.
Zu jeder Vorlage hält sie **Variablenbereiche** (etwa Zahlenintervalle, Mengen oder Konstanten) sowie **Regeln** wie Rundung, Nennergrenzen, Klammer- und Operatorvorgaben.
Für jede Vorlage sind außerdem **Metadaten** hinterlegt, darunter Schwierigkeitsgrad, erwartete Größenordnung des Ergebnisses und optionale Hinweise.
Alle **globalen Prüf- und Bewertungsvorgaben** des Tests, etwa Gesamtpunktzahl, Punktverteilung je Kategorie und Qualitätsgrenzen, werden zentral in einer Regel-Tabelle abgelegt.
**Bilder und andere Assets** werden als Dateien im Projekt gespeichert; die Datenbank verwaltet dazu nur den relativen Pfad, einen Inhalts-Hash und Maße sowie eine Zuordnung zur jeweiligen Vorlage.
Zur Nachvollziehbarkeit protokolliert sie die **Herkunft** der importierten Quellen und führt **Rohtext-Blöcke** aus den Testdokumenten, damit Vorlagen später eindeutig kuratiert werden können.
Insgesamt besteht das Schema aus wenigen klaren Tabellen wie „category“, „template“, „template\_var“, „template\_rule“, „rule\_global“, „asset“, „template\_asset“, „source\_block“, „provenance“ und optional „quality\_report“.
