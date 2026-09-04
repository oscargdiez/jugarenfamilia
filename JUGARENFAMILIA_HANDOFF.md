# JugarEnFamilia.es — Project Handoff Document
*Last updated: September 2026*

---

## 🎮 Project Overview

A multiplayer browser-based version of the classic Spanish word game "Stop/Tutti Frutti/Alto el lápiz". Built as a single HTML file with Firebase for real-time multiplayer.

**Tagline:** "el juego de siempre, con los de siempre"

---

## 🌐 Live URLs

| Environment | URL |
|---|---|
| Production | https://jugarenfamilia.es |
| GitHub Pages | https://oscargdiez.github.io/jugarenfamilia/ |

---

## 🛠 Tech Stack

| Layer | Tech | Notes |
|---|---|---|
| Frontend | Single HTML file | No framework, vanilla JS |
| Realtime | Firebase Realtime Database | europe-west1 |
| AI validation | OpenRouter API | Free tier, fallback model chain |
| Hosting | GitHub Pages | Free SSL |
| Domain | jugarenfamilia.es | Namecheap |

---

## 🔑 Credentials & Config

### Firebase
- **Project:** `stop-9f0ea`
- **Database URL:** `https://stop-9f0ea-default-rtdb.europe-west1.firebasedatabase.app`
- **Config:** embedded in HTML (public, protected by Firebase rules)

### OpenRouter API
- **Key:** `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`
- **Key name:** Alto-Game
- **Spending cap:** $0 (safe, free tier only)
- **Free limit:** 200 requests/day
- **Models:** `qwen/qwen3-8b:free` → `mistralai/mistral-small-3.2-24b-instruct:free` → `meta-llama/llama-3.2-3b-instruct:free`

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

### Staging workflow (index_tmp.html)
- Claude produces `jugarenfamilia_tmp.html` + `gitinfo_tmp.txt`
- Drop both to `C:\Users\User\Downloads\`, run `D:\09_ALTO\deploy_tmp.bat`
- Test at `https://jugarenfamilia.es/index_tmp.html`
- When happy: Claude produces `jugarenfamilia.html` + `gitinfo.txt`, deploy with `deploy.bat`
- Staging and production are independent — staging never auto-promotes
- Claude ALWAYS states the version number when deploying (e.g. "deploying v260903.20-tmp")

1. Download both files to `C:\Users\User\Downloads\`
2. Double-click `D:\09_ALTO\deploy.bat`
3. Live at jugarenfamilia.es in ~30 seconds ✅

### gitinfo.txt format
```
REPO=https://github.com/oscargdiez/jugarenfamilia.git
BRANCH=main
NAME=oscargdiez
EMAIL=oscar.g.diez@gmail.com
MESSAGE=description of changes
```

---

## ⏪ Rollback

Double-click `D:\09_ALTO\rollback.bat` and pick the commit hash.

---

## ⚠️ CRITICAL RULES FOR CLAUDE

**Version bump on every deploy:** Always update the footer version string at line ~3202:
`<div style="font-size:9px;opacity:0.5">vYYMMDD.HHMM</div>`
State the version number clearly when presenting deploy files. This is how the user confirms the live site has refreshed.



**The home screen buttons and emojis break silently if ANY of these happen:**

### 1. Init calls wiped
`buildEmojiGrid()`, `applyLang()`, `tryRestore()` MUST be the last lines of the `<script type="module">` block. Any code injected near the end of the module can accidentally swallow them.

### 2. T object corruption ← THE MOST COMMON CAUSE
Many strings appear TWICE in the file: once as a value in the T translation object, and once hardcoded in a JS function. When replacing hardcoded strings with `t()`, ALWAYS target the specific JS function context (e.g. `getElementById(...).textContent = '...'`). **Never replace the bare string** — it will hit the T object value first, creating a circular reference (`t()` calls T before T is defined) that crashes the module silently.

Example of what went wrong:
```
# BAD — hits T object value first:
c.replace("'los mejores jugadores del mundo'", "t('worldSub')")

# GOOD — targets only the JS call:
c.replace("getElementById('lb-group-name').textContent = 'los mejores jugadores del mundo';",
          "getElementById('lb-group-name').textContent = t('worldSub');")
```

### 3. JS syntax errors
A single syntax error in the `<script type="module">` block kills ALL buttons and emojis silently. ALWAYS run `node --check` before deploying.

### 4. Non-module scripts are safe
SVG flags and other non-module `<script>` tags in `<head>` are completely isolated. They can't break the module.

**Mandatory pre-deploy checklist:**
```python
import re, subprocess

