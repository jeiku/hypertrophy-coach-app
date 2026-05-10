# CANONICAL DATA AND PLAN GENERATION CONTRACT

**Status:** Implementation-freeze candidate (rev 3)  
**Source of truth:** `PRD.md` v0.4 Canonical Draft

## 0) Scope
- Deterministic MVP Core (no AI dependency).
- Planned data (`PlanVersion` tree) is separate from execution data (`WorkoutSession`/`SetLog`).
- Regeneration creates a new plan version and never rewrites historical execution logs.

---

## 1) Data Model (delta-resolved)

### 1.1 Key entity fixes
- `body_weight_log` unique index: `(user_id, logged_on) WHERE deleted_at IS NULL`.
- `post_workout_check_in` has no `user_id`; RLS is dependent through `workout_session.user_id`.
- `set_log` uniqueness must avoid same-exercise collisions after substitutions.

### 1.2 SetLog canonical keys
`set_log` columns include:
- `id uuid PK`
- `user_id uuid not null`
- `workout_session_id uuid not null`
- `exercise_instance_id uuid null`
- `set_prescription_id uuid null`
- `exercise_id uuid not null`
- `set_index smallint not null`
- `set_type set_type not null`
- `slot_occurrence smallint not null default 1` (disambiguates repeated same exercise in one session)
- `mutation_id uuid not null`
- `deleted_at timestamptz null`

**Unique indexes**
1. `uq_setlog_idempotent` on `(user_id, mutation_id)` where `deleted_at is null`
2. `uq_setlog_logical_slot` on `(workout_session_id, slot_occurrence, set_type, set_index)` where `deleted_at is null`

### 1.3 Workout session idempotency keys
Add table `workout_session_mutation`:
- `id uuid PK`
- `user_id uuid not null`
- `mutation_id uuid not null`
- `operation text not null` (`start|complete|partial|resume|abandon|skip`)
- `workout_session_id uuid not null`
- `request_hash text not null`
- `response_snapshot jsonb not null`
- `created_at timestamptz not null default now()`
- unique `(user_id, mutation_id)`

Used for start/complete/partial idempotent replay.

---

## 2) RLS/Security (fully explicit)

## 2.1 Soft delete write policy
- **Direct client DELETE denied** on all soft-deletable user tables.
- Soft delete only via service-role RPC/Edge Function (`soft_delete_*`) that validates ownership and writes `deleted_at`.

## 2.2 Restricted direct UPDATE policy
Direct client UPDATE is **denied** for:
- `plan_version`
- `workout_session`
- `set_log`
- `recommendation`
- `processed_mutation`
- `workout_session_mutation`

These are write-only through service-role Edge Functions/RPCs.

## 2.3 Table-by-table write paths

### Direct client INSERT/UPDATE allowed (own row only)
- `user_profile` (except protected fields controlled server-side)
- `goal_plan` (manual targets only; system-calculated fields may be overridden by service only)
- `equipment_profile`
- `equipment_profile_item`
- `equipment_calendar_entry`
- `user_exercise_preference`
- `user_limitation`
- `body_weight_log`
- `food_log`
- `saved_meal`
- `notification_preference`

### Service-role only writes
- `plan_version`, `workout_day`, `exercise_instance`, `set_prescription`
- `workout_session`, `set_log`, `post_workout_check_in`
- `recommendation`
- `processed_mutation`, `workout_session_mutation`
- `exercise` catalog

## 2.4 Dependent RLS definitions
- `post_workout_check_in` SELECT allowed only when exists joined `workout_session` with `workout_session.user_id = auth.uid()`.
- `workout_day`, `exercise_instance`, `set_prescription` SELECT via parent plan ownership chain.

## 2.5 Tamper guards
- mutation tables reject client upserts.
- service function recomputes ownership of every referenced ID (never trust client ownership claims).

---

## 3) Plan Generation Contract (corrected)

