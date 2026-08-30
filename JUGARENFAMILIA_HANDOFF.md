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
| AI validation | OpenRouter API | Free tier, auto model routing |
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
- **Models used:** `openrouter/auto` with fallbacks: Qwen3, Nemotron, Mistral Small

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
- Answer submission, ¡Alto! button
- Accent-insensitive comparison (México = Mexico)
- Stop penalty (configurable, host sets)

### Validation Screen (host)
- 📖 Wikipedia lookup (host + guests, language-aware, did-you-mean suggestions, 4s timeout)
- 🤖 AI validation via OpenRouter (ask on contentious answers only)
  - Strictness: 🟢 Libre / 🟡 Normal / 🔴 Estricto
  - Responds in game language
  - AI results broadcast to all guests via Firebase
- 👍👎 Per-entry voting (all players, live counts)
- 😂🔥👏🤔😱💀 Per-entry emoji reactions (attached to specific answers, visible to all)
- ✕ Manual invalidation by host
- 🗳️ Democratic mode: majority vote auto-invalidates entries
- ← Revisar button on scores screen (undo, restores pre-round scores)
- AI results + dict results + invalidations all broadcast to guest waiting screen

### Waiting Screen (guests)
- Read-only view of all answers
- Wikipedia lookups (private, guest-initiated)
- Sees host's Wikipedia results, AI results, invalidations, votes in real time
- Per-entry emoji reactions
- ✏️ Pencil bounce animation while waiting

### Scoring & Leaderboard
- Wins-based leaderboard (not points)
- Group identity: group name + player name + emoji = unique ID
- Group leaderboard (private per group)
- Global leaderboard (worldwide, by wins)
- Scores screen with ← Revisar button

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

---

## 🚀 Deploy Workflow

### Standard deploy (one double-click!)
Claude always downloads these files to `C:\Users\User\Downloads\`:
- `jugarenfamilia.html` — the game (required)
- `gitinfo.txt` — commit message and git config (required)
- `preview.png` — WhatsApp OG image (optional, only when changed)
- `JUGARENFAMILIA_HANDOFF.md` — this doc (optional, when updated)

Then double-click `D:\09_ALTO\deploy.bat` — it:
1. Reads `gitinfo.txt` from Downloads (falls back to local copy)
2. Copies `jugarenfamilia.html` → `index.html`
3. Copies `preview.png` if present
4. Copies `JUGARENFAMILIA_HANDOFF.md` if present
5. Git commits with the message from `gitinfo.txt`
6. Pushes to GitHub → auto-deploys to jugarenfamilia.es in ~30 seconds
7. **Cleans up all downloaded files** from Downloads folder

### Files in D:\09_ALTO\
- `deploy.bat` — deploy script (never changes)
- `gitinfo.txt` — updated by Claude each session
- `index.html` — current live game
- `preview.png` — WhatsApp preview image
- `CNAME` — contains `jugarenfamilia.es`
- `JUGARENFAMILIA_HANDOFF.md` — this document

### gitinfo.txt format
```
REPO=https://github.com/oscargdiez/jugarenfamilia.git
BRANCH=main
NAME=oscargdiez
EMAIL=oscar.g.diez@gmail.com
MESSAGE=feat: description of what changed this session
```
Claude updates the MESSAGE line each session to describe what was built.

---

## 🗺 Roadmap

### 🔧 Medium term
1. Google login — proper identity across devices
2. WhatsApp deep link invites — direct notification to join
3. 🎵 Background soundtrack — seamless looping audio (acoustic guitar), auto-play muted, tap to unmute. Need MP3 from Pixabay first
4. 🔔 Sound effects — pencil scratch on submit, bell on ¡Alto!, fanfare on winner screen
5. Letter reveal animation
6. 🎮 Solo practice mode — play alone against clock, localStorage personal best
7. 🎭 End of game reaction stats — recap screen: most voted answer, funniest player (most 😂), most controversial (mixed votes), most 🔥 answer. Needs reaction data preserved across rounds.

### 🚀 Bigger features
8. Public rooms — join random game with strangers
9. Tournaments — scheduled games with brackets
10. Second game under jugarenfamilia.es
11. Custom theme packs — Harry Potter, Netflix, Football clubs…
12. Cloudflare Workers backend — hides API keys, private GitHub repo

### 💰 Business
13. Google Analytics
14. Privacy policy page
15. Google AdSense — ads between rounds
16. Premium tier — host pays (€2.99/mo or €19.99/yr). Unlocks: no ads, extra themes, saved categories, permanent rooms. AI validation is the killer feature.
17. App Store via Capacitor/Cordova
18. Social sharing — post score to Instagram stories

---

## 🐛 Known Issues / Watch List

- OpenRouter free tier: 200 requests/day — resets daily. Key capped at $0 spend.
- GitHub repo is public — API key visible in source. Mitigated by $0 spending cap.
- iOS Chrome sticky counter — using fixed topbar workaround, test each session
- `.es` domain DNS can be slow to propagate (up to 48h)

---

## 🔧 Pending Tech Tasks

- [ ] Set up auto-deploy script (.bat) — copies HTML from Downloads to project folder, git commits and pushes to GitHub automatically
- [ ] Move to private GitHub repo + Cloudflare Pages for security
- [ ] Cloudflare Workers proxy for API keys

---

## 📝 Deploy Script (TODO)

A Windows `.bat` file that:
1. Copies `jugarenfamilia.html` from `Downloads\` to the project folder
2. Renames it to `index.html`
3. Copies `preview.png` if present
4. `git add .`
5. `git commit -m "Update game"`
6. `git push`

This would make deploying a 1-second operation.
