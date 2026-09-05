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

### Staging workflow (index_tmp.html)
- Claude produces `jugarenfamilia_tmp.html` + `gitinfo_tmp.txt`
- Drop both to `C:\Users\User\Downloads\`, run `D:\09_ALTO\deploy_tmp.bat`
- Test at `https://jugarenfamilia.es/index_tmp.html`
- When happy: promote staging to production by copying `index_tmp.html` → `index.html`, bump version (drop `-tmp`), deploy with `deploy.bat`
- Staging is ALWAYS based on latest production (`cp index.html index_tmp.html`) — never from old stale staging file
- Claude ALWAYS states the version number when deploying

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

**Version bump on every deploy** — update the footer version string.
Format: `vYYMMDD.NN` (e.g. `v260905.49`). Staging gets `-tmp` suffix.
State the version clearly when presenting deploy files.

**Check with user BEFORE building anything. Deploy immediately after building without asking.**

**Font size rule — STRICT:** Only use these sizes: `11px / 13px / 16px / 20px / 26px / 32px` plus intentional hero exceptions (17px for icon buttons, 24px for emoji/icon elements, 28px for stop button, 15px for logo tagline, 48/52/56/80/120px and clamp for hero displays). No new sizes without explicit justification.

**The home screen buttons and emojis break silently if ANY of these happen:**

### 1. Init calls wiped
`buildEmojiGrid()`, `applyLang()`, `tryRestore()` MUST be the last lines of the `<script type="module">` block.

### 2. T object corruption ← THE MOST COMMON CAUSE
Many strings appear TWICE in the file: once as a value in the T translation object, and once hardcoded in a JS function. When replacing hardcoded strings with `t()`, ALWAYS target the specific JS function context. **Never replace the bare string.**

### 3. JS syntax errors
A single syntax error in the `<script type="module">` block kills ALL buttons and emojis silently. ALWAYS run `node --check` before deploying.

### 4. Non-module scripts are safe
SVG flags and other non-module `<script>` tags in `<head>` are completely isolated.

**Mandatory pre-deploy checklist:**
- Version string updated ✅
- File size < 300KB ✅
- Init block present (`buildEmojiGrid`, `tryRestore`) ✅
- No `-tmp` in version string for production ✅

**Always start from the uploaded working file** — never from a local copy that may have drifted.
**First thing every session — make a backup:** `cp index_tmp.html index_tmp_backup_sN.html` before any edits.

---

## 🎨 Typography System

Two fonts, six slots. **Do not add new sizes outside these slots.**

| Slot | Font | Size | Use |
|---|---|---|---|
| xs | Special Elite | 11px | Muted metadata, version string, tiny labels |
| sm | Special Elite | 13px | Standard labels, scores, badges, secondary UI |
| md | Special Elite | 16px | Section headers, screen titles, name input |
| body | Caveat | 20px | Player names, secondary content, `.lbl` labels |
| primary | Caveat | 26px | Answers, gameplay text, answer inputs |
| hero | Caveat | 32px+ | Timer numbers, letter display, room code |

**Role assignment:**
- **Special Elite** → all UI chrome (labels, scores, buttons, navigation, metadata, category labels in leaderboard)
- **Caveat** → all player-generated content + category labels in playing/result screens

**Letter-spacing:** only on decorative all-caps elements (JUGARENFAMILIA.ES, SIN REGISTRO footer, MULTIJUGADOR label, stop button, room code). Default everywhere else.

**Note:** Caveat has a smaller x-height than Special Elite, so Caveat 20px looks visually smaller than Special Elite 16px at the same pixel size. This is expected and fine.

---

## ✅ Features Built

### Core Game
- Real-time multiplayer via Firebase (room codes, host/guest model)
- Session restore / welcome-back screen with emoji picker
- Collision-safe room code generation (`genUniqueCode()`)
- Room auto-deleted from Firebase 60s after game ends (leaderboard data preserved)
- Letter selection (easy pool, language-aware: EN keeps K, ES/FR drop it)
- Correct remaining timer for rejoiners (`roundStartTime` saved in Firebase)
- Answer submission, ¡Alto! button, stop penalty (configurable)
- Accent-insensitive duplicate detection (`normalize()`)