with open('jugarenfamilia.html', 'r', encoding='utf-8') as f:
    c = f.read()

# 1. Node syntax check
scripts = re.findall(r'<script type="module">(.*?)</script>', c, re.DOTALL)
clean = re.sub(r'import \{.*?\} from ".*?";\n', '', scripts[0])
with open('/tmp/chk.mjs', 'w', encoding='utf-8') as f2: f2.write(clean)
r = subprocess.run(['node', '--check', '/tmp/chk.mjs'], capture_output=True, timeout=10)
assert r.returncode == 0, r.stderr.decode()

# 2. Init block present
assert '// Init\nbuildEmojiGrid();' in c
assert 'tryRestore();' in c

# 3. File size sane
assert len(c) < 300000
```

**Always start from the uploaded working file** — never from a local copy that may have drifted. Ask the user to upload the current index.html if unsure.
**First thing every session — make a backup:** `cp index_tmp.html index_tmp_backup_sessionN.html` before any edits. If a bad edit corrupts the file, restore in one command. No exceptions.

---

## ✅ Features Built

### Core Game
- Real-time multiplayer via Firebase (room codes, host/guest model)
- Session restore / welcome-back screen with emoji picker
- Collision-safe room code generation (`genUniqueCode()`)
- Room auto-deleted from Firebase 60s after game ends (leaderboard data preserved)
- Letter selection (easy/full pool, language-aware)
- Correct remaining timer for rejoiners (`roundStartTime` saved in Firebase)
- Answer submission, ¡Alto! button, stop penalty (configurable)
- Accent-insensitive duplicate detection (`normalize()`)

### Validation Screen (host)
- 📖 Wikipedia lookup (host + guests, language-aware, did-you-mean suggestions)
- 🤖 AI validation via OpenRouter (fallback chain, 12s timeout, Reintentar on timeout)
- Automatic validation (AI) on by default, Estricta mode by default
- AI button/results hidden when AI is disabled
- 👍👎 Per-entry voting with live counts
- 😂🔥👏 Per-entry emoji reactions (visible to all, float shown to sender too)
- 🛑 Stop caller banner at top of validation screen
- 🗳️ Democratic mode (majority vote auto-invalidates, batched Firebase writes)
- ← Revisar: undo scoring, clears all validation state in Firebase

### Waiting Screen (guests)
- Read-only answers, properly populated on rejoin during validate phase
- Sees host's Wikipedia results, AI results, invalidations, votes in real time

### Scoring & Leaderboard
- Wins-based leaderboard (group + global)
- All leaderboard strings translated (worldSub, noGroup, noGroupLocal, wins etc.)

### UX/Polish
- Two-row validation entry layout: answer + buttons row / reactions + votes row
- Long answers wrap naturally; single long words hyphenate (`hyphens: auto`)
- Emoji picker: 32 emojis in clean 4×8 grid (removed 🦀🌵🍦🎪)
- No manual join box on home screen — join via link only
- 🌐 SVG flags for language buttons — identical on Windows, iOS, Android, Mac
- WhatsApp share button (room link in game language)

### Languages
- 🇪🇸 ES 🇬🇧 EN 🇫🇷 FR 🇩🇪 DE 🇧🇷 PT 🇮🇹 IT
- Full UI + categories + themes + rules translated
- `applyLang()` calls `applyQuickJoinLang()` on language switch
- All dynamic strings use `t()`: Wikipedia not-found, winner line, pts badge, invite toggle, experimental panel, AI panel, rules button, leaderboard strings, stop caller, timer ran out, etc.
- **Rule for new features:** always add translations for all 6 languages immediately

### Debug Mode
- Type `__debug__` as player name → debug bar at bottom
- 14 screen buttons with realistic fake data (3 players, letter M, duplicates, invalids)
- Controls: language switcher, AI on/off, long word test, democratic mode, refresh
- Zero Firebase calls — fully offline

---

## 🐛 Known Issues / Watch List

- OpenRouter free tier: 200 req/day cap, resets daily (50/day on unfunded account)
- Free models rotate out without warning — if AI breaks, check OpenRouter logs and swap models in the `models[]` array. Re-add `console.log('[AI full data]', JSON.stringify(data))` before the raw extraction line to debug. Max 3 models in the array (OpenRouter limit).

---

## 🗺 Flagged for Future Discussion

- ~~**Daily Challenge mode**~~ — BUILT in Session 7. See session log.
- **AI 3-step validation pipeline** — Step 1: letter check as model health test (if model fails "does X start with letter Y?" skip it). Step 2: category validation. Step 3: optional confidence. Self-healing model selection, much more accurate.
- **AI model config in Firebase** — store `models[]` array in Firebase so it can be updated without a redeploy. Part of future Cloudflare Workers backend work.

- ~~**Fixed shell layout**~~ — BUILT in Session 6. See session log.
- ~~**Timer visibility on mobile**~~ — resolved by fixed shell
- **Help/Rules icon during game** — now always accessible via ? button in shell header
- **iOS layout refactor — still open (Session 9+)** — replace `position:fixed` shell header with a true fixed layout using `html, body { height:100%; overflow:hidden }` + inner scrollable content div. This eliminates the iOS Safari keyboard viewport resize bug where the page jumps and the shell header shifts when the keyboard opens during gameplay.

  **Full spec:**
  - `html, body { height: 100%; overflow: hidden; margin: 0; }` — page never scrolls at body level
  - Shell header stays as-is visually but is part of a flex column layout instead of `position:fixed`
  - New wrapper: `#app { display: flex; flex-direction: column; height: 100%; }` containing shell header + `#content { flex: 1; overflow-y: auto; -webkit-overflow-scrolling: touch; }`
  - All `.screen` divs live inside `#content` — they scroll within the content div, not the page
  - Remove `padding-top: calc(2rem + 50px)` from `.page` (no longer needed to clear fixed header)
  - `overflow-y: scroll` on body (current approach) gets removed
  - Urgency overlay and countdown overlay: `position: absolute` within `#app` instead of `position: fixed` — or keep fixed but scope to the app div
  - Daily play and daily result screens: currently outside `.page` div — need to be brought inside `#content` with correct padding
  - **Risk areas:** shell header height calculation, timer visibility, urgency pulse overlay, countdown overlay, emoji picker positioning, any `position:sticky` elements (`.topbar` in playing screen)
  - Test on: iOS Safari (primary), iOS Chrome, Android Chrome, Windows Chrome
  - This is the biggest layout refactor of the project — do it in staging first, test every screen with debug mode before promoting

