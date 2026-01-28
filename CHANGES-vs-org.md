# Änderungen gegenüber dem Original (`vscode-wakatime-master_org`)

Dieses Dokument beschreibt alle Abweichungen dieser angepassten WakaTime-Extension im Vergleich zum unveränderten Original (Ordner `vscode-wakatime-master_org`).

---

## Übersicht

| Aspekt | Original | Diese Version |
|--------|----------|----------------|
| **AI vs. Human** | Heuristik: 4+ große Pastes in 500 ms = AI | Heuristik: „Kürzliche User-Interaktion“ + gleiche Zeile/±2 Zeilen |
| **Tab/Fokus** | Tab-Wechsel = Interaktion (Paste danach = Human) | Tab/Fokus zählen **nicht** als Interaktion |
| **Interaktion** | Zeitstempel pro Datei | Zeitstempel + **Zeilenbereich** (line, lineEnd) pro Datei |
| **Nah-Regel** | – | Änderung nur Human, wenn in **gleicher Zeile oder ±2 Zeilen** der letzten Interaktion |
| **Heartbeat bei reinem AI** | Fokus = `activeTextEditor` → oft Chat → kein Heartbeat | Bearbeitete Datei wird übergeben → Heartbeat für richtige Datei |
| **Senden bei AI** | Nur alle 30 s Puffer | Gleich wie Human: **nur 30 s Puffer** (kein sofortiges Senden) |
| **Projektname** | Workspace-/Ordnername | **`.wakatime-project`** in Datei- oder übergeordneten Verzeichnissen (erste Zeile = Projektname) |
| **Activity-Log** | Keins | Strukturiertes Log in `wakatime.log` mit `[WakaTime]`; `send`-Event: **Batch** (files, projects, sources, is_writes, count) |

---

## Geänderte Dateien

- **`src/constants.ts`** – neue Konstanten, Heartbeat-Category `'coding'`
- **`src/wakatime.ts`** – Hauptlogik (Interaktion, AI/Human, Log, Heartbeat/Send)
- **`src/web/wakatime.ts`** – kleine Anpassungen für Web-Build (AI-Category, gleiche Event-Logik)

**Nicht geändert:** `extension.ts`, `options.ts`, `utils.ts`, `package.json`, `logger.ts`, `dependencies.ts`, `desktop.ts`, etc.

---

## 1. `src/constants.ts`

### Neu

- **`RECENT_USER_INTERACTION_MS = 5000`**  
  Zeitfenster in ms: Nur wenn der User in dieser Zeit in der Datei interagiert hat (Cursor/Selektion oder Tippen), zählt die nächste Einfügung als Human.

- **`INTERACTION_NEAR_LINES = 2`**  
  Änderung zählt nur als Human, wenn sie in der **gleichen Zeile** oder **max. 2 Zeilen** neben der letzten Interaktion (Cursor/Selektion/Tippen) liegt. Sonst = AI.

### Geändert

- **`Heartbeat.category`**  
  Zusätzlich erlaubter Wert: **`'coding'`** (neben `'debugging' | 'ai coding' | 'building' | 'code reviewing'`).

---

## 2. `src/wakatime.ts`

### A) User-Interaktion pro Datei (neu)

- **`lastUserInteractionInFile`**  
  Pro Datei: `{ time, line, lineEnd }` – Zeitpunkt der letzten **echten** Interaktion und Zeilenbereich.
  - **Interaktion** = Cursor/Selektion **oder** Tippen (ein Zeichen).
  - **Keine Interaktion:** reines Tab-Wechseln oder Fokus setzen.

- **`hadRecentUserInteractionInFile(file, changeLine?)`**  
  Liefert `true`, wenn
  - für diese Datei in den letzten 5 s interagiert wurde **und**
  - (optional) `changeLine` im Bereich `[last.line - 2, last.lineEnd + 2]` liegt.

- **`getChangeLine(e)`** / **`getChangeLineEnd(e)`**  
  Erste bzw. letzte Zeile der Änderung aus `contentChanges` (min/max der Ranges).

**Wann wird Interaktion gesetzt?**

- **`onChangeSelection`:** Klick/Cursor/Selektion in der Datei → `lastUserInteractionInFile[file] = { time, line: startLine, lineEnd: endLine }` (Selection kann mehrere Zeilen umfassen).
- **`onChangeTextDocument`:** Einzelzeichen tippen oder löschen → `lastUserInteractionInFile[file] = { time, line: changeLine, lineEnd: changeLine }`.
- **`onChangeTab`:** Es wird **keine** Interaktion mehr gesetzt (Tab/Fokus zählen nicht).

### B) AI vs. Human (ersetzt Original-Logik)

- **Original:** AI nur wenn `recentlyAIPasted(now)` (4+ große Pastes in 500 ms) und `hasAICapabilities`.
- **Neu:**
  - **Human:** Einzelzeichen/Löschung **oder** große Einfügung/andere Edits **nur wenn** kürzliche User-Interaktion **in derselben Datei** und **nah** zur Änderung (gleiche Zeile oder ±2 Zeilen).
  - **AI:** AI-Chat-Sidebar **oder** (bei Cursor/Windsurf) große Einfügung/andere Edits **ohne** solche Interaktion oder **weit weg** von der letzten Interaktion.

- **Entfernt:** `recentlyAIPasted()` und die Abhängigkeit von „4 Pastes in 500 ms“.

### C) Activity-Log (neu)

