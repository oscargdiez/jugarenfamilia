# JugarEnFamilia.es — Project Handoff Document
*Last updated: August 2026*

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
| Old Netlify (backup) | https://altoeljuego.netlify.app |

---

## 📁 Key Files

| File | Location | Notes |
|---|---|---|
| Main game | `/mnt/user-data/outputs/jugarenfamilia.html` | Deploy as `index.html` |
| OG preview image | `/mnt/user-data/outputs/preview.png` | Deploy alongside index.html |
| This document | `/mnt/user-data/outputs/JUGARENFAMILIA_HANDOFF.md` | |

---

## 🛠 Tech Stack

| Layer | Tech | Notes |
|---|---|---|
| Frontend | Single HTML file | No framework, vanilla JS |
| Realtime multiplayer | Firebase Realtime Database | europe-west1 |
| AI validation | OpenRouter API | Free tier, fallback model chain |
| Hosting | GitHub Pages | Unlimited deploys, free SSL |
| Domain | jugarenfamilia.es | Registered on Namecheap |
| DNS | Namecheap BasicDNS | Points to GitHub IPs |

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
- **Free limit:** 200 requests/day, resets daily
- **Models used:** `qwen/qwen3-8b:free` primary, fallback to `mistralai/mistral-small-3.2-24b-instruct:free`, then `meta-llama/llama-3.2-3b-instruct:free`

### GitHub
- **Repo:** github.com/oscargdiez/jugarenfamilia (public)
- **Account:** oscargdiez
- **Pages:** enabled, main branch, / root

### Namecheap DNS Records
```
A Record  @    185.199.108.153
A Record  @    185.199.109.153
A Record  @    185.199.110.153
A Record  @    185.199.111.153
CNAME     www  oscargdiez.github.io
```

---

## 🎨 Design System

| Element | Value |
|---|---|
| Background | `#f7f3e8` (cream paper) |
| Accent red | `#c0392b` |
| Ruled lines | `#b0c8e0` (blue, 0.3 opacity) |
| Font headings | Caveat (handwritten) |
| Font labels | Special Elite (typewriter) |
| Theme | Notebook paper aesthetic |

---

## ✅ Features Built

### Core Game
- Real-time multiplayer via Firebase (room codes, host/guest model)
- Session restore / welcome-back screen with emoji picker
- Letter selection (easy/full pool, language-aware per language)
- Fixed countdown timer (works on mobile)
- Correct remaining timer for players who rejoin mid-round
- Answer submission, ¡Alto! button
- Accent-insensitive comparison (México = Mexico)
- Stop penalty (configurable, host sets)

### Validation Screen (host)
- 📖 Wikipedia lookup (host + guests, language-aware, did-you-mean suggestions, 4s timeout)
- 🤖 AI validation via OpenRouter (ask on contentious answers only)
  - Strictness: 🟢 Libre / 🟡 Normal / 🔴 Estricto
  - Responds in game language
  - AI results broadcast to all guests via Firebase
  - Fallback model chain (qwen → mistral → llama) with 12s timeout
  - "Reintentar" retry button on timeout
- 👍👎 Per-entry voting (all players, live counts)
- 😂🔥👏🤔😱💀 Per-entry emoji reactions (attached to specific answers, visible to all)
- ✕ Manual invalidation by host
- 🗳️ Democratic mode: majority vote auto-invalidates entries (batched Firebase writes)
- 🛑 Stop caller banner: shows who called ¡Alto! at top of validation screen
- ← Revisar button on scores screen (undo, restores pre-round scores, clears all validation state)
- AI results + dict results + invalidations all broadcast to guest waiting screen

### Waiting Screen (guests)
- Read-only view of all answers (properly populated on rejoin)
- Wikipedia lookups (private, guest-initiated)
- Sees host's Wikipedia results, AI results, invalidations, votes in real time
- Per-entry emoji reactions
- ✏️ Pencil bounce animation while waiting

### Scoring & Leaderboard
- Wins-based leaderboard (not points)
- Group identity: group name + player name + emoji = unique ID
- Group leaderboard (private per group)
- Global leaderboard (worldwide, by wins)
- Scores screen with ← Revisar button (fully clears validation state in Firebase)

