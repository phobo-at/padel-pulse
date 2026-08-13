# Padel Pulse — vollständiger App-Review

**Review-Datum:** 22. Juli 2026  
**Review-Stand:** Commit `646d1940af6057fee07b2c2201d3790ce03c9b1c`  
**Umfang:** Android-App, iOS-App, Scoring, Persistenz, Kamera, Remote-Eingaben, Datenschutz, Accessibility, Dokumentation sowie Build- und Test-Setup

## Executive Summary

Die App hat eine klare, lokal-first Architektur, eine sehr gut lesbare Courtside-Oberfläche und auf iOS bereits eine beachtliche Testbasis. Im geprüften Stand gibt es jedoch **sieben P1-Findings**, die vor einem iOS-Release beziehungsweise im nächsten Android-Release behoben werden sollten. Besonders kritisch sind veränderbare Matchregeln: Sowohl Android als auch iOS können dadurch während eines laufenden Spiels einen Punkt oder sogar den Matchsieg dem falschen Team zuweisen. Zusätzlich ist ein Undo nach dem Matchball nicht mit der bereits gespeicherten History gekoppelt.

Die öffentliche Privacy-Aussage „keine Daten an Server“ stimmt bei aktivem Betriebssystem-Backup nicht: Android nimmt die SharedPreferences mit der Match-History standardmäßig in Auto Backup auf; iOS kann App-Daten in das Geräte-/iCloud-Backup aufnehmen. Auch der iOS-MIT-Badge widerspricht der tatsächlich eingecheckten Source-Available-Lizenz.

**Release-Empfehlung:** iOS weiterhin als Beta behandeln und F-01, F-02, F-03, F-05 sowie F-06 vor einer Veröffentlichung schließen. Für Android sollten F-01 bis F-05 und F-07 in den nächsten Hotfix-/Minor-Release aufgenommen werden.

## Prioritätsskala

- **P0 – kritisch:** akuter Crash, Sicherheitsvorfall oder praktisch unvermeidbarer schwerer Datenverlust
- **P1 – hoch:** falsches Matchergebnis, widersprüchliche Daten, rechtlich/öffentlich relevante Falschaussage oder zentraler Workflow defekt; Release-Blocker
- **P2 – mittel:** relevanter Zuverlässigkeits-, UX-, Accessibility- oder Wartbarkeitsmangel
- **P3 – niedrig:** Dokumentationsdrift, Politur oder langfristige Härtung

## Finding-Übersicht

| ID | Prio | Plattform | Bereich |
|---|---|---|---|
| F-01 | P1 | Android + iOS | Golden-Point-Moduswechsel verfälscht Game-Sieger |
| F-02 | P1 | Android + iOS | Geändertes Satz-Ziel verfälscht Matchsieger |
| F-03 | P1 | Android + iOS | Matchball-Undo und History sind inkonsistent |
| F-04 | P1 | Android | Falscher Aufschläger im Tiebreak |
| F-05 | P1 | Android + iOS | Privacy-/Backup-Aussagen widersprechen Plattformverhalten |
| F-06 | P1 | iOS | MIT-Badge widerspricht Source-Available-Lizenz |
| F-07 | P1 | Android | Laufender Match übersteht Prozessende nicht |
| F-08 | P2 | Android | Timerzustand bei Undo/Reset inkonsistent |
| F-09 | P2 | Android | Stiller History-Verlust bei Decode-Fehler |
| F-10 | P2 | iOS | Share-Dauer wächst nach Matchende weiter |
| F-11 | P2 | Android + iOS | Share-Funktionen fehlen/abweichend von Dokumentation |
| F-12 | P2 | Android + iOS | Destruktive Pfade ohne einheitliche Bestätigung |
| F-13 | P2 | Android + iOS | Kamera-Permissions und Fehlerzustände unvollständig |
| F-14 | P2 | Repository | Keine wirksamen Android-Tests/CI-Gates |
| F-15 | P2 | Android | Small-Screen-/TalkBack-Härtung fehlt |
| F-16 | P3 | Repository | Dokumentations- und Versionsdrift |

## Findings

### F-01 — P1 — Ein Moduswechsel bei Advantage schreibt den Punkt dem falschen Team gut (Android + iOS)