## 3.1 Request
```json
{
  "mutationId": "uuid",
  "generateMode": "initial|regenerate",
  "basePlanVersionId": "uuid|null",
  "profileSnapshot": {
    "experienceLevel": "beginner|intermediate",
    "goal": "bulk|cut|recomp",
    "daysPerWeek": 4,
    "preferredTrainingDays": [1,2,4,6],
    "sessionLengthMin": 60,
    "timezone": "America/Chicago"
  },
  "focusMuscles": ["chest"],
  "preferredExerciseIds": [],
  "dislikedExerciseIds": [],
  "hardBlockExerciseIds": [],
  "regenerationReason": "user_request"
}
```

Server hydrates day-equipment from `equipment_calendar_entry + equipment_profile_item`; **client cannot supply equipment contents**.

## 3.2 Response
```json
{
  "planVersion": {"id":"uuid","versionNumber":7,"status":"active","generatorVersion":"rules-v1.2.0"},
  "workoutDays": [
    {
      "id":"uuid",
      "dayIndex":1,
      "label":"Upper A",
      "weekdayIso":1,
      "estimatedDurationMin":58,
      "slots":[
        {
          "slotIndex":1,
          "movementIntent":"horizontal_push",
          "exerciseInstanceId":"uuid",
          "exerciseId":"uuid",
          "sets":[
            {"setPrescriptionId":"uuid","setType":"working","setIndex":1,"repMin":8,"repMax":12}
          ]
        }
      ]
    }
  ],
  "validationWarnings": [],
  "explanation": "..."
}
```

---

## 4) API Contracts (all MVP writes require mutationId)

### Common rules for all write endpoints
- Required request fields: `mutationId`, `clientTimestamp`, `ifMatchVersion` (optimistic lock token when resource has version).
- Idempotency: keyed by `(user_id, mutation_id)` in `processed_mutation` or `workout_session_mutation`.
- Duplicate same hash => `200` replay previous response.
- Duplicate different hash => `409 MUTATION_ID_REUSE_CONFLICT`.
- Validation errors => `422 INPUT_VALIDATION_ERROR`.
- Optimistic lock mismatch => `409 VERSION_CONFLICT` with current canonical payload.

### 4.1 `PUT /v1/profile`
Request:
```json
{"mutationId":"uuid","clientTimestamp":"2026-05-10T10:00:00Z","ifMatchVersion":3,"profile":{}}
```
Response:
```json
{"data":{"profile":{},"version":4}}
```

### 4.2 `PUT /v1/equipment/profiles/{id}/items` (full replace)
Request:
```json
{"mutationId":"uuid","clientTimestamp":"...","ifMatchVersion":5,"items":[{"equipmentType":"dumbbell","quantity":2,"availability":"available","metadata":{"maxPerHandKg":30}}]}
```
Behavior:
- service verifies profile ownership
- full replace transactional: missing old rows soft-deleted
Response includes new version + normalized items.

### 4.3 `POST /v1/plans/generate`
Request/response per section 3.

### 4.4 `POST /v1/plans/{id}/regenerate`
Request includes `mutationId`, reason, optional overrides.
Response includes new active plan + archived prior plan IDs.
Optimistic lock on base plan version required.

### 4.5 `POST /v1/workout-sessions/start`
Request:
```json
{"mutationId":"uuid","clientTimestamp":"...","scheduledForDate":"2026-05-10","workoutDayId":"uuid","planVersionId":"uuid"}
```
Response:
```json
{"data":{"workoutSessionId":"uuid","status":"in_progress","version":1}}
```
Idempotent by `workout_session_mutation`.

### 4.6 `POST /v1/workout-sessions/{id}/set-logs`
Request:
```json
{"mutationId":"uuid","clientTimestamp":"...","ifMatchVersion":2,"setLog":{"exerciseId":"uuid","exerciseInstanceId":"uuid|null","setPrescriptionId":"uuid|null","slotOccurrence":1,"setType":"working","setIndex":1,"repsCompleted":10,"loadValue":50,"loadUnit":"kg","rir":2,"isSkipped":false,"loggedAt":"..."}}
```
Response: canonical set log row + session version increment.

### 4.7 `POST /v1/workout-sessions/{id}/complete`
Request:
```json
{"mutationId":"uuid","clientTimestamp":"...","ifMatchVersion":7,"completionType":"full|partial","checkIn":{"fatigueRating":3,"sorenessRating":2,"painFlag":false,"formBreakdownFlag":false}}
```
Response: session status completed/partial + recommendation trigger flags.
Idempotent via `workout_session_mutation`.

