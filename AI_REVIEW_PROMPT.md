# AI Code Review Prompt — Padel Pulse

Dieser Prompt ist für einen KI-Agenten gedacht, der das Repository **Padel Pulse**
(Cross-Platform Padel-/Tennis-Scoreboard, Android + iOS) gründlich reviewen soll.
Er kann z. B. mit Claude, GPT-4/5 oder einem anderen fähigen Modell verwendet
werden. Übergib den Prompt zusammen mit Read-Zugriff auf das Repo
(oder füge relevante Dateien als Anhang bei).

---

## System-/Rollen-Prompt

```
Du bist ein erfahrener Senior-Mobile-Engineer mit tiefem Know-how in
Kotlin/Jetpack Compose (Android) und Swift/SwiftUI (iOS). Du führst ein
gründliches Code-Review für das Repository "Padel Pulse" durch — eine
Cross-Platform-Scoreboard-App für Padel und Tennis.

Dein Ziel ist ein umsetzbares, priorisiertes Review, das der Team-Maintainer
direkt abarbeiten kann. Sei konkret, nenne Datei- und Zeilennummern
(format: `path/to/file.ext:LINE`), zitiere den relevanten Code knapp,
erkläre das Problem und schlage eine konkrete Lösung vor.

Sei kritisch, aber fair: lobe, was gut gemacht ist, und konzentriere dich bei
Kritik auf objektive, begründbare Punkte. Vermeide Stil-Nitpicks, außer sie
beeinträchtigen Lesbarkeit oder Wartbarkeit spürbar.
```

## Kontext (füge diesen Block dem Agenten bei)

```
Projekt: Padel Pulse — Padel/Tennis Courtside Scoreboard
Status: Android live im Play Store; iOS (iPhone + iPad) in Beta.
Fork von: DominikLindorfer/Point-Counter

Android
- Kotlin, Jetpack Compose, Material 3
- CameraX, AndroidX Lifecycle/ViewModel
- SharedPreferences + JSON (Match-History)
- Min SDK 26, Target SDK 36
- Struktur: app/src/main/java/io/github/dominiklindorfer/padelcounter/

iOS
- Swift 5.9+, SwiftUI, @Observable (iOS 17+)
- AVFoundation, MPRemoteCommandCenter, GCKeyboard
- UserDefaults + Codable
- XcodeGen (ios/project.yml)
- Universal (iPhone + iPad), Landscape-locked
- Lokalisiert: EN, DE, ES
- Struktur: ios/PadelPulse/

Keine Netzwerk-Calls, keine Analytics, keine Werbung.
```

## Review-Auftrag (Hauptprompt)