**Fundstellen:**

- `app/src/main/java/io/github/dominiklindorfer/padelcounter/MatchState.kt:79-83`
- `app/src/main/java/io/github/dominiklindorfer/padelcounter/MainActivity.kt:731-753`
- `ios/PadelPulse/Models/MatchState.swift:74-78`
- `ios/PadelPulse/Views/SettingsSidebarView.swift:122-140`

**Reproduktion:**

1. Im Advantage-Modus bis 40:40 spielen.
2. Team 1 gewinnt den nächsten Punkt; interner Stand ist 4:3.
3. In den Settings auf Golden Point wechseln.
4. Team 2 gewinnt den nächsten Punkt.

**Ist:** Die Golden-Point-Prüfung sieht zuerst `team1Points >= 4`, erklärt Team 1 zum Game-Sieger und schreibt 1:0 Spiele. Der ausführbare Review-Harness lieferte `mode-switch games=1-0`.

**Auswirkung:** Ein real gewonnener Punkt für Team 2 wird als gewonnenes Spiel für Team 1 verbucht. Beide Plattformen enthalten dieselbe Logik.

**Empfehlung:** Matchregeln nach dem ersten Punkt unveränderlich machen oder einen Regelwechsel nur über einen bestätigten Neustart erlauben. Zusätzlich einen Regressionstest für den Übergang aus 4:3/3:4 ergänzen. Die Scoring-Engine sollte außerdem den Gewinner aus dem gerade punktenden Team und nicht nur aus absoluten Schwellen ableiten.

### F-02 — P1 — Das Absenken von „Sets to Win“ kann den falschen Matchsieger setzen (Android + iOS)

**Fundstellen:**

- `app/src/main/java/io/github/dominiklindorfer/padelcounter/MatchState.kt:129-139`
- `app/src/main/java/io/github/dominiklindorfer/padelcounter/MainActivity.kt:757-777`
- `ios/PadelPulse/Models/MatchState.swift:124-136`
- `ios/PadelPulse/Views/SettingsSidebarView.swift:133-140`

**Reproduktion:**

1. Mit `setsToWin = 2` spielen; Team 1 gewinnt Satz 1.
2. In Satz 2 auf `setsToWin = 1` reduzieren.
3. Team 2 gewinnt Satz 2.

**Ist:** Beide Teams stehen 1:1, aber die Gewinnerprüfung testet Team 1 zuerst und setzt `winner = 1`. Der Review-Harness bestätigte `target-switch sets=1-1 winner=1`.

**Auswirkung:** Der Matchsieg kann dem Team zugesprochen werden, das den entscheidenden Satz nicht gewonnen hat.

**Empfehlung:** `setsToWin` nach Matchbeginn sperren. Zusätzlich die Matchgewinner-Ermittlung an den aktuellen Satzgewinner koppeln und ungültige Konfigurationen durch Invarianten/Assertions abfangen.

### F-03 — P1 — Undo nach dem Matchball rollt den gespeicherten History-Eintrag nicht zurück (Android + iOS)

**Fundstellen:**

- Android speichert unmittelbar bei Matchende: `app/src/main/java/io/github/dominiklindorfer/padelcounter/MatchState.kt:236-239`; Undo restauriert nur Laufzeitstatus: `:257-266`
- iOS speichert unmittelbar bei Matchende: `ios/PadelPulse/ViewModels/MatchViewModel.swift:183-188`; Undo restauriert nur den Snapshot: `:235-246`
- Der vorhandene iOS-Test prüft nur den sichtbaren Zustand, nicht die History: `ios/PadelPulseTests/MatchViewModelTests.swift:236-260`

**Reproduktion:** Match abschließen, anschließend per Bluetooth-Remote/Hardware-Shortcut Undo auslösen und den Matchball erneut spielen.

**Ist:** Der erste abgeschlossene Match bleibt in der History. Beim erneuten Matchball wird ein zweiter Eintrag gespeichert. Android startet nach diesem Undo zusätzlich den Timer neu, da `matchRunning` beim Undo nicht restauriert wird.

**Auswirkung:** Phantom- beziehungsweise Duplikat-Matches und falsche Dauer-/Statistikwerte.

