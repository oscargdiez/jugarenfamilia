# Multiplayer Scoring Overhaul - Design

Design complete 26 Sep 2026. Nothing built yet. Production at time of design: v260922.301. All builds go to staging (`-tmp`) first.

Living version of this doc (Claude Doc): https://claude.ai/artifact/9bCUk2yRwx1wHQG3hmjhoV

The goal: make multiplayer scoring richer and fairer. Reward clean, fast ¡Alto! calls, let the table reward great and funny answers, show a clear per-round breakdown, and keep a history of each group's games.

| Area | State |
| --- | --- |
| Answer points | Decided (no change) |
| Alto bonus/penalty | Decided |
| Reaction bonuses | Decided |
| Ties and medals | Decided |
| Round log data model | Decided |
| Scores screen cards | Decided |
| Final screen and share | Decided |
| Groups | Decided |
| History | Decided |
| Languages | Decided |
| Stats | Dropped |

---

## Current system (v301, for reference)

Only cumulative totals are stored; per-round detail is wiped by `nextRound()`.

- **Answers:** unique valid = 100, duplicate valid = 50, invalid or empty = 0. Computed in `finishValidation`, always from `preRoundScores`.
- **Alto penalty:** flat, set by the lobby slider (0 to 200, default 50). Applied if the caller has any invalid or empty answer. No bonus for a clean call.
- **Reactions:** four emojis (🔥 👏 😂 😬) stored in `entryReactions/{player__cat}/{reactor_emoji}`. Unlimited, can be put on your own answers, worth no points. `goBackToValidation` wipes them.
- **Medals:** `MEDALS[i]` by sort position, so tied players get different medals.
- **End of game:** `endGame` gives a win to `sorted[0]` only (even in solo games) and updates `global/` and `groups/{nameKey}/` win/game counts. Player keys include group + name + emoji.
- **Groups:** free-text name typed by the host in the lobby; host device keeps the last 5 names in `alto_groups`. Group name is only written to the room on Comenzar.
- **Room lifetime:** deleted 45 minutes after the game ends.

---

## Answer points

No change: unique valid 100, duplicate valid 50, invalid or empty 0.

The old plan's "+50 originality bonus" is dropped. In multiplayer "unique" already means nobody else in the room wrote it, and there is no wider pool to compare against like daily has. Reaction bonuses replace it.

With 8 categories the answer maximum is 800 per round. In free mode it is 100 x number of categories.

---

## Alto caller bonus and penalty

The caller gets +250 (scaled to the number of categories), minus 100 for each invalid answer. The lobby penalty slider is removed.

```
adjustment = round10(250 x n / 8) - 100 x invalid
```

n = number of categories in the round. Only the bonus scales; the -100 per invalid is fixed.

| Invalid (8 categories) | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| Adjustment | +250 | +150 | +50 | -50 | -150 | -250 | -350 | -450 | -550 |

| Categories | 2 | 4 | 5 | 8 | 12 | 20 |
| --- | --- | --- | --- | --- | --- | --- |
| Clean-call bonus | +60 | +130 | +160 | +250 | +380 | +630 |

**Rules**

- Duplicates count as correct; only invalid answers count against the caller.
- Empty answers cannot happen: the existing guard stops ¡Alto! unless every answer has 2+ characters.
- Round ends on timeout: no adjustment for anyone.
- Democratic mode: "invalid" means invalid after the vote.

**Why this is fair.** Calling ¡Alto! cuts everyone else's time short. Finishing all 8 correctly first is a real feat and is rewarded strongly. Spamming guesses to end the round early is punished hard.

| Caller's round | Answers | Adjustment | Total |
| --- | --- | --- | --- |
| 8 valid (6 unique, 2 duplicates) | 700 | +250 | 950 |
| 6 valid, 2 invalid | 600 | +50 | 650 |
| 5 valid, 3 invalid | 500 | -50 | 450 |
| 8 guesses, all invalid | 0 | -550 | -550 |

Calling stops paying off between 2 and 3 invalid answers. Intentional: 6 of 8 at speed is still a good round.

**Scaling is already proportional.** Every category is worth 100, so -100 per invalid always means "one answer's worth". As a share of the round maximum:

| Caller's result | 8 categories (max 800) | 2 categories (max 200) |
| --- | --- | --- |
| All correct | +250 (31%) | +60 (31%) |
| Half invalid | -150 (-19%) | -40 (-20%) |
| All invalid | -550 (-69%) | -140 (-70%) |

Scaling the -100 down as well would break this (a 2-category caller with both answers invalid would get +10, rewarding spam). Keep the formula as is.

---

## Reaction bonuses

| Reaction | Points | Counts on |
| --- | --- | --- |
| 🔥 | +30 | Valid answers only |
| 👏 | +20 | Valid answers only |
| 😂 | +10 | Valid or invalid (a funny wrong answer still earns a laugh) |
| 😬 | 0 | Anything, just for fun |

