# Arbeitsweise im Projekt FlagOS Competition

Du arbeitest an FlagOS Competition (PWA, Ziel https://competition.flagos.app, passwortgeschützte Testphase) –
der Oberfläche für Turnier und Spieltag (Spielplan, Ergebnisse, Tabellen, Public Display).
Vorher hieß das Projekt „Tournament Control“ (`flagos-pwa`). Konzept: `FlagOS/20_Product/30_Features/20_Competition/FlagOS_Competition_Concept_V0.2_Draft.md`.
Antworte auf Deutsch. Produkt-Owner ist Sven; er führt Git-Merges und Deploys selbst aus.
Verfahren, Stack und Deployment sind aus FlagOS Broadcast und Stadium übernommen (`~/Developer/FlagOS/flagos-broadcast-pwa`, `…/flagos-stadium-pwa`).

## Feste Regeln
- Nichts ohne Rückfrage auf Live-Systeme anwenden (Supabase-Migrationen, Daten, Einstellungen, Server).
- **Namensgebung (entschieden 16.10.2026, Konzept Abschnitt 7.2):** Die bestehenden Tabellen, Typen und die View heißen weiter `tournament_*` (technischer Name), das Produkt heißt Competition. **Neue Tabellen und Funktionen bekommen das Präfix `competition_`**, neue Spalten an bestehenden Tabellen normale Namen. Nichts umbenennen, nichts löschen: Broadcast Studio (live), iOS-App und diese PWA hängen an den Namen. Das „Gerade ziehen“ passiert erst vor der Live-Schaltung (Konzept 7.3); bis dahin Fundstücke in die Sammelliste am Ende des Konzepts eintragen. Typen und Services (`Tournament…`) spiegeln die Datenbank und bleiben deshalb so; Tabellennamen nur in der Zugriffsschicht `src/services/competitionApi.ts` verwenden, nicht in Seiten; die Seiten arbeiten mit `src/types/competition.ts`.
- `app_role` nie als Competition-Recht werten; Rechte gelten pro Competition (geplant: `competition_members`).
- Importierte, noch nicht freigegebene Spieltage (`team_events.approval_status = 'pending'`) dürfen nie öffentlich erscheinen.
- **Sicherheit bei jedem Schritt** (Prüfung am 16.10.2026: die Rohtabellen `tournament_*` sind für `anon` pauschal lesbar, Freigabe wirkt dort nicht):
  - Neue Tabellen/Views immer mit Row-Level-Security; kein pauschales `USING (true)` für `anon` bei Daten mit Freigabestatus.
  - Jede Migration bekommt einen Zugriffstest (nicht angemeldet, Fremder, Team-Vertretung, Feldleitung, Turnierleitung) und ein Rollback-Skript; Vorbild `flagos-broadcast-pwa/tests/crew-access.sql`.
  - Keine zweite Beziehung (Fremdschlüssel) zwischen Tabellen, die schon verbunden sind (`tournaments` ↔ `tournament_teams` usw.): sie macht die Einbettung `tournament_teams (*)` mehrdeutig und bricht iOS-App und Competition (Migration 050 → 050b). Der Zugriffstest prüft das.
  - In der PWA nie darauf verlassen, dass die Oberfläche etwas ausblendet; die Datenbank-Regeln entscheiden.
  - Der öffentliche Schlüssel liegt in jedem Build; alles, was damit lesbar ist, ist öffentlich.
- Keine Geheimnisse ins Repo oder in den Build (nur der Supabase-anon-Schlüssel, in `.env.local`).
- Git-Befehle für Sven blockweise ausgeben, OHNE `#`-Kommentarzeilen.
- Nur die eigenen Änderungen committen; fremde, nicht committete Änderungen nie mit `git add -A` einsammeln. `.claude/` gehört nicht ins Repo.
- Unterscheide bestätigte Entscheidungen und Vorschläge (Konzept, Abschnitt 9).
- Tests: `50_Testing/10_Test_Plans/2026-10-16-FlagOS_Competition_Testplan.md` (im Projektordner FlagOS). Bei jeder Datenbankänderung zählt Abschnitt 3.10 (Broadcast Studio und iOS-App bleiben unverändert nutzbar).

## Technik
- React 19 mit React-Compiler-Lint (kein manuelles useCallback, keine Ref-Zuweisung im Render, kein setState direkt im Effect), TypeScript, Vite, Tailwind v4, Supabase JS, vite-plugin-pwa.
- Navigation über React Router (`src/App.tsx`), Seitenleiste in `src/layouts/Layout.tsx` (Icons, Logo = Startseite, Konto/Einstellungen unten). Admin unter `/admin` (alter Einstieg `/#admin` leitet weiter; Supabase-Login, `is_platform_admin()`). Aussehen und Bausteine wie Broadcast/Stadium (`src/components/ui.tsx`).
- Daten kommen aus FlagOS-Supabase (`.env.local`: `VITE_SUPABASE_URL`, `VITE_SUPABASE_ANON_KEY`). Der Service-Worker cached nur die App-Hülle, nie Daten.
- Dev-Server-Port 5173 (5174 = Broadcast, 5175 = Stadium).
- Lokaler Code-Ort auf Svens Mac: `~/Developer/FlagOS/flagos-competition-pwa`. GitHub: `DaRealZwenno/flagos-competition-pwa` (früher `flagos-pwa`), Branch `main` = Stand, der deployt wird.

## Ablauf pro Änderung
1. Auf einem Feature-Branch arbeiten (`feat/…` oder `fix/…`), nie direkt auf `main`.
2. Vor Abgabe prüfen, alles muss fehlerfrei sein:
   - `npx tsc -b`
   - `npx eslint .`
   - `npm test`
   - `npx vite build`
   - Bei UI-Änderungen: im Browser bzw. mit Playwright in Handy-Ansicht (390 × 844, Touch), iPad (1180 × 820) und Laptop (1440 × 900) prüfen.
3. Kurz zusammenfassen, was sich geändert hat und warum (auf Deutsch, ohne Schrittprotokoll).
4. README bzw. Konzept ergänzen, wenn sich Verhalten oder Struktur ändert.
5. Git-Befehle an Sven ausgeben (oder einen Pull Request öffnen, wenn du direkt mit GitHub arbeitest):

   git switch -c <branch>
   git add -A
   git commit -m "<kurze deutsche Beschreibung>"
   git push -u origin <branch>

   git checkout main
   git pull
   git merge <branch>
   git push
   npm run deploy

   git branch -d <branch>

## Deploy (macht Sven auf dem Mac, nicht du)
`npm run deploy` startet `deploy/deploy.sh` (Hosting: Serverprofis, cPanel, Upload per SSH/rsync; Zugangsdaten in `deploy/deploy.config`, nicht eingecheckt). Das Skript:
- deployt nur von `main`, sauber und identisch mit `origin/main`,
- baut, entfernt Fremddateien, bricht ab bei `.map`/`.env` oder Nicht-anon-Schlüsseln im Build,
- lädt erst `.htaccess` und Assets, dann Einstiegsdateien (index.html, sw.js, manifest, version.json); alte Assets bleiben 30 Tage,
- prüft danach ohne Zugangsdaten: alles 401, HTTP → HTTPS,
- behält die letzten 5 Releases lokal.
Optionen: `npm run deploy -- --dry-run` (ohne Upload), `-- --rollback` (Releases zeigen), `-- --rollback <ID>` (zurück), `-- --force` (nur Git-Prüfungen überspringen, Notfall).
Server-Regeln stehen in `deploy/htaccess.template` (Basic Auth, Sicherheits-Header, CSP). Neue externe Abrufe brauchen einen Eintrag in `connect-src` bzw. `img-src` der CSP (Team-Logos kommen aus dem Supabase-Storage).
Zugänge: dieselbe Passwortdatei wie Broadcast (`/home/hoffandt/.htpasswds/broadcast.flagos.app/passwd`) – Benutzer pflegt Sven in cPanel → Verzeichnisschutz bei broadcast.flagos.app.

## Updates auf den Geräten
Die PWA sucht beim Öffnen, beim Zurückwechseln und alle 30 Minuten nach einer neuen Version und zeigt dann „Update verfügbar“. Aktiv wird die neue Version erst nach „Neu laden“, nie automatisch. Die laufende Version steht unten in der Seitenleiste (`v0.1.0 · Commit`).

## Datenbank und Umgebungen (Stand 31.10.2026)

Dieses Repo ist **öffentlich**: keine Projekt-Kennungen, Schlüssel, internen Pläne oder Termine in diese Datei oder in Commits schreiben. Details stehen im privaten Backend-Repo (`supabase/README.md`).

- **Zwei Umgebungen:** Live und Probebühne (Staging). Zuerst Probebühne, Live **nur mit ausdrücklicher Freigabe von Sven** (Migrationen, Daten, Einstellungen, Storage, Server).
- **Lokal gegen die Probebühne:** `npm run dev` liest `.env.development.local` (Probebühne), der Produktions-Build liest `.env.local` (Live). Beide Dateien nicht im Repo; nur der öffentliche anon-Schlüssel gehört in den Build.
- **Migrationen nur im Backend-Repo** (Zeitstempel-Namen, je Migration Rollback und Zugriffstest). Hier keine eigenen Migrationen.
- **Aufräumen vor neuen Funktionen:** keine neuen Tabellen mit eigenem Team-, Personen- oder Mitgliederkonzept; neue Fachtabellen bauen auf der gemeinsamen Kernstruktur (Teams, Personen, Rechte, Wettbewerbe) auf.
- **Neue Namen kommen** (u. a. `tournament*` → `competition*`): neuen Code nicht unnötig auf die alten Namen bauen.
