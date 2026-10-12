# ci-templates – Hinweise für Claude

Wiederverwendbare GitHub-Actions-Workflows (`workflow_call`) für alle Flutter-Apps von Nicolas. **Zuerst `README.md` lesen.**

## Grundsätze

- Eine Änderung hier trifft alle Apps. Robust und rückwärtskompatibel ändern: neue Eingaben mit Standardwert, bestehende nicht umbenennen.
- Vorlagen enthalten keine Auslöser, kein `concurrency` und keine Rechte. Das legt die aufrufende App fest.
- Keine Secrets, Schlüssel, Tailnet-Namen oder IP-Adressen. Secrets kommen als `KEYSTORE_BASE64`, `KEYSTORE_PASSWORD`, `KEY_ALIAS` vom Aufrufer.
- Keine lokalen Composite-Actions (`./.github/actions/…`): sie lösen im Repo des Aufrufers auf.
- Eingaben nie direkt in `run:`, immer über `env:`.
- Jeder Job hat `timeout-minutes`.
- Vor jedem Push `actionlint .github/workflows/*.yml examples/*.yml`. Das Signatur-Skript lässt sich mit `yq` herausziehen und lokal mit einem Test-Keystore ausführen.
- Neue oder geänderte Vorlage zuerst über den Branch in einer App testen (zum Beispiel Focus Flow), dann nach `main` mergen, dann das Tag `v1` verschieben.

## Konventionen

- Doku, Kommentare und Namen der Schritte auf Deutsch, Bezeichner (Eingaben, Jobs, Dateien) auf Englisch.
- Commits nur mit der GitHub-noreply-Adresse `241658994+nicolasasauer@users.noreply.github.com` (das Repo ist öffentlich).
- Die Entscheidungen hinter diesen Workflows stehen im Dokument „Meine Präferenzen bei Code-Projekten und Android-Apps“, Abschnitt „Workflows“: drei Stufen, getrennte Dateien, `latest`-APK nur für arm64, CI als Pflicht-Check.