### UX/Polish
- 🎊 Continuous confetti on final screen (winner's emoji)
- Welcome back screen (returning players see name + emoji before rejoining)
- Emoji picker (index-based IDs, cross-browser highlight fix)
- Floating emoji reactions during validation (48px, 5s duration)
- Collapsible rules + help section (all 6 languages)
- Language flags top-right (auto-remembered in localStorage)
- WhatsApp share button (room link in game language)
- Favicon (notebook paper + red !)
- OG/Twitter meta tags (English, 6 locale alternates)
- Copyright footer (auto-updating year)
- Autocomplete suppression on mobile Chrome

### Categories & Themes
- 8 themes fully translated in all 6 languages
- Custom categories (checkbox picker)
- Free mode (host types own categories)
- Language-aware letter pools (easy/full per language)

### Settings (host lobby)
- Theme selector
- Easy/full letters toggle (language-aware)
- Rounds (3-12)
- Time per round (30-120s, default 90s)
- Stop penalty (0-200pts, default 50)
- AI strictness level (Libre/Normal/Estricto)
- Validation mode (host decides / democratic)
- Group name with browser memory + dropdown suggestions

### Languages
- 🇪🇸 Spanish, 🇬🇧 English, 🇫🇷 French, 🇩🇪 German, 🇧🇷 Portuguese, 🇮🇹 Italian
- Full UI + categories + themes + rules + help all translated
- Scores screen buttons fully translated (next round, end game, review)

### Debug Mode
- Type `__debug__` as player name → unlocks debug bar at bottom of screen
- 14 screen buttons: Home, Welcome, QuickJoin, Lobby Host, Lobby Guest, Playing, Playing+Stop, Validate, Validate Demo (democratic), Waiting, Scores, Scores Last, Final, Leaderboard
- All screens rendered with realistic fake data (3 players, letter M, duplicate answers, one invalid, votes pre-populated, fake AI result)
- Zero Firebase calls — fully offline debug
- Password: just type `__debug__` as name and tap "Crear sala"

---

## 🚀 Deploy Workflow

### Standard deploy (one double-click!)
Claude ALWAYS provides these files at the end of each session:
- `jugarenfamilia.html` — the game (required)
- `gitinfo.txt` — commit message and git config (required, always with HTML)
- `JUGARENFAMILIA_HANDOFF.md` — this doc (when updated)

**Steps:**
1. Download all files from Claude to `C:\Users\User\Downloads\`
2. Double-click `D:\09_ALTO\deploy.bat`
3. Live at jugarenfamilia.es in ~30 seconds ✅

### Version number
Claude always states the version number (e.g. `v260831.2145`) when deploying. Check the footer of the live site to confirm the deploy took effect.

### gitinfo.txt format
```
REPO=https://github.com/oscargdiez/jugarenfamilia.git
BRANCH=main
NAME=oscargdiez
EMAIL=oscar.g.diez@gmail.com
MESSAGE=feat: description of what changed this session
```

---

## ⏪ Rollback Workflow

If a deploy breaks something:
1. Double-click `D:\09_ALTO\rollback.bat`
2. Pick the hash of the last good version
3. Live in ~30 seconds ✅

Alternatively, if Claude provides a `jugarenfamilia_WORKING_BACKUP.html`:
1. Rename it to `jugarenfamilia.html`
2. Use the matching `gitinfo_RESTORE.txt` renamed to `gitinfo.txt`
3. Run `deploy.bat`

---

## 🐛 Known Issues / Watch List

- OpenRouter free tier: 200 requests/day — resets daily. Key capped at $0 spend.
- GitHub repo is public — API key visible in source. Mitigated by $0 spending cap.
- iOS Chrome sticky counter — using fixed topbar workaround, test each session
- `.es` domain DNS can be slow to propagate (up to 48h)
- Rooms auto-delete from Firebase 60s after game ends (leaderboard data in `global/` and `groups/` is preserved)

## ⚠️ Critical Deploy Rule
Claude MUST run `node --check` on the extracted module script AND verify `buildEmojiGrid()` + `tryRestore()` are present at the end of the module before outputting any file. Injecting code near the end of the module script has repeatedly wiped the init block, breaking the home screen silently.

---

## 🗺 Roadmap

### 🔧 Medium term
1. Google login — proper identity across devices
2. WhatsApp deep link invites — direct notification to join
3. 🎵 Background soundtrack — seamless looping audio (acoustic guitar), auto-play muted, tap to unmute
4. 🔔 Sound effects — pencil scratch on submit, bell on ¡Alto!, fanfare on winner screen
5. Letter reveal animation
6. 🎮 Solo practice mode — play alone against clock, localStorage personal best
7. 🎭 End of game reaction stats — most voted answer, funniest player, most controversial

### 🚀 Bigger features
8. Public rooms — join random game with strangers
9. Tournaments — scheduled games with brackets
10. Second game under jugarenfamilia.es
11. Custom theme packs — Harry Potter, Netflix, Football clubs…
12. Cloudflare Workers backend — hides API keys, private GitHub repo

### 💰 Business
13. Privacy policy page
14. Google AdSense — ads between rounds
15. Premium tier — host pays (€2.99/mo or €19.99/yr)
16. App Store via Capacitor/Cordova
17. Social sharing — post score to Instagram stories

---

## 📝 Session Log

### Session 1 (Aug 29)
- Initial build: multiplayer, Firebase, rooms, scoring, themes, 6 languages

### Session 2 (Aug 30)
- Quick join screen, guest lobby confirmation card
- Per-entry emoji reactions and votes for all players
- Private Wikipedia lookups, flying emoji animations
- Experimental features panel (pwd: monica8) with AI validation and democratic mode
- Guest language auto-switch, new book icon, Google Analytics
- Deploy/rollback scripts

### Session 3 (Aug 31)
- Full audit pass: 15 bugs fixed
- Room cleanup from Firebase (60s delay after game ends)
- Collision-safe room code generation
- Stop caller banner in validation screen
- AI fallback model chain + retry button on timeout
- Timer on rejoin uses elapsed time correctly
- Guest rejoin during validate shows full state
- Democratic mode batches Firebase writes
- Scores screen buttons fully translated
- Debug mode: type `__debug__` to inspect all 14 screens
- Fixed recurring init block wipe bug (node --check now mandatory before deploy)
