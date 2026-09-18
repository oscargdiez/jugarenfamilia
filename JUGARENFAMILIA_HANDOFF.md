# JugarEnFamilia.es — Project Handoff Document
*Last updated: September 2026 — Session 18*

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

### OpenRouter API
- **Key:** stored in Cloudflare Worker only — never in the HTML or GitHub repo
- **Key name:** Alto
- **Spending cap:** $4 (free tier only)
- **Models:** `inclusionai/ling-3.0-flash-fin:free` → `nvidia/nemotron-3-super-120b-a12b:free` → `z-ai/glm-5.2:free`
- **Logs:** visible at openrouter.ai → Logs → filter by API key "Alto"

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
- **IMPORTANT:** Version check bubble only works on production (fetches `/index.html`), not staging

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

**Always start from the uploaded working file** — never from a local copy that may have drifted.
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
- Room auto-deleted from Firebase 60s after game ends
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

### Validation Screen (host)
- 📖 Wikipedia lookup (host + guests, language-aware)
- 🤖 Robot validation via Cloudflare Worker → OpenRouter (fallback chain, 20s timeout)
- Robot result shows coloured verdict: green=válido, red=inválido, amber=no sé (all 3 langs)
- Automatic validation ON by default, Estricta by default
- 👎 Per-entry voting (democratic mode only — thumbs down only, answers valid by default)
- 🔥👏😂😬🤬 Per-entry emoji reactions (5 emojis, fire first)
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
- Session restore on same device/browser: host and guest share localStorage
- Font sizes: Caveat x-height smaller than Special Elite — visually looks different at same px
- iCloud Safari sync: iPhone + iPad share localStorage if Safari sync enabled — player may not be able to replay on second Apple device
- Update bubble on iOS appears slower than on Windows (Safari caching) — working correctly, just slower
- Unverified name label not yet implemented — dropped in favour of registration gate (see Flagged for Future)
- **File size ~310KB** — significantly over 300KB soft limit. Growing. Watch carefully each session.
- `daily/{date}/{lang}/players/{safeName}` legacy index still being written on submit — kept for backfill compatibility, can be retired once `names/{safeName}/played/` has been live for a few weeks
- Emoji picker grid overflows its container at non-100% zoom on Windows — needs flex-wrap and relative sizing (flagged for future session)

## 🗒 Small Fixes Backlog
- Quick join accent restoration — if pre-filled name matches a verified claim, fetch `displayName` from Firebase and restore accented version into field

---

## 🗺 Flagged for Future

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
