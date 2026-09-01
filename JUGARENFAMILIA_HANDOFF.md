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

Double-click `D:\09_ALTO\rollback.bat` and pick the commit hash.

---

## ⚠️ CRITICAL RULES FOR CLAUDE

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
assert len(c) < 250000
```

**Always start from the uploaded working file** — never from a local copy that may have drifted. Ask the user to upload the current index.html if unsure.

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

- OpenRouter free tier: 200 req/day cap, resets daily
- iOS Chrome sticky counter — not solved (flagged for layout redesign)
- Emoji picker emojis still broken on Windows (only lang buttons have SVG fix)

---

## 🗺 Flagged for Future Discussion

- **Fixed shell layout** — non-scrolling outer frame, scrollable content area. Solves timer visibility, keeps branding visible, helps mobile UX. Needs full design discussion — touches every screen.
- **Timer visibility on mobile** — related to fixed shell above
- **Help icon during game** — ❓ during playing/validation, shows quick rules overlay
- **Language as lobby setting** — currently global preference, should be per-game decision in lobby alongside rounds/timer/categories
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