### 4.8 `POST /v1/workout-sessions/{id}/substitute`
Request includes `mutationId`,`ifMatchVersion`,`exerciseInstanceId`,`replacementExerciseId|null`.
Response includes updated slot mapping + load guidance + new version.

### 4.9 `POST /v1/recommendations/{id}/accept|ignore`
Request requires `mutationId`,`ifMatchVersion`.
Response includes updated recommendation status.

### 4.10 `POST/PATCH/DELETE /v1/nutrition/logs`
- Write endpoints require `mutationId` and optimistic token for patch/delete.
- Delete endpoint performs soft delete via service function.

### 4.11 `POST /v1/account/deletion-request`
Request includes `mutationId`.
Response: pending deletion timestamp + hard-delete-by timestamp.

---

## 5) Generator Templates Normalization

All slots use canonical `movement_intent` values only.

### 5.1 Mappings (no shorthand ambiguity)
- `squat` -> `knee_dominant_squat_pattern`
- `hinge` -> `hip_dominant_hinge_pattern`
- `arms` -> split into `elbow_flexion` + `elbow_extension`
- `calves/core` -> split into `calf_plantarflexion` + `core_anti_extension_or_rotation`
- `rear/lat accessory` -> `horizontal_pull` or `vertical_pull` based on exercise tag

### 5.2 Canonical templates (example 4-day)
Upper A: `horizontal_push`, `horizontal_pull`, `vertical_push`, `vertical_pull`, `elbow_flexion`, `elbow_extension`  
Lower A: `knee_dominant_squat_pattern`, `hip_dominant_hinge_pattern`, `single_leg`, `calf_plantarflexion`, `core_anti_extension_or_rotation`

(Equivalent normalized templates are defined for 2/3/5/6-day splits.)

---

## 6) Offline Local Draft + Sync (strengthened)

### 6.1 Local set draft schema
```ts
type LocalSetLogDraft = {
  localSetLogId: string;
  mutationId: string;
  workoutSessionId: string;
  exerciseId: string;
  exerciseInstanceId: string | null;
  setPrescriptionId: string | null;
  slotOccurrence: number;
  setType: 'working'|'warmup';
  setIndex: number;
  repsCompleted: number | null;
  loadValue: number | null;
  loadUnit: 'kg'|'lb'|'bodyweight'|'assisted_level'|null;
  rir: number | null;
  isSkipped: boolean;
  loggedAt: string;
  syncState: 'pending'|'acked'|'conflict';
};
```

### 6.2 Retry order
1. create/start session
2. substitutions (if any)
3. set-log mutations in original client order
4. check-in mutation
5. complete/partial mutation

### 6.3 Merge/conflict rules
- append-only set logs win (preserve all non-duplicate mutation IDs).
- exact same mutation replay => ack with prior response.
- stale `ifMatchVersion` + divergent payload => `409 VERSION_CONFLICT` + canonical server state.
- client must rebase pending queue on returned canonical version.

### 6.4 Complete/partial idempotency
- completion mutations keyed in `workout_session_mutation`.
- duplicate completion mutation does not re-run recommendation side effects.

---

## 7) Exercise Seed Catalog (full 50 records)

All records are seed-ready and include every required field.

