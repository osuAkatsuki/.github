# First Places Recalculation Scripts - Comprehensive Analysis

## Executive Summary

**Status: ✅ SCRIPTS WILL WORK CORRECTLY**

The recalculation scripts accurately implement the first-place logic used throughout the Akatsuki codebase. They will correctly fix historical inaccuracies and ensure the `scores_first` table is accurate.

---

## Script Logic Analysis

### Ranking Logic Verification

#### Vanilla Mode (rx=0):
- **Table**: `scores`
- **Sort**: `score DESC, time ASC`
- **Matches**: score-service uses `mode.sort` which returns `"score"` for vanilla modes (0-3)
- **Tiebreaker**: Earlier timestamp wins (same as score-service: line 264 in leaderboards.py)

#### Relax Mode (rx=1):
- **Table**: `scores_relax`
- **Sort**: `pp DESC, time ASC`
- **Matches**: score-service uses `mode.sort` which returns `"pp"` for relax modes (4-6)
- **Tiebreaker**: Earlier timestamp wins

#### Autopilot Mode (rx=2):
- **Table**: `scores_ap`
- **Sort**: `pp DESC, time ASC`
- **Matches**: score-service uses `mode.sort` which returns `"pp"` for autopilot (mode 8)
- **Tiebreaker**: Earlier timestamp wins

### Filtering Logic Verification

#### User Restriction Check:
```sql
users.privileges & 1 = 1
```
- **Meaning**: User has `USER_PUBLIC` privilege (not restricted)
- **Matches**: score-service uses `(u.privileges & 1 > 0 OR u.id = :user_id)` in rank calculation
- **Matches**: bancho-service-rs unban handler calls `recalculate_user_first_places` when unrestricted
- **✅ CORRECT**

#### Beatmap Status Check:
```sql
beatmaps.ranked > 1
```
- **Meaning**: Ranked (2), Approved (3), Qualified (4), or Loved (5)
- **Matches**: score-service `has_leaderboard` property returns `self.status >= RankedStatus.RANKED` (2)
- **Matches**: score-service only calls `handle_first_place` when `beatmap.has_leaderboard` is True
- **✅ CORRECT**

#### Score Completion Check:
```sql
scores.completed = 3
```
- **Meaning**: Score was successfully completed/passed
- **Matches**: score-service always uses `completed = 3` in leaderboard queries
- **✅ CORRECT**

---

## How First Places Work in Production

### Score Submission Flow (score-service):
1. User submits score
2. `find_score_rank` calculates rank using window function
3. If rank == 1 AND not restricted AND beatmap has leaderboard:
   - Call `handle_first_place`
   - DELETE existing first place for that beatmap/mode/rx
   - INSERT new first place entry

### User Ban Flow (bancho-service-rs):
1. User is banned via `peppy:ban` Redis event
2. `handlers/ban.rs` calls `scores::remove_first_places`
3. For each first place held by banned user:
   - Find next best eligible score from an unrestricted user
   - If found: `transfer_first_place` (UPDATE)
   - If not found: `remove_first_place` (DELETE)

### User Unban Flow (bancho-service-rs):
1. User is unbanned via `peppy:unban` Redis event
2. `handlers/unban.rs` calls `scores::recalculate_user_first_places`
3. For each of user's scores:
   - Check if it's better than current first place
   - If yes: `replace_first_place` (REPLACE INTO)

---

## What Can Go Wrong (Historical Issues)

### Scenario 1: Restricted User Holds First Place
**How it happens**: User got #1, then later got restricted, but first place wasn't removed

**Script handles it**:
- Step 1: Deletes if no valid replacement exists
- Step 2: Updates to correct holder if valid replacement exists

### Scenario 2: Missing First Place Entries
**How it happens**:
- Score-service bug or crash during submission
- Race condition between score submission and first place update
- Database inconsistency from manual operations

**Script handles it**:
- Step 3: Inserts missing entries by finding best score per beatmap/mode/rx

### Scenario 3: Wrong User Has First Place
**How it happens**:
- User A had #1, user B got a better score, but first place didn't update
- Restricted user's score is still recorded as #1

**Script handles it**:
- Step 2: Updates to correct holder based on current best score
- Step 3: Inserts if missing entirely

---

## Database Schema Verification

### scores_first Table:
```sql
CREATE TABLE `scores_first` (
  `beatmap_md5` char(32) NOT NULL,
  `mode` tinyint(1) NOT NULL,          -- vanilla mode (0-3)
  `rx` tinyint(1) NOT NULL,            -- 0=vanilla, 1=relax, 2=autopilot
  `scoreid` bigint NOT NULL,
  `userid` int NOT NULL,
  PRIMARY KEY (`beatmap_md5`,`mode`,`rx`),
  UNIQUE KEY `scores_first_scoreid_uindex` (`scoreid`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1;
```

**Primary Key**: `(beatmap_md5, mode, rx)` - ensures one first place per beatmap/mode/relax combination
**Unique Key**: `scoreid` - ensures a score can only be #1 in one slot

Scripts respect both constraints ✅

---

## Edge Cases Handled