- **`writeActivityLog(event, data)`**  
  Schreibt eine Zeile in **`~/.wakatime/wakatime.log`** im Format:
  ```text
  <ISO-Timestamp>  [WakaTime]  <event>  key=value  key=value  ...
  ```
  Kein „CUSTOM LOG“; Werte mit Leerzeichen werden in Anführungszeichen gesetzt.

- **Events:**
  - **`change`** – bei jeder Dokumentänderung: `file`, `project`, `source` (human/ai/unknown), `line`, `lineEnd`, `lines`, `changes`, `chars`.
  - **`heartbeat`** – wenn ein Heartbeat in den Puffer kommt: `file`, `project`, `source` (category), `line`, `lines`, `is_write`.
  - **`send`** – wenn Heartbeats ans Backend gesendet werden (ein Aufruf kann **mehrere** Heartbeats umfassen): `files`, `projects`, `sources`, `is_writes` (jeweils kommasepariert, gleiche Reihenfolge), `count` (Anzahl).

### D) Heartbeat bei „nur AI“ (Fokus im Chat)

- **`onEvent(isWrite, documentForHeartbeat?)`**  
  Zweiter Parameter optional: die **bearbeitete** Datei (z. B. bei AI-Änderung aus `onChangeTextDocument`).

- Bei **AI-Änderung** wird **`onEvent(true, e.document)`** aufgerufen.  
  Im Debounce-Callback wird dann **`doc = documentForHeartbeat ?? activeTextEditor?.document`** und ggf. `editor`/`selection` daraus abgeleitet.  
  Damit wird der Heartbeat für die **richtige Datei** erzeugt, auch wenn der Fokus im Chat liegt.

### E) Senden: einheitlich 30 s Puffer (auch für AI)

- In **`appendHeartbeat`** wird **nur** gesendet, wenn **30 s seit letztem Send** vergangen sind – für Human und AI gleich.  
  Kein sofortiges Senden mehr bei AI-Heartbeats.

### F) Projektname aus `.wakatime-project` (Desktop)

- **`getProjectName(uri)`** sucht vom Verzeichnis der Datei aufwärts bis Workspace-Root nach einer Datei **`.wakatime-project`**.  
  Wird sie gefunden, wird die **erste Zeile** (getrimmt) als Projektname verwendet.  
  Sonst Fallback: Workspace-Ordner-Name wie bisher.

### G) Entfernt

- **`recentlyAIPasted(time)`** – wird nicht mehr verwendet.
- **`AI_RECENT_PASTES_TIME_MS`** – aus `constants.ts` entfernt (keine „4 Pastes in 500 ms“-Heuristik mehr).

---

## 3. `src/web/wakatime.ts`

- **Web-Build nutzt dieselbe AI/Human-Logik wie Desktop:**  
  `lastUserInteractionInFile`, `hadRecentUserInteractionInFile`, `getChangeLine`/`getChangeLineEnd`, `onChangeSelection` setzt Interaktion (Cursor/Selektion), `onChangeTextDocument` wie Desktop (Human bei kürzlicher Interaktion + Nah-Regel, sonst AI wenn capable), **`onEvent(isWrite, documentForHeartbeat?)`** mit `documentForHeartbeat` bei AI-Änderung; Senden wie Desktop: **nur 30 s Puffer**.  
- **`Heartbeat.category`** darf **`'coding'`** sein.  
- **Unterschied zu Desktop:** Kein `writeActivityLog` im Web (kein Activity-Log in `wakatime.log`).

---

## Log-Beispiele (diese Version)

```text
2026-01-28T20:15:00.123Z  [WakaTime]  change  file=/path/to/file.js  project=my-project  source=human  line=15  lineEnd=15  lines=42  changes=1  chars=12
2026-01-28T20:15:00.456Z  [WakaTime]  heartbeat  file=/path/to/file.js  project=my-project  source=coding  line=15  lines=42  is_write=1
2026-01-28T20:15:01.000Z  [WakaTime]  send  files=/path/a.js,/path/b.js  projects=proj-a,proj-b  sources=coding,ai coding  is_writes=0,1  count=2
```
(Einzelner Send: `files=/path/to/file.js  projects=my-project  sources=coding  is_writes=1  count=1`)

---

## Kurz: Was ist neu gegenüber _org?

| Thema | Original | Diese Version |
|-------|----------|----------------|
| Konstante 5 s Interaktion | – | `RECENT_USER_INTERACTION_MS = 5000` |
| Konstante „nah“ (Zeilen) | – | `INTERACTION_NEAR_LINES = 2` |
| Heartbeat-Category `'coding'` | nicht im Typ | erlaubt |
| Interaktion = Tab/Fokus | ja | **nein** (nur Cursor/Selektion/Tippen) |
| Interaktion mit Zeilenbereich | nein | ja (`line`, `lineEnd`) |
| „Nah“-Prüfung (gleiche Zeile / ±2) | – | ja |
| AI-Erkennung | 4 Pastes in 500 ms | Kürzliche Interaktion + Nah-Regel |
| Activity-Log | nein | ja, strukturiert in `wakatime.log`; Send-Log: Batch (files, projects, sources, is_writes, count) |
| Heartbeat-Datei bei AI (Fokus im Chat) | `activeTextEditor` | `documentForHeartbeat` (bearbeitete Datei) |
| Senden bei AI | nur 30 s Puffer | nur 30 s Puffer (wie Human) |
| `recentlyAIPasted` | ja | entfernt |

---

*Stand: Vergleich mit `vscode-wakatime-master_org`.*
