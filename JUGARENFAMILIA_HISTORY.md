# JugarEnFamilia.es — History (append-only)

> **This file is append-only: never remove or rewrite anything in it.** It is the project's memory: every session log, every decision and its reasons.
> The **current state** lives in `JUGARENFAMILIA_HANDOFF.md`: read that first, every session. Come here for the *why* behind something (search by name, version or date).
>
> **How to add to it:** at the end of each session, append a new `### Session N (date)` entry at the very end (what was built, versions, decisions and reasons, test results), then update `JUGARENFAMILIA_HANDOFF.md` so it describes the new current state.
>
> **Split made in Session 30 (1 Nov 2026).** Everything below the line is the old `JUGARENFAMILIA_HANDOFF.md` exactly as it was at that point, word for word (Sessions 1–30). Some statements in it were later superseded; the current file is the authority on how things are now.

---

# JugarEnFamilia.es — Project Handoff Document
*Last updated: 31 October 2026 — Session 30*

> ⚠️ **HANDOFF INTEGRITY RULE — DO NOT DELETE CONTENT**
> This document is append-and-update only. Never remove sections, rules, known issues, backlog items, or session log entries. Only add new content and update existing entries. A truncated handoff causes the next session to lose critical context.

---

## 🎮 Project Overview

A multiplayer browser-based version of the classic Spanish word game "Stop/Tutti Frutti/Alto el lápiz". Built as a single HTML file with Firebase for real-time multiplayer.

**Tagline:** "el juego de siempre, con los de siempre"

---

## 🌐 Live URLs

| Environment | URL |
|---|---|
| Production | https://jugarenfamilia.es |
| Staging | https://jugarenfamilia.es/index_tmp.html |
| GitHub Pages | https://oscargdiez.github.io/jugarenfamilia/ |

---

## 🛠 Tech Stack

| Layer | Tech | Notes |
|---|---|---|
| Frontend | Single HTML file | No framework, vanilla JS |
| Realtime | Firebase Realtime Database | europe-west1 |
| AI validation | Cloudflare Worker → OpenRouter | Worker at https://api.oscar-g-diez.workers.dev/ |
| Hosting | GitHub Pages | Free SSL |
| Domain | jugarenfamilia.es | Namecheap |

---

## 🔑 Credentials & Config

### Firebase
- **Project:** `stop-9f0ea`
- **Database URL:** `https://stop-9f0ea-default-rtdb.europe-west1.firebasedatabase.app`
- **Config:** embedded in HTML (public, protected by Firebase rules)
- **Schema additions (Session 12):**
  - `daily/{date}/{lang}/players/{safeName}: true` — legacy cross-device dedup index (still written, used for backfill by fetchPlayedState)
  - `names/{safeName}/secret: hash` — SHA-256 hash of email or PIN for name claim
  - `names/{safeName}/displayName: string` — original display name with accents
- **Schema additions (Session 13):**
  - `daily/{date}/{lang}/scores/{playerKey}/safeName: string` — for emoji-independent matching
- **Schema additions (Session 15):**
  - `names/{safeName}/played/{date}/{lang}: true` — cross-device played state, per language, source of truth for Hecho button label
- **Schema additions (Session 16):**
  - `names/{safeName}/contests/{date}/{lang}: N` — count of daily recontest uses (max 1 per lang per day)
- **Schema additions (Session 18):**
  - `daily/{date}/{lang}/scores/{playerKey}/speedMultiplier: float` — speed multiplier now saved to Firebase on submit (was localStorage only)
- **Schema additions (Session 22):**
  - `daily/{date}/{lang}/scores/{playerKey}/aiResponses: { "2": "...", "4": "..." }` — raw AI reasoning text for invalid/unsure answers only (capped 1000 chars, `<think>` stripped). Only written if at least one invalid/unsure answer. Valid answers excluded.
  - `names/{safeName}/contests/{date}/{lang}_log/{idx}/aiResponse: string` — raw AI reasoning text written on every recontest, for contest quality analysis. **No longer written since v346** — see `daily/{date}/{lang}/scores/{playerKey}/contestLog/{idx}` (Session 27).
- **Security rules (Session 25 — permanent, no expiry):** until 29 Sep 2026 the database ran on Firebase "test mode" rules that would have locked ALL reads/writes on 28 Sep 01:00 Spain time. Replaced with per-path rules, published by Oscar and checked live (daily, create room, join via link all OK):
  ```json
  { "rules": {
      "rooms":{".read":true,".write":true}, "daily":{".read":true,".write":true}, "names":{".read":true,".write":true},
      "global":{".read":true,".write":true}, "groups":{".read":true,".write":true}, "groupNames":{".read":true,".write":true},
      "groupsMeta":{".read":true,".write":true,".indexOn":["lastPlayed"]},
      "groupsLb":{".read":true,".write":true}, "history":{".read":true,".write":true}, "historyIndex":{".read":true,".write":true} } }
  ```
  Same openness as before for the listed paths; any other root path is refused. **Any new top-level path in the code needs a rules line.** When Build 5 reaches production: delete `global/`, `groups/`, `groupNames/` in the console AND remove their three lines from the rules. (The `lastPlayed` index for Build 5 is already in.)
- **Schema additions (Session 25, Build 5a — staging only so far):** `rooms/{code}/groupId`, `rooms/{code}/groupName`; `groupsMeta/{groupId}: {name, createdAt, lastPlayed}`; `groupsLb/{groupId}/{nameKey}: {name, lastEmoji, wins, games}` (nameKey = normalised name, no emoji); `history/{groupId}/{gameId}: {endedAt, lang, host, groupName, scores, players, roundLog, currentRound, rounds, reactionPoints}`; `historyIndex/{groupId}/{gameId}: {endedAt, winners[], lang}`. gameId = time-sortable base36. localStorage `alto_mygroups` = `[{id,name,last,def}]` max 10; old `alto_groups` / `alto_last_group` are removed on load.
- **Daily revalidation marker (Session 25):** `daily/2026-09-29/{lang}/scores/{playerKey}/revalidated: timestamp` — written by the one-off revalidation after the AI outage (see Session 25 log).

### OpenRouter API
- **Key:** stored in Cloudflare Worker only — never in the HTML or GitHub repo
- **Key name:** Alto
- **Spending cap:** $4 (free tier only)
- **Models:** `inclusionai/ling-3.0-flash-fin:free` (primary, used for both daily validation and recontest). The nemotron and glm models were removed from recontest in Session 22 — they were silently failing and votes counted as invalid.
- **Logs:** visible at openrouter.ai → Logs → filter by API key "Alto"
- **UPDATE Session 25 (29 Sep 2026) — the free Ling is gone.** OpenRouter withdrew `inclusionai/ling-3.0-flash-fin:free` (last good call 28 Sep 17:19); every daily/practice/contest call got a 404 and all answers became dudoso. Since v260926.313 the game uses ONE shared list `AI_MODELS` (top of the module script, next to `DAILY_CONSENSUS`) for daily, practice, contests and multiplayer, always sent as an OpenRouter fallback list (`models: [...]`):
  1. `inclusionai/ling-3.0-flash-fin` — **PAID** (~$0.04 in / $0.12 out per M tokens; a few cents a month). Needs credit on the account; Oscar added credit on 29 Sep.
  2. `nvidia/nemotron-3-super-120b-a12b:free` — backup; in practice always answers "not sure" (thinking model; the 800-token limit probably runs out before the verdict).
  3. `z-ai/glm-5.2:free` — backup; **found DEAD (404) by the scout on 29 Sep** — replace when the scout results are in.
  The old "Spending cap: $4 (free tier only)" note above is out of date now that a paid model is first. **Oscar has set a credit limit on the key (29 Sep).**
- **UPDATE Session 27 (8 Oct 2026): `AI_MODELS` = `inclusionai/ling-3.0-flash-sante` (PAID, one provider NovitaAI, $0.042/$0.1232 per M tokens) → `inclusionai/ling-3.0-flash-fin` (paid) → `cohere/north-mini-code:free`.** Sante FREE was withdrawn by OpenRouter on 8 Oct. Expected cost ~30 cents/month; the key's credit limit is the cap; if credit runs out OpenRouter falls through to free Cohere. See the Session 27 log for scout numbers.
- **Paid Ling has two providers (NovitaAI, DeepInfra)** and they do not always agree, even at temperature 0 — the same answer can get VALID one call and INVALID the next. The free version ran only on Novita. Fix idea in Flagged for Future (pin one provider).
- **Privacy settings (openrouter.ai/settings/privacy), 29 Sep:** "Allow free endpoints that train on request data" = ON; "Allow free endpoints that publish prompts" = OFF (Oscar may turn it on — only letter/category/word are ever sent); ZDR all off; paid training off. Many `:free` models answer 404 "No endpoints found matching your data policy" unless these are on — the scout cannot tell that apart from a removed model.
- **Worker (`api.oscar-g-diez.workers.dev`, Cloudflare):** forwards the request body unchanged to OpenRouter with the key from the worker; only checks the `Origin` header (jugarenfamilia.es / www). ⚠️ A script can fake `Origin`, so anyone could run ANY model through it on Oscar's credit — see Flagged for Future (worker controls the model list).

### GitHub
- **Repo:** github.com/oscargdiez/jugarenfamilia (public)
- **Pages:** enabled, main branch, / root

### Namecheap DNS
```
A Record  @    185.199.108.153
A Record  @    185.199.109.153
A Record  @    185.199.110.153
A Record  @    185.199.111.153
CNAME     www  oscargdiez.github.io
```

---

## 🚀 Deploy Workflow