### Validation Screen (host)
- 📖 Wikipedia lookup (host + guests, language-aware, did-you-mean suggestions)
- 🤖 AI validation via Cloudflare Worker → OpenRouter (fallback chain, 20s timeout)
- Automatic validation ON by default, Estricta by default
- 👍👎 Per-entry voting with live counts
- 😂🔥👏 Per-entry emoji reactions
- 🛑 Stop caller banner at top of validation screen (now shows in real game, all modes)
- 🗳️ Democratic mode (majority vote auto-invalidates)
- ← Revisar: undo scoring

### Daily Challenge
- Letter + 6 categories picked deterministically from date seed
- 90s countdown timer with speed bonus (1.0–2.0× multiplier)
- AI consensus validation (2 models parallel, tiebreaker if split)
- Scoring: valid=10pts, unsure=5pts, invalid=0pts (displayed ×10 in UI: 100/50/0)
- Speed breakdown shown: `⚡×1.34 velocidad | base 100 pts → 134 pts`
- In-progress save/restore (page refresh safe)
- Categories saved to Firebase on submit (immune to category changes breaking old results)
- Daily leaderboard with lang switcher (ES/EN/FR flags)
- Originality bonus: +5pts (shown as +50) if answer unique among all players
- Accent-insensitive originality check (ratón = raton)
- Firebase stores `langs` per player for future use; `LANG_FLAGS` const preserved
- Letter validity sanity check on in-progress restore
- Stale in-progress keys from previous days cleaned up on fresh start
- Double-submit race guard (`_submitting` flag)
- PlayerKey has random suffix to prevent collision

### UX/Polish
- Fixed shell layout: logo (left), letter+round+timer (center), room+? (right)
- Logo click goes home on ALL screens: quickjoin→home, final→home, leaderboard→confirm+leave, daily-play/result→home, lobby→leaveGame
- Floating timer numbers (spawn from edges, accelerate as time runs out)
- SVG flags — identical on Windows, iOS, Android, Mac
- WhatsApp share button
- 720px max-width (was 560px) for desktop comfort
- Language flags removed from player chips/scoreboards (lang stored in Firebase for future use)
- "clasificación en tiempo real" label removed (leaderboard is manual-refresh, not real-time)

### Languages
- 🇪🇸 ES 🇬🇧 EN 🇫🇷 FR (Italian, German, Portuguese dropped in Session 8)
- Full UI + categories + themes + rules translated
- **Rule for new features:** always add translations for all 3 languages immediately

### Debug Mode
- Type `__debug__` as player name → debug bar at bottom
- 14 screen buttons with realistic fake data
- Zero Firebase calls — fully offline

---

## 🐛 Known Issues / Watch List

- OpenRouter free tier models rotate without warning — if AI breaks, check openrouter.ai logs
- Daily challenge: "clasificación en tiempo real" label removed but refresh is still manual
- Font sizes: Caveat x-height smaller than Special Elite — visually looks different at same px

---

## 🗺 Flagged for Future

### Player Identity System (designed, not built)
- **Concept:** name is unique (first-come-first-served), email is proof of ownership, emoji is cosmetic
- **Flow:** enter name → if taken, prompted for email to verify ownership → email stored in Firebase as hidden key → on new device, enter name+email to restore identity
- **No email sending needed** — trust-based, family game stakes are low
- **Why email not device ID:** travels across devices naturally, truly unique
- **Implementation:** `playerKey` in Firebase switches from `name_timestamp_random` to email hash