**Empfehlung:** Den beim Abschluss erzeugten `SavedMatch` samt ID im Undo-Snapshot referenzieren und bei Undo transaktional entfernen/ersetzen. Alternativ den Match erst nach expliziter Bestätigung finalisieren. Regressionstests müssen History-Anzahl und gespeicherte Werte vor/nach Matchball-Undo prüfen.

### F-04 — P1 — Android rotiert den Aufschläger im Tiebreak nicht

**Fundstellen:**

- Android wechselt `servingTeam` ausschließlich beim Game-Wechsel: `app/src/main/java/io/github/dominiklindorfer/padelcounter/MatchState.kt:226-235`
- iOS enthält die notwendige Tiebreak-Sonderregel: `ios/PadelPulse/ViewModels/MatchViewModel.swift:151-179`

**Ist:** Im Android-Tiebreak bleibt derselbe Aufschläger bis zum Ende des Tiebreaks angezeigt. Regelkonform wäre ein Wechsel nach Punkt 1 und danach nach jeweils zwei Punkten.

**Auswirkung:** Die zentrale Courtside-Anzeige weist den falschen Aufschläger aus.

**Empfehlung:** Die getestete iOS-Logik nach Android portieren und plattformgleiche Tests für die Sequenz `1,2,2,1,1,2,2,1` ergänzen.

### F-05 — P1 — Privacy Policy widerspricht Speicherung und Betriebssystem-Backups

**Fundstellen:**

- Policy: `PRIVACY_POLICY.md:7-16`
- Android aktiviert Backup: `app/src/main/AndroidManifest.xml:11-13`
- Android-Regeldateien enthalten keine Excludes: `app/src/main/res/xml/backup_rules.xml:8-13`, `app/src/main/res/xml/data_extraction_rules.xml:6-12`
- Matchdaten liegen in SharedPreferences: `app/src/main/java/io/github/dominiklindorfer/padelcounter/MatchStorage.kt:70-71`
- iOS speichert History und laufende Matches in UserDefaults: `ios/PadelPulse/Storage/MatchStorage.swift:5-6`, `ios/PadelPulse/ViewModels/MatchViewModel.swift:330-331`
- Android fordert Mikrofonzugriff an, die Android-Permissions-Sektion nennt ihn nicht: `app/src/main/AndroidManifest.xml:7`, `PRIVACY_POLICY.md:20-23`

**Ist:** Die Policy sagt gleichzeitig, dass keine persönlichen Daten gespeichert werden, Match-History aber lokal gespeichert wird. Teamnamen können personenbezogen sein. Sie sagt außerdem „No data is sent to any server“ und „removed when the app is uninstalled“. Android Auto Backup umfasst standardmäßig SharedPreferences und kann sie in Google Drive sichern beziehungsweise nach einer Neuinstallation wiederherstellen. iCloud Backup umfasst standardmäßig App-Daten, sofern diese nicht explizit ausgeschlossen sind.

**Offizielle Referenzen:**

- Android: <https://developer.android.com/identity/data/autobackup>
- Apple: <https://developer.apple.com/documentation/foundation/optimizing-your-app-s-data-for-icloud-backup>

**Auswirkung:** Öffentliches Datenschutzversprechen und reales Plattformverhalten stimmen nicht überein.

**Empfehlung:** Entweder Backups für Matchdaten explizit ausschließen oder die Policy präzise auf „kein eigenes Backend/keine Analytics; lokale Daten können Teil des verschlüsselten OS-Backups sein“ umstellen. Lokal gespeicherte Datenarten, Löschwege, Restore-Verhalten und Android-Mikrofonberechtigung vollständig nennen. Vor Veröffentlichung rechtlich prüfen lassen.

### F-06 — P1 — iOS wird als MIT beworben, ist aber Source-Available ohne Verteilungsrecht

**Fundstellen:**

- MIT-Badge: `ios/README.md:3-8`
- Tatsächliche Einschränkungen: `ios/LICENSE:7-25`
- Root-README beschreibt die Lizenz korrekt: `README.md:135-138`

**Ist:** Der prominente Badge signalisiert MIT, während `ios/LICENSE` nur private, nicht-kommerzielle Nutzung erlaubt und Weiterverteilung verbietet.

