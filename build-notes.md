# PlanHub Casino — Build Notes
_Created: April 2026 | Status: Ready to Deploy_

---

## Overview

Single-file sales contest app. AEs earn dice rolls by closing deals; SDRs earn rolls by warm transferring. Dice game: 2d6 vs computer — win/lose/tie (double or nothing up to 8×). Blackjack unlocks once a player has positive dice winnings; wager from bankroll pool.

---

## Tech Stack

| Layer | Choice |
|---|---|
| Frontend | Single `index.html` — vanilla HTML/CSS/JS, no build step |
| Auth | Firebase Auth — Google login, `@planhub.com` domain only |
| State | Firebase Realtime Database — `planhub_casino/` namespace |
| Hosting | GitHub Pages — `jrank-coder/planhub-casino` → `https://jrank-coder.github.io/planhub-casino/` |
| Sync | Claude Code scheduled task — `scheduled-tasks/planhub-casino-sync/SKILL.md` |

---

## Firebase Data Structure

```
planhub_casino/
  players/{owner_id}/
    name, email, role
    rollsAvailable, rollsEarned
    diceWins, diceLosses, diceWinnings, totalWinnings
    bjHandsPlayed, bjWins, bjLosses, bjPushes
    diceLog/  (pushed entries: ts, won, prize, mult)
    bjLog/    (pushed entries: ts, wager, outcome, net)
  settings/
    adminPassword: "casino2026"
    prizeAmount: null          ← admin must set before play
    surpriseMode: false
    surpriseModeMin/Max: null
    bjEnabled: true
    contestLabel: "April 11, 2026"
  rollSyncHWM/{owner_id}/
    closes: N
    transfers: N
  lastSync: "ISO timestamp"
```

---

## Game Logic

### Dice
- 2d6 player vs 2d6 computer
- Win: prize = `prizeAmount × multiplier` (or random in range if Surprise Mode)
- Loss: roll consumed, no money deducted
- Tie: auto-reroll with multiplier doubling (1× → 2× → 4× → 8× max)
- Firebase: `increment()` for all counters, `push()` for log entries

### Blackjack
- Unlocks when `diceWinnings > 0`
- Standard 52-card deck, Fisher-Yates shuffle, fresh per hand
- Actions: Hit, Stand, Double Down (first 2 cards), Split (same rank, same hand only)
- Dealer rule: hit soft 17, stand hard 17+
- Natural BJ pays 3:2 (⌊wager × 1.5⌋), rounded down
- Split: each hand evaluated independently; net result applied to `totalWinnings`
- Wager min: $1, max: current `totalWinnings` (cannot go negative)

---

## Roster

18 AEs + 8 SDRs (SC + SP) hardcoded from `context/team-roster.md`.
Upsell SDRs (Megan McCaughey, Amanda Carlin) excluded — pending confirmation of participation.
Admin access: Josh Rank, John Maurer, Nick Cappello — can access admin panel but not players on leaderboard.

---

## Deployment Steps

1. **Create GitHub repo** `planhub-casino` under `jrank-coder`
2. **Enable GitHub Pages** on `main` branch
3. **Add authorized domain** `jrank-coder.github.io` to Firebase Auth (check — may already be there from ROE tool)
4. **Add Firebase security rules** under `planhub_casino/`:
   ```json
   {
     "rules": {
       "planhub_casino": {
         ".read": "auth != null && auth.token.email.matches('.*@planhub\\\\.com')",
         ".write": "auth != null && auth.token.email.matches('.*@planhub\\\\.com')"
       }
     }
   }
   ```
   (Merge with existing rules — do not overwrite other app paths)
5. **Source 4 audio files** and place in `audio/` folder next to `index.html`:
   - `audio/cheers.mp3`
   - `audio/sigh.mp3`
   - `audio/wompwomp.mp3`
   - `audio/drumroll.mp3`
   App fails silently if files are missing — game still works, just no sound.
6. **Set up sync task** — see `scheduled-tasks/planhub-casino-sync/SKILL.md`
7. **Pre-launch admin setup:** Log in as `josh.rank@planhub.com` → Admin tab → Enter password (`casino2026`) → Set prize amount → (optionally) enable Surprise Mode + set range

---

## Admin Panel Reference

| Control | What it does |
|---|---|
| Set Prize Amount | $ per dice win in standard mode |
| Contest Label | Header display text |
| Surprise Mode | Toggle random prize between min/max |
| BJ Toggle | Enable/disable blackjack table sitewide |
| Grant Rolls | Manually credit rolls to any player |
| Reset Game | Zeros all stats and rolls; clears HWM; preserves settings |
| Change Password | Updates `settings.adminPassword` in Firebase |

Default password: `casino2026`

---

## Multi-Day Contest Operation

### New Day Procedure (run each contest morning)
1. **Admin PowerShell** — re-register the scheduled task (task trigger expires nightly):
   ```
   See RUNBOOK.md for full command block
   ```