- **Windows / large screen — too small** — game renders as a narrow 560px column on wide monitors. Options: raise `max-width` to 720-800px for more breathing room, or add a responsive breakpoint above ~900px that goes multi-column (e.g. categories + answers side by side on playing screen, validation entries in 2 columns). Validation screen benefits most. Lower priority than iOS fix.

- **Language as lobby setting** — currently global preference, should be per-game decision in lobby alongside rounds/timer/categories
- **Democratic mode — minimum player guard** — with 1 player (host alone), majority=1 and host's single 👎 auto-invalidates, defeating the purpose. Should show a toast and block selecting democratic mode if only 1 player in lobby.
- **Emoji picker on Windows** — SVG treatment like flags, or Twemoji for everything

---

## 🗺 Roadmap

1. Google login
2. WhatsApp deep link invites
3. 🎵 Background soundtrack
4. 🔔 Sound effects
5. Letter reveal animation
6. 🎮 Solo practice mode
7. 🎭 End-of-game reaction stats
8. Public rooms / Tournaments
9. AdSense / Premium tier
10. Cloudflare Workers backend (hides API keys, private repo)

---

## 📝 Session Log

### Session 1 (Aug 29)
Initial build: multiplayer, Firebase, rooms, scoring, themes, 6 languages

### Session 2 (Aug 30)
Quick join, guest lobby card, per-entry reactions/votes, Wikipedia lookups, flying emojis, experimental panel, AI validation, democratic mode, Google Analytics, deploy/rollback scripts

### Session 3 (Aug 31)
Full audit — 15 bugs fixed. Room cleanup, collision-safe codes, stop caller banner, AI fallback chain, timer on rejoin, democratic mode fixes, debug mode (14 screens)


### Session 8 (Sep 8) — continued
**Validation row overhaul:**
- Emojis: trimmed to 5 — `👏 😂 😬 😱 🎯`, thumbs 👍👎 hidden in host mode (democratic only)
- SVG book icon replaced with 📖 emoji everywhere (saved ~1.4KB)
- Action buttons (📖 🤖 ✕) unified to fixed 32×32px squares, identical size
- ✕ button now red border and text
- Emoji centering fixed with inline-flex + equal padding
- Subtitle text: normal mode `📖 Wikipedia · 🤖 IA · ✕ anular`, democratic `👍👎 votar · 📖 Wikipedia · 🤖 IA`
- Fixed double 📖 bug in democratic mode subtitle