### Category System Overhaul (PRIORITY for Session 10)
- 40 categories defined across 12 groups with valid-letter strings per language
- JSON table ready in chat "+++ ALTO CATEGORY TABLE"
- Picker needs: letter-aware (only show categories valid for today's letter in all 3 langs), group-aware (max 1 per group per day), 6 categories per day
- Current `DAILY_CATS` flat array to be replaced
- EN easy letters include K (ES/FR don't) — already in `LETTERS_EASY` object
- Always use easy letter pool (drop hard letters option)
- Category names in EN and FR still need translating before implementation

### Typography — Future Audit
- 25 distinct font sizes existed before Session 9 — reduced to 6-slot system
- Some elements still have non-standard sizes (17px icon buttons, 24px emoji, 28px stop button) — documented as intentional exceptions
- Full letter-spacing audit: some elements may still have leftover spacing from before the sweep
- `line-height` overrides removed globally — may need selective restoration if text feels cramped

### iOS Layout Refactor
- Replace `position:fixed` shell with true fixed layout using `html, body { height:100%; overflow:hidden }` + inner scrollable content div
- Eliminates iOS Safari keyboard viewport resize bug
- Big refactor — do in staging first

### Other
- Democratic mode: minimum 2 players guard
- Language as lobby setting (currently global)
- Emoji picker: SVG/Twemoji treatment for consistency across platforms
- `final` screen logo click: goes home cleanly (built Session 9)

---

## 🗺 Roadmap

1. Player identity system (email-based, no registration)
2. Category system overhaul (letter-aware + group-aware daily picker)
3. Google login
4. WhatsApp deep link invites
5. 🎵 Background soundtrack
6. 🔔 Sound effects
7. Letter reveal animation
8. 🎮 Solo practice mode
9. Public rooms / Tournaments
10. Cloudflare Workers backend (hides API keys, private repo)

---

## 📝 Session Log

### Session 1 (Aug 29)
Initial build: multiplayer, Firebase, rooms, scoring, themes, 6 languages

### Session 2 (Aug 30)
Quick join, guest lobby card, per-entry reactions/votes, Wikipedia lookups, flying emojis, experimental panel, AI validation, democratic mode, Google Analytics, deploy/rollback scripts

### Session 3 (Aug 31)
Full audit — 15 bugs fixed. Room cleanup, collision-safe codes, stop caller banner, AI fallback chain, timer on rejoin, democratic mode fixes, debug mode (14 screens)

### Session 4 (Sep 1)
AI button/results hidden when off, validation entry layout, debug controls, SVG flags, exhaustive i18n pass

### Session 5 (Sep 1)
AI validation fully fixed and tuned. Relajada/Estricta modes. Cloudflare Worker proxy. 3 languages (ES/EN/FR).

### Session 6 (Sep 2)
Fixed shell layout. Home screen refresh. Lobby redesign. Rules panel. Urgency pulse.

### Session 7 (Sep 3)
Validation screen layout fix. Countdown before each round. Daily Challenge promoted to production. Democratic mode overhaul. AI result sharing.

### Session 8 (Sep 4/8)
Floating timer numbers. Round timer bug fix. Validation row overhaul. Guest AI button. Category system (partial). Cloudflare Worker for OpenRouter key. Dropped IT/DE/PT.
**Last version: v260908.17**

### Session 9 (Sep 5)
**Daily challenge improvements:**
- AI icons (✅❌🤔) + pts per answer in leaderboard expanded panel (bug fix: was using wrong case)
- Speed multiplier ⚡×1.34 shown on leaderboard rows
- Daily scores displayed ×10 (100pts/answer) — Firebase unchanged, display-only
- Originality check fixed: accent-insensitive (`normalize()` instead of broken regex)
- Categories saved to Firebase on submit — immune to category changes breaking old results
- Lang switcher (ES/EN/FR SVG flags) on daily leaderboard — view any language's scores
- "clasificación en tiempo real" label removed
- Fixed leaderboard crash on malformed Firebase entries (missing `name` field)
- Fixed DAILY_LABELS reference before definition crash
- 4 daily challenge bugs fixed: unsure=5pts, letter sanity check on restore, delayed autosubmit (Firebase init), stale in-progress cleanup
- Double-submit race guard + playerKey collision prevention

**Multiplayer improvements:**
- Language flags removed from player chips and scoreboards (data kept in Firebase)
- Stop caller banner now shows in real game (was debug-only bug)
- Logo click works on all screens (quickjoin, final, leaderboard, daily-countdown)

**Typography overhaul:**
- 6-slot system: SE 11/13/16px + Caveat 20/26/32px
- Special Elite for all UI chrome; Caveat for content and category labels (except daily leaderboard expanded panel uses SE for categories)
- val-name (player name on validation screen) more muted (opacity 0.6)
- 720px max-width (was 560px)
- Letter-spacing removed globally except decorative all-caps elements
- Logo tagline stays Caveat 15px (intentional exception)
- Name input: Caveat 20px (intentional — Caveat x-height looks smaller than SE but is correct)

**Last version deployed: v260905.49**