**Auswirkung:** Nutzer und Contributors können sich auf eine falsche Lizenzinformation verlassen.

**Empfehlung:** Badge sofort auf „Source Available / Personal Use“ ändern und alle weiteren Metadaten (Repository-Beschreibung, App-Listing, Credits) auf Konsistenz prüfen.

### F-07 — P1 — Android verliert laufende Matches bei Prozessende

**Fundstellen:**

- Matchzustand liegt nur im ViewModel: `app/src/main/java/io/github/dominiklindorfer/padelcounter/MatchState.kt:160-212`
- Persistiert wird nur abgeschlossene History: `app/src/main/java/io/github/dominiklindorfer/padelcounter/MatchStorage.kt:66-105`
- Die Policy behauptet In-Progress-Persistenz auch für Android: `PRIVACY_POLICY.md:16`

**Ist:** Activity-Recreation kann das ViewModel überstehen, ein vom Betriebssystem beendeter Prozess aber nicht. Es gibt weder `SavedStateHandle` noch einen persistierten Match-Snapshot.

**Auswirkung:** Ein laufendes Match kann nach Speicherdruck, App-Kill oder Geräteproblem vollständig verloren gehen.

**Empfehlung:** Den vollständigen Android-Matchzustand inklusive Undo-, Timer- und Serverstatus versioniert persistieren; bei jedem relevanten Zustandswechsel atomar schreiben und Restore-/Migrationsfälle testen.

### F-08 — P2 — Android-Timer ist bei Undo und Reset nicht an den Matchzustand gekoppelt

**Fundstellen:**

- Undo restauriert weder `matchStartTimeMs` noch `matchRunning`: `app/src/main/java/io/github/dominiklindorfer/padelcounter/MatchState.kt:257-266`
- Reset setzt zwar das ViewModel zurück: `:269-279`; der lokale Compose-Wert `elapsed` bleibt aber erhalten: `app/src/main/java/io/github/dominiklindorfer/padelcounter/MainActivity.kt:598-630`

**Ist:** Undo des ersten Punktes lässt den Timer weiterlaufen. Reset während eines laufenden Matches zeigt auf dem frischen Scoreboard weiterhin die alte Dauer, bis ein neuer Punkt fällt. Nach Matchball-Undo beginnt der Timer beim nächsten Punkt von vorn.

**Empfehlung:** Wie auf iOS einen vollständigen Undo-Snapshot inklusive Timerstatus verwenden und die angezeigte Dauer aus einem einzigen beobachtbaren ViewModel-State ableiten.

### F-09 — P2 — Android verwirft eine beschädigte History still und überschreibt sie beim nächsten Save

**Fundstellen:** `app/src/main/java/io/github/dominiklindorfer/padelcounter/MatchStorage.kt:83-105`

**Ist:** Ein Decode-Fehler in nur einem Datensatz führt zu `emptyList()`. Die defekten Rohdaten bleiben zunächst liegen; beim nächsten abgeschlossenen Match wird diese leere Liste geladen und die gesamte bisherige History überschrieben. Es gibt keine Schema-Version oder Migration. iOS quarantänisiert beschädigte Bytes dagegen explizit.

**Auswirkung:** Vollständiger, stiller Verlust der Match-History nach partieller Korruption oder inkompatibler Schemaänderung.

**Empfehlung:** Versioniertes Format, atomare Writes, Quarantäne/Backup der Rohdaten und möglichst datensatzweises Recovery ergänzen. Fehler sichtbar protokollieren, ohne Inhalte oder Teamnamen in Produktionslogs auszugeben.

### F-10 — P2 — iOS-Share-Dauer wächst nach Matchende weiter

**Fundstellen:**

- Share erstellt beim Buttondruck einen neuen Snapshot: `ios/PadelPulse/Views/MatchOverOverlayView.swift:91-96`
- Die Dauer wird jedes Mal als `now - matchStartTimeMs` berechnet: `ios/PadelPulse/ViewModels/MatchViewModel.swift:264-277`

**Ist:** Das automatisch gespeicherte History-Match erhält die korrekte Dauer am Matchende. Wartet der Nutzer danach auf dem Match-Over-Screen und teilt erst später, enthält die Share-Karte zusätzlich die Wartezeit.

