# Projekt: YouTube-Transkripte für den Gicht-Kanal – Stand & Kontext

> Übergabedokument für Claude Code. Fasst zusammen, was im bisherigen Chat erarbeitet wurde,
> damit neue Varianten der YouTube-Analyse darauf aufbauen können.
> Stand: Oktober 2026

---

## 1. Ziel & Hintergrund

- **Betreiber:** Jochen Hein, Ernährungsberater für Gicht.
- **YouTube-Kanal:** `https://www.youtube.com/@jochenhein-gicht-lotse` – ca. 45 öffentliche Videos, erste Videos April 2017.
- **Websites:** gicht-coach.de (Beratung), gicht-lotse.de (Informationen) – beide WordPress.
- **Ziel des Workflows:** Für jedes Video ein sauberes Transkript erzeugen (Zeichensetzung, Absätze) und in Google Drive ablegen.
- **Weitere Verwendung der Transkripte:**
  1. Wissensbasis für einen **Chatbot** (geplant mit dem Anbieter BRYGHT; öffentlich per Link / auf der Website nutzbar).
  2. Ausgangsmaterial für **abgeleitete Inhalte**: Kurzbeschreibungen (KB), Instagram-Posts (IG) u. a.

---

## 2. Aktuelle Lösung: n8n-Workflow (n8n Cloud)

Datei: `youtube-transkripte-workflow-v2.json` (importierbar in n8n; Credentials müssen nach dem Import neu zugewiesen werden).

### Ablauf

```
Start (manuell)
  → Konfiguration (Set-Node: alle Parameter zentral)
  → Apify: Actor starten (POST, async)
  → 30 Sek. warten  ⇄  Apify: Status abfragen  → Fertig? (SUCCEEDED)
                                              ↳ Läuft noch? (RUNNING/READY) → zurück zu Warten
                                              ↳ sonst → Fehler melden (Stop and Error)
  → Apify: Ergebnisse holen (Dataset-Items)
  → Nur Videos mit Transkript (Filter: transcript_text nicht leer)
  → Loop über Videos (Split in Batches, 1 pro Durchlauf)
      → Claude: Text aufbereiten (HTTP → Anthropic Messages API)
      → Google Drive: Create From Text (als Google Doc)
      → zurück in den Loop
```

Die 30-Sekunden-Warteschleife gilt nur für den **einen** Apify-Lauf (alle Videos zusammen), nicht pro Video.

### Konfiguration (Set-Node „Konfiguration“)

| Feld | Typ | Wert (aktuell / Beispiel) |
|---|---|---|
| `channel_url` | String | `https://www.youtube.com/@jochenhein-gicht-lotse` |
| `max_videos` | Number | Test: `3`, Vollauf: `50` |
| `language` | String | `de` |
| `start_date` | String | Test: `2017-04-01` · Vollauf: `2017-05-01` |
| `end_date` | String | Test: `2017-04-30` · Vollauf: leer lassen (= bis heute) |
| `ki_modell` | String | `claude-haiku-4-5-20251001` (Qualität reicht; günstiger als Sonnet) |
| `prompt` | String | System-Prompt für die Textaufbereitung (siehe unten) |
| `drive_folder_id` | String | wird nicht mehr genutzt (Ordner wird im Drive-Node per „From list“ gewählt) |

Datumsformat strikt `JJJJ-MM-TT` – Apify validiert per Regex (ein Tippfehler `2017-04-30-` führte zu „Bad request“).

---

## 3. Apify

- **Actor:** `starvibe/youtube-video-transcript` (Actor-ID `Uwpce1RSXlrzF6WBA`)
- **API-Endpunkte:**
  - Start: `POST https://api.apify.com/v2/acts/starvibe~youtube-video-transcript/runs`
  - Status: `GET https://api.apify.com/v2/actor-runs/{runId}` → `data.status`
  - Ergebnisse: `GET https://api.apify.com/v2/datasets/{defaultDatasetId}/items?clean=true&fields=video_id,title,url,published_at,transcript_text`
- **Auth:** Header `Authorization: Bearer <APIFY_TOKEN>` (in n8n als Header-Auth-Credential „Apify API“).
- **Input-Felder:** `channel_url` **oder** `youtube_url` (genau eins), `max_videos` (1–200, Default 10), `language` (ISO 639-1), `start_date` / `end_date` (ISO 8601, nur bei Kanal), `include_transcript_text` (bool, Default false → **muss true sein**).
- **Output-Felder (Auswahl):** `video_id`, `title`, `url`, `published_at`, `transcript_text`, `transcript` (Array mit Zeitstempeln), `description`, `duration_seconds`, `view_count`, `like_count`, `comment_count`, `thumbnail`, `is_auto_generated`, `available_languages`, `status`, `message`.
- **Nur öffentliche Videos:** Der Actor liest die öffentliche Kanalseite → private/nicht gelistete Videos werden ohnehin nicht erfasst.
- **Kosten:** ca. 5 $ / 1.000 Ergebnisse; Free Tier 50 Transkripte/Tag.