**Production deploys this session:** v260908.1–v260908.6

**v260908.4:** Guest AI button available in all modes (was democratic-only bug)
**v260908.5:** Guest AI results broadcast to host; guest restores to validation screen on refresh (showGuestValidation instead of refreshGuestValidation)
**v260908.6:** Guest refresh skips quickjoin when saved session matches URL room code — goes straight to welcome-back flow
**v260908.7:** quickJoinRoom now calls saveSession() + routes by room phase (not always enterLobby) — fixes guest rejoining mid-game going to wrong screen

**v260908.8–v260908.14 additions:**
- Daily leaderboard: tap row to expand answers accordion (with chevron indicator)
- Daily leaderboard: saves aiResults to Firebase on submit, shows ✅/❌ per answer
- Language flags (🇪🇸🇬🇧🇫🇷🇮🇹) shown on player chips, scores, final screen
- Daily categories overhauled: dropped Marca comercial/Animal marino/Pez/Insecto, added Nombre de persona/Medio de transporte/Prenda de ropa/Cosa en la cocina (then further refined)
- Dropped German (DE) and Portuguese (PT) — 4 languages remain: ES, EN, FR, IT — 14KB saved

**Next session — category system overhaul (PRIORITY):**
- Paste in the category table JSON from the prompt sent to another chat
- Implement letter-aware category picker: each category declares which letters it supports
- Group-aware picker: max one category per group per day (no Nombre de mujer + Nombre de hombre same day)
- Remove DE and PT from game category data (done) and from the new category table

**Category table prompt:** saved in handoff — send to a fresh Claude chat to generate the JSON table before next session.

**Session 8 full summary:**
- Floating timer numbers: spawn from left/right edges, accelerate 7s→1s as time runs out, single spawn at round start
- Round timer bug fixed: roundStartTime:null written in startGame/nextRound/playAgain + stale guard in enterPlaying
- Validation row: 5 emojis (👏😂😬😱🎯), thumbs democratic-only, 📖 replaces SVG icon, 32×32 action buttons, ✕ red
- Democratic mode: green/red/orange entry tinting, result badge icons only (✅❌🤝), result left of thumbs
- AI verdicts: funny messages in 6 languages ("¡El robot cree que es válido!")
- Guest AI button: now shows in all modes, results broadcast bidirectionally via Firebase
- Restore/refresh: guests now restore correctly to validation screen in all paths (tryRestore, continueSession, quickJoinRoom)

**v260908.3 additions:**
- Democratic mode entry tinting: green=valid majority, red=invalid, orange=draw (no border-left, matches red style)
- Vote buttons height 32px to match action buttons
- Democratic result badge: icon only (✅ ❌ 🤝), no text — fits on iOS
- Tie emoji changed to 🤝
- Result badge renders left of thumbs so thumbs stay anchored right
- AI funny verdicts: "¡El robot cree que es válido/inválido!" in all 6 languages
- Fixed double 90s spawn at round start (guard before clearInterval)
- Fixed democratic tinting bug (ups variable was undefined)

### Session 8 (Sep 8)
**Floating timer numbers:**
- Replaced the shell header timer (which disappeared on iOS when keyboard opens) with floating timer numbers that drift upward from the screen edges
- Uses `reactions-stage` (position:fixed) so immune to scroll/keyboard issues on iOS
- Spawns on left and right edges (5-30% and 70-95% of viewport width), avoids center
- Dynamic spawn rate: starts at 7s intervals, accelerates to 1s as time runs out
- Initial float spawns immediately at round start showing full time
- Pure linear animation (1.8s), no easing
- Turns urgent (larger) at ≤10s, matches urgency pulse

**Round timer bug fix:**
- Rounds 2+ sometimes started from a low number (e.g. 7s) due to stale `roundStartTime` from previous round in Firebase
- Fixed: `roundStartTime:null` now explicitly written in `nextRound`, `startGame`, and `playAgain`
- Added guard in `enterPlaying`: if elapsed >= maxSecs, treat as fresh start

**iOS layout investigation:**
- Attempted full refactor (`height:100%;overflow:hidden` + flex column + inner scroll container)
- Partially worked but iOS still misbehaved in edge cases; rolled back
- Floating timer numbers chosen as pragmatic solution — immune to the iOS keyboard bug

**Last version deployed: v260908.14**