|slug|movement_intent|primary|secondary|equipment_required|experience_min|beginner_friendly|fatigue_cost|setup_complexity|substitution_tags|setup_cue|execution_cue|common_mistake_cue|safety_note|is_active|
|---|---|---|---|---|---|---:|---:|---:|---|---|---|---|---|---|
|goblet-squat|knee_dominant_squat_pattern|quads|glutes/core|dumbbell|beginner|true|3|2|squat,dumbbell|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|bodyweight-squat|knee_dominant_squat_pattern|quads|glutes|bodyweight|beginner|true|2|1|squat,no_equipment|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|back-squat|knee_dominant_squat_pattern|quads|glutes/core|barbell|intermediate|false|5|4|squat,barbell|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|front-squat|knee_dominant_squat_pattern|quads|core/glutes|barbell|intermediate|false|5|4|squat,barbell|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|leg-press|knee_dominant_squat_pattern|quads|glutes|machine_selectorized|beginner|true|4|2|squat,machine|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|romanian-deadlift-dumbbell|hip_dominant_hinge_pattern|hamstrings|glutes/back|dumbbell|beginner|true|4|2|hinge,dumbbell|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|romanian-deadlift-barbell|hip_dominant_hinge_pattern|hamstrings|glutes/back|barbell|intermediate|false|5|3|hinge,barbell|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|trap-bar-deadlift|hip_dominant_hinge_pattern|glutes|quads/hamstrings|trap_bar|intermediate|false|5|4|hinge,trap_bar|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|hip-thrust-barbell|hip_dominant_hinge_pattern|glutes|hamstrings|barbell,bench|beginner|true|4|3|hinge,glute|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|glute-bridge-bodyweight|hip_dominant_hinge_pattern|glutes|hamstrings|bodyweight|beginner|true|2|1|hinge,no_equipment|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|walking-lunge-dumbbell|single_leg|quads|glutes/core|dumbbell|beginner|true|4|2|single_leg,dumbbell|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|split-squat-dumbbell|single_leg|quads|glutes|dumbbell|beginner|true|4|2|single_leg,dumbbell|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|bulgarian-split-squat|single_leg|quads|glutes/core|dumbbell,bench|intermediate|false|5|3|single_leg,bench|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|seated-leg-curl-machine|hip_dominant_hinge_pattern|hamstrings|calves|machine_selectorized|beginner|true|3|2|hamstring,machine|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|lying-leg-curl-machine|hip_dominant_hinge_pattern|hamstrings|calves|machine_selectorized|beginner|true|3|2|hamstring,machine|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|leg-extension-machine|knee_dominant_squat_pattern|quads|none|machine_selectorized|beginner|true|3|1|quad,machine|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|standing-calf-raise|calf_plantarflexion|calves|none|bodyweight,dumbbell|beginner|true|2|1|calves|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|seated-calf-raise|calf_plantarflexion|calves|none|machine_selectorized|beginner|true|2|1|calves,machine|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|push-up|horizontal_push|chest|triceps/front_delts|bodyweight|beginner|true|3|1|push,bodyweight|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|dumbbell-bench-press-flat|horizontal_push|chest|triceps/front_delts|dumbbell,bench|beginner|true|4|2|push,dumbbell|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|barbell-bench-press-flat|horizontal_push|chest|triceps/front_delts|barbell,bench|intermediate|false|5|3|push,barbell|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|dumbbell-incline-press|horizontal_push|chest|front_delts/triceps|dumbbell,bench|beginner|true|4|2|push,dumbbell|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|machine-chest-press|horizontal_push|chest|triceps/front_delts|machine_selectorized|beginner|true|3|1|push,machine|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|cable-fly|horizontal_push|chest|front_delts|cable|beginner|true|3|2|chest_isolation,cable|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|dumbbell-shoulder-press-seated|vertical_push|front_delts|triceps/upper_chest|dumbbell,bench|beginner|true|4|2|vertical_push,dumbbell|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|barbell-overhead-press|vertical_push|front_delts|triceps/core|barbell|intermediate|false|5|3|vertical_push,barbell|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|machine-shoulder-press|vertical_push|front_delts|triceps|machine_selectorized|beginner|true|3|1|vertical_push,machine|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|dumbbell-lateral-raise|lateral_deltoid_isolation|side_delts|traps|dumbbell|beginner|true|2|1|delts_isolation|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|cable-lateral-raise|lateral_deltoid_isolation|side_delts|traps|cable|beginner|true|2|2|delts_isolation,cable|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|triceps-pushdown-cable|elbow_extension|triceps|none|cable|beginner|true|2|1|triceps,cable|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|overhead-triceps-extension-dumbbell|elbow_extension|triceps|none|dumbbell|beginner|true|3|1|triceps,dumbbell|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|close-grip-push-up|elbow_extension|triceps|chest/front_delts|bodyweight|beginner|true|3|1|triceps,bodyweight|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|one-arm-dumbbell-row|horizontal_pull|back_lats_upperback|biceps/rear_delts|dumbbell,bench|beginner|true|4|2|row,dumbbell|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|chest-supported-row-dumbbell|horizontal_pull|back_lats_upperback|biceps/rear_delts|dumbbell,bench|beginner|true|3|2|row,dumbbell|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|barbell-row|horizontal_pull|back_lats_upperback|biceps/rear_delts|barbell|intermediate|false|5|3|row,barbell|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|seated-cable-row|horizontal_pull|back_lats_upperback|biceps/rear_delts|cable|beginner|true|3|2|row,cable|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|lat-pulldown-wide|vertical_pull|back_lats_upperback|biceps|cable,machine_selectorized|beginner|true|3|1|vertical_pull,lat|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|lat-pulldown-neutral|vertical_pull|back_lats_upperback|biceps|cable,machine_selectorized|beginner|true|3|1|vertical_pull,lat|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|assisted-pull-up|vertical_pull|back_lats_upperback|biceps|pullup_bar,machine_selectorized|beginner|true|4|2|vertical_pull,pullup|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|pull-up-bodyweight|vertical_pull|back_lats_upperback|biceps|pullup_bar|intermediate|false|4|2|vertical_pull,pullup|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|face-pull-cable|horizontal_pull|rear_delts|upper_back|cable|beginner|true|2|1|rear_delts,cable|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|rear-delt-fly-dumbbell|horizontal_pull|rear_delts|upper_back|dumbbell|beginner|true|2|1|rear_delts,dumbbell|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|rear-delt-fly-machine|horizontal_pull|rear_delts|upper_back|machine_selectorized|beginner|true|2|1|rear_delts,machine|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|dumbbell-biceps-curl|elbow_flexion|biceps|forearms|dumbbell|beginner|true|2|1|biceps,dumbbell|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|hammer-curl-dumbbell|elbow_flexion|biceps|brachialis/forearms|dumbbell|beginner|true|2|1|biceps,dumbbell|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|cable-biceps-curl|elbow_flexion|biceps|forearms|cable|beginner|true|2|1|biceps,cable|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|plank|core_anti_extension_or_rotation|core|glutes|bodyweight|beginner|true|2|1|core,no_equipment|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|dead-bug|core_anti_extension_or_rotation|core|hip_flexors|bodyweight|beginner|true|2|1|core,no_equipment|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|cable-crunch|core_anti_extension_or_rotation|core|none|cable|beginner|true|2|1|core,cable|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|
|pallof-press|core_anti_extension_or_rotation|core|obliques|cable,bands|beginner|true|2|1|core,anti_rotation|Set stable stance and brace.|Move through full controlled range.|Using momentum and losing control.|Stop if sharp pain appears.|true|