Body-Expression im Node „Apify: Actor starten“ (Datum nur senden, wenn gesetzt):

```js
{{ (() => { const c = $('Konfiguration').first().json; const body = { channel_url: c.channel_url, max_videos: c.max_videos, language: c.language, include_transcript_text: true }; if (c.start_date) body.start_date = c.start_date; if (c.end_date) body.end_date = c.end_date; return JSON.stringify(body); })() }}
```

---

## 4. Claude (Textaufbereitung)

- **Endpunkt:** `POST https://api.anthropic.com/v1/messages`
- **Header:** `x-api-key: <ANTHROPIC_API_KEY>` (Header-Auth-Credential „Anthropic API“), `anthropic-version: 2023-06-01`
- **API-Key:** aus der Claude Console (platform.claude.com) – getrennt vom Claude-Abo, Guthaben nötig.
- **Timeout:** 600000 ms; Retry 3× mit 5 s Pause.

Body-Expression:

```js
{{ JSON.stringify({ model: $('Konfiguration').first().json.ki_modell, max_tokens: 32000, system: $('Konfiguration').first().json.prompt, messages: [{ role: 'user', content: $json.transcript_text }] }) }}
```

Antworttext extrahieren:

```js
{{ $json.content.filter(c => c.type === 'text').map(c => c.text).join('\n\n') }}
```

### System-Prompt (aktuell)

```
Du bist ein sorgfältiger Lektor. Du erhältst das automatisch erzeugte Transkript eines deutschsprachigen YouTube-Videos zum Thema Gicht und Ernährung. Es enthält kaum Satzzeichen und keine Absätze.

Deine Aufgabe:
1. Setze korrekte Satzzeichen (Punkte, Kommata, Fragezeichen, Doppelpunkte usw.) und korrigiere die Groß- und Kleinschreibung.
2. Gliedere den Text in sinnvolle Absätze (ein Gedanke bzw. Thema pro Absatz). Trenne Absätze durch eine Leerzeile.
3. Behalte den Wortlaut bei: nichts kürzen, nichts zusammenfassen, nichts hinzufügen, nicht umformulieren.
4. Korrigiere nur eindeutige Erkennungsfehler bei Fachbegriffen (z. B. Harnsäure, Purine, Allopurinol, Febuxostat), wenn sie offensichtlich falsch transkribiert wurden.
5. Reine Füllwörter wie „äh“ oder „ähm“ darfst du entfernen.

Gib ausschließlich den bearbeiteten Text zurück – ohne Einleitung, Überschrift, Markdown oder Kommentar.
```

---

## 5. Google Drive (Ablage)

- **Node:** Google Drive v3 · Resource *File* · Operation **Create From Text** · Option **Convert to Google Document = an**.
- **Ziel:** Parent Drive *My Drive* · Parent Folder per **From list** → Ordner **„Gicht-Kompass“**.
- **Credential:** Google Drive OAuth2 über n8n Cloud („Sign in with Google“, n8n-eigene OAuth-App – **keine** eigene Google-Cloud-Console nötig).

### Dateinamen-Konvention

```
JJJJ-MM-TT-<Videotitel>-<KÜRZEL>
Beispiel: 2017-04-29-Gicht Warum ist mein Harnsäure-Wert erhöht?-TS
```

| Kürzel | Bedeutung |
|---|---|
| `TS` | Transkript |
| `KB` | Kurzbeschreibung (geplant) |
| `IG` | Instagram-Post (geplant) |

Vorteil: Sortierung nach Name = chronologisch; alle Ableitungen eines Videos stehen untereinander. Datum stammt aus `published_at` (UTC – kann bei nächtlichen Uploads um einen Tag abweichen).

File-Name-Expression:

```js
{{ (() => { const v = $('Loop über Videos').item.json; const d = (v.published_at || '').slice(0, 10); return (d ? d + '-' : '') + v.title.replace(/[\\/]/g, '-').trim() + '-TS'; })() }}
```

### Mehrere Google-Konten

