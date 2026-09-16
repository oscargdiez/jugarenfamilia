# JugarEnFamilia.es — Session 18 Handoff

## Current production version
**v260916.252** — recontest rewrite + speedMultiplier to Firebase

---

## What shipped in session 18

### Step 1 — speedMultiplier saved to Firebase on submit
- Added `speedMultiplier` to the Firebase `set()` payload on game submit (~line 2751)
- Updated cross-device path to read `entry.speedMultiplier || (entry.totalScore / entry.baseScore) || 1` instead of deriving only (~line 2386)

### Steps 3-6 — Recontest rewrite (atomic)
All shipped as one change. Summary of every fix:

1. **Practice mode guard** — `if (G_daily.practiceMode) return;` at top of `window.recontest`
2. **Prompts** — updated to benefit-of-the-doubt language (was "strict, only overturn if certain")
3. **`parseVerdict`** — DUDOSO now returns `'valid'` instead of `'invalid'`
4. **Threshold** — kept at `votes >= 2` (2-of-3 models must return valid)
5. **Score recount** — full scratch recount replacing buggy delta logic (was adding display units to raw units = 10x inflation)
6. **Firebase written first** — `await update()` before any UI changes; inner try/catch reverts `G_daily.aiResults` and re-enables buttons on failure
7. **localStorage** — written after Firebase success; now includes `speedMultiplier`
8. **`aiIcon` map** — added `invalid:'❌'` (was missing)
9. **Toast** — uses `toastPts = (newBaseScore - oldBaseScore) * 10` (was `addedPts * 10`)
10. **`String(idx)` keys** — Firebase and aiResults writes now use `String(idx)`; reads use `G_daily.aiResults[String(i)] || G_daily.aiResults[i]` to handle both string and numeric keys from JSON round-trips

### What was NOT done (Steps 2 — same-device reload from Firebase)
Step 2 from the previous handoff (same-device reload reads from Firebase instead of localStorage) was not built. Still on the backlog.

---

## Architecture facts (updated)

**`speedMultiplier`:**
- ✅ Saved to localStorage on submit
- ✅ Saved to Firebase on submit (new in this session)
- ✅ Read from Firebase on cross-device path (new in this session)
- ✅ Written to Firebase on recontest overturn (new in this session)
- ✅ Written to localStorage on recontest overturn (new in this session)
- Cross-device path: `entry.speedMultiplier || (entry.totalScore / entry.baseScore) || 1`
- `G_daily.speedMultiplier` guard in recontest: `G_daily.speedMultiplier || (G_daily.totalScore / G_daily.baseScore) || 1`

**Recontest (`window.recontest`):**
- Button only on `❌` (invalid verdicts) — not unsure
- 1 contest per player per lang per day (gate: `used >= 1`)
- 2-of-3 models must return valid to overturn
- DUDOSO from models counts as valid (benefit of the doubt)
- Score recalculated from scratch (no deltas)
- Firebase written first, UI updated after confirmation
- `speedMultiplier` stored explicitly in Firebase on overturn

**`baseScore` units:** raw units. 1 valid = 10, 1 unsure = 5. Display multiplies by 10.
So 5 valid answers = `baseScore: 50` = displayed as `500 pts`.

**`G_daily.aiResults`:** object with numeric string keys after JSON round-trip (`"0"`, `"1"` etc). Always use `String(idx)` when writing. When reading, use `G_daily.aiResults[i] || G_daily.aiResults[String(i)]` or normalise on load.

**`G_daily.answers`:** populated from game play and from `showDailyResult`. Available at recontest time on same device.

**`G_daily.playerKey`:** set on submit, saved to localStorage. Available same-device only. Recontest is only available on same device (button only shows with `isToday` check) so this is fine.

---

## Backlog for next session

### Step 2 — Same-device reload reads from Firebase
**File:** `index.html` — already-played localStorage path (~line 2351)
**Change:** if localStorage key exists, read `playerKey` from it, fetch `daily/{date}/{lang}/scores/{playerKey}` from Firebase, pass Firebase data to `showDailyResult`. Fall back to localStorage if fetch fails or `playerKey` missing.
**Loading state:** show brief "cargando…" while Firebase fetch runs — don't flash blank screen.
**Verify:** play the game, reload the page, confirm result screen shows Firebase data. Check fallback works by temporarily breaking the Firebase path.
**Risk:** MEDIUM. Extra Firebase read per returning visit (fine on Spark plan at current scale). Adds latency — needs loading state.

---

## AI validation analysis (from Firebase export Sep 2–16)

- 813 answers analysed across all 3 langs
- Overall invalid rate: 31% (ES 30%, EN 29%, FR 35%)
- Worst categories: Película animada (47–80% invalid), Árbol ES (71%), Type de magasin FR (71%)
- Clear false positives: Frank Sinatra, Fahrenheit, Paris, Rumba, Oboe, Jazmin, Marguerite, Foucault, Finding Nemo, Matelas
- Decision: leave prompt as-is (v001), rely on recontest as safety valve
- `DAILY_CONSENSUS` left as `false` (single model) — monitor before changing

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
- Deploy files must be named `jugarenfamilia.html` and `gitinfo.txt`
- Always state the version string explicitly when handing over deploy files

## File locations
- Production: `D:\09_ALTO\index.html`
- Deploy: `deploy.bat`
- Rollback: `rollback.bat`
- Firebase: `https://stop-9f0ea-default-rtdb.europe-west1.firebasedatabase.app`
- Live: `https://jugarenfamilia.es`
