# ci-templates

Gemeinsame, wiederverwendbare GitHub-Actions-Workflows für alle Flutter-Apps von Nicolas (Android, Windows). Jede App ruft sie mit wenigen Zeilen auf und gibt nur die Unterschiede mit. So laufen CI, Test-APK und Release in allen Repos gleich.

> **Stand:** Die Workflows sind mit actionlint geprüft und das Signatur-Skript ist lokal getestet. Ein echter Lauf in einer App steht noch aus (erster Test: Focus Flow). Bis dahin rufen die Apps die Vorlagen über einen Branch oder `@main` auf. Danach kommt das bewegliche Tag `v1`.

## Die drei Stufen

Tests laufen bei jedem Push. Ein voller Build läuft nur, wenn er etwas findet, das Analyse und Tests nicht sehen.

| Stufe | Workflow | Wann | Ergebnis |
| --- | --- | --- | --- |
| 1. CI | [`flutter-ci.yml`](.github/workflows/flutter-ci.yml) | Jeder Push und jeder Pull Request | Formatierung, Analyse, Tests. Ziel: höchstens 5 Minuten. Pflicht-Check für den Merge. |
| 2. Build-Check | [`android-build-check.yml`](.github/workflows/android-build-check.yml) | Nur bei Änderungen an `pubspec.yaml`, `android/**` oder den Workflows, dazu einmal pro Woche | Unsigniertes Release-APK (arm64), nichts wird veröffentlicht. Findet Gradle-, R8- und Abhängigkeitsfehler früh. |
| 3a. Test-APK | [`android-test-apk.yml`](.github/workflows/android-test-apk.yml) | Jeder Merge auf main, auf Knopfdruck auch aus anderen Branches | Signiertes APK nur für arm64. Auf main zusätzlich auf dem Pre-Release `latest`. |
| 3b. Android-Release | [`android-release.yml`](.github/workflows/android-release.yml) | Von Hand oder per Tag `v*` | Signiertes App Bundle für Play plus APK. Tests laufen vorher. |
| 3c. Windows-Release | [`windows-release.yml`](.github/workflows/windows-release.yml) | Von Hand oder per Tag `v*` | ZIP und MSIX für den Microsoft Store. |

Aufrufbeispiele zum Kopieren liegen in [`examples/`](examples/).

## Verwendung in einer App

```yaml
jobs:
  ci:
    uses: nicolasasauer/ci-templates/.github/workflows/flutter-ci.yml@main
    with:
      apt-packages: libsqlite3-dev
```

Auslöser, `concurrency` und Berechtigungen legt immer die aufrufende Datei fest. Die Vorlagen enthalten sie bewusst nicht.

### Eingaben

Alle Workflows kennen `working-directory` (Standard `.`, im Monorepo der App-Ordner) und `flutter-version` (Standard `3.44.0`).

| Eingabe | Workflows | Bedeutung |
| --- | --- | --- |
| `app-name` (Pflicht) | Test-APK, Releases | Dateiname ohne Endung, zum Beispiel `focus-flow` |
| `apt-packages` | CI, Android | Zusätzliche Linux-Pakete, zum Beispiel `libsqlite3-dev` |
| `pre-test-command`, `pre-build-command` | CI, Android | Befehl vor Analyse oder Build, zum Beispiel `dart run build_runner build` |
| `java-version` | Android | Standard `17` |
| `gradle-jvmargs` | Android | Heap für den Gradle-Daemon, Standard `-Xmx3G …` |
| `build-args` | Android, Windows | Zusätzliche Argumente, zum Beispiel `--dart-define=NAME=wert` |
| `check-format`, `test-args` | CI | `dart format` prüfen, Argumente für `flutter test` |
| `run-tests` | Test-APK, Windows | Tests vor dem Build. Die CI hat sie meist schon geprüft. |
| `publish`, `release-tag` | Test-APK | Auf dem Pre-Release veröffentlichen (nur auf main) |
| `all-abis` | Android-Release | Auch armeabi-v7a und x86_64 bauen. Meist unnötig. |
| `github-release-tag` | Releases | Tag, an dem ein GitHub-Release hängt. Leer erzeugt nur ein Artefakt. |
| `build-msix`, `msix-version-prefix` | Windows | Store-Paket bauen, Version `<Präfix>.<Commit-Anzahl>.0` |