### Session 7 (Sep 3)
**Validation screen layout fix:**
- Moved pts-badge + action buttons to bottom row — answer now has full width on iPhone
- Bottom row layout: [pts · emojis · thumbs] | [📖 · 🤖 · ✕] separator before action buttons

**Countdown before each round:**
- 10s full-screen overlay (big red number, cdPop bounce animation, Special Elite font)
- Shows "PREPARADOS / GET READY / PRÊTS…" label in all 6 languages
- Synced via Firebase `countdownStart` timestamp — rejoiners start from correct number
- Host drives transition: writes `phase:'playing'` at 0, guests enter simultaneously
- Applies to both multiplayer (startGame + nextRound) and daily challenge
- `roundStartTime` now set at `phase:'playing'`, not at countdown start — timer always starts at full configured time
- Daily inputs disabled during countdown, enabled + focused when timer starts

**Daily Challenge promoted to production:**
- Removed 4 console.logs from daily AI validation
- `__debug__` stripped from display name before Firebase save (bypass logic preserved)
- `dailyRulesHTML()` function added — ? button on daily screens now works (was throwing silent error)
- Daily rules panel shows only daily-relevant rules, not multiplayer helpHTML

**AI assistant renamed:**
- "Validación automática" → "🤖 Asistente IA" everywhere (33 occurrences, all 6 languages)
- Help text updated: "su opinión es orientativa — el anfitrión tiene la última palabra"
- Daily rules: "en el reto diario su decisión es definitiva" (accurate — no host override in daily)
- AI result now shows as "🤖 Válido / 🤖 Inválido / 🤖 Dudoso" (robot attribution)

**AI result sharing:**
- First player to press 🤖 triggers the API call — result broadcasts to Firebase
- Button disappears for all players when result arrives (via `applyAIResults`)
- `cls` stored in Firebase payload — CSS class detection no longer relies on emoji scanning
- AI results preserved on ← Revisar (were previously wiped)
- Pre-populated on render if result already exists in Firebase (rejoiners see result immediately)

**Democratic mode overhaul:**
- Promoted from password-protected experimental panel to main config section
- Experimental panel + `promptExperimental` + `promptAIPassword` + password logic fully removed
- ✕ cancel button hidden in democratic mode — host cannot manually override
- `toggleInvalid` hard-guarded against democratic mode
- Guests get 🤖 Asistente IA button in democratic mode (any player can trigger, one call shared)
- Vote restoration: if votes swing back below majority, entry is un-invalidated
- Democratic result labels localized in all 6 languages (were hardcoded Spanish)
- Tie badge only shown when ALL players have voted (not on incomplete split)
- `refreshGuestValidation` rebuilds on vote changes too (not just invalidAnswers)
- `finishValidation` re-reads vote majority from Firebase snapshot before scoring
- Duplicate vote render loop removed
- Dead `pct` variable removed from `renderVotes`

**Other:**
- 🤖 robot emoji restored everywhere (was changed to 🔍 in Session 5)
- "Puntuación" consistent everywhere (was "Puntajes" in some places)
- File size limit raised to 300KB in checklist (was 250KB)

**Flagged for future:**
- Democratic mode: minimum 2 players guard (1 player = host can self-invalidate)
- Tie display before all votes in (now shows nothing until all voted — may want "N/M votado" indicator)

**Last version deployed: v260908.14**

Additional changes in final deploys (v260908.1):
- Democratic mode moved above AI assistant in config section
- AI on/off toggle removed — AI assistant always on; `setAIEnabled` is now a no-op stub that forces true
- Validation screen subtitles update dynamically based on mode: democratic shows "📖 Wikipedia · 👍👎 votar" instead of "✕ anular"
- 🔍 magnifying glass restored on "Revisar respuestas" section title in rules panel (🤖 was wrong there)
- File size limit raised to 300KB in pre-deploy checklist

### Session 6 (Sep 2)
**Fixed shell layout — fully built:**
- Fixed top bar (50px, never changes height) with: ¡Alto! logo (left, locked in 80px grid column), letter badge + round info + timer (center), room pill + ? button (right)
- Logo tap: lobby → leaveGame() directly; playing/validate/waiting/scores → confirm dialog
- Room pill: non-interactive badge, only shows on playing/validate/waiting/scores
- ? button → full-screen Rules panel (rulesHTML + helpHTML, localized close button)
- Flags moved to shell header on home/welcome screens (left-aligned, absolutely positioned at content edge)
- Shell logo hidden (visibility:hidden, space reserved) on home/welcome so flags align cleanly
- CSS grid layout (80px 1fr 80px) for shell — logo can never shift regardless of center content
- overflow-y: scroll on body — scrollbar always reserved, prevents layout shift between screens

