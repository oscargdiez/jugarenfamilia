# JugarEnFamilia.es — Project Handoff Document
*Last updated: September 2026 — Session 14*

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
  - `daily/{date}/{lang}/players/{safeName}: true` — cross-device daily dedup index
  - `names/{safeName}/secret: hash` — SHA-256 hash of email or PIN for name claim
  - `names/{safeName}/displayName: string` — original display name with accents
- **Schema additions (Session 13):**
  - `daily/{date}/{lang}/scores/{playerKey}/safeName: string` — for emoji-independent matching
  - `daily/{date}/{lang}/scores/{playerKey}/verified: bool` — (TODO: not yet written) for unverified label

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
- Claude ALWAYS states the version number when deploying
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

**Mandatory pre-deploy checklist:**
- Version string updated ✅
- File size < 300KB ✅ (currently ~303KB due to debug infrastructure — watch for further growth)
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

### Alto Caller Penalty (current — to be replaced by Session 15 overhaul)
- Caller penalised -50pts if they have any invalid/empty answer
- Will be replaced by sliding bonus/penalty scale (see Flagged for Future)

### Validation Screen (host)
- 📖 Wikipedia lookup (host + guests, language-aware)
- 🤖 Robot validation via Cloudflare Worker → OpenRouter (fallback chain, 20s timeout)
- Robot result shows coloured verdict: green=válido, red=inválido, amber=no sé (all 3 langs)
- Automatic validation ON by default, Estricta by default
- 👎 Per-entry voting (democratic mode only — thumbs down only, answers valid by default)
- 👏😂😬 Per-entry emoji reactions (3 emojis: clap, laugh, grimace)
- 🛑 Stop caller banner: shows on host validate AND guest waiting screen
- 🗳️ Democratic mode: majority 👎 votes auto-invalidates. Valid by default. Min 3 players (guard TODO next session)
- ← Revisar: undo scoring — scores stay visible to guests during re-validation, recalculates cleanly from preRoundScores

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
- **Cross-device dedup** — Firebase `players/{safeName}` index prevents same name playing twice per day
- Daily button: orange tinted outlined style, compact date (`7 sep`), 19px font

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

### Languages
- 🇪🇸 ES 🇬🇧 EN 🇫🇷 FR
- Full UI + categories + themes + rules translated
- All AI/IA references replaced with Robot throughout all 3 languages
- **Rule for new features:** always add translations for all 3 languages immediately

### Debug Mode (overhauled Session 14)
- Type `__debug__` as player name → debug bar at bottom
- **📱 iPhone mode toggle** — constrains viewport to 390×844 phone frame with notch, all fixed elements scoped inside
- Screen buttons grouped into rows: MULTI / DAILY / OVERLAY
- **MULTI:** Home, Welcome, QuickJoin, Lobby Host, Lobby Guest, Playing, Playing+Stop, Validate, Validate (no AI), Validate Demo, Waiting, Waiting Demo, Scores Host, Scores Guest, Scores Last, Final Host, Final Guest, Leaderboard
- **DAILY:** Daily Play, Daily Result, Practice Play, Practice Result
- **OVERLAY:** Countdown, Claim Name, Help
- Zero Firebase calls — fully offline
- Test mode for daily: name starting with `__` (not `__debug__`) — skips Firebase + localStorage

---

## 🐛 Known Issues / Watch List

- OpenRouter free tier models rotate without warning — if Robot breaks, check openrouter.ai logs
- Session restore on same device/browser: host and guest share localStorage
- Font sizes: Caveat x-height smaller than Special Elite — visually looks different at same px
- iCloud Safari sync: iPhone + iPad share localStorage if Safari sync enabled — player may not be able to replay on second Apple device
- Daily challenge same-device replay block stores player name at submit time — pre-v184 plays fall back to checking all 3 lang keys; may not catch edge cases from old plays
- Update bubble on iOS appears slower than on Windows (Safari caching) — working correctly, just slower
- Unverified name label not yet implemented — old scores have no `verified` field (treat as verified when built)
- File size ~303KB — 3KB over 300KB soft limit due to debug infrastructure. Watch for further growth.
- Democratic mode minimum player guard not yet built — should block start with < 3 players (next session)
- Solo normal game (1 player) should behave like practice — not yet implemented (next session)

---

## 🗺 Flagged for Future

