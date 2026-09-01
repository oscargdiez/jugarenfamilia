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

## 📁 Key Files

| File | Notes |
|---|---|
| `jugarenfamilia.html` | The game — deploy as `index.html` |
| `JUGARENFAMILIA_HANDOFF.md` | This doc |

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

### OpenRouter API
- **Key:** `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`
- **Spending cap:** $0 (safe, free tier only)
- **Models:** `qwen/qwen3-8b:free` primary, fallback to `mistralai/mistral-small-3.2-24b-instruct:free`, then `meta-llama/llama-3.2-3b-instruct:free`

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

Claude ALWAYS provides `jugarenfamilia.html` + `gitinfo.txt` together. Claude ALWAYS states the version number when deploying.

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

Double-click `D:\09_ALTO\rollback.bat` and pick the commit hash. Or if Claude provides a `jugarenfamilia_WORKING_BACKUP.html`, rename it and deploy.

---

## ⚠️ CRITICAL RULES FOR CLAUDE

**The home screen buttons break silently if ANY of these happen:**

1. **Init calls wiped** — `buildEmojiGrid()`, `applyLang()`, `tryRestore()` MUST be the last lines of the `<script type="module">` block. Any code injected near the end of the module can accidentally swallow them. ALWAYS check after every edit.

2. **T object corruption** — Many strings appear TWICE: once as a value in the T translation object, and once hardcoded in JS functions. When replacing hardcoded strings with `t()`, ALWAYS target the specific JS function context (e.g. `getElementById(...).textContent = '...'`), never the bare string. Replacing the T object value with `t()` creates a circular reference that crashes the module silently.

3. **JS syntax errors** — A single syntax error in the `<script type="module">` block kills ALL buttons and emojis silently (no visible error). ALWAYS run `node --check` before deploying.

4. **node --check is reliable** — Use it. False positives don't happen with proper module files.

**Mandatory pre-deploy checklist:**
```python
# 1. Node syntax check
scripts = re.findall(r'<script type="module">(.*?)</script>', c, re.DOTALL)
clean = re.sub(r'import \{.*?\} from ".*?";\n', '', scripts[0])
with open('/tmp/chk.mjs', 'w') as f: f.write(clean)
subprocess.run(['node', '--check', '/tmp/chk.mjs'])

# 2. Init block present
assert '// Init\nbuildEmojiGrid();' in c
assert 'tryRestore();\n</script>' in c or 'tryRestore();\n\n</script>' in c

# 3. File size sane
assert len(c) < 250000
```

**Always start from the uploaded working file** — never from a local copy that may have drifted.

---

## ✅ Features Built

### Core Game
- Real-time multiplayer via Firebase (room codes, host/guest)
- Session restore / welcome-back screen
- Collision-safe room code generation (`genUniqueCode()`)
- Room auto-deleted from Firebase 60s after game ends (leaderboard data preserved)
- Letter selection (easy/full pool, language-aware)
- Correct remaining timer for rejoiners (uses `roundStartTime` from Firebase)
- Answer submission, ¡Alto! button, stop penalty
- Accent-insensitive duplicate detection

### Validation Screen
- 📖 Wikipedia lookup (host + guests, language-aware, did-you-mean)
- 🤖 AI validation (OpenRouter, fallback chain, 12s timeout, Reintentar button)
- 👍👎 Per-entry voting with live counts
- 😂🔥👏 Per-entry emoji reactions
- 🛑 Stop caller banner at top of validation screen
- 🗳️ Democratic mode (majority vote auto-invalidates, batched Firebase writes)
- ← Revisar: undo scoring, restore pre-round state, clear all validation data

### Guest/Waiting Screen
- Read-only answers, properly populated on rejoin
- Sees host's Wikipedia results, AI results, invalidations, votes in real time
- Per-entry emoji reactions

### UX/Polish
- Floating emoji reactions visible to sender AND other players
- Emoji picker: 32 emojis in clean 4×8 grid
- Long answers wrap naturally; single long words hyphenate (`hyphens: auto`)
- Validation entries: two-row layout (answer row + reactions/votes row), never overflows
- AI button/results hidden when AI is off
- No manual join box on home screen (join via link only)
- WhatsApp share button

### Languages
- 🇪🇸 ES 🇬🇧 EN 🇫🇷 FR 🇩🇪 DE 🇧🇷 PT 🇮🇹 IT
- Full UI + categories + themes + rules translated
- `applyLang()` calls `applyQuickJoinLang()` so language switches update all screens
- Wikipedia not-found messages translated
- Leaderboard strings translated
- Experimental panel, AI strictness, validation mode labels translated
- Winner line, pts badge, invite toggle all use `t()`

### Debug Mode
- Type `__debug__` as player name → debug bar appears at bottom
- 14 screen buttons with realistic fake data
- Controls: language switcher, AI on/off, long word test, democratic mode, refresh
- Zero Firebase calls — fully offline

---

## 🐛 Known Issues / Watch List

- OpenRouter free tier: 200 req/day cap
- iOS Chrome sticky counter — not solved yet (flagged for layout redesign)
- Flag emojis don't show on Windows (flagged — SVG solution attempted but caused breakage)
- Language selection is global preference, not per-game (flagged for lobby setting)

---

## 🗺 Flagged for Future Discussion

- **Fixed shell layout** — non-scrolling outer frame with scrollable content, keeps branding visible, solves timer visibility on mobile. Needs full design discussion before touching.
- **Timer visibility on mobile** — related to fixed shell layout above
- **Help icon during game** — ❓ accessible during playing/validation, shows quick rules
- **Language as lobby setting** — currently global, should be per-game decision made in lobby
- **Flag SVGs on Windows** — need safe injection method that doesn't break module

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
10. Cloudflare Workers backend (hides API keys)

---

## 📝 Session Log

### Session 1 (Aug 29)
Initial build: multiplayer, Firebase, rooms, scoring, themes, 6 languages

### Session 2 (Aug 30)
Quick join, guest lobby card, per-entry reactions/votes, Wikipedia lookups, flying emojis, experimental panel, AI validation, democratic mode, Google Analytics, deploy/rollback scripts

### Session 3 (Aug 31)
Full audit — 15 bugs fixed. Room cleanup, collision-safe codes, stop caller banner, AI fallback chain, timer on rejoin, democratic mode fixes, debug mode (14 screens)

### Session 4 (Sep 1)
- AI button/results hidden when off
- Validation entry layout: two-row, never overflows, wraps with hyphenation
- Debug controls: language, AI, long word, democratic, refresh
- Floating emoji visible to sender
- Emoji picker: 4×8 clean grid (removed 🦀🌵🍦🎪)
- Removed manual join box from home screen
- Exhaustive i18n pass: experimental panel, AI panel, rules btn, guest confirmation, winner line, spinners, invite toggle, pts badge, Wikipedia not-found, leaderboard strings
- **Root cause of recurring button breakage documented above** — T object corruption + init block wipe