**Home screen refresh:**
- Crear sala → (clean, no emoji) + MULTIJUGADOR subtitle
- 📅 Reto diario · [date] + TÚ CONTRA EL MUNDO subtitle
- Date generated in locale-aware format for all 6 languages (English ordinals: 2nd, 3rd etc.)
- Reto diario button is a stub (toast "próximamente") — full implementation Session 7
- How to play dropdown removed from home (Rules panel covers it)

**Lobby redesign:**
- Reordered: share code → players → group name → ¡Comenzar! → ← Salir de la sala → ⚙️ Configuración ▼
- Configuration section is a collapsible dropdown (collapsed by default)
- ← Salir de la sala is now a proper styled button (not tiny muted link)
- Lobby logo removed (shell handles branding)
- Room pill hidden on lobby (code already visible in share card)

**Welcome-back screen:**
- Full ¡Alto! logo + tagline restored (same treatment as home)
- Flags visible, shell logo hidden

**Rules panel:**
- Full-screen overlay (not bottom sheet)
- Title: "Reglas del juego" / "How to play" etc. (not "Help/Ayuda")
- Contains: basic rules (how to play steps + scoring) + divider + AI/advanced features
- r3 updated: "pulsa ¡Alto! para terminar la ronda" (not "grita")
- Stop penalty warning added in all 6 languages
- Close button: plain "Cerrar" / "Close" etc.

**Other fixes:**
- Timer min-width + vertical-align so it never shifts the header layout
- Urgency pulse: subtle red tint animation at ≤10s (0.9s cycle, 8% opacity, clears on submit/stop)
- "Puntajes" → "Puntuación" everywhere in ES
- Welcome screen tagline now translates with language switch (was hardcoded ES)
- btn-create emoji/arrow preserved correctly in applyLang
- Staging workflow: deploy_tmp.bat + deploy2.bat added to D:\09_ALTO\

**Last version deployed: v260903.1**

### Session 5 (Sep 1)
- AI validation fully fixed and tuned across many iterations
- Model: `models[]` array with 3 verified working free models (ling-3.0-flash-fin, nemotron-3-super, glm-5.2) — max 3 per OpenRouter limit
- Stripped `<think>...</think>` blocks; reads `msg.reasoning` as fallback if `msg.content` is null
- Response parser: accent-normalised, regex-based, catches valido/invalido variants
- Removed old strictness levels (Libre/Normal/Estricto); replaced with Relajada/Estricta (2 modes)
- Added ✏️ Faltas de ortografía toggle (Sí/No, default No)
- Game language enforced in prompt — answers in wrong language = INVALIDO
- AI on/off moved out of experimental panel into main lobby (visible to host)
- Experimental panel now only contains democratic mode
- Generic system prompt + user prompt, works across all categories and special categories
- All new strings translated in 6 languages with correct gender agreement (relajada/estricta etc.)
- Emoji grid fixed on welcome-back screen (was not populating emoji-grid-wb)
- Stale "emoji picker broken on Windows" removed from known issues (was already fixed in session 4)
- Renamed "AI" to "Validación automática" across all UI, how to play, and translations
- How to play updated in all 6 languages: new Relajada/Estricta/spelling descriptions, old Libre/Normal/Estricto removed
- Automatic validation ON by default, Estricta by default
- Yes/No spelling toggle translated in all 6 languages
- Renamed 🤖 to 🔍 everywhere (26 occurrences)
- Confetti bug fixed: not stopping on guests' screens when host clicked Play Again (stopConfetti added to enterLobby)
- enterLobby now calls setAIEnabled/setAIStrictness/setAISpelling to reflect correct UI state on load
- Daily Challenge design finalised — ready to build next session (see Flagged for Future)
- Last version deployed: v260901.1427
- Daily Challenge design finalised — ready to build next session

### Session 4 (Sep 1)
- AI button/results hidden when off
- Validation entry layout: two-row, never overflows, wraps with hyphenation
- Debug controls: language, AI, long word, democratic, refresh
- Floating emoji visible to sender
- Emoji picker: 4×8 clean grid
- Removed manual join box from home screen
- Exhaustive i18n pass: all dynamic strings now use t() across all 6 languages
- SVG flags: identical on Windows and iOS ✅
- Documented and resolved recurring button breakage root causes (T object corruption, init block wipe)
