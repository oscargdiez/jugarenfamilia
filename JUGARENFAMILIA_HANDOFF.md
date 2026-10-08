# JugarEnFamilia.es — Project Handoff Document
*Last updated: September 2026 — Session 26*

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
- **(Session 26)** Judge prompt changed in v320 (language rule). Any future prompt change: update `tools/ai_scout.js` too — `test_judge_prompt.mjs` fails if they differ.
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

**Last versions: production v261008.347, staging v261007.343-tmp (older).**

---

## ▶ START HERE NEXT SESSION (Session 28)

1. Upload the project zip (with `tests\` and `tools\`). Read this handoff. Production = **v261008.347** (Session 27: shared AI judge call, 2000 tokens, retry on empty, `aiInfo`; news "Robots mejorados"; ¿Error? ⏳ state; tagline removed + new Google description). Staging = v261001.332-tmp. New staging builds start from production v347 (v347: timeout retry skips Cohere; v346: every ¿Error? logged in the daily score `contestLog`; v345: daily seed per date+language from 10 Oct; v340–344: daily once per name, groups UI + sync by registered name + no duplicate names, stuck-round fixes, join-screen logo; v334–339: news, ¿Error? ⏳, tagline removed + Google description, share/page title, daily judged 3 at a time + timeout retry + timing in `aiInfo`). **First thing: check `aiInfo.s` / `timeout` / `error` in a daily export — still slow? still many timeouts? Then decide on tokens (1200?) or reasoning effort low, and the quality step (scout re-run with Vivora, Tarragon, Robin, Broccoli).** **News messages expire: history 14 Oct, robots 16 Oct** — ask Oscar if he wants new ones.
2. Check a recent daily export: are the long reasoning replies gone? What does `aiInfo` show — which models judge, how often `retried` / `cut`, any Cohere empty replies without `cut`? Daily speed OK?
3. Ask Oscar: Firebase backup exported? Any problems in real games since v324? If all is well → he deletes `global/`, `groups/`, `groupNames/` + their three rules lines.
4. Tests: run only when a change needs them. Full kit: `./run_all.sh ../index.html` → 239 unit + 77 + 12 + 19 + 28 browser (run long browser ones one by one, 300 s sandbox limit).
5. Next candidates (plan with Oscar first): help pages (Session 26 draft + open decisions), room-HTML fix (Security item 2), security build + AI step 1, AI step 2. Small parked: home emoji grid cut at 320 px, `#qj-name` inline `oninput` (rule 8), hard-coded Spanish in `doSubmit`, English words slipping through in FR/ES (Robin, Broccoli), tomato/botanical rule, unexplained host-session loss.

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
