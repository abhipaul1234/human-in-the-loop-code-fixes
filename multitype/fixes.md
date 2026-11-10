# multitype: fixes

A typing-battle web app. All game rules live in database functions on a hosted Postgres backend, written as numbered migrations. Migrations are append-only, so both the original AI-written SQL and its later fix are still in the repository. For every entry below, the **AI-written code** is the original migration and the **fixed code** is the later migration that replaced it. All entries are "Already fixed in later migrations".

## Logic & data

### Every 1v1 match was recorded as a draw (High)

**Why it's a problem.** The winner logic guarded each branch with `IF v_p1 IS NOT NULL AND v_p2 IS NOT NULL`. In PostgreSQL, `record IS NOT NULL` is true only when every column of the record is non-null. Here `left_at` is always NULL (it is the filter) and one of `user_id`/`guest_id` is always NULL, so the test was always false, no branch ran, and the match was saved as completed with no winner. Three "fix" migrations (005, 007, 008) changed other things before the real cause was found.

**AI-written code** (migration 004, carried through 005, 007 and 008)

```sql
SELECT * INTO v_p1 FROM public.match_players WHERE match_id = p_match_id AND player_slot = 1 AND left_at IS NULL;
SELECT * INTO v_p2 FROM public.match_players WHERE match_id = p_match_id AND player_slot = 2 AND left_at IS NULL;

IF v_p1 IS NOT NULL AND v_p2 IS NOT NULL THEN      -- always false
    IF v_p1.final_wpm > v_p2.final_wpm THEN
        v_winner_id := v_p1.id;
    -- ...
ELSIF v_p1 IS NOT NULL THEN
    v_winner_id := v_p1.id; -- Opponent left        (also unreachable)
```

**Fixed code** (migration 009)

```diff
-    IF v_p1 IS NOT NULL AND v_p2 IS NOT NULL THEN
+    IF v_p1.id IS NOT NULL AND v_p2.id IS NOT NULL THEN
+        -- both must carry valid metrics: a missing metric must never silently become a draw
+        IF v_p1.final_wpm IS NULL OR v_p1.final_accuracy IS NULL
+           OR v_p2.final_wpm IS NULL OR v_p2.final_accuracy IS NULL THEN
+            RETURN;
+        END IF;
-    ELSIF v_p1 IS NOT NULL THEN
+    ELSIF v_p1.id IS NOT NULL THEN
```

**Status.** Already fixed in later migrations

**Lesson.** Never test a whole record with `IS NOT NULL`; test its primary key.

### Honest results were rejected while the match was still "countdown" (Moderate)

**Why it's a problem.** The match only moved from `countdown` to `active` when a browser called `set_match_active`. A result submitted before that call arrived was rejected, so the winner function never ran with real scores. The first repair (008) accepted the countdown state but also added a 10-second extra window, which rejected honest submissions for 10 seconds after the timer ended.

**AI-written code** (migration 004, then 008)

```sql
IF v_match.status NOT IN ('active', 'completed') THEN       -- 004: 'countdown' is refused
    RAISE EXCEPTION 'Match is not active or completed.';
END IF;
```

```sql
-- 008: the "fix" adds 10 extra seconds
IF p_final_progress < 100 AND now() < (v_match.started_at + ((v_match.duration_seconds + 10) || ' seconds')::interval) THEN
    RAISE EXCEPTION 'Premature result submission. Timer has not expired and passage is not complete.';
END IF;
```

**Fixed code** (migration 009; 008 already accepts `countdown` in the status check)

```diff
-IF p_final_progress < 100 AND now() < (v_match.started_at + ((v_match.duration_seconds + 10) || ' seconds')::interval) THEN
+IF p_final_progress < 100
+   AND now() < v_match.started_at + (v_match.duration_seconds * interval '1 second') THEN
     RAISE EXCEPTION 'Premature result submission. Timer has not expired and passage is not complete.';
 END IF;
```

**Status.** Already fixed in later migrations

**Lesson.** Do not let the database's view of the game depend on a browser call arriving first; and test the edge of a time window from both sides.

### Pressing Ready again during the countdown reset the start time (Moderate)

**Why it's a problem.** `set_player_ready` kept working while the match was in `countdown`. Un-readying or re-readying inside the countdown recalculated the match state, which reset or cancelled the server-chosen start time that both players were already counting down to.

**AI-written code** (migration 001)

```sql
IF v_match_status NOT IN ('waiting', 'ready', 'countdown') THEN
    RAISE EXCEPTION 'Match has already progressed past the ready phase.';
END IF;
...
UPDATE public.match_players
SET ready_at = CASE WHEN p_is_ready THEN now() ELSE NULL END
WHERE id = v_player_id;
-- ...then the state machine recomputes status and started_at for every call
```

**Fixed code** (migration 018)

```diff
+    -- Both players are ready and the server start time is fixed: nothing may move it.
+    IF v_match_status = 'countdown' THEN
+        RETURN json_build_object('status', v_match_status, 'started_at', v_started_at);
+    END IF;
+
     UPDATE public.match_players
     SET ready_at = CASE WHEN p_is_ready THEN now() ELSE NULL END
     WHERE id = v_player_id;
```

**Status.** Already fixed in later migrations

**Lesson.** Once a state machine has fixed a start time, make that state read-only.

## Reliability & quality

### Players stuck in "already in a team room" after closing the tab (Moderate)

**Why it's a problem.** Creating or joining a team room refused anyone who still had an open membership row in a waiting, countdown or active room. Those rows were only cleaned up by a maintenance function that ran when somebody polled that same room, so a player who closed the tab or pressed Back stayed "inside" the abandoned room and could not start another one. The error looked random because an unrelated cleanup sometimes ran first.

**AI-written code** (migration 016)

```sql
-- One active team room per player
IF EXISTS (
    SELECT 1 FROM public.team_match_players p JOIN public.team_matches m ON m.id = p.match_id
     WHERE p.uid = v_uid AND p.left_at IS NULL AND m.status IN ('waiting', 'countdown', 'active')
) THEN
    RAISE EXCEPTION 'You are already in a team room. Leave it first.';
END IF;
-- cleanup of stale members (_team_maintain) only runs when someone polls THAT room
```

**Fixed code** (migration 023)

```sql
CREATE OR REPLACE FUNCTION public._team_release_open(p_uid UUID, p_keep UUID) RETURNS VOID ...
    -- before a player creates / joins / rematches, their earlier open memberships are resolved
    PERFORM public._team_maintain(v_id);
    IF v_status = 'waiting' AND v_p.ready_at IS NULL THEN
        PERFORM public._team_remove_player(v_id, v_p.id);          -- starting a new room leaves the old lobby
    ELSIF v_status IN ('waiting', 'countdown') AND v_p.last_heartbeat_at < now() - interval '15 seconds' THEN
        PERFORM public._team_remove_player(v_id, v_p.id);          -- stale seat released
    ELSIF v_status = 'active' AND v_p.last_heartbeat_at < now() - interval '15 seconds' ... THEN
        -- forfeited
    ELSIF v_status = 'active' THEN
        RAISE EXCEPTION 'You are still in a running team match. Finish or forfeit it first.';
    ...
```

(`team_create_room` and `team_join_room` now call `_team_release_open` first, under a per-player advisory lock.)

**Status.** Already fixed in later migrations

**Lesson.** Clean up stale state at the moment you check it, not only when someone happens to look at it.