**Auswirkung:** Geteiltes Ergebnis und History zeigen unterschiedliche Matchdauern.

**Empfehlung:** Finaldauer beim Matchball einfrieren und für Save sowie Share denselben finalisierten Snapshot verwenden.

### F-11 — P2 — Beworbene Share-Funktionen fehlen beziehungsweise sind plattforminkonsistent

**Fundstellen:**

- iOS-README behauptet History-Share als Text und Bild: `ios/README.md:116`
- `SavedMatch.shareText()` ist implementiert, aber ohne Aufrufer: `ios/PadelPulse/Models/SavedMatch.swift:44-87`
- iOS-History bietet nur Löschen, keinen Share-Button: `ios/PadelPulse/Views/MatchHistoryView.swift:95-228`
- Android-History teilt nur `text/plain`: `app/src/main/java/io/github/dominiklindorfer/padelcounter/MatchHistoryScreen.kt:255-292`

**Ist:** iOS kann nur direkt am Matchende ein Bild teilen; vergangene Matches lassen sich aus der History nicht teilen. Android bietet in der History Text, aber kein Bild.

**Empfehlung:** Erwartetes Produktverhalten festlegen und Dokumentation oder Implementierung angleichen. Für iOS den vorhandenen Text-Builder und Image-Renderer in die History-Actions integrieren.

### F-12 — P2 — Destruktive Aktionen umgehen Bestätigungen

**Fundstellen:**

- Android „New Match“ resetet sofort: `app/src/main/java/io/github/dominiklindorfer/padelcounter/MainActivity.kt:352-363`
- Android „Clear all“ löscht sofort: `app/src/main/java/io/github/dominiklindorfer/padelcounter/MatchHistoryScreen.kt:82-85`
- iOS-UI bestätigt „New Match“, der Cmd+N-Befehl resetet jedoch direkt: `ios/PadelPulse/Views/ScoreBoardView.swift:138-147`, `ios/PadelPulse/App/PadelPulseApp.swift:49-52`

**Auswirkung:** Laufender Spielstand oder gesamte History können durch einen Fehl-Tap beziehungsweise Shortcut ohne Rückweg verloren gehen.

**Empfehlung:** Einheitliche Confirmations für laufende Matches und „Delete all“ verwenden; Einzel-Löschen optional über Undo/Snackbar absichern. Alle Eingabepfade müssen dieselbe Domain-Action aufrufen.

### F-13 — P2 — Kamera-Berechtigungen und Fehlerzustände sind nicht robust geführt

**Fundstellen:**

- Android fordert Kamera, Mikrofon und auf alten Geräten Storage gemeinsam an und öffnet die Preview nur, wenn alles gewährt wurde: `app/src/main/java/io/github/dominiklindorfer/padelcounter/MainActivity.kt:211-227`
- Android erzwingt Audioaufnahme: `app/src/main/java/io/github/dominiklindorfer/padelcounter/CameraOverlay.kt:249-252`
- iOS verwirft Fehler beim Erzeugen von Kamera-/Mikrofoneingängen mit `try?`: `ios/PadelPulse/Services/CameraService.swift:32-45`

**Ist:** Android kann bei verweigertem Mikrofon nicht einmal eine stumme Kamera-/Videofunktion anbieten und gibt nach Denial kein erklärendes UI-Feedback. iOS kann bei verweigertem Zugriff eine leere Preview mit weiterhin sichtbaren Controls zeigen; ein expliziter Authorization-State fehlt.

**Empfehlung:** Kamera-, Mikrofon- und Photos-Rechte getrennt und just-in-time behandeln; stummes Video als Fallback anbieten; denied/restricted dauerhaft sichtbar erklären und einen Link zu den Systemeinstellungen anbieten. Fehlerzustände in Service und UI explizit modellieren.

### F-14 — P2 — Android hat praktisch keine Tests und das Repository keinen CI-Gate

**Fundstellen:**