## Signierung

Alle Apps nutzen dieselben drei Repo-Secrets:

- `KEYSTORE_BASE64` (Keystore, base64-kodiert)
- `KEYSTORE_PASSWORD` (wird auch als Schlüsselpasswort verwendet)
- `KEY_ALIAS`

Die Vorlagen stellen den Schlüssel auf **beide** Arten bereit, die in den Apps vorkommen, damit keine App ihr Gradle-Skript ändern muss:

1. Umgebungsvariablen `ANDROID_KEYSTORE_PATH`, `ANDROID_KEYSTORE_PASSWORD`, `ANDROID_KEY_ALIAS`, `ANDROID_KEY_PASSWORD` (Focus Flow, Auto Tracker).
2. `android/key.properties` mit `storeFile`, `storePassword`, `keyAlias`, `keyPassword` (JSON Viewer, Markdown Viewer, FlexiPlan, StudiPlan).

| Fall | Test-APK | Release |
| --- | --- | --- |
| Secrets fehlen | Warnung, Debug-Schlüssel | Abbruch |
| Passwort oder Alias passen nicht (`keytool`-Vorabprüfung) | Warnung, Debug-Schlüssel | Abbruch |

Ein APK, das mit dem Debug-Schlüssel signiert ist, lässt sich nicht über die installierte App legen. Der Release bricht deshalb lieber ab.

## Versionsnummer

`versionCode` ist die Anzahl der Commits im Branch (`git rev-list --count HEAD`, deshalb `fetch-depth: 0`). Sie steigt mit jedem Merge und ist für das `latest`-APK und ein Release desselben Commits gleich. Android verweigert ein Update mit kleinerem Code, Play lehnt jeden schon gesehenen ab. Die Store-Version von Windows ist `<Präfix>.<Commit-Anzahl>.0`.

## Berechtigungen und Branch-Schutz

- Wer veröffentlicht (`publish: true` oder `github-release-tag`), muss dem aufrufenden Job `permissions: contents: write` geben. Eine aufgerufene Vorlage kann nicht mehr Rechte haben als der Aufrufer.
- Der Pflicht-Check im Branch-Schutz von `main` heißt `ci / Analyse und Tests`, wenn der aufrufende Job `ci` heißt. Die CI darf deshalb keinen Pfadfilter haben, sonst startet sie bei manchen Änderungen nicht und der Merge bleibt hängen. Den Build-Check darf man filtern, er ist kein Pflicht-Check.
- Windows-Läufe zählen auf privaten Repos doppelt gegen das Minutenkontingent (macOS zehnfach). Deshalb nur von Hand oder per Tag.

## Was die Vorlagen aus Erfahrung enthalten

- Der Datei-Beobachter von Gradle ist abgeschaltet (`org.gradle.vfs.watch=false`). In Focus Flow hing ein Release-Build daran mehr als 30 Minuten.
- Der Gradle-Heap ist auf 3 GB begrenzt, weil ein Heap von 8 GB den Runner volllaufen ließ (Focus Flow). Beides steht in `~/.gradle/gradle.properties` und hat damit Vorrang vor der `gradle.properties` der App.
- `timeout-minutes` bei jedem Job, damit ein Hänger früh scheitert.
- Der Gradle-Cache wird zwischen Läufen wiederverwendet.

## Stolpersteine beim Pflegen

- **Keine lokalen Composite-Actions** in den Vorlagen verwenden. Ein Pfad wie `./.github/actions/x` löst in einer aufgerufenen Vorlage im Repo des Aufrufers auf, nicht hier. Deshalb steht etwas Code mehrfach in den Dateien.
- Eingaben nie direkt in `run:` einsetzen, sondern über `env:` weitergeben.
- Eine Änderung hier trifft alle Apps. Erst in einer App über den Branch testen, dann auf `main` mergen, dann `v1` verschieben.
- Die Prüfung mit actionlint läuft bei jedem Push (`lint.yml`). Lokal: `actionlint .github/workflows/*.yml examples/*.yml`.

## Lizenz

MIT, siehe [`LICENSE`](LICENSE).