Claude ALWAYS provides `jugarenfamilia.html` + `gitinfo.txt` together.
Claude ALWAYS states the version number clearly when presenting deploy files.
Claude ALWAYS deploys immediately after building — no need to ask.
Claude ALWAYS checks with user BEFORE building anything.
Claude ALWAYS confirms a plan first even when Oscar suggests the change himself (plan → Oscar's OK → build). Added Session 24.

### Production deploy
1. Download `jugarenfamilia.html` + `gitinfo.txt` to `C:\Users\User\Downloads\`
2. Double-click `D:\09_ALTO\deploy.bat`
3. Live at jugarenfamilia.es in ~30 seconds ✅

### Staging workflow
- Claude produces `jugarenfamilia_tmp.html` + `gitinfo_tmp.txt`
- Drop both to `C:\Users\User\Downloads\`, run `D:\09_ALTO\deploy_tmp.bat`
- Test at `https://jugarenfamilia.es/index_tmp.html`
- When happy: promote staging to production by copying `index_tmp.html` → `index.html`, bump version (drop `-tmp`), deploy with `deploy.bat`
- Staging is ALWAYS based on latest production — never from old stale staging file
- **Staging deliverables are ONLY `jugarenfamilia_tmp.html` + `gitinfo_tmp.txt`.** `deploy_tmp.bat` does not copy the handoff or the design doc, so those are delivered with the production promotion only (Session 24).
- A long feature can be built as several staging versions and promoted once, after Oscar has tested it in a real multiplayer game (done this way in Session 24).
- **IMPORTANT:** Version check bubble only works on production (fetches `/index.html`), not staging
- **Tools delivery (Session 26):** `deploy.bat` AND `deploy_tmp.bat` also install, if present in Downloads, `ai_scout.js` → `D:\09_ALTO\tools\ai_scout.js` and `tests_kit.zip` → `D:\09_ALTO\tests\` (unpacked to `%TEMP%\alto_kit_tmp` first; installed with `robocopy /MIR` only if `tests\run_all.sh` is inside, otherwise the old tests are kept and an error is shown). Never pushed (`tests/` and `tools/` are in `.gitignore`). Tools-only delivery = install and skip GitHub. Claude delivers tools as exactly `ai_scout.js` and `tests_kit.zip` (zip must contain a top-level `tests/` folder). Both scripts are plain ASCII with CRLF line endings.
- **IMPORTANT:** `gitinfo_tmp.txt` must use `BRANCH=main` — there is no separate `tmp` branch on the remote. Both production and staging deploy to the `main` branch (different files: `index.html` vs `index_tmp.html`).

### gitinfo.txt format
```
REPO=https://github.com/oscargdiez/jugarenfamilia.git
BRANCH=main
NAME=oscargdiez
EMAIL=oscar.g.diez@gmail.com
MESSAGE=description of changes (ASCII only — no emoji or special chars)
```

---

## ⏪ Rollback

Double-click `D:\09_ALTO\rollback.bat` and pick the commit hash.

---

## ⚠️ CRITICAL RULES FOR CLAUDE

**Version bump on every deploy** — update the footer version string.
Format: `vYYMMDD.NN` (e.g. `v260907.156`). Staging gets `-tmp` suffix.
State the version clearly when presenting deploy files.

**Commit messages must be plain ASCII — no emoji or special characters** — they break the git command in deploy.bat.

**Check with user BEFORE building anything. Deploy immediately after building without asking.**

**Font size rule — STRICT:** Only use these sizes: `11px / 13px / 16px / 20px / 26px / 32px` plus intentional hero exceptions (17px for icon buttons, 24px for emoji/icon elements, 28px for stop button, 15px for logo tagline, 48/52/56/80/120px and clamp for hero displays). No new sizes without explicit justification.
- **Session 12 exception:** Daily + Practice buttons use `19px` to fit on one line on small iPhone screens. Emojis inside those buttons use `17px`.

**The home screen buttons and emojis break silently if ANY of these happen:**

### 1. Init calls wiped
`buildEmojiGrid()`, `applyLang()`, `tryRestore()` MUST be the last lines of the `<script type="module">` block.

### 2. T object corruption ← THE MOST COMMON CAUSE
Many strings appear TWICE in the file: once as a value in the T translation object, and once hardcoded in a JS function. When replacing hardcoded strings with `t()`, ALWAYS target the specific JS function context. **Never replace the bare string.**

### 3. JS syntax errors
A single syntax error in the `<script type="module">` block kills ALL buttons and emojis silently. ALWAYS run `node --check` before deploying.

### 4. Non-module scripts are safe
SVG flags and other non-module `<script>` tags in `<head>` are completely isolated.

### 5. applyLang wipes child elements ← LEARNED IN SESSION 10
`s(id, text)` uses `textContent` which destroys child spans. When an element contains both text AND a child span, use `innerHTML` in applyLang instead. Never use `s()` on elements with child spans.

### 6. str_replace context — READ ENOUGH LINES
Always read enough context (10+ lines) to include closing `</div>` tags. Accidentally eating closing divs breaks ALL subsequent screens silently. Always verify screen IDs after structural edits: `grep -c 'id="s-' file.html` should return 12.

### 7. SVG flags — inline use ← LEARNED IN SESSION 11
The FLAGS SVG strings have `display:block` baked in (via the `S` variable). This breaks when used inline with text. The FLAGS object is defined in a non-module `<script>` and exposed as `window.FLAGS`. For inline use, replace `display:block` with `display:inline-block;vertical-align:middle` before injecting.

### 8. inline oninput with value reassignment ← LEARNED IN SESSION 12
Do NOT use `oninput="this.value = ..."` on input fields — the value reassignment suppresses subsequent `oninput` events on both iOS and Windows. Use `addEventListener('input', ...)` in JS instead, with `setSelectionRange` to preserve cursor position.

### 9. Claim pill visibility ← LEARNED IN SESSION 12
The claim pill uses CSS-only show/hide (`#claim-pill { display:none }` / `#claim-pill.free, .claimed, .reserved { display:inline-block }`). Never add inline `style` to the pill button — it will override the CSS and break show/hide. JS only sets `className`.

### 10. finishValidation bases scores on preRoundScores ← LEARNED IN SESSION 14
`finishValidation` always starts from `room.preRoundScores` (not `room.scores`) to prevent double-counting when host goes back to validation and recalculates. `goBackToValidation` does NOT write scores to Firebase — scores remain visible to guests during re-validation.

### 11. _playedLangs cache — do not read before fetchPlayedState resolves ← SESSION 15
`_playedLangs` is a module-level object `{ es: bool, en: bool, fr: bool }` populated async by `fetchPlayedState(safeName)`. It is empty on page load until the Firebase read completes. `updateDailyButtonDate()` falls back to localStorage (`alto_daily_{date}_{lang}`) as a fast same-device path while the fetch is in flight. Never treat `_playedLangs[lang] === undefined` as "not played" without also checking localStorage.

### 12. Recontest goes through Cloudflare Worker ← SESSION 16
The daily recontest (`window.recontest`) calls `https://api.oscar-g-diez.workers.dev/` exactly like `askAI` — NOT the Anthropic API directly. The Anthropic API only works in Claude artifacts context, not in the live game (CORS/auth would fail).

### 13. Democratic validation stale state fix ← SESSION 16
When host transitions to validate phase, Firebase update now includes `invalidAnswers: {}, votes: {}` alongside `phase: 'validate'` — atomic clear for all clients. `goBackToValidation` already did this. First-entry path now does too.

### 14. G_daily.aiResults key format ← SESSION 18
`G_daily.aiResults` keys become strings after JSON round-trip (`"0"`, `"1"` etc). Always write with `String(idx)`. When reading, use `G_daily.aiResults[String(i)] || G_daily.aiResults[i]` to handle both numeric and string keys safely.

### 15. Recontest score delta bug — fixed in Session 18
Old code added `addedPts` (100 or 50) directly to `baseScore` which was in raw units — causing 10× score inflation in display. Always recount from scratch: iterate all answered slots, count valid/unsure from `G_daily.aiResults`, recompute `baseScore` and `totalScore` from zero.

**Mandatory pre-deploy checklist:**
- Version string updated ✅
- File size < 300KB ✅ (currently ~310KB — watch for growth)
- Init block present (`buildEmojiGrid`, `tryRestore`) ✅
- No `-tmp` in version string for production ✅
- All 12 screen IDs present (`grep -c 'id="s-'` = 12) ✅
- JS syntax clean (`node --check`) ✅
- Test kit green (`D:\09_ALTO\tests\run_all.sh <file>` → `ALL GREEN`) ✅ — added Session 24, see the test kit section
- (File size: over 300KB since Session 23 — Oscar confirmed not a concern; v260926.311 is 351KB)

**Always start from the uploaded working file** — never from a local copy that may have drifted.

### Test kit (Session 24)
`D:\09_ALTO\tests\` (not committed — `deploy.bat` only adds named files; it travels in the project zip). Claude runs it in the sandbox: `./run_all.sh ../index_tmp.html`. Five Node suites (130 checks) extract the real functions from the HTML by name and run them with mocks; `restore/restore_scenarios.py` (19 checks) runs the real HTML in headless Chromium with a fake Firebase module shared between browser contexts (normal window vs incognito) and tabs. Run it before and after every build; extend it with each build. See `tests/README.md`. Also useful: rendering screens with the real CSS + real fonts (`npm pack @fontsource/caveat @fontsource/special-elite`) at 414/390/375/360/320px to check iPhone fit before shipping.
**Session 25 test kit changes:** new `suites/test_build5a_groups.mjs` (45 checks: Mis grupos rules, default names ES/EN/FR, endGame writes, history record rebuilds awards, trim to 20, yearly cleanup, practice/no group/double tap, write failure still reaches Final); new `restore/groups_scenarios.py` (28 real-browser checks: default group, picker new/rename/switch, guest line live, refresh keeps group, full game lands in groupsLb/history, guest-as-host preselects the family group, rename from another device, practice not counted); `suites/test_build1_scoring.mjs` + `test_build4_final_share.mjs` updated for the 5a code; fake Firebase module now supports `query/orderByChild/startAt/endAt/limitToFirst`; `run_all.sh` now fails a crashed suite (it used to print ALL GREEN when a suite crashed) and runs every `restore/*_scenarios.py`. **The updated suites need the 5a code — against production v313 use the Session 24 kit from the zip.** Totals on staging v260926.314-tmp: 178 unit + 28 groups + 19 restore = ALL GREEN.
**Tools (Session 25):** `D:\09_ALTO\tools\ai_scout.js` — paste into the Console on jugarenfamilia.es: reads OpenRouter's live free text-model list + `AI_MODELS`, asks each model 14 known questions (from the 29 Sep daily) twice with exactly the daily prompt, prints a ranked table (score, unsure, flips, avg time, status, misses) and a suggested `AI_MODELS`.
**First thing every session — make a backup:** `cp index.html index_backup_sN.html` before any edits.

---

## 🎨 Typography System

Two fonts, six slots. **Do not add new sizes outside these slots.**

| Slot | Font | Size | Use |
|---|---|---|---|
| xs | Special Elite | 11px | Muted metadata, version string, tiny labels |
| sm | Special Elite | 13px | Standard labels, scores, badges, secondary UI |
| md | Special Elite | 16px | Section headers, screen titles, name input |
| body | Caveat | 20px | Player names, secondary content, `.lbl` labels |
| primary | Caveat | 26px | Answers, gameplay text, answer inputs, name fields |
| hero | Caveat | 32px+ | Timer numbers, letter display, room code |

**Session 12 exception:** Daily + Practice home screen buttons use `19px` (between md and body slots) to fit English text on one line on small iPhones. Documented in HTML comment.

**Role assignment:**
- **Special Elite** → all UI chrome (labels, scores, buttons, navigation, metadata)
- **Caveat** → all player-generated content + category labels in playing/result screens

---

## 🏷 Label Casing Rules

| Type | Rule | Examples |
|---|---|---|
| Buttons (tappable actions) | Sentence case | `Crear sala`, `Salir de la sala`, `← Volver` |
| Inline hints / sublabels | lowercase | `comparte el código`, `nombre del grupo` |
| Toasts | Sentence case | `Escribe tu nombre primero` |
| Decorative / badge text | ALL CAPS | `MULTIJUGADOR · CON FAMILIA Y AMIGOS` |

---

## ✅ Features Built

### Core Game
- Real-time multiplayer via Firebase (room codes, host/guest model)
- Session restore / welcome-back screen with emoji picker
- Collision-safe room code generation (`genUniqueCode()`)
- Room auto-deleted from Firebase 45 minutes after game ends (extended from 60s in Session 22). Delete timer stored in `G._deleteRoomTimeout` and cancelled if host taps "Jugar de nuevo".
- Letter selection (easy pool only — hard letters option hidden, always easy)
- Language-aware easy pool: EN keeps K, ES/FR drop it (`LETTERS_EASY` object)
- Correct remaining timer for rejoiners (`roundStartTime` saved in Firebase)
- Answer submission, ¡Alto! button
- ¡Alto! guard: all answers must have 2+ characters before Alto can be called (toast in all 3 langs)
- Accent-insensitive duplicate detection (`normalize()`)

### Lobby Config Persistence (Session 16)
- All lobby settings saved to `alto_lobby_config` in localStorage on every change
- Restored on `enterLobby` before `buildThemePicker` — host sees their last setup every time
- Saved: theme, selectedGroups, rounds, time, penalty, validationMode, aiStrictness, aiSpelling

### Category Modes (Session 16)
Three theme modes replacing the old 7 themes:
- **🎲 Clásico** — fixed 8 classic categories (Nombre, Apellido, Ciudad, País, Animal, Objeto, Fruta, Color), shown as non-interactive pills when selected
- **✏️ Categorías** — 13 group toggle pills (all active by default), picks 8 random categories per round from `DAILY_CAT_DATA` filtered by active groups + letter validity. Min 3 groups enforced. Fresh pick every round via `nextRound`. Groups saved to Firebase as `selectedGroups`. Categories translated via `getDailyCatNames()`.
- **✍️ Libre** — free text, one category per line, min 2

**CAT_GROUPS** — 13 groups with emoji + ES/EN/FR names: animales, geografia, comida, arte_musica, historia_cultura, ciencia, deporte, ciudad_urbanismo, naturaleza, hogar_objetos, identidad_sociedad, entretenimiento, marcas.

**`pickRandomCats(letter, groups)`** — filters DAILY_CAT_DATA by active groups and letter validity, shuffles, returns 8 keys. Returns null if pool < 8 (toast shown). No max-1-per-group constraint for multiplayer.

**`room.catMode`** — `'random'` or `'fixed'`, written to Firebase. `nextRound` checks this and picks fresh categories when `'random'`.

### Alto Caller Penalty
- Caller penalised -50pts if they have any invalid/empty answer
- Will be replaced by sliding bonus/penalty scale (see Flagged for Future)
- **REPLACED in v260926.311 (Session 24):** caller gets `round10(250 x categories/8) - 100 x invalid`; lobby penalty slider removed. See Session 24.

### Validation Screen (host)
- 📖 Wikipedia lookup (host + guests, language-aware)
- 🤖 Robot validation via Cloudflare Worker → OpenRouter (fallback chain, 20s timeout)
- Robot result shows coloured verdict: green=válido, red=inválido, amber=no sé (all 3 langs)
- Automatic validation ON by default, Estricta by default
- 👎 Per-entry voting (democratic mode only — thumbs down only, answers valid by default)
- 🔥👏😂😬🤬 Per-entry emoji reactions (5 emojis, fire first)
  - **Now (v260926.311):** 4 emojis 🔥👏😂😬, one per answer per player, none on your own (faint + toast), points 🔥30 👏20 😂10 😬0 when the lobby toggle is on. See Session 24.
- 🛑 Stop caller banner: shows on host validate AND guest waiting screen
- 🗳️ Democratic mode: majority 👎 votes auto-invalidates. Valid by default. Min 3 players enforced.
- ← Revisar: undo scoring — scores stay visible to guests during re-validation, recalculates cleanly from preRoundScores
- **Stale state fix (Session 16):** `phase:'validate'` transition now atomically clears `invalidAnswers:{}` and `votes:{}` in Firebase — guests no longer see stale invalidations from previous rounds

### Guest Waiting Screen
- Title "Esperando..." / "Waiting..." / "En attente..."
- "el anfitrión está revisando ✏️" with bouncing pencil inline
- Stop caller banner shown below the text
- Guest sees full validation content below (read-only)
- When host goes back to validation from scores: banner "el anfitrión está revisando de nuevo ✏️" shown on scores screen, scores frozen (not re-rendered)

### Daily Challenge
- **58 categories across 13 groups** with valid-letter strings per language (ES/EN/FR)
- **Per-language seeded daily** — ES/EN/FR each get their own letter + 6 categories
- Letter picked from `LETTERS_EASY[LANG]` — EN gets K, ES/FR don't
- Max 1 category per group per day
- Only categories valid for today's letter in today's language
- 90s countdown timer with speed bonus (1.0–2.0× multiplier)
- Robot consensus validation (2 models parallel, tiebreaker if split)
- Scoring: valid=100pts, invalid=0pts, unsure=50pts (displayed correctly ×10)
- Speed breakdown: `base 500 pts ×1.12⚡→ 560 pts +200 originalidad✨`
- In-progress save/restore (page refresh safe) — only saves after timer starts
- Timer stops when tab is backgrounded (visibilitychange handler)
- Categories saved to Firebase on submit
- Daily leaderboard with lang switcher (ES/EN/FR SVG flags)
- **Per-language localStorage keys** — `alto_daily_{date}_{lang}` — play in all 3 langs independently
- Originality bonus: +50pts if answer unique among all players (flat — to be overhauled, see Flagged for Future)
- Originality shows on initial load (not just after flag tap)
- Originality badges (+50✨) shown per answer in ALL players' panels
- Accent-insensitive originality check
- Letter validity sanity check on in-progress restore
- Double-submit race guard (`_submitting` flag)
- PlayerKey has random suffix to prevent collision
- **Test mode** — name starting with `__` skips Firebase write + localStorage save, can replay unlimited
- **DAILY_OVERRIDES** — add entries keyed by `YYYY-MM-DD` to override normal picker for themed days
- **Submit guard** — blocks submit if zero answers filled (toast in all 3 langs)
- **Cross-device played state** — `names/{safeName}/played/{date}/{lang}: true` written on submit. `fetchPlayedState()` reads this on load and on name change, caches in `_playedLangs`. Hecho label is per-lang and device-independent.
- Daily button: orange tinted outlined style, compact date (`7 sep`), 19px font
- **Daily recontest (Sessions 16+18):** `¿Error?`/`Error?`/`Erreur?` text link shown next to ❌ verdicts on result screen. Tapping runs 3 model calls in parallel via Cloudflare Worker. 2-of-3 must agree valid to overturn. Max 1 contest per lang per day (stored in `names/{safeName}/contests/{date}/{lang}`). Only shown for today's result (not historical). Practice mode blocked. If overturned: ❌→✅, pts recalculated from scratch. If upheld: button removed, toast shown. See Session 18 for full rewrite details.

### Daily Recontest — Architecture (Session 18 rewrite)
- **Practice mode guard:** `if (G_daily.practiceMode) return;` — recontest blocked in practice
- **Prompt:** benefit-of-the-doubt — "Give the benefit of the doubt — if the answer could reasonably be valid, overturn it"
- **`parseVerdict`:** DUDOSO returns `'valid'` (benefit of the doubt)
- **Threshold:** 2-of-3 models must return valid to overturn
- **Score recount:** full scratch recount — iterates all answered slots, counts valid/unsure from `G_daily.aiResults`, recomputes `baseScore` and `totalScore` from zero. No deltas.
- **`speedMultiplier`:** read from `G_daily.speedMultiplier || (G_daily.totalScore / G_daily.baseScore) || 1`
- **Firebase written first:** `await update()` before any UI changes. Inner try/catch reverts `G_daily.aiResults[String(idx)]` and re-enables buttons on failure.
- **Firebase payload on overturn:** `aiResults/${String(idx)}`, `baseScore`, `totalScore`, `speedMultiplier`
- **localStorage updated** after Firebase success, including `speedMultiplier`
- **`aiIcon` map:** `{ valid:'✅', unsure:'🤔', invalid:'❌' }` — `invalid` was previously missing
- **Toast:** uses `toastPts = (newBaseScore - oldBaseScore) * 10`
- **`loadDailyLeaderboard`** called at end — triggers originality recompute which reads the now-correct `G_daily.totalScore`

### speedMultiplier — full state (Session 18)
- ✅ Calculated on submit: `1.0 + (secsLeft / totalSecs)`
- ✅ Saved to `G_daily.speedMultiplier` on submit
- ✅ Saved to localStorage on submit
- ✅ Saved to Firebase on submit (new Session 18): field `speedMultiplier` in scores entry
- ✅ Read from Firebase on cross-device path: `entry.speedMultiplier || (entry.totalScore / entry.baseScore) || 1`
- ✅ Written to Firebase on recontest overturn (new Session 18)
- ✅ Written to localStorage on recontest overturn (new Session 18)

### Practice Mode (Session 12)
- Solo daily-style play — same flow as daily (countdown, 90s, AI validation, score)
- Random letter + 6 categories each time (no date lock, unlimited replays)
- No Firebase write, no localStorage save, no leaderboard
- Originality scoring disabled (no Firebase data to compare against)
- Button: outlined, same row as Daily, 19px font, `✏️ Práctica`
- Result screen shows "Práctica" title, hides leaderboard card
- `resetGDaily(practiceMode)` fully resets all G_daily fields on entry
- `initDaily()` extracted as named function, shared by Daily and Practice

### Name Claim System (Session 12)
- **Pill** on home screen name input — appears after 800ms debounce
  - 🟠 Orange: `¡Disponible! Resérvalo →` — name unclaimed
  - 🟢 Green: `✓ Nombre reservado` — claimed by you (verified on this device)
  - 🔴 Red: `¿Eres tú? Verifica →` — claimed by someone else
- **Overlay screen** — friendly copy, GDPR reassurance, email or 4-digit PIN
- **Firebase path:** `names/{safeName}/secret` (SHA-256 hash) + `names/{safeName}/displayName`
- After verify: restores original accented display name from Firebase into input field
- localStorage key: `alto_claim_{safeName}` — never prompts again on this device
- Name normalisation: lowercase + accent-strip for key, original preserved for display
- Fully translated ES/EN/FR, pill text updates on language switch

### Update Bubble (Session 12)
- Fetches `/index.html?_=timestamp` every 5 min + on tab focus + on window focus
- Compares version string — shows red pill bottom-right when newer version available
- Only shows on home screen (`G.screen === 'home'` guard)
- Tap → `window.location.reload(true)`
- iOS note: takes longer to appear due to Safari caching (works, just slower)
- CSS: opacity transition (not display:flex toggle) for iOS compatibility

### UX/Polish
- Fixed shell layout: logo (left), letter+round+timer (center), room+? (right)
- Logo click goes home on ALL screens
- Floating timer numbers (spawn from edges, accelerate as time runs out)
- SVG flags — identical on Windows, iOS, Android, Mac. `window.FLAGS` exposed globally.
- WhatsApp share: icon-only button on same row as link + copy button
- 720px max-width for desktop comfort
- Solo hint in lobby: disappears when 2nd player joins
- Name input: auto-capitalises first letter, lowercases rest, restores on page refresh
- `🎲 Clásico. Crear sala →` button label in all 3 langs
- `¡Hecho! Ver puntuación` / `Done! See score` / `Fait ! Voir le score` already-played label
- **Floating reaction cleanup (Session 16):** `stopReactions()` clears `reactions-stage` DOM and resets `G_shownReactions` set on `enterLobby`, `nextRound`, and `enterPlaying`
- **Quick join name fallback (Session 16):** pre-fills from `alto_session.name` OR `alto_name`

### Languages
- 🇪🇸 ES 🇬🇧 EN 🇫🇷 FR
- Full UI + categories + themes + rules translated
- All AI/IA references replaced with Robot throughout all 3 languages
- **Rule for new features:** always add translations for all 3 languages immediately

### Debug Mode (overhauled Session 14)
- Type `__debug__` as player name → debug bar at bottom
- **📱 iPhone mode toggle** — constrains viewport to 390×844 phone frame with notch, all fixed elements scoped inside
- Screen buttons grouped into rows: MULTI / DAILY / OVERLAY
- Zero Firebase calls — fully offline
- Test mode for daily: name starting with `__` (not `__debug__`) — skips Firebase + localStorage

---

## 🐛 Known Issues / Watch List

- OpenRouter free tier models rotate without warning — if Robot breaks, check openrouter.ai logs
- Session restore on same device/browser: host and guest share localStorage — **FIXED v260926.309 (Session 24):** per-tab session in sessionStorage + `?room=` in the address
- Font sizes: Caveat x-height smaller than Special Elite — visually looks different at same px
- iCloud Safari sync: iPhone + iPad share localStorage if Safari sync enabled — player may not be able to replay on second Apple device
- Update bubble on iOS appears slower than on Windows (Safari caching) — working correctly, just slower
- Unverified name label not yet implemented — dropped in favour of registration gate (see Flagged for Future)
- **File size ~310KB** — significantly over 300KB soft limit. Growing. Watch carefully each session.
- `daily/{date}/{lang}/players/{safeName}` legacy index still being written on submit — kept for backfill compatibility, can be retired once `names/{safeName}/played/` has been live for a few weeks
- Emoji picker grid overflows its container at non-100% zoom on Windows — needs flex-wrap and relative sizing (flagged for future session)
- **(Session 25)** A contest (`recontest`) asks the SAME model again (Ling, temperature 0), so it rarely overturns; any error or unclear reply counts as "upheld" and still uses up the contest. The code comments mention "1 of 3" but it is 1 call to 1 model.
- **(Session 25)** Paid Ling answers are not fully consistent between calls (two providers) — see OpenRouter section.
- **(Session 25)** `#qj-name` (invite-link name field) still uses inline `oninput` with value reassignment — breaks rule 8; pre-existing, not fixed yet.
- **(Session 25)** `lbRef()` (`leaderboard` path) is dead code — never called.
- **(Session 26)** Judge prompt changed in v320 (language rule). Any future prompt change: update `tools/ai_scout.js` too — `test_judge_prompt.mjs` fails if they differ. **(Session 27)** Changed again in v353 (rule 4 sound-alike misspellings, rule 6 category fit, contest category sentence).
- **(Session 26)** Mixed versions: during a staging test all players must be on `index_tmp.html` — Revisar-for-guests and the tap fix run on each client.

## 🗒 Small Fixes Backlog
- Quick join accent restoration — if pre-filled name matches a verified claim, fetch `displayName` from Firebase and restore accented version into field

---

## 🗺 Flagged for Future

### Security — from the Session 25 review (agreed with Oscar)
Context: family game, no payments, little personal data — nothing urgent, but these are real. Order agreed:
1. **Done / no code:** credit limit on the OpenRouter key (set by Oscar 29 Sep). Oscar to export a Firebase backup now and then (Realtime Database → Data → ⋮ → Export JSON) — the free Spark plan has no automatic backups.
2. **Small fix, no hurry (Oscar: "then"):** multiplayer 🤖 result is stored in the room as ready-made HTML (`rooms/{code}/aiResults/{key}/html`) and guests render it with `innerHTML` (handleRoom, `el.innerHTML = data.html`) → anyone who knows a room code can inject HTML/script into every guest's page. Fix: store only `cls` (valid/invalid/unsure) and let each device build the label locally in its own language. Same pass: check every `innerHTML` that shows player names/answers uses `escHtml` (Marcador names already fixed in Build 5a).
3. **"Security build" after Build 5, together with AI step 1:**
   - **Firebase is fully open** (anyone can read/change/delete everything — proven in Session 25 with Console scripts). Fix: Firebase Anonymous Auth (invisible, free) + real rules: players change only their own entries; daily scores write-once (no overwrite/delete); type/size validation; no deleting whole nodes. Needs a careful migration plan (existing data has no owner ids).
   - **Firebase App Check** (reCAPTCHA v3) so only the website talks to the database — an extra layer, not a replacement for rules.
   - **Name claims are weak:** `names/{safeName}/secret` = unsalted SHA-256 of a 4-digit PIN or an email, publicly readable → a PIN is found by brute force in well under a second; anyone can also just overwrite `secret` (open rules). The claim screen says the email is not stored, but its hash is (publicly). Fix with the auth/rules work: secret not readable, longer PINs, ideally stop asking for emails; update the claim privacy text.
   - **Cookie consent:** the site loads Google Analytics (gtag `G-V2VJ4X7M5Z`) and has no consent banner. **Oscar agreed to a small consent banner** (ES/EN/FR, analytics only loads after "accept"; remember the choice). Not legal advice — keep it simple and standard.
   - Fine as is: Firebase `apiKey` in the page (not a secret), OpenRouter key only in the worker, HTTPS.

### AI model management — AFTER BUILD 5 (agreed with Oscar, Session 25)
**Step 1 — the worker controls the model list (security fix + no game update to change models).** The worker keeps the active list in Cloudflare KV and replaces `model`/`models` on every request with its own list (also caps `max_tokens`); the game keeps `AI_MODELS` only as a last resort. Closes the "anyone can run any model on our credit" gap. Oscar pastes the new worker code in Cloudflare and creates the KV binding.
**Step 2 — daily self-check + automatic scouting (optional, after step 1).** Cloudflare Cron Trigger once a day: ask the current first (free) model 3 known test questions; if it fails (404/429/wrong), read OpenRouter's free text-model list and test candidates one at a time, STOP at the first that gets all test questions right, put it first in the KV list (paid Ling always stays last). Log to Firebase (date, what failed, new model) and show it in debug. Check first: free-plan CPU limit per run (parsing the full model list may exceed it; $5/month plan would cost more than the paid Ling saves).
**Related small items:** pin one provider for Ling (OpenRouter `provider` routing) so identical answers get identical verdicts; make contests ask a DIFFERENT model than the first judgment (real second opinion); scout script improvements — skip a model after 4 rate-limited (429) calls in a row, and print OpenRouter's error message (so "gone", "blocked by privacy" and "busy" can be told apart); if the game prompt ever changes, regenerate the scout from the game code.

### Step 2 — Same-device reload reads from Firebase (next priority)
**What:** when a player reloads the page after having played, the result screen should read from Firebase instead of localStorage, so any recontest overturn by another device is reflected.
**File:** `index.html` — already-played localStorage path (~line 2351)
**Change:** if localStorage key exists, read `playerKey` from it, fetch `daily/{date}/{lang}/scores/{playerKey}` from Firebase, pass Firebase data to `showDailyResult`. Fall back to localStorage if fetch fails or `playerKey` missing.
**Loading state:** show brief "cargando…" while Firebase fetch runs — don't flash blank screen.
**Verify:** play the game, reload the page, confirm result screen shows Firebase data. Check fallback works.
**Risk:** MEDIUM. Extra Firebase read per returning visit. Adds latency — needs loading state.

### Registration Gate (discussed Session 15 — NOT YET BUILT)

**Decision:** require a registered name (green pill — email or PIN verified) to play Daily, Practice, or Multiplayer. Any mode. No exceptions.

**What this enables:**
- Played state (`_playedLangs`) is always meaningful — no anonymous players to track
- `daily/{date}/{lang}/players/{safeName}` dedup index becomes redundant — can be dropped
- Score restoration on Device 2 is guaranteed by safeName — no fallback logic needed

**UX flow:**
- Name field typed → pill debounces → orange/green/red
- Any game entry point tapped without green pill → claim overlay opens with a one-line explanation, not a blocking error toast
- After successful registration → proceed directly into the game they tried to start
- Multiplayer: same gate on `createRoom` and `joinRoom`

**Strings to write (all 3 langs):**
- Overlay subtitle when opened as a gate (vs voluntarily): e.g. `"Reserva tu nombre para empezar — solo tarda un momento"` — friendly, not punishing

**Code locations to change:**
- `startDailyChallenge()`, `startPractice()`, `createRoom()`, `joinRoom()` / quick join — if `_claimState !== 'mine'` → `claimOpen()` instead of toast
- `claimOpen()` — needs optional `reason` parameter for different subtitle

**External copy to update (BEFORE shipping):**
- `<meta name="description">` (line ~16): currently `"Multijugador, sin registro, gratis."` → needs rewrite
- `<meta property="og:description">` (line ~18): currently `"No sign up needed. Just share a link and play!"` → needs rewrite
- `<meta name="twitter:description">` (line ~33): same as OG → needs rewrite
- `claim-privacy` overlay text: currently framed as reassurance for optional action — needs reframe as benefit copy

**Firebase cleanup once shipped:**
- Stop writing `daily/{date}/{lang}/players/{safeName}` index
- Remove backfill logic from `fetchPlayedState()`

---

### Multiplayer Scoring & History Overhaul (BIG FEATURE)

#### New scoring system:
- Valid unique answer → 100pts
- Valid duplicate answer → 50pts
- Originality bonus → +50pts flat
- Invalid → 0pts
- **Alto caller sliding bonus/penalty:**
  - 0 invalid → +150pts, 1 → +50pts, 2 → 0pts, 3 → -100pts, 4 → -200pts, 5 → -300pts, 6 → -400pts

#### Scores/result UI overhaul:
- Per-player expandable cards showing answers, validity, originality, emoji reactions
- Alto bonus/penalty clearly labelled
- Round scores + cumulative total separated
- Share button consistent with daily share format

#### Game history:
- Firebase path: `history/{gameId}` with ~2 week retention
- Per-round data, final scores, group name, host badge
- Navigate with ◀ ▶
- Per-player stats accumulate over time

### Daily Originality Overhaul
- Truly unique → `round(50 × log10(playerCount))` pts
- Rare (<5%) → flat +25pts
- Common (≥5%) → no bonus

### Botanical / scientific answers (Session 26, parked by Oscar)
Tomate as Fruta was rejected in Estricta (borderline). Oscar thinks it should count. Options: A) prompt rule "scientifically correct answers count" (then pimiento, pepino, calabaza, aguacate... also count as fruit); B) only well-known facts (tomato = fruit, whale = mammal); C) leave it to the host (current). Any prompt change → update the scout and re-run it.

### Daily Recontest — future tuning
- Current: 3 models in parallel, 2 of 3 majority to overturn, 1 contest per lang per day, benefit-of-the-doubt prompt, DUDOSO counts as valid
- If too lenient in practice: tighten prompt or require all 3
- If too strict: already relaxed significantly in Session 18

### Automatic AI Multiplayer Mode
- Third validation mode: fully automatic Robot validation, no host review step

### iOS Layout Refactor
- Replace `position:fixed` shell with true fixed layout
- Eliminates iOS Safari keyboard viewport resize bug

### Other
- Language as lobby setting (currently global)
- Public rooms / Tournaments
- Background soundtrack + sound effects
- Letter reveal animation
- Favourites/friends list

---

## 📊 AI Prompt Tuning via Score Data (future — data accumulating now)

Firebase already stores everything needed to study AI mistakes and tune prompts:
- `daily/{date}/{lang}/scores/{playerKey}` — answers, aiResults, categories, letter, safeName
- `names/{safeName}/contests/{date}/{lang}: N` — which answers players felt strongly enough to challenge

**The tuning loop:**
1. Pull all score entries → flatten to rows of `(category, letter, answer, aiVerdict)`
2. Cross-reference with contest data — contested answers that were overturned by 2-of-3 models = confirmed AI mistakes
3. Use those as a labelled dataset to identify prompt weaknesses
4. The recontest prompt is already separate from the main validation prompt — tune independently

**What's missing to make this complete:**
- A `contestOutcome: 'overturned'|'upheld'` field written to the score entry when a recontest completes — currently the score updates but no explicit outcome flag is stored
- Add this when building the tuning pipeline

**AI validation analysis (Firebase export Sep 2–16):**
- 813 answers analysed across all 3 langs
- Overall invalid rate: 31% (ES 30%, EN 29%, FR 35%)
- Worst categories: Película animada (47–80% invalid), Árbol ES (71%), Type de magasin FR (71%)
- Clear false positives: Frank Sinatra, Fahrenheit, Paris, Rumba, Oboe, Jazmin, Marguerite, Foucault, Finding Nemo, Matelas
- Decision: leave prompt as-is (v001), rely on recontest as safety valve
- `DAILY_CONSENSUS` left as `false` (single model) — monitor before changing

---

## 📝 Session Log

### Session 1 (Aug 29)
Initial build: multiplayer, Firebase, rooms, scoring, themes, 6 languages

### Session 2 (Aug 30)
Quick join, guest lobby card, per-entry reactions/votes, Wikipedia lookups, flying emojis, AI validation, democratic mode, deploy/rollback scripts

### Session 3 (Aug 31)
Full audit — 15 bugs fixed. Room cleanup, collision-safe codes, stop caller banner, AI fallback chain, timer on rejoin, debug mode

### Session 4 (Sep 1)
AI button/results hidden when off, validation entry layout, debug controls, SVG flags, exhaustive i18n pass

### Session 5 (Sep 1)
AI validation fully fixed and tuned. Relajada/Estricta modes. Cloudflare Worker proxy. 3 languages (ES/EN/FR).

### Session 6 (Sep 2)
Fixed shell layout. Home screen refresh. Lobby redesign. Rules panel. Urgency pulse.

### Session 7 (Sep 3)
Validation screen layout fix. Countdown before each round. Daily Challenge promoted to production. Democratic mode overhaul. AI result sharing.

### Session 8 (Sep 4/8)
Floating timer numbers. Round timer bug fix. Validation row overhaul. Guest AI button. Cloudflare Worker for OpenRouter key. Dropped IT/DE/PT.
**Last version: v260908.17**

### Session 9 (Sep 5)
Daily challenge: AI icons + pts in leaderboard, speed multiplier, scores x10, originality fix, categories saved to Firebase, lang switcher, 4 bug fixes, double-submit guard.
Multiplayer: stop caller banner fixed, logo click all screens.
Typography overhaul: 6-slot system, Special Elite for UI, Caveat for content, 720px max-width.
**Last version: v260905.49**

### Session 10 (Sep 6)
UX & Polish: solo hint, name inputs, group name hint, WhatsApp button, easy letters hidden, home screen compacted.
Waiting screen overhaul. Validation screen polish.
Bug fixes: stop banner guests, broken HTML, applyLang child spans, duplicate IDs, CSS variables.
**Last version: v260905.85**

### Session 11 (Sep 6)
Category system overhaul: 58 categories / 13 groups, per-language seeded daily, letter validity per language, max 1 category per group per day, EN/FR translations for all 58 categories.
Daily challenge fixes: null startTimestamp bug, timer negative fix, per-language localStorage keys, originality on initial load, daily rules scoring x10.
New features: test mode, DAILY_OVERRIDES infrastructure.
UI/Language fixes: Robot throughout, error messages translated, button casing, daily button size.
**Last version: v260906.107**

### Session 12 (Sep 7)
Practice Mode, name claim system, cross-device daily dedup, update bubble, home screen UX, name input auto-capitalise, daily submit guard.
**Last version deployed: v260907.156**

### Session 13 (Sep 8-9)
Reserved name Firebase identity, historical daily leaderboard, emoji independence, share result, multiplayer display name restore, version check before game start, no-name guard, same-device replay block, cross-device played check, timer fixes.
**Last version deployed: v260907.195**

### Session 14 (Sep 12-13)
Debug bar overhaul, emoji reactions 5→3, democratic thumbs-down only, optimistic vote UI, Alto guard, scores freeze on back-to-validation.
**Last version deployed: v260912.218**

### Session 15 (Sep 13)
Cross-device played state overhaul (Firebase names path, fetchPlayedState, backfill), registration gate documented, session 15 fixes deployed.
**Last version deployed: v260913.219**

### Session 16 (Sep 13)
Democratic stale state fix, quick join name fallback, floating reaction cleanup, category modes overhaul (3 modes), lobby config persistence, classic category pills, 5 entry reactions, daily recontest (first version), leaderboard spoiler guard, AI prompt tuning documented.
**Last version deployed: v260913.243**

### Session 17 (Sep 14)
Leaderboard spoiler guard fix: switching to an unplayed lang hides answer panels and chevrons, shows play-to-see-answers hint. Scores still visible.
**Last version deployed: v260914.251** *(note: handoff from this session was severely truncated — this was the root cause of context loss in Session 18)*

### Session 18 (Sep 16)
- **speedMultiplier saved to Firebase** on submit (was localStorage only)
- **Cross-device path** updated to read `entry.speedMultiplier` first, derive as fallback
- **Recontest full rewrite:** practice mode guard, benefit-of-the-doubt prompt, DUDOSO→valid, scratch recount, Firebase-first write with rollback on failure, localStorage update includes speedMultiplier, aiIcon map includes invalid, correct toast pts, String(idx) keys throughout
- Handoff restored from Session 16 version after Session 17 truncation
**Last version deployed: v260916.252**

### Alto button — require all categories filled (flagged Sep 17, needs design decision)

**Problem:** current speed multiplier formula `1.0 + (secsLeft / totalSecs)` can be unfair. A player who answers 3 categories and calls Alto early can outscore a player who answered 5 categories but took longer. Speed bonus shouldn't reward abandoning categories.

**Proposed fix:** block the Alto button until all categories have 2+ characters (consistent with existing 2-char guard). This removes the strategic incentive to call Alto early with fewer answers.

**Design question not yet resolved:** what happens if a player genuinely can't think of an answer for one category?
- **Option A — Strict:** must attempt every category (even if invalid). Simple, no new UI.
- **Option B — Skip button per category:** marks category as intentionally blank, counts as "filled" for Alto guard. Cleaner UX but adds complexity.
- **Option C — Speed bonus only if all categories filled:** keep Alto available anytime, but speedMultiplier = 1.0 if any category blank. No UI change needed.
- **Option D — Per-answer speed bonus:** `mult = 1.0 + (secsLeft / totalSecs) × (answered / totalCats)`. Proportional reward.

**Current preference:** Option A (strict) or Option C (no mult if blank) — need to decide before building.

**Also related:** Scolopendre (myriapode) marked invalid as "Insecte ou arachnide" FR — correct rejection. Stellaire marked invalid as "Fleur" FR — likely false negative (Stellaria is a real flower). Both observed Sep 17.

### Session 18 — additional deploys (Sep 17)

After initial v252 deploy:
- **v260916.253:** recontest prompt tightened — unambiguous category match required, spelling mistakes not grounds to overturn, DUDOSO upholds original invalid verdict
- **v260916.254:** contest log written to Firebase (`names/{safeName}/contests/{date}/{lang}_log`), playerKey stored in Firebase played path (instead of `true`), fetchPlayedState preserves playerKey string, Step 2 fast path implemented (cross-device restore fetches score directly by playerKey, falls back to safeName scan for legacy)

**Last version deployed: v260916.254**

### Grammar-based categories (concept, flagged Sep 17)

**Idea:** add grammar categories to the category pool — Verbo, Sustantivo, Adjetivo, Adverbio, Sustantivo abstracto, etc.

**Why appealing:**
- Grammar categories are more objective than semantic ones — "is this a verb?" is a cleaner AI validation question than "is this a monument?"
- Fewer false negatives/positives expected
- Adds variety and difficulty to the game

**Key design constraint:** category viability varies by letter. Some work for almost any letter (Verbo, Sustantivo), others would be very hard for certain letters (Adverbio con F in French, for example). Category definitions would need to specify which letters they're valid for — similar to how DAILY_CAT_DATA already has per-language letter validity strings.

**Before building:** need to design the full list of grammar categories and map out letter validity per language (ES/EN/FR) carefully. This is a design task before a coding task.

**Likely approach:** mixed mode — grammar categories as occasional entries in the existing pool rather than a grammar-only mode, so they appear alongside semantic categories like Animal or Ciudad.

### Session 18 — further deploys (Sep 17, continued)

**v260916.255:** cap `secsLeft = Math.max(0, G_daily.secsRemaining)` at submit time — prevents sub-1.0 speed multiplier from JavaScript `setInterval` timer jitter (confirmed by historical ×0.97 score). Timer jitter can cause `secsRemaining` to go negative on slow/busy devices.

**v260916.256:** speed multiplier display now reads `entry.speedMultiplier` / `G_daily.speedMultiplier` directly in all 4 places (leaderboard `speedTag`, `computeAndUpdateOriginality` speed text, share result text, result screen). Previously all derived from `totalScore / baseScore` ratio which was wrong (totalScore includes originality, making ratio inflated). Legacy fallback to ratio derivation kept for entries without `speedMultiplier` field.

**Root cause of Sylvie's ×1.25 display:** she never presses Alto (timer runs out, `secsLeft=0`, `speedMultiplier=1.0` stored correctly in Firebase), but leaderboard was deriving 1.25 from `totalScore/baseScore`. Fixed in v256 — now shows no speed badge (correct).

**Last version deployed this session: v260916.256**

### Known issue — speed multiplier positive drift (unresolved)
Some players consistently get speed multipliers of 1.17, 1.20 etc without pressing Alto. Timer jitter on busy devices can cause `setInterval` to fire late or skip ticks. The negative case (sub-1.0) is now fixed. The positive drift case (multiplier slightly above 1.0 without pressing Alto) is not yet explained or fixed — likely same root cause (interval imprecision) but in the other direction. Flagged for future investigation.

### Session 18 — further deploys (Sep 17, continued again)

**v260916.257:** `computeAndUpdateOriginality` — read `aiResults` from Firebase (`myData.aiResults`) not in-memory (`G_daily.aiResults`), so manual Firebase corrections reflect in originality recompute.

**v260916.258:** `computeAndUpdateOriginality` — fix string key access for `answers` and `aiResults` (Firebase returns string keys after JSON round-trip, e.g. `"0"`, `"1"`); use `myData.totalScore` from Firebase instead of `G_daily.totalScore` from memory for `totalWithOrig` calculation; fix other players' answers key access too.

**v260916.259:** `computeAndUpdateOriginality` — now fires for any lang the player has played in (`_playedLangs[lbLang]`), not just when `lbLang === LANG` (UI language). `lbLang` passed through function signature and used in Firebase write path. Fixed in both `loadDailyLeaderboard` and `actualizar` call sites.

**Last version deployed this session: v260916.259**

---

### KNOWN ISSUE — Originality recompute is per-viewer, not global ✅ FIXED SESSION 19

---

### Session 19 (Sep 17)

**Global originality recompute — full rewrite of `computeAndUpdateOriginality`:**

- `computeUniqueness()` extracted from inside `renderLeaderboard` to module scope — pure function, shared by both `renderLeaderboard` and `computeAndUpdateOriginality`
- `computeAndUpdateOriginality` now loops ALL players in `allScores`, not just the viewer's own entry
- Firebase updates batched in parallel (`Promise.all`) — only writes `{ originality }`, never `totalScore`
- After writes: mutates `allScores` in memory with corrected values, calls `renderLeaderboard(allScores, false)` — no re-fetch (avoids partial snapshot risk)
- Viewer's own result screen (`G_daily.*`, `reapplyOriginality()`) still updated as before
- Guard: if no entries need updating, returns early — no writes, no re-render

**`totalScore` contamination fix:**
- Old code wrote `totalScore: baseScore*speed + originality` to Firebase in `computeAndUpdateOriginality` — this skewed the leaderboard sort and inflated the speed multiplier fallback ratio
- New code never writes `totalScore` from this function — `totalScore` in Firebase is always pure `baseScore × speedMultiplier` from now on
- Today's entries (Sep 17) may still show contaminated scores in the leaderboard — pre-existing data, self-heals from tomorrow

**Score display fix — `showDailyResult` recompute:**
- `baseScore` and `totalScore` are now recomputed from `aiResults` at the start of `showDailyResult` — never trusted from localStorage/Firebase which can be stale after a recontest
- `G_daily.speedMultiplier` is the only value trusted from storage (cannot be derived)
- Fixes the `300 × 1.08 → 370` mismatch caused by localStorage `totalScore` getting out of sync with `aiResults` after a recontest

**Originality badge guard:**
- `+50✨` badge on result screen only shown if `G_daily.aiResults[i] === 'valid'` (what the screen actually shows)
- Same guard added to `reapplyOriginality()` for lang-switch path
- Prevents badge appearing on an answer shown as ❌ when Firebase aiResults diverges from localStorage (e.g. after a recontest that changed Firebase but the screen reloaded from localStorage)

**Speed line format cleanup:**
- Removed intermediate total from speed breakdown line in all 3 places
- Before: `base 300 pts ×1.08⚡→ 370 pts +100 originalidad✨`
- After: `base 300 pts ×1.08⚡ +100 originalidad✨`
- Fits better on iPhone, less redundant with the header total

**`myBaseTotal` fix in `computeAndUpdateOriginality`:**
- Now prefers `G_daily.totalScore` (in-memory, from recomputed `showDailyResult`) over `myData.totalScore` (Firebase, potentially contaminated by old code)
- Prevents double-counting originality in the header total for players whose Firebase `totalScore` was already inflated

**Last version deployed: v260917.265**

### Known issue — same-device reload reads from localStorage not Firebase (backlog)
`startDailyChallenge` hits localStorage first if `stored` exists on this device — Firebase is only read on the cross-device path. So a recontest overturn on Device 2 won't be reflected when the player reloads on Device 1 (they see the old localStorage result). The `showDailyResult` recompute fix (Session 19) makes the display consistent with whatever `aiResults` is in localStorage — but if localStorage `aiResults` itself is stale, the score will still be wrong. Full fix: same-device reload should read from Firebase. Flagged in Session 15 backlog, still not built.

---

### Session 20 (Sep 18)

**Leaderboard grand total display:**
- Leaderboard rows now show `totalScore + originality` as the headline number, not just `totalScore`
- Before: `480 pts ⚡×1.20 +200✨` — score and originality badge were separate, score was base×speed only
- After: `680 pts (⚡×1.20 +200✨)` — grand total is the headline, speed and originality in parentheses
- `speedTag()` replaced with `scoreTag()` which builds the parenthetical from speed and/or originality
- Format rules: `⚡×N.NN` only shown when speed multiplier > 1.00 (Alto was pressed); `+N✨` only when originality > 0; parenthetical omitted entirely when neither applies
- Sort order updated to rank by `totalScore + originality` — previously only sorted by `totalScore`
- Both top-10 rows and the "me outside top 10" row updated

**`shareDailyResult` score summary fix:**
- Was computing originality as `(totalScore - baseScore) * 10` — relied on old contamination assumption where totalScore included originality. Now uses `G_daily.originality * 10` directly
- Removed `→ totalScore pts` intermediate total from share text, consistent with speed line cleanup from Session 19
- Fixed wrong emoji in per-answer originality badge: was `+50⚡` should be `+50✨`

**`totalScore` contamination note (Sep 18 data):**
- Marina's entry was manually corrected in Firebase console — her `totalScore` was `81` (contaminated by old code) instead of `56` (correct `Math.round(500 × 1.11)`)
- From v260917.265 onwards no new contamination occurs — `totalScore` in Firebase is always pure `base × speed`
- Any entries written by players on cached old code before they picked up v265 may still be contaminated

**Auto-reload on stale version:**
- On every page load, fetches `index.html` with `cache: 'no-cache'` (GitHub Pages responds with `304 Not Modified` if unchanged — negligible cost)
- Extracts version string from fetched HTML, compares with `RUNNING` constant
- If newer version detected: removes sessionStorage guard, calls `location.reload(true)`
- `sessionStorage` key `alto_ver_checked` prevents infinite reload loop — set to `RUNNING` before fetch, cleared before reload
- Regex `/v(\d{6}\.\d+)(?!\d)(?!-tmp)/` — `(?!\d)` prevents partial match on longer numbers, `(?!-tmp)` excludes staging versions
- `RUNNING` constant and display div are the only two occurrences of the version string — standard version bump updates both automatically, no extra step needed
- Fully silent — no UI, no toast; `.catch(() => {})` means offline/blocked fetch is a no-op

**Last version deployed: v260918.269**

---

### Session 21 (Sep 19)

**Spoiler guard fix — flag tap after page refresh (v270):**
- `switchDailyLbLang` was computing `notPlayed = !_playedLangs[lang]` — which is always `true` on page refresh because `fetchPlayedState()` is async and hasn't resolved yet
- Fixed: `notPlayed = !isHistory && !_playedLangs[lang] && !localStorage.getItem('alto_daily_' + getDailyKey() + '_' + lang)`
- `isHistory = _lbDate !== null` — past dates are always open regardless
- Mirrors the same pattern already used in `updateDailyButtonDate()`

**Spoiler guard fix — refresh button bypass (v272):**
- `refreshDailyOriginality` was calling `renderLeaderboard(snap.val())` with no `hideAnswers` argument — always showed answers regardless of active lang
- Fixed: same `!isHistory && !_playedLangs && !localStorage` logic applied before calling `renderLeaderboard`
- Also added `&& !hideAnswers` guard to `computeAndUpdateOriginality` call inside it — no point computing originality for an unplayed lang
- Three functions that trigger leaderboard render: `switchDailyLbLang` (fixed v270), `lbHistNav` (always open for past dates — correct), `refreshDailyOriginality` (fixed v272). All now consistent.

**Step 2 — same-device reload reads from Firebase (v271):**
- `startDailyChallenge` localStorage path now tries Firebase first before showing result
- Fetches by safeName scan (`daily/{date}/{lang}/scores`) — same as cross-device path
- Falls back to localStorage silently if Firebase fails or returns no entry
- Also sets `G_daily.key/letter/categories/theme` on this path (previously skipped — needed by recontest and originality recompute)
- playerKey not in localStorage — uses safeName scan fallback (same as cross-device)
- Outer try/catch wraps everything — any failure still falls through gracefully

**Firebase schema addition (Session 21):**
- `daily/{date}/{lang}/scores/{playerKey}/contestLog/{idx}: { original, overturned, timestamp }` — written on successful recontest overturn. Captures original AI verdict and overturned verdict per answer index. For AI effectiveness analysis. Written alongside existing `aiResults/${idx}` update. Original verdict captured from `G_daily.aiResults` before overwrite.
- Note: `aiResultsOriginal` was considered and rejected — `contestLog` alone is sufficient to identify AI mistakes without cluttering Firebase.

**iOS home screen cache — no fix needed:**
- Safari has two cache layers: stored (SSD) and memory (RAM). `location.reload(true)` updates stored cache but serves from memory this session. Next launch gets fresh version.
- Our version check is working correctly — one-session lag is unavoidable iOS limitation
- Fix for stuck users: delete shortcut from home screen and re-add it

**Multiplayer cumulative scoring fix (v274):**
- Bug: `preRoundScores` in Firebase was never cleared between rounds. At round 3, `finishValidation` used round 1's `preRoundScores` as base (still truthy), losing round 2 entirely.
- Fix: `preRoundScores: null` added to `nextRound()` update call. Firebase `update()` with `null` deletes the field. `finishValidation`'s `||` fallback then correctly snapshots current cumulative `room.scores` as new base.
- Critical rule: `preRoundScores` must always be `null` between rounds and only set by `finishValidation` at scoring time.

**AI verdict consistency observations:**
- Same answer (e.g. "Electricidad", "Catherine Deneuve") can get different verdicts for different players — each player's answers validated independently at submit time with separate model calls
- All-unsure results (all 6 answers 🤔) = likely connection failure, not genuine uncertainty
- Electricity ruled invalid — it is a natural phenomenon, not an invention
- Misspellings ruled invalid (e.g. "Eifel" → should be "Eiffel", "Edimbugo" → "Edimburgo")
- Ciclisme (Catalan/Spanish) invalid in FR game — wrong language

**Last version deployed: v260918.274**

---

## 🚩 Flagged for Next Session

**A — Validation error detection:**
If all answered slots come back `unsure` after AI validation, flag it as a likely connection failure. Either show the player a warning ("hubo un problema validando tus respuestas") or log it to Firebase for monitoring. All-unsure on 4+ answers is essentially impossible legitimately.

**B — Full revalidation button:**
If player has 2+ unsure verdicts on today's result screen, show a "Revalidar todo" button. Reruns AI validation on all unsure answers at once (same 3-model majority system as recontest). Shares existing 1-per-day recontest limit per lang. Result is final. Today only (same guard as recontest).

**C — Multiplayer tied scores — medal fairness:**
When two or more players finish with the same score, they get different medals based on arbitrary sort order. Should show the same medal to all tied players (two golds if tied for 1st, etc). Affects both `showScores` (between rounds) and `showFinal` (end screen).

**D — WhatsApp share + Jugar de nuevo + ghost emojis + free mode examples (DEPLOYED v301):**
All built and deployed to production in v301.

**E — Free mode multiple example sets (BACKLOG):**
Idea: multiple themed sets of 20 example categories selectable via button row above textarea. Sets planned: "Lo que te rodea" (current), "Cultura y entretenimiento", "Mundo animal", "Gente y sociedad", "Comida y fiestas". Draft categories discussed in Session 22. Save for dedicated category session.

**F — Category pool expansion (BACKLOG):**
Discussed adding new categories: Personaje de ficcion, Genero musical, Divinidad mitologica, Novela famosa, Festividad. Also discussed group size limits and daily picker logic. Save for dedicated category session.

---

### Session 21 continued (Sep 20)

**v275 — Remove angry emoji, contest unsure answers:**
- `ENTRY_EMOJIS` reduced from 5 to 4: `['🔥','👏','😂','😬']` — 🤬 removed
- `canContest` now triggers on `invalid || unsure` — players can contest 🤔 unsure verdicts too
- Three issues found and fixed before deploying v275 (became v276):
  1. `originalVerdict` in contestLog was hardcoded `'invalid'` — fixed to read from `G_daily.aiResults`
  2. `originalVerdict` declared twice (inner and outer scope) — removed inner duplicate
  3. Firebase failure revert was hardcoding `'invalid'` — fixed to restore `originalVerdict`
- Failed recontest on unsure stays unsure (no penalty) — toast differs: "mantenemos el resultado" vs "respuesta inválida"

**v277 — Per-entry contest limit:**
- Old system: 1 contest per lang per day (global counter) — contesting one entry removed all other ¿Error? buttons
- New system: 1 contest per entry per day — each invalid/unsure answer can be contested independently
- Firebase schema change: `names/{safeName}/contests/{date}/{lang}` was a number, now an object `{"0":true,"2":true}` — one boolean per slot index
- Already-contested entries show "Ya revisado / Already reviewed / Déjà révisé" label with disabled button when tapped — no AI call made
- Global button sweep removed — after a contest, all other buttons re-enable
- Log path changed to `contests/{date}/{lang}_log/{idx}` — per-entry log objects
- Old number-type Firebase entries are harmless — child reads on a number node return null, no migration needed
- To reset for testing: delete specific idx nodes under `names/{safeName}/contests/{date}/{lang}` in Firebase console

**v278 — Contest Method A/B:**
- `const CONTEST_METHOD = 'B'` added near `DAILY_CONSENSUS` — flip to `'A'` to revert, one character change
- Method A: strict scope, 2-of-3 threshold, DUDOSO upholds
- Method B: generous scope only, strict spelling, 1-of-3 threshold, DUDOSO upholds
- Both methods use same 3 models, same `parseVerdict` structure
- `method: CONTEST_METHOD` recorded in every contestLog entry for comparison
- Currently live: Method B

**v279 — Fix per-answer pts badge after overturn:**
- `ptsEl` was showing `toastPts` (net gain delta) instead of absolute value
- For invalid→valid: was accidentally correct (+100 either way)
- For unsure→valid: showed +50 (delta) instead of +100 (absolute) — the bug
- Fixed to always show `+100` since overturned answer is always worth 100pts
- Toast still correctly shows net gain (+50 from unsure, +100 from invalid)

**v280/v281 — Tighten Method B prompt:**
- Removed existence check ("must be a real word in the language") — incorrect for proper nouns, names, places
- Method B gate is now spelling only: any misspelling beyond accents/tildes = INVALIDO
- If spelling passes, generous on category scope
- DUDOSO upholds in both methods (removed DUDOSO→valid from Method B)
- Prompt structure: check spelling first → if passes, be generous on scope

**Firebase schema additions (Session 21 continued):**
- `names/{safeName}/contests/{date}/{lang}/{idx}: true` — per-entry contest flag (replaces counter)
- `names/{safeName}/contests/{date}/{lang}_log/{idx}: { idx, category, letter, word, originalVerdict, newVerdict, votes, method, timestamp }` — per-entry contest log with method field

**Last version deployed: v260918.281**

**v282 — Already reviewed UX:**
- Tapping ¿Error? on an already-contested entry now shows a toast "Ya revisado hoy / Already reviewed today / Déjà révisé aujourd'hui" and removes the button — instead of showing a disabled "Ya revisado" label which was too long for iPhone

**Last version deployed: v260918.282**

---

### Session 22 (Sep 22–25)

**Recontest overhaul — was always returning invalid (0/3 votes):**
- Root cause 1: Three recontest models (`nvidia/nemotron`, `ling`, `z-ai/glm`) were silently failing or timing out — every failure counted as an invalid vote. Fixed: recontest now uses single model `inclusionai/ling-3.0-flash-fin:free`, same as daily validation.
- Root cause 2: `max_tokens: 10` was cutting off the model before it could output a verdict. Fixed: bumped to 800 to match normal validation.
- Root cause 3: Model puts verdict at end of reasoning chain — `parseVerdict` was scanning full text and hitting "invalid" mentioned mid-thought. Fixed: reads last word of response first, then falls back to full-text scan on `content` only (not reasoning).
- Root cause 4: "Previous judge said INVALIDO" anchoring in Method B user prompt primed model to uphold. Fixed: Method B now asks fresh neutral question.
- Root cause 5: Leading space in `cat` argument passed to `recontest()` — `' ${cat}'` → `'${cat.trim()}'`.

**Proper noun rule added to both prompts:**
- Daily validation system prompt: rule 3 now explicitly states "proper nouns are international — person names, surnames, singers, actors, writers, fictional characters, song/film/TV titles, brands, cities, and countries are accepted in any language; only common nouns must be in [lang]"
- Recontest Method B user prompt: same note added inline
- Fixes: Lou Reed rejected as non-French, Lee Hazlewood, city/country names in wrong language

**`dailyAICallModel` return shape changed:**
- Now returns `{ verdict, rawText }` instead of plain string
- `dailyAIValidate` (consensus path) updated to destructure `.verdict` — would have silently broken if `DAILY_CONSENSUS` ever enabled
- Main validation loop destructures both, stores `rawText` in `aiResponses` for invalid/unsure entries

**AI reasoning stored in Firebase:**
- `scores/{playerKey}/aiResponses` — raw model reasoning for invalid/unsure answers, written at submit time. From Session 22 submissions onwards.
- `contests/{date}/{lang}_log/{idx}/aiResponse` — raw model reasoning for every recontest. From Session 22 onwards.
- Both capped at 1000 chars, `<think>` blocks stripped.
- Firebase size impact: negligible (~3KB/player worst case, skipped for all-valid players)

**WhatsApp share — host kicked out of room (v301):**
- Root cause: `shareWhatsApp()` called `window.open('https://wa.me/...', '_blank')`. On mobile this navigates away, triggering reload and session restore that felt like being kicked out.
- Fix: `navigator.share()` first (native OS share sheet, no navigation). Falls back to `window.open` on desktop. AbortError (user cancelled) handled silently.

**"Jugar de nuevo" fixed (v301):**
- Root cause 1: Room deleted 60s after game end. Host tapping "Jugar de nuevo" reset Firebase to `phase:'lobby'` — but 60s later delete fired anyway, destroying the new game.
- Root cause 2: Both host and guest saw same button. Guest tap showed `toastHostOnly`.
- Fix: delete timeout stored in `G._deleteRoomTimeout`, cancelled in `playAgain()`, extended to 45 minutes. Button split: host gets active "Jugar de nuevo", guest gets disabled "Esperando al anfitrion...". New i18n key `waitHost` in ES/EN/FR.

**Ghost thumbs/emojis between rounds fixed (v301):**
- `enterPlaying` now clears `#val-content`, `#guest-val-content`, `#sc-list` at the start of each round. All three are fully rebuilt when their screens are shown — safe to clear early.

**Free mode "Usar ejemplos" button (v301):**
- Button above textarea fills it with 20 fun example categories in current language (ES/EN/FR).
- Switching language clears the textarea (Option A — fresh start, no stale categories).
- `FREE_CAT_EXAMPLES` object holds 20 categories × 3 languages.

**Free mode "Elegir 8 al azar cada ronda" toggle (v301):**
- Appears below textarea, enabled when 9+ categories written.
- When active: picks fresh 8 random categories each round from full list stored in Firebase as `room.allFreeCats`.
- Toggle state persisted in `alto_lobby_config`.
- `createRoom` stores `catMode: 'free-random'` and `allFreeCats` in Firebase. `nextRound` handles this catMode.

**Firebase schema additions (Session 22):**
- `daily/{date}/{lang}/scores/{playerKey}/aiResponses: { idx: string }` — AI reasoning for invalid/unsure answers
- `names/{safeName}/contests/{date}/{lang}_log/{idx}/aiResponse: string` — AI reasoning for every recontest
- `room.allFreeCats: string[]` — full free-mode category list when random mode active
- `room.catMode: 'free-random'` — new catMode value for free mode random picks

**Last version deployed: v260922.301**

---

### Session 23 (Sep 26) - design only, no code changes

**No builds, no deploys. Production unchanged at v260922.301.**

Full design session for the multiplayer scoring overhaul. The complete, agreed design is in **`MULTIPLAYER_SCORING_DESIGN.md`** (in the project folder, alongside this handoff). **Read it before starting any multiplayer work.** It SUPERSEDES the "Multiplayer Scoring & History Overhaul (BIG FEATURE)" section earlier in this handoff; where they differ, the design file wins.

Living version (Claude Doc): https://claude.ai/artifact/9bCUk2yRwx1wHQG3hmjhoV

**Key decisions (summary, details in the design file):**
- Answer points unchanged (100 unique / 50 duplicate / 0 invalid). The old "+50 originality bonus" is dropped.
- Alto caller: +250 scaled by categories (round10(250 x n/8)) minus 100 per invalid. Lobby penalty slider removed.
- Reactions: one reaction per answer, none on your own. Fire +30, clap +20, laugh +10, grimace 0. Lobby toggle "Reacciones con puntos", default on. goBackToValidation must stop wiping reactions.
- Ties: dense medals (gold, gold, silver, bronze); all tied winners get the win. Covers flagged item C.
- roundLog per round in the room, keyed by category index; feeds cards, final screen and history.
- Scores cards: all collapsed, this round only, red A badge for the caller. Final screen shows every round plus three awards: Popularidad, Risas, Originalidad.
- Share text: scores only, plus awards line and group link.
- Solo games AND incomplete games are practice (no leaderboard, no history). Solo start button "Practicar solo"; early Terminar shows a confirm.
- Groups get hidden random IDs; "Mis grupos" list per device (max 10), lives in the lobby picker; home screen unchanged. Default group "Grupo de [host]". Players keyed by name without emoji.
- Global leaderboard removed. Group leaderboard keeps wins/games/win %.
- History per group, last 20 games, navigate by game with language flag + date. Groups idle 1 year deleted.
- Clean start: old groups/, groupNames/, global/ to be deleted in Firebase console when Build 5 ships.
- Per-player stats dropped.

**Build plan (5 builds, all via staging):** 1 Scoring engine, 2 Reaction bonuses, 3 Scores cards, 4 Final and share, 5 Groups and history.

**Oscar's manual steps for Build 5:** add `".indexOn": "lastPlayed"` on `groupsMeta` in Firebase rules; delete `groups/`, `groupNames/`, `global/`.

**Flagged items update:** A and B (validation error detection, full revalidation button) not needed for now. C is folded into Build 1. File size over 300KB is not a concern per Oscar.

**deploy.bat updated:** now also copies, commits and cleans up `MULTIPLAYER_SCORING_DESIGN.md` from Downloads, same as the handoff. deploy.bat itself is not copied by any script: replace it in D:\09_ALTO by hand.

**Next session:** start Build 1 on staging, working from `MULTIPLAYER_SCORING_DESIGN.md`.

**Last version deployed: v260922.301**

---

### Session 24 (Sep 26) - Multiplayer scoring Builds 1-4 + fixes → PRODUCTION v260926.311

**Summary:** Builds 1-4 of `MULTIPLAYER_SCORING_DESIGN.md` built on staging (v260926.302-tmp → v260926.310-tmp), tested by Oscar in a real multiplayer game, then promoted unchanged (only the version string) to **production v260926.311**. Build 5 (groups and history) is next — see the end of this entry. Entries below are in build order.

**Build 1 started from:** production v301 (the old index_tmp.html was stale v300-tmp and was not used). Follows `MULTIPLAYER_SCORING_DESIGN.md`.

**Scaled ¡Alto! adjustment (replaces flat penalty):**
- `finishValidation`: caller gets `round10(250 x n/8) - 100 x invalid`, n = categories this round. Empty answers counted as invalid defensively (the 2-char guard should prevent them).
- Caller only counts if `room.calledStop` and `stopCaller` is a player in the room. Timeout writes `stopCaller:'Tiempo'`, which is never treated as a caller.
- Host toast: `🛑 Oscar ¡Alto! +250 pts` / `−150 pts`. Old `toastPenalty`/`toastPenaltySuffix` keys removed.
- Democratic mode: vote results are applied to `G.invalidAnswers` before scoring, so "invalid after the vote" works with no extra code.

**Penalty slider removed:** lobby HTML, `saveLobbyConfig`, `restoreLobbyConfig`, `startGame`, `createRoom`, `applyLang`, `penalty` T keys (ES/EN/FR), debug room. `stopPenalty` is no longer written to rooms. Old `penalty` values in `alto_lobby_config` are simply ignored.

**roundLog written:** `finishValidation` writes `roundLog/{currentRound}` in the same atomic update as scores. Shape per design doc: `letter`, `categories`, `caller` ("" on timeout), `players/{name}: { answers[], status[] (u/d/x/e), got[] ({f:0,c:0,l:0} placeholders until Build 2), pts:{answers, alto, reactions:0, round, total} }`. Keyed by category index. Rewritten in full on every recalculation. A player in the room with no answers gets all `e`.

**Tied medals:** new `denseMedals(sorted)` helper next to `MEDALS`. Used in `showScores` and `showFinal`. Group leaderboard (`renderLb`) still uses `MEDALS[i]`; it is rewritten in Build 5.

**Tie winner line:** `showFinal` names all tied winners: "¡Empate! 🦊 Oscar y 🌸 Marta con 900 puntos" (EN "It's a tie! ... with", FR "Égalité ! ... avec"). New T keys `tieLine`, `tieAnd`. Confetti uses first winner's emoji.

**Practice games (no leaderboard):**
- `startGame` writes `practice: true` if only 1 player at start.
- `endGame` treats the game as practice if `room.practice` OR `currentRound < rounds`: skips all `global/` and `groups/` writes, still goes to `phase:'final'`.
- Complete games: every player tied for top score gets a win.

**Terminar confirm:** scores-screen Terminar now calls `confirmEndGame()`. Before the last round the host gets a native `confirm()` with T key `endEarlyConfirm` (ES/EN/FR per design doc). On the last round it ends directly. Final screen unchanged.

**Practicar solo button:** new `updateStartBtn()` + `G_lobbyPlayerCount`. Called from `renderPlayers` and `applyLang` (replaces `s('btn-start', t('start'))`, which would otherwise reset the label on language switch). T key `soloStart`: "✏️ Practicar solo →" / "✏️ Practise solo →" / "✏️ S'entraîner en solo →". Existing "(¡mejor con más!)" hint kept.

**Rules and hints:**
- `rulesHTML` box rewritten: "🛑 Quien pulsa ¡Alto! gana +250 pts si todo es válido, y pierde 100 por cada respuesta anulada (con 8 categorías)." + EN/FR.
- `homeRulesHTML` multiplayer scoring: two new rows, `altoOk` (+250 pts) and `altoBad` (−100 pts), ES/EN/FR.
- `validateSub2`: "única = 100 · repetida = 50 · anulada = 0 · ¡Alto! −100 por anulada" + EN/FR. `setValidationMode` had a duplicated inline copy of this string for democratic mode; now uses `t('validateSub2')`.

**Bug fixed on the way (pre-existing since v301):** after "Jugar de nuevo", `preRoundScores` from the previous game's last round was never cleared, so round 1 of the new game would be scored on top of old totals. `startGame` and `playAgain` now clear `preRoundScores`, `roundLog` (and `playAgain` clears `practice`).

**Debug:** `DBG.scores` now has María and `__debug__` tied at 400, to check dense medals and the tie line.

**Tests run:** extracted the real `finishValidation`, `denseMedals`, `endGame`, `confirmEndGame`, `showFinal`, `updateStartBtn` into Node with a mocked Firebase. 35/35 logic checks pass (all design-doc tables for caller adjustment and category scaling, timeout, recalculation idempotence, absent player, democratic invalidation, dense medals, shared wins, solo/incomplete practice, confirm cancel/OK/last round), plus string rendering in all 3 languages.

**Firebase schema additions (Session 24):**
- `room.roundLog/{round}` — see above
- `room.practice: true/null` — solo-start flag
- `room.stopPenalty` — no longer written

**Noticed, not fixed (out of scope):** `doSubmit` hard-codes Spanish `stopCaller:'Tiempo'` and "✏️ Respuestas enviadas. Esperando a los demás…". `reviewSub` T key is dead (no callers).

**Staging test checklist for Oscar:**
- All testers on `index_tmp.html` (scoring runs on the host, screens render on every client).
- Lobby: no penalty slider; 1 player shows "Practicar solo", 2+ shows "¡Comenzar!"; switch language, label stays correct.
- ¡Alto! with all valid: host toast +250. With invalids: +150 / +50 / −50 ... Timeout: no toast.
- Revisar → recalc: totals don't double.
- Ties: two players same score show 🥇🥇; final screen shows "¡Empate!".
- Terminar before last round: confirm appears; Cancel does nothing; OK ends. Last round: no confirm.
- Jugar de nuevo then a new game: round 1 totals start from 0.
- Firebase console: `rooms/{code}/roundLog/1` exists with the expected shape.

**v260926.303-tmp — Revisar keeps decisions (bug found in testing, pre-existing):**
- `goBackToValidation` no longer wipes `invalidAnswers`, `votes` or `entryReactions` — it only sets `phase:'validate'`, then calls `showValidation(room)` + `applyVotesAndReactions(room, true)` so everything is drawn straight away. (Keeping reactions was planned for Build 2; done here.)
- `showValidation` seeds `G.invalidAnswers` from `room.invalidAnswers` instead of wiping it, and draws marked answers as `val-entry invalid` with the ↩ button. Also fixes a hidden bug: a host reload mid-review lost the marks on screen, and the next ✕ overwrote all saved marks in Firebase.
- Host restore paths (`continueSession`, `restoreToScreen`) now also call `applyVotesAndReactions(room, true)`.
- Stale-mark guards: `nextRound` now clears `invalidAnswers` (it never did); first entry into validate passes `{...room, invalidAnswers:{}, votes:{}}` to `showValidation` because the snapshot can hold last round's marks (same keys in Clásico).
- `G_myVotes` / `G_myEntryEmojis` are reset in `enterPlaying` each round (previously only Revisar reset them). `applyVotesAndReactions` rebuilds the player's own reaction highlights from Firebase, so they survive reload and Revisar.
- Tests: 18/18 on the real functions (mark 2 invalid → calculate → Revisar → marks, reactions and own highlights still there → undo one → recalc correct; host reload keeps marks and next toggle doesn't wipe; new round shows no stale marks; democratic votes kept). Build 1 suite still 35/35.

**Plan agreed:** keep building all 5 builds on staging, test the full feature in a real multiplayer game, fix bugs, then promote once. Handoff and design doc go up with the production promotion only (deploy_tmp.bat doesn't copy them).

**v260926.304-tmp — Build 2: reaction bonuses:**
- Lobby checkbox `#reaction-points` "🌶️ Reacciones con puntos" (EN "Reactions score points", FR "Réactions avec points"), default on, under the seconds slider, with a language-neutral hint `🔥 +30 · 👏 +20 · 😂 +10 · 😬 0` (11px). T key `reactionPoints`; `applyLang` sets `lbl-reaction-points`. Saved as `reactionPoints` in `alto_lobby_config`; written to the room as `reactionPoints: true/false` in `startGame`. Rooms without the field count as on.
- `reactToEntry`: one reaction per answer (each tap nulls all four of the player's emojis on that answer in the same write, then sets the chosen one; same emoji again removes). No reactions on own answers (`isOwnEntry(key)` = key starts with `playerName + '__'`). Strip redrawn instantly from `G_cachedReactions` (fixes pre-existing stale count when the last reaction on an answer was removed).
- `renderEntryEmojis`: own answers get 4 `disabled` buttons; emojis you received get class `got` (full opacity). New CSS: `.entry-emoji-btn:disabled` (0.25 opacity, no hover), `.entry-emoji-btn:disabled.got` (opacity 1).
- `G_cachedReactions` set in `applyVotesAndReactions`, reset in `enterPlaying`.
- `finishValidation`: reactions present at Calculate only. Self-reactions ignored; one per reactor per answer (if an old client left several, the lowest value is kept). 🔥 +30 / 👏 +20 on valid (u/d) only; 😂 +10 on valid or invalid; 😬 0; nothing on empty. `got[ci] = {f,c,l}` always recorded (even with points off, and fire/clap on invalid answers too) for Build 4 awards. Points go to `pts.reactions` only when `room.reactionPoints !== false`.
- Debug: `DBG.entryReactions` sample (incl. reactions on `__debug__`'s own answer to show the greyed `got` style); `dbgFakeRoom` uses it with `reactionPoints: true`.
- Tests: 28/28 Build 2 (scoring, points on/off, self, one-per-reactor, empty, Revisar undo, tap/swap/remove, others' reactions kept, own-answer strip, name-prefix edge). Build 1 35/35, Revisar 18/18 (two expectations updated: that test room has a 🔥 that now correctly scores +30).
- Not in Build 2: help/rules text (help pass after Build 5), reaction points shown per player (Build 3 cards).

**v260926.305-tmp — Build 3: Scores cards:**
- `showScores` renders one card per player from `room.roundLog[currentRound]` via new `scoreCardHTML()`; falls back to the old simple `.score-row` list if there is no roundLog for the round. Players missing from the roundLog get a row without a panel.
- Collapsed row: medal + emoji (`.sc-who`), name + red A badge (`.sc-namewrap` > `.sc-name` + `.sc-alto`), round points `.sc-round` (signed), total `.sc-total`, chevron. Tap toggles (`window.toggleScoreCard(i)`, names by index in `G_scNames`).
- Panel: per category `.sc-ans-cat` / `.sc-ans-mid` (answer `.sc-ans-val` + reactions received `.sc-ans-got`, which wraps below a long answer) / mark `.sc-ans-mark` (`+100`, `+50`, red `✕`, blank for empty with `—` answer). Breakdown `.sc-breakdown`: `respuestas N · A ±N (caller only, red) · reacciones +N (only when reactionPoints on)`. T keys `bdAnswers`, `bdReactions` (ES/EN/FR).
- Open cards persist across re-renders (`G_scOpen`), reset when the round changes (`G_scRound`) — needed because `handleRoom` redraws the Scores screen on every room update.
- New helpers: `escHtml()` (player text in the cards is escaped), `signed()` (uses a real minus sign).
- iPhone layout: rendered with the real CSS + real Caveat/Special Elite fonts in headless Chromium at 414/390/375/360/320px, incl. a stress case (8 FR categories, long names/answers, 5-digit negative total, A badge). No overflow at any width. Under 360px: ` pts` unit hidden (`.sc-unit`), tighter gaps, name group min 3.5em.
- Debug: `DBG.roundLog['2']` (Carlos caller +60 with an invalid, reactions, dups, empty); debug totals now María 690 / Carlos 560 / __debug__ 690 (tie kept); `preRoundScores` María 300; `DBG.invalidAnswers` adds `Carlos__Fruta`.
- Tests: 18/18 Build 3 (collapsed by default, A badge only on caller, escaping, marks, breakdown parts, open persists / resets per round, points off, timeout, fallback, missing player, minus sign). Earlier suites still 35 + 18 + 28.
- Only 11/13/20px used. Final screen unchanged until Build 4.

**v260926.306-tmp — Build 4: Final screen and share:**
- Refactor: answer rows and breakdown line pulled out of `scoreCardHTML` into shared `roundRowsHTML(d, cats)` and `breakdownHTML(pts, isCaller, room)` (Scores and Final cards use the same code).
- `roundLogEntries(room)`: normalises `roundLog` to `[[round, entry], ...]` — Firebase returns numeric-keyed objects as arrays (`[null, r1, r2]`); only rounds 1..currentRound.
- `computeAwards(room)`: 🔥 Popularidad = 🔥x30 + 👏x20 received on ANY answer (Oscar agreed: invalid answers count too); 😂 Risas = 😂 count; ✨ Originalidad = unique valid answers (`u`). Only players in `room.scores`; only awards > 0; ties list all names. Works with reaction points off (reads `got`).
- Final screen: new `#final-awards` row of `.aw-chip`s under the winner line; `#final-list` now `finalCardHTML()` cards (collapsed: medal, emoji, name, final total; opened: per round `.fn-round-h` heading "Ronda N · letter [A badge] ... ±round", then answer rows + breakdown). `window.toggleFinalCard(i)`. Falls back to the old list if no roundLog. Winner/tie line and confetti unchanged.
- Share: new `#btn-share-final` "Compartir"/"Share"/"Partager" for everyone. `finalShareText(room)`: `¡ALTO! · group · N rondas` (group omitted if empty; "1 ronda" singular via `rndLabel`), one line per player with dense medals then `4.` rank numbers (ties share), awards line `🔥 names · 😂 names · ✨ names`, `jugarenfamilia.es` (Build 5 swaps in `?g=ID`). `window.shareFinal`: `navigator.share` first (AbortError silent), else clipboard + button shows "¡Copiado!" 2s, else textarea/execCommand fallback. Stored room in `G_finalRoom`.
- New T keys (ES/EN/FR): `awPopularity`, `awLaughter`, `awOriginality`, `shareBtn`, `copied`, `roundsWord`. Note: `DAILY_LABELS` has its own separate `shareBtn`/`shareCopied` — not a clash.
- CSS: `.aw-row` (hidden when empty), `.aw-chip` (13px SE, names 20px Caveat), `.fn-round-h`, `.fn-round-pts`.
- Debug: `DBG.roundLog['1']` added (timeout round, different categories P: Color/País/Objeto) so the debug Final screen shows a 2-round game; awards: Popularidad María, Risas tie María/__debug__, Originalidad __debug__.
- iPhone: real CSS + fonts rendered at 414/390/375/360/320px, normal ES game and FR stress case (long tied names in award chips, 8 categories, all-invalid ¡Alto! −550). No overflow; at 320px the 3-button row wraps the third button to its own line (existing `.btn-row` behaviour).
- Tests: 20/20 Build 4 (awards incl. invalid-answer fire, ties, zero, departed player; Firebase array shape; share text ES/EN/FR, singular, no group, rank numbers incl. tied 4th; final render, round headings, A badge, toggle, fallback). All suites 119/119.

**v260926.307-tmp — Cards made consistent with the Daily leaderboard (Oscar's review):**
- Scores and Final cards now REUSE the daily leaderboard classes instead of copies: `.daily-lb-row`, `.daily-lb-rank`, `.daily-lb-name`, `.daily-lb-score`, `.daily-lb-speed` (bracketed extra), `.daily-lb-chevron`, `.daily-lb-me` (own row highlight), `.daily-lb-answers` (lighter panel with border), `.daily-lb-answer-row/-cat/-val/-ai/-pts(.zero)`. Checked: no code queries these classes globally (only `.daily-lb-lang-btn`). Rows also carry `.mp-row` for multiplayer-only tweaks.
- Shared row builder `mpRowHTML({id, rank, name, emoji, total, extra, isCaller, onclick, open})`: rank column = medal or dense rank number (`denseRanks()` helper, also used by share text), "name emoji", A badge, `N pts`, `(+round)` on Scores only (like daily's `(⚡x1.10 +100✨)`), chevron.
- Answer marks now match daily: ✅ +100 / ✅ +50 / ❌ 0 (grey `.zero`); empty = small grey italic — with no icon/points. Reactions sit between answer and icon (`.sc-ans-mid` wrapper so they drop below a long answer).
- Removed old card CSS (`.sc-row`, `.sc-panel`, `.sc-who`, `.sc-name`, `.sc-namewrap`, `.sc-total`, `.sc-round`, `.sc-chevron`, `.sc-ans-row/-cat/-val/-mark`). Kept: `.mp-namewrap`, `.sc-alto`, `.sc-ans-mid`, `.sc-ans-got`, `.sc-breakdown`, `.sc-unit`, `.fn-round-h`, awards CSS.
- Name before emoji across multiplayer (matches daily): lobby chips, winner line T key `winnerLine` = '{name} {emoji} ...' (ES/EN/FR), tie line names, Scores/Final cards and fallback lists, group leaderboard rows.
- Checked: new cards rendered directly above a daily sample at 414/390/375/360/320 px — no overflow; tests 124/124 (new: name before emoji, own-row highlight, rank column 🥇🥈🥉4 5, daily marks, empty answer, bracketed extra; winner/tie lines verified in 3 languages).

**v260926.308-tmp — Reaction emoji strength (Oscar's idea, option "D2"):**
- All tappable reaction emojis now at full strength (`.entry-emoji-btn` opacity 1; was 0.4). Your selection is shown by the existing `.active` background box.
- Own answers: buttons get class `own` (opacity 0.4, no hover scale); reactions you received on your own answers get `own got` (full strength). Replaced the old `disabled` attribute + `:disabled` CSS so a tap still reaches the handler.
- `reactToEntry` on your own answer shows a toast, T key `toastOwnReaction`: "No puedes reaccionar a tus propias respuestas" / "You can't react to your own answers" / "Tu ne peux pas réagir à tes propres réponses". Nothing is written.
- Options A/B/C/D/D2 were mocked with the real validation CSS before choosing. Tests 125/125.

**v260926.309-tmp — Refresh / session restore fix (bug found in Oscar's live test):**
- Symptom: host refreshing on the Scores screen landed on home (later: on the GUEST's welcome screen). Cause: `alto_session` lived only in localStorage, one slot shared by every tab of the browser; whatever joined/restored last overwrote it. (Guest in incognito has separate storage, so something in Oscar's normal window had saved Oscarjr.) Also `tryRestore()` ran twice per load (duplicated init block — was in production too).
- `saveSession` now writes localStorage AND sessionStorage (per tab, survives refresh) and puts `?room=CODE` in the address (`setRoomInUrl`, keeps other params like `?v=29`). `clearSession` clears both and removes `?room`.
- `tryRestore`: with `?room=` → restores directly only with a session for that exact room, this tab's own (sessionStorage) first, shared (localStorage) as fallback; otherwise the join screen. Without `?room=` → tab session, else shared. After a successful restore the tab is pinned (sessionStorage + URL).
- New `showQuickJoin(code, prefillName)` helper (used by tryRestore and startFresh). `OPENED_ROOM` = `?room=` at page load. "No soy yo — cambiar" (`startFresh`): if the page was opened with a room code → that room's join screen with an empty name (was: home, and with the address change the second player would have had no way back in); plain address → home as before.
- Removed the duplicated init block (first copy, before the debug createRoom hook). `buildEmojiGrid/applyLang/tryRestore` now run once. Name prefill in init prefers the tab session.
- Known fallback: a brand-new tab on the plain address offers "welcome back" as the last player saved anywhere in this browser (e.g. after closing the browser) — same as before.
- Note: there is no join-by-code box on the home screen any more (`joinRoom` references `#inp-code`, which is not in the HTML) — joining is only via invite link. Left as is.
- Tests: real-browser reproduction harness (`/tmp/repro2.py` idea: real staging HTML, fake Firebase module served via Playwright routing, shared in-memory DB, separate browser contexts for normal/incognito, tabs in same context for same window). 19/19 scenarios: host normal + guest incognito refresh; two tabs same window (second player via invite link → No soy yo → join); repeated refreshes; third tab joining; reopened tab; leaving clears address; No soy yo on plain address → home. Old build fails the same-window scenario. Unit suites 125/125; real-click reaction test still passes.

**v260926.308-tmp follow-up (Oscar asked to triple-check the non-disabled own emojis):** audited all 4 render sites and the single `reactToEntry` handler; nothing depended on `disabled`; `.own` class unique; host scoring ignores self-reactions anyway. Real Chromium click test: own tap → pill in ES/EN/FR, nothing saved; other's tap saves; swap works; no JS errors.

**Process note:** v308 was built without explicit plan confirmation — rule is plan → Oscar's OK → build, even when he suggests the change.

**v260926.310-tmp — Breakdown line wording (Oscar's idea, refined):**
- `breakdownHTML` now reads `base 600 pts (A +50, reacciones +60)` — same "base" wording as the daily result line. A part only for the caller (red, sign after the label: `A −150`); reactions part only when reaction points are on AND non-zero; no brackets when there are no extras (`base 700 pts`). Used by Scores cards and every round of the Final cards.
- T key `bdAnswers` replaced by `bdBase: 'base'` (ES/EN/FR); `bdReactions` unchanged.
- Checked: fits one line at 320px in the FR worst case (`base 350 pts (A −150, réactions +190)`); tests 130/130 incl. the four agreed examples.

**v260926.311 — PRODUCTION (promotion of v260926.310-tmp):**
- Staging file promoted unchanged; only `RUNNING` and the footer changed (`v260926.310-tmp` → `v260926.311`). The 3 remaining `-tmp` strings in the file are the version-check regexes (they exclude staging versions on purpose) — same as before.
- Checklist: version ✅, 351KB (waived) ✅, init block once ✅, no `-tmp` in version ✅, 12 screens ✅, all 4 script blocks `node --check` ✅, test kit ALL GREEN on the production file (130 + 19) ✅.
- Deployed with `deploy.bat` together with this handoff and the updated design doc. Test kit delivered as `tests_kit.zip` → extract into `D:\09_ALTO\` (gives `D:\09_ALTO\tests\`).
- What changes for players: new ¡Alto! scale (no slider), reactions with points (lobby toggle, default on), one reaction per answer and none on your own, Scores cards and Final cards in the daily leaderboard style, awards and share, Revisar keeps decisions, refresh restores each tab's own player, Jugar de nuevo starts from 0, solo and unfinished games don't count, ties share medals and wins, names shown before emojis.
- Leaderboard: still the OLD `global/` + `groups/` system until Build 5 (with the new practice / tie rules already applied). No Firebase console steps were needed for this release.
- Mixed versions during deploy are safe: host scoring ignores self-reactions and multi-reactions from old clients; new screens fall back to the simple list for rooms without `roundLog`; the auto-reload moves everyone to v311 on their next load.

---

### Session 25 (Sep 26–29) - Build 5a on staging, Firebase rules, AI outage → PRODUCTION v260926.313

**Firebase rules — urgent fix.** The database was on "test mode" rules expiring 28 Sep 01:00 (all reads/writes would have been refused). Replaced with permanent per-path rules (see Credentials → Firebase); Oscar published and checked daily / create room / join. Includes the Build 5 paths and the `lastPlayed` index.

**Build 5a — groups infrastructure (staging v260926.312-tmp, then v260926.314-tmp = 5a + AI fix). NOT TESTED BY OSCAR YET.**
- Mis grupos on each device (`alto_mygroups`, max 10, default pinned, most recently played preselected; the default cannot be removed but can be renamed). Default group created on first host: "Grupo de Oscar" / "Oscar's group" / "Groupe d'Oscar" (FR elision before vowels), written once in the creating host's language.
- Lobby picker replaces the free-text group box: `🎮 name ▾` opens an inline panel (top 3 + "Ver todos (N)", × on non-default, 📌 on default, ✏️ Cambiar nombre, + Nuevo grupo; input on its own line). `groupId` + `groupName` written to the room as soon as the host picks; guest card shows `🎮 name` live (handleRoom lobby branch). Rename → local list + room + `groupsMeta` transaction (fills createdAt/lastPlayed if missing). Host lobby refreshes group names from `groupsMeta` in the background (renames from other devices).
- `endGame`: ignores a second call once the room is `final` (double tap); real games (not solo, all rounds) with a groupId write `groupsMeta` (lastPlayed) → `groupsLb` (ties share wins, lastEmoji) → `history` + `historyIndex`, then `phase:'final'`; after that, housekeeping (trim to newest 20 games; delete up to 3 groups idle > 1 year across all 4 paths). Group-write failures never block the Final screen. No more writes to `global/`, `groups/`, `groupNames/`.
- Every player who reaches Final of a real game gets the group added/bumped in Mis grupos. Marcador "Mi grupo" tab reads `groupsLb/{groupId}` (names escaped, `partidas` via T key). Mundial tab left untouched (frozen old data) until 5b. Welcome-back shows `room.groupName` (moved from 5b).
- T keys: `grpLabel, grpNew, grpRename, grpSave, grpSeeAll` (ES/EN/FR); removed `groupName, groupHint, groupHint2`. Removed `getSavedGroups, saveGroupName, onGroupInput, selectGroup, hideGroupSuggestions, prefillGroup, globalKey, groupKey`.
- Checked at 414/390/375/360/320 px (ES + long FR names): no overflow.
- Oscar's test list (staging): default group + guest line; picker new/rename/switch/× with the guest watching; refresh; full game → Marcador + Firebase paths; guest hosts next → family group preselected; unfinished game not counted.

**AI outage (29 Sep) → v260926.313 PRODUCTION (AI fix only, no 5a).** OpenRouter withdrew the free Ling (see Credentials → OpenRouter). Daily/practice/contests sent only that model → 404 → everything dudoso (multiplayer survived via its fallback list, but fell to Nemotron, which answers "not sure"). Fix: shared `AI_MODELS` list, paid Ling first, sent as `models` everywhere (`dailyAICallModel` accepts a list or a single name; contest `workerCall(AI_MODELS)`; multiplayer `models: AI_MODELS`). Checks: version before/after, syntax, 12 screens, Session 24 kit 149/149 on the production file, request-body check. Oscar confirmed ✅/❌ back on production. Same change applied to staging (v260926.314-tmp).

**Cleanup of 29 Sep daily (console scripts on jugarenfamilia.es, Firebase REST):**
- Reset contest marks `names/*/contests/2026-09-29` → 5 players (david, marina, demarte, cocorico, mathieu) got their ¿Error? buttons back.
- Revalidation of every 🤔 answer still unjudged (skipping contested ones) with the daily prompt + paid Ling; trial run first (47 changes), then real run: 48 answers changed, 0 AI errors, 9 players updated; scores recalculated with the game formula (valid 10, unsure 5, × speedMultiplier). Manual overrides agreed with Oscar: Góngora valid, Jus de pomme valid, "5" (Partie du corps) invalid; fix-ups: Cocoricó "González" valid, David "Juan Carlos I" valid; David's "Jondanie" (typo, not Jordanie) set valid by mistake then reverted to invalid.

**AI scout** built (see Test kit → Tools). First run in progress at the end of the session: GLM 5.2 free DEAD (404); paid Ling 13/14; **`inclusionai/ling-3.0-flash-sante:free` 14/14, no flips — candidate to go first in `AI_MODELS` with paid Ling as safety net** (decide when the full table is in); `qwen/qwen3.8-27b:free` all 429 (overloaded).

**AI scout results (29 Sep, scout v3: 15 daily questions = 5 per language + 9 contests) → PRODUCTION v260926.315 / staging v260926.316-tmp.** Sante free 15/15 · 9/9 · 2.1 s; Cohere North Mini Code free 15/15 · 9/9 · 4.1 s; Liquid LFM 2.5 (2.6B) free 15/15 · 9/9 · 4.8 s (small model, not trusted beyond the test); dots-3 note free 15/15 · 7 s; Nemotron Super free 15/15 but NVIDIA "temporarily overloaded" errors; paid Ling Fin 14/15 (missed "5" as body part) · 9/9 · 2.4 s; Nemotron content-safety = always unsure (a safety filter); GLM 5.2 free GONE (paid only now); Inkling models need agent harnesses (403); Qwen 3.8 + Gemma 4 free BUSY. **New `AI_MODELS` = Sante free → Cohere free → Ling Fin paid** (the paid one only runs if both free fail). Checks: version before/after, syntax, 12 screens, request body, Session 24 kit on production, full kit on staging — all green. **Oscar to check: a multiplayer game with the 🤖 button** — multiplayer uses a DIFFERENT prompt the scout does not test (Nemotron scored 15/15 on the daily prompt but answered "not sure" in multiplayer). Best contest judges for the later "contests use a different model" item: Sante, Ling Fin, Cohere (all 9/9). Scout v3 kept in `tools/ai_scout.js` (also skips busy models after 4 × 429, shows GONE / BLOCKED (privacy) / BUSY / NO CREDIT, per-language ES·EN·FR columns, suggests only models with 4+/5 in every language).

**One judge prompt for every mode → PRODUCTION v260926.317 / staging v260926.318-tmp.** Multiplayer (🤖 button, `askAI`) had its own older prompts (separate Relajado/Estricto texts, never asked about the starting letter, a language rule built but never sent). Now `dailyBuildPrompt(category, letter, opts)` / `dailyBuildPromptWithWord(word, category, letter, opts)` is THE judge prompt: no `opts` = the daily prompt, byte-identical to before (so the scout results hold); multiplayer passes `{ strict: G_aiStrictness === 'estricto', looseSpelling: !!G_aiSpelling }`: Estricto swaps rule (2) to "only well-known, clearly belonging" and makes doubt → INVALIDO; Relajado appends "prefer VALIDO on any close call"; spelling allowed swaps rules (4)/(5). Multiplayer now also checks the starting letter. Kept as they were: multiplayer answer reading (also accepts yes/no) and its 12 s timeout; contest prompt unchanged. Checks: daily prompt identical to v315 for 3 langs × 3 answers, each option changes the text, askAI sends the shared prompt (both option combos), checklist, Session 24 kit on production, full kit on staging — all green. Not re-scouted (Oscar's call: same base prompt).

**Process notes:** my sandbox dropped for a while mid-session (files survived). Test harness lesson: fake DBs in script tests must apply writes, or multi-step fixes look wrong.

---

### Session 26 (Sep 29) - Build 5b + round/validation fixes on staging v260929.319-tmp (NOT TESTED BY OSCAR YET)

**Start state:** production v260926.317, staging v260926.318-tmp (5a). Kit on staging ALL GREEN before starting. Oscar: multiplayer 🤖 works on staging (one solo practice game — note: solo games do not exercise the group features). Built on top of 5a so all of Build 5 is tested together. Plan agreed item by item before building (items 1-8 below).

**1. Own card open on the round Scores screen.** `showScores`: once per round your own card starts open (`G_scAutoOpened`); others closed. If you close it, it stays closed for that round (the screen redraws on every room update). Late roundLog in the same round still opens it once. A player not in the round gets nothing opened. Final screen unchanged (all collapsed).

**2. Group page = the Leaderboard screen (`s-leaderboard`), two tabs (Oscar chose tabs + word badge).** Title `🎮 group name` (`#lb-group-name`, now the h1). Tabs `#btn-gp-lb` "🏆 Marcador" / `#btn-gp-hist` "📜 Historial" (`.gp-tab.on` = red). Marcador: wins/games/win % rows (`groupLbHTML`), now DENSE medals on tied wins (was `MEDALS[i]`). Historial: nav row `◀ [SVG flag] 25 sept · 21:40 ▶` (`.gp-when` flex row — flags never inline; `gameWhen(ts)` in the viewer's language), ◀ older / ▶ newer, disabled at ends; then winner/tie line, award chips and the player cards of that game rebuilt from the `history` record with the Final code. Host badge: small grey `anfitrión`/`host`/`hôte` (`.mp-host`, 11px) on the host row in history only. Only `historyIndex` loads on open; a game loads when viewed and is cached (`G_gp.cache`); a render token drops stale loads. State in `G_gp {gid,name,from,tab,idx,pos,cache,token,dbg}`.
- Entry points: Final "Marcador" → Marcador tab (`showLeaderboard`); lobby 📜 → Historial (`openHistoryFromLobby`: host = picked `G_group`, guest = room group via `G_lobbyRoomGroup`); `?g=ID` link → Historial (`openGroupFromLink`).
- ← volver (`lbBack`, also the shell logo on this screen): link/debug → home; from a room: phase final → Final; lobby → same lobby (player list + group line caught up) or `enterLobby` if the host started a new game (Jugar de nuevo) meanwhile; other phases → `restoreToScreen`.
- Shared helpers pulled out of `showFinal`: `winnerLineText(room)`, `awardChipsHTML(room)`, `gameCardsHTML(room, prefix)` ('fn' Final / 'hs' history — keeps element ids unique; `finalCardHTML(..., prefix, isHost)`, `toggleFinalCard(i, prefix)`), `mpRowHTML({..., isHost})`.
- Language switch on the group page re-renders it (applyLang).

**3. Mundial removed.** Tab, `global` branch of `renderLb`, `lbTab`, `G_lbTab`, `G_lbRoom` gone (whole old `renderLb` replaced). T keys removed: `myGroup`, `world`, `worldSub`, `noGroupGlobal`, `lbTitle`. `noGroup` reworded ("Esta partida no tiene grupo." — only old rooms). New T keys (ES/EN/FR): `gpTabLb`, `gpTabHist`, `hostBadge`, `histEmpty`, `histError`, `grpNotFound`. Firebase console cleanup of `global/`, `groups/`, `groupNames/` still waits for the production promotion.

**4. Historial buttons.** Host: `[🎮 name ▾] [📜]` (`.grp-row`, `#btn-grp-hist`, 17px icon button). Guest: `🎮 name [📜]` (`#gc-group-row` > `#gc-group` + `#btn-gc-hist`; row hidden with no group). **handleRoom has a new `leaderboard` branch:** countdown/playing pull you into the game (and drop `?g`); lobby changes leave you on the page. Before this, a guest on Marcador when the host started was stuck (pre-existing bug).

**5. `?g=ID` link.** `tryRestore`: without `?room=`, a valid `?g=` opens the group page (Historial) — name from `groupsMeta`, group added to Mis grupos with its lastPlayed; unknown group → toast `grpNotFound` + home; malformed id ignored. `?room=` wins over `?g=`. `?g` stays while viewing (refresh-safe) and is removed on leaving (`setGroupInUrl`). Share text last line: `jugarenfamilia.es/?g=ID` for real group games; practice / unfinished / no group keep `jugarenfamilia.es`.

**7. Revisar takes guests back to validation (both modes).** handleRoom `scores` branch: phase `validate` → guests `showGuestValidation` + `applyVotesAndReactions` + AI results. Everything kept (✕ marks, 👎 votes, reactions, 🤖 results); guests can change votes and reactions (reactions count for points). Banner `#wait-revising` (moved from the Scores screen, T key `hostRevisingScores`) shows whenever the round was already scored (`roundLog[currentRound]` exists), so it also shows after a refresh mid-review. Old `#sc-revising` removed. Recalculate → everyone to the new Scores as before.

**8. Lost taps / "ghost" reactions and thumbs — ROOT CAUSE FOUND AND FIXED.** Not leftover data (checked: Firebase clean at every round start). Cause: on the guest screen every room update redrew buttons nobody touched — every emoji row that had any reaction was re-created on ANY update, and any 👎 anywhere rebuilt the whole guest validation screen. A tap that lands while its button is being replaced is lost → tap twice; worse as rounds go on (more reactions/votes → more redraws), worst in democratic mode. Fix:
- `setHTML(el, html)`: writes innerHTML only if it differs from what it last wrote (`el.__h`). Used by `applyVotesAndReactions`, `reactToEntry`, `voteEntry`.
- `applyVotesAndReactions` now walks every `votes_*` / `emojis_*` element on screen and takes MY state from Firebase for each (so counts go back to 0 when the last vote/reaction is removed, and highlights are always the truth).
- `refreshGuestValidation` no longer rebuilds: only host ✕ marks trigger `applyGuestMarks` (switches `invalid` / `democratic-valid` + badge text in place; badge keeps `data-pts`).
- **Stuck guest (also found):** a guest who got "all answers in" and "phase validate" in one snapshot stayed on "Respuestas enviadas…" and could not react/vote (e.g. back from another app). handleRoom `playing` branch now sends a non-host with all answers in and phase `validate` straight to validation.

**Debug:** `dbg('leaderboard')` now shows the real group page with in-memory data (`G_gp.dbg`: tie on wins, 2 games ES/FR, host María/Carlos) — no Firebase.

**Tests (kit updated — deliver `tests_kit.zip`, extract into `D:\09_ALTO\`):** new `suites/test_build5b_group_page.mjs` (35: own card open/closed/next round/late roundLog/late joiner, dense medals, history order, dates ES/EN/FR, host badge + ids, winner line, share link 4 cases, in-place guest marks with zero rebuilds, setHTML writes once); new `restore/build5b_scenarios.py` (77 real-browser checks with host + 2 guests, democratic, 3 full rounds: one tap always registers, other players' reactions/votes never replace your buttons, both guests reach validation every round, Revisar brings guests back with decisions kept and a changed reaction changes the score, own card open on all 3 devices, lobby 📜 + host start pulls guest in, back targets, Final → Marcador/Historial, flag + date flex row, host badge, EN switch, play-again while browsing, `?g=` on a fresh device, unknown/malformed ids). Existing suites: extraction updated for the new helpers + a `querySelectorAll` mock; build3 own-row regex accepts `expanded`. Totals on v260929.319-tmp: 213 unit + 77 + 28 + 19 = ALL GREEN. iPhone check with real fonts at 414/390/375/360/320 (ES + FR, long names/group names): group page both tabs, host/guest lobby rows, waiting screen — no overflow.

**Oscar's staging test list (all players on `index_tmp.html`, 3+ players for democratic):**
- Lobby: 📜 next to the group (host and guest) opens Historial; guest browsing when host starts is pulled into the game.
- Reactions and 👎 register with ONE tap, every round (the main bug).
- Everyone sees their own card open on the round Scores.
- Revisar: guests go back to validation with the banner; change a vote/reaction; recalculated scores reflect it.
- Full game (all rounds, 2+ players) → Final → Marcador (tied wins share medal) → Historial (flag, date, host badge, cards) → back.
- Compartir text ends with `jugarenfamilia.es/?g=...`; open that link on another phone → group page, group in Mis grupos.
- Plus the Build 5a list from Session 25.

**Last versions: production v260926.317 (unchanged), staging v260929.319-tmp.**

---

### Session 26 (Sep 29, continued) - Judge prompt language rule → PRODUCTION v260929.320 / staging v260929.321-tmp

**Why:** today's FR daily accepted Oscar's "Jabali" as Mammifère (Spanish word; French is "sanglier"), and his ES result accepted "Guava" as Fruta (English; Spanish is "guayaba"). Rule (3) said "Only common nouns must be in {lang}" as half a sentence after a long list of what is allowed in any language; the models read mostly the list. Oscar fixed his own score by hand in Firebase. All other FR scores and verdicts of the day checked and correct (maths, originality, Jars/Jirafe/Julious Caesar/Jaques ❌, Jamaica ✅ under the proper-noun rule).

**Change (prompt only):**
- Rule (3) of `dailyBuildPrompt` (the ONE judge prompt: daily, practice, contest first ruling, multiplayer) now reads: "(3) Language. Proper nouns are international — [same list as before] are accepted in any language. Everything else must be the {lang} word: animals, plants, fruits, food, drinks, objects, body parts, jobs, colours, sports, instruments and any other common noun. A correctly spelled word from another language is INVALIDO even if it means the right thing (e.g. {example}). Loanwords normally used in {lang} are fine (e.g. pizza, sushi, jazz)."
- `FOREIGN_WORD_EXAMPLE` (next to the prompt): FR `Spanish "jabalí" instead of French "sanglier"`, ES `English "guava" instead of Spanish "guayaba"`, EN `French "fraise" instead of English "strawberry"` — always a word from ANOTHER language, so it can never be read as a correct answer.
- Contest prompt (Method B) note: same common-noun sentence added, so a contest cannot overturn a correct ❌ on a foreign word.
- Multiplayer Estricto / Relajado / spelling swaps untouched and checked (they replace rules 2, 4, 5 by exact text).

**AI scout v4 (`tools/ai_scout.js`):** prompt copy updated to v320 (identical to the game in ES/EN/FR — checked by the kit). 6 new daily questions (Guava ES Fruta ❌, Golf ES Deporte ✅, Olivo EN Tree ❌, Orange EN Fruit ✅, Jabali FR Mammifère ❌, Jamaica FR Pays ✅) → 21 questions, 7 per language; 2 new contest questions (Jabali, Guava must stay ❌) → 11. Per-language columns now show /7; suggestion rule is "at most one miss overall, no flips, no errors". **Oscar to run it once on jugarenfamilia.es after deploying v320** to confirm Sante / Cohere / Ling still judge well with the new wording; the v315 results were measured on the old prompt.

**Tests:** new `suites/test_judge_prompt.mjs` (26): rule text + example + loanwords in 3 languages, no unfilled placeholders, Estricto/Relajado/spelling swaps still apply with the language rule kept, contest note, and **scout prompt byte-identical to the game** (fails if they drift). Production v320: diff vs v317 = prompt lines + version only; Session 25 kit gives IDENTICAL results on v317 and v320 (only the expected 5a failures). Staging v321-tmp: full kit ALL GREEN except one build5b browser check that failed once under load and passed 9 reruns in a row (77/77) — flaky timing, not yet identified which check; harden it next session (the scenario uses fixed waits of 450–500 ms).

**Deploy scripts (Session 26, after v320):** `deploy.bat` / `deploy_tmp.bat` updated to install `ai_scout.js` and `tests_kit.zip` (see Deploy Workflow). Not testable in Claude's sandbox (no cmd): checked statically (blocks balanced, no unescaped `( ) >` in echo inside blocks, `TOOLS` flag not read in the block that sets it, success path exits before `:toolsonly`, ASCII + CRLF) and by diff (rest of the scripts unchanged). Oscar copies both `.bat` files into `D:\09_ALTO\` by hand once. Suggested first real test: only `ai_scout.js` in Downloads → run `deploy_tmp.bat` → lands in `tools\`, "GitHub not touched".

**v260929.322-tmp (staging):** Popularidad award emoji 🔥 → 🙌 (it counts fire + applause, so 🔥 was misleading). One line in `computeAwards`; chips, history and share text all follow. Oscar's call; 🫶 rejected (not shown on Windows 10 / older phones).

**Process note (Oscar):** use fewer tool calls — run the test kit only when a change needs it, not for every tiny edit.

**v260929.323-tmp (staging) — refresh during the countdown (found by Oscar in a solo 3-window test):** only the host's page switched the room from `countdown` to `playing`, so a host who refreshed (or whose phone locked) during the 10 s froze the room; guests sat on the lobby screen. Once, the host also landed on the join screen ("Unirse a la sala", name prefilled) and joining there made him a GUEST called Oscar → countdown "1" → back to the same screen, game dead (nobody host). Root cause of that lost session NOT found (normal Chrome; in a repeat test the session survived: `{"isHost":true}`; Oscar had also refreshed the guests the first time; harness "everyone refreshes" case keeps the session). Fixes:
- `startRoundOnce(code)`: every client whose countdown reaches 0 runs a `runTransaction` on the room that sets `phase:'playing'` + `roundStartTime` ONLY if the phase is still `countdown` (others abort by returning undefined; `null` → return null so Firebase retries with the real value). Host still enters immediately as before; everyone else via handleRoom.
- `quickJoin`: joining with exactly `room.host`'s name makes you host again (`G.isHost = room.host === name`); validate-phase rejoin as host shows the host validation screen. Trade-off accepted: anyone typing the host's exact name becomes host (family game, names visible anyway).
- Tests: new `restore/countdown_scenarios.py` (12: host away → guests start alone; host returns into the round keeping the role; everyone refreshes → all back in; host via join screen with own name → host again, validates; other names stay guests). **Harness fixes:** fake `runTransaction` now aborts on `undefined` like Firebase (it used to write null = delete the room); `wait()` in the scenario files now pumps Playwright events instead of `time.sleep` (sleeping blocked the fake Firebase, which is served from the Python process — very likely the cause of the earlier flaky build5b check). build5b 77/77, restore and groups scenarios re-run on v323.
- Next production promotion number: v260929.324.

**v260929.324 — PRODUCTION (promotion of v260929.323-tmp = Build 5a + 5b + all Session 26 fixes):** tested by Oscar before promotion — 2-player game; solo 3-window game (normal Chrome host, Edge + incognito guests): Revisar brings guests back, no lost taps / ghost reactions, Final scores, Historial with flag + date + anfitrión + 🙌 chip + tied 🥈🥈; countdown refresh (host away, everyone refreshing) recovers. Promotion changed only the version string (diff checked). Checklist: 12 screens, init once, 4 scripts syntax OK, no -tmp, 383KB (waived), unit suites 239/239 on the production file; browser scenarios (77 + 12 + 19 + 28) ran on the identical v323-tmp.
- **Oscar, before deploying:** export a Firebase backup (Realtime Database → ⋮ → Export JSON).
- **Oscar, 1–2 weeks after, if all is well:** delete `global/`, `groups/`, `groupNames/` in the console and remove their three lines from the rules. Until then `rollback.bat` can still bring back the old leaderboard with its data.
- From this release the old global/group leaderboard is no longer written; groups start empty (clean start, as designed). Everyone's Mis grupos starts from their default group.

**AI scout v4 results (Oscar, 29 Sep, prompt v320):** Cohere north-mini-code:free 21/21 (7/7 each language), ¿Error? 11/11, 0 flips, 4.6 s. Sante ling-3.0-flash-sante:free 21/21, 11/11, **1 flip**, 2.2 s. Ling Fin (paid) 20/21 — **accepted FR Jabali** and overturned it in ¿Error?, 2 flips. dots-3-note-preview:free 21/21, 10/11 (overturned EN Orange), 0 flips — preview model. Nemotron 3 Ultra 21/21 but 2 timeouts. Everything else failed, busy or unavailable (inkling 403 = agentic harnesses only).

**v260929.325 — PRODUCTION (+ staging v260929.326-tmp, same code):** `AI_MODELS` = Cohere → Sante → Ling Fin (was Sante → Cohere → Ling Fin). Oscar wanted a paid fallback kept (free models are often busy at the same time; a slightly weaker verdict beats none), so Ling Fin stays last. dots NOT added: OpenRouter documents a **3-entry cap** on fallback lists (400 error above that — documented for the Messages `fallbacks` param; not stated for the `models` list we use, so we don't risk a 4th entry). Comment above `AI_MODELS` records the scout numbers and the cap. Checks: version grep before/after, 12 screens, init once, syntax, prod diff 9 lines (order, comment, version). No kit run: the change is the model list only; the scout is its test.

**Help pages — Spanish DRAFT, parked by Oscar ("we need to think about this more"). Not built.** The ? shows one of four texts: home/welcome → `homeRulesHTML` (overview of both modes); daily-play/daily-result (also Práctica) → `dailyRulesHTML`; every multiplayer screen → `rulesHTML` + `helpHTML`; lobby "¿Cómo se juega?" box → `rulesHTML`. Wrong today: helpHTML Marcador (global leaderboard), Grupo ("type a group name"), daily "the Robot's decision is final" (¿Error? exists, in dailyRulesHTML and homeRulesHTML d4). Missing: reactions + points + switch, awards (🙌 😂 ✨, also with points off), ties share medal/win, practice/unfinished games don't count, groups/Mis grupos/history/📜/share link, Revisar brings everyone back, Scores cards (own opens), daily dudoso 50, Práctica on the daily help.
Draft (ES):
- **rulesHTML steps:** 🎲 Cada ronda sale una letra al azar. Todas las respuestas deben empezar por esa letra · ✏️ Rellena todas las categorías lo más rápido posible · 🛑 Cuando tengas todas, pulsa ¡Alto! y la ronda termina para todos. Si se acaba el tiempo, se envía lo que haya · 🔍 Se revisan las respuestas, se reacciona con emojis y se calculan los puntos. **Scoring:** única 100 · repetida 50 · anulada o en blanco 0 · ¡Alto! todo válido +250 · ¡Alto! por cada anulada −100 · 🔥 recibido (válida) +30 · 👏 (válida) +20 · 😂 (cualquier respuesta) +10 · 😬 0. **Note box:** El bonus de ¡Alto! está calculado para 8 categorías; con menos, es proporcional. Las reacciones solo suman si el anfitrión activa «Reacciones con puntos». Una sola reacción por respuesta, y nunca a las tuyas. Cuentan las que haya cuando el anfitrión pulsa Calcular.
- **helpHTML sections:** 🔍 Revisar respuestas (📖 Wikipedia, todos · 🤖 Robot orientativo, decide el anfitrión · ✕ anula / ↩ recupera · ← Revisar: todos vuelven a la revisión con todo lo decidido y aún se pueden cambiar votos y reacciones) · 🗳️ Modo democrático (válidas por defecto; 👎; más de la mitad en contra → anulada; mínimo 3) · 🤖 Asistente Robot (lo activa el anfitrión; comprueba la letra; 🟢 Relajada / 🔴 Estricta / ✏️ Faltas de ortografía) · 😂 Reacciones y premios (🔥 👏 😂 😬; premios 🙌 Popularidad = más 🔥 y 👏, 😂 Risas, ✨ Originalidad = más respuestas únicas válidas; salen aunque las reacciones no den puntos) · 📊 Puntuación y final (tarjetas, la tuya se abre sola; final con ganador, premios, rondas y Compartir; empate = comparten medalla y victoria) · 🎮 Grupos (el anfitrión elige entre sus grupos o crea uno; nadie escribe ni recuerda nada; al terminar, el grupo se añade a la lista de todos los que jugaron) · 📜 Marcador e historial (📜 en la sala o Marcador al final; 🏆 victorias/partidas/%; 📜 últimas 20 partidas con ◀ ▶; el enlace de Compartir abre el historial en cualquier móvil) · ✏️ Partidas que no cuentan (jugar solo y partidas terminadas antes de la última ronda = práctica).
- **dailyRulesHTML:** 📅 Una letra y 6 categorías, iguales para todo el mundo cada día; cada idioma tiene su propio reto · ✏️ Rellena y pulsa ¡Alto! en cuanto termines: cuanto antes, más bonus · 🤖 El Robot valida cada respuesta; ¿Error? para que lo revise otra vez (una vez por respuesta) · ✏️ Práctica: mismas reglas, letra al azar, intentos ilimitados, no cuenta para la clasificación. **Scoring:** válida 100 · dudosa 🤔 50 · inválida o en blanco 0 · velocidad ×1.0–2.0 · originalidad (nadie más la puso) +50.
- **homeRulesHTML:** shorter versions of the above + «Tus partidas se guardan en el historial de tu grupo».
- **Left out on purpose:** category modes, name reservation pill, rejoin-with-your-name, yearly clean-up.
- **Open decisions:** (1) keep «Cuentan las que haya cuando el anfitrión pulsa Calcular»? (2) add «En caso de duda, decide el anfitrión» (tomato-type disputes)? (3) Ayuda is about twice as long — fine in the scrolling panel, or trim? Then translate EN/FR and build on staging.

**¿Error? clarified (Oscar asked):** it asks ONE model — the first available in `AI_MODELS` (the list is only OpenRouter's fallback chain) — with the generous contest prompt; VALIDO overturns, INVALIDO/DUDOSO upholds. The old "1-of-3 threshold" wording was only a stale comment. Oscar chose to keep it single-model (asking all three would let Ling Fin's Jabali / dots' Orange mistakes through). Comment above `CONTEST_METHOD` corrected in v327.

**Daily EN leaderboard review (Oscar's screenshot, 30 Sep):** scores correct; "Koreano" ❌ = the v320 language rule working. Generous verdicts noted (not acted on): "Kite" as Transport ✅, "Katy y yo" as Film ✅ (title not known).

**v260929.327-tmp → PRODUCTION v260929.328 — "play to see the answers" line (`#daily-lb-not-played`):** it lingered under a board you are allowed to see and could show in the wrong language. Only the flag tap (`switchDailyLbLang`) set it; the result screen, ◀ ▶, ↻ refresh, ¿Error? reload and a UI language switch did not, and a language switch never re-translated it. Fix: `_lbUpdateNotPlayedHint()` (shown only for TODAY on a tab you have not played — `_playedLangs` or localStorage; text refreshed every time, emptied when hidden) called from `_lbUpdateNavUI()` (day nav, result screen — now after the tab is set — and `applyDailyLang` on language switch), from `switchDailyLbLang` and from refresh. Checked: helper in 5 cases (node), setLang → applyLang → applyDailyLang path confirmed, checklist; Oscar tested on staging. No full kit (change limited to this line).

**v260929.329-tmp → v330-tmp → PRODUCTION v260929.331 — News on the home page (Oscar's idea):** short messages to tell players about outages and new features. Oscar chose to keep them **in the HTML** (not Firebase), so each new message = a deploy; Claude writes ES/EN/FR on request.
- `NEWS` array (above `getDailyKey`): `{ id, type: 'info'|'warning', until: 'YYYY-MM-DD', es, en?, fr? }`. **All** active messages show, in array order, stacked under the home main card (`#news-card.news-list`, one `.news-item` each with its own ✕). Expired (`until` < today, local date) or dismissed ones are skipped; container hidden when empty.
- ✕ → `dismissNews(id)`: id stored in localStorage `alto_news_dismissed` (last 50), hidden on that device for good.
- Style: one line, 13px Special Elite on `--paper-dark`, dashed border; **warning = red left edge only** (Oscar: subtle is fine, no pink background). Rendered from `applyLang` (language switch + init).
- **Writing rules (Oscar):** very short, non-technical, ONE line, start with an emoji, no jargon (no "anfitrión", "revisar"…), avoid "ayer"/"yesterday" (messages stay up for days). FR may wrap to 2 lines on 320px phones — acceptable.
- Messages in v331: `2026-09-30-robot` warning until 2026-10-03 "🤖 ¡Robot arreglado! Perdón por el fallo" / "🤖 Robot fixed! Sorry for the trouble" / "🤖 Robot réparé ! Désolés pour la panne"; `2026-09-30-historial` info until 2026-10-14 "📜 Nuevo: el historial de tu grupo" / "📜 New: your group's game history" / "📜 Nouveau : l'historique du groupe".
- Checked: phone widths 390 (ES) and 320 (FR) with real fonts — no overflow from the news, one line at 390, ✕ + reload keep it hidden, no page errors.
- **Found on the way (not fixed, pre-existing):** at 320 px the home emoji grid is slightly too wide — the last column is cut on the right. Flag for a small layout fix.

**Last versions: production v260929.331, staging v260929.330-tmp (same code).**

---

### Session 27 (1–2 Oct 2026)

**Practice "Tiro al Arco" ❌ (Modalidad olímpica):** Oscar agreed it is right — the name is "tiro con arco" ("tiro al arco" is Latin American usage). No change.

**Firebase console "read-only and non-real-time mode":** the console does this on big nodes (`daily`, `rooms`…). Fix: open a small node directly by URL, `/` written as `~2F`, e.g. `https://console.firebase.google.com/project/stop-9f0ea/database/stop-9f0ea-default-rtdb/data/~2Fdaily~2F2026-09-30~2Fes~2Fscores`, or click down level by level until the node is small.

**Oscar edited some daily answers by hand on 1 Oct** (FR Ratatouille: spelling corrected + made valid — "first time cheating"). Answers in the 1 Oct export may not be what the Robot judged; ask before drawing conclusions from them.

**Finding (1 Oct daily export): the AI ran out of tokens.** 11 of 26 saved replies were 1000 chars of reasoning ("Let's think…", one loop repeating "There's Bentley"). With `max_tokens: 800`, thinking models used everything on reasoning and returned an EMPTY answer; the code then fell back to `msg.reasoning` and took the FIRST "invalid"/"no"/"valid" anywhere — so those verdicts were close to random (the reasoning even quotes the prompt "…is INVALIDO even if…"). Running out of tokens is not an error for OpenRouter, so the `models` fallback list does NOT move on. Clean wrong verdicts noted separately (not a token problem): Gin tonic ❌ Bebida, Ratatouille ❌ Film d'animation, Robin ✅ Oiseau FR (English word; Oscar's answer, judged ✅ on first pass — no reply saved), Broccoli ✅ Verdura ES (Spanish = brócoli). "Baritono, saxofon" ❌ Instrumento judged fair (flipping words to fit the letter is fine for people's names, not for objects; "Barítono" alone is a real brass instrument).

**v261001.332-tmp → PRODUCTION v261002.333 — one shared AI judge call:**
- New `aiJudgeCall(systemPrompt, prompt, models, timeoutMs)` (+ `aiParseVerdict`, `aiWords`, constants `AI_MAX_TOKENS = 2000`, `AI_TIMEOUT_MS = 20000`, `AI_VERDICT_WORDS`) right above `dailyAICallModel`. Used by the daily/practice (`dailyAICallModel` is now a never-throwing wrapper), ¿Error? contests (`parseVerdict` and `workerCall` removed) and the multiplayer 🤖 (`askAI`). Previously three copies with three different parsers.
- `max_tokens` 800 → 2000 (a ceiling, not a target: short answers are unchanged). All three timeouts now 20 s (multiplayer was 12, contest 15).
- Verdict = LAST VALIDO/INVALIDO/DUDOSO word of the answer (accents stripped). Plain yes/sí/no only for replies of ≤3 words. Reasoning text is never mined, except when the reasoning itself ENDS with a verdict word.
- Empty answer or cut off with no verdict → the same `AI_MODELS` list is asked ONCE more (Oscar chose same list, not jumping to another model) → still nothing → 🤔 dudoso (contest: upheld, as before). HTTP/network errors still throw to the caller as before.
- Saved raw text now keeps the END of the text (`slice(-1000)`); empty answers are saved as `[reasoning only, cut] …tail`.
- New daily score field **`aiInfo`**: per answer `{ model, cut?, retried?, error? }` (model = `data.model` from OpenRouter). Contest log entries gain `model`, `cut`, `retried`. Multiplayer logs `[AI] multiplayer {word, category, cls, model, cut, retried}` to the console only.
- **Bug fixed on the way:** multiplayer `askAI` catch used `isTimeout` without defining it → the catch itself threw, the 🤖 button stayed on ⏳ and "Robot lento… Reintentar" never showed. Now defined (`TimeoutError`/`AbortError`).
- `tools/ai_scout.js`: `max_tokens` 800 → 2000 so the scout measures what the game does.
- Downsides discussed with Oscar: bad cases are slower (up to ~2×20 s for one answer; the daily judges its 6 answers one by one), retries add requests on free models, a long paid Ling answer costs up to ~2.5× more (fractions of a cent). If speed becomes a problem: OpenRouter `reasoning` effort setting — test with the scout first, not all models honour it.
- Checked: version grep before/after, 12 screens, init once, 4 scripts syntax OK, 3 `-tmp` left = the version-check regexes (as before), 387 KB (waived), unit suites 239/239 on staging and production files, 17 mock tests of `aiJudgeCall` (retry → valid, double empty → dudoso + cut + tagged raw, reasoning ending in verdict accepted without retry, last word wins, long "no" ignored, model recorded, HTTP error throws). Browser scenarios not run (no screen flow touched). Promotion changed only the version string (diff checked).
- **Staging test (Oscar, 2 Oct daily ES):** 5 answers all `cohere/north-mini-code:free`, all verdicts from real answers, one `retried: true` (Broccoli) without `cut` — Cohere returned an empty reply without hitting the limit; watch whether this repeats (maybe Cohere reports running out differently). Same day on production v331 the old problem was still there: 8 reasoning texts, likely wrong calls Bimi ❌, Blanco roto ❌, Roma 🤔 (loop "There's Ratatouille…").

**Ideas noted, not built:** save the AI reply for ✅ answers too (to see why e.g. Robin passed); English-word examples in the FR/ES prompt rule (prompt change → re-run scout).

**PRODUCTION v261002.334 — News message (straight to production, Oscar's call):** `2026-09-30-robot` apology removed (was expiring 3 Oct); new info `2026-10-02-robots` until 2026-10-16: "🤖 ¡Robots mejorados! Ahora piensan más" / "🤖 Robots upgraded: they think first!" / "🤖 Robots améliorés ! Ils pensent mieux". Oscar first chose "…Ahora piensan antes de hablar" but it wrapped to 2 lines on every phone width (checked with real CSS + Special Elite at 414–320 px); the short version is one line from 360 px up, 2 lines at 320 (same as the history message). Checklist OK; diff = news lines + version only.

**PRODUCTION v261002.335 — ¿Error? shows it is thinking (Oscar: "it just stays there the same"):** before, the only feedback was opacity 0.5 → 0.3 (inline). Now `recontest` swaps the label to `⏳…` (+ `title` "Revisando… / Checking… / Vérification…") and adds class `.thinking` (pulse `@keyframes reconsiderPulse` 0.35↔1, no underline, `min-width: 3.9em`). On a connection error the class, label and title are restored so it can be tried again; overturned/upheld paths unchanged (row updates or button removed + toast). The agreed label "⏳ Revisando…" was dropped after a render check: it squeezed the answer column and made the row jump (42→62 px at 390); `⏳…` with the min-width keeps the row height identical to "¿Error?"/"Erreur?" at 390 and 320 px (real CSS + Special Elite). Font size unchanged (11px, existing). Checklist OK.

**PRODUCTION v261002.336 — tagline removed, new Google description (Oscar: the tagline was "a bit cheesy"):**
- "el juego de siempre, con los de siempre" removed from under the logo on all three screens (quick-join `#qj-tagline`, plus the two `.logo-tag` blocks), together with the `tagline` T keys (ES/EN/FR), the `.logo-tag` CSS rule, the `.logo-tag` loop in `applyLang` and the `qj-tagline` lines in `applyQuickJoinLang`.
- `<meta name="description">`, `og:description` and `twitter:description` (the last two were English) now all: "El clásico Stop o Tutti Frutti de papel y lápiz, ahora online: juega con familia y amigos, con corrección automática y reto diario. Gratis y sin registro." (~154 chars). Oscar chose it over other options; "corrección automática" preferred to "autovalidación" / "el robot corrige" (plainer for people who have not played).
- **Not changed (flag):** `og:title` / `twitter:title` are still English ("¡Alto! — Play with Family & Friends") while the description is Spanish and `og:locale` is es_ES. Ask Oscar whether to make the share title Spanish. WhatsApp/Google cache previews, so changes show up with a delay.
- Checklist OK, unit suites 239/239.

**PRODUCTION v261002.337 — share title + locale cleanup:** link previews are ONE for everybody (meta tags are static; WhatsApp/Google don't run JS or know the player's language). Per-language previews would need separate URLs (`/en/`, `/fr/` mini pages with their own meta that send into the game) — parked, only if Oscar starts sharing with EN/FR groups. Chosen instead: trilingual via the game's names — `og:title` and `twitter:title` = "¡Alto! — Stop · Tutti Frutti · Petit Bac" (Petit Bac = French name; "Scattergories" left out, it is a brand). Description stays Spanish (v336). Removed stray `og:locale:alternate` de_DE, pt_BR, it_IT (kept en_GB, fr_FR). `<title>` (browser tab + Google headline) unchanged: "¡Alto! — JugarEnFamilia.es" — offered to Oscar as an option, not changed. Checklist OK.

**PRODUCTION v261002.338 — page `<title>`** (browser tab + Google headline) = "¡Alto! — Stop · Tutti Frutti · Petit Bac", same as the share title (was "¡Alto! — JugarEnFamilia.es"; the domain already shows in Google's result line). Nothing in the JS sets `document.title`. Checklist OK.

**7 Oct — players complain the daily validation is slow.** Exports 6+7 Oct (~106 judged answers): zero `cut`, zero `retried`, Cohere answers cleanly — the token fix worked. But **13 answers (~12%) had `error: true, model: ""`** = almost certainly the 20 s timeout: each cost a full 20 s wait (answers were judged one by one) AND became 🤔 even when obviously right (Vela, Victor Hugo, Vodka, Rondó, Ventana…); every contested one was overturned. A timeout cancels the request, so the Sante/Ling fallback never gets a chance.
Quality notes (Cohere north-mini-code, a small coding model): mostly right on clear cases; slips: "Vivora" ✅ ×2 (b/v misspelling), "Tarragon" ✅ as French spice (English word — same pattern as Robin, Broccoli), flipped words inconsistent ("Tower, Eiffel" ✅ vs "Valencia, agua de" ❌). Next quality step (not built): scout re-run with these real cases, then sharpen the prompt (swapped b/v, g/j, missing h = misspelled) or try a stronger model first.

**PRODUCTION v261007.339 — faster daily judging:**
- Daily answers judged **3 at a time** (`AI_PARALLEL = 3`, small worker pool) instead of one by one. 3 rather than 6 to stay under free-model per-minute limits when several people play.
- `aiJudgeCall`: on a **timeout** the same `AI_MODELS` list is asked once more with a fresh 20 s; a second timeout throws (marked `timedOut`, `ms`). Other HTTP errors still throw straight away. The empty/cut retry is skipped when a timeout retry already happened (max 2 calls per answer). Applies to daily, contests and multiplayer.
- Returns `ms`; `dailyAICallModel` error result now has `error: 'timeout' | 'http'` instead of `true`.
- `aiInfo` per answer now: `{ model, s (seconds, 1 decimal), cut?, retried?, timeout?, error? }`. Old entries have `error: true`.
- Checked: version grep before/after (338 → 339), 12 screens, init once, 4 scripts syntax OK, unit suites 239/239, 16 mock tests (timeout → retry → answer; 2 timeouts → dudoso with error 'timeout'; 429 → 'http'; clean answer = 1 call; pool never above 3 in flight, 6 answers in ~2 rounds).
- Note: the sandbox was reset during the session; v339 was built from the delivered v338 file, not the zip.

**STAGING v261007.340-tmp (built on production v339) — Oscar's feedback after a multiplayer with groups + a daily played twice:**
1. **Daily played twice with the same name.** Cause: the "already played" check used `_playedLangs`, filled ONCE at page load (`fetchPlayedState`) — a tab left open since earlier, a tap before it loaded, or a name change slipped through. Fix: (a) `startDailyChallenge` does a fresh `get(names/{safe}/played/{date}/{lang})` before the cached check (skipped for `__` test names and debug); (b) at submit, a `runTransaction` on that same path takes the slot (`cur ? undefined : playerKey`) BEFORE the score is written; if another device/tab already has it → no second score, the local `alto_daily_…` is removed, toast "Ya jugaste hoy con este nombre" and `startDailyChallenge()` shows the saved score. Offline/transaction error → saves as before.
2. **Group list UI.** 📌 removed (it read as "pin this"; `.grp-pin` CSS gone): the default group is simply first with no ×. × now 20px, `--ink-sec`, 44px wide. Tapping × opens an inline row "¿Quitar «X»? Sí · No" (`G_grpConfirm`, `askRemoveMyGroup`, `cancelRemoveMyGroup`, T keys `grpRemoveQ`/`grpYes`/`grpNo` ES/EN/FR, CSS `.grp-confirm`, `.grp-cbtn`). "Quitar", not "Borrar": it only removes from the list, history stays. `.grp-item` got `min-height: 40px` so the confirm row doesn't jump.
3. **Groups follow a registered name.** Only when the current name is registered AND verified on this device (`alto_claim_{safe}.verified`). Firebase `names/{safe}/groups/{id} = {name, last, def?}` and `names/{safe}/groupsRemoved/{id} = time`. `syncMyGroups()` pulls + merges (removed elsewhere → removed here unless played here after; remote default wins over a local one, an unplayed local default is dropped and tombstoned, a played one becomes a normal group; newer `last` brings its name), then `saveMyGroups` pushes. `saveMyGroups` → `pushMyGroups` (additive per group, only after a pull this session: `_grpSyncedFor`). `removeMyGroupUI` → `pushMyGroupRemoval`. Pull triggers: page load, claim pill finding a verified name, successful claim/verify, opening the group panel. `names/` is open read/write (known; the security build closes it) — someone technical could read a name's group list.
- Checked: version grep (339 → 340-tmp), 12 screens, init once, 4 scripts syntax OK, unit suites 239/239, 11 two-device sync tests on a fake DB (merge, one default, removal propagates, re-played survives removal, unverified device neither gets nor pushes, default not removable, two remote defaults → one), panel render at 390/320 (rows 40px, no overflow). Daily dedup not browser-tested (needs two live devices) — Oscar to test on staging.
- **To test on staging:** daily in two tabs/devices with the same name (second must show the first score); group × → confirm; registered name verified on phone + computer → same groups on both, removing on one removes on the other.

**STAGING v261007.341-tmp (= v340 + multiplayer stuck-round fixes).** Reports: Oscar's 4-player democratic game stuck after round 1 on "Esperando a los demás…" with numbers flying up; his brother's 5-player game stuck at the end of round 3 (suspected a player leaving or two ¡Alto! at once).
- **Cause of the hang:** the round only moves to review when `allAnswers` has EVERY name in `room.players`. `leaveGame` only cleared the leaver's own phone — the room still waited for them forever. Same for a locked phone / lost connection. Two simultaneous ¡Alto! do NOT hang (both submit); they only let the second press overwrite `stopCaller`.
- **Numbers flying up:** the floating countdown (`setTimerBubble`/`spawnTimerFloat`) was never stopped on ¡Alto!/submit, only when a player's own timer reached 0 — it kept spawning the frozen number on the waiting screen. Now `setTimerBubble(0,false)` in `doSubmit` and in the calledStop branch of `handleRoom`.
- New `activePlayers(room)` (players minus `room.left`), `missingAnswers(room)`, `allAnswered` uses them. `leaveGame` writes `left/{name}: true` (fire and forget); every join/rejoin write (`players/…` in join, create-room payload, `continueSession`) clears `left/{name}`. Democratic majority (both places) counts only players not `left` (inlined, so the test kit's isolated `finishValidation` still runs).
- Waiting screen: `renderWaitingFor(room)` shows "✏️ Respuestas enviadas. Esperando a: Marina, Ana"; 15 s after the host's own submit (`G.submittedAt`, `WAIT_SKIP_MS`) the host gets `button.pri` "Seguir sin Marina, Ana →" → `skipMissingPlayers()` writes `allAnswers/{p} = {skipped:true}` (blank answers, 0 pts, still in the game). `doSubmit` no longer writes answers if the room already left 'playing' (a late phone waking up after a skip), so it just goes to review. The old hard-coded Spanish waiting text is now T keys `waitingOthers` / `waitingFor` / `skipMissing` (ES/EN/FR).
- ¡Alto!: `runTransaction` on `rooms/{code}/stopCaller` (`cur ? undefined : me`), then `calledStop:true` only if committed; on error falls back to the old write.
- Not handled: closing the tab/app without tapping leave doesn't mark `left` (unreliable on phones) — the host's skip button covers it.
- Checked: version grep (340-tmp → 341-tmp), 12 screens, 4 scripts syntax OK, unit suites 239/239, 8 logic tests (left player not awaited, missing names, skip unblocks, voters count, empty room, first ¡Alto! wins). Not live-tested with several phones — Oscar to test: 3+ players, one locks the phone before ¡Alto! → host sees "Esperando a…" and after 15 s "Seguir sin…"; one taps leave mid-round → round continues without them; two press ¡Alto! together → only the first gets the bonus; floating numbers stop after sending answers.

**STAGING v261007.342-tmp (= v341 + no duplicate group names in my list).** Groups are identified by a hidden random id, so the same name in DIFFERENT people's lists is fine (separate histories). Within one list, `saveGroupEdit` now compares with `normalize()` (capitals/accents/spaces ignored): "+ Nuevo grupo" with a name I already have selects the existing group (toast `grpExists` "Ya tienes «{g}», lo he elegido"); renaming to a name another of my groups has is refused (toast `grpNameTaken`), renaming a group to its own name with different capitals is allowed. T keys ES/EN/FR. Checked: version grep, 12 screens, syntax, suites 239/239.

**STAGING v261007.343-tmp (= v342 + header logo on the join screen).** `shellLogoClick` did nothing on `quickjoin` (only lobby/game/daily/final/leaderboard reacted). Now on `quickjoin`: `G.roomCode = ''`, `setRoomInUrl('')` (drops `?room=` so a refresh doesn't reopen the join screen), `show('home')`. No confirm needed — not in the room yet. In-game behaviour unchanged (confirm before leaving). Checked: version grep, 12 screens, syntax, suites 239/239, fake-Firebase two-browser scenario (invite link → join screen → logo → home, URL clean, guest not in room, refresh stays home, host still in lobby, no page errors).

**PRODUCTION v261007.344 = staging v343-tmp promoted (v340–343).** Oscar promoted without a hand-made multiplayer test, so before promoting the full browser kit was run against it with the fake Firebase: restore 19/19, countdown 12/12, groups 28/28, build5b 77/77, plus a NEW suite `tests/restore/stuck_scenarios.py` (11/11: guest taps leave → `left/Tom`, round goes on, ¡Alto! credited to the first; guest disappears → "Esperando a: Tom", no skip before 15 s, floating numbers stop, skip button after 15 s → review, `allAnswers/Tom.skipped`). Test-kit updates (not app bugs): groups G3 now expects no 📌 and no × on the default group; build5b "flag + date" check no longer hard-codes "sept" (it failed only because it's October). Updated `tests_kit.zip` delivered (deploy.bat installs it into `tests\`). Unit suites 239/239. Promotion changed only the version string (diff checked), 3 `-tmp` = version-check regexes, 12 screens, init once, syntax OK, 399 KB (waived).
Still worth a real-phone check when convenient: daily in two tabs with the same name; group sync phone ↔ computer with a verified registered name.

**PRODUCTION v261008.345 — dailies no longer repeat across languages (Oscar noticed ES = the previous day's FR).** Cause: `getDailySetup` seeded `mulberry32` with `YYYYMMDD + {es:0,en:1,fr:2}`, so FR on day D = EN on D+1 = ES on D+2 (same seed → same letter and categories). E.g. FR 6 Oct = EN 7 Oct = ES 8 Oct (T: Animal prehistórico, Deporte, Especia, Monumento, Nombre, Utensilio); EN 6 Oct = ES 7 Oct (V…). Fix: `dailySeed(key, lang)` — FNV-1a hash of `'alto-daily|' + key + '|' + lang` for dates ≥ `DAILY_SEED_V2_FROM = '2026-10-10'`; older dates keep the old formula so today's/past dailies and in-progress games don't change mid-day. `DAILY_OVERRIDES` (themed days) and practice (random seed) untouched; scores keep their own saved categories. Checked with the real setup code over Oct 2026–Mar 2027: old method 151 exact repeats, new method 0; all 12 dailies of 1/5/8/9 Oct identical to before. Checklist: version grep (344 → 345), 12 screens, init once, syntax, suites 239/239.

**PRODUCTION v261008.346 — every ¿Error? logged inside the daily score.** Oscar couldn't find his re-checks: the full log went to `names/{name}/contests/{date}/{lang}_log/{idx}` and only OVERTURNED ones added a small `contestLog/{idx}: {original, overturned, timestamp}` to the score. Now every contest (upheld or overturned) writes `daily/{date}/{lang}/scores/{playerKey}/contestLog/{idx}` = `{word, category, letter, original, result, overturned (bool), method, aiResponse, model, cut?, retried?, s, timestamp}` (upheld: own `update`; overturned: in the same update as the new score). The `_log` copy under `names/` is no longer written (old entries kept, nothing deleted). The marker `names/{name}/contests/{date}/{lang}/{idx}: true` stays — it enforces one contest per answer across devices. NOTE the old overturned entries use `overturned: 'valid'` (a string), new ones `overturned: true` + `result`. Nothing in the app or test kit reads either log. Checked: version grep (345 → 346), 12 screens, syntax, suites 239/239, fake-Firebase browser run (daily → ¿Error? on one upheld + one overturned answer → both in contestLog with word/category/verdicts/reply/model, aiResults + baseScore updated only for the overturned one, names markers written, no `_log`, no page errors).

**8 Oct daily export review (87 judged answers, first day with `s` timings):** median 7.2 s per answer, 75% under 18 s, 18 over 20 s. 16 timeouts (18%, up from ~12% on 6–7 Oct): 8 recovered on the retry, 8 failed twice → 🤔 after 40 s (Triceratops, Tiranosaurio, Trapo, T-Rex… all rescued by ¿Error?). Cohere judged 78/79, Sante 1 — the fallback chain never helps on a timeout (request cancelled). 2 reasoning loops cut at 2000 ("magasin de produits pour animaux…", "Eisenhower" film). 3-at-a-time works: e.g. Cocoricó's answers summed 141 s, she waited ~55 s. Quality notes: FR "Sportif" (= athlete) confused — Motocross ✅ (wrong, a sport), Maradona ❌ until contest; "Edison, Thomas alba" ❌ vs "Ringer, Catherine" ✅ (flip inconsistent); "Ma sorcière bien-aimée" (real FR title of Bewitched) stayed 🤔 after a contest loop; Molviedro / Marquet 🤔 instead of ❌; generous: "Tomate seco en polvo" ✅ Especia (contest), "Torre" ✅ Monumento. Also: a contest on a 🤔 that is upheld logs `result: 'invalid'` but the answer correctly stays 🤔.

**PRODUCTION v261008.347 — timeout retry skips the first model.** `aiJudgeCall`: on a timeout the retry is sent with `models.slice(1)` (Sante → Ling Fin) instead of the full list, so it doesn't queue at Cohere again. Empty/cut retries still use the same full list (Oscar: same judge). Single-model lists unchanged. Applies to daily, practice, contests, multiplayer. Checked: version grep (346 → 347), 12 screens, syntax, suites 239/239, 7 mock tests (1st try full list; timeout retry = [sante, ling] and answered by sante; empty-answer retry = full list; 2 timeouts still throw; single model unchanged).
**Next (agreed):** re-run the scout with today's real cases added (Sportif/Motocross, Edison flip, Ma sorcière bien-aimée, Molviedro, Tomate seco en polvo) and compare speed; if Sante is as accurate and faster, consider putting it first.

**AI scout v5 (`tools/ai_scout.js`, 8 Oct):** reads replies exactly like the game since v333 (`gameVerdict` = last VALIDO/INVALIDO/DUDOSO word of the answer, ≤3-word yes/no, empty answer → only a reasoning that ENDS with a verdict; contests overturn only on 'valid'), `max_tokens` 2000 and a 20 s limit (`TIMEOUT_MS`) like the game, `AI_MODELS` in the game's order (Cohere → Sante → Ling Fin), `ONLY = 'game'` by default (just those 3, 249 calls, ~30–45 min; `null` = all free models). 13 new daily questions from real 1–8 Oct cases (Triceratops, Tomate seco en polvo, Vivora, Broccoli, "Edison, Thomas Alva", Eisenhower as Film, Thyme, Sportif Motocross/Maradona, Ma sorcière bien-aimée, Molviedro, FR Tarragon, FR Robin) → 34 questions (ES 11, EN 10, FR 13); 4 new contest questions → 15. New columns: flips, avgSec, slowest10pct, timeouts; the old "at most one miss" suggestion is replaced by a ranking (hard real cases, nobody is expected to be perfect). Game prompt copy unchanged — `test_judge_prompt.mjs` 26/26. Dry-run with fake replies completed all 249 calls. Old v4 kept as `tools/ai_scout_v4_backup.js` in the sandbox only.

**Scout v5 results (Oscar, 8 Oct, game models only):** **Sante free is GONE** — OpenRouter: "This model is unavailable for free. The paid version is available now" (it still answered once on the morning of 8 Oct). Cohere free 28/34 (ES 9/11, EN 10/10, FR 9/13), 2 unsure, 4 flips, ¿Error? 8/15, avg 7.5 s, slowest 10% 14.8 s, 3 timeouts, 10 errors "Failed to fetch". Ling Fin (paid) 24/34 (ES 5/11 — mostly lost connections: Góngora, Garrapata…), 3 flips, ¿Error? 12/15, avg 3.5 s, slowest 10% 8.1 s, 2 timeouts, 7 errors. The "Failed to fetch" errors were the connection dropping during the run and were counted as misses — scores not clean.

**PRODUCTION v261008.348 — dead Sante free removed:** `AI_MODELS` = Cohere free → Ling Fin paid (comment above it updated with the v5 numbers). Since v347 a timeout retry skips the first model, so it now goes straight to Ling Fin (fast, paid — about 1 in 5 answers, fractions of a cent; the key's credit limit stays). `DAILY_CONSENSUS` (off) still references `MODELS[2]` — unused. Checked: version grep (347 → 348), 12 screens, syntax, suites 239/239.

**Scout v5.1 (`tools/ai_scout.js`):** a dropped connection is retried once after 10 s; a question whose calls were all lost is NOT counted (per-language and ¿Error? totals shrink instead of counting a miss; table shows "N errors (M lost connection)"); a progress line every 10 calls; sorted by percentage; at the end a "Misses per model" block with every miss in full (the table truncates them). `AI_MODELS` = Cohere → Ling Fin; `EXTRA` = paid Sante (`inclusionai/ling-3.0-flash-sante`) also tested when `ONLY = 'game'`, to compare. Dry-run with a burst of lost connections and a dead model OK. **Next:** Oscar reruns it → decide (1) whether ¿Error? should use Ling Fin (12/15 vs 8/15), (2) whether paid Sante is worth adding, (3) whether to scout all free models (`ONLY = null`) for a new free second.

**Router note (8 Oct):** Oscar's home router needed a restart twice, both times while the scout was running (it rarely drops otherwise). The scout's traffic is tiny (one small request every few seconds), no mechanism known — but don't run the scout on the home network again: use the phone hotspot, and/or `REPEAT = 1`, `PAUSE_MS = 6000`.

**Scout v5.1 results (8 Oct, clean run):** Sante PAID (`inclusionai/ling-3.0-flash-sante`) 30/34 (ES 8/11, EN 10/10, FR 12/13), contests 10/10 (5 contest calls lost to connection), 1 flip, avg 3.0 s, slowest 10% 4.6 s, 0 timeouts; misses Tomate seco en polvo (unsure), Vivora ✅, Broccoli ✅, Motocross ✅. Ling Fin 27/34, 13/15, 0 flips, 3.0/4.8 s, 0 timeouts; misses FR "5" ✅ as body part (!), Tomate seco ✅, Vivora 🤔, Broccoli ✅, Motocross ✅, FR Tarragon ✅, FR Robin 🤔; contest overturned Guava + Tomate seco. Cohere free 27/34, 13/15, 4 flips, 7.4/15.6 s, 1 timeout; misses Tomate seco, Vivora, Motocross, FR Tarragon, FR Robin ✅, Ma sorcière bien-aimée 🤔 (also upheld on contest), Molviedro 🤔. (The earlier v5 run's Cohere ¿Error? 8/15 was noise from lost connections.) **All three miss Vivora, Broccoli, Tomate seco en polvo, Motocross-as-Sportif → prompt issues, not model issues.**
Prices (OpenRouter, 8 Oct): Sante $0.042 in / $0.1232 out per M tokens, one provider (NovitaAI); Ling Fin same on Novita, DeepInfra $0.06 / $0.18. ≈ $0.00003 per judgement, ~30 cents/month at 300 judgements/day.

**PRODUCTION v261008.349 — AI order Sante (paid) → Ling Fin (paid) → Cohere (free)** (Oscar agreed). Normal: Sante judges, ~3 s. Timeout: retry skips Sante → Ling Fin. Credit exhausted: OpenRouter falls through to free Cohere. Comment above `AI_MODELS` records the numbers and prices. Scout `AI_MODELS` updated to the same order, `EXTRA = []`. Checked: version grep (348 → 349), 12 screens, syntax, suites 239/239.
**Next (proposed):** sharpen the judge prompt for the shared misses (swapped letters b/v, g/j, ll/y, missing h = misspelled; "Deportista/Sportif/Athlete" = a person, not a sport; foreign spellings like Broccoli; a spice must be a spice, not a prepared food) → scout ONLY Sante on the hotspot to confirm (prompt change ⇒ update the scout copy too, `test_judge_prompt.mjs` checks they match).

**PRODUCTION v261008.350 — News:** `2026-10-02-robots` ("¡Robots mejorados! Ahora piensan más", contradicted "faster") replaced by `2026-10-08-robot-fast` info until **2026-10-12** (Oscar: news should only stay a few days): "⚡ Robot más rápido y más listo" / "⚡ Faster, smarter Robot" / "⚡ Robot plus rapide et plus malin". New id, so people who closed the old one still see it. One line at 414/390/375/360/320 px in all 3 languages (real CSS + Special Elite). History message unchanged (until 14 Oct). Checklist OK, diff = news lines + version.
**News rule (Oscar, 8 Oct): keep messages up only a few days (~4).**

**PRODUCTION v261008.351 — News wording changed (Oscar):** same id `2026-10-08-robot-fast`, same end date 12 Oct, now "⚡ ¡El Robot ahora corrige más rápido!" / "⚡ The Robot now checks faster!" / "⚡ Le Robot corrige plus vite !". One line at 414–360 px in all 3 languages; ES wraps to 2 lines at 320 px (accepted, like the history message). Checklist OK, diff = 3 news lines + version.

**PRODUCTION v261008.352 — robot news removed** (Oscar: test the new AI order first before announcing it). `NEWS` now only has `2026-09-30-historial` (until 14 Oct). The "faster Robot" wording, if wanted later: ES "⚡ ¡El Robot ahora corrige más rápido!" / EN "⚡ The Robot now checks faster!" / FR "⚡ Le Robot corrige plus vite !" (one line ≥360 px; ES 2 lines at 320). Checklist OK.

**Scout v5.1 second clean run (8 Oct):** Sante 30/34, contests 13/15, 0 flips, 3.4 s / 6.2 s, 0 timeouts (misses Tomate seco 🤔, Vivora ✅, Broccoli ✅, Motocross ✅; contests overturned Tomate seco + Michelin as Sportif). Ling Fin 29/34, 11/14, 2 flips (still FR "5" ✅ body part; contests overturned Jabali + Guava). Cohere 28/34, 13/15, 5.2 / 10.7 s, 2 timeouts (Maradona ❌, Tarragon/Robin ✅). Confirms the v349 order. Oscar considered pricier models (OpenRouter < $0.50/M in, 8 Oct: `anthropic/claude-haiku-5.5` $0.10/$0.50, `openai/gpt-6-luna` $0.10/$0.50 thinking on, `deepseek/deepseek-v4.1-flash` $0.036/$1.00, `qwen/qwen3.8-omni-flash` $0.15/$0.47) — parked: try the prompt fix first, look at these only if it doesn't work.

**PRODUCTION v261008.353 — sharper judge prompt** (Oscar left the wording to Claude). All three models made the same misses, so it's the prompt:
- Rule (4) adds: "Wrong, swapped, missing or extra letters are misspellings even when the word sounds the same (b/v, g/j, ll/y, c/s/z, a missing or extra h, a single letter doubled or a double letter single), e.g. ${spellEx}." `SPELLING_EXAMPLE` lives INSIDE `dailyBuildPrompt` (the test kit evals the function alone): ES "vivora"/"víbora", "broccoli"/"brócoli"; EN "brocoli"/"broccoli", "jiraffe"/"giraffe"; FR "carote"/"carotte", "farmacie"/"pharmacie".
- New rule (6) Category fit: the answer must be the kind of thing the category names; a category of people (athlete/sportsperson (Deportista, Sportif), writer, singer, scientist, historical figure) needs a person, not a sport, brand, place or object; a spice must be a spice or herb, not a dish or prepared/powdered food; only people's names may be written surname first ("Edison, Thomas"), anything else must be its normal name, not reordered to fit the letter.
- Multiplayer "allow spelling mistakes" swap now replaces the WHOLE new rule (4) (checked: lenient rule in, strict sentence out, rule 6 kept); Estricto/Relajado unchanged.
- Contest prompt (method B): "Letters that sound the same are still spelling mistakes (b/v, g/j, ll/y, c/s/z, a missing or extra h)" + "be generous with the category scope … — but it must still be the kind of thing the category names: a category of people … needs a person, not a sport, brand or object, and a spice must be a spice or herb, not a dish or prepared food."
- Prompt ~1.8k chars (was ~1.3k) — a few more input tokens, negligible cost.
- Scout v5.2: same prompt copied (`test_judge_prompt.mjs` 26/26: identical), default `ONLY = ['inclusionai/ling-3.0-flash-sante']` (~10 min). Checked: version grep (352 → 353), 12 screens, syntax, suites 239/239.
- **Next:** Oscar runs scout v5.2 (Sante only, hotspot). Expect Vivora/Broccoli/Motocross/Tomate seco fixed; watch that nothing else broke (e.g. Edison, Maradona, Ma sorcière bien-aimée must stay ✅). If it doesn't help → try the pricier models.

**Scout v5.2 result (Oscar, 9 Oct, Sante only, prompt v353):** **34/34** (ES 11/11, EN 10/10, FR 13/13), **¿Error? 15/15**, 0 unsure, 1 flip (question not identified), avg 3.3 s, slowest 10% 5.3 s, 0 timeouts. All four shared misses fixed; Edison / Maradona / Ma sorcière bien-aimée still ✅. Caveat: the rules were written with these test questions in mind — confirm on real dailies. Expected daily wait now ~7–10 s for 6 answers (was ~40–60 s on 8 Oct, ~1.5–2.5 min before v339).

**Session 27 summary (1–9 Oct):** production went v331 → v353. AI: one shared judge call, 2000 tokens, retry on empty/cut, timeouts retried on the next model, 3 answers at a time, `aiInfo` timings, every ¿Error? logged in the daily score, model order Sante paid → Ling Fin → Cohere free, sharper prompt (sound-alike misspellings, category fit). Daily: once per name (fresh check + transaction), no cross-language repeats from 10 Oct, ¿Error? ⏳ state. Multiplayer: rounds no longer stuck on a player who left/disappeared (leave marker, "Esperando a…", host "Seguir sin…" after 15 s), first ¡Alto! wins, floating numbers stop. Groups: no 📌, × with inline confirm, follow a verified registered name across devices, no duplicate names. Other: tagline removed, Spanish Google/share description, trilingual title "¡Alto! — Stop · Tutti Frutti · Petit Bac", join-screen logo goes home. Test kit: new `restore/stuck_scenarios.py`, groups/build5b checks updated (`tests_kit.zip` delivered with v344). Tools: AI scout v5.2.

**PRODUCTION v261009.354 — Robot news back** (Oscar, after the 34/34 scout): new id `2026-10-09-robot-fast` (so people who closed the 8 Oct one still see it), info, until **2026-10-13**: "⚡ ¡El Robot ahora corrige más rápido!" / "⚡ The Robot now checks faster!" / "⚡ Le Robot corrige plus vite !" (same wording as v351: one line ≥360 px, ES 2 lines at 320). Listed first, above the history news (until 14 Oct). Checklist OK, diff = news lines + version.

**9 Oct daily review (first full day on Sante + v353 prompt, 11 players, 60 answers):** per answer median 3.0 s (7.2 s on 8 Oct), slowest 10% 9.4 s, max 25 s, 5 over 10 s; Sante 59/60, Ling Fin 1 (after a timeout); 3 `retried` (empty reply) → most of the 15–20 s waits. Approx. wait per player 6–10 s for most, 18–30 s for the slowest few (was ~40–60 s). Spelling now strict as intended: Gaugin ❌, Gerard de Pardieu ❌, Encima (árbol) ❌, Espon ❌, Leetuan ❌. Lituania ✅ as an EN country (proper nouns any language) and "Lewis Carl" ✅ athlete after contest. **Problem: ¿Error? too generous on category** — overturned Edredón / Esterilla / Espirometría as Invento and "El pájaro espino" (a TV miniseries) as Película; "Library" as Type of shop overturned for David but upheld for Venirse primos (same model, borderline → coin flip).

**PRODUCTION v261009.355 — stricter ¿Error? (contest method B system prompt):** "be generous with the category scope — if the answer could reasonably belong … rule VALIDO" + "If in doubt about category fit only, rule VALIDO" replaced by: "give the answer a fair second look: the first judge may have been too strict or may not have known the answer (an uncommon but real person, film, place or thing). Overturn (VALIDO) only if the answer clearly belongs to the category as most people would understand it — do not stretch the category or invent a link (a library is not a shop; a quilt or a mat is not an invention). [kind-of-thing sentence kept] If in doubt, about spelling or about category fit, rule INVALIDO." The contest user prompt (the "Note:" the kit compares) is unchanged. Scout v5.3: same system prompt copied, 5 new contest questions (Library shop ❌, Edredón ❌, Esterilla ❌ as Invento; Gyrophare Invention ✅, Maradona Sportif ✅) → 20 contest questions, still `ONLY` = Sante (~11 min, hotspot). Checked: version grep (354 → 355), 12 screens, syntax, suites 239/239.

**Scout v5.3 result (Oscar, 9 Oct, Sante, prompt v355):** daily 34/34, 0 flips, avg 2.9 s, slowest 10% 4.7 s, 0 timeouts; contests 17/20 — Library and Esterilla now upheld ✅ (fixed), but Guava OVERTURNED (regression caused by v355's "may not have known the answer (an uncommon but real … thing)"), Edredón still overturned, Gyrophare upheld (too strict). Edredón/Gyrophare as "Invento" are judgement calls — not tuned further (avoid over-fitting the prompt to a handful of test words).

**PRODUCTION v261009.356 — Guava fix:** contest prompt "…(an uncommon but real person, film, place or thing) — but a common noun from another language is still INVALIDO, however real it is." Scout v5.4 copies it. Checked: version grep (355 → 356), 12 screens, syntax, suites 239/239. **Plan (Oscar agreed):** leave the re-check alone now and judge it on a week of real dailies (`contestLog`). If "Invento" keeps causing trouble, consider treating it more loosely in both directions.

**Scout v5.4 result (Oscar, 9 Oct, prompt v356):** daily 34/34, 0 flips, 3.2 s / 5.9 s; contests 17/20 — Guava STILL overturned despite the explicit sentence, Juan Carlos I now UPHELD (new miss; always rescued before), Gyrophare upheld, Edredón back to correct. Conclusion: each contest question is asked once and borderline answers wobble run to run — prompt tweaks v355/v356 were chasing noise (fix one, break another). Decision: stop tuning on the scout; judge the re-check on real `contestLog` data.

**PRODUCTION v261009.357 — re-check "slightly more generous" (Oscar's idea):** contest method B system prompt category part is now: "If spelling is correct, be slightly more generous than the first judge: rule VALIDO if the answer reasonably belongs to the category, even if it is uncommon or a borderline case. It must still be the kind of thing the category names: [people / spice sentence]. A common noun from another language stays INVALIDO. If in doubt about spelling, rule INVALIDO." Both "in doubt about category → VALIDO" (v354) and "→ INVALIDO" (v355–356) tie-breakers are gone. Strict spelling line (sound-alike letters) unchanged; daily judge unchanged. Scout v5.5 copies it. Checked: version grep (356 → 357), 12 screens, syntax, suites 239/239.
**Re-check prompt history (method B), for reference:** ≤v352 generous + "in doubt about category → VALIDO"; v353 + kind-of-thing sentence + sound-alike letters; v355 "overturn only if clearly fits, in doubt → INVALIDO" + library/quilt examples; v356 + foreign common nouns stay INVALIDO; v357 "slightly more generous than the first judge, reasonably belongs".

**Scout v5.5 result (Oscar, 10 Oct, prompt v357):** daily 34/34, 1 flip, avg 3.0 s, slowest 10% 4.7 s, 0 timeouts; contests **18/20** (best of all versions) — Juan Carlos I and Gyrophare rescued again; Library, Edredón, Esterilla, Michelin upheld. Misses: Guava overturned (consistent across v355–357 → real, not noise; "guava" seen as close to "guayaba"), Tomate seco en polvo overturned (expected cost of generosity). Decision: keep v357, stop tuning on the scout.

**10 Oct daily review (17 players, ~85 answers, v357):** all Sante, typically 2–4 s; one player ~12 s (three answers slow together), two retries 12–16 s; most dailies ~5–8 s. Verdicts almost all right: correct ❌ Masseur (FR word in ES), Echar / Enebre (ES words in FR), Marvel (studio, not a series), Escalier (not a building), Écrevisse (not a fish), Leggins / Lewis Caroll (misspellings); correct ✅ Éperlan, Murder She Wrote, Los Brincos, Mano izquierda, Madroño, Médula. ¿Error?: 12 re-checks, 1 overturned (Monumento as building, fair), 11 upheld, all reasonable — v357 is not a rubber stamp. **One clear mistake: EN "Luxemburg" ❌ as European country (German / old English spelling), upheld on contest, while "Lituania" ✅ passed the same day** — the strict spelling rule read it as a typo of Luxembourg. Proposed fix ("A proper noun written the way another language spells it (Luxemburg, Lituania, Londres) is not a misspelling") — **Oscar: no, it's OK as is.** Minor, left alone: Manomanista 🤔 (real Basque pelota player), EN "Lama" ✅ mammal (llama).

**STAGING v261010.358-tmp — compact multiplayer lobby (Oscar's request; text mockup agreed first).** Built on production v357.
- **Share card:** one row `[📋 Copiar enlace] [WhatsApp] CODE` — the code is small/grey (`.code-sm`, 13px): nobody can type a code anywhere (no join-by-code box), it's only for reference. Heading and URL box removed (`#lbl-share-code`, `#share-url` kept as hidden spans: `copyLink` reads `#share-url`).
- **"Eres el anfitrión" line hidden** (`#lob-role{display:none}`, element kept for debug); instead a 👑 on the host's chip (`renderPlayers(players, langs, hostName)`, callers pass `room.host`; names now go through `escHtml`).
- **Group picker moved into the players card** (`.lob-group.host-only`, same ids `#grp-current`, `#grp-panel`, `#btn-grp-hist`; `#lbl-group-name` hidden span).
- **New settings card:** mode as a big two-button switch (`.seg.big`, Caveat 20px, 16px below 360 px; buttons size to their text so FR "Démocratique" fits at 320) + one-line hint `#mode-hint` (`MODE_HINT` ES/EN/FR: "el anfitrión decide qué vale" / "todos votan qué vale"); under it `#btn-config-toggle` = a summary line `#cfg-summary` ("⚙️ 🎲 Clásico · 5 rondas · 90 s · 🔥 · 🤖 estricta", `updateCfgSummary()`), shows "⚙️ Configuración" when open; `#config-body` opens INSIDE the card. Then ¡Comenzar! and "Salir de la sala" as a text link (`button.leave-link`, host and guest).
- **One pill style** (`button.pill` / `.on`, 13px Special Elite) for every switch; old inline-style toggling in `setValidationMode` / `setAIStrictness` / `setAISpelling` replaced by `pillPair()` (+ `aria-pressed`). "Reacciones con puntos" and "Elegir 8 al azar" are now Sí/No pills (`#rp-yes/#rp-no`, `#fcr-yes/#fcr-no`, `setReactionPoints`, `setFreeCatRandom`, `syncFreeCatPills`, `syncLobbyPills`); their checkboxes stay as hidden inputs (startGame and the config read `.checked`). Sí/No texts via `.yes-txt/.no-txt` in applyLang.
- **"Usar ejemplos" turns "8 al azar" on** (examples are always ≥9).
- **Settings remembered per name on the device:** `alto_lobby_config:{safeName}` (+ `alto_lobby_config` = last used, the starting point for a name with none). Now also saves the custom category list (`freeCats`). `_cfgRestoring` stops the setters from saving half-restored values (enterLobby's early setter calls used to save the previous values under the new name).
- **Guests see the host's choices** under the player list: `#guest-info` "🗳️ Democrático · 🎲 Clásico · 5 rondas · 90 s" from `room.lobbyInfo {mode, themeIdx, rounds, time}` (themeIdx so it shows in the guest's language). Written by `pushLobbyInfo()` — ONLY from the host's lobby (re-checked when the 400 ms debounce fires), only when changed, cancelled by startGame. **Lesson:** a first version without those guards wrote during a round and broke build5b's "your buttons are never replaced" check (an extra room write re-renders guests) — 77/77 again after the guards, twice.
- Guest card compacted: emoji + name + "✅ ESTÁS DENTRO" on one row, group row below.
- Checked: version grep (357 → 358-tmp), 12 screens, init once, syntax, unit 239/239, browser restore 19, countdown 12, groups 28, build5b 77 (×2), stuck 11; a two-browser lobby test (fonts real) at 320/360/390/414 in ES/EN/FR: no horizontal overflow, mode buttons not cut, ¡Comenzar! visible without scrolling (bottom at ~550–590 px), guest summary live, Libre + examples → 8 al azar ON, per-name memory (Oscar's settings back after leaving; a new name "Pepa" starts from the last used; Oscar keeps his own after Pepa changes hers). File 412 KB (waived). The lobby test is `/tmp/claude-work/lobby_test.py` in the sandbox only — not yet in the kit.
- **Oscar to test on staging (index_tmp.html):** host and guest phones; open/close settings; switch mode (guest line updates); Libre + Usar ejemplos; leave and re-create the room with the same name → settings back.

**STAGING v261010.359-tmp — lobby tweaks after Oscar's first look ("not bad at all!"):**
- Share card: short label `#lbl-share-code` = T `shareCode` now "¡comparte!" / "share!" / "partage !" (was "comparte el código con tu familia"…), then the code back in BIG RED (`.room-code`, 56px Caveat, the original style; `padding-left:12px` balances the letter-spacing so it centres), then [📋 Copiar] [WhatsApp]. `.code-sm` no longer used.
- Settings bar: `updateCfgSummary()` → "⚙️ Configuración (Clásico, 5 rondas, 90 s)" / "⚙️ Settings (Classic, 5 rounds, 90 s)" / "⚙️ Paramètres (Classique, 5 manches, 90 s)" — gear only (emoji stripped from the theme name and `configLabel`), no reactions/robot in it; same text open or closed (`#btn-config-label` no longer shown).
- Reactions row: T `reactionPoints` 🌶️ → 👏 ("👏 Reacciones con puntos" / "👏 Reactions score points" / "👏 Réactions avec points"), the "🔥 +30 · 👏 +20 · 😂 +10 · 😬 0" line removed (→ help page). All four option labels now measured 20px Caveat, same row layout.
- ¡Comenzar! bottom now ~615–670 px (was ~550–590): visible without scrolling on 844-px-tall phones; on small phones (iPhone SE class) a short scroll.
- Checked: version grep (358 → 359-tmp), 12 screens, syntax, unit 239/239, groups 28, build5b 77, lobby test 390 ES / 320 FR / 414 EN (no overflow, mode buttons not cut, per-name memory OK).
- **Help page note:** the reaction points (🔥 +30 · 👏 +20 · 😂 +10 · 😬 0) must be explained there now that the lobby no longer shows them.

**Last versions: production v261009.357, staging v261010.359-tmp (compact lobby + tweaks).**

---

## ▶ START HERE NEXT SESSION (Session 28)

1. Upload the project zip (with `tests\` and `tools\`). Read this handoff (Session 27 log, especially the summary at its end). Production = **v261009.357**. Staging = **v261010.359-tmp** (compact lobby + tweaks, waiting for Oscar's test → promote as v360). First thing: `cp index.html index_backup_s28.html`.
2. **Check the new Robot on real play** (v357 re-check = "slightly more generous than the first judge" — check `contestLog`: real-but-uncommon answers rescued? foreign words and wrong-kind answers (library as shop, quilt as invention) kept ❌? Don't tune on single scout flips): ask Oscar for a recent daily export. Look at `aiInfo` (model should be Sante; `s` seconds — expect ~3 s, a whole daily ~7–10 s; any `timeout`/`retried`/`cut`/`error`), `aiResponses` and `contestLog` (each ¿Error?: word, original → result, model). Look for wrong verdicts the v353 rules may have caused (too strict on spelling? people categories? reordered titles like "Bel Air, el príncipe de" now ❌ by design).
3. Robot news is back in v354 until 13 Oct (wording below); if the Robot misbehaves, remove it: ES "⚡ ¡El Robot ahora corrige más rápido!" / EN "⚡ The Robot now checks faster!" / FR "⚡ Le Robot corrige plus vite !" — keep news only ~4 days (Oscar's rule). The group-history news expires 14 Oct.
4. Still to confirm on real phones (v344): daily in two tabs with the same name (second shows the first score); group sync phone ↔ computer with a verified registered name; a real 3+ player game where someone leaves / locks the phone (host sees "Esperando a…", "Seguir sin…" after 15 s).
5. Ask Oscar: Firebase backup exported? If all is well since v324 → delete `global/`, `groups/`, `groupNames/` + their three rules lines (window has arrived).
6. **Do NOT run the AI scout on Oscar's home network** (router restarted twice during scouting on 8 Oct) — hotspot only. Re-run only after a prompt change or when a model fails. Pricier candidates parked (Haiku 5.5, GPT-6 Luna, DeepSeek V4.1 Flash, Qwen 3.8 Omni Flash) if the prompt fix proves not enough.
7. Tests: run only when a change needs them. Full kit: `./run_all.sh ../index.html` → 239 unit + browser restore 19, countdown 12, groups 28, build5b 77, stuck 11 (run long ones one by one, 300 s sandbox limit).
8. Next candidates (plan with Oscar first): help pages (Session 26 draft + open decisions — now also: leaving/skipping players, groups follow a registered name), room-HTML security fix (Security item 2), security build + AI step 1 (also closes: group lists under `names/` are publicly readable), AI step 2. Small parked: home emoji grid cut at 320 px, `#qj-name` inline `oninput` (rule 8), tomato/botanical rule, unexplained host-session loss, per-language share previews (`/en/`, `/fr/`).

<details><summary>Previous START HERE (Session 27 → 28 draft) — kept for the record</summary>

### (draft written mid-Session 27)

1. Upload the project zip (with `tests\` and `tools\`). Read this handoff. Production = **v261008.353** (Session 27: shared AI judge call, 2000 tokens, retry on empty, `aiInfo`; news "Robots mejorados"; ¿Error? ⏳ state; tagline removed + new Google description). Staging = v261001.332-tmp. New staging builds start from production v353 (v353: sharper judge prompt — sound-alike misspellings, category fit; v350–352: robot news added then removed — only the history news is left; v349: AI order Sante paid → Ling Fin → Cohere free; v348: Sante free removed; v347: timeout retry skips Cohere; v346: every ¿Error? logged in the daily score `contestLog`; v345: daily seed per date+language from 10 Oct; v340–344: daily once per name, groups UI + sync by registered name + no duplicate names, stuck-round fixes, join-screen logo; v334–339: news, ¿Error? ⏳, tagline removed + Google description, share/page title, daily judged 3 at a time + timeout retry + timing in `aiInfo`). **First thing: check `aiInfo.s` / `timeout` / `error` in a daily export — still slow? still many timeouts? Then decide on tokens (1200?) or reasoning effort low, and the quality step (scout re-run with Vivora, Tarragon, Robin, Broccoli).** **News messages expire: history 14 Oct, robots 16 Oct** — ask Oscar if he wants new ones.
2. Check a recent daily export: are the long reasoning replies gone? What does `aiInfo` show — which models judge, how often `retried` / `cut`, any Cohere empty replies without `cut`? Daily speed OK?
3. Ask Oscar: Firebase backup exported? Any problems in real games since v324? If all is well → he deletes `global/`, `groups/`, `groupNames/` + their three rules lines.
4. Tests: run only when a change needs them. Full kit: `./run_all.sh ../index.html` → 239 unit + 77 + 12 + 19 + 28 browser (run long browser ones one by one, 300 s sandbox limit).
5. Next candidates (plan with Oscar first): help pages (Session 26 draft + open decisions), room-HTML fix (Security item 2), security build + AI step 1, AI step 2. Small parked: home emoji grid cut at 320 px, `#qj-name` inline `oninput` (rule 8), hard-coded Spanish in `doSubmit`, English words slipping through in FR/ES (Robin, Broccoli), tomato/botanical rule, unexplained host-session loss.

</details>

<details><summary>Previous START HERE (Session 27) — kept for the record</summary>

## ▶ START HERE NEXT SESSION (Session 27)

1. Upload the project zip (with `tests\` and `tools\`). Read this handoff. Production = **v260929.331** (Build 5 live + AI order Cohere → Sante → Ling Fin + daily "play to see" fix + home News). Staging = v260929.330-tmp (same code). New staging builds start from production v331. **News messages expire by themselves (robot 3 Oct, history 14 Oct)** — ask Oscar if he wants new ones.
2. Ask Oscar: Firebase backup exported? Any problems in real games on v324? If all is well 1–2 weeks after 29 Sep → he deletes `global/`, `groups/`, `groupNames/` + their three rules lines.
3. AI scout v4 done (results in the Session 26 log); `AI_MODELS` reordered in v325. Re-run the scout only after a prompt change or when a model starts failing.
4. Tests: run only when a change needs them (Oscar's request). Full kit: `./run_all.sh ../index.html` → 239 unit + 77 + 12 + 19 + 28 browser. Browser scenarios take ~5 min in total: run long ones one by one (the sandbox stops single commands after 300 s; background jobs may be killed).
5. Next candidates (plan with Oscar first): **help pages** (Spanish draft + open decisions in the Session 26 log), the small room-HTML fix (Security item 2), then the security build + AI step 1, AI step 2, the help-page pass (now also: group page, history, share link, Revisar for guests, 🙌 awards, countdown). Parked: tomato/botanical rule (Flagged for Future), unexplained host-session loss (Session 26 v323 note), home emoji grid too wide at 320 px.

</details>

<details><summary>Previous START HERE (Session 26) — kept for the record</summary>

## ▶ START HERE NEXT SESSION (Session 26)

1. Upload the project zip (must include `tests\` and `tools\`). Read this handoff, then `MULTIPLAYER_SCORING_DESIGN.md` (Groups, History, Languages sections; Build 5 row).
2. Production = **v260926.317** (scout models + one shared judge prompt). Staging = **v260926.318-tmp** (Build 5a + AI fix) — **still to be tested by Oscar** (test list in the Session 25 log). Run the kit on staging first: `./run_all.sh ../index_tmp.html` → ALL GREEN (178 + 28 + 19). Note: the updated suites need 5a code, so they fail on production v313 — that is expected.
3. ~~Finish the AI model choice~~ — done in v260926.315 (Sante free → Cohere free → Ling Fin paid). Multiplayer now uses the same prompt (v317), so the scout covers it; ask Oscar how the multiplayer 🤖 check went anyway.
4. Then **5b**: group page (Leaderboard screen) with history ◀ ▶ (SVG flag + date), full game view (ranking, awards, per-round breakdown from `roundLog`), Mundial/global leaderboard removed, `?g=ID` share link + share text link, Historial buttons (host picker and guest line). Plan with Oscar first.
5. When Build 5 goes to production: Oscar deletes `global/`, `groups/`, `groupNames/` in the console and removes their three rules lines (the `lastPlayed` index is already in).
6. After Build 5: the small room-HTML fix (Security item 2, no hurry), then the "security build" together with AI model management step 1 (anonymous auth + rules, App Check, name claims, cookie consent banner), then AI step 2, then the help-page pass. See Flagged for Future → Security.

**Open notes carried forward**
- After Build 5: one help-page pass (ES/EN/FR) for practice games, ties, reactions, awards, groups and history (recorded in the design doc). Build 1 only updated the ¡Alto! rules text.
- No join-by-code box on the home screen (only invite links); `joinRoom` still references a missing `#inp-code`. Decide if a code box is wanted.
- `doSubmit` hard-codes Spanish `stopCaller:'Tiempo'` and "Respuestas enviadas. Esperando a los demás…"; `reviewSub` T key is dead.
- The Claude Doc copy of the design (link in the design doc) has drifted from the `.md` file; the `.md` is the source of truth.
- Handy: a `__debug__` name shows the debug bar; its Scores and Final screens have a 2-round sample game with a tie, reactions, an A caller, a duplicate, an invalid and an empty answer. Debug room now has `groupId: 'dbggarcia01'`.
- ~~Oscar to set a credit limit on the OpenRouter key~~ — done 29 Sep. Oscar: export a Firebase backup now and then.

**Last versions deployed: v260926.317 (production), v260926.318-tmp (staging).**
- JS syntax clean (`node --check`) ✅
- Test kit green ✅
- (File size: over 300KB since Session 23 — Oscar confirmed not a concern; v317 is 360KB, v318-tmp is 371KB)

**Always start from the uploaded working file** — never from a local copy that may have drifted.

---

</details>

<details><summary>Previous START HERE (Session 25) — kept for the record</summary>

Session 25 plan: Build 5 — Groups and history

1. Upload the project zip (must include `tests\`). Read this handoff, then `MULTIPLAYER_SCORING_DESIGN.md` (Groups, History, Languages sections and the Build plan row for Build 5).
2. Run the test kit on production `index.html` first — it must be ALL GREEN before starting.
3. Plan Build 5 with Oscar before writing any code (rule above). Suggested split into two staging steps: **5a** group IDs, Mis grupos lobby picker, default group, rename, group written to the room live, `endGame` writes to `groupsMeta/` + `groupsLb/` + `history/` + `historyIndex/`, 20-game trim, yearly cleanup, clean start (clear old `alto_groups`); **5b** group page (Leaderboard screen) with history ◀ ▶ (flag + date), global leaderboard removed, `?g=ID` share link and share text link, welcome-back group fix, "partidas" T key fix.
4. Update `suites/test_build1_scoring.mjs` endGame assertions (they check the old `global/`/`groups/` writes) in the same build, and add suites for groups/history.
5. Oscar's manual Firebase steps when Build 5 goes to production: add `".indexOn": "lastPlayed"` on `groupsMeta` in the rules; delete `groups/`, `groupNames/`, `global/` in the console.

**Open notes carried forward**
- After Build 5: one help-page pass (ES/EN/FR) for practice games, ties, reactions, awards, groups and history (recorded in the design doc). Build 1 only updated the ¡Alto! rules text.
- No join-by-code box on the home screen (only invite links); `joinRoom` still references a missing `#inp-code`. Decide if a code box is wanted.
- `doSubmit` hard-codes Spanish `stopCaller:'Tiempo'` and "Respuestas enviadas. Esperando a los demás…"; `reviewSub` T key is dead.
- The Claude Doc copy of the design (link in the design doc) has drifted from the `.md` file; the `.md` is the source of truth.
- Handy: a `__debug__` name shows the debug bar; its Scores and Final screens now have a 2-round sample game with a tie, reactions, an A caller, a duplicate, an invalid and an empty answer.

**Last version deployed: v260926.311 (production). Staging (index_tmp.html) = v260926.310-tmp, identical code.**

</details>

### v261010.360-tmp (staging, 10 Oct) — supersedes the "Staging = v359-tmp → promote as v360" line above
- **Group label:** `#lbl-group-name` is now a visible `p.lbl` ("grupo / group / groupe") above the group picker.
- **Validation mode moved inside Configuración**, first item above "tema": `p.lbl#lbl-validation-mode` (lowercase, no emoji, like "tema"), `.seg.mode-seg` with the two normal 13px pills (natural width, padding 6px 9px, so "🗳️ Démocratique" fits on one line at 320px FR), and `#mode-hint` is now a `p.lbl` (same style as the theme sublabel "8 categorías fijas…"). The `.seg.big` and `.mode-hint` CSS are gone.
- **Settings bar:** it is now the first thing in the host card (no top border/margin). Closed = `⚙️ Configuración (Normal, Clásico, 5 rondas, 90 s)` (mode added); open = just `⚙️ Configuración ▴`. `updateCfgSummary()` reads the open state; `toggleLobbyConfig()` calls it. It updates on every change through `saveLobbyConfig()`. It reads `G_validationMode` inside try/catch because that `let` is declared further down the script.
- The open/closed state of the settings is kept between rooms in the same page (not reset by enterLobby). That's fine; the bar text follows it.
- **Tests:** unit 239 ✓; countdown 12, groups 28, restore 19, stuck 11 ✓; two-browser lobby test (`lobby360.py`) at 390 ES / 320 FR / 414 EN: no overflow, no page errors, per-name memory ✓.
- **build5b is flaky** on the check "Marta's buttons are the same elements after Pepe reacted and voted elsewhere": it failed 0–3 times per run on **both v359 and v360** in this container, so it is not caused by this change. Investigate next session: something re-renders Marta's row during that step (timing), maybe the lobbyInfo write or a heartbeat.
- **Promote:** after Oscar's test → production as **v361**.

### v261010.361-tmp (staging, 10 Oct)
- **"90s" without a space** in the settings bar and in the guests' waiting line (`renderGuestInfo`).
- **Validation mode buttons look like the theme buttons:** `.seg.mode-seg` is a 2-column grid; `.mode-seg button.pill` copies `.theme-btn` (Caveat 20px, radius, paper-dark, border) and `.on` copies `.theme-btn.active` (red, bold). Markup is `<span class="mode-ico">👑</span><span class="mode-txt">Normal</span>`; applyLang now fills only `.mode-txt` (emoji stripped from T modeHost/modeDemocratic), so `s('mode-host', …)` is gone. Under 360px the mode cards drop to 16px (existing slot) so bold "Démocratique" fits; the emoji then wraps above, like the theme buttons do.
- Tests: all suites green (build5b 77/77 this run; still worth watching the flaky check noted under v360).
- **Promote:** after Oscar's test → production as **v362**.

### v261010.362-tmp (staging, 10 Oct)
- MODE_HINT.democratic now says the minimum: "todos votan qué vale (mín. 3 jugadores)" / "everyone votes on what counts (min. 3 players)" / "tout le monde vote (min. 3 joueurs)". The start check (`toastDemoMin`) is unchanged. Text-only change: checklist run, no test suites (Oscar's call).
- **Promote:** after Oscar's test → production as **v363**.

### v261010.363: PRODUCTION (10 Oct). Supersedes the "promote as v360/v361/v362/v363" lines above
- Production = staging v362-tmp (compact lobby: "¡comparte!" + red code, group label, settings bar with live "(mode, theme, rounds, 90s)", validation mode inside settings as theme-style cards, hint with "mín. 3 jugadores" for Democrático, 👏 reactions row, per-name settings memory, guests see the host's choices) **plus a news line**:
  `2026-10-10-lobby` until 2026-10-14: "🎛️ ¡Sala renovada! Más compacta y recuerda tus ajustes" / "🎛️ New room screen: more compact, remembers your settings" / "🎛️ Nouvelle salle : plus compacte, elle garde tes réglages".
- Checklist: version grep before/after ✓, init block ✓, no -tmp outside the version-regex lookaheads ✓, 12 screens ✓, JS syntax ✓, 412 KB (size limit waived).
- Still open: the flaky build5b check (see v360 entry); the help page should explain reaction points and validation modes.

## START HERE (Session 29). Supersedes the Session 28 START HERE above
1. Upload the project zip (with `tests\` and `tools\`) and read this handoff. **Production = v261010.363**; staging = v262 content (identical apart from the news line). First thing: `cp index.html index_backup_s29.html`.
2. Ask Oscar how the new lobby went in real games: is the per-name memory surprising? Do hosts with 2 players trip on a remembered Democrático (the toast still blocks the start)?
3. Open items:
   - the flaky build5b check "Marta's buttons are the same elements…" (fails on v359 too, so not caused by the lobby change);
   - add reaction points and the two validation modes to the help page;
   - the news `2026-10-10-lobby` expires on 14 Oct;
   - the empty "Lobby compacto — mockup" design canvas can be deleted.
4. Rules unchanged: confirm the plan before building, deploy right after building, check the version with grep before and after, ASCII commit messages, the 6-slot fonts (16px is an existing slot, used for the mode cards under 360px).

### v261014.364: PRODUCTION (14 Oct), straight to production (Oscar's call; unit suites only)
- **Robot outage 13–14 Oct:** Sante answered **"No endpoints found for inclusionai/ling-3.0-flash-sante."** (404, tested from the console through the Worker) from ~15:00 UTC on 13 Oct. Its public endpoints page still listed Novita at 100% uptime, but no traffic. Ling Fin has **no endpoints** at all. So every answer fell to Cohere free: 25–40 s, timeouts → DUDOSO, and a wrong rejection of "Neerlandés" (Idioma). OpenRouter credit was fine. The Generations and Upstream logs showed only Cohere, because a model with no endpoints is skipped before any upstream call.
- **AI_MODELS = ['inclusionai/ling-3.0-flash', 'cohere/north-mini-code:free']**. The general Ling 3.0 Flash (Sante's base) has 2 providers (Novita, DeepInfra) and costs $0.021/$0.063 per M. A console test answered through Novita. **Not yet measured with the scout.** tools/ai_scout.js v5.6 has ONLY = Ling 3.0 Flash; run it on the hotspot and compare with Sante's 30/34 + 10/10. Note: DAILY_CONSENSUS (false) uses MODELS[2], which no longer exists; fix that before ever switching it on.
- **Quick check for a dead model:** paste in the console on the site: `fetch('https://api.oscar-g-diez.workers.dev/',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({model:'<id>',messages:[{role:'user',content:'Di solo: VALIDO'}],max_tokens:50,temperature:0})}).then(r=>r.text()).then(console.log)`
- **"Animal prehistórico" → "Animal extinto"** (EN "Extinct animal", FR "Animal disparu"). Same position in the pool, so daily seeds are unchanged. Letters: ES ABDEIMPRTUV (+E Estegosaurio, +U Uro), EN ABDIMPSTVW (−R, +W Woolly mammoth), FR unchanged. Judge rule (6) gains: "An extinct animal must no longer exist (dinosaur, mammoth, dodo) — a living species is INVALIDO". The contest prompt B gains a matching clause. The scout copies were updated too (test_judge_prompt 26/26).
- **One-letter answers:** `aiTooShort(word)` (fewer than 2 letters/digits) → `AI_LOCAL_SHORT` (invalid, model 'local', 0 s), with no robot call. Used in dailyAICallModel (daily/practice), the contest, and the multiplayer 🤖 button. "Ñu" and "T-Rex" still go to the robot.
- **Daily review 13–14 Oct (other notes):** inconsistent regional capitals (Sevilla accepted by Sante, Nantes rejected by Cohere). Good Sante calls: "Blanco y negro" (drink), "Barrenero", "Normand" (Mabel Normand).
- **Scout v5.6 result (14 Oct, Oscar on hotspot), Ling 3.0 Flash:** 32/34 (ES 11/11, EN 10/10, FR 11/13), contests 17/20, 1 flip, avg 2.3 s, slowest 10% 4 s, 0 timeouts. This is better than Sante's 30/34 on judging; on contests Sante scored 18/20 with the v357 prompt. Misses: FR "Tarragon" and FR "Robin" accepted (English words in a French game; Sante also accepted Robin). Wrong overturns on re-check: ES "Guava", EN "Library", ES "Edredón". A firebase WebSocket ERR_NAME_NOT_RESOLVED in the same console was a hotspot DNS blip, not the game. **Keep Ling 3.0 Flash.**

### v261014.365: PRODUCTION (14 Oct)
- The practice countdown is now **5 s** (`startDailyCountdown`: `G_daily.practiceMode ? 5 : 10`). The daily keeps 10 and multiplayer is unchanged. The overlay is opaque (full-screen paper), so categories are never visible during the countdown. Checklist only.

### v261015.366-tmp (staging, 15 Oct): robot instruction changes, waiting for the scout
- **Dailies 14–15 Oct with Ling 3.0 Flash:** all answers went to Ling, 1–4 s (max 7.8 s when 3 answers ran in parallel), 0 timeouts, 0 Cohere fallbacks. Errors: "Van gogh" (Pintor famoso) rejected on first judgement AND on re-check, while "Van Gogh" was accepted for others; re-check overturned "Vivora" (Reptil, the prompt's own misspelling example) and "Helia Kazan" (Elia Kazan with an H added); "Vallas, salto de" (Deporte) was accepted despite the no-reordering rule. The re-check is too generous (also seen in scout: Guava, Library, Edredón).
- **Daily rule (1):** "Ignore accents, tildes and capital letters — Ramon=Ramón=ramón is fine."
- **Re-check (method B):** "check it again with fresh eyes. Ignore capital letters."; the spelling rule now has the per-language example (`spellEx`, same words as daily rule 4, defined inside the contest function); NEW "Never accept a name or word changed to fit the letter: an added, dropped or swapped first letter makes it INVALIDO. Only people's names may be written surname first; any other answer reordered to fit the letter is INVALIDO."; "be slightly more generous than the first judge … reasonably belongs … borderline case" was REPLACED by "rule VALIDO if the answer genuinely belongs to the category, including uncommon but real answers". 
- **Scout v5.7:** same prompts (test_judge_prompt 26/26), 'Animal prehistórico' → 'Animal extinto', +2 judge cases (Van gogh ✓, Vallas, salto de ✗), +3 re-check cases (Vivora ✗, Helia Kazan ✗, Van gogh ✓). Now 36 + 23. Baseline to beat (v5.6 on v364): 32/34, re-checks 17/20.
- Unit suites green; browser suites not run (prompt-only change).

### v261015.367: PRODUCTION (15 Oct) = staging v366-tmp
- **Scout v5.7 on v366 (Oscar, hotspot), Ling 3.0 Flash:** 34/36 (ES 12/13, EN 10/10, FR 12/13), re-checks 20/23, 3 flips, avg 2.9 s, slowest 10% 4.4 s, 0 timeouts.
- **Fixed vs v364:** Van gogh is accepted on both the judge and the re-check; re-check now upholds Vivora, Helia Kazan and Guava; FR Tarragon is rejected.
- **Still wrong:** FR "Robin" accepted; "Vallas, salto de" accepted (the reorder rule is ignored); the re-check overturns Library (Type of shop), Edredón and Esterilla (Invento), so it stretches categories. Flips went from 1 to 3: watch the dailies.
- **Possible next step:** a re-check line along the lines of "the answer must be what the category names, not something merely related to it (a library is not a shop; a quilt is not an invention)". Only if real dailies keep showing it.

## 👥 Follow System (friends): design RESTORED 15 Oct
*Designed with Oscar on 12 Sep (Session 14, chat "++ ALTO 12"). It was written into the handoff then as "### Follow System (designed Session 14, not yet built)", but that section later went missing; only the one-liner "Favourites/friends list" under Other survived. Restored here from the chat. Not built.*

**Decisions (Oscar, 12 Sep):**
- **One-way following** (Instagram/Twitter model): you follow someone to see their scores; no confirmation, requests or notifications. Rejected in this order: mutual friend requests (too complex: pending state, accept/decline UI), permanent groups joined by a link, and a "bulk mutual friend request" link (everyone who taps it becomes friends). Oscar: "a follow on leaderboards + search box is simpler".
- **★ on each daily leaderboard row**, tap to follow or unfollow. Three states: grey/empty = not following; yellow/filled = you follow them; yellow + small "follows you"/👥 badge = mutual.
- **Search box on the leaderboard** to filter rows by name as you type (needed as player numbers grow; useful even without follows). A separate "find a player" screen, which works even if the friend hasn't played today, was discussed as the cleaner add flow; it was not decided.
- **"Following" toggle** on the leaderboard shows only people you follow. You always see yourself.
- **Registered names only:** unregistered names show no ★ and cannot be followed. The help panel gets a "¿Por qué registrar tu nombre?" section listing the benefits (leaderboard, cross-device history, follow and be followed, mutual badge, verified badge). Oscar may enforce registration later.
- **Cross-device:** stored in Firebase under your name. The 12 Sep design said `following/{safeName}: [safeName1, …]`.
- **View layer only:** it never changes official scores. Earlier notes (Sessions 13–14) also flag using it to filter history and daily originality ("how original was I among my friends").
- Still open from 12 Sep: should a followed player who hasn't played today show as "not played yet" or be hidden?

**Since then (notes for whoever builds it):**
- **Groups now exist** and follow a verified registered name (`names/{safe}/groups`, see the groups sync entry). Following could live next to it (`names/{safe}/following/{otherSafe} = time`), reusing the same pull/merge pattern. A "players from my groups" filter could cover part of the need; ask Oscar whether that's enough as a first step.
- **"Follows you" badge** needs a reverse index (`names/{safe}/followers/{otherSafe}`) written together with `following`, so a leaderboard doesn't have to scan everyone's lists. Two writes per follow; unfollow removes both.
- **Security:** `names/` is currently open read/write. A follow graph there is readable and editable by anyone until the security build closes it, so build it after (or together with) that.
- **Registration status:** check whether the "unverified name label" (`Peter? 🎸`, designed Session 13) was ever built. The ★ rule depends on knowing which leaderboard rows are registered.
- **Decided 15 Oct (Oscar):** with "Siguiendo" on, followed players who haven't played that day are SHOWN at the bottom, greyed out, as "aún no ha jugado" ("no jugó" when browsing past days).
- **Layout decided 15 Oct (Oscar)**, replacing the 12 Sep three-state "★ + badge" idea:
  - One star per row, between the rank and the name (its own tap target, so tapping it doesn't open the answers panel). States: **grey outline ☆** = not following; **silver filled ★** = you follow them; **gold filled ★** = you follow each other. No separate "follows you" badge. Outline vs filled is what separates "not following" from the other two, so it doesn't rely only on grey vs silver.
  - Someone who follows you but whom you don't follow keeps a grey ☆; it turns gold only when you follow them back.
  - No star on your own row or on unregistered names.
  - Above the list: pills `[ Todos ] [ ★ Siguiendo ]` (same pill style as the lobby) and 🔍, which opens a type-to-filter search box in place of the pills.
  - With Siguiendo on: show ALL followed players, whatever their rank (today the list is top 10 + me), keeping their overall rank numbers (🥇, 🥉, 5…), your own row included. Followed players who haven't played go at the bottom, greyed out: "aún no ha jugado" / "no jugó" on past days.
  - Mockup:
    ```
     [ Todos ]  [ ★ Siguiendo ]          🔍
     🥇 ★ Marina 🎸          580 pts (⚡×1.3) ▶   gold (mutual)
     🥈 ☆ Pepe 🦊            540 pts          ▶   grey outline
     🥉 ★ Abuela 🌻          510 pts (+50✨)  ▶   silver
      5   Oscar 🌈           460 pts          ▶   you
        ★ Tío Juan 🎺   aún no ha jugado            (Siguiendo only, grey)
    ```
  - Use the text glyphs ☆ / ★ (U+2606/U+2605) coloured with CSS, not emoji. Explain the colours in the help page.
- **Legend decided 15 Oct (Oscar):** one 11px line under the pills: "☆ seguir · ★ sigues · ★ os seguís" (stars coloured grey/silver/gold); EN "☆ follow · ★ following · ★ mutual", FR "☆ suivre · ★ suivi · ★ mutuel". Shown only until the player has followed someone (then the help page explains it). Unregistered players see instead: "☆ Registra tu nombre para seguir a otros jugadores", which opens name registration.

### v261015.368-tmp (staging, 15 Oct): Follow system built
- **Data:** `names/{me}/following/{them} = {n: name as shown incl. emoji, t}` and `names/{them}/followers/{me} = t`, both written in one `update()`; unfollow nulls both. The tap is optimistic and reverts with a toast if the save fails. Daily scores now carry **`reg: true`** when played under a name registered AND verified on that device (`claimLsKey(safe).verified`). The ☆ shows only on rows with `reg` (or on someone you already follow), so stars appear on scores from this version on.
- **Who can follow:** `G_follow.me = myGroupsSyncName()` (verified name on this device, same rule as groups). `followLoad()` runs in parallel with the scores read in `loadDailyLeaderboard`, so a new follower's gold ★ appears on the next load. `followEnsure()` in `renderLeaderboard` reloads when the name changes; the claim/verify success also redraws.
- **UI** (`#lb-follow-bar` between the date nav and the list): pills `#lb-mode-all` / `#lb-mode-following` (`pillPair`, mode saved in `alto_lb_mode`, hidden for unregistered), `#lb-search-btn` 🔍/✕ toggling `input#lb-search` (type=search, 16px so iOS doesn't zoom; listener via addEventListener). `#lb-follow-legend`: the coloured legend until you follow someone; for unregistered players with a name, the "Registra tu nombre…" line, which opens `claimOpen()`.
- **Rows:** `renderLeaderboard` now builds rows through `rowHtml(e, idx, id)`. Row ids stay the overall index (answer panels and `toggleDailyAnswers` unchanged; 'me' for my row outside the top 10). Search: every matching entry, any rank. Siguiendo: me + followed, overall ranks, then followed-but-not-played rows (`.lb-notplayed`, name from `following.n`, escaped) with "aún no ha jugado" (`np-long`) or "sin jugar" (`np-short`, under 360px), or "no jugó" on past days. Star: `button.lb-star.s-none|s-follow|s-mutual` (☆/★ text glyphs, 20px, colours var(--ink-muted) / #9aa3ad / #d4a017), `event.stopPropagation()` so it never opens the answers. Rows without a star get `.lb-star-sp` so names line up. The star column costs about 12px of name width.
- **Known/accepted:** at 320px, your own row with the "(+50✨)" tag already squeezed the name before this change ("Os…"); it's now "O…". Search only covers the day being viewed (finding any registered name needs the names list, which is for after the security build). Followed players are per leaderboard language: a friend who played only EN shows as "aún no ha jugado" on the ES tab.
- **Tests:** NEW `tests/restore/follow_scenarios.py` (34 checks: unregistered view, stars, both Firebase lists, gold/silver/grey, legend, Siguiendo with ranks + not-played, mode after reload, unfollow, past day, search, answer panels, second device, the other side sees gold, 320 FR / 320 ES / 390 ES / 414 EN without overflow). All suites green; build5b's flaky "Marta's buttons…" failed in 2 runs then passed 3/3 on both v367 and v368. The groups unit test evals this code outside a browser, hence the `typeof document` guard on the search listener.
- **Still to do:** help page text for the stars (and why register); the security build must cover `following`/`followers`.

### v261015.369-tmp (staging, 15 Oct): follow fixes after Oscar's first look
- **No stars on Oscar's board:** all of that day's scores came from production (before `reg`). Fix: `followCheckReg()` looks up `names/{safe}/displayName` (exists only for registered names; the secret is not read) for on-screen rows without `reg`, cached in `_regCache` for the visit, then redraws. At most ~a dozen small reads per board. Older days get stars too.
- **Search always visible** (Oscar): the 🔍/✕ toggle is gone. `input#lb-search` (placeholder "🔍 buscar" / "🔍 search" / "🔍 chercher") sits beside the pills, `flex: 1 1 90px`: one line at 390/414px, its own line under the pills at 320px. Look: Caveat 20px with an underline, like the game's other text boxes and the row names (the old 16px Special Elite matched nothing). Filters the current view: Todos = everyone that day, any rank; Siguiendo = followed players incl. the not-played rows. `G_follow.searchOpen` removed.
- Tests: follow_scenarios 38/38 (+ Ana: registered, score without reg → ☆ after lookup; search box font Caveat 20px; pills + box both visible; search in Siguiendo). All suites green (build5b 77/77 this run).

### v261015.370-tmp (staging, 15 Oct)
- **Legend always shown** for registered players (Oscar: "for now"), replacing "until you follow someone"; the colours matter most once you follow people. Unregistered players still get the register line.
- Search placeholder is plain text without the emoji: "buscar" / "search" / "chercher".
- follow_scenarios 38/38 (legend check flipped), unit suites green, checklist clean.

### v261015.371-tmp (staging, 15 Oct): daily ties share a medal
- **Daily leaderboard ranks are now dense, like multiplayer's `denseMedals`** (Oscar chose this over sports-style 1,1,3 as "fairer"): 600, 600, 580, 580, 550, 500 → 🥇 🥇 🥈 🥈 🥉 4. `renderLeaderboard` builds `ranks[]` from the rounded grand total ((totalScore + originality) × 10); `rankLabel(i)` = medal for ranks 1–3, else the number. Applies in Todos, Siguiendo and search.
- **Order within a tie:** earliest `timestamp` first (the sort used to have no tiebreak).
- `G_daily.myRank` is now the dense rank, so the share text medal/position agrees with the board.
- **Known trade-off (accepted):** with many daily players a big tie at the top means many golds, and the next score gets 🥈 even with many people above it. The "top 10" cut is still the first 10 rows, so it can split a tie.
- follow_scenarios 40/40 (+F14 ties: ranks and order), unit suites green, checklist clean.

### v261031.372: PRODUCTION (31 Oct) = staging v371-tmp + reveal fix + news
- Brings to production: the follow system (v368–370), always-visible search, the legend always shown, and daily ties sharing medals (v371).
- **Bug (Oscar):** played ES, open the EN tab (not played), ◀ to yesterday, open a player, ▶ back to today → today's EN answers were visible. Cause: `lbHistNav` reloaded the board without the "not played" flag (only `switchDailyLbLang` passed it). Fix in `loadDailyLeaderboard` itself: if `key` is today and the tab's language isn't played (`_playedLangs[lbLang]` or `alto_daily_{key}_{lbLang}`), `hideAnswers = true` whatever the caller passes, which covers ▶, flags, refresh and future callers. Debug `__` names don't write that localStorage key, so their own board hides answers after playing; acceptable.
- **News:** `2026-10-31-follow` until 2026-11-04: "⭐ ¡Sigue a tu gente en el reto diario y mira cómo les va!" / "⭐ Follow your people on the daily challenge and see how they did!" / "⭐ Suis tes proches dans le défi du jour et vois leurs scores !" (Oscar: "sigue a tu gente" rather than "familia").
- Tests: follow_scenarios 42/42 (+F15 reproduces the bug: fails on v371, passes on v372). ALL GREEN, checklist clean, 429 KB.
- **Next:** help page (stars + why register, ties); security build must cover `following` / `followers`.
- **News text changed before deploy (Oscar):** option B, "Toca ☆ en la clasificación para seguir a tu gente" / "Tap ☆ on the leaderboard to follow your people" / "Touche ☆ dans le classement pour suivre tes proches". This replaces the "⭐ ¡Sigue a tu gente…" line above. It uses the same ☆ glyph as the board, so it reads as the button to tap. Same id and dates.

### v261031.373-tmp (staging, 31 Oct): help rewritten, lean
- **Oscar:** a fresh look ("don't take the Sep draft literally, keep it lean"). Agreed text, one ❓ text per context, all in `HELP = {home, daily, multi}[lang]` as `[title, body]` pairs, drawn by `helpSectionsHTML(which)` with the existing `.help-section/.help-title/.help-text` classes (Caveat 20 titles, Special Elite 13 text). `shellHelpOpen` routes home/welcome → home, daily-play/daily-result → daily, everything else (lobby, round, review, waiting, scores, group page, final) → multi.
  - **home:** ¡Alto! one-liner · 🎲 Clásico en grupo · 📅 Reto diario + Práctica · 🔒 Reserva tu nombre (opcional): daily on any device, groups go with you, follow with ☆. No multiplayer points here (Oscar: "enough to see them inside a room").
  - **daily:** what it is (same for everyone, each language its own, one go per name, ¡Alto! early = bonus) · Puntos ✅100 🤔50 ❌0 ⚡×1–×2 ✨+50 · 🤖 ¿Error? · 🏆 ties share medals, ◀ ▶, today's answers after playing, ☆/★/★ legend + Siguiendo · ✏️ Práctica.
  - **multi:** how to play (round ends for everyone, time out sends what's there) · Puntos incl. **¡Alto!: "unos 30 por categoría (+250 con 8), menos 100 por cada respuesta tuya anulada"** (= `round10(250·n/8) − 100·bad`) · 🔍 Normal / Democrático (min 3) / Wikipedia / Robot only gives an opinion · 😂 reactions + points + awards · 🎮 group, 📜, ties share, **"Jugar solo es práctica: no se guarda"**, games ended before the last round don't count.
  - Left out on purpose: Robot Relajada/Estricta/spelling options, category themes, "Seguir sin…", per-name settings memory.
- **Removed dead code:** `rulesHTML`, `helpHTML`, `homeRulesHTML`, `dailyRulesHTML`, `toggleRules`, `updateRulesBtn` and their callers (the lobby "¿Cómo se juega?" box went with the lobby redesign). The T key `rulesBtn` is left unused.
- **Name reservation texts made honest (option B):** `descFree` now lists the real benefits (daily on all devices, groups, following) instead of "nadie más podrá usarlo"; `privacy` now says only an encrypted version of the email/PIN is kept, never the email, never contacted (the old "no lo guardamos en ningún servidor propio… si usas un PIN, solo tú lo sabes" overpromised: the SHA-256 is in Firebase, and a 4-digit PIN hash is trivially reversible while `names/` is open); `successClaimed` is just "¡Nombre reservado!". ES/EN/FR.
- ⚠️ **MUST DO NEXT (Oscar: "we really need to do A"): the reserved-name gate.** Today a reserved name is NOT protected: `_claimState === 'taken'` only changes the pill text ("¿Eres tú? Verifica →"); anyone can still create a room, join, or play the daily under it, which can even use up the owner's one daily (`names/{safe}/played`). Plan A: when the name is reserved and not verified on this device, Crear sala / Reto diario / Unirse ask to verify (opens the claim panel) or to choose another name. Practice could stay open. **When A is built, update the texts again**: `descFree` can then promise "nadie más podrá usarlo" again, and the home ❓ "🔒 Reserva tu nombre" line too. Also the security build: hash a PIN with a per-name salt / move secrets out of the open `names/` node.
- Tests: ALL GREEN (follow 42, build5b 77, …). Help checked at 320 FR / 390 ES / 390 EN: 4–5 sections each, fits one phone screen. The home overflow at 320 px is the known emoji-grid issue, not the help.

### v261031.374-tmp (staging, 31 Oct) = v373 help + two fixes
- **Siguiendo no longer lists followed players who haven't played** (Oscar: confusing). This replaces the 15 Oct "show them greyed at the bottom" decision. Shows me + followed players who played that day. If none played: "nadie a quien sigues ha jugado todavía" (`FT.noneYet`; `FT.noFollow` when you follow nobody; `FT.noMatch` when searching). The `.lb-notplayed` / `.lb-np` rows, CSS and the np/npShort/npPast strings are removed.
- **Home emoji grid at small widths (open since Session 26):** at 320 px the 8th column was cut, and at 360 px it spilled past the card edge (both measured). `.emoji-grid` columns are now `repeat(8, minmax(0, 1fr))`, and under 390 px `.emoji-opt` uses 20px with padding 2px 0 (existing slot). Same class on the invitation and welcome-back grids. Measured at 320/360/375/390/414: 8 columns, last emoji inside the card, no overflow.
- follow_scenarios 47/47 (F5/F6/F8/F9 updated for no not-played rows; +F16 emoji grid at 5 widths; F16 fails on v373 at 320 and 360). ALL GREEN.

### v261031.375-tmp (staging, 31 Oct): help gets its old look back
- Oscar: the v373 text was right but "too sparse": he missed the old look (big emoji per step, handwritten text, points table) and asked for about the old length, not less. `HELP` is now a list of items: `['h', heading]`, `['s', icon, text]` → `.rules-step/.rules-icon/.rules-text` (Caveat 20), `['pts', title, [[label, pts]…]]` → `.rules-scoring` rows with red `.rules-pts`, `['note', text]` → `.help-note` (paper-dark box, Special Elite 11), `['hr']` → `.help-hr`. New CSS: `.help-h` (Caveat 20 bold), a dashed line under the points table, points never wrap.
  - home: 🎲 Clásico (🔗 🔤 🛑) · divider · 📅 Reto diario (📅 🏆 ✏️) · note 🔒 Reserva tu nombre.
  - daily: 📅 ✏️ 🤖 steps · Puntos table (válida 100, dudosa 50, inválida 0, velocidad ×1–×2, única +50) · 🏅 📆 👀 steps (coloured ☆★★) · note ✏️ Práctica.
  - multi: 🎲 ✏️ 🛑 steps · Puntos table (única 100, repetida 50, anulada/vacía 0, ¡Alto! sin fallos (con 8 categorías) +250, ¡Alto! por cada anulada −100) · 👑 🗳️ 🤖 😂 🎮 steps · note: solo = práctica, unfinished games don't count.
- French: non-breaking space before : ? ! in the help (a lone ":" started a line at 320 px).
- Checked at 320 FR / 390 ES / 390 EN: no points row overflows. Heights: home ~500–620 px, daily ~700–850, multi ~860–1010 (scrolls). ALL GREEN.

### v261031.376: PRODUCTION (31 Oct) = v375-tmp + home help layout
- **Home help** reworked to follow the original menu help (Oscar): a section title, an intro line without an icon (invisible `·` with `opacity:0`, the same small offset as the original, on purpose), then emoji steps; no emoji repeated in the panel (Práctica uses 🔁). New item type `['i', text]`.
- **Help titles `.help-h` = 22px bold Caveat**, exactly the original help titles (pre-v373 inline style). Oscar: keep consistent with the old look. 22px is outside the six font slots, but it was already in the original help. The note box is 11px Special Elite (the old home "Validación" box was 12px).
- Includes everything since v372: lean help content in the rules-card look (v373/375), honest name reservation texts, Siguiendo without not-played rows, the emoji grid fix (v374). News `2026-10-31-follow` ("Toca ☆ en la clasificación…") still runs until 2026-11-04.
- ALL GREEN (follow 47, build5b 77, groups 28, restore 19, countdown 12, stuck 11, unit suites), checklist clean, 421 KB.

## Session 29 summary (10–31 Oct). Production v261010.363 → v261031.376
- **Lobby:** compact redesign (v358–363): "¡comparte!" + red code, settings bar with live summary, validation mode inside settings as theme-style cards, per-name settings memory, guests see the host's choices.
- **Robot:** Sante and Ling Fin disappeared from OpenRouter (13 Oct) → `AI_MODELS = ['inclusionai/ling-3.0-flash', 'cohere/north-mini-code:free']` (v364). Judge ignores capital letters; neutral re-check prompt with a spelling example and no letter-fitting (v367). Scout v5.7: 34/36, re-checks 20/23. One-letter answers rejected locally. "Animal prehistórico" → "Animal extinto". Practice countdown 5 s.
- **Follow system** (designed 12 Sep, restored and built): ☆/silver ★/gold ★, Todos/★ Siguiendo, always-visible search, legend, `reg` flag + registered-name lookup for older scores (v368–371, v374). Daily ties share medals like multiplayer. Fix: today's answers no longer revealed after ◀ ▶.
- **Help** rewritten (v373–376) + honest name reservation texts. **Emoji grid** fixed below 390 px.

## START HERE (Session 30). Supersedes earlier START HERE sections
1. Upload the project zip (with `tests\` and `tools\`; replace `tests\restore\follow_scenarios.py` with the v376 one, 47 checks) and read this handoff from "Session 29 summary". **Production = v261031.376.** First thing: `cp index.html index_backup_s30.html`.
2. ⚠️ **Must do first: the reserved-name gate (plan A, see the v373 entry).** → **BUILT on staging v261031.377-tmp (Session 30), see below; waiting for Oscar's test, then promote.**
   ⚠️ **Must do first: the reserved-name gate (plan A, see the v373 entry).** A reserved name is not protected today. Gate Crear sala / Reto diario / Unirse when the name is reserved and not verified on this device (verify or pick another name; practice can stay open). Then update the texts back: `descFree` ("nadie más podrá usarlo") and the home ❓ 🔒 note. Confirm the plan with Oscar first.
3. Then: the security build (`names/` open: secrets readable, 4-digit PIN hashes trivially reversible, `following`/`followers` editable), AI step 1.
4. Watch the dailies for Ling 3.0 Flash: re-check over-generosity (Library, Edredón, Esterilla), FR English words (Robin), flips.
5. Small parked: ~~flaky build5b "Marta's buttons…" check~~ (fixed in the v381 test kit), the "top 10" cut can split a tie, own-row name squeezed at 320 px with the "(+50✨)" tag, search covers only the viewed day, the unused T key `rulesBtn`.
6. Rules unchanged: confirm the plan before building (mockups as plain text, or a quick screenshot before long tests: Oscar prefers seeing it before a 10-minute test run), deploy right after building, check the version with grep before and after, ASCII commit messages, the six font sizes (documented exceptions: 19px daily/practice buttons, 22px help titles), the handoff is append-only.

## Session 30 (31 Oct)
- Start: production v261031.376 confirmed with grep, backup `index_backup_s30.html`, checklist clean, test kit ALL GREEN on production.

### v261031.377-tmp (staging, 31 Oct): reserved-name gate (plan A)
- **Plan confirmed by Oscar before building.** `nameGateOk(name, nameElId, retry)` (in the claim block, after `claimClose`) is called by `createRoom`, `joinRoom`, `quickJoin` (invite link, field `qj-name`) and `startDailyChallenge` (also covers "¡Hecho! Ver puntuación" and the duplicate-daily re-call), right after the empty-name check.
  - Passes: names starting with `__` (test/debug), safe name shorter than 2, `alto_claim_{safe}.verified` on this device, no `names/{safe}/secret`, or the Firebase read failing / taking more than 5 s (never lock anyone out on a bad network).
  - Blocks: reserved and not verified here → sets `_claimCurrentName`, `_claimState='taken'`, `_gatePending = {retry, nameElId}` and opens `claimOpen(name)` in **gate mode** (overlay class `gate`): title `gateTitle`, text `gateDesc`, button `btnGate` ("Verificar y jugar →"), and `#claim-other` "Usar otro nombre" (CSS-only show via `#claim-overlay.gate #claim-other`).
  - After a correct email/PIN, `claimSubmit` restores the accented display name into the field the gate came from (`nameElId`: `inp-name` or `qj-name`), closes, and runs `retry` (the same entry function again, which now passes). Wrong PIN: error, nothing happens.
  - "Usar otro nombre" (`claimGateOther`) closes and focuses + selects that name field. `claimClose` always clears `_gatePending` and the `gate` class; opening from the pill clears it too (normal panel, no continue).
  - **Not gated:** Práctica, session restore, welcome back.
  - Also closes the invite-link host takeover (`quickJoin` made you host when your name equalled `room.host`) for reserved names.
- **Found:** there is no "join by code" field on the page any more (`#inp-code` doesn't exist), so `joinRoom` is dead code; all joins go through the invite link. Gate added there anyway; the test injects the field.
- **Texts restored (ES/EN/FR):** `descFree` again promises "nadie más podrá jugar con él" / "nobody else can play with it" / "personne d'autre ne pourra jouer avec"; home ❓ note "🔒 Reserva tu nombre (opcional): nadie más podrá usarlo, …" (EN "nobody else can use it", FR "personne d'autre ne peut l'utiliser"). New CLAIM_LABELS keys: `gateTitle`, `gateDesc`, `btnGate`, `gateOther`. Fonts: existing slots only (13px link, 26/32/13/11 in the panel as before).
- **Still true / not done:** the protection is client-side only. While `names/` is open, someone with the console can still write under any name. The security build is still needed (salted PIN, unreadable secrets, rules). Meta descriptions still say "sin registro", which is still correct (reserving is optional).
- **Tests:** NEW `tests/restore/gate_scenarios.py` (40 checks): Crear sala gated + wrong/right PIN + room created by itself + device verified; verified device not gated; "Usar otro nombre" focus; pill opens the normal panel; joinRoom; invite link with accent restore (José) + focus; host takeover; daily gated / Práctica open / daily after PIN; free name, `__` name and Firebase error let through; EN/FR texts + 320 px fit; descFree + help note. Fails on v376 (as it should). Full kit on v377-tmp: ALL GREEN (gate 40, follow 47, build5b 77, groups 28, restore 19, countdown 12, stuck 11, unit 239). 426 KB.
- **Delivered:** `jugarenfamilia_tmp.html` + `gitinfo_tmp.txt` + `tests_kit.zip` (adds gate_scenarios.py). The handoff goes with the production promotion.

### v261031.378: PRODUCTION (31 Oct) = staging v377-tmp (reserved-name gate)
- Promoted at Oscar's request, no code change from v377-tmp apart from the version string. Version checked with grep before (prod v376, staging v377-tmp) and after (v378 in both places). Checklist clean (426 KB, 12 screens, init block, syntax OK, no -tmp). Test kit ALL GREEN on v378 (gate 40, follow 47, build5b 77, groups 28, restore 19, countdown 12, stuck 11, unit 239).
- The texts promising "nadie más podrá usarlo" (descFree + home help note) are now true in production.

### Session 30 code audit (31 Oct). Doc: https://claude.ai/artifact/6mLawbbQxMnDKr75nvXfRH
Read-only audit of v377-tmp: parser pass over the 372 top-level definitions + scripted scans (unescaped templates, Firebase calls, element ids, CSS, fonts, translations) + reading the risky paths. 15 findings. **Nothing fixed yet; each fix needs Oscar's OK as usual.** Line numbers are v377/v378.
- **S1 HIGH, reproduced on v376:** the daily leaderboard puts other players' text into innerHTML unescaped: answers in the answer panel (`${val}`, ~line 3908) and names (`${e.name}`, ~3943). A 38-char daily answer `<img src=x onerror="...">` (input allows 40) runs script for every visitor who opens that row. Fix: `escHtml` on name, answer, category there and in showDailyResult (~3577).
- **B1 HIGH, proven:** validation-row buttons (✕ toggleInvalid, 👎 voteEntry, reactions reactToEntry, 📖, 🤖 askAI) are built as `onclick="fn('${key}',...)"` with key = player + '__' + category, and rooms store translated category names. FR "Film d'animation" → syntax error → every button on that row does nothing (host can't invalidate). Host 🤖 (~5517) also uses `ans.replace(/'/g,"\'")` inside a template literal, which is a no-op → any answer with an apostrophe breaks it. Fix: data- attributes + one delegated listener.
- **B2 HIGH:** `startDailyTimer` counts down with `secsRemaining--`; phones suspend background pages, so switching apps freezes the clock (look answers up, keep the time, better speed bonus). Reload already restores from startTimestamp. Fix: compute from startTimestamp each tick + on visibilitychange. **B6 (low):** same for multiplayer `startTimer`, derive from roundStartTime.
- **Medium:** S2 names/answers unescaped in validation rows (~5456, 5524), floating reactions (~6220), dictionary word (~5660-5725); S3 robot result HTML in the room (already Security item 2); B3 `. # $ [ ] /` in names or free categories are invalid Firebase keys (players/${name}, votes/${key}) → writes throw, nothing cleans them (fake Firebase in the kit doesn't reject them, so untested); P1 daily reads the whole day's scores several times per visit (leaderboard, originality, already-played x2); H1 handoff size/contradictions; B4 daily date from the phone clock.
- **Low:** B5 name fields lowercase everything after the first letter ("Tío juan") + #qj-name rule 8; B7 quickJoin sets G.playerName before checking the room, a failed join leaves it and the daily uses G.playerName before the name field; D1 dead code (applyDictLookup, buildReactionBar, clearGuestDictResults, calcSpeedBonus, playerDisplayName, showJoin, THEMES proxy, lbRef, LANG_FLAGS, DEBUG_NAME, G_debugMode, joinRoom (no #inp-code on the page), the dictLookup room write, ft-rnd-cur/ft-rnd-tot/lbl-round-of lookups, debug daily preview's #daily-inputs, ~18 CSS classes, ~12 T keys); D2 legacy daily/.../players index still written + backfill; D3 duplicate CSS selectors (.page, .guest-confirmed x3, .rules-scoring/.rules-score-row).
- **Fine:** font slots, translations complete in ES/EN/FR, room listener/state machine, restore paths, room size, page weight (415 KB raw, ~115 KB gzipped).
- **Proposed new rule 16:** any player-typed text (names, answers, group names, free categories) goes into the page through `escHtml()` and never inside an `onclick` string.
- **Proposed builds:** v379-tmp escaping (S1-S3) → buttons and names (B1, B3, B5, B7) → clocks (B2, B6, B4?) → cleanup (D1-D3 + P1 cheap fix: read your own entry via names/{safe}/played/{date}/{lang} = playerKey). Then the handoff split (HANDOFF.md current state + HISTORY.md append-only, nothing deleted) and the security build.
- **Oscar's answers (31 Oct):** promote v377-tmp → done as v378. B4 (Spain's date for everyone via server time): undecided. About 20-25 daily players, so P1's bigger fix (move aiResponses/aiInfo/contestLog out of the score entry) and P2 (100-connection ceiling) are not needed yet; revisit at ~100 daily players.

## START HERE (Session 31). Supersedes earlier START HERE sections
1. Upload the project zip (with `tests\` and `tools\`; the kit now has `restore/gate_scenarios.py`, 40 checks) and read this handoff from "Session 29 summary". **Production = v261031.378.** First thing: `cp index.html index_backup_s31.html`.
2. Open the audit doc (link above). Next build = **escaping (S1, S2, S3)**, the leaderboard one first; add the hostile-text scenario first. Confirm the plan with Oscar.
3. Then buttons/names, clocks, cleanup, handoff split, security build. B4 needs Oscar's decision.
4. Rules unchanged (plan first, deploy right after building, grep the version before and after, ASCII commit messages, six font slots, append-only handoff).
- **Decision (Oscar, 31 Oct): B4 dropped.** The daily stays on each device's local date: it's an international game and each country's players use their own clock. Changing the phone's clock to see tomorrow early is accepted ("just a game"). Don't propose a server or Madrid date again. B2 (the daily clock freezing on an app switch, also by accident) stays in the clock build.

### v261031.379-tmp (staging, 31 Oct): escaping build (audit S1, S2, S3 + B1)
- **Plan confirmed by Oscar.** Built from production v378.
- **S1 (daily):** `escHtml` on the leaderboard name (`e.name`), the answer panel's category and answer, and your own result rows (`showDailyResult`).
- **S2 (rooms):** `escHtml` on the validation rows (player name and answer, host and guest), the category headers on validation cards, the playing-screen category labels and input ids (`a_${cat}`), floating reaction name + emoji, emojis in the lobby chips / score rows / final rows / `mpRowHTML`, and the dictionary output (the word, plus Wikipedia/dictionary titles, extracts and definitions, which are external text).
- **S3 (robot):** new `aiClsOf(d)` (whitelist valid/invalid/unsure; older rooms' `html` only used to recover the verdict) and `aiLabelHtml(cls)` (the old label code moved out of `askAI`). The host now writes `aiResults/{key} = {cls, key}`, with no html. Every reader (guest/host validation builders, `applyAIResults`) builds the label locally, so each device shows its own language. Mixed versions: a v378 guest in a v379 host's room won't see the robot verdict (it reads `html`). Rooms are short-lived; all players should be on the same version during the staging test anyway.
- **B1, fixed here instead of in the next build:** new `jsq(v)` = JS backslash-escape then `escHtml`, for values inside quoted JS strings in attributes (`onclick="fn('${jsq(v)}')"`). Applied to every onclick carrying player data: validation 📖 / 🤖 / ✕ / 👎 / reactions (key, answer, category, letter), dictionary suggestions, the 🤖 retry, the daily ¿Error? button. Ids with `${key}` use `escHtml(key)`. Fixes FR "Film d'animation" and answers like L'ours, and closes attribute injection through `"` in answers. The data-attribute refactor in the audit plan is no longer needed. The old `gSafeAns` / `safeAns` variables are now unused (left in place; remove in the cleanup build).
- **Not in this build:** B3 (`. # $ [ ] /` in names or free categories, still invalid Firebase keys), B5, B7, and the clocks.
- **New rule 16 (proposed in the audit, now in force):** player-typed text goes into the page through `escHtml()`, and into an onclick string only through `jsq()`.
- **Tests:** NEW `tests/restore/hostile_scenarios.py` (24 checks): H1 daily leaderboard with a hostile name and answer; H2 FR Libre room with Film d'animation, `<i id=pwncat>` category, Tom & Jerry, guest named `<b id=pwnp>Ana`, hostile answers (script, attribute injection, apostrophe); H3 ✕ / reaction / 🤖 on the Film d'animation row actually work (fake robot reply via a route on the worker URL); H4 a hostile stored `aiResults` entry (html with onerror + a bad cls) is not shown as HTML, and the host stores only `{cls, key}`; H5 scores screen. 18 of these fail on v378. test_build2_reactions + test_revisar_keeps_decisions now also grab escHtml and, as optional (grabOpt), jsq/aiClsOf/aiLabelHtml, so they run on v378 and v379. Full kit on v379-tmp: ALL GREEN (hostile 24, gate 40, follow 47, build5b 77, groups 28, restore 19, countdown 12, stuck 11, unit 239).

### v261031.380-tmp (staging, 31 Oct): names (audit B3, B5, B7), on top of v379-tmp
- **Plan confirmed by Oscar.** Oscar can't test multiplayer right now, so v379 and v380 will be promoted together after one real game.
- **B3:** new `KEY_BAD_RE` / `hasBadKeyChars` / `tidyNameInput` / `nameCharsOk` (block "NAME CHARACTERS", before the name gate). Both name fields drop `. # $ [ ] /` while typing (cursor kept in place). `createRoom`, `joinRoom`, `quickJoin` and `startDailyChallenge` refuse a name that still has them (saved before v380), with toast `toastNameChars`, before the name gate. Free categories with them are refused on Empezar with `toastCatChars` ("Quita . # $ [ ] / de «{cat}»"), and the text is not changed silently. New T keys in ES/EN/FR. Built-in category names (all 174, three languages) were checked: none contain these characters. The daily key was already cleaned (`playerKey`), so the daily refusal is only for consistency.
- **B5:** only the first letter is forced to a capital; the rest stays as typed (Tío Juan, Mari Carmen). Side effect, accepted: "PEPE" stays "PEPE" (claims, daily `safeName` and group keys are lowercased anyway; only room names and the host-rejoin-by-exact-name compare case). `#qj-name` lost its inline oninput (rule 8, the watch-list item is closed) and uses `addEventListener` + `tidyNameInput`.
- **B7:** `startDailyChallenge` and `dailySubmit` use the name field, never `G.playerName`.
- **Test kit:** the fake Firebase (`restore_base.py`) now refuses keys with `. # $ [ ]` at any depth, like the real one (`check_keys`). No existing scenario relied on such keys. NEW `restore/names_scenarios.py` (20 checks: tidying in both fields, cursor, capitals, the old saved "J.R." refused with a toast (ES/EN) for room and daily, free category "Ciudad/Pueblo" refused then accepted after fixing it, the daily saved as Pepa after a failed invite join as Marina). 10+ fail on v379-tmp.
- **Full kit on v380-tmp:** all green except build5b's known flaky "Marta's buttons…" check, which failed 2 checks in the busy full run. Rerun 3 times each on v380-tmp, v379-tmp and v378: 0 failures. Total: names 20, hostile 24, gate 40, follow 47, build5b 77, groups 28, restore 19, countdown 12, stuck 11, unit 239.
- **Next (audit plan):** clocks (B2 daily clock from `startTimestamp`, B6 multiplayer from `roundStartTime`), then cleanup (D1-D3 + P1 cheap fix, including the now-unused `gSafeAns` / `safeAns`), then the handoff split.

### v261101.381-tmp (staging, 1 Nov): clocks (audit B2, B6), on top of v380-tmp
- **Plan confirmed by Oscar.** v379 + v380 + v381 are to be promoted together after one real multiplayer game.
- **B2 (daily):** the countdown existed in 3 copies (normal start in `startDailyTimer`, restore after reload, and "back from another screen"), each doing `secsRemaining--`. They are replaced by `dailySecsLeft()` (= totalSecs − round((now − startTimestamp)/1000)), `dailyTick()` and `dailyRunTimer()`, plus a `visibilitychange` listener that ticks at once on return (and submits if time ran out). A phone that suspends the page no longer freezes the clock, and the speed multiplier no longer rewards time spent away. Measured: 50 s away gave ×1.76 on v380 and ×1.42 on v381.
- **Correction to the audit:** the third copy ("navigated away and came back", in `startDailyChallenge`) is effectively dead. `resetGDaily(false)` runs first and clears `G_daily.key`, so the reload-restore path (already real-time) handles it. Leaving the daily for the home screen never paused the clock. The copy now uses the shared tick anyway; remove it in the cleanup build.
- **B6 (multiplayer):** `startTimer` counts down to `endAt = now + secs`, with `t = round((endAt − now)/1000)` each tick. The existing visibilitychange handler (Firebase elapsed check) is unchanged. `doSubmit` and `dailySubmit` both already guard against a second call.
- **Not changed:** the 10 s / 5 s countdown overlay still counts by subtraction (it only delays the start; the categories are hidden).
- **Tests:** NEW `restore/clock_scenarios.py` (12 checks), using Playwright `clock.install()` + `clock.fast_forward()` (which fires waiting timers once, like a suspended page): C1 practice, 30 s away costs 30 s and running out while away submits on return; C2 daily, leaving for home and coming back (regression check, passes on v380 too); C3 the speed multiplier after 50 s away; C4 multiplayer guest, 25 s away. 5 checks fail on v380-tmp.
- **The flaky build5b check "Marta's buttons are the same elements…" is fixed (test only).** A MutationObserver probe showed that in failing runs Marta's `#emojis_…` / `#votes_…` buttons were rebuilt 16–32 ms after being tagged, before Pepe's first tap reached her: the tail of her own switch to validation, not other players' taps. The game behaves as intended. The test now waits 600 ms before tagging. 5/5 clean alone and in the full run (it had failed 2-3 times in the previous runs on v380/v381).
- **Full kit on v381-tmp: ALL GREEN** (clock 12, names 20, hostile 24, gate 40, follow 47, build5b 77, groups 28, restore 19, countdown 12, stuck 11, unit 239).
- **Added to the cleanup build (from Oscar, 1 Nov; noted in an earlier session but never written here):** the comment `/* Help texts moved to helpContent()/helpSectionsHTML() (v373) */` (~line 1700) names a function that doesn't exist (`helpContent`). It should say the help texts live in `HELP` (~line 2397) and are drawn by `helpSectionsHTML()` (~line 2492). Comment only, no behaviour change.

### v261101.382-tmp (staging, 1 Nov): cleanup (audit D1, D2, D3, P1 cheap fix), on top of v381-tmp
- **Plan confirmed by Oscar.** No behaviour change for players apart from fewer database reads. 429 KB → 417 KB.
- **D1, dead code removed** (each had no reference left): `applyDictLookup`, `buildReactionBar`, `clearGuestDictResults`, `calcSpeedBonus`, `playerDisplayName`, `window.showJoin`, the `THEMES` proxy, `lbRef` (watch-list item closed), `LANG_FLAGS`, `DEBUG_NAME`, `G_debugMode`, **`window.joinRoom`** (no code field on the page; every join goes through the invite link and `quickJoin`), the `dictLookup` room writes (no reader), `gSafeAns` / `safeAns`, the `ft-rnd-cur` / `ft-rnd-tot` / `lbl-round-of` lookups, and the dead "came back from another screen" resume path in `startDailyChallenge` (`resetGDaily` clears the key first; a comment marks the spot). 18 unused CSS classes removed (btn-leave-room, cat-grid, caveat, code-sm, config-toggle, daily-already-played/-score, democratic-badge, democratic-draw, entry-reactions, help-section/-text/-title, lang-bar, opt-note, rules-box, rules-toggle, share-box, stamp; 46 lines). Kept because they are built at runtime: `s-none/s-follow/s-mutual`, `screen-welcome`.
- **T object (translations):** removed with the parser (acorn), not text replacement: 12 unused keys in all 3 languages (reviewSub, endGame2, toastNoGroup, toastFreeCatsRandom, invited, rulesBtn, expBtn, wrongPwd, expActivated, shellLeaveYes, shellLeaveNo, toastRoomCopied). **New finding:** `T` held keys defined 2–6 times in one language, including leftover **Portuguese** ("escreve as tuas categorias", "Jogadores do Barça") and **Italian** ("di", "scrivi le tue categorie") values for `of`, `freeCatLabel`, `freeCatPlaceholder` (plus `calculate`, `stopCallerMsg`, `timerRanOut` twice). The last copy wins and was always the right language, so players never saw them. 30 earlier copies removed; every effective value was checked identical before saving. **T now has exactly 140 keys in each of ES/EN/FR, no duplicates.** This is the "T object corruption" from rule 2.
- **Debug daily/practice preview fixed:** it drew into `#daily-inputs`, which no longer exists, so it was empty. It now uses `#daily-inputs-card` with the real row markup.
- **Stale help comment fixed:** now says the texts live in `HELP` and are drawn by `helpSectionsHTML()`.
- **D2:** the legacy `daily/{date}/{lang}/players/{safeName}` index is no longer written, and `fetchPlayedState`'s backfill from it is gone (it only checked today's date, so it only helped plays from before 13 Sep, at up to 3 reads per name typed). Old data can stay in Firebase or be deleted in the console.
- **D3:** duplicate CSS merged, same result: `.page` (the first copy was a subset), `.guest-confirmed` + `.gc-emoji` + `.gc-status` (kept at the later position), `.rules-scoring` (one rule with both dashed lines), `.rules-score-row` (+gap).
- **P1 cheap fix:** new `fetchMyDailyEntry(safeName, key)`: playerKey from `_playedLangs` or `names/{safe}/played/{date}/{lang}`, then one score entry; the whole-day scan is only a fallback for older entries stored as `true`. Used by both "already played" paths (this device / another device). Reopening a played daily now downloads the whole day once (for the leaderboard) instead of twice.
- **Visual check:** computed styles (19 properties + position/size) of every element on all 25 debug screens at 390 px, v381 vs v382, fresh page per screen, random animations skipped: the only differences are the repaired debug preview (inputs now drawn, page taller) and the bouncing ✏️ caught mid-animation.
- **Tests:** NEW `restore/cleanup_scenarios.py` (10 checks: own entry read directly, the whole day at most once, an older `played: true` entry still found, no legacy index written, debug preview inputs; 4 fail on v381). `gate_scenarios.py` lost G6 (it called `joinRoom`): 38 checks now. Full kit on v382-tmp: ALL GREEN (cleanup 10, clock 12, names 20, hostile 24, gate 38, follow 47, build5b 77, groups 28, restore 19, countdown 12, stuck 11, unit 239).
- **Audit plan done** except the handoff split (H1). Staging stack waiting for a real multiplayer game: v379 (escaping) + v380 (names) + v381 (clocks) + v382 (cleanup), to be promoted together.


### Session 30 (1 Nov) — handoff split (audit item H1)
- The handoff had grown to ~250 KB / ~93k tokens with six START HERE sections and statements contradicted further down (robot models, recontest "3 models", Registration Gate scope, items flagged as future but already built). A session could read only about a quarter of it per pass.
- Split agreed with Oscar: `JUGARENFAMILIA_HANDOFF.md` now holds only the current state (~31 KB, read in full every session; facts edited in place); this file holds everything above, untouched, and grows append-only. `deploy.bat` gains the lines to copy and commit this file (Oscar replaces the bat by hand).
- Facts in the new handoff were checked against the code of v261101.382-tmp (constants, schema keys, AI settings, news, test files), not only copied from the log.

### v261101.383: PRODUCTION (1 Nov) = staging v382-tmp (audit builds v379 + v380 + v381 + v382)
- Oscar asked to promote the staging stack without the planned real multiplayer game first (his call; the kit's two-browser multiplayer scenarios cover the changed flows). Diff vs v382-tmp: version string only. Version grep before (prod v378, staging v382-tmp) and after (v383 twice). Checklist: 12 screens, init once, syntax OK, 3 `-tmp` = version-check regexes, 417 KB.
- Delivered with the handoff split (new short `JUGARENFAMILIA_HANDOFF.md` + this file) and the updated `deploy.bat` (Oscar replaces it by hand).



## Session 31 (1–3 Nov) — Production v261101.383 → v261103.385

### Housekeeping (1 Nov)
- `deploy.bat` from Session 30 confirmed in place in `D:\09_ALTO\` (the history file now reaches GitHub).
- **Firebase cleanup done by Oscar:** JSON backup exported; `global/`, `groups/`, `groupNames/` deleted in the console; their three lines removed from the rules (checked first: no code in v383 reads or writes them; only `rooms`, `daily/*/*/scores`, `names`, `groupsMeta`, `groupsLb`, `history`, `historyIndex` are used). Site checked afterwards, works. `daily/*/*/players` left in place (harmless; optional cleanup).
- Real multiplayer games on v383: not reported yet.

### Daily robot review, 24 Oct – 1 Nov (114 dailies, ~640 robot calls)
- Ling 3.0 Flash judged practically everything. Daily-by-day median 1.8–2.7 s, 95th percentile 3–10 s. Cohere free was only used twice (two ¿Error? checks, 11–38 s).
- **Robot failures came in bursts that hit the whole first batch of 3 answers:** 26 Oct 11:27 Cocoricó (3 answers timed out twice, 40 s; the phone then slept mid-judging, 22–27 min; all 6 🤔, she spent 5 ¿Error? on Italia, Iglesia, Iguana, IBM…); 26 Oct 18:07 Giocoso (3 HTTP errors at once; ¿Error? on Menuisier, Mangue, Mercure); 29 Oct Cocoricó (2 HTTP errors). The error text was not saved, so the cause (worker / OpenRouter / provider) is unknown.
- **¿Error? too generous:** 111 checks, 27 overturned: 7 were just connection failures, ~7 right (Barrenero, Patatas a la importancia, Piña, Mesa camilla, Omoda…), ~12 wrong (Roberto as writer, Paulin as scientist, Rúcula as spice, Jolie as film, Radar as tool, Volvo 390 as brand, Audi as tech brand, Deutsch in EN, Judea as city, Orlando hat, Géants as TV series, Admira; borderline Gulag ×2, IG). Not only on 🤔: several were ❌ before.
- **First judge patterns:** bare first names accepted as people (Roberto, Alan, Nino, Roman); accent-only differences rejected in ES (Violin = violín, "Vals vienes" = vals vienés); near-foreign spellings accepted in FR (Giraffe, Piton, Jazmin) while clearly Spanish words were caught (Jirafe, Pirámide, Romero); other generous calls: Barca as invention, MG as tech brand, Gabard, Incubus, Guzmán. Too strict: Poto (pothos), Violin, Velocirraptor (Fundéu's preferred spelling; "Velociraptor" was ✅ the same day). Good calls: Visón marino ✅ / Vison ❌, Rinoceronte negro del oeste ✅ / Rinoceronte negro ❌, Pulpo not a fish, E.T. not animated, Vena aorta ❌, Oran not a capital.
- **Midnight bug found (30 Oct, David, EN):** daily started just before midnight, submitted 00:01, ¿Error? at 00:02 → the 4 checks were judged against the NEXT day's categories (Litographie as "Animated film", Leggins as "Fruit"), and the contest log went under 31 Oct as a nameless entry (filtered out of the leaderboard, harmless). Cause: `showDailyResult` always drew today's categories from `getDailySetup()`, and `recontest` used `getDailyKey()`.

### v261103.384-tmp (staging): Build A (robot reliability, midnight) + Build B (judge rules)
Plan agreed by Oscar (A with the automatic second look; B "no changes").
- **A1 midnight:** `showDailyResult` uses `G_daily.categories` (set by every path that opens it); ¿Error? is offered on the game shown (`isToday = !practiceMode`); the leaderboard opens on `G_daily.key` (yesterday's board for a daily finished after midnight); `recontest` uses `G_daily.key`. The result header date still shows today's date (cosmetic, left).
- **A2:** `aiJudgeCall` retries once after 1.5 s on a connection/HTTP error (timeouts keep their own retry on `models.slice(1)`).
- **A3:** the error text is saved: `aiInfo[i].msg` (≤200 chars); `dailyAICallModel` returns `errMsg`.
- **A4 automatic second look:** new `dailyAutoRejudge()` (called at the end of `showDailyResult`): answers that are 🤔 AND have `aiInfo.error` AND no `aiInfo.rejudged` are judged once more (in parallel), ⏳ while checking, ¿Error? hidden on those rows meanwhile. Writes `aiInfo/{i}` = `{model, s, rejudged:true, first, firstMsg?, error?, msg?}`, and on a verdict `aiResults/{i}`, `aiResponses/{i}` (if ❌), recounted `baseScore`/`totalScore` (scratch recount), plus localStorage. Still failing → stays 🤔 with `rejudged:true` (no loop) and ¿Error? offered. Practice / `__` names: in memory only. Does not use up ¿Error?. `G_daily.aiInfo` is set on submit and by both "already played" paths (from the Firebase entry). New helper `dailyContestBtnHTML(i, word, cat)`.
- **B, judge prompt (`dailyBuildPrompt`, all modes):** rule (1) "An answer that is right apart from its accents is the {lang} word, even if without them it looks like a word from another language" with `ACCENT_EXAMPLE` (es violin/vals vienes, en cafe/naive, fr eleve/foret); rule (3) "The same goes for a common noun spelled the way another language spells it, even when it is close…" with `NEAR_FOREIGN_EXAMPLE` (es Giraffe/Jasmine, en Girafe/Jasmin, fr Giraffe/Piton/Jazmin); rule (6) a first name on its own is INVALIDO in a category of people unless famous under one name (Madonna, Shakira, Pelé), a well-known surname alone is fine (Pasteur, Goya), not when the category asks for a first name. Multiplayer Estricta/Relajada/spelling swaps untouched.
- **B, ¿Error? prompt (method B):** ignores accents (same examples); "rule VALIDO only if the answer clearly and genuinely belongs to the category… A vague or partial answer stays INVALIDO: a first name on its own…, a model instead of a brand, a region instead of a city, a brand of another kind (a car brand is not a tech brand). If in doubt, rule INVALIDO." (replaces "including uncommon but real answers"); people categories need "a specific person"; near-foreign spellings stay INVALIDO.
- **Scout v5.8** (`tools/ai_scout.js`): same prompts; +14 daily and +11 re-check real cases from 24 Oct–1 Nov.
- **Tests:** `test_judge_prompt.mjs` 26 → 41 (new rules in every language, kept with the multiplayer options, contest strictness, scout contest system prompt identical). NEW `restore/robot_scenarios.py` (24 checks): R1 robot failures (fail twice → automatic second look ✅ with `first`/`firstMsg`; fail once → fixed by the retry; always failing → 🤔, `rejudged`, ¿Error? offered, no loop after reopening; no ¿Error? used; localStorage updated); R2 midnight (clock installed at 23:59:30 UTC: result rows show the played categories, leaderboard on the game's day, ¿Error? logged under the game's day with its category, nothing under the new day). On v383, R2 reproduces David's case (13 failures). Full kit ALL GREEN.
- Staging delivered 3 Nov with scout v5.8 and `tests_kit.zip`.

### Scout v5.8 on v384-tmp (Oscar, phone hotspot, 3 Nov)
- Ling 3.0 Flash: **judge 42/50** (ES 16/22 · EN 11/11 · FR 15/17), 1 unsure, 4 flips; **¿Error? 28/34**; avg 1.9 s, slowest 10% 2.8 s, 0 timeouts.
- Old 36 daily questions: 33 (misses Tomate seco en polvo, Robin, "Vallas, salto de"; v5.7 baseline 34/36, within flip noise). New: fixed Roberto, Alan, Nino, Giraffe, Jazmin, Vals vienes; controls Visón marino, Pasteur, Victor kept; still wrong Violin (❌), Poto (🤔), Piton, Barca, MG.
- Re-checks: old 23 → 21 (misses Library, Esterilla; baseline 20/23). New: fixed Roberto, Paulin, Rúcula, Audi (kept ❌), Poto, Violin (overturned); still wrong Deutsch, Orlando hat, Radar (overturned); one loss: Patatas a la importancia now upheld (obscure real dish).
- Verdict: promote.

### v261103.385: PRODUCTION (3 Nov) = staging v384-tmp
- Diff vs v384-tmp: version string only. Version grep before (v384-tmp ×2) and after (v385 ×2). Checklist: 12 screens, init block once, no `-tmp` in the version (the 3 regex `-tmp` remain), `node --check` clean, full kit on the production file ALL GREEN. 426 KB raw, ~117 KB gzipped.

### Daily speed bonus redesign — agreed 3 Nov, NOT built (next session)
- **Problem (Oscar):** ¡Alto! in the daily works with one answer filled, and the speed multiplier (1 + secsLeft/90, up to ×2) applies to the whole score, so skipping hard categories to finish early pays. Real case 29 Oct ES: Venirse primos, 5 ✅ + 1 blank, ×1.52 → 1010 beat two 5 ✅ + 1 ❌ players (840). 6 of 114 dailies had blanks plus a bonus.
- **Rejected:** blocking ¡Alto! until all filled (players just type junk, same effect; in the daily ¡Alto! only affects yourself, unlike multiplayer); B (bonus × share of valid answers, too mild); C (bonus only for a perfect card: one robot mistake wipes it; originality can tip it anyway).
- **Chosen: option D with ½:** misses = blanks + ❌ + ½ × 🤔; multiplier = 1 + (secsLeft/90) × max(0, 1 − misses/2). Originality unchanged (outside). ⅓ was the other candidate; Oscar chose ½ "and let's see how that feels" (easy to soften later).
- Replays of all 114 dailies (24 Oct–1 Nov) with D: 79 scores unchanged (perfect cards, players who used the whole clock); with ½, the winner changes on 26 Oct FR (Giocoso 6 ✅ 850 beats David 5 ✅ 19 s left) and 31 Oct FR (Giocoso 800 beats Venirse primos 5 ✅ 45 s left, 780); ⅓ vs ½ differs on only 3 of 27 boards. Best possible score unchanged: 1200 + 300 originality = 1500 (realistic top ~1300; best seen Cocoricó 25 Oct 1160).
- Build plan approved (details in the handoff Backlog).

### Process notes
- The conversation got long (9 Firebase exports read in full); Oscar asked to build D in a fresh session.
- Correction made in the conversation: with ½, Venirse primos 31 Oct FR is 780, not 790 as first stated.

---

## Session 32 (3 Nov) — Production v261103.385 → v261103.388

### Start
- Backup `index_backup_s32.html`. Staging v384-tmp = production v385 apart from the version string.
- Oscar confirmed the first real multiplayer games on v383+ went fine: 🤖 verdicts for guests, ✕ / 👎 / reactions on every row, names with capitals.

### ½ vs ⅓ (explained to Oscar, then Claude's view)
- Oscar remembered ½ as "almost impossible for 1 miss to beat 0 misses". Corrected: with ½, one miss can still win if the perfect card used almost the whole clock. Figures (6 categories, no originality): best one-miss score = 1000 today / 833 with ⅓ / 750 with ½; a perfect card is safe from it with ≥ 45 s / 35 s / 22.5 s left; two misses: 800 / 533 / 400.
- When skipping one category pays (seconds it must save): perfect card finishing at 0 s → 18 s today / 27 s ⅓ / 36 s ½; finishing with 30 s left → 24 s / 51 s / never.
- Claude's view: keep ½ (skipping only pays when truly stuck); the one point for ⅓ is 🤔 (the robot's doubt costs ¼ of the bonus), softened by ¿Error? and the automatic second look. Review after a week of ½ (reminder in the handoff Backlog). Switching later needs a date gate like `DAILY_SPEED_V2_FROM`, because scores are recounted from scratch (changing `DAILY_MISS_CUT` alone would rewrite past days on screen).

### v261103.386-tmp: daily speed bonus, option D with ½ (staging)
- Helpers before `NEWS`: `DAILY_SPEED_V2_FROM = '2026-11-04'`, `DAILY_MISS_CUT = 0.5`, `dailySpeedV2(key, practice)`, `dailyMisses(cats, answers, aiResults)` (blank / ❌ = 1, 🤔 or unjudged = ½), `dailySpeedMult(speedRaw, misses, v2)`, `dailyRecount()` (scratch recount from `G_daily`), `dailySpeedNote(rc)`, `dailySpeedTextHTML(originality)`, `dailyAltoText(blanks, total)`, `dailyUpdateAltoBtn()`.
- New field `speedRaw` (secsLeft / totalSecs, 4 decimals) in the Firebase entry and localStorage. Entries without it (before v386, or written by an old cached client) keep their stored `speedMultiplier`. `speedMultiplier` now stores the reduced value, so the leaderboard ⚡× tag, originality recount and share text need no change.
- `dailyRecount` is used in submit, `showDailyResult` (new 7th parameter `speedRaw`; the 3 reopen paths pass it), ¿Error? and `dailyAutoRejudge` (which now also writes `speedMultiplier`). An overturn brings the bonus back.
- ¡Alto! button: "¡Alto! (2 en blanco) →" / "(2 blank)" / "(2 vides)" while typing (via `dailyCheckInput`, also after resuming an in-progress daily and when re-enabled after the empty-card toast); plain "¡Alto! →" with none or all blank.
- Result line: "base N pts ×M⚡ [+O originalidad✨] (bonus −50%: 1 fallo)" (EN "miss/misses", FR "erreur" with nbsp), same xs line, nowrap note. Help: new pts row "Cada fallo (en blanco o ❌) −½ bonus" (EN/FR).
- News: `from` field added to news entries (`activeNews` filter). New `2026-11-04-speed` (from the start date, until 2026-11-07). All expired news removed (lobby, robot-fast, historial); only `2026-10-31-follow` (until 4 Nov) remains.
- Staging start date kept at 4 Nov so a staging test can't change the real leaderboard of 3 Nov (same Firebase); Práctica uses the new rule at once.
- Tests: NEW `suites/test_build6_speed_bonus.mjs` (36: reference values 800 / 780 / 670, ×2 / ×1.5 / ×1.75 / ×1, old day, Práctica, legacy entry, ¿Error? and second look restoring the bonus, notes ES/EN/FR, button, news filter); NEW `restore/speed_scenarios.py` (21: button counts blanks while typing, 1 ❌ + 1 blank = ×1.00 with note, `speedRaw` saved in both places, ¿Error? → ×1.49 back, reopening, a day before the start keeps the full bonus, Práctica cut). Full kit ALL GREEN. Delivered with `tests_kit.zip`.

### 🤔 analysis (Oscar's idea: drop 🤔, keep only ✅/❌ once the robot is good enough)
- Dailies 24 Oct–1 Nov (exports uploaded by Oscar): 610 answers judged, 23 🤔 (3.8%).
  - Robot failures: 10, on two bad days only (26 Oct Cocoricó 5 + Giocoso 3; 29 Oct Cocoricó 2): ¿Error? overturned 9, upheld 1. The v384 automatic second look should now fix these without ¿Error?.
  - Model DUDOSO: 13 (~1.4/day): overturned 7, upheld 6. Every 🤔 was contested (23/23): an upheld 🤔 keeps its 50 pts, so ¿Error? on a 🤔 has no downside.
  - The DUDOSO cases are the genuinely borderline ones (Roberto, Jabato, Fiesta, Admira, Ben thorman, IG, Matelier, Jolie, Chi, Patatas a la importancia, Poto, Paulin, Radar), and ¿Error? didn't settle them well (Poto kept 🤔; Roberto and Radar overturned).
- Conclusion (Claude): keep 🤔 for now; forcing yes/no turns a safe 50 into a coin flip on exactly the answers the robot gets wrong most, and a wrong ❌ now costs a full miss on the bonus. Revisit after further judge-prompt work if the scout gets Poto and Radar right. Whatever happens, keep a "couldn't check" state for robot failures.

### v261103.387-tmp: daily category renamed (staging)
- Oscar: "Superhéroe o villano" → "hero and villain". Claude pointed out that the robot only sees the category name, so plain "Héroe o villano" would also take real people (national heroes, dictators); Oscar chose to say "fiction" in the name (players see the rule; no prompt change, no scout).
- Final names: ES **Héroe o villano de ficción**, EN **Fictional hero or villain**, FR **Héros ou vilain de fiction** (Claude suggested "méchant"; Oscar prefers "vilain").
- Renamed in place in `DAILY_CAT_DATA` (same position and letters), so the daily picks don't change: 90 days × 3 languages compared old vs new, 270/270 identical; the category appears on 28 of them. Letters not widened (that would change future picks).
- 3 Nov ES (letter A) already has the category: players before the deploy keep the old name on their saved rows; leaderboard and originality match answers by slot, so nothing mixes.
- Screenshot at 320 px with real fonts: wraps to 2 lines on the play screen and 3 on the result card, no clipping.

### v261103.388: PRODUCTION (3 Nov) = staging v387-tmp
- Diff vs v387-tmp: version string only. Version grep before (v387-tmp ×2, v385 ×2) and after (v388 ×2). Checklist: 12 screens, init block once, no `-tmp` in the version (the 3 regex `-tmp` remain), `node --check` clean, full kit on the production file ALL GREEN. ~429 KB raw, ~118 KB gzipped.
- The speed bonus starts 2026-11-04 (= the day after the production deploy); news 4–7 Nov.

---

## Session 32, continued (4 Nov) — Production v261103.388 → v261104.395

### Talks (no build)
- Workflow: Projects and GitHub. Claude's advice: GitHub is the bigger win (no zip upload, Claude pushes staging; rollback = `git revert` + push, or the Revert button on a merged PR). A Project only saves pasting the rules; keep the handoff in the repo, not as a project file. If GitHub is connected: commit `tests/` and `tools/` (no secrets), never `firebase_backups/`. Not set up yet.
- Long chats: wrap after each production deploy, before big builds and after heavy data analysis; the handoff is the only reliable carry-over between chats.
- Ideas to make the game more fun (Claude's top 3: 🔥 streaks, weekly league among followed players, themed days; kids mode as the bigger bet). Oscar picked themed days, tied to dates.
- Dates checked (web): Monet died 5 Dec 1926 (centenary, the strongest hook; Disney born 5 Dec 1901, Mozart died 5 Dec 1791), Agatha Christie disappeared 3 Dec 1926, Thanksgiving 26 Nov 2026, Children's Day 20 Nov, Santa Cecilia 22 Nov. Avoid sad or political anniversaries (Franco 20 Nov, JFK 22 Nov, Lennon 8 Dec); frame deaths as homages.

### v389–v394 (staging): first themed day, Santa Cecilia (Sunday 22 Nov)
- `DAILY_OVERRIDES['2026-11-22']`: letter B; ES Instrumento, Cantante o banda famosa, Compositor, Baile o danza, Canción, Género musical; EN Instrument, Singer or famous band, Composer, Dance, Song, Music genre; FR Instrument, Chanteur ou groupe célèbre, Compositeur, Danse, Chanson, Genre musical. Titles "🎵 Santa Cecilia: día de la música" / "🎵 St Cecilia's Day: music" / "🎵 Sainte-Cécile : la musique". Oscar: a title goes by its first letter as written (The Beatles = T, La Bamba = L).
- The override mechanism existed but was never used before. 90 days × 3 languages compared with production: only the 3 language-days of 22 Nov change.
- News: `2026-11-20-cecilia` (20–21 Nov, "El domingo es Santa Cecilia…") and `2026-11-22-cecilia` (22 Nov, "Hoy es Santa Cecilia…").
- Preview: `?dia=YYYY-MM-DD` (`DAILY_PREVIEW_DAY`, `dailyPreviewKey()`) plays that day, only for test names starting with `__` (which never save); the date header follows it. Ignored for normal names and malformed dates.
- Rain (`rain: [emojis]` in the override entry): iterated with Oscar over v389–v394.
  - v389: a 4 s shower on top on home (once per app open) and on the result. Oscar saw the notes sitting static at the top edge before falling (they started at top −20px with a random delay).
  - v391: home = sparse and continuous, on a new `#theme-bg-stage` (fixed, z-index −1) behind the content, while the home screen shows; result = short burst on top. Rain only lives on home and the daily result (`show()` stops it elsewhere); started at app open when home is showing (init line after `tryRestore()`).
  - v392: three layers per note: `.theme-rain` falls linearly (`themeFall`) > `.tr-sway` sways ±amp with ease-in-out alternate (sine-like, `themeSway`) > `.tr-tilt` tilts ±10° on the same wave (`themeTilt`). Random phase per note (negative delay, already moving), swing 1.25–2 s, width 8–20px.
  - v393: 20% of home notes in front (on `#confetti-stage`), fully solid; the rest behind at opacity 0.6.
  - v394: depth: front notes 21–24px solid, behind 14–20px faint (max 24px as before); tilt lags 2/8 of a cycle behind the sway.
  - Never while playing; none with "reduce motion"; Práctica never rains.
- v390: the themed-day result title uses the primary slot (26px) instead of hero 32px (Oscar chose A over "Reto diario" + subtitle B).
- Tests: NEW `suites/test_build7_themed_day.mjs` (21) and `restore/themed_scenarios.py` (35). `clock_scenarios.py` C3 fixed: it expected the old speed bonus and failed from 4 Nov on production too (date bomb). It now checks `speedRaw` (secsLeft/90). Scout v5.9: +13 cases for Canción/Song/Chanson and Género musical (La Bamba / The Boxer / La Bohème = ❌); Oscar to run it once on the hotspot.
- Oscar asked for economy with tool use: for small changes run only the targeted checks; the full kit runs once, before a production promotion.

### v261104.395: PRODUCTION (4 Nov) = staging v394-tmp
- Diff vs v394-tmp: version string only. Checklist: 12 screens, init block once, no `-tmp` in the version, `node --check` clean, full kit on the production file. ~437 KB raw, ~121 KB gzipped.

### Scout v5.9 result (Oscar, 4 Nov, after the v395 deploy; no code change)
- ling-3.0-flash: 58/63 (ES 24/27 · EN 13/15 · FR 21/21), 1 unsure, 7 flips, ¿Error? 28/34, avg 1.9 s, 0 timeouts.
- Santa Cecilia cases: 11/13. Misses: EN Song "The Boxer" → 🤔 (expected ❌, starts with T) and EN Music genre "Banjo" → ✅ (expected ❌, an instrument). Both favour the player, English only. Decision (Claude, not contested): no prompt change before 22 Nov (it would affect every day and need a new scout); look at the day's exports afterwards.
- Other misses are the known ones (Vallas salto de, Barca, MG; ¿Error? Tomate seco en polvo, Library, Esterilla, Deutsch, Orlando hat, Radar).

## Session 33 (12–13 Nov) — Production v261104.395 → v261113.397

### Start
- Handoff read in full; backup `index_backup_s33.html`. Production v395 = staging v394-tmp (same code), 12 screens, Santa Cecilia (22 Nov) in place.

### Talk: new daily categories (PARKED by Oscar, "flag and hold")
- Simulation of the next 365 Spanish dailies with the current pool (58 categories, 13 groups, max 1 per group, 6 per day): each group appears on ~45% of days, so categories in small groups repeat most: Parte del cuerpo 85×/year, Marca de coche 83× (marcas has only 3), Película 65, Mamífero 58, Tipo de edificio 57; least: Monumento 14, Mueble 16.
- Conclusion: to make the daily more surprising, add categories to the SMALL groups (marcas, ciencia, ciudad_urbanismo, deporte, entretenimiento).
- Proposed 18 (ES / EN / FR): Marca de comida o bebida, Cadena de tiendas o restaurantes; Elemento químico, Fenómeno meteorológico, Piedra preciosa o mineral; Parte de una casa; Equipo de fútbol; Juego de mesa o videojuego; Río, Isla; Raza de perro; Queso, Salsa; Canción, Género musical (already made for Santa Cecilia); Dios o diosa mitológica; Juguete; Nacionalidad. Others listed: Juego infantil, Montaña o volcán, Libro famoso, Fiesta o festividad, Electrodoméstico, Joya o complemento, Emoción o sentimiento. Optional wildcard group "sorpresa" ("Algo que llevas a la playa", "Algo redondo"…), more 🤔 expected.
- Build design (not built): a `from:'YYYY-MM-DD'` field per new category so the daily only uses it from that date (otherwise a mid-day deploy changes today's categories; past days are safe because scores store their categories); Práctica and multiplayer "Categorías" get them at once (they join the existing group pills); hand-picked letter strings per language; scout cases; news line; tests that every day before the date is unchanged.
- From the week's data: Invento / Invention caused the most disputes (Electricidad, Escoba, Escurridor, Électricité, Élections) — rename or clarify it when categories are picked up again.

### Dailies 4–12 Nov (9 exports, analysed in the sandbox)
- 107 dailies (ES 52, EN 25, FR 30), ~12/day, 12 names; new: Ally Burnett (EN, 9 Nov), Flo (FR, 12 Nov); Giocoso plays unregistered.
- 642 answers: 463 ✅, 98 ❌, 76 blank, 5 🤔 (0.8%, was 3.8% 24 Oct–1 Nov). No robot errors, so the automatic second look never ran (`rejudged` 0); 3 retries, 1 cut (FR "Eloïse", 34 s → 🤔). Median reply 1.9 s, 90% within 3.8 s; 12 Nov slower (9–11 s replies).
- Speed bonus ½: 54 of 107 finished early; of those 18 full bonus, 24 partial, 12 cut to zero. Neither trigger for ⅓ seen. **Oscar (12 Nov): nobody complains, keep ½.** Review closed.
- ¿Error?: 93 contests, 19 overturns. Right: Tarantino, Volante, Topacio, Barón Rojo, Rúcula, Encornets, Tauro, Escurridor (borderline). Wrong: Verano Azul (a TV series as Película), Brutalista (a style as Tipo de edificio), Barco velero (Equipamiento deportivo), Bibliotech (Type of shop), Ballena jurásica and Birloche (made up), Odisea (EN Film), Rats. Debatable: Escoba and Électricité as inventions.
- Same word, same day, opposite ¿Error? verdicts: Électricité (FR 11 Nov: Giocoso overturned, Mathieu upheld) and Escurridor (ES 12 Nov). Temperature is already 0 → the two Ling providers.
- First-pass misses: ¡Viven! (Película) and Bruja Avería (Héroe o villano) rejected; Bentley and BMW accepted as Marca tecnológica. Fiction category otherwise fine (Batman ✅ ×3, Blackman ❌).
- Correction made in the conversation: Claude first called England (ES País) and Nederland (FR Pays européen) rule breaks; both prompts accept country and city names in any language on purpose. **Oscar (12 Nov): keep accepting them.**
- 10 Nov ES Oscar "Bruce Springsteen" ✅ with a contestLog word "Bruce Springteen" upheld: Oscar fixed it by hand in Firebase.

### v261112.396-tmp (staging): stricter ¿Error? + one provider
- ¿Error? (contest method B system prompt) +2 rules after "an extinct animal must no longer exist": "It must really exist under that name: a description made up to fit (a common word plus an adjective, e.g. ${madeEx}) is INVALIDO; an obscure but real name is fine. It must be exactly the kind the category names, not a close neighbour: a TV series is not a film, a style is not a type of building, a boat is not sports equipment, a library is not a shop." `madeEx` per language ("ballena jurásica" / "jurassic whale" / "baleine jurassique"). First-pass daily prompt unchanged.
- `AI_PROVIDER` sent as OpenRouter's `provider` in every `aiJudgeCall` (first: `{ order: ['novita'], allow_fallbacks: true }`); `aiJudgeCall` returns `provider` (= `data.provider`, ≤ 40 chars); saved as `p` in `aiInfo` (daily and automatic second look) and in `contestLog`.
- Console check (Oscar, 13 Nov): the worker passes `provider` through (`deepinfra → DeepInfra`, `novita → Novita`; replies null only because the test used 50 tokens).
- Scout v6.0: same prompt, `PROVIDER` setting, provider count in the log and table, +22 real cases (19 overturns incl. England / Nederland ✅ and Trucha plateada ✅, plus Viven ✅, Bruja Avería ✅, Bentley as Marca tecnológica ❌).
- Tests: `test_judge_prompt.mjs` 41 → 45; `robot_scenarios.py` 24 → 27 (provider in every request, `aiInfo.p`, `contestLog.p`; fake robot replies with `provider`).

### Scout v6.0 provider runs (Oscar, hotspot)
| | Novita (12 Nov) | DeepInfra (13 Nov) |
|---|---|---|
| Judge | 52/66 (same 63 as v5.9: 50) | 55/66 (55/63) |
| New daily cases (Viven, Bruja Avería, Bentley) | 2/3 | 0/3 |
| ¿Error? | 42/53 (old 34: 27, new 19: 15) | 40/53 (old 34: 25, new 19: 15) |
| Avg / slowest 10% | 3.1 s / 5.4 s | 1.8 s / 3.2 s |
| Unsure / flips | 3 / 8 | 2 / 7 |
- New rules work on both: Verano Azul, Bibliotech, Ballena jurásica, Birloche, Électricité ❌ on both; real answers (Tarantino, Volante, Topacio, Encornets, England, Nederland) still overturned. Novita still overturned Barco velero, Rats, Escoba and upheld Barón Rojo; DeepInfra still overturned Brutalista, Odisea, Rats and upheld Trucha plateada.
- Accuracy about a tie; DeepInfra half the latency. The unpinned v5.9 run (4 Nov, 1.9 s, 58/63) matches DeepInfra, so production had mostly been on DeepInfra already. **Chosen: DeepInfra first, Novita as backup.**
- Santa Cecilia cases on DeepInfra: La Bamba, La Bohème, The Boxer ✅ (all favour the player): left as decided on 4 Nov, look at the 22 Nov exports.

### v261113.397: PRODUCTION (13 Nov)
- = v396-tmp with `AI_PROVIDER = { order: ['deepinfra'], allow_fallbacks: true }` (comment with both scout runs). Scout `PROVIDER` the same; `robot_scenarios.py` checks `deepinfra`. Oscar asked to go straight to production (the DeepInfra scout run measured exactly this setup).
- Checked: version grep before (v396-tmp ×2) and after (v397 ×2), 12 screens, init block unchanged, 3 regex `-tmp` only, `node --check` clean, full kit on the production file.


## START HERE (Session 33). Supersedes earlier START HERE sections
1. Upload the project zip (with `tests\` and `tools\`), or clone the repo if GitHub is connected. Read this whole file. First thing: `cp index.html index_backup_s33.html`.
2. **Versions:** production = **v261104.395** (4 Nov) = staging `index_tmp.html` v261104.394-tmp (same code). Live since then: the speed bonus option D ½ (from 4 Nov), the category "Héroe o villano de ficción", and the **first themed day: Santa Cecilia on Sunday 22 Nov** (letter B, music, emoji rain on home and result; news 20–22 Nov).
3. **Scout v5.9 done (4 Nov):** 11/13 on the Santa Cecilia cases; misses EN "The Boxer" → 🤔 and EN genre "Banjo" → ✅ (both favour the player). Decided: no prompt change before the day. Preview: `https://jugarenfamilia.es/?dia=2026-11-22` with a `__` name.
4. **After 22 Nov:** ask Oscar how the themed day went, and look at its exports (answers in the two new categories). Then plan the next themed days: Monet centenary on Sat 5 Dec (art), Christmas. Ideas list in Backlog → Product ideas.
5. **Ask Oscar** how the speed bonus feels (review ½ → ⅓ after a week); watch `speedRaw` in the exports, the fiction category, and `aiInfo.rejudged`.
6. News: `2026-10-31-follow` and `2026-11-04-speed` (until 7 Nov) can be removed when `NEWS` is next edited after 7 Nov; the Cecilia entries after 22 Nov.
7. Economy (Oscar, 4 Nov): small changes get only the targeted checks; the full kit runs once, before a production promotion. Don't re-run long suites that aren't affected.
8. At the end of the session: append the session's log to the history file, and edit this file so it describes the new current state.

### Session 33, continued (14 Nov): models, security design report (no code change)
- Talk about models: Opus 5.5 for planning, analysis and robot/scoring/Firebase builds; Sonnet 5.5 for small mechanical edits; Fable 5.2 (one free session) best spent on designing the security build.
- Prompt written for a separate Fable session: design only, one report as a doc, token-saving reading rules (handoff in full, no history file, grep index.html for Firebase calls instead of reading it whole, no tests/tools, one-shot report, open questions at the end). Cookie banner and AI step 1 left out. Oscar attached the zip, the Firebase rules, the Worker code and a `/names` export.
- Report: https://claude.ai/artifact/LiEbfZ3G6LQ8dXfvtx9EBS ("JugarEnFamilia - Security Build Design"). Summary and Claude's review are in the handoff (Backlog → Security).
- Found by the report: the OpenRouter key is a constant in the Worker code, not a Worker secret; the code was shared in that session. Claude advised rotating the key and moving it to a secret. The handoff line "the key lives only in the worker" was corrected.
- Claude's review: sound design, but big for the game's size (~5 sessions, ~4 weeks), rules complex and untested, 14-day enrolment window is an open door; disagreed with releasing dormant claims after 90 days; suggested a cheaper first step (regular exports + rules that block whole-node deletes/overwrites).
- **Oscar chose C (14 Nov): record the report, decide later.**
