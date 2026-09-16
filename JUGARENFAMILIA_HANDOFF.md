# JugarEnFamilia.es — Session 17 Handoff

## Current production version
**v260916.251** — rollback to v260913.243 base + countdown label change only.

---

## What shipped in session 17

### ✅ Countdown label — gender neutral
- ES: `PREPARADOS` → `¡VAMOS!`
- EN: `GET READY` → `LET'S GO!`
- FR: `PRÊTS` → `C'EST PARTI!`
- 7 occurrences updated: 3 T object keys, 3 fallback strings, 1 static HTML default

### ✅ Firebase manual corrections (today's corrupted scores)
Scores were corrupted by a buggy recontest implementation (see below). Fixed manually in Firebase console:
- `[ES] Pierrot_1789568685446_4198` — michael jackson (Cantante) corrected to valid. `baseScore:30, totalScore:45`
- `[EN] Venirse primos_1789568088129_3258` — `baseScore:30, totalScore:36`
- `[ES] Oscar_1789548011864_1259` — `baseScore:50, totalScore:59`
- `[FR] Venirse primos_1789568298471_1479` — `baseScore:20, totalScore:22`
- `[FR] Oscar — Elche` corrected from unsure to valid manually

### ❌ Rolled back — recontest changes
Everything below was attempted and rolled back due to score corruption bug. Do not re-apply without following the rebuild plan in the next section.

---

## AI validation analysis (from Firebase export Sep 2–16)

- 813 answers analysed across all 3 langs
- Overall invalid rate: 31% (ES 30%, EN 29%, FR 35%)
- Worst categories: Película animada (47–80% invalid), Árbol ES (71%), Type de magasin FR (71%)
- Clear false positives: Frank Sinatra, Fahrenheit, Paris, Rumba, Oboe, Jazmin, Marguerite, Foucault, Finding Nemo, Matelas
- Decision: leave prompt as-is (v001), rely on recontest as safety valve
- `DAILY_CONSENSUS` left as `false` (single model) — monitor before changing

---

## Recontest rebuild — next session

### Why it was rolled back
The recontest code had a **units mismatch bug**: `baseScore` is stored in raw units (1 valid = 10, 1 unsure = 5, so 5 valid = baseScore 50, displayed as 500pts). The recontest added `addedPts = 100` (display value) instead of `10` (raw unit), causing 10× score inflation written to Firebase.

### What the correct behaviour should be
- Button only on `❌` (invalid verdicts) — not unsure
- 1-of-3 models enough to overturn (was 2-of-3, never overturned anything)
- DUDOSO from models counts as valid (benefit of the doubt)
- Two outcomes only: `valid` or `invalid` — no unsure edge cases
- Score recalculated from scratch (no deltas)
- Firebase written first, UI updated after confirmation
- `speedMultiplier` stored explicitly in Firebase on submit

### Architecture facts (read the code before assuming)

**`baseScore` units:** raw units. 1 valid = 10, 1 unsure = 5. Display multiplies by 10.
So 5 valid answers = `baseScore: 50` = displayed as `500 pts`.

**`speedMultiplier`:** calculated at game end as `1.0 + (secsLeft / totalSecs)`, range 1.0–2.0.
- ✅ Saved to localStorage on submit (inside `saveData`)
- ❌ NOT saved to Firebase on submit — needs to be added (Step 1)
- Cross-device path derives it as `entry.totalScore / entry.baseScore` — fragile

**Same-device reload path:** reads localStorage → `showDailyResult`. `speedMultiplier` is available. ✅

**Cross-device path:** fetches all scores from Firebase, finds entry by `safeName`, calls `showDailyResult`. Derives `speedMultiplier` from `totalScore / baseScore`. Already reads from Firebase correctly ✅ — does NOT use localStorage.

**`G_daily.speedMultiplier`:** set by `showDailyResult` from whatever is passed in. Available at recontest time on same device. Needs hard guard: `G_daily.speedMultiplier || (G_daily.totalScore / G_daily.baseScore) || 1`.

**`G_daily.aiResults`:** object with numeric string keys after JSON round-trip (`"0"`, `"1"` etc). Always use `String(idx)` when writing. When reading, use `G_daily.aiResults[i] || G_daily.aiResults[String(i)]` or normalise on load.

**`G_daily.answers`:** populated from game play and from `showDailyResult`. Must be available at recontest time — verify this holds for both same-device and cross-device paths.

**`G_daily.playerKey`:** set on submit, saved to localStorage. Available same-device. Not available cross-device (different localStorage). Recontest is only available on same device (button only shows with `isToday` check) so this is fine.

**Practice mode:** `G_daily.practiceMode` skips Firebase writes. Add explicit early return at top of recontest.

**Contest count path:** `names/{safeName}/contests/{date}/{lang}` — written before model calls. If score write fails after models return, contest count is already consumed. Acceptable — don't retry, just toast an error.

**Originality:** recalculated automatically when `loadDailyLeaderboard` fires after recontest. If contested answer flips to valid, it is eligible for originality bonus. This is intentional. ✅

**Race condition (pre-existing):** two players contesting simultaneously could both pass the `used >= 1` gate. Low probability with current player base — noted, not fixing now.

---

### Build plan — step by step