**Rule:** each player can put **one reaction on each answer** (any of the four), and none on their own answers. Tapping a different emoji on the same answer swaps it; tapping the same one removes it. 😬 is a deliberate "meh" that uses your reaction on that answer and gives 0.

**Lobby toggle:** "🌶️ Reacciones con puntos" (EN "Reactions score points", FR "Réactions avec points"). **Default on.** Hosts can switch it off for a classic game.

- **On:** points as above, no cap.
- **Off:** reactions are cosmetic, with the same one-per-answer and no-self rules. Scoring is answers plus ¡Alto! only.
- Saved in `alto_lobby_config` and in the room as `reactionPoints: true/false`.
- Scores cards and the rules panel only show the reactions line when it is on.

**Size check** (max bonus per player per round, 8 valid answers, all 🔥 from everyone): 2 players 240, 4 players 720, 6 players 1200, against an answer maximum of 800. The extreme case is unlikely in practice since fire goes to genuinely good or funny answers. `roundLog` records every reaction, so a cap can be added later if real games get out of hand. Alternatives considered: cap at 25 x categories, lower values, averaging.

**Rules**

- You cannot react to your own answers.
- Reactions lock when the host taps Calculate. Only reactions present at that moment score.
- If an answer is invalidated after receiving 🔥 or 👏, those do not score (😂 still does).
- Fix needed: `goBackToValidation` must stop wiping `entryReactions`, or re-checking an answer erases everyone's bonuses.

---

## Ties and medals

Tied players share a medal and the next score gets the next medal (dense ranking).

| Scores | 900 | 900 | 700 | 500 | 300 |
| --- | --- | --- | --- | --- | --- |
| Medal | 🥇 | 🥇 | 🥈 | 🥉 | none |

- Applies to the Scores screen between rounds and the Final screen.
- Every player tied for first gets a win on the group leaderboard (complete, non-solo games only).
- Tied final: the winner line names all winners ("¡Empate!" in ES, with EN/FR equivalents); confetti uses the first winner's emoji.

---

## Round log data model

`finishValidation` writes a complete snapshot of each round to `room.roundLog/{round}`. The Scores cards, Final screen, share text and History all read from it.

```
roundLog/{round}:
  letter: "M"
  categories: ["Nombre", "Ciudad", ...]
  caller: "Oscar"            // or "" on timeout
  players/{name}:
    answers: ["Marta", "Madrid", ...]     // by category index
    status:  ["u", "d", "x", "e", ...]    // unique, duplicate, invalid, empty
    got:     [{"f":1,"c":0,"l":2}, ...]   // reactions received per answer: fire, clap, laugh
    pts: { answers: 550, alto: 150, reactions: 110, round: 810, total: 2140 }
```

- Keyed by category **index**, not name. Free-mode category names can contain characters Firebase does not allow in keys (`. # $ [ ] /`).
- Rewritten in full every time `finishValidation` runs, so going back to validation and recalculating stays correct.
- `pts.total` is the cumulative score after this round.
- About 3 KB per round for 6 players.
- History copies `roundLog` plus final scores into its own path when the game ends.
- `playAgain` clears `roundLog`.

---

## Scores screen (between rounds)

One expandable card per player, **all collapsed by default**. Between rounds the cards show **this round only**; the full round-by-round breakdown lives on the Final screen.

**Collapsed card:** medal, emoji, name, red **A** badge if this player called ¡Alto!, this round's points (+810) and running total (2140).

**Expanded card:**

- This round's answers, one line each: category, answer, status mark (unique, duplicate, invalid) and reactions received.
- Breakdown line: `answers 550 · A +250 · reactions +150`. The A part only appears for the caller; the reactions part only when reaction points are on.

**The A badge**

- Red, same red as the invalid marks, next to the name.
- Same badge whether the call went well or badly; the number in the breakdown line tells the story (+250 or -150).
- No badge on a round that ended by timeout.

Fonts stay within the 6-slot system.

---

## Final screen and share

Reuse the Scores cards, but expanded they show **every round** (round, letter, answers, points). Plus a row of end-of-game awards built from `roundLog`.

**Awards (three)**

| Award | Counts |
| --- | --- |
| 🔥 Popularidad / Popularity / Popularité | Points from 🔥 and 👏 received over the whole game (30 and 20 each) |
| 😂 Risas / Laughter / Rires | Number of 😂 received |
| ✨ Originalidad / Originality / Originalité | Number of unique valid answers (nobody else wrote them) |