### ✅ Tiebreaker (Same Score/PP):
- Scripts use `ORDER BY {sort_column} DESC, time ASC`
- Matches production: earlier timestamp wins
- Window function `ROW_NUMBER() OVER (PARTITION BY beatmap_md5, play_mode ORDER BY ...)`

### ✅ Multiple Modes Per Beatmap:
- Scripts partition by `beatmap_md5` AND `play_mode`
- Each mode (std/taiko/ctb/mania) gets independent first place

### ✅ Multiple Relax Variations:
- Scripts loop through rx values (0, 1, 2)
- Each relax type gets independent first place

### ✅ Idempotency:
- Script can be run multiple times safely
- Step 1 (DELETE): Only removes definitely wrong entries
- Step 2 (UPDATE): Updates to current best
- Step 3 (INSERT): Only inserts missing entries

---

## Cross-Service Consistency Check

### score-service/app/repositories/leaderboards.py:212
```python
# Rank calculation
AND (u.privileges & 1 > 0 OR u.id = :user_id)
ORDER BY s.{sort_column} DESC, s.time ASC
```

### score-service/app/usecases/leaderboards.py:124
```python
sort_column=mode.sort  # "score" for vanilla, "pp" for relax/ap
```

### bancho-service-rs/src/usecases/scores.rs:77
```rust
scoring.is_ranked_higher_than(&score, &current_first_place)
// Uses score/pp comparison with timestamp tiebreaker
```

### Recalculation Script:
```sql
ORDER BY scores.score DESC, scores.time ASC  -- vanilla
ORDER BY scores_relax.pp DESC, scores_relax.time ASC  -- relax
```

**All match** ✅

---

## Potential Issues & Mitigation

### ⚠️ Script Does NOT Handle:
1. **Scores with unranked mods**: Scripts don't filter by mods, but production also doesn't - first place is independent of mods
2. **Beatmap status changes**: If a ranked map becomes unranked, existing first places remain (but this is expected behavior)

### ✅ Script DOES Handle:
1. Restricted users holding first places
2. Missing first place entries
3. Wrong users holding first places
4. All game modes (std/taiko/ctb/mania)
5. All relax variations (vanilla/relax/autopilot)

---

## Performance Considerations

### Window Function Optimization:
```sql
ROW_NUMBER() OVER (
    PARTITION BY beatmap_md5, play_mode
    ORDER BY score DESC, time ASC
) AS rn
```
- Modern MySQL (8.0+) optimizes this well
- More efficient than correlated subqueries
- Uses indexes on (beatmap_md5, play_mode, score, time)

### Index Requirements:
Scripts will benefit from:
- `scores.beatmap_md5, scores.play_mode, scores.completed`
- `scores_relax.beatmap_md5, scores_relax.play_mode, scores_relax.completed`
- `scores_ap.beatmap_md5, scores_ap.play_mode, scores_ap.completed`
- `users.id, users.privileges`
- `beatmaps.beatmap_md5, beatmaps.ranked`

**Recommendation**: Run with EXPLAIN to verify index usage before production run

---

## Testing Recommendations

### Before Running on Production:

1. **Dry Run Analysis**:
   ```sql
   -- Count entries that would be deleted
   SELECT COUNT(*) FROM scores_first sf
   INNER JOIN users ON users.id = sf.userid
   LEFT JOIN valid_scores ON ...
   WHERE users.privileges & 1 = 0 AND valid_scores.beatmap_md5 IS NULL;
   ```

2. **Sample Verification**:
   - Pick 10-20 beatmaps manually
   - Check current first place holder
   - Verify script would assign correct holder
   - Compare with live leaderboards

3. **User 1001 Test** (Already Done ✅):
   - Your account had accurate recalculation
   - Verified scores match expectations

4. **Performance Test**:
   - Run on staging database first
   - Monitor execution time
   - Check for table locks

---

## Deployment Recommendations

### Safe Deployment Steps:

1. **Backup First**:
   ```sql
   CREATE TABLE scores_first_backup AS SELECT * FROM scores_first;
   ```

2. **Run Vanilla Script First**:
   - Test on rx=0 only (largest dataset)
   - Verify results before continuing

3. **Monitor Production**:
   - Watch for first place announcements in-game
   - Check user reports
   - Verify leaderboards match scores_first

4. **Rollback Plan**:
   ```sql
   TRUNCATE scores_first;
   INSERT INTO scores_first SELECT * FROM scores_first_backup;
   ```

---

## Conclusion

### ✅ Scripts are Safe to Run

The recalculation scripts:
1. **Use correct sorting logic** (score for vanilla, pp for relax/ap)
2. **Use correct tiebreaker logic** (time ASC)
3. **Filter users correctly** (privileges & 1 = 1)
4. **Filter beatmaps correctly** (ranked > 1)
5. **Handle all edge cases** (missing entries, restricted users, wrong holders)
6. **Match production behavior** (consistent with score-service and bancho-service-rs)
7. **Are idempotent** (safe to run multiple times)

### Recommendation: **APPROVE FOR PRODUCTION USE**

The scripts will accurately fix historical inaccuracies and ensure the `scores_first` table matches the actual best scores from unrestricted users on ranked/approved/loved beatmaps.