#### Step 1 — Add `speedMultiplier` to Firebase on submit
**File:** `index.html` — Firebase `set()` call on submit (~line 2748)
**Change:** add `speedMultiplier` to the payload object
**Also:** update cross-device path (~line 2389) to read `entry.speedMultiplier || (entry.totalScore / entry.baseScore) || 1`
**Verify:** grep confirms `speedMultiplier` appears in Firebase payload. Deploy and confirm field appears in Firebase console on next play.
**Risk:** LOW. No impact on anything else.

#### Step 2 — Same-device reload reads from Firebase
**File:** `index.html` — already-played localStorage path (~line 2351)
**Change:** if localStorage key exists, read `playerKey` from it, fetch `daily/{date}/{lang}/scores/{playerKey}` from Firebase, pass Firebase data to `showDailyResult`. Fall back to localStorage if fetch fails or `playerKey` missing.
**Loading state:** show brief "cargando…" while Firebase fetch runs — don't flash blank screen.
**Verify:** play the game, reload the page, confirm result screen shows Firebase data. Check fallback works by temporarily breaking the Firebase path.
**Risk:** MEDIUM. Extra Firebase read per returning visit (fine on Spark plan at current scale). Adds latency — needs loading state.

#### Steps 3–6 — Recontest rewrite (ship as one atomic change)

**Paper verify BEFORE writing any code:**

Case A — invalid→valid, 4 valid before, mult=1.50:
- Before: `baseScore=40, totalScore=60`
- Recount after: validCount=5, unsureCount=0 → `newBaseScore=50`
- `newTotalScore = round(50 * 1.50) = 75`
- Toast: `(50-40)*10 = +100 pts`
- Firebase writes: `baseScore:50, totalScore:75, speedMultiplier:1.50, aiResults/idx:'valid'`

Case B — invalid→valid, 3 valid 1 unsure before, mult=1.20:
- Before: `baseScore=35, totalScore=42`
- Recount after: validCount=4, unsureCount=1 → `newBaseScore=45`
- `newTotalScore = round(45 * 1.20) = 54`
- Toast: `(45-35)*10 = +100 pts`
- Firebase writes: `baseScore:45, totalScore:54, speedMultiplier:1.20, aiResults/idx:'valid'`

**Changes to make:**

1. **Threshold:** `votes >= 2` → `votes >= 1`
2. **DUDOSO:** `parseVerdict` returns `'valid'` for anything not explicitly invalid
3. **Score recount — replace delta logic entirely:**
```js
const oldBaseScore = G_daily.baseScore;
G_daily.aiResults[String(idx)] = newRes;
let validCount = 0, unsureCount = 0;
for (let i = 0; i < (G_daily.categories.length || 6); i++) {
  const v = G_daily.aiResults[String(i)] || G_daily.aiResults[i];
  if (G_daily.answers[i]) {
    if (v === 'valid') validCount++;
    else if (v === 'unsure') unsureCount++;
  }
}
const newBaseScore = validCount * 10 + unsureCount * 5;
const mult = G_daily.speedMultiplier || (G_daily.totalScore / G_daily.baseScore) || 1;
const newTotalScore = Math.round(newBaseScore * mult);
const toastPts = (newBaseScore - oldBaseScore) * 10;
```
4. **Firebase write first, UI after:**
```js
try {
  await update(scoreRef, {
    [`aiResults/${String(idx)}`]: newRes,
    baseScore: newBaseScore,
    totalScore: newTotalScore,
    speedMultiplier: mult,
  });
  // UI updates here
  // localStorage update here
  toast('+' + toastPts + ' pts');
} catch(e) {
  G_daily.aiResults[String(idx)] = currentVerdict; // revert
  btn.disabled = false;
  toast(errorMessage);
}
```
5. **localStorage update:**
```js
const lsKey = 'alto_daily_' + key + '_' + LANG;
const stored = localStorage.getItem(lsKey);
const d = stored ? JSON.parse(stored) : {};
if (!d.aiResults) d.aiResults = {};
d.aiResults[String(idx)] = newRes;
d.baseScore = newBaseScore;
d.totalScore = newTotalScore;
d.speedMultiplier = mult;
localStorage.setItem(lsKey, JSON.stringify(d));
```
6. **aiIcon map:** `{ valid:'✅', unsure:'🤔', invalid:'❌' }` — was missing `invalid`
7. **Practice mode:** add `if (G_daily.practiceMode) return;` at top of recontest
8. **Prompt:** update system prompt — `"Only overturn if confident"` → `"Give benefit of the doubt"`
9. **Toast:** `toastPts` as calculated above — not `addedPts * 10`

**Verify after build:**
- Play game, get an invalid answer, contest it
- Check Firebase before and after — confirm `baseScore`, `totalScore`, `speedMultiplier` all correct
- Reload page — confirm scores still correct
- Check leaderboard shows correct score
- Try contesting when already used — confirm blocked correctly
- Confirm practice mode cannot contest

---

## Standard rules reminder
- Check before building, deploy immediately after
- Grep version before AND after bump
- Pre-deploy checklist: file < 300KB, 12 screen IDs, no -tmp, JS syntax clean
- 6-slot font system — no new sizes
- Commit messages plain ASCII
- SVG flags have `display:block` — no inline with text
- Never inline `oninput` with value reassignment
- Claim pill visibility CSS-only

## File locations
- Production: `D:\09_ALTO\index.html`
- Deploy: `deploy.bat`
- Rollback: `rollback.bat`
- Firebase: `https://stop-9f0ea-default-rtdb.europe-west1.firebasedatabase.app`
- Live: `https://jugarenfamilia.es`