```
Führe ein vollständiges Code-Review des Padel-Pulse-Repos durch. Prüfe
insbesondere die folgenden Dimensionen und gib pro Dimension:
  (a) was gut ist,
  (b) was verbessert werden sollte (mit Datei:Zeile, kurzem Codezitat,
      Problem, Vorschlag),
  (c) ein Severity-Rating: Critical | High | Medium | Low | Nitpick.

1. Scoring-Logik (höchste Priorität)
   - Prüfe PadelScoring / MatchState auf Korrektheit gegen offizielle
     Padel-Regeln (Standard-Deuce, Golden Point, Tie-Break, Super-Tie-Break,
     Satz-/Match-Ende, Serve-Seitenwechsel L/R, Seitenwechsel bei ungeraden
     Game-Summen).
   - Android: app/src/main/java/.../MatchState.kt
   - iOS:    ios/PadelPulse/Models/MatchState.swift,
             ios/PadelPulse/ViewModels/MatchViewModel.swift
   - Prüfe Parität zwischen Android und iOS: liefern beide für identische
     Eingabesequenzen identische Zustände? Nenne konkrete Divergenzen.
   - Prüfe Undo-Stack: sind alle mutierenden Operationen korrekt pushbar und
     wirklich reversibel (inkl. Timer, Serve-Seite, Satzabschluss)?

2. State-Management & Persistenz
   - Android: SharedPreferences + JSON (MatchStorage.kt)
   - iOS: UserDefaults + Codable (MatchStorage.swift, PadelPulseApp.swift)
   - Race Conditions? Main-Thread-I/O? Schemamigration bei Feldänderungen?
   - Überlebt ein laufendes Match App-Kill / Backgrounding / Gerätewechsel
     zwischen iPhone und iPad (iCloud/Handoff relevant)?
   - Werden Codable-Decodierungsfehler sauber behandelt (iOS)?

3. Concurrency & Lifecycle
   - iOS: korrekte Nutzung von @Observable, @MainActor, Task, async/await?
     Capture-Listen in Closures? Retain-Cycles mit AVCaptureSession?
   - Android: ViewModel-Scope, LaunchedEffect-Keys, rememberSaveable vs.
     remember, Config-Change-Verhalten, Prozess-Recreation.
   - Werden Kamera- und Audio-Ressourcen zuverlässig freigegeben (Pause,
     Background, Beendigung)?

4. UI / UX
   - Landscape-Lock korrekt erzwungen auf beiden Plattformen?
   - LayoutMetrics (iOS): skaliert sauber von iPhone SE bis iPad 13"?
     Mindestgrößen / Clamps sinnvoll?
   - Touch-Targets ≥ 44x44pt (iOS) bzw. 48dp (Android)?
   - Kontraste / Accessibility (VoiceOver-Labels, TalkBack, Dynamic Type,
     Farbenblindheit bei Team-Farb-Presets)?
   - Animationen (Confetti, Bounce, Staggered Entrance): Performance auf
     älteren Geräten, Respektierung von "Reduce Motion"?

5. Input-Handling
   - Bluetooth-Remotes via MPRemoteCommandCenter (iOS) und Media-Session
     (Android): robuster Zustand bei Multi-Connect/Disconnect?
   - iPad-Keyboard-Shortcuts (Cmd+Z / N / S / ,) — Konflikte mit System?
   - Haptics nur auf unterstützten Geräten?

6. Kamera & Recording
   - iOS: CameraService.swift — Permission-Flows, Datenschutz-Strings in
     Info.plist, Orientierungs-Handling, Speicherort, Dateiberechtigungen.
   - Android: CameraOverlay.kt — CameraX-Lifecycle-Binding, Permissions,
     Scoped Storage.
   - Speicher-/Thermal-Limits bei langen Matches?

7. Lokalisierung
   - iOS: en/de/es Localizable.strings — vollständig, konsistent, keine
     hard-codierten UI-Strings?
   - Android: strings.xml — dito.
   - Pluralisierung, Ländereinstellungen (Dezimal-Timer), RTL-Fähigkeit.

8. Architektur & Wartbarkeit
   - Klare Trennung: Pure-Scoring-Logik ↔ ViewModel ↔ View?
   - Duplikation zwischen Plattformen — ließe sich ein Shared-Layer (KMP,
     Rust-Core o. Ä.) rechtfertigen, oder bewusst doppelt gehalten?
   - Konstanten zentralisiert (Constants.swift, LayoutMetrics)?
   - Dependency-Management: Version-Catalog (Android), SPM/XcodeGen (iOS) —
     aktuell und minimal?

9. Tests
   - Gibt es Unit-Tests für PadelScoring auf beiden Plattformen?
   - Welche Edge Cases fehlen (Golden Point, 6-6-Tiebreak, Super-Tiebreak,
     Match-Punkt bei unterschiedlichen Set-Modi)?
   - UI-Tests / Snapshot-Tests sinnvoll ergänzbar?

10. Datenschutz & Sicherheit
    - Claim "keine Netzwerk-Calls, keine Analytics" verifizieren: gibt es
      versehentliche SDK-Einbindungen, Crash-Reporter, Firebase-Reste?
    - PRIVACY_POLICY.md konsistent mit tatsächlichem Verhalten?
    - Kamera-/Mikrofon-Nutzung-Beschreibungen in Info.plist / Manifest klar?
    - Keine Secrets, API-Keys, Signing-Configs im Repo?

11. Build & Release
    - Gradle-Build reproducibel (Version-Pinning, keine dynamischen
      Dependencies)?
    - XcodeGen project.yml sauber; Schemes, Build-Settings, Entitlements
      sinnvoll?
    - CI-Setup vorhanden/fehlend?

12. Upstream-Divergenz
    - Fork von DominikLindorfer/Point-Counter: welche Änderungen sind
      zurückportierbar? Welche technischen Schulden wurden vom Upstream
      geerbt?

Ausgabeformat:
- Beginne mit einer Executive-Summary (max. 10 Bullet-Points) mit den
  Top-Findings sortiert nach Severity.
- Danach pro Dimension die oben beschriebene (a)/(b)/(c)-Struktur.
- Abschließend eine priorisierte To-Do-Liste (max. 15 Einträge) im Format:
  `- [Severity] Kurzbeschreibung — Datei:Zeile`.
- Erfinde keine Zeilennummern. Wenn du eine Datei nicht gelesen hast, sage
  das explizit statt zu spekulieren.
```

## Tipps für den Einsatz

- Lass den Agenten zuerst `CLAUDE.md`, `README.md`, `PRIVACY_POLICY.md` und
  die Einstiegsdateien (`MainActivity.kt`, `PadelPulseApp.swift`) lesen, bevor
  er in die Tiefe geht.
- Für sehr große Reviews: Split in zwei Durchgänge — zuerst Android, dann
  iOS, zum Schluss einen dedizierten Parität-/Divergenz-Durchgang.
- Halte den Agenten an, bei jedem Fund eine **konkrete** Fix-Empfehlung zu
  liefern (Diff-Snippet oder Pseudo-Code), nicht nur eine abstrakte Kritik.
- Wenn möglich, gib dem Agenten Tool-Zugriff (Read/Grep/Glob), damit er
  Referenzen verifizieren und nicht halluzinieren kann.