- Only shown when someone earned it (count above 0). Ties list all names.
- Work even when reaction points are off (reactions are still given cosmetically).
- Names are nouns, so they work for any player in every language (the app does not know players' gender).
- Dropped: best ¡A!, best single round.

**Share text: scores only.** Same plain-text style as the daily share so it works in WhatsApp. Awards included (one line). Built in the sharer's language. Includes the group link (`?g=ID`).

```
¡ALTO! · Los Pérez · 5 rondas
🥇 Oscar 3240
🥈 Marta 2980
🥉 Luis 2410
🔥 Marta · 😂 Luis · ✨ Oscar
jugarenfamilia.es/?g=...
```

---

## Solo and incomplete games = practice

- A game that **starts** with only one player is practice: no group leaderboard update, no history. (Today solo games count a win, so anyone can farm free wins; this fixes that.)
- If two start and one leaves, it is still a real game; the missing player gets empty answers.
- A game ended **before all rounds are played** is also practice: no history, no leaderboard.
- Tapping Terminar before the last round shows a confirm: "La partida no está completa y no contará. ¿Terminar igualmente?" (EN "The game isn't complete and won't count. End anyway?", FR "La partie n'est pas terminée et ne comptera pas. Terminer quand même ?"). After the final round, Terminar works as today with no confirm.
- While only one player is in the lobby, the start button reads "✏️ Practicar solo →" (EN "✏️ Practise solo →", FR "✏️ S'entraîner en solo →"). The ✏️ matches the existing Práctica button. No hint line under it.

---

## Groups

Each group has a hidden random **ID**; the name is only a label. Names can repeat (a thousand families can each have "Familia") without mixing, so no name reservation, PIN or account is needed. Groups are multiplayer only; daily is not affected.

**Access comes from having played with the group.** A guest never has to type a group name or remember anything.

### Mis grupos (on each device)

An upgrade of the existing `alto_groups` list.

- Up to 10 groups, sorted by last played. Top 3 shown, "Ver todos" for the rest.
- The host's default group is always pinned first.
- The most recently played group is preselected when opening a room (the default group is pinned but not automatically selected).
- When an 11th group arrives, the one played least recently drops off the device. It is not deleted from Firebase; playing with that group again brings it back.
- A small × removes a group by hand (for a one-off party).

### Where it lives: the lobby

"Crear sala" takes the host straight to the lobby, so the Mis grupos list becomes the lobby group picker. **The home screen is unchanged.**

- **Host:** `[Los Pérez ▾] [📜 Historial]`. The picker holds the list, plus "Nuevo grupo" and rename.
- **Guests:** group name plus a Historial button.
- **Group written to the room as soon as the host picks it** (not only on Comenzar as today), so guests see the name and Historial while waiting.

Trade-offs accepted: browsing history at home means opening a room (harmless if never started); guests can only browse the group of the room they are in, and their other groups appear in their picker when they host.

### Default group

The first time someone hosts, a default group is created with its own ID and pinned in their list. Its name is plain text in the host's language, written once: "Grupo de Oscar" / "Oscar's group" / "Groupe d'Oscar". Two hosts called Oscar never collide. It is per device (a host with a phone and a tablet gets two default groups; known limitation).

### How players reach a group

| Situation | What happens |
| --- | --- |
| In the lobby | Group name shown; Historial opens its page |
| After a game (complete, not solo) | Every player who reaches the final screen gets the group added to or bumped in their list |
| Share link | `?g=ID` opens the group page and adds the group to the list. Also the recovery path for a new phone |
| Hosting | Lobby picker: your list plus "Nuevo grupo". A guest can host the family game and it lands in the same group |

### Renaming

Anyone hosting a group can rename it from the lobby picker. The name lives in `groupsMeta`, so everyone's list shows the new name next time it loads.

### Players inside a group

Keyed by normalised name, **without emoji**. The leaderboard shows each player's most recent emoji. Two players with the same name in one group merge into one row (rare in a family; fixed by typing "Marta P.").

### Group leaderboard

Keeps its current columns: wins, games, win %. **The global leaderboard is removed** (comparing wins across different groups, rounds and categories is meaningless). The Leaderboard screen becomes the group page.

### Firebase paths

```
groupsMeta/{groupId}:              name, createdAt, lastPlayed
groupsLb/{groupId}/{player}:       name, lastEmoji, wins, games
historyIndex/{groupId}/{gameId}:   endedAt, winner, lang   // small, for ◀ ▶ navigation
history/{groupId}/{gameId}:        full game incl. roundLog // loaded one game at a time
```

The index split keeps browsing cheap: opening a group downloads only the small index; a full game loads only when viewed.

### Retention and cleanup

- **History:** the last 20 games per group, however old. Trimmed when a game ends.
- **Dead groups:** groups with no game in over a year are deleted, a few at a time, whenever any game ends. Needs a one-time Firebase rules change: `".indexOn": "lastPlayed"` on `groupsMeta`.

### Clean start (no migration)

- Nothing old is kept. When Build 5 goes live, delete `groups/`, `groupNames/` and `global/` in the Firebase console.
- The old name list on each device (`alto_groups`, plain strings) is cleared the first time the new code runs.

### Small fixes found on the way

- The welcome-back screen shows `alto_last_group` (last name typed) instead of the room's actual group. Switch to the room's group name.
- The group leaderboard has "partidas" hard-coded in Spanish; needs a T key.

---

## History

History belongs to the group (see Groups).

- **Navigation:** by game, not by date (a group can play several games in one evening): `◀ 🇪🇸 25 sep · 21:40 ▶`. The flag shows the language that game was played in. SVG flags have `display:block` built in, so flag and date sit side by side in a small flex row, never inline in text.
- **Location:** the group page (the existing Leaderboard screen). Screen count stays at 12.
- **Reached from:** the lobby Historial button, the share link, and after a game.
- **One entry per game:** each "Jugar de nuevo" in the same room is a separate game.
- **Content per game:** final ranking with medals, host badge, awards, and each player's round-by-round breakdown from `roundLog`.
- **Loading:** ◀ ▶ reads only `historyIndex`; the full game from `history` loads when viewed.

---

## Languages (ES, EN, FR)

**One group, whatever language each game is played in.** The host of each game decides its language. The group's name is text in the language of the host who created it, and is never translated. Each game is tagged with its own `lang`, shown as a flag next to the date in history. A family where the French cousin sometimes hosts stays one group with one leaderboard. Each device's UI labels still follow that device's language setting, as today.

| Data | Stored as | Shown |
| --- | --- | --- |
| Answer status | `u` `d` `x` `e` | Marks and labels in the viewer's language |
| Reactions received | `f` `c` `l` counts | Emojis |
| Awards | Keys: popularity, laughter, originality | Award names in the viewer's language |
| Game date | Timestamp | Formatted per viewer ("25 sep", "25 Sep", "25 sept.") |
| Game language | `lang` per game | SVG flag in history nav |
| Default group name | Plain text in the creating host's language | As stored, same for everyone |
| Renamed group | Typed name | As typed |
| Categories | As played (`displayCats`, host's language) | As played |
| Answers | As typed | As typed |

**Housekeeping**

- Every new string goes into the T object in ES, EN and FR in the same build.
- Rules panel: ¡Alto! scale and reaction points rewritten in all three; old flat-penalty text and the `validateSub2` hint replaced.
- Remove dead strings: lobby penalty slider, global leaderboard tab (`worldSub`, `noGroupGlobal`), old group hint text.
- Share text built in the sharer's language; "¡ALTO!" brand unchanged in all three.

---

## Stats (dropped)

Multiplayer does not need per-player stats; group history covers what matters. Ideas considered (vocabulary, palabra comodín, Alto accuracy) can be revisited if a registration system is ever built.

---

## Build plan

Five builds, each on staging first, tested, then promoted. Line numbers as of v301.

| Build | Scope | Main code touched |
| --- | --- | --- |
| 1. Scoring engine | Alto scale, slider removed, tied medals, ties share wins, solo and incomplete games as practice (no leaderboard, "Practicar solo" button label, early-end confirm), `roundLog` written | `finishValidation` ~5309, `showScores` ~5359, `showFinal` ~5536, `endGame` ~5422, `playAgain` (clear `roundLog`), lobby slider ~1086, `saveLobbyConfig` ~3913, `createRoom` / `startGame` `stopPenalty` ~4213, rules panel ~1556, `validateSub2` in T and `setValidationMode`, start button ~1041, debug data ~5697 |
| 2. Reaction bonuses | One reaction per answer, no self-reactions, lobby toggle (default on), scoring, keep reactions on go-back | `reactToEntry` ~5123, `renderEntryEmojis` ~5156, `goBackToValidation`, `finishValidation`, lobby config |
| 3. Scores cards | Expandable cards, this round only, A badge | `showScores`, new CSS |
| 4. Final and share | Final cards with every round, three awards, share text | `showFinal`, share function |
| 5. Groups and history | Group IDs, Mis grupos list and lobby picker, default group, renaming, share link, group written to room live, global leaderboard removed, history on group page with language flag, 20-game trim, yearly cleanup, clean start, welcome-back fix, "partidas" fix | `endGame`, `renderLb` and Leaderboard screen, lobby group input and suggestions ~1036, `getSavedGroups` / `saveGroupName` ~1930, lobby host panel and guest card, welcome-back screen ~5658 |

**Oscar's manual steps for Build 5:** add the `lastPlayed` index to Firebase rules; delete `groups/`, `groupNames/`, `global/` in the console.

**Risks**

- **Translations:** every new string needs ES, EN and FR at the same time.
- **Mixed versions:** during a staging test, all players in a room must be on the same file, since scoring runs on the host but cards render on every client.
