# Akatsuki Achievement System - Analysis & Action Plan

**Dates:** January 2026
**Investigator:** Claude Code
**Database:** Production MySQL (mysql-master01)

---

## ⚠️ IMPORTANT: Implementation Guidelines

**All changes outlined in this plan MUST follow proper development workflow:**

1. **✅ DO:** Create pull requests to the appropriate repositories for all code changes
2. **✅ DO:** Submit database schema changes as PRs to the `mysql-database` repository
3. **❌ NEVER:** Push directly to master/main branches
4. **❌ NEVER:** Directly edit the MySQL database in production
5. **✅ DO:** Follow the standard CI/CD deployment pipeline for all changes

**Database modifications must go through the mysql-database repository's migration system.**

**Sprint Workflow:**

- Implementation will be broken into sprints, each with specific deliverables
- At the end of each sprint, PRs will be created using `gh pr create -R osuAkatsuki/<repo>` for review
- Implementation will pause and wait for feedback before proceeding to the next sprint
- Each sprint should be independently testable and deployable

**Git Repository Guidelines:**

- **❌ NEVER:** Create PRs to `RealistikOsu/USSR` (upstream fork source)
- **✅ ALWAYS:** Create PRs to `osuAkatsuki/score-service` (our repository)
- **✅ DO:** Use `gh pr create -R osuAkatsuki/<repo>` to ensure correct target repository
- Verify git remote configuration before creating PRs

---

## Executive Summary

**Last Updated:** January 19, 2026

The Akatsuki achievement system has undergone major improvements through Sprints 1-4, addressing critical security and functionality issues. The system now has all 128 achievement definitions and secure code-based condition checking.

**Implementation Status (January 2026):**

- ✅ **Sprint 1 COMPLETE:** mode_str bug fixed, database migration deployed
- ✅ **Sprint 2 COMPLETE:** Decorator-based architecture for 96 existing achievements (security issue eliminated)
- ✅ **Sprint 3 COMPLETE:** 32 new achievement definitions added (128 total achievements)
- ✅ **Sprint 4 COMPLETE:** API/frontend mode support (akatsuki-api + hanayo)

**Remaining Work:**

- 🟡 **Sprint 5 PENDING:** Stat-based backfill (35 achievements, ~15-25K users)
- 🟡 **Sprint 6 PENDING:** Mod/combo backfill (15 achievements, ~10-15K users)
- 🟡 **Sprint 7 PENDING:** Star rating backfill (68 achievements, ~5-13K users)

**Key Findings (Initial Analysis - January 2026):**

1. ~~**Missing medals**~~: ✅ FIXED - Added 32 medals, now 128 total achievements
2. ~~**Security vulnerability**~~: ✅ FIXED - Replaced unsafe code execution with decorator-based architecture
3. **Zero achievements crisis**: 38,607 users (40%) still have zero achievements - backfilling needed (Sprints 5-7)
4. ~~**Broken display**~~: ✅ FIXED - API/frontend now support mode-specific achievements

**Current Status (Post-Sprint 4):**

- ✅ 128 achievements defined across all game modes (96 existing + 32 new)
- ✅ Decorator-based architecture (safe, type-safe, version-controlled)
- ✅ Mode-specific achievements working end-to-end (score-service → API → frontend)
- ✅ Relax/autopilot mode support (modes 0-8 fully supported)
- ✅ Production bugs fixed (SQL reserved keyword, race conditions)
- ✅ Performance optimizations (mode filtering, loop merging)
- ✅ Structured logging and observability
- 🔴 **REMAINING:** 38,607+ users still have ZERO achievements (requires backfilling - Sprints 5-7)

**Completed Work:**

- ✅ Added 32 new medals → **128 total achievements** (up from 96)
- ✅ Fixed API/frontend mode support (affected 27,948 users, 49% of achievement holders)
- ✅ Eliminated security vulnerability (unsafe execution replaced with safe code)
- ✅ Fixed critical production bugs (SQL syntax, race conditions)
- ✅ Performance improvements (60% reduction in achievement checks per score)

**Next Steps:** Begin Sprint 5 (stat-based backfill) to address the 40% zero-achievement coverage gap. See ACHIEVEMENT_BACKFILL_ANALYSIS.md for comprehensive backfill investigation and "Implementation Lessons Learned" section below for detailed lessons from Sprints 1-4.

## Current System Status

### Database Statistics

**Achievement Definitions:**

```
Total: 128 achievements in less_achievements table (as of 2026-01-19)
  - osu!std:  36 achievements (10 pass, 10 FC, 4 combo, 4 plays, 4 rank, 4 hits)
  - taiko:    27 achievements (8 pass, 8 FC, 3 hits, 4 rank, 4 hits)
  - catch:    27 achievements (8 pass, 8 FC, 3 hits, 4 rank, 4 hits)
  - mania:    27 achievements (8 pass, 8 FC, 3 hits, 4 rank, 4 hits)
  - all:      24 achievements (11 mod intro, 13 hush-hush secrets)

Coverage: 128/339 (38%) of official osu! medals
Missing: 211 medals (82 packs, 22 spotlights, 115 advanced secrets)
```

**Unlocks by Mode:**

```
Mode 4 (std_rx):    493,953 unlocks (54%) - Most popular
Mode 0 (std):       296,745 unlocks (32%)
Mode 8 (std_ap):     32,015 unlocks (3.5%)
Mode 3 (mania):      32,031 unlocks (3.5%)
Mode 7 (old std_ap): 28,472 unlocks (3.1%) - Historical data
Mode 2 (catch):       9,857 unlocks (1.1%)
Mode 1 (taiko):       8,886 unlocks (1.0%)
Mode 6 (catch_rx):    7,979 unlocks (0.9%)
Mode 5 (taiko_rx):    6,314 unlocks (0.7%)

Total: 916,252 unlocks
Unique users with achievements: 57,077
Users with ZERO achievements: 38,607+ (just vanilla modes, likely 50K-70K total)
──────────────────────────────────────────────
Achievement coverage: ~60% of eligible users
```

**⚠️ Coverage Gap:** 40% of users who have passed scores have ZERO achievements. This represents a critical gap in the achievement system. See ACHIEVEMENT_BACKFILL_ANALYSIS.md for detailed investigation and remediation plan.

