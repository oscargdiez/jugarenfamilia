# JugarEnFamilia.es — Handoff (current state)
*Last updated: 12 November 2026 — end of Session 32*

> **Two files, two jobs.**
> - **This file = the CURRENT state only.** Read ALL of it at the start of every session. Each fact appears once and is kept up to date: when something changes, edit it here (replace the old statement) and log the change in the history file.
> - **`JUGARENFAMILIA_HISTORY.md` = everything that ever happened.** Append-only: never remove or rewrite anything there. It holds the full handoff as it was up to Session 30 and every session log since (session logs 1–32, every decision with its reasons, scout results, old START HERE sections). Search it when you need the *why* behind something.
> - `MULTIPLAYER_SCORING_DESIGN.md` = the design of multiplayer scoring, groups and history (Builds 1–5, all built). Reference only.
> - Nothing was deleted when this split was made (Session 30): the old handoff is in the history file word for word.

---

## ▶ START HERE (Session 33)
1. Upload the project zip (with `tests\` and `tools\`), or clone the repo if GitHub is connected. Read this whole file. First thing: `cp index.html index_backup_s33.html`.
2. **Versions:** production = **v261104.395** (4 Nov) = staging `index_tmp.html` v261104.394-tmp (same code). Live since then: the speed bonus option D ½ (from 4 Nov), the category "Héroe o villano de ficción", and the **first themed day: Santa Cecilia on Sunday 22 Nov** (letter B, music, emoji rain on home and result; news 20–22 Nov).
3. **Scout v5.9 done (4 Nov):** 11/13 on the Santa Cecilia cases; misses EN "The Boxer" → 🤔 and EN genre "Banjo" → ✅ (both favour the player). Decided: no prompt change before the day. Preview: `https://jugarenfamilia.es/?dia=2026-11-22` with a `__` name.
4. **After 22 Nov:** ask Oscar how the themed day went, and look at its exports (answers in the two new categories). Then plan the next themed days: Monet centenary on Sat 5 Dec (art), Christmas. Ideas list in Backlog → Product ideas.
5. **Ask Oscar** how the speed bonus feels (review ½ → ⅓ after a week); watch `speedRaw` in the exports, the fiction category, and `aiInfo.rejudged`.
6. News: `2026-10-31-follow` and `2026-11-04-speed` (until 7 Nov) can be removed when `NEWS` is next edited after 7 Nov; the Cecilia entries after 22 Nov.
7. Economy (Oscar, 4 Nov): small changes get only the targeted checks; the full kit runs once, before a production promotion. Don't re-run long suites that aren't affected.
8. At the end of the session: append the session's log to the history file, and edit this file so it describes the new current state.

---

## 🎮 Project
Multiplayer browser version of the Spanish word game "Stop / Tutti Frutti / Alto el lápiz", plus a solo daily challenge. One HTML file, vanilla JS, Firebase for real time. Three languages: ES, EN, FR. About 20–25 daily players (Oscar, 1 Nov 2026), mostly family and friends, also abroad.

| | |
|---|---|
| Production | https://jugarenfamilia.es (`index.html`) |
| Staging | https://jugarenfamilia.es/index_tmp.html |
| GitHub Pages | https://oscargdiez.github.io/jugarenfamilia/ — repo github.com/oscargdiez/jugarenfamilia (public), main branch, / root |
| Domain | jugarenfamilia.es (Namecheap): A records @ → 185.199.108–111.153, CNAME www → oscargdiez.github.io |
| Realtime | Firebase Realtime Database `stop-9f0ea`, europe-west1, `https://stop-9f0ea-default-rtdb.europe-west1.firebasedatabase.app`. Config is in the HTML (public). Free Spark plan: no automatic backups, 100 simultaneous connections. |
| Robot | Cloudflare Worker `https://api.oscar-g-diez.workers.dev/` → OpenRouter. The key lives only in the worker. |
| Analytics | Google Analytics `G-V2VJ4X7M5Z` (no consent banner yet, see Security) |

---

## 🚀 Deploy workflow
**Rules:** check with Oscar BEFORE building anything (plan → Oscar's OK → build, even when he suggested the change; mockups as plain text, or a quick screenshot before long test runs). Deploy right after building, without asking. Always state the version. Run tests only when a change needs them; use few tool calls.

- **Production:** Claude delivers `jugarenfamilia.html` + `gitinfo.txt` (+ `JUGARENFAMILIA_HANDOFF.md`, `JUGARENFAMILIA_HISTORY.md`, `MULTIPLAYER_SCORING_DESIGN.md` when changed). Oscar drops them in `C:\Users\User\Downloads\` and runs `D:\09_ALTO\deploy.bat` (copies `jugarenfamilia.html` → `index.html`, commits, pushes; live in ~30 s).
- **Staging:** Claude delivers `jugarenfamilia_tmp.html` + `gitinfo_tmp.txt` only (`deploy_tmp.bat` does not copy the docs). Test at `index_tmp.html`. Staging always starts from the latest production file. Several staging builds can stack and be promoted once.
- **Promotion:** copy `index_tmp.html` → `index.html`, new version without `-tmp`, checklist, kit, deliver as production.
- **Tools delivery:** both bat files also install from Downloads `ai_scout.js` → `D:\09_ALTO\tools\` and `tests_kit.zip` → `D:\09_ALTO\tests\` (zip must contain a top-level `tests/` with `run_all.sh`). Tools-only delivery skips GitHub. `tests/` and `tools/` are git-ignored.
- **Project zip for the next session:** Oscar runs `make_handoff.bat` → `Downloads\09_ALTO.zip` (whole folder except `.git`, `firebase_backups`, caches).
- **Rollback:** `D:\09_ALTO\rollback.bat`, pick the commit.
- **The bat files are replaced by hand** in `D:\09_ALTO\` (no script copies them). They must stay plain ASCII with CRLF line endings.
- **gitinfo.txt** (staging: `gitinfo_tmp.txt`, still `BRANCH=main` — there is no tmp branch):
  ```
  REPO=https://github.com/oscargdiez/jugarenfamilia.git
  BRANCH=main
  NAME=oscargdiez
  EMAIL=oscar.g.diez@gmail.com
  MESSAGE=vYYMMDD.NN - plain ASCII description (no emoji, no accents: they break deploy.bat)
  ```
- **Version:** `vYYMMDD.NN` (NN keeps counting up across days), staging adds `-tmp`. It appears twice in the file (the `RUNNING` constant and the footer). Grep it BEFORE and AFTER bumping. The 3 other `-tmp` strings in the file are the version-check regexes, on purpose.

### Pre-deploy checklist
- Version grep before and after ✅
- Init block present once, last lines of the module script: `buildEmojiGrid()`, `applyLang()`, `tryRestore()` ✅
- No `-tmp` in the version for production ✅
- 12 screens: `grep -c 'id="s-'` = 12 ✅
- JS syntax clean: extract the `<script type="module">` and `node --check` it ✅
- Test kit `./run_all.sh ../index_tmp.html` → ALL GREEN, when the change needs it ✅
- File size: no limit (Oscar). ~437 KB raw, ~121 KB gzipped as served.

---

## ⚠️ Critical rules for Claude
1. **Init calls** `buildEmojiGrid()`, `applyLang()`, `tryRestore()` must be the last lines of the module script, once.
2. **The `T` translation object breaks silently.** Many strings exist twice (in `T` and hardcoded in a function). Never replace a bare string: target the function context. Session 30 found `T` holding keys defined up to 6 times with leftover Portuguese/Italian values; it was cleaned with the parser. Now: exactly 140 keys in each of `es`/`en`/`fr`, no duplicates. Edit `T` with the parser (acorn) or by exact unique context, then check the counts.
3. **One syntax error in the module script kills every button.** Always `node --check`.
4. Non-module scripts in `<head>` (SVG flags) are isolated.
5. **`applyLang` / `s(id, text)` uses `textContent`** and wipes child spans. Use `innerHTML` (or target the child) for elements with children.
6. **Read enough context** (10+ lines) before structural edits; check the 12 screen ids after.
7. **SVG flags** (`window.FLAGS`) have `display:block` baked in. For inline use, swap to `display:inline-block;vertical-align:middle`. Flags never inline with text in rows (use a flex row).
8. **Never inline `oninput` with value reassignment** (suppresses events on iOS/Windows). Use `addEventListener('input', …)` with `setSelectionRange`. (No violations left since v380.)
9. **Claim pill visibility is CSS-only** (`#claim-pill` classes `free/claimed/reserved`). JS sets only `className`; `style.display` is used only to clear it.
10. **`finishValidation` scores from `room.preRoundScores`**, never `room.scores` (no double counting on Revisar). `preRoundScores` must be null between rounds (`nextRound`, `startGame`, `playAgain` clear it).
11. **`_playedLangs`** is filled async by `fetchPlayedState`; never treat `undefined` as "not played" without also checking localStorage `alto_daily_{date}_{lang}`.
12. **Robot calls go through the Cloudflare Worker**, never the Anthropic or OpenRouter API directly.
13. Entering validation clears `invalidAnswers` and `votes` atomically with `phase:'validate'`.
14. **`G_daily.aiResults` keys become strings** after a JSON round trip: write `String(idx)`, read both forms.
15. **Daily scores are always recounted from scratch** from `aiResults` (no deltas).
16. **Player text (names, answers, group names, free categories) goes into the page only through `escHtml()`, and into an `onclick` string only through `jsq()`.** (v379, after the audit's S1/S2/B1.)
17. **Firebase keys can't contain `. # $ [ ] /`.** Names and free categories are keys (`players/{name}`, `votes/{name__cat}`); v380 refuses them at input/start. Anything new used as a key must be checked the same way.
18. **Timers follow real time** (`startTimestamp`, `endAt`), never `secs--` (phones suspend pages). (v381)
19. **Every new top-level Firebase path needs a line in the rules** (see Firebase).
20. **New strings: always ES, EN and FR at once.** French uses a non-breaking space before `: ? ! ;` in visible text.

---

## 🎨 Typography and casing
Two fonts, six slots. No new sizes without a documented reason.

| Slot | Font | Size | Use |
|---|---|---|---|
| xs | Special Elite | 11px | metadata, version, tiny labels, notes |
| sm | Special Elite | 13px | labels, scores, badges, pills, secondary UI |
| md | Special Elite | 16px | headers, titles; 16px also for some inputs (iOS no-zoom) and mode cards under 360px |
| body | Caveat | 20px | names, `.lbl`, help steps, search box |
| primary | Caveat | 26px | answers, gameplay text, inputs |
| hero | Caveat | 32px+ | timer, letter, room code |

Documented exceptions: 19px daily/practice home buttons (17px emojis inside); 22px help titles `.help-h`; 17px icon buttons; 24px emoji elements; 28px stop button; hero displays 48/52/56/80/120px; floating timer numbers 36/42px; 10px in the debug bar only. Special Elite = UI chrome; Caveat = player content and category labels in play/result screens.
Casing: buttons Sentence case (`Crear sala`); hints/sublabels lowercase (`nombre del grupo`); toasts Sentence case; decorative badges ALL CAPS.

---

## 🧭 Architecture map (index.html, ~8,100 lines)
- `<head>`: meta (title/description "¡Alto! — Stop · Tutti Frutti · Petit Bac", Spanish description), GA, `<style>` (~900 lines), non-module script with `window.FLAGS` (SVG flags).
- Markup: 12 screens `#s-…` (home, quickjoin, welcome, lobby, playing, validate, waiting, scores, final, leaderboard (= group page), daily-play, daily-result) + overlays (claim, help, countdown) + shell header (logo · letter/round/timer · room pill/?).
- Module script, roughly in order: `T` translations (+`t(key)`) · categories (`CAT_GROUPS`, `DAILY_CAT_DATA` 58 categories / 13 groups with per-language letter strings, `THEMES_ALL`) · `applyLang` · groups (`alto_mygroups`, sync by verified name) · follow system · help (`HELP` drawn by `helpSectionsHTML()`) · `show()` screen switch · daily (setup/seed, play, timer, submit, AI judging, result, ¿Error?, leaderboard) · news · rooms (create, quickJoin, lobby, countdown, playing, ¡Alto!, validation, reactions/votes, scoring, scores, final, group page) · claim name + name gate + name characters · version check · debug mode · init.
- Key globals: `G` (room session), `G_daily`, `G_follow`, `G_gp` (group page), `LANG`, `_playedLangs`, `_claimState`.
- Debug: type `__debug__` as the name → debug bar; `dbg('<screen>')` draws any of 25 screens with sample data, offline. Names starting with `__` are test names: the daily doesn't save, the name gate skips them.

---

## 🔑 Firebase
### Rules (published Session 25; `global`/`groups`/`groupNames` lines removed 1 Nov, Session 31)
```json
{ "rules": {
    "rooms":{".read":true,".write":true}, "daily":{".read":true,".write":true}, "names":{".read":true,".write":true},
    "groupsMeta":{".read":true,".write":true,".indexOn":["lastPlayed"]},
    "groupsLb":{".read":true,".write":true}, "history":{".read":true,".write":true}, "historyIndex":{".read":true,".write":true} } }
```
Everything listed is open read/write (the security build fixes this). Any other root path is refused. `global/`, `groups/`, `groupNames/` were deleted by Oscar on 1 Nov (backup exported first). `daily/*/*/players` is no longer written since v382; it is still in the database (harmless, optional cleanup).
**Console tip:** big nodes open read-only. Open a small node by URL with `/` as `~2F`, e.g. `…/data/~2Fdaily~2F2026-09-30~2Fes~2Fscores`. Oscar should export a backup now and then (Realtime Database → ⋮ → Export JSON).

### Data (current)
- **`rooms/{code}`** (4 letters): `host`, `players/{name}: emoji`, `langs/{name}`, `scores/{name}`, `left/{name}: true`, `phase` (lobby|countdown|playing|validate|scores|final), `categories[]`, `catMode` (fixed|random|free-random), `selectedGroups`, `allFreeCats[]`, `rounds`, `timePerRound`, `currentRound`, `currentLetter`, `usedLetters`, `roundStartTime`, `calledStop`, `stopCaller` (`'Tiempo'` on timeout = never a caller), `allAnswers/{name}: {cat: answer} | {skipped:true}`, `invalidAnswers/{name__cat}`, `votes/{name__cat}/{voter}`, `entryReactions/{name__cat}/{reactor_emoji}`, `reactions/{id}` (floating), `aiResults/{name__cat}: {cls, key}` (v379; older rooms had `html`), `preRoundScores`, `roundLog/{round}: {letter, categories, caller, players/{name}: {answers[], status[] u/d/x/e, got[] {f,c,l}, pts {answers, alto, reactions, round, total}}}`, `practice`, `reactionPoints`, `validationMode`, `gameLang`, `groupId`, `groupName`, `lobbyInfo {mode, themeIdx, rounds, time}`. Deleted 45 min after the game ends (`G._deleteRoomTimeout`, cancelled by Jugar de nuevo).
- **`daily/{date}/{lang}/scores/{playerKey}`**: `name` ("Name emoji"), `safeName`, `reg` (registered + verified on that device), `baseScore`, `totalScore` (= base × speed, never includes originality), `originality`, `speedMultiplier` (the multiplier actually applied, after the miss cut), `speedRaw` (secsLeft/90, v386; missing on older entries), `answers{}`, `aiResults[]`, `aiResponses{idx}` (reply text for invalid/unsure, ≤1000 chars), `aiInfo[] {model, s, cut?, retried?, timeout?, error? ('timeout'|'http'), msg? (error text, v384), rejudged?, first?, firstMsg? (automatic second look, v384)}`, `categories[]`, `timestamp`, `contestLog/{idx} {word, category, letter, original, result, overturned, method, aiResponse, model, cut?, retried?, s, timestamp}` (older overturned entries: `overturned: 'valid'`), `revalidated` (29 Sep only). `playerKey` = name cleaned + time + random.
- **`names/{safeName}`** (safe = lowercase, accents stripped, non-alnum → `_`): `secret` (SHA-256 of email or 4-digit PIN, unsalted, readable — see Security), `displayName` (with accents), `played/{date}/{lang}: playerKey` (or `true` before Session 27; taken with a transaction = one daily per name), `contests/{date}/{lang}/{idx}: true` (one ¿Error? per answer; `{lang}_log` entries are old, no longer written), `groups/{id}: {name, last, def?}`, `groupsRemoved/{id}: time`, `following/{other}: {n, t}`, `followers/{other}: t`.
- **`groupsMeta/{groupId}`** `{name, createdAt, lastPlayed}` · **`groupsLb/{groupId}/{nameKey}`** `{name, lastEmoji, wins, games}` · **`history/{groupId}/{gameId}`** (full game: endedAt, lang, host, groupName, scores, players, roundLog, currentRound, rounds, reactionPoints) · **`historyIndex/{groupId}/{gameId}`** `{endedAt, winners[], lang}`. gameId = time-sortable base36; last 20 games kept per group; groups idle > 1 year deleted (3 per game end).
- **localStorage:** `alto_name`, `alto_last_name` (invite-link name), `alto_emoji`, `alto_lang`, `alto_session` (+ per-tab copy in sessionStorage; `?room=CODE` in the URL), `alto_claim_{safe}` `{verified}`, `alto_daily_{date}_{lang}` (played result), `alto_daily_inprogress_{date}_{lang}`, `alto_lobby_config` + `alto_lobby_config:{safe}` (per-name settings), `alto_mygroups` `[{id,name,last,def}]` max 10 (`MYGROUPS_KEY`; old `alto_groups` / `alto_last_group` are removed on load), `alto_lb_mode`, `alto_news_dismissed`. sessionStorage `alto_ver_checked` stops the auto-reload looping.

---

## 🤖 Robot (AI judge)
- **`AI_MODELS = ['inclusionai/ling-3.0-flash', 'cohere/north-mini-code:free']`** (v364), sent as OpenRouter's `models` fallback list (keep ≤ 3 entries). Ling 3.0 Flash is paid ($0.021 / $0.063 per M tokens, providers Novita + DeepInfra); Cohere free is the safety net if credit runs out (slow, 25–40 s). The OpenRouter key has a credit limit (Oscar). OpenRouter privacy settings: free endpoints that train = ON, publish prompts = OFF.
- **One judge call** `aiJudgeCall(systemPrompt, prompt, models, timeoutMs)` for daily, practice, ¿Error? and the multiplayer 🤖: `AI_MAX_TOKENS = 2000`, `AI_TIMEOUT_MS = 20000`. Verdict = the LAST VALIDO/INVALIDO/DUDOSO word of the answer (yes/no only for ≤3-word replies; reasoning used only if it ENDS with a verdict). Empty/cut answer → asked once more (same list); timeout → retried once with `models.slice(1)`; connection/HTTP error → retried once after 1.5 s (v384); still nothing → 🤔 dudoso, with the error text in `aiInfo.msg`. One-letter answers are rejected locally (`aiTooShort`, `AI_LOCAL_SHORT`). The daily judges 3 answers at a time (`AI_PARALLEL = 3`). **Automatic second look (v384, `dailyAutoRejudge`):** when the result screen opens, an answer that is 🤔 because of a robot failure (`aiInfo.error`, no `rejudged`) is judged once more; the score is recounted from scratch; still failing → stays 🤔 with ¿Error? offered; never loops; doesn't use up ¿Error?.
- **One judge prompt** `dailyBuildPrompt(category, letter, opts)` / `dailyBuildPromptWithWord(…)`. Rules: letter; category fit (people categories need a person, a spice must be a spice, extinct animals must be extinct, only people's names may be written surname first); (3) language: proper nouns are international, everything else must be the game language's word (`FOREIGN_WORD_EXAMPLE`); (4) spelling, including sound-alike letters (`spellEx` inside the function); ignore accents and capitals. v384 additions: (1) a word right apart from its accents IS the game-language word (`ACCENT_EXAMPLE`, e.g. violin = violín); (3) a common noun spelled the way another language spells it is INVALIDO even when close (`NEAR_FOREIGN_EXAMPLE`, e.g. FR Giraffe/Piton/Jazmin); (6) a first name alone is not a person unless famous under one name (Madonna, Shakira, Pelé), a famous surname alone is fine, first-name categories excepted. Multiplayer options: Estricta (default) / Relajada / allow spelling mistakes.
- **¿Error? (contest)** = `CONTEST_METHOD = 'B'`: ONE model (first available), a neutral "fresh eyes" prompt with strict spelling and no letter-fitting; since v384 it overturns only answers that CLEARLY fit (vague/partial answers stay INVALIDO: first name alone, model instead of brand, region instead of city, brand of another kind; doubt → INVALIDO), ignores accents and keeps near-foreign spellings INVALIDO; VALIDO overturns, INVALIDO/DUDOSO upholds; one per answer per day; uses the date and categories of the game played (v384 midnight fix). `DAILY_CONSENSUS = false` (if ever enabled: it still references `MODELS[2]`, fix first).
- **Any prompt change → update `tools/ai_scout.js`** (the kit's `test_judge_prompt.mjs` fails if they differ) and re-run the scout. Scout = paste into the Console on jugarenfamilia.es; **only on the phone hotspot** (Oscar's router restarted twice during scouting). Baseline **v5.8 on v384** (3 Nov): judge 42/50 (ES 16/22 · EN 11/11 · FR 15/17), re-checks 28/34, 4 flips, avg 1.9 s, 0 timeouts. Remaining misses: daily Tomate seco en polvo, Robin, "Vallas, salto de", Violin, Poto (🤔), Piton, Barca, MG; re-check Library, Esterilla, Deutsch, Orlando hat, Radar (overturned), Patatas a la importancia (upheld). (v5.7 on v367: 34/36, 20/23.)
- **Dead-model check** (Console on the site): `fetch('https://api.oscar-g-diez.workers.dev/',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({model:'<id>',messages:[{role:'user',content:'Di solo: VALIDO'}],max_tokens:50,temperature:0})}).then(r=>r.text()).then(console.log)`. A model with no endpoints is skipped silently by OpenRouter (logs show only the fallback).
- **Accepted on purpose (don't re-propose):** EN "Luxemburg" ❌ (Oscar: fine as is); tomato-as-fruit left to the host; Ling's verdicts can differ between providers; the stricter ¿Error? now keeps the obscure real dish "Patatas a la importancia" ❌ (price of stopping ~12 wrong overturns in 9 days).

---

## ✅ Features (current behaviour)
### Home
Name field (first letter capitalised, rest as typed; `. # $ [ ] /` can't be typed) with the claim pill (🟠 free / 🟢 mine / 🔴 reserved, 800 ms debounce), emoji grid (8 columns, fits 320 px), Clásico · Crear sala, Reto diario + Práctica (19px), news list, ❓ help, update bubble (red pill when a newer production version exists; on load the page auto-reloads once to a newer version).

### Name reservation (claim) and gate
- Reserve a name with an email or 4-digit PIN (hashed). Verified on a device → `alto_claim_{safe}.verified`. Verifying restores the accented display name.
- **Gate (v378):** Crear sala, invite-link join and Reto diario (incl. "¡Hecho! Ver puntuación") call `nameGateOk()`. A reserved name not verified on this device opens the claim panel in gate mode ("Verificar y jugar →" / "Usar otro nombre"); after a correct PIN the action continues by itself. Passes: verified device, free name, `__` test names, Firebase read error or > 5 s. Práctica, session restore and welcome back are not gated. Also stops taking over a host by typing their name.
- Reasons to reserve (texts): nobody else can use it, the daily counts on every device, groups follow you, follow other players.

### Multiplayer
- **Create / join:** host creates a room (4-letter code, collision-safe); guests join ONLY by invite link (`?room=CODE` → join screen). There is no join-by-code box. Joining with the host's exact name makes you host again (recovers a host who lost the session; reserved names are protected by the gate).
- **Lobby:** "¡comparte!" + big red code + [📋 Copiar] [WhatsApp] (native share sheet first). Players with 👑 on the host. Group picker (`🎮 name ▾` + 📜 history). Settings bar "⚙️ Configuración (Normal, Clásico, 5 rondas, 90s)" opening: validation mode (👑 Normal / 🗳️ Democrático, min. 3 players), theme (🎲 Clásico 8 fixed categories · ✏️ Categorías: 13 group pills, 8 random per round · ✍️ Libre: own list, "Usar ejemplos", "8 al azar" with 9+), rounds, seconds, 👏 reactions with points (Sí/No), 🤖 Robot (Estricta/Relajada, spelling). One pill style. Settings remembered per name on the device. Guests see the host's choices live. 1 player → "✏️ Practicar solo".
- **Round:** 10 s countdown (any client can start the round via a transaction), random letter (easy pool; EN keeps K), timer from `roundStartTime` (real time). ¡Alto! needs every answer ≥ 2 characters; the first ¡Alto! wins (transaction). Time up sends what's there.
- **Stuck rounds:** leave writes `left/{name}`; the waiting screen shows "Esperando a: …"; after 15 s the host gets "Seguir sin … →" (those players get blank answers).
- **Validation (host):** per answer 📖 Wikipedia, 🤖 Robot (opinion only), ✕/↩ anular. Democrático: answers valid by default, 👎 votes, majority invalidates. Reactions 🔥 👏 😂 😬, one per answer, none on your own. Guests see everything read-only plus votes/reactions. Revisar sends everyone back to validation with all decisions kept.
- **Scoring:** unique 100 · duplicate 50 · invalid/empty 0 · ¡Alto! caller `round10(250 × categories/8) − 100 × invalid` · reactions (if on) 🔥 +30 / 👏 +20 on valid, 😂 +10 on any, 😬 0. Ties share medals (dense) and wins.
- **Scores:** cards like the daily leaderboard (your own opens automatically), "base N pts (A ±N, reacciones +N)". **Final:** winner or tie line, awards 🙌 Popularidad / 😂 Risas / ✨ Originalidad, all rounds per player, Compartir (text with scores, awards and `jugarenfamilia.es/?g=ID`), Marcador, Jugar de nuevo (host).
- **Practice:** solo games and games ended before the last round don't count (no leaderboard, no history).
- **Groups and history:** each device keeps "Mis grupos" (max 10, default "Grupo de {host}", rename, × with confirm, no duplicate names); with a verified reserved name the list follows you across devices. Group page = `#s-leaderboard`: 🏆 Marcador (wins, games, %) and 📜 Historial (last 20 games, ◀ ▶ with flag + date, host badge, full game view). `?g=ID` opens a group's history on any phone.
- **Session restore:** per tab (sessionStorage + `?room=`); welcome back on reload; refresh-safe in every phase.

### Daily challenge
- One letter + 6 categories per day and language (`dailySeed` = FNV-1a of date+language since 10 Oct; max 1 category per group; `DAILY_OVERRIDES` for themed days). Each language is its own challenge. The date is the phone's local date (international game; changing the clock is accepted — decided, don't re-propose).
- 10 s countdown, 90 s clock from `startTimestamp` (keeps running in another app; time up while away = submit on return). In-progress answers survive a reload.
- Scoring: valid 100 · dudoso 50 · invalid/empty 0, × speed, + originality 50 per valid answer nobody else gave (recomputed for everyone when the board loads). Stored as 10 / 5 / 0 (and originality 5) and shown ×10: `totalScore = round(base × speed)`.
- **Speed bonus (v386, from `DAILY_SPEED_V2_FROM` = 2026-11-04; Práctica always):** misses = blanks + ❌ + ½ × 🤔; multiplier = 1 + (secsLeft/90) × max(0, 1 − misses × `DAILY_MISS_CUT`), `DAILY_MISS_CUT = 0.5`. Days before the start date keep 1 + secsLeft/90. `dailyRecount()` recounts base, misses and multiplier from scratch in submit, `showDailyResult`, ¿Error? and `dailyAutoRejudge` (an overturn brings the bonus back); entries without `speedRaw` keep their stored multiplier. ¡Alto! button shows "¡Alto! (2 en blanco) →" while typing (still needs ≥ 1 answer); the result line adds "(bonus −50%: 1 fallo)" when the bonus was cut; help has a "−½ bonus" row. Any change to the cut needs its own date gate (recounts would otherwise rewrite past days).
- One daily per name and language across devices (fresh check + transaction on `names/{safe}/played`). Reopening shows the saved result, read directly via the playerKey (`fetchMyDailyEntry`).
- ¿Error? on ❌/🤔 answers (the game just played, once per answer, ⏳ while checking). A daily finished just after midnight keeps its own day: result rows, leaderboard and ¿Error? use `G_daily.key` / `G_daily.categories` (v384). Answers left 🤔 by a robot failure get one automatic second look (see Robot).
- **Leaderboard:** dense medals (ties share, earlier time first), top 10 + you, ◀ ▶ past days, flags per language; today's answers hidden on tabs you haven't played. Follow system: ☆ follow / silver ★ following / gold ★ mutual on registered rows, [Todos] [★ Siguiendo] (followed players who played), search box, legend; unregistered players see "Registra tu nombre…". Share result text.
- **Práctica:** same flow, random letter, 5 s countdown, unlimited, nothing saved.

### Other
- **Help (❓):** `HELP = {home, daily, multi}[lang]` items (`h`, `i`, `s`, `pts`, `note`, `hr`) drawn by `helpSectionsHTML()`, rules-card look, chosen by screen.
- **Themed days (v389):** `DAILY_OVERRIDES[date] = {letter, theme:{es,en,fr}, categories:{es:[6],en:[6],fr:[6]}, rain:[emojis]}`; categories are display names (any wording; the robot reads them). The theme replaces "Reto diario" on the play label (body slot) and the result title (primary slot 26px on themed days, hero 32px otherwise). Leaderboard, originality, speed bonus and ¿Error? work as on any day. Overrides don't shift other days' picks. Rain: home = sparse continuous notes on `#theme-bg-stage` (behind the content, z-index −1), 20% in front on `#confetti-stage` (big 21–24px, solid) and the rest behind (14–20px, opacity 0.6); daily result = a 4 s burst on top once per day per app open. Each note: linear fall > sine-like sway (ease-in-out alternate, random phase) > tilt on the same wave lagging ¼ cycle. Never while playing, never in Práctica, none with reduce motion; `show()` stops it on any other screen. Preview any date with `?dia=YYYY-MM-DD` and a `__` test name (saves nothing). Current themed days: 2026-11-22 Santa Cecilia (B, music).
- **News:** `NEWS` array in the HTML (`{id, type info|warning, from?, until, es, en, fr}`; `from` since v386), all active ones stacked on the home page, ✕ hides one per device. **Rules (Oscar):** very short, one line, start with an emoji, no jargon, keep up only ~4 days, no "ayer".
- **Debug mode** (see Architecture).

---

## 🧪 Test kit (`D:\09_ALTO\tests\`, not in git)
Run in the sandbox: `cd tests && ./run_all.sh ../index_tmp.html` → must end in `ALL GREEN`. Long browser suites can be run one by one (`cd restore && ALTO_HTML=… python3 x_scenarios.py`). Totals on v395:
- **Unit (Node, extract real functions by name, mocks):** build1 scoring 38, build2 reactions 29, build3 score cards 26, build4 final/share 22, build5a groups 45, build5b group page 35, build6 speed bonus 36, build7 themed day 21, judge prompt 41, revisar 18 (= 311). New helpers some suites need are grabbed with `grabOpt` so the kit also runs on older versions.
- **Browser (Playwright, real HTML, fake Firebase shared between contexts):** restore 19, countdown 12, groups 28, build5b 77, stuck 11, follow 47, gate 38, hostile 24, names 20, clock 12, cleanup 10, robot 24 (v384: retry, automatic second look, midnight), speed 21 (v386: blanks on the button, cut, ¿Error? restoring, old day, Práctica), themed 35 (v389–v394: Santa Cecilia, rain layers, reduce motion, ?dia= preview). clock C3 checks `speedRaw` since v389 (it was date-dependent).
- **Harness notes:** the fake Firebase (`restore/restore_base.py`) refuses keys with `. # $ [ ]` like the real one and aborts transactions on `undefined`; `wait()` pumps Playwright events (never `time.sleep`, it blocks the fake DB); `page.clock.install()` + `fast_forward()` simulates a suspended phone; a fake robot reply can be routed on the worker URL. Let a view settle (~600 ms) before tagging elements (the old build5b flake).
- Also useful: render screens with real fonts (`npm pack @fontsource/caveat @fontsource/special-elite`) at 414/390/375/360/320 px; compare computed styles of all debug screens between two versions (Session 30 method).

---

## 🗺 Backlog (open items only)
### Security — NEXT BUILD (the "security build", agreed with Oscar; do with AI step 1)
- **Firebase is open:** anyone can read/change/delete. Fix: Anonymous Auth + real rules (own entries only, daily scores write-once, validation, no whole-node deletes) + App Check. Needs a migration plan (old data has no owners). Must also cover `following`/`followers` and group lists under `names/`.
- **Name claims are weak:** unsalted SHA-256 of a 4-digit PIN, publicly readable (brute force in under a second), `secret` can be overwritten, the claim write is not a transaction. Fix with the rules: secret unreadable, per-name salt, longer PINs, ideally no emails. The name gate (v378) is client-side only until then.
- **Cookie consent:** small banner (ES/EN/FR), analytics only after "accept" (Oscar agreed).
- **AI step 1:** the worker owns the model list (Cloudflare KV) and caps `max_tokens`, so nobody can run other models on Oscar's credit; the game's `AI_MODELS` becomes the last resort. Oscar pastes the worker code and creates the KV binding.
- **AI step 2 (optional):** daily self-check + automatic scouting via a Cloudflare Cron Trigger (check CPU limits first).
- Related small AI items: pin one provider for Ling; make ¿Error? ask a different model than the first judge; scout skips a model after 4 × 429 and prints OpenRouter's error text.

### Product ideas (not decided unless noted)
- **Speed bonus ½ → ⅓ review (reminder, Session 32):** after about a week of ½ in production, ask Oscar how it feels. Switch to ⅓ if (a) nobody presses ¡Alto! early any more (everyone runs the clock down) or (b) players complain about 🤔 eating the bonus. Claude's view: keep ½ (skipping only pays when truly stuck: it must save 36 s+ on a card that would finish at 0 s; never with 30 s+ to spare; ⅓ = 27 s / 51 s). The switch needs a date gate like `DAILY_SPEED_V2_FROM` (scores are recounted from scratch, so changing `DAILY_MISS_CUT` alone would rewrite past days on screen), plus the help row "−½ bonus", a news line and the tests.
- **Themed days, next candidates (4 Nov):** Sat 5 Dec Monet centenary (+ Disney 125, Mozart): art (painter, colour, flower, animated film, composer, instrument), rain 🎨🖌️🌸; 24–25 Dec Christmas (🎄❄️⭐); also Children's Day 20 Nov (🧸), Agatha Christie 3 Dec (🕵️), Nobel 10 Dec (💡), Inocentes 28 Dec (twist day). Frame deaths as homages; skip sad or political dates. Other fun ideas: 🔥 streaks, weekly league among followed players, answer of the day vote, monthly summary card, kids mode (bigger bet), teams, surprise category.
- **GitHub workflow (not set up):** Claude would clone the repo, push staging itself, production after Oscar's "promote" (or PRs Oscar merges); rollback = `git revert`. Needs `tests/` and `tools/` committed; never `firebase_backups/`.
- **Drop 🤔 (only ✅/❌)?** Idea from Oscar (Session 32). Measured on 24 Oct–1 Nov: 610 answers, 23 🤔 (3.8%): 10 robot failures (two bad days; the v384 second look should now fix these), 13 model DUDOSO (~1.4/day, the genuinely borderline answers; ¿Error? overturned 7, upheld 6, not reliably right). Every 🤔 gets contested (no downside). Decision for now: keep 🤔. Revisit after more judge-prompt work if the scout gets Poto and Radar right. Keep a "couldn't check" state for robot failures whatever happens.
- Daily originality overhaul: unique → `round(50 × log10(players))`, rare (< 5%) +25, common 0.
- Grammar categories (Verbo, Sustantivo, Adjetivo…) mixed into the pool — needs per-language letter validity design first.
- Free-mode example sets (5 themed sets of 20) and category pool expansion (Personaje de ficción, Género musical, Divinidad mitológica, Novela famosa, Festividad).
- Botanical/scientific answers (tomato as fruit): parked by Oscar, host decides.
- Multiplayer ¡Alto!: rule is "every answer ≥ 2 characters" (strict option A). The daily's speed-bonus fairness question is decided and built (v386).
- Automatic AI multiplayer mode (no host review); language as a lobby setting; public rooms/tournaments; sound; letter reveal animation; per-language share previews (`/en/`, `/fr/` pages) only if Oscar shares with EN/FR groups.
- iOS layout refactor (true fixed layout, keyboard resize bug).
- Registration required for all modes (designed Session 15): only the optional reservation + gate exist. If ever enforced: update the meta descriptions ("sin registro").

### Small parked items
- "Top 10" cut can split a tie; own row name squeezed at 320 px with "(+50✨)"; leaderboard search covers only the viewed day; Ling re-check over-generous (watch `contestLog`); FR English words (Robin); the Claude Doc copy of the scoring design has drifted from the `.md` (the `.md` wins); unexplained host-session loss once (Session 26); `og:title` is trilingual, description Spanish (decided).
- Result header date after midnight shows today's date for a daily finished just after midnight (rows, leaderboard and ¿Error? are right since v384; cosmetic).
- Robot misses still open after v384 (scout v5.8): Violin (accent rule loses to "looks English"), Poto, Piton as reptile, Barca as invention, MG as tech brand; ¿Error? still overturns Deutsch (EN), Orlando hat, Radar.
- P2: Spark plan's 100-connection ceiling and the full-day leaderboard download — revisit at ~100 daily players.

### Known limitations (not bugs to fix)
- iCloud Safari sync shares localStorage between iPhone and iPad.
- Safari on iOS shows a new version one launch late (memory cache); fix for a stuck home-screen shortcut: delete and re-add it.
- Closing the app without "Salir" doesn't mark the player as left (the host's "Seguir sin…" covers it).
- During a staging test all players must be on `index_tmp.html` (mixed versions: e.g. a v378 guest doesn't show a v379 host's robot verdict).

---

## 🧾 Decisions that stand (don't re-propose)
- Daily date = phone's local date (B4 dropped, 1 Nov).
- Daily ¡Alto! stays available at any time (one answer minimum); fairness comes from the scoring (option D, ½ per miss, built v386), not from blocking the button (3 Nov).
- Daily category "Superhéroe o villano" is now ES "Héroe o villano de ficción" / EN "Fictional hero or villain" / FR "Héros ou vilain de fiction" (Oscar prefers "vilain" to "méchant"). Renamed in place in `DAILY_CAT_DATA`: keep its position and letters (daily picks depend on the order). Fiction is said in the name, not in the judge prompt (v387).
- Keep 🤔 for now (data in Product ideas, 3 Nov).
- Themed days: a title or name goes by its first letter as written (The Beatles = T, La Bamba = L). Themed-day result title = primary 26px (option A, 4 Nov). Rain design as in the Architecture map (agreed step by step with Oscar, v389–v394).
- Testing economy (4 Nov): targeted checks for small changes; the full kit before production.
- Siguiendo shows only followed players who played (v374); ranks are dense; legend always shown.
- One judge prompt for every mode; ¿Error? asks one model.
- News in the HTML, not Firebase; ~4 days each.
- Joining with the host's exact name makes you host.
- Capitals kept as typed in names ("PEPE" stays "PEPE").
- Size limit waived.
- Process: plan first; test only what changed; few tool calls; tell Oscar plainly when a claim was wrong.