### Unverified Name Label (designed Session 13, not yet built)
- Write `verified: true/false` to score entry at submit
- If `entry.verified === false` → display as `Peter? 🎸` in muted grey on leaderboard
- If `entry.verified` is `undefined` (old entries) → treat as verified (no badge)
- Incentivises registration without blocking play

### Cancel Button on Daily Countdown
- 10s countdown before daily starts — add cancel/back button
- No penalty — game not started yet, no localStorage/Firebase written
- User returns to home screen cleanly

### Democratic Mode Minimum Players Guard (next session)
- Block starting game in democratic mode if fewer than 3 players in lobby
- Toast in all 3 langs explaining why
- Mode toggle stays available in lobby — only start button blocks

### Solo Normal Game = Practice (next session)
- If 1 player in lobby + normal mode → start button says "Empezar práctica solo"
- No Firebase history written, no leaderboard, treat as practice
- AI validation still works

### Multiplayer Scoring & History Overhaul (BIG FEATURE — Session 15+)

#### New scoring system (replaces current):
- Valid unique answer → 100pts
- Valid duplicate answer → 50pts
- Originality bonus (only you wrote it among all players that game) → +50pts flat
- Invalid → 0pts
- **Alto caller sliding bonus/penalty** (replaces flat -50pts penalty):
  - 0 invalid answers → +150pts
  - 1 invalid → +50pts
  - 2 invalid → 0pts
  - 3 invalid → -100pts
  - 4 invalid → -200pts
  - 5 invalid → -300pts
  - 6 invalid → -400pts
  - Calculated at `finishValidation` time (final state of invalidAnswers)
  - Only applies to the player who actually called Alto
  - Empty/short answers already blocked by 2-char guard

#### Scores/result UI overhaul (visually consistent with daily):
- Per-player expandable cards showing: answers per category, validity icon (✅❌🤔), originality badge (+50✨), emoji reactions received (👏×2 😂×1)
- Alto bonus/penalty clearly labelled on caller's card
- Round scores + cumulative total clearly separated
- Final screen: same cards, totals across all rounds, winner highlighted
- Share/reshare button consistent with daily share format
- Shown on round scores screen AND final screen

#### Game history:
- Firebase path: `history/{gameId}` with ~2 week retention (purge entries older than 14 days on game start)
- Per game stored: `groupName`, `hostName`, `createdAt`, `lang`, `mode`, `players` (name+emoji), per-round data (letter, categories, allAnswers, invalidAnswers, altoCallerId, altoBonuses), final scores
- Navigate previous games with ◀ ▶ (chronological, not by date)
- Each card shows: group name (or unlabelled if no group), date/time, mode, host badge, players+final scores
- Tap to expand full detail (round by round, answers, originality, Alto bonus)
- ~2 weeks retention
- Solo/practice games excluded
- Players who leave mid-game → null scores from that point
- All players can access history
- World leaderboard dropped entirely
- Group leaderboard dropped — group name shown on history card
- Per-player stats within a group accumulate over time (wins, avg score, originality rate, Alto success rate, most emoji reactions received)
- Favourites/friends filter — future feature, flagged

### Daily Originality Overhaul (separate future feature)
- Current: flat +50pts if unique. To be replaced with:
- **Two tiers:**
  - Truly unique (only you wrote it) → `round(50 × log10(playerCount))` pts — no cap, grows with scale
  - Rare (<5% of players wrote it) → flat +25pts
  - Common (≥5%) → no bonus
- Score updates live throughout the day as more players submit
- History captures final score from previous day (next day when browsing back)
- Friends filter = view layer only (fun insight), does not affect official score
- Formula stress test: 10 players=50pts, 100=100pts, 1000=150pts, 10000=200pts — logarithmic, no cap

### Automatic AI Multiplayer Mode
- Third validation mode: fully automatic Robot validation, no host review step
- Round ends → Robot validates all → scores shown

### iOS Layout Refactor
- Replace `position:fixed` shell with true fixed layout
- Eliminates iOS Safari keyboard viewport resize bug

### Special Categories Overhaul
- Current special themes (Música, Deportes etc) use old flat category lists
- Daily category system (58 cats, 13 groups) is much better quality
- Could power a "Random" multiplayer mode drawing from daily category set