### Achievement Categories

**Star Rating Achievements:**

- Rising Star (1-2★), Constellation Prize (2-3★), Building Confidence (3-4★)
- Insanity Approaches (4-5★), These Clarion Skies (5-6★), Above and Beyond (6-7★)
- Supremacy (7-8★), Absolution (8-9★), Event Horizon (9-10★), Phantasm (10-11★)

**Combo Achievements:**

- 500, 750, 1000, 2000 combo milestones

**Mod-Based Achievements (Mode-Agnostic):**

- Blindsight (Hidden), Rock Around The Clock (HardRock)
- Time And A Half (DoubleTime), Sweet Rave Party (Nightcore)
- Finality (SuddenDeath), Perfectionist (Perfect)
- Risk Averse (NoFail), Slowboat (HalfTime)
- Dial It Right Back (Easy), Burned Out (SpunOut)
- Are You Afraid Of The Dark? (Flashlight)

## Missing Medals from Official osu!

**Analysis Date:** 2026-01-16
**Sources:** https://inex.osekai.net/medals and https://github.com/ppy/osu-queue-score-statistics

**Akatsuki Currently Has:** 96 achievements
**Official osu! Has:** 128+ achievements
**Akatsuki is MISSING:** 32 achievements

### Complete Missing Medals Table

**Total Missing: 32 medals**

#### Category 1: 4th Tier Hit Count Medals (3 medals)

| Official ID | Proposed Akatsuki ID | Name                     | Mode      | Requirement                   | Backfillable |
| ----------- | -------------------- | ------------------------ | --------- | ----------------------------- | ------------ |
| 291         | 97                   | 30,000,000 Drum Hits     | Taiko (1) | Total hit count >= 30,000,000 | ✅ Yes       |
| 292         | 98                   | 20,000,000 Fruits Caught | Catch (2) | Total hit count >= 20,000,000 | ✅ Yes       |
| 293         | 99                   | 40,000,000 Keys Pressed  | Mania (3) | Total hit count >= 40,000,000 | ✅ Yes       |

**Implementation:** Easy - stat-based, same pattern as existing tier 3
**Benefit:** Completes the progression for dedicated players

#### Category 2: Rank Milestone Medals - Per Mode (16 medals)

**Note:** Official osu! has 4 GLOBAL rank medals (IDs 50-53). Akatsuki will implement per-mode (16 total).

| Official ID | Proposed Akatsuki ID | Name                  | Mode      | Requirement                          | Backfillable |
| ----------- | -------------------- | --------------------- | --------- | ------------------------------------ | ------------ |
| 50          | 100                  | Top 50,000: osu!std   | Std (0)   | rank_score_index <= 50,000 in mode 0 | ✅ Yes       |
| 51          | 101                  | Top 10,000: osu!std   | Std (0)   | rank_score_index <= 10,000 in mode 0 | ✅ Yes       |
| 52          | 102                  | Top 5,000: osu!std    | Std (0)   | rank_score_index <= 5,000 in mode 0  | ✅ Yes       |
| 53          | 103                  | Top 1,000: osu!std    | Std (0)   | rank_score_index <= 1,000 in mode 0  | ✅ Yes       |
| 50          | 104                  | Top 50,000: osu!taiko | Taiko (1) | rank_score_index <= 50,000 in mode 1 | ✅ Yes       |
| 51          | 105                  | Top 10,000: osu!taiko | Taiko (1) | rank_score_index <= 10,000 in mode 1 | ✅ Yes       |
| 52          | 106                  | Top 5,000: osu!taiko  | Taiko (1) | rank_score_index <= 5,000 in mode 1  | ✅ Yes       |
| 53          | 107                  | Top 1,000: osu!taiko  | Taiko (1) | rank_score_index <= 1,000 in mode 1  | ✅ Yes       |
| 50          | 108                  | Top 50,000: osu!catch | Catch (2) | rank_score_index <= 50,000 in mode 2 | ✅ Yes       |
| 51          | 109                  | Top 10,000: osu!catch | Catch (2) | rank_score_index <= 10,000 in mode 2 | ✅ Yes       |
| 52          | 110                  | Top 5,000: osu!catch  | Catch (2) | rank_score_index <= 5,000 in mode 2  | ✅ Yes       |
| 53          | 111                  | Top 1,000: osu!catch  | Catch (2) | rank_score_index <= 1,000 in mode 2  | ✅ Yes       |
| 50          | 112                  | Top 50,000: osu!mania | Mania (3) | rank_score_index <= 50,000 in mode 3 | ✅ Yes       |
| 51          | 113                  | Top 10,000: osu!mania | Mania (3) | rank_score_index <= 10,000 in mode 3 | ✅ Yes       |
| 52          | 114                  | Top 5,000: osu!mania  | Mania (3) | rank_score_index <= 5,000 in mode 3  | ✅ Yes       |
| 53          | 115                  | Top 1,000: osu!mania  | Mania (3) | rank_score_index <= 1,000 in mode 3  | ✅ Yes       |

**Implementation:** Moderate - requires global rank tracking per mode
**Benefit:** Motivates competitive players
**Decision:** Per-mode (top 50k in osu!std is separate from top 50k in taiko)

#### Category 3: Hush-Hush Secret Medals (13 medals)

| Official ID | Proposed Akatsuki ID | Name                              | Mode      | Requirement                                     | Backfillable |
| ----------- | -------------------- | --------------------------------- | --------- | ----------------------------------------------- | ------------ |
| 6           | 116                  | Don't let the bunny distract you! | All       | PFC Chatmonchy - Make Up! Make Up!              | ❌ No        |
| 15          | 117                  | S-Ranker                          | All       | Five S/SS ranks on different maps within 24h    | ❌ No        |
| 16          | 118                  | Most Improved                     | All       | D rank then A+ on same map within 24h           | ❌ No        |
| 17          | 119                  | Non-stop Dancer                   | Std (0)   | Pass Yoko Ishida - paraparaMAX I with 3M+ score | ⚠️ Maybe     |
| 38          | 120                  | Consolation Prize                 | All       | D rank with 100k+ score                         | ⚠️ Maybe     |
| 39          | 121                  | Challenge Accepted                | All       | Pass approved map with A+ rank                  | ⚠️ Maybe     |
| 40          | 122                  | Stumbler                          | All       | PFC with <85% accuracy                          | ⚠️ Maybe     |
| 41          | 123                  | Jackpot                           | All       | Pass map with identical score digits            | ⚠️ Maybe     |
| 42          | 124                  | Quick Draw                        | All       | First to submit score on ranked/qualified map   | ❌ No        |
| 43          | 125                  | Obsessed                          | All       | Retry map 100+ times, pass within 24h           | ❌ No        |
| 44          | 126                  | Nonstop                           | All       | PFC map with 8:41+ drain time                   | ⚠️ Maybe     |
| 45          | 127                  | Jack of All Trades                | All       | 5,000+ playcount in all four modes              | ✅ Yes       |
| 54          | 128                  | Twin Perspectives                 | Mania (3) | Pass mania map with 100+ combo                  | ⚠️ Maybe     |