- Es gibt Drive „A“ (johein1503@gmail.com, in n8n verbunden) und Drive „B“ (andere Gmail-Adresse).
- Kopieren A → B: Ordner freigeben → in B „Kopie erstellen“ → Präfix „Kopie von “ per **Google Apps Script** entfernen (Script `kopieVonEntfernen()` mit `DriveApp.getFolderById(...)`, `startsWith('Kopie von ')`, `setName(...)`).
- Für Drive B in n8n einfach eine zweite Google-Drive-Credential per „Sign in with Google“ anlegen.

---

## 6. Hinweise für den Chatbot / Wissensbasis

- **Bestes Format für Claude-basierte Bots:** reiner Text – Markdown (`.md`) oder `.txt`, UTF-8. Bei claude.ai-Projekten funktionieren Google Docs (via Drive-Anbindung) und `.docx` ebenfalls.
- **Kopfbereich je Datei empfohlen** (damit der Bot Quelle/Datum nennen kann):

  ```
  # <Videotitel>

  Typ: Transkript
  Veröffentlicht: JJJJ-MM-TT
  Quelle: <YouTube-URL>

  ---

  <Text>
  ```

- **Datenmenge:** 45 Transkripte ≈ 300–400 k Tokens + 20–30 weitere Dokumente → Bot braucht Retrieval (RAG), nicht „alles in den Kontext“.
- **Nährwert-Tabellen (Excel):** als **CSV UTF-8** bereitstellen; eine Kopfzeile mit Einheiten, keine verbundenen Zellen/Leerzeilen. Für RAG robuster: jede Zeile als selbsterklärender Satz, z. B.
  `Linsen, getrocknet: Purine 127 mg/100 g, Harnsäure 305 mg/100 g`.
- **Analyse der Nutzerfragen** gewünscht (anonym): Gesprächsprotokoll, Themen-Clustering, Liste unbeantworteter Fragen, Monatsbericht. DSGVO beachten (Gesundheitsdaten, AVV, Löschfristen, Hinweis im Chatfenster).
- **Offene Fragen an BRYGHT:** unterstützte Dateiformate / Drive-Sync, verwendetes Modell, Quellenangaben, Serverstandort & AVV, Leitplanken (keine Diagnosen), Kostenmodell & Dokumentlimits, Einbettung auf WordPress, Daten-Export, Analytics.

---

## 7. Lessons Learned (n8n-Praxis)

- Beim JSON-Import gingen einzelne Einstellungen verloren: **Generic Auth Type** (→ manuell „Header Auth“ wählen), **Operation** im Drive-Node, Inhalt eines JSON-Body-Felds. Nach jedem Import alle Nodes prüfen.
- Expressions immer im Modus **Expression** (fx) einfügen; Werte wie Modellnamen im Modus **Fixed**, ohne Anführungszeichen.
- Google-OAuth-Fehler `EAUTH … refresh token invalid` → Credential neu verbinden („Sign in with Google“).
- **Safari blockiert das OAuth-Pop-up** → Pop-ups für die n8n-Domain erlauben (Safari → Einstellungen → Websites → Pop-up-Fenster).
- Einzelne Nodes per „Execute step“ testen geht nur, wenn vorherige Nodes Daten liefern → besser ganzen Workflow mit kleinem `max_videos` testen.

---

## 8. Ideen für neue Varianten (Ausgangspunkt für Claude Code)

1. **Ableitungen aus vorhandenen Transkripten:** Dateien `*-TS` aus „Gicht-Kompass“ lesen → Claude erzeugt Kurzbeschreibung (`-KB`) und Instagram-Texte (`-IG`) → gleicher Namensstamm.
2. **Markdown-Ausgabe für den Chatbot:** Convert to Google Document aus, Dateiname mit `.md`, Kopfbereich (Abschnitt 6) automatisch einfügen.
3. **Inkrementeller Lauf für neue Videos:** `start_date` = Datum des letzten verarbeiteten Videos; optional per Zeitplan (Schedule Trigger).
4. **Themen-Analyse über alle Videos:** Themen/Schlagworte je Video extrahieren → Übersichtstabelle (z. B. Google Sheet) für Content-Planung und Lückenanalyse.
5. **Alternative ohne n8n:** Python-/Node-Skript (Apify-API + Anthropic-API + Google-Drive-API) – z. B. mit Claude Code lokal am iMac.

### Kostenrahmen (Orientierung)

- Apify: ~0,25 $ für 45 Videos.
- Claude Haiku 4.5: wenige Dollar für alle 45 Transkripte (Sonnet ≈ 3× teurer).