**Safe default:** generator scope is locked to these 50 slugs in MVP Core.

---

## 8) Acceptance Tests (expanded)

1. **RLS tamper tests**: user cannot update/delete protected tables directly.
2. **Dependent RLS check**: post_workout_check_in not accessible without session ownership.
3. **History mutation denial**: direct client update on set_log/workout_session denied.
4. **Same-exercise substitution collision**: repeated same exercise in session does not violate uniqueness due to `slot_occurrence`.
5. **Start idempotency**: duplicate start mutation returns same session.
6. **Complete idempotency**: duplicate complete mutation returns same status, no duplicate side effects.
7. **Local draft/server conflict**: stale completion mutation returns 409 with canonical payload; client rebase path validated.
8. **Equipment full-replace ownership**: user cannot replace another user’s profile items; own profile replace soft-deletes removed items.
9. **Regeneration with in-progress session**: current session remains on old plan; new plan applies to not-started sessions.
10. **Unsafe nutrition targets**: out-of-bounds calories/protein rejected with suggested safe range.

---

## 9) Deferred Decisions
1. Non-binary formula pathway (**DEFERRED**; safe default manual targets).
2. Advanced multi-device field-level merge UI (**DEFERRED**; safe default server-canonical + rebase).
3. Expanded catalog beyond seed 50 (**DEFERRED**; safe default fixed 50-slug registry).