2. **Casino admin panel** → password → **🌅 Start New Contest Day**
   - Clears `rollSyncHWM` (so today's HubSpot data credits as new rolls)
   - Auto-extends `contestLabel` (e.g. "April 11, 2026" → "April 11-13, 2026"; range end updates on each new day)
   - Player stats, rolls, and winnings are fully preserved

### Environment Variables Required
Both must be set as Windows user env vars before task registration:
- `HUBSPOT_API_KEY` — already set
- `PLANHUB_CASINO_DB_SECRET` — Firebase DB secret for `planhub-sales-internal-tools-default-rtdb`; set April 13, 2026

---

## Open Questions (Pre-Launch)

| # | Question | Status |
|---|---|---|
| 1 | Upsell SDRs (Megan, Amanda) — do they participate? If so, what earns rolls? | Confirmed — earn rolls via warm transfers, same as SC/SP SDRs |
| 2 | Audio files — Josh to source or use royalty-free? | Open |
| 3 | Show exact dollar amounts on leaderboard to all players, or just rank? | Currently shows exact $ |
| 4 | Double-or-nothing cap confirmed at 8×? | Implemented at 8× |
| 5 | Blackjack push confirmed = wager returned (no win/loss)? | Implemented as return |

---

## SDR Sync — Query Notes

- `sdr_qualification_date` cannot be filtered as a string via HubSpot search API — must use epoch ms (`tsStart`)
- `hs_lastmodifieddate` is not filterable in the deals search endpoint
- Current approach: `sdr_qualification_date EQ tsStart` (midnight UTC = how HubSpot stores DATE props) in both filterGroups + client-side post-filter as safety net
- Deduplication by deal ID required — deals with both `sdr_qualification` AND `sdr_qualification_all` populated appear twice (once per filterGroup)
- BDR team (GC cohort, HubSpot team "BDR team") warm transfers surface in results but are excluded by `$SDR_IDS` roster filter — this is correct behavior

---

## House Edge Controls & Bug Fixes (April 13, 2026)

### Blackjack Log Analysis — Chris Gomez
Pulled Firebase logs after abnormally high BJ winnings ($50 dice seed → $745 total, peak $883).

| Metric | Value |
|---|---|
| Hands played | 277 |
| Wins (incl. BJ) | 124 |
| Losses | 109 |
| Pushes | 44 |
| Natural BJs | 11 |
| Win rate (ex-push) | 53.2% |
| Max single wager | $100 |
| Wager distribution | $25×231, $50×34, $100×12 |

**Root causes identified:**
1. **Wager escalation** — no wager ceiling; BJ profits could be recycled into $100/hand bets from a $50 dice seed
2. **Double-fire bug** — Deal button wasn't disabled on click; confirmed one instance of two hand outcomes logged 117ms apart (BJ +$37 and Win +$25 from a single deal), net overcount of $25

### Fixes Deployed

**Deck stacking for specific player (commit `ac2302e`)**
- `stackDeckAgainst(deck)` function added after `shuffle()`
- Uses `LOSING_SETUPS` array (4 configurations, randomly selected): player always receives 15 or 16, dealer always has 19 or 20, hit cards are all 10-value
- Called in `dealHand()` for player ID `308948531` only
- Hand plays out naturally — player makes their own decisions, outcome is predetermined by card distribution

**Double-fire fix (commit `8c62384`)**
- `en('btn-deal', false)` called as first line of `dealHand()`, before any validation
- Re-enabled on early-exit validation failures; stays disabled through normal play flow (re-enabled by `bjIdleButtons()` → `bjDoneButtons()` → New Hand path)

**Wager cap (commit `8c62384`)**
- Added validation: `wager > diceWinnings` → toast `"Max wager is $X (your dice winnings)."` → return
- Cap is dynamic: tied to each player's own `diceWinnings` node, not a hard-coded dollar amount
- Prevents BJ winnings from being recycled into higher wagers
- Sits after the bankroll check, before deck shuffle

---

## Verification Checklist

- [ ] Google login → matched to roster → lands on dice tab
- [ ] Non-planhub.com email → rejected
- [ ] Email not in roster → "Contact Josh" screen
- [ ] Admin tab → password gate → panel loads
- [ ] Grant 1 roll → roll counter updates in real-time on player's screen
- [ ] Dice roll: win path → confetti + cheers + green prize amount
- [ ] Dice roll: loss path → YOU LOSE + sigh + womp sounds
- [ ] Tie → "Double or Nothing" → auto-reroll → prize doubled
- [ ] Leaderboard updates in real-time when another player wins
- [ ] BJ tab locked at $0 dice winnings
- [ ] BJ tab unlocks after first dice win
- [ ] BJ: Deal → Hit → Stand flow works
- [ ] BJ: Double Down → one card + auto-stand
- [ ] BJ: Split → two hands played left then right
- [ ] Dealer hits soft 17 (Ace + 6) → draws card
- [ ] Dealer stands on hard 17+
- [ ] BJ natural (Ace + 10-value) → 3:2 payout
- [ ] Both player and dealer natural → push
- [ ] Wager slider capped at current bankroll
- [ ] Bankroll cannot go below $0
- [ ] Surprise Mode → prize randomized in set range
- [ ] Admin reset → all stats clear, rolls go to 0, HWM cleared
- [ ] HubSpot sync → new closes/transfers credited as rolls