### Other
- Language as lobby setting (currently global)
- Public rooms / Tournaments
- Background soundtrack + sound effects
- Letter reveal animation
- Favourites/friends list for filtering history and originality

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
**Practice Mode:** solo daily-style play, no Firebase/leaderboard, random categories, `initDaily()` extracted as shared function, full `resetGDaily()` state isolation.
**Name claim system:** orange/green/red pill on home screen, overlay with email/PIN, SHA-256 hash in Firebase, displayName restore after verify, fully translated ES/EN/FR.
**Cross-device daily dedup:** Firebase `players/{safeName}` index, name normalised (lowercase + accent-strip).
**Update bubble:** version polling, opacity CSS transition (iOS-safe), home-screen only guard.
**Home screen UX:** `🎲 Clásico. Crear sala →`, orange tinted daily button, `¡Hecho!` label, compact date, 19px font exception for iOS, Practice button same row as Daily.
**Name input:** auto-capitalise via `addEventListener` (inline oninput suppresses events — learned the hard way), restores from localStorage on page refresh.
**Daily submit guard:** blocks if zero answers filled.
**Bug fix:** inline `oninput` with value reassignment suppresses subsequent events on iOS/Windows — moved to `addEventListener`.
**Last version deployed: v260907.156**

### Session 13 (Sep 8-9)
**Reserved name Firebase identity:** Device 2 with a verified name now sees their result screen instead of a dead-end toast. `safeName` field written into every Firebase score entry. Score retrieved by `safeName` match (primary) or normalised name fallback.
**Historical daily leaderboard:** Prev/next date nav on result screen. Shows `DD/MM/YY` for past days, "hoy/today/aujourd'hui" for today. Originality/refresh suppressed when viewing history. Lang switch uses currently viewed date. `_lbDate` resets to today on result screen entry.
**Emoji independence:** Leaderboard row highlight and originality matching now use `safeName` first, name-trim fallback for old entries.
**Leaderboard label:** "clasificacion / leaderboard / classement" (removed "del dia").
**Share result:** "Compartir/Share/Partager" button on answers card. Copies formatted text: name, letter, date+time, rank, answers with validation icons, score summary. Works for daily and practice.
**Multiplayer display name restore:** Verified names auto-correct to accented form on `createRoom`/`joinRoom`.
**Version check before game start:** `reloadIfOutdated()` on all five entry points.
**No-name guard:** `startDailyChallenge` and `startPractice` block and toast if name empty.
**Same-device daily replay block:** inline warning banner below daily button for different name on same device.
**Cross-device played check:** `checkDailyPlayedCrossDevice()` on page init.
**Timer expiry with empty answers:** `dailySubmit(fromTimer)` bypasses fill guard.
**Double timer interval bug fixed:** `clearInterval` before all `setInterval` calls in resume paths.
**Last version deployed: v260907.195**

### Session 14 (Sep 12-13)
**Debug bar overhaul:** iPhone mode toggle (390×844 phone frame, notch, all fixed elements scoped inside), screen buttons reorganised into MULTI/DAILY/OVERLAY rows, added Validate (no AI), Validate Demo, Waiting Demo, Scores Guest, Scores Last, Final Guest, Daily Play, Daily Result, Practice Play, Practice Result, Countdown overlay, Claim Name overlay, Help overlay.
**Emoji reactions reduced:** 5 → 3 emojis (👏😂😬 — clap, laugh, grimace). Applies to normal and democratic mode.
**Democratic mode — thumbs down only:** Removed 👍 button. Answers valid by default, 👎 majority invalidates. Rules panel updated all 3 langs. Validation subtitle updated. Auto-invalidate logic simplified (no ups/majorityFor/isDraw). Orphan CSS removed.
**Democratic vote optimistic UI:** `voteEntry` now immediately updates DOM before Firebase echo — single click highlights + counts. `G_cachedVotes`, `G_cachedPlayers`, `G_cachedVoteMode` globals cache state for optimistic render.
**Alto guard:** `callStop` checks all `.ans-inp` inputs have `trim().length >= 2` before doing anything. Toast in all 3 langs. Timer not cleared on early return.
**Scores freeze on back-to-validation:** `finishValidation` now bases calculation on `room.preRoundScores` (not `room.scores`) — safe to recalculate multiple times without double-counting. `goBackToValidation` no longer writes scores to Firebase. Guest scores screen shows "el anfitrión está revisando de nuevo ✏️" banner when phase flips back to validate, scores frozen until phase returns to scores.
**New translation keys:** `toastStopFill`, `hostRevisingScores` — all 3 langs.
**Last version deployed: v260912.213-tmp (staging) — pending production promotion**