**Implementation:** Moderate - each medal requires custom logic
**Benefit:** Adds fun/engagement factor, hidden until unlocked
**Note:** Requires `hidden` flag in database, frontend shows "???" until unlocked

#### Summary of Missing Medals

**Total new medals: 32**

- 3 hit count tier 4 (IDs 97-99)
- 16 rank milestones (IDs 100-115)
- 13 hush-hush (IDs 116-128)

**New total achievements: 128** (96 existing + 32 new)

**Backfill Potential:**

- ✅ **Immediately backfillable: 19 medals** (3 tier 4 hits + 16 rank milestones)
- ⚠️ **Potentially backfillable: 8 medals** (hush-hush if historical data exists)
- ❌ **Not backfillable: 5 medals** (require real-time tracking)

**Deferred Medals:**

- Beatmap pack completion medals (~20-40 medals) - deferred to future (requires pack infrastructure)
- Daily challenge medals - not applicable (Akatsuki doesn't have daily challenges)

### Code Architecture

**Primary Processing Location:** score-service

**Flow:**

```
Score Submission
  ↓
app/api/score_sub.py (lines 581-628)
  ↓
app/usecases/score.py::unlock_achievements()
  ↓
Evaluates conditions from cache (app/state/cache.py)
  ↓
app/usecases/user.py::unlock_achievement() [background job]
  ↓
INSERT INTO users_achievements
```

**Key Files:**

- `score-service/app/state/cache.py` - Achievement loading/caching
- `score-service/app/models/achievement.py` - Achievement dataclass
- `score-service/app/usecases/score.py` - Unlock logic
- `score-service/app/usecases/user.py` - Database operations
- `score-service/app/api/score_sub.py` - Score submission endpoint
- `score-service/app/constants/mode.py` - Mode enum definitions
- `akatsuki-api/app/v1/user_achievements.go` - Public API endpoint
- `mysql-database/migrations/0001_initial_db.up.sql` - Schema definition

## Issues Discovered

### 🔴 Critical Issues

#### 1. mode_str Tuple Bug (IndexError Risk)

**Location:** `score-service/app/constants/mode.py:9-18`

**Problem:**
The tuple has "std!ap" at index 7, but `Mode.STD_AP` enum value is 8, causing an IndexError when the mode is printed or logged.

**Impact:**

- `repr(Mode.STD_AP)` raises `IndexError: tuple index out of range`
- Latent bug that will crash if autopilot mode is printed/logged
- Introduced during mode 7 → mode 8 refactor (commit 216168f, April 21, 2024)

**Root Cause:**
When autopilot mode was refactored from 7 to 8, the `mode_str` tuple was not updated to include an entry at index 8.

**Fix:**
Add None for index 7 and move "std!ap" to index 8 in the tuple.

#### 2. API and Frontend Do NOT Support Mode-Specific Achievements

**Locations:**

- `akatsuki-api/app/v1/user_achievements.go:56-58`
- `hanayo/web/src/js/pages/profile.js` (initialiseAchievements function)

**Problem:**
While score-service correctly implements mode-specific achievement tracking, the public API and website frontend **completely ignore modes** when displaying achievements to users.

**Evidence from Code:**

**score-service (CORRECT):**

```python
# Lines 20-22 in app/usecases/score.py
user_achievements = await app.usecases.user.fetch_achievements(
    score.user_id,
    score.mode,  # <-- MODE IS PASSED
)
```

**akatsuki-api (BROKEN):**

```go
// Lines 56-58 in app/v1/user_achievements.go
err := md.DB.Select(&ids, `SELECT ua.achievement_id FROM users_achievements ua
INNER JOIN users ON users.id = ua.user_id
WHERE `+whereClause+` ORDER BY ua.achievement_id ASC`, param)
// NO MODE FILTER - returns ALL achievements across ALL modes!
```

**hanayo (BROKEN):**

```javascript
// hanayo/web/src/js/pages/profile.js
function initialiseAchievements() {
  api(
    "users/achievements" + (currentUserID == userID ? "?all" : ""),
    { id: userID }, // <-- NO MODE PARAMETER
    function (resp) {
      // Displays achievements without mode context
    }
  );
}
```

**Evidence from Database:**

27,948 users (49% of users with achievements) have unlocked achievements in multiple modes:

```sql
-- Example: User 1000 has "Rising Star" in both std and std_rx
user_id=1000, achievement_id=5, modes: 0,4

-- Example: User 1001 has "Supremacy" in 4 different modes
user_id=1001, achievement_id=7, modes: 0,4,7,8
```

**Impact:**

- **High severity** - Affects 27,948 users (49% of achievement holders)
- Achievements appear "flattened" across modes - users who earned "Rising Star" in both std and std_rx only see it once
- No way for users to see which modes they earned achievements in
- Profile page has mode selector, but achievements don't change when mode is switched
- Same issue affects `/api/v1/users/achievements` endpoint used by any external tools

**User Experience Issue:**

1. User plays std mode, earns "Rising Star" (1-2★ achievement)
2. User switches to std_rx mode, earns "Rising Star" again
3. User visits profile page
4. Achievement is shown once, no indication it was earned in both modes
5. User switches mode selector - achievements don't update

**Correct Behavior Should Be:**

1. API accepts `mode` parameter: `/api/v1/users/achievements?id=1000&mode=0`
2. Returns only achievements for that specific mode
3. Frontend passes current mode from mode selector
4. Achievements update when user switches modes
5. Each mode shows independent achievement progress

**Fix Required:**

**akatsuki-api changes:**

1. Add `mode` query parameter to UserAchievementsGET
2. Update SQL query to filter: `WHERE ... AND ua.mode = ?`
3. Return mode information in response
4. Maintain backward compatibility (default to mode 0 if not specified?)

**hanayo changes:**

1. Pass `mode` parameter in API call based on current mode selector
2. Reload achievements when mode is switched
3. Display mode-specific achievement progress

#### 3. Security Risk: Unsafe Code Execution

**Location:** `score-service/app/state/cache.py:15`

**Problem:**
Achievement conditions are stored as Python lambda strings in the database and executed using unsafe code evaluation at runtime. This allows arbitrary code to run if the database is compromised.

Example condition from database: `"(score.mods & 1 == 0) and 1 <= score.sr < 2 and mode_vn == 0"`

**Current Implementation:**

```python
# app/state/cache.py:15
achievements[row["id"]] = {
    "file": row["file"],
    "name": row["name"],
    "cond": eval(row["cond"], {"__builtins__": {}}),  # ⚠️ UNSAFE
}
```

**Impact:**

- High-risk security vulnerability
- No validation of condition syntax before storage
- Database compromise could lead to arbitrary code execution
- Backfilling 6.3M historical scores would mean running potentially compromised code millions of times
- Makes achievement logic difficult to test, review, and maintain

**Solution:** **Move achievement logic to code using decorators** (See Task 1.3)

- Eliminates database-driven code execution
- Makes conditions type-safe, testable, and version-controlled
- Allows code review for all achievement logic changes
- Required before safe backfilling can begin

### ⚠️ High Priority Issues

#### 4. Missing UNIQUE Constraint on users_achievements

**Location:** `mysql-database/migrations/0001_initial_db.up.sql:1260-1277`

**Problem:**
The `users_achievements` table lacks a UNIQUE constraint on `(user_id, achievement_id, mode)`, allowing users to unlock the same achievement multiple times for the same mode.

**Current Schema:**

```sql
CREATE TABLE `users_achievements` (
  `id` int NOT NULL AUTO_INCREMENT,
  `user_id` int NOT NULL,
  `achievement_id` int NOT NULL,
  `mode` int NOT NULL COMMENT 'TODO: this should have key',
  `created_at` int NOT NULL,
  PRIMARY KEY (`id`),
  KEY `user_id` (`user_id`),
  KEY `achievement_id` (`achievement_id`),
  KEY `time` (`created_at`)
) ENGINE=InnoDB
```

**Evidence:**
41 duplicate entries found in production:

```
user_id=1201, achievement_id=21, mode=4: unlocked 2 times
user_id=1201, achievement_id=73, mode=4: unlocked 2 times
user_id=7871, achievement_id=23, mode=4: unlocked 2 times
... (38 more)
```

**Impact:**

- Data integrity issue (0.004% of records)
- Inaccurate achievement counts
- Wasted database storage
- Potential display bugs in UI

**Fix:**
Add UNIQUE constraint after cleaning up existing duplicates.

#### 5. Missing Index on mode Column

**Location:** `mysql-database/migrations/0001_initial_db.up.sql:1263`

**Problem:**
Schema comment explicitly notes: `mode int NOT NULL COMMENT 'TODO: this should have key'`

The `mode` column is queried in WHERE clauses but lacks an index.

**Impact:**

- Performance degradation on 916K+ row table
- Slower achievement queries during score submission
- Inefficient query execution plan

**Fix:**
Add index on mode column.

### ⚠️ Medium Priority Issues

#### 6. Legacy achievements Table Unused

**Location:** `mysql-database/migrations/0001_initial_db.up.sql:51-65`

**Problem:**
The `achievements` table exists with 89 entries but is never queried by any service. All code uses `less_achievements` instead.

**Impact:**

- Wasted database storage
- Confusion for developers
- Dead code maintenance burden

**Recommendation:**

- Verify no external tools/scripts use this table
- Create backup before deletion
- Drop table via migration

#### 7. Background Job Failure Handling

**Location:** `score-service/app/api/score_sub.py:32`

**Problem:**
Achievement unlocking is scheduled as a background job. If the job fails silently, the achievement won't be persisted to the database.

**Impact:**

- Potential lost achievements if job queue crashes
- No retry mechanism
- No monitoring/alerting for failures

**Recommendation:**

- Add error handling and logging to background jobs
- Implement retry logic for failed unlocks
- Add monitoring/alerting for job failures

### 📚 Historical Data Notes

#### Mode 7 Mystery Solved

**Finding:** 28,472 achievements exist with `mode=7` in the database.

**Explanation:**
Mode 7 was the **original autopilot standard mode** before April 21, 2024. On that date, commit 216168f refactored autopilot from mode 7 to mode 8:

```
commit 216168f4d58a9589c32b962f3633dd59995fc938
Author: Josh Smith <cmyuiosu@gmail.com>
Date:   Sun Apr 21 10:22:25 2024 -0400

    Refactor to AP STD as gamemode 8 (from 7) (#88)
```

**Timeline:**

- Before April 21, 2024: `Mode.STD_AP = 7`
- After April 21, 2024: `Mode.STD_AP = 8`
- Last mode 7 unlock: April 21, 2024
- Zero mode 7 unlocks since refactor (verified)

**Conclusion:**
All 28,472 mode 7 achievements are **legitimate historical data** from when autopilot was mode 7. No migration or cleanup needed.

## Implementation Plan

**Priority Order:**

1. Add missing achievement definitions (32 new medals)
2. Refactor to decorator-based architecture (removes security risk, prerequisite for backfilling)
3. Backfill historical achievements (fixes 40% of users with zero achievements)
4. Fix API/frontend mode support (fixes display for 27,948 users)

### Phase 0: Add Missing Medals

#### Step 0.1: Database Migration - Add 32 New Achievements

**Action:**
Create migration to insert 32 new achievement definitions into `less_achievements` table:

- 3 hit count tier 4 medals (IDs 97-99)
- 16 rank milestone medals (IDs 100-115)
- 13 hush-hush medals (IDs 116-128)
- Add `hidden` boolean column for hush-hush medals (default false)
- Add index on `mode` column for performance
- Add UNIQUE constraint on `(user_id, achievement_id, mode)`

**Prerequisites:**

- Clean up 41 duplicate achievement entries (keep earliest timestamp)

**Testing:**

- Verify all 128 achievements present in database
- Verify unique constraint prevents duplicates
- Verify `hidden` column exists and defaults to false

### Phase 1: Critical Fixes

#### Step 1.1: Fix mode_str Tuple Bug

**File:** `score-service/app/constants/mode.py`

**Action:**
Update tuple to include entry at index 8:

- Add None for index 7 (unused - historical autopilot)
- Move "std!ap" to index 8

**Testing:**
Verify `repr(Mode.STD_AP)` returns "std!ap" without IndexError.

**Deployment:** Standard score-service deployment via CI/CD

#### Step 1.2: Add Mode Support to API and Frontend

**Files:**

- `akatsuki-api/app/v1/user_achievements.go`
- `hanayo/web/src/js/pages/profile.js`

**akatsuki-api Changes:**

1. Add optional `mode` query parameter to UserAchievementsGET
2. Update SQL query to include mode filter when provided
3. For backward compatibility, return all modes if mode not specified
4. Consider adding mode information to response JSON

**hanayo Changes:**

1. Update initialiseAchievements() to pass current mode from mode selector
2. Store achievements per mode in cache
3. Reload achievements when user switches modes
4. Update achievement display to show mode-specific progress

**Testing:**

1. Verify API returns only mode 0 achievements when `?mode=0` specified
2. Verify API returns all modes when mode parameter omitted (backward compat)
3. Test mode switching in hanayo updates achievement display
4. Verify 27,948 affected users now see correct mode-specific achievements

**Impact:**

- Fixes display for 27,948 users (49% of achievement holders)
- Allows users to see independent achievement progress per mode
- Aligns frontend with database's mode-specific design

#### Step 1.3: Replace eval() with Decorator-Based Architecture

**Files:**

- `score-service/app/achievements/` (new module)
- `score-service/app/state/cache.py` (refactor)

**Current Issue:**

- Achievement conditions stored as Python lambda strings in MySQL
- Evaluated using unsafe `eval()` at runtime
- Security risk: arbitrary code execution from database

**New Architecture:**

**1. Create decorator-based registration system:**

```python
# app/achievements/registry.py
from dataclasses import dataclass
from typing import Callable

@dataclass
class Achievement:
    id: int
    file: str
    name: str
    mode: int
    condition_func: Callable

achievement_registry: dict[int, Achievement] = {}

def achievement(id: int, file: str, name: str, mode: int):
    """Decorator to register achievement condition functions"""
    def decorator(func: Callable):
        achievement_registry[id] = Achievement(
            id=id,
            file=file,
            name=name,
            mode=mode,
            condition_func=func
        )
        return func
    return decorator
```

**2. Convert each achievement to a function:**

```python
# app/achievements/osu_skill_pass.py
from app.achievements.registry import achievement

@achievement(id=1, file="osu-skill-pass-1", name="Rising Star", mode=0)
def rising_star(score, mode_vn: int) -> bool:
    """Pass a 1-2★ map without NoFail in osu!std"""
    return (score.mods & 1 == 0) and 1 <= score.sr < 2 and mode_vn == 0

@achievement(id=2, file="osu-skill-pass-2", name="Constellation Prize", mode=0)
def constellation_prize(score, mode_vn: int) -> bool:
    """Pass a 2-3★ map without NoFail in osu!std"""
    return (score.mods & 1 == 0) and 2 <= score.sr < 3 and mode_vn == 0

# ... 94 more achievements
```

**3. Update condition evaluation:**

```python
# app/usecases/score.py
from app.achievements.registry import achievement_registry

async def unlock_achievements(score):
    for ach_id, achievement in achievement_registry.items():
        if achievement.mode != score.mode:
            continue

        # Call the registered function instead of eval()
        if achievement.condition_func(score, mode_vn=score.mode):
            await unlock_achievement(score.user_id, ach_id, score.mode)
```

**4. Database migration:**

- Keep `less_achievements` table for metadata
- Remove `cond` column (no longer needed)
- Achievement logic now lives in version-controlled code

**Benefits:**

- ✅ Eliminates security vulnerability
- ✅ Type-safe, IDE-friendly code
- ✅ Unit testable achievement conditions
- ✅ Code review for achievement logic changes
- ✅ Git history for condition modifications
- ✅ Can import same functions for backfill scripts
- ✅ Easier to add new achievements

**Migration Strategy:**

1. Create new achievements module with all 128 achievement functions (96 existing + 32 new)
2. Add comprehensive unit tests for each achievement condition
3. Run side-by-side validation (both systems) on staging for existing 96 achievements
4. Switch to decorator system once validated
5. Remove eval() code after confidence period

**Files to Create:**

- `app/achievements/__init__.py` - Module initialization
- `app/achievements/registry.py` - Decorator and registry infrastructure
- `app/achievements/osu_skill_pass.py` - 10 functions (1-10★)
- `app/achievements/osu_skill_fc.py` - 10 functions (1-10★)
- `app/achievements/osu_combo.py` - 4 functions
- `app/achievements/taiko_skill_pass.py` - 8 functions
- `app/achievements/taiko_skill_fc.py` - 8 functions
- `app/achievements/catch_skill_pass.py` - 8 functions
- `app/achievements/catch_skill_fc.py` - 8 functions
- `app/achievements/mania_skill_pass.py` - 8 functions
- `app/achievements/mania_skill_fc.py` - 8 functions
- `app/achievements/statistics.py` - 16 functions (4 plays + 12 hit counts)
- `app/achievements/hit_count_tier4.py` - 3 NEW functions (tier 4)
- `app/achievements/rank_milestones.py` - 16 NEW functions (per-mode)
- `app/achievements/mod_introductions.py` - 11 functions
- `app/achievements/hush_hush.py` - 13 NEW secret medal functions
- `tests/test_achievements_osu.py` - Unit tests for osu!std
- `tests/test_achievements_taiko.py` - Unit tests for taiko
- `tests/test_achievements_catch.py` - Unit tests for catch
- `tests/test_achievements_mania.py` - Unit tests for mania
- `tests/test_achievements_hush_hush.py` - Unit tests for secret medals

**Priority:** High (prerequisite for safe backfilling)

### Phase 2: Achievement Backfill

See ACHIEVEMENT_BACKFILL_ANALYSIS.md for detailed analysis.

**Backfill Phases:**

#### Step 2.1: Stat-Based Backfill (High Priority)

- 16 existing stat-based achievements (plays, hit counts tier 1-3)
- 3 NEW hit count tier 4 achievements
- 16 NEW rank milestone achievements (per-mode)
- **Total: 35 achievements**
- **Impact: 15,000-25,000 users gain achievements (~40% of zero-achievement users)**
- **Note:** Hush-hush medals NOT backfilled (require specific conditions during live play)

#### Step 2.2: Mod + Combo Backfill (High Priority)

- 11 mod introduction achievements
- 4 combo achievements
- **Total: 15 achievements**
- **Impact: Additional 10,000-15,000 users (~60-80% total coverage)**

#### Step 2.3: Star Rating Backfill (Medium Priority)

- 68 star-rating achievements (all pass/FC medals)
- Requires 6.3M+ API calls to performance-service
- **Impact: Remaining ~5,000-13,000 zero-achievement users (near-complete coverage)**
- **Note:** Rate-limited, careful planning required

**Result:** Reduce zero-achievement users from 38,607 (40%) to <3% of player base.

### Phase 3: Data Integrity

#### Step 3.1: Clean Up Duplicate Achievements

**Process:**

1. Identify duplicates (keeping earliest unlock)
2. Create backup table before deletion
3. Delete duplicate entries
4. Verify no duplicates remain

**Affected Users:** 27 users with 41 duplicate entries
**Data Loss:** None (keeping earliest unlock timestamp)
**Rollback Plan:** Restore from backup table

#### Step 3.2: Consolidate Mode 7 → Mode 8

See ACHIEVEMENT_BACKFILL_ANALYSIS.md for detailed process.

**Impact:** 4,134 users will see their historical autopilot achievements under mode 8

### Phase 4: Cleanup & Monitoring

#### Step 4.1: Remove Legacy achievements Table

**Prerequisites:**

- Verify no external tools/scripts use this table
- Create backup of table data

**Migration:** Create up/down migrations for dropping and restoring table

#### Step 4.2: Add Achievement Unlock Monitoring

**Metrics to Track:**

- Achievement unlocks per minute
- Background job failure rate
- Duplicate prevention (after constraint added)
- Condition evaluation errors
- Per-mode unlock distribution

**Implementation:**

- Datadog custom metrics
- Alert on unlock failure rate > 1%
- Dashboard for achievement health

#### Step 4.3: Add Logging for Background Jobs

**Location:** `score-service/app/api/score_sub.py`

**Enhancement:**
Add try/catch with logging for background job scheduling.

## Implementation Lessons Learned (Sprints 1-4)

**Status:** Sprints 1-4 completed and deployed to production (January 2026)

This section documents issues discovered during implementation that were not identified in the initial analysis, along with their fixes.

### Sprint 1: Database Migration + mode_str Fix

**Status:** ✅ Completed - PR #137 (score-service)

**Issues Discovered:**

1. **SQL Reserved Keyword Collision (CRITICAL PRODUCTION BUG)**
   - **Issue:** `desc` is a MySQL reserved keyword, causing syntax errors in production
   - **Location:** `score-service/app/state/cache.py` - Achievement loading query
   - **Error:** `asyncmy.errors.ProgrammingError: (1064, "You have an error in your SQL syntax... near 'desc FROM less_achievements'")`
   - **Fix:** Escape with backticks: `SELECT id, file, name, \`desc\` FROM less_achievements`
   - **Lesson:** Always escape potentially reserved keywords in SQL queries

### Sprint 2: Decorator Architecture - Existing 96 Achievements

**Status:** ✅ Completed - PR #138 (score-service)

**Issues Discovered:**

1. **Achievement Registry Design - Database Sync**
   - **Issue:** Need to load achievement metadata (id, name, desc) from database even though conditions are in code
   - **Reason:** Frontend displays achievement names/descriptions; can't duplicate in code
   - **Solution:** Hybrid approach - conditions in code, metadata in database
   - **Architecture:** Registry maps achievement files to condition functions; cache loads metadata from DB

2. **Import Organization Best Practice**
   - **Issue:** Need to explicitly import all achievement modules to trigger decorator registration
   - **Location:** `app/achievements/__init__.py`
   - **Solution:** Import all achievement modules at package level
   - **Lesson:** Decorator-based registries require explicit imports

### Sprint 3: Decorator Architecture - New 32 Achievements

**Status:** ✅ Completed - PR #139 (score-service)

**Issues Discovered:**

1. **Beatmap Ranked Status Validation Gap**
   - **Issue:** `twin_perspectives` achievement claims to check "ranked maps" but can't verify ranked status
   - **Location:** `app/achievements/hush_hush.py:twin_perspectives()`
   - **Root Cause:** `unlock_achievements()` receives only Score and Stats objects, not Beatmap object
   - **Impact:** Achievement checks `beatmap.has_leaderboard` at call site, which includes LOVED/QUALIFIED (not just RANKED/APPROVED)
   - **Current Fix:** Returns False with TODO comment explaining architectural limitation
   - **Lesson:** Achievement condition functions need access to all relevant data

2. **Achievement Organization - File Structure**
   - **Issue:** Initially scattered achievements by type (pass/FC), but mode-specific achievements are more cohesive
   - **Refactor:** Consolidated to by-mode organization (all std achievements in one place, etc.)
   - **Benefit:** Easier to find and maintain mode-specific achievements
   - **Lesson:** Organize by primary access pattern (mode) rather than secondary characteristics

3. **Performance Optimization - Loop Merging**
   - **Issue:** Two separate loops iterating `new_achievements` (logging loop + amplitude tracking loop)
   - **Impact:** Unnecessary iteration overhead on score submission hot path
   - **Fix:** Merged into single loop - PR #140
   - **Lesson:** Look for loop merging opportunities in hot paths

### Sprint 4: API + Frontend Mode Support

**Status:** ✅ Completed - PRs #92-95 (akatsuki-api), PRs #238-242 (hanayo)

**Issues Discovered:**

1. **Relax/Autopilot Mode Support Complexity**
   - **Issue:** Initial implementation only supported vanilla modes (0-3), missing relax (4-6) and autopilot (8)
   - **Solution (akatsuki-api):**
     - `fullMode` variable (0-8) for querying `users_achievements` table
     - `vanillaMode` variable (0-3, calculated as `fullMode % 4`) for filtering `less_achievements` table
     - Two-stage filtering: SQL query by fullMode, then in-memory filter by vanillaMode
   - **Solution (hanayo):**
     - `combinedMode` calculation: `favouriteMode + (preferRelax === 1 ? 4 : preferRelax === 2 ? 8 : 0)`
     - Reload achievements on both mode switcher AND rx/ap switcher
   - **Lesson:** Consider all mode variants when designing mode-aware features

2. **Frontend State Management - Multiple Reload Triggers**
   - **Issue:** Achievements need to reload on two different events: mode switch AND rx switch
   - **Solution:** Called `initialiseAchievements()` in both handlers
   - **Lesson:** Identify all state change triggers that affect displayed data

3. **UI State Bugs - Load More Button**
   - **Issue:** Using `.remove()` on "Load more" button caused it to disappear permanently when switching modes
   - **Fix:** Changed to `.hide()` and `.show()` - PR #241
   - **Lesson:** Prefer show/hide over add/remove for elements that may be toggled multiple times

4. **UI State Bugs - Locked Achievement Display**
   - **Issue:** `displayAchievements(8, true)` only showed achieved medals, hiding locked medals on mode switch
   - **Fix:** Changed to `displayAchievements(8, false)` - PR #240
   - **Lesson:** When reloading UI state, match the initial load behavior

5. **Race Condition - Empty State Duplication**
   - **Issue:** Calling `initialiseAchievements()` multiple times without clearing container caused duplicate empty state messages
   - **Fix:** Added `.empty()` before `.append()` on empty state path
   - **Lesson:** Functions designed for one-time initialization need cleanup logic when called repeatedly

6. **Race Condition - Click Handler Accumulation**
   - **Issue:** Multiple calls to `initialiseAchievements()` bound multiple click handlers to "Load more" button
   - **Fix:** Added `.off("click")` before `.on("click", ...)`
   - **Lesson:** Always clean up event handlers before rebinding

7. **Variable Naming Clarity**
   - **Issue:** Generic variable name `m` for mode value was unclear
   - **Fix:** Renamed to `fullMode` - PR #95
   - **Lesson:** Use descriptive variable names for complex concepts

### Additional Improvements Discovered

1. **Achievement Unlock Race Condition**
   - **Issue:** Two concurrent score submissions might try to unlock same achievement simultaneously
   - **Error:** `IntegrityError` on INSERT when UNIQUE constraint added
   - **Fix:** Changed to `INSERT IGNORE` - PR #143
   - **Benefit:** Idempotent operation, no error handling needed
   - **Lesson:** Use INSERT IGNORE for idempotent operations in concurrent systems

2. **Mode Filtering Performance Optimization**
   - **Issue:** Checking all 128 achievements per score submission
   - **Optimization:** Filter achievements by mode before checking unlock status - PR #143
   - **Benefit:** Reduces iterations from 128 to ~50 achievements per score (~60% reduction)
   - **Lesson:** Filter early in pipeline to reduce unnecessary processing

3. **Logging and Observability**
   - **Issue:** No logging when achievements are unlocked (makes debugging difficult)
   - **Fix:** Added structured logging with achievement details - PR #139
   - **Lesson:** Add logging for significant state changes

4. **Type Safety - Mypy Strict Mode**
   - **Issue:** Various type errors discovered during development
   - **Fix:** PR #141 - Fixed all mypy errors
   - **Lesson:** Enable mypy early and fix type errors as they arise

### Key Takeaways for Future Sprints

**Best Practices Established:**

1. **SQL Safety:** Always escape reserved keywords
2. **Concurrency:** Use idempotent operations (INSERT IGNORE, UPSERT)
3. **Performance:** Filter by mode/criteria early; merge loops in hot paths
4. **UI State:** Use show/hide over add/remove; clean up handlers before rebinding
5. **Observability:** Add structured logging for state changes
6. **Type Safety:** Keep mypy passing
7. **Naming:** Use descriptive names for complex concepts
8. **Architecture:** Organize by primary access pattern

**Issues Still Deferred:**

1. **Beatmap ranked status in achievements:** Need architectural change to pass Beatmap object
2. **Achievement backfilling:** Ready to begin Sprint 5

## Testing Plan

### Unit Tests

**Test File:** `score-service/tests/test_achievements.py`

**Test Cases:**

1. Ensure mode_str has entries for all Mode enum values
2. Verify unique constraint prevents duplicate unlocks
3. Test safe condition evaluation
4. Validate achievement condition syntax

### Integration Tests

**Test Scenarios:**

1. Score submission triggers achievement unlock
2. Duplicate unlock attempt is prevented by database
3. Mode-specific achievements only unlock for correct mode
4. Mod-based achievements unlock across all modes
5. Background job failure doesn't crash score submission

### Production Validation

**After Deployment:**

1. Monitor Datadog for achievement unlock rate (should match historical baseline)
2. Check error logs for IndexError exceptions (should be zero after mode_str fix)
3. Query for duplicate achievements (should be zero after constraint)
4. Verify performance improvement on mode-filtered queries

## Risk Assessment

| Issue                         | Risk Level | Impact if Unfixed                                                       | Mitigation                                               |
| ----------------------------- | ---------- | ----------------------------------------------------------------------- | -------------------------------------------------------- |
| mode_str IndexError           | High       | Service crash when autopilot mode is logged/printed                     | Fix in code (1 line change)                              |
| **API/Frontend ignore modes** | **High**   | **49% of users see wrong achievements**                                 | **Add mode parameter to API & frontend**                 |
| **Unsafe code execution**     | **High**   | **Arbitrary code execution if DB compromised; blocks safe backfilling** | **Replace with decorator-based architecture (2-3 days)** |
| Missing UNIQUE constraint     | Medium     | Continued duplicate achievements                                        | Add constraint after cleanup                             |
| Missing mode index            | Medium     | Slow queries, poor performance                                          | Add index via migration                                  |
| Legacy table                  | Low        | Wasted storage, confusion                                               | Drop table after verification                            |
| Background job failures       | Low        | Lost achievements (rare)                                                | Add error handling and monitoring                        |

## Rollback Plans

### If mode_str Fix Causes Issues

- Revert commit via CI/CD
- Deploy previous version of score-service
- No database changes required

### If Database Migration Fails

- Run down migration to restore previous schema
- Restore from backup if data loss occurred
- Re-evaluate cleanup strategy for duplicates

### If Performance Degrades

- Drop added indexes if causing write performance issues
- Analyze query execution plans
- Consider composite indexes instead

## Maintenance Recommendations

### Short Term (Next 2 weeks)

1. ✅ Fix mode_str tuple bug
2. ✅ Add validation for achievement conditions
3. ✅ Clean up 41 duplicate entries
4. ✅ Add database constraints

### Medium Term (Next 2 months)

1. Add achievement unlock monitoring
2. Improve background job error handling
3. Drop legacy achievements table

### Long Term (Next 6 months)

1. Create admin panel for achievement management
2. Add achievement preview/testing tools
3. Implement achievement versioning
4. Add achievement categories and progress tracking

## Appendix A: Database Schema

### users_achievements (Current)

```sql
CREATE TABLE `users_achievements` (
  `id` int NOT NULL AUTO_INCREMENT,
  `user_id` int NOT NULL,
  `achievement_id` int NOT NULL,
  `mode` int NOT NULL COMMENT 'TODO: this should have key',
  `created_at` int NOT NULL,
  PRIMARY KEY (`id`),
  KEY `user_id` (`user_id`),
  KEY `achievement_id` (`achievement_id`),
  KEY `time` (`created_at`)
) ENGINE=InnoDB AUTO_INCREMENT=916268
```

### users_achievements (Proposed)

```sql
CREATE TABLE `users_achievements` (
  `id` int NOT NULL AUTO_INCREMENT,
  `user_id` int NOT NULL,
  `achievement_id` int NOT NULL,
  `mode` int NOT NULL,
  `created_at` int NOT NULL,
  PRIMARY KEY (`id`),
  KEY `user_id` (`user_id`),
  KEY `achievement_id` (`achievement_id`),
  KEY `time` (`created_at`),
  KEY `mode` (`mode`),  -- NEW
  UNIQUE KEY `unique_user_achievement_mode` (`user_id`, `achievement_id`, `mode`)  -- NEW
) ENGINE=InnoDB
```

### less_achievements

```sql
CREATE TABLE `less_achievements` (
  `id` int PRIMARY KEY,
  `file` varchar(128),
  `name` varchar(128),
  `desc` varchar(128),
  `cond` varchar(64)  -- Python lambda string
) ENGINE=InnoDB
```

## Appendix B: Mode Reference

| Mode | Enum Name | Description     | Score Table  | Achievements        |
| ---- | --------- | --------------- | ------------ | ------------------- |
| 0    | STD       | Standard        | scores       | 296,745             |
| 1    | TAIKO     | Taiko           | scores       | 8,886               |
| 2    | CATCH     | Catch           | scores       | 9,857               |
| 3    | MANIA     | Mania           | scores       | 32,031              |
| 4    | STD_RX    | Standard Relax  | scores_relax | 493,953             |
| 5    | TAIKO_RX  | Taiko Relax     | scores_relax | 6,314               |
| 6    | CATCH_RX  | Catch Relax     | scores_relax | 7,979               |
| 7    | (legacy)  | Autopilot (old) | N/A          | 28,472 (historical) |
| 8    | STD_AP    | Autopilot       | scores_ap    | 32,015              |

## Appendix C: Achievement Condition Examples

**Star Rating Achievement:**

```
(score.mods & 1 == 0) and 1 <= score.sr < 2 and mode_vn == 0
# No NoFail mod, 1-2 star difficulty, standard mode
```

**Combo Achievement:**

```
500 <= score.max_combo and mode_vn == 0
# 500+ combo, standard mode
```

**Mod Achievement (Mode-Agnostic):**

```
(score.mods & 8 != 0) and score.passed
# Hidden mod enabled, score passed (any mode)
```

## Appendix D: Related Commits

- **216168f** (2024-04-21): Refactor to AP STD as gamemode 8 (from 7)
- Location: score-service/app/constants/mode.py
- Changed STD_AP from 7 → 8
- Did not update mode_str tuple (causing bug)

## Conclusion

The Akatsuki achievement system has **multiple critical issues** requiring immediate attention:

**Issue #1: 40% of Users Have Zero Achievements**

- 38,607+ users with passed scores have ZERO achievements
- This is a broken user experience, not just incomplete historical data
- Can be largely fixed within hours (see ACHIEVEMENT_BACKFILL_ANALYSIS.md)

**Issue #2: API/Frontend Ignore Modes**

- 27,948 users (49% of achievement holders) affected
- Mode selector on profile doesn't change achievement display
- Users see flattened achievements across all modes

**Priority Order:**

1. **🔴 Add 32 missing medal definitions (prerequisite for implementation)**
2. **🔴 Replace unsafe code execution with decorator-based architecture (PREREQUISITE for safe backfilling)**
3. **🔴 Achievement backfill Step 1 + 2 (URGENT, fixes 25,000-35,000 of 38,607 zero-achievement users)**
4. **🔴 Add mode support to API/frontend (critical, affects 27,948 users with mode-specific achievements)**
5. Fix mode_str IndexError (critical, simple fix)
6. Mode 7 → Mode 8 consolidation (high, affects 4,134 users)
7. Clean up duplicate achievements (high, affects data integrity)
8. Achievement backfill Step 3 (medium, fixes remaining ~5,000-13,000 zero-achievement users)
9. Drop legacy table (medium, reduces confusion)
10. Add monitoring (low, improves observability)

---

**Document Version:** 2.0
**Last Updated:** 2026-01-16
**Changes in v2.0:**

- Added comprehensive Missing Medals section with exact counts (32 medals)
- Updated Executive Summary with 4 critical issues
- Restructured Implementation Plan with phases and steps (no time estimates)
- Integrated content from detailed implementation plan

**Next Review:** After Phase 0 & 1 implementation