- Android-Unit-Test prüft nur `2 + 2`: `app/src/test/java/io/github/dominiklindorfer/padelcounter/ExampleUnitTest.kt:12-16`
- Instrumentation-Test prüft nur den Package-Namen: `app/src/androidTest/java/io/github/dominiklindorfer/padelcounter/ExampleInstrumentedTest.kt:17-23`
- Keine CI-Workflow-Dateien im Repository
- `gradlew` ist als Modus `100644` eingecheckt und daher unter Unix/macOS nicht direkt ausführbar

**Auswirkung:** Die Android-Varianten von F-01, F-02, F-04, F-07 und F-08 werden nicht automatisch erkannt. Der dokumentierte Aufruf `./gradlew ...` scheitert bereits an den Dateirechten.

**Empfehlung:** iOS-Scoringtests plattformgleich auf Android portieren, ViewModel-/Storage-Tests ergänzen, Wrapper ausführbar einchecken und CI für Android `test`, `lint`, `assemble` sowie iOS Build/Tests einrichten.

### F-15 — P2 — Android-Small-Screen- und Accessibility-Härtung ist unvollständig

**Fundstellen:**

- Skalierung sinkt bis 0,4: `app/src/main/java/io/github/dominiklindorfer/padelcounter/MainActivity.kt:207-210`
- Das globale Material-Mindestmaß wird auf 0 gesetzt: `:231`
- Schrift- und Icongrößen der Toolbar werden mit demselben Faktor verkleinert: `:280-374`
- Teamflächen haben keine explizite Team-/Aktions-Semantik: `:917-924`

**Auswirkung:** Auf kleineren Landscape-Geräten werden Labels und Icons sehr klein; Touch- und TalkBack-Bedienung ist nicht mit derselben Sorgfalt abgesichert wie auf iOS.

**Empfehlung:** Mindest-Touchziele von 48 dp beibehalten, Typografie separat klemmen und für Teamflächen `Role.Button`, aussagekräftige Labels, Value und Hint definieren. TalkBack, große Schrift und mindestens ein kleines Phone sowie ein Tablet in UI-Tests aufnehmen.

### F-16 — P3 — Dokumentation und tatsächliches Verhalten sind mehrfach auseinander gelaufen

**Fundstellen und Beispiele:**

- Root-Buildanleitung klont das Upstream-Repository statt dieses Repositories: `README.md:103-110`
- API 36 wird dort als Android 15 bezeichnet: `README.md:110`
- `CLAUDE.md` nennt iOS `1.0.0-beta`, das Projekt setzt `MARKETING_VERSION = 1.0.0`: `CLAUDE.md:48`, `ios/project.yml:34`
- iOS-README verspricht neue Zufallsnamen „on each new match“: `ios/README.md:76`; `resetMatch()` erzeugt keine neuen Namen: `ios/PadelPulse/ViewModels/MatchViewModel.swift:248-259`
- README beschreibt History-Share, das UI enthält ihn nicht (siehe F-11)

**Empfehlung:** Eine einzige Release-/Feature-Quelle definieren und README, Privacy, Lizenzbadge sowie Store-Metadaten daraus bei CI validieren.

## Positive Befunde

- Die Scoring-Engine ist auf beiden Plattformen weitgehend pur und strukturell gut testbar.
- Die iOS-Tests decken Standard-, Golden-Point-, Tiebreak-, Timer-, Undo-, Persistenz- und Lokalisierungsfälle bereits substanziell ab.
- iOS quarantänisiert beschädigte Persistenzdaten, statt sie kommentarlos zu überschreiben.
- Kamera-Sessions und temporäre iOS-Videos werden grundsätzlich über Lifecycle-/Cleanup-Pfade beendet beziehungsweise entfernt.
- Im Quellcode wurden keine eigenen Netzwerkaufrufe, Analytics-, Ads- oder Tracking-SDKs gefunden; Android deklariert keine `INTERNET`-Permission.
- iOS hat gute Accessibility-Grundlagen: 44-pt-Toolbar-Ziele, VoiceOver-Labels, Reduce-Motion-Behandlung und adaptive Layoutmetriken.
- Die geprüften iOS-Referenzscreens zeigen eine klare visuelle Hierarchie, sehr große Courtside-Scores und redundante Aufschlag-Cues über Rahmen, Racket und L/R-Glyph.
- Lokalisierungsdateien für EN/DE/ES sind syntaktisch valide und besitzen einen Parity-Test.

## Ausgeführte Checks

| Check | Ergebnis | Einordnung |
|---|---|---|
| Arbeitsbaum vor Review | sauber | keine vorhandenen Nutzeränderungen überlagert |
| Statische Prüfung aller Kotlin-/Swift-Produktionsdateien | abgeschlossen | ca. 6.772 Zeilen Produkt- und Testcode erfasst |
| Scoring-Reproduktions-Harness | ausgeführt | beide P1-Regelfehler bestätigt |
| `xcodegen` aus `ios/project.yml` in `/tmp` | erfolgreich | Projektspezifikation valide |
| `Info.plist` und drei `Localizable.strings` | valide | `plutil -lint` erfolgreich |
| Asset-Catalog-JSON | valide | `jq empty` erfolgreich |
| `ios/scripts/deploy.sh` Syntax | valide | `bash -n` erfolgreich |
| Android `./gradlew test lint assembleDebug` | nicht gestartet | Wrapper ist nicht ausführbar |
| Android via `bash gradlew ...` | blockiert | auf dem Review-Rechner ist keine Java Runtime installiert |
| iOS Device-/Simulator-Build | umgebungsbedingt fehlgeschlagen | CoreSimulator `1051.50.0` ist älter als die von Xcode 26.6 erwartete Build-Version `1051.55.0`; Asset-Kompilierung bricht vor vollständigem App-Build ab |
| iOS Tests | nicht ausgeführt | derselbe CoreSimulator-/Xcode-Mismatch |
| Live-Kamera, Bluetooth-Remote, TalkBack/VoiceOver | nicht auf Hardware getestet | benötigt reale Geräte und passende Berechtigungszustände |

Der fehlgeschlagene iOS-Build ist in diesem Lauf kein belegter App-Codefehler; die lokale Xcode-/CoreSimulator-Installation ist inkonsistent. Deshalb ist der Review für Runtime-, Kamera- und Accessibility-Verhalten als statischer Review plus Referenzbildprüfung zu verstehen.

## Empfohlene Umsetzungsreihenfolge

1. **Scoring-Invarianten sichern:** Regeln nach erstem Punkt sperren, F-01/F-02 fixen und identische Regressionstests auf beiden Plattformen hinzufügen.
2. **Matchabschluss transaktional machen:** Finaldauer und Saved-Match-ID erfassen; Undo, erneuter Matchball und Share aus einem finalisierten Snapshot bedienen.
3. **Android-Parität herstellen:** Tiebreak-Aufschlag, vollständige Undo-/Timer-Snapshots und In-Progress-Persistenz portieren.
4. **Privacy und Lizenz korrigieren:** Backup-Entscheidung treffen, Policy aktualisieren, MIT-Badge entfernen.
5. **Datenhaltbarkeit härten:** versionierte/atomare Android-Persistenz mit Quarantäne und Migration.
6. **Release-Gates einführen:** Wrapper-Rechte, CI, Android-Tests, iOS-Toolchain reparieren und beide Apps auf realer Hardware smoke-testen.
7. **UX/Accessibility abrunden:** Confirmation-Flows vereinheitlichen, Camera-Denials modellieren, Android Small-Screen/TalkBack testen und Share-Parität herstellen.

## Abnahmekriterien für den nächsten Review

- Alle P1-Reproduktionen sind durch neue, auf Android und iOS grüne Tests abgedeckt.
- Matchball → Undo → Matchball erzeugt exakt einen History-Eintrag mit stabiler Dauer.
- Android zeigt im Tiebreak die korrekte Aufschlagsequenz und restauriert einen laufenden Match nach Prozessneustart.
- Privacy Policy entspricht nachweislich den gewählten Android-/iOS-Backupregeln und allen Runtime-Permissions.
- Lizenzhinweise sind in Root-README, iOS-README und Distribution konsistent.
- Android `test lint assembleDebug` sowie iOS Build und Tests laufen in CI reproduzierbar grün.
- Kamera-Denial, Bluetooth-Remote, TalkBack/VoiceOver, Reduce Motion und kleine Landscape-Geräte sind auf realer Hardware geprüft.
