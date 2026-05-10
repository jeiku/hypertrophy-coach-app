# CANONICAL DATA AND PLAN GENERATION CONTRACT

**Status:** Implementation Freeze - Build Ready (All prior blockers resolved)  
**Source of truth:** `PRD.md` v0.4 Canonical Draft  
**Revision date:** 2026-05-10

## 0) Scope, freeze rules, and blockers
- MVP only; no scope expansion.
- Deterministic rules-based plan generation only.
- Planned data (`plan_version` tree) remains separate from execution data (`workout_session` + `set_log`).
- Regeneration always creates a new `plan_version`.
- Completed history is immutable.

Canonical exercise seed is checked in at `db/seeds/001_exercises_canonical_50.sql` with deterministic UUIDs/slugs and required `instruction_cues` + `safety_cues` fields.

---

## 1) Canonical enums

### 1.1 Business enums
- `experience_level`: `beginner|intermediate`
- `goal_type`: `bulk|cut|recomp`
- `session_status`: `not_started|in_progress|completed|partial|abandoned|skipped|completed_outside_app|deleted`
- `limitation_type`: `pain|mobility|medical_restriction|temporary_discomfort`
- `limitation_severity`: `low|moderate|high`
- `sex`: `male|female`
- `weight_unit`: `kg|lb`
- `set_source`: `prescribed|added`
- `set_state`: `active|skipped|deleted`

### 1.2 `movement_intent` enum
`horizontal_push`, `vertical_push`, `horizontal_pull`, `vertical_pull`, `squat_pattern`, `hinge_pattern`, `lunge_pattern`, `hip_thrust_pattern`, `knee_flexion_isolation`, `elbow_flexion_isolation`, `elbow_extension_isolation`, `lateral_raise_pattern`, `rear_delt_raise_pattern`, `calf_plantarflexion`, `core_flexion`, `core_anti_extension`, `core_anti_rotation`.

---

## 2) Complete MVP Postgres schemas (Supabase)

All tables have `created_at timestamptz not null default now()`, `updated_at timestamptz not null default now()` unless noted.
All soft-delete tables use `deleted_at timestamptz null` and are excluded by default in API queries.

## 2.1 `user_profile`
- `id uuid pk default gen_random_uuid()`
- `user_id uuid not null unique references auth.users(id)`
- `display_name text not null check (char_length(display_name) between 1 and 80)`
- `birth_date date not null`
- `sex sex not null`
- `height_cm numeric(5,2) not null check (height_cm between 120 and 240)`
- `experience_level experience_level not null`
- `timezone text not null`
- `onboarding_completed_at timestamptz null`
- `version int not null default 1`
Indexes: `(user_id)`, `(updated_at desc)`.

## 2.2 `goal_plan`
- `id uuid pk`
- `user_id uuid not null references auth.users(id)`
- `goal_type goal_type not null`
- `days_per_week int not null check (days_per_week between 2 and 6)`
- `session_length_min int not null check (session_length_min between 30 and 120)`
- `focus_muscles text[] not null default '{}'`
- `target_calories int null`
- `target_protein_g numeric(6,2) null`
- `target_carbs_g numeric(6,2) null`
- `target_fat_g numeric(6,2) null`
- `maintenance_calories_override int null`
- `active boolean not null default true`
- `version int not null default 1`
- `deleted_at timestamptz null`
Constraints: unique active goal per user: partial unique index `(user_id) where active=true and deleted_at is null`.

## 2.3 `equipment_profile`
- `id uuid pk`
- `user_id uuid not null references auth.users(id)`
- `name text not null check (char_length(name) between 1 and 80)`
- `active boolean not null default true`
- `version int not null default 1`
- `deleted_at timestamptz null`
Indexes: `(user_id, active)`, unique `(user_id, lower(name)) where deleted_at is null`.

## 2.4 `equipment_profile_item`
- `id uuid pk`
- `equipment_profile_id uuid not null references equipment_profile(id)`
- `user_id uuid not null references auth.users(id)`
- `equipment_key text not null`
- `available boolean not null default true`
- `notes text null`
- `client_item_id uuid null` (offline-created id from client)
- `version int not null default 1`
- `deleted_at timestamptz null`
Constraints:
- unique `(equipment_profile_id, equipment_key) where deleted_at is null`
- unique `(user_id, client_item_id) where client_item_id is not null`
- check ownership chain via trigger: item.user_id must match profile.user_id.
Indexes: `(equipment_profile_id)`, `(user_id, updated_at desc)`.

## 2.5 `equipment_calendar_entry`
- `id uuid pk`
- `user_id uuid not null references auth.users(id)`
- `weekday int not null check (weekday between 1 and 7)`
- `equipment_profile_id uuid not null references equipment_profile(id)`
- `version int not null default 1`
- `deleted_at timestamptz null`
Constraints: unique `(user_id, weekday) where deleted_at is null`.

## 2.6 `plan_version`
- `id uuid pk`
- `user_id uuid not null references auth.users(id)`
- `goal_plan_id uuid not null references goal_plan(id)`
- `based_on_plan_version_id uuid null references plan_version(id)`
- `version_number int not null`
- `is_active boolean not null default true`
- `generate_mode text not null check (generate_mode in ('initial','regenerate'))`
- `generator_input_snapshot jsonb not null`
- `generator_output_snapshot jsonb not null`
- `warnings jsonb not null default '[]'::jsonb`
- `deleted_at timestamptz null`
Constraints: unique `(user_id, version_number)`.
DB invariant: exactly one active non-deleted plan version per user enforced by partial unique index `(user_id) where is_active=true and deleted_at is null`.

## 2.7 `workout_day`
- `id uuid pk`
- `plan_version_id uuid not null references plan_version(id)`
- `user_id uuid not null references auth.users(id)`
- `day_index int not null check (day_index between 1 and 7)`
- `day_label text not null`
- `estimated_duration_min int not null`
- `version int not null default 1`
- `deleted_at timestamptz null`
Constraints: unique `(plan_version_id, day_index) where deleted_at is null`.

## 2.8 `exercise_instance`
- `id uuid pk`
- `workout_day_id uuid not null references workout_day(id)`
- `plan_version_id uuid not null references plan_version(id)`
- `user_id uuid not null references auth.users(id)`
- `exercise_id uuid not null`
- `slot_index int not null`
- `movement_intent movement_intent not null`
- `trim_priority int not null check (trim_priority between 1 and 5)`
- `optional boolean not null default false`
- `substituted_from_exercise_instance_id uuid null references exercise_instance(id)`
- `version int not null default 1`
- `deleted_at timestamptz null`
Constraints: unique `(workout_day_id, slot_index) where deleted_at is null`.

## 2.9 `set_prescription`
- `id uuid pk`
- `exercise_instance_id uuid not null references exercise_instance(id)`
- `workout_day_id uuid not null references workout_day(id)`
- `user_id uuid not null references auth.users(id)`
- `set_index int not null check (set_index >= 1)`
- `rep_min int not null check (rep_min between 1 and 50)`
- `rep_max int not null check (rep_max between rep_min and 100)`
- `target_rir numeric(3,1) null check (target_rir between 0 and 5)`
- `is_warmup boolean not null default false`
- `version int not null default 1`
- `deleted_at timestamptz null`
Constraints: unique `(exercise_instance_id, set_index) where deleted_at is null`.

## 2.10 `workout_session`
- `id uuid pk`
- `user_id uuid not null references auth.users(id)`
- `plan_version_id uuid not null references plan_version(id)`
- `workout_day_id uuid not null references workout_day(id)`
- `scheduled_for_date date not null`
- `started_at timestamptz null`
- `completed_at timestamptz null`
- `status session_status not null default 'not_started'`
- `completed_outside_app boolean not null default false`
- `notes text null`
- `pain_discomfort_locations text[] not null default '{}'`
- `pain_discomfort_severity int not null default 1 check (pain_discomfort_severity between 1 and 5)`
- `form_breakdown_flag boolean not null default false`
- `form_breakdown_notes text null`
- `version int not null default 1`
- `deleted_at timestamptz null`
Constraints: unique `(user_id, scheduled_for_date, workout_day_id) where deleted_at is null and status <> 'deleted'`.

## 2.11 `set_log`
- `id uuid pk`
- `workout_session_id uuid not null references workout_session(id)`
- `user_id uuid not null references auth.users(id)`
- `exercise_instance_id uuid not null references exercise_instance(id)`
- `set_prescription_id uuid null references set_prescription(id)`
- `set_source set_source not null`
- `set_index int not null check (set_index >= 1)`
- `state set_state not null default 'active'`
- `reps_completed int null check (reps_completed between 0 and 100)`
- `load_value numeric(7,2) null check (load_value >= 0)`
- `load_unit weight_unit null`
- `rir numeric(3,1) null check (rir between 0 and 5)`
- `client_set_log_id uuid null`
- `substitution_group_id uuid null`
- `version int not null default 1`
- `deleted_at timestamptz null`
Rules:
- Prescribed set: `set_source='prescribed'` requires non-null `set_prescription_id`.
- Added set: `set_source='added'` requires `set_prescription_id is null`.
- unique `(workout_session_id, set_prescription_id) where set_prescription_id is not null and deleted_at is null`.
- unique `(workout_session_id, exercise_instance_id, set_index, set_source) where deleted_at is null`.
- unique `(user_id, client_set_log_id) where client_set_log_id is not null`.
Skipped sets: `state='skipped'`, keep row immutable except audit fields.
Deleted sets: `state='deleted'` + `deleted_at` populated (soft delete).

## 2.12 `body_weight_log`
- `id uuid pk`
- `user_id uuid not null references auth.users(id)`
- `logged_on date not null`
- `weight_value numeric(6,2) not null check (weight_value between 30 and 350)`
- `weight_unit weight_unit not null`
- `weight_kg numeric(6,2) generated always as (case when weight_unit='kg' then weight_value else round((weight_value/2.20462)::numeric,2) end) stored`
- `timezone text not null`
- `logged_at_client timestamptz null`
- `client_log_id uuid null`
- `version int not null default 1`
- `deleted_at timestamptz null`
Constraints:
- unique `(user_id, logged_on) where deleted_at is null`
- unique `(user_id, client_log_id) where client_log_id is not null`
Timezone/date behavior:
- Server computes `logged_on` using payload local date; if only timestamp given, convert to user_profile timezone then take date.
Weekly trend:
- `weekly_avg_kg = avg(weight_kg)` over rows where `logged_on between current_date-6 and current_date`, excluding deleted.
- `weekly_trend_kg = weekly_avg_kg(current 7 days) - weekly_avg_kg(previous 7 days)`.

## 2.13 `food_log`
- `id uuid pk`
- `user_id uuid not null references auth.users(id)`
- `logged_on date not null`
- `meal_type text not null`
- `saved_meal_id uuid null references saved_meal(id)`
- `name text not null`
- `calories int not null check (calories >= 0)`
- `protein_g numeric(6,2) not null check (protein_g >= 0)`
- `carbs_g numeric(6,2) not null check (carbs_g >= 0)`
- `fat_g numeric(6,2) not null check (fat_g >= 0)`
- `client_log_id uuid null`
- `version int not null default 1`
- `deleted_at timestamptz null`

## 2.14 `saved_meal`
- `id uuid pk`
- `user_id uuid not null references auth.users(id)`
- `name text not null`
- `ingredients jsonb not null`
- `calories int not null`
- `protein_g numeric(6,2) not null`
- `carbs_g numeric(6,2) not null`
- `fat_g numeric(6,2) not null`
- `version int not null default 1`
- `deleted_at timestamptz null`
Unique: `(user_id, lower(name)) where deleted_at is null`.

## 2.15 `recommendation`
- `id uuid pk`
- `user_id uuid not null references auth.users(id)`
- `workout_session_id uuid null references workout_session(id)`
- `recommendation_type text not null`
- `payload jsonb not null`
- `status text not null check (status in ('pending','accepted','ignored','expired')) default 'pending'`
- `acted_at timestamptz null`
- `version int not null default 1`
- `deleted_at timestamptz null`

## 2.16 `post_workout_check_in`
- `id uuid pk`
- `user_id uuid not null references auth.users(id)`
- `workout_session_id uuid not null unique references workout_session(id)`
- `energy_score int not null check (energy_score between 1 and 5)`
- `difficulty_score int not null check (difficulty_score between 1 and 5)`
- `soreness_score int not null check (soreness_score between 1 and 5)`
- `notes text null`
- `pain_discomfort_locations text[] not null default '{}'`
- `pain_discomfort_severity int not null default 1 check (pain_discomfort_severity between 1 and 5)`
- `form_breakdown_flag boolean not null default false`
- `form_breakdown_notes text null`
- `version int not null default 1`
- `deleted_at timestamptz null`


## 2.18 `exercise_catalog`
- `id uuid pk` (deterministic seed IDs)
- `slug text not null unique`
- `name text not null`
- `movement_intent movement_intent not null`
- `equipment_key text not null`
- `primary_muscles text[] not null default '{}'`
- `secondary_muscles text[] not null default '{}'`
- `contraindication_tags text[] not null default '{}'`
- `instruction_cues text[] not null check (array_length(instruction_cues,1) >= 1)`
- `safety_cues text[] not null check (array_length(safety_cues,1) >= 1)`
- `is_active boolean not null default true`
Constraints: unique `(slug)` and seed row count acceptance test `= 50 active canonical rows`.

## 2.19 `user_exercise_preference`
- `id uuid pk`
- `user_id uuid not null references auth.users(id)`
- `exercise_id uuid not null references exercise_catalog(id)`
- `preference text not null check (preference in ('preferred','disliked','locked_in','locked_out'))`
- `reason text null`
- `active boolean not null default true`
- `version int not null default 1`
- `deleted_at timestamptz null`
Constraints: unique `(user_id, exercise_id) where deleted_at is null`.
Generator scoring impact (deterministic additive weight): `preferred +25`, `disliked -35`, `locked_in force-include when feasible`, `locked_out hard-exclude`.

## 2.17 `user_limitation`
- `id uuid pk`
- `user_id uuid not null references auth.users(id)`
- `limitation_type limitation_type not null`
- `affected_body_region text not null`
- `affected_movement_intents movement_intent[] not null default '{}'`
- `affected_exercise_ids uuid[] not null default '{}'`
- `severity limitation_severity not null`
- `pain_flag boolean not null default false`
- `notes text null`
- `active boolean not null default true`
- `version int not null default 1`
- `deleted_at timestamptz null`

---

## 3) RLS policies (table-by-table)
Assume helper function `auth.uid()` and service role bypass.

- `user_profile`, `goal_plan`, `equipment_profile`, `equipment_profile_item`, `equipment_calendar_entry`, `body_weight_log`, `food_log`, `saved_meal`, `user_limitation`, `post_workout_check_in`: `USING (user_id = auth.uid())`, `WITH CHECK (user_id = auth.uid())`.
- `plan_version`, `workout_day`, `exercise_instance`, `set_prescription`, `workout_session`, `set_log`, `recommendation`: client role **read-own only**; writes denied, backend service performs writes.

Ownership-chain read policies:
- `workout_day`: exists plan_version where `plan_version.id = workout_day.plan_version_id and plan_version.user_id=auth.uid()`.
- `exercise_instance`: exists workout_day owned by user.
- `set_prescription`: exists exercise_instance owned by user.
- `set_log`: exists workout_session owned by user and exercise_instance owned by user.

Cross-user denial acceptance criteria: any user B select/update on user A row returns zero rows / forbidden.

---

## 4) Idempotency ledgers

### 4.1 `processed_mutation`
Covers all non-session writes including `PUT /v1/equipment/profiles/{id}/items`.
- unique `(user_id, mutation_id)`.
- Replay same hash => return stored response/status.
- Same mutation_id different hash => `409 MUTATION_ID_REUSE_CONFLICT`.

### 4.2 `workout_session_mutation`
Session execution mutations only (`start`, set-log CRUD/skip, substitution, complete/partial/abandon/skip, recommendation actions tied to session).

---

## 5) Write API contracts (full request/response)
All write requests include: `mutationId`, `clientTimestamp`, optional `clientRequestId`, and `ifMatchVersion` when versioned.

### 5.1 Onboarding/profile
- `PUT /v1/profile` -> `{data:{profile,version}}`
- `POST /v1/onboarding/complete` -> `{data:{completedAt,profileVersion,goalPlanId,equipmentProfileId}}`

### 5.2 Equipment
- `PUT /v1/equipment/profiles/{id}` request `{profile:{name,active}}` response `{data:{equipmentProfile,version}}`
- `PUT /v1/equipment/profiles/{id}/items` (full replace)
Request:
```json
{"mutationId":"uuid","clientTimestamp":"...","ifMatchVersion":4,"items":[{"id":"optional-uuid","clientItemId":"optional-uuid","equipmentKey":"barbell","available":true,"notes":"optional"}]}
```
Response:
```json
{"data":{"equipmentProfileId":"uuid","items":[{"id":"uuid","equipmentKey":"barbell","available":true,"notes":null,"version":1}],"version":5}}
```
- `PUT /v1/equipment/profiles/{id}/calendar` request `{entries:[{weekday:1,equipmentProfileId:"uuid"}]}` response `{data:{entries,version}}`.

### 5.3 Plan generation/regeneration
- `POST /v1/plans/generate` request `{mutationId,clientTimestamp,goalPlanId,equipmentProfileId,calendarWeekStart}`
- `POST /v1/plans/{id}/regenerate` request `{mutationId,clientTimestamp,reason,ifMatchVersion,preserveInProgress:true}`
Response shape (both):
```json
{"data":{"planVersion":{"id":"uuid","versionNumber":3,"isActive":true,"generateMode":"initial|regenerate"},"workoutDays":[{"id":"uuid","dayIndex":1,"dayLabel":"Push A","estimatedDurationMin":60}],"exerciseInstances":[{"id":"uuid","workoutDayId":"uuid","exerciseId":"uuid","slotIndex":1,"movementIntent":"horizontal_push","trimPriority":1,"optional":false}],"setPrescriptions":[{"id":"uuid","exerciseInstanceId":"uuid","setIndex":1,"repMin":6,"repMax":10,"targetRir":2.0}],"explanations":[{"code":"EQUIPMENT_MATCH","message":"Matched dumbbell profile"}],"warnings":[],"infeasibleErrors":[]}}
```
`infeasibleErrors` must be non-empty when HTTP 422.


### 5.4 Workout execution
- `POST /v1/workout-sessions/start` -> `{data:{workoutSession}}`
- `POST /v1/workout-sessions/{id}/set-logs` create
- `PATCH /v1/workout-sessions/{id}/set-logs/{setLogId}` update
- `DELETE /v1/workout-sessions/{id}/set-logs/{setLogId}` soft delete (`state='deleted'`)
- `POST /v1/workout-sessions/{id}/set-logs/{setLogId}/skip` (`state='skipped'`)
- `POST /v1/workout-sessions/{id}/substitutions`
- `POST /v1/workout-sessions/{id}/complete|partial|abandon|skip`
Status transitions:
- `not_started -> in_progress|skipped|completed_outside_app|deleted`
- `in_progress -> completed|partial|abandoned`
- terminal: `completed|partial|abandoned|skipped|completed_outside_app|deleted`

### 5.5 Body weight logs
- `POST /v1/body-weight-logs` request `{loggedOn,weightValue,weightUnit,timezone,loggedAtClient?,clientLogId?}`
- `PATCH /v1/body-weight-logs/{id}` same mutable fields
- `DELETE /v1/body-weight-logs/{id}` soft delete
Response includes computed fields: `{weightKg,weeklyAvgKg,weeklyTrendKg}`.


## 5.7 Regeneration behavior contract
- **Same-day not_started session:** if scheduled date is local today and status `not_started`, regenerate may rebind to new active `plan_version` unless a local draft exists (see §5.8).
- **Same-day in_progress session:** never rebound; remains attached to original plan_version until terminal.
- **Future not_started sessions:** eligible for rebind to newest active plan_version; preserve `scheduled_for_date`.
- **Local-draft-protected sessions:** server must return `409 DRAFT_CONFLICT` with authoritative mapping metadata when client indicates unsynced draft mutations for that session.
- **Past not_started sessions:** remain historical; no automatic rebind.

### 5.6 Food logs/saved meals/recommendations/limitations
- `POST /v1/limitations` request `{limitationType,affectedBodyRegion,affectedMovementIntents,affectedExerciseIds,severity,painFlag,notes}` response `{data:{limitation,version}}`
- `PATCH /v1/limitations/{id}` request same mutable fields + `ifMatchVersion` response `{data:{limitation,version}}`
- `POST /v1/recommendations/{id}/accept|ignore` request `{mutationId,clientTimestamp}` response `{data:{recommendation,status,actedAt,version}}`
- `POST /v1/food-logs` request `{loggedOn,mealType,savedMealId?,name,calories,proteinG,carbsG,fatG,clientLogId?}` response `{data:{foodLog,version}}`
- `PATCH /v1/food-logs/{id}` and `DELETE /v1/food-logs/{id}` standard versioned responses
- `POST /v1/saved-meals` request `{name,ingredients,calories,proteinG,carbsG,fatG}` response `{data:{savedMeal,version}}`
- `PATCH /v1/saved-meals/{id}` and `DELETE /v1/saved-meals/{id}` standard versioned responses.


---

## 6) Generator templates (explicit 5-day and 6-day)

## 6.1 5-day intermediate template
- D1: `horizontal_push(4,6-10,p1)`, `horizontal_pull(4,6-10,p1)`, `vertical_push(3,8-12,p2)`, `vertical_pull(3,8-12,p2)`, `lateral_raise_pattern(3,12-20,p4,opt)`
- D2: `squat_pattern(4,5-8,p1)`, `hinge_pattern(3,6-10,p1)`, `lunge_pattern(3,8-12,p2)`, `calf_plantarflexion(3,10-15,p4,opt)`, `core_anti_extension(3,10-15,p3)`
- D3: `vertical_pull(4,6-10,p1)`, `horizontal_pull(3,8-12,p1)`, `rear_delt_raise_pattern(3,12-20,p3,opt)`, `elbow_flexion_isolation(3,8-15,p3)`
- D4: `hinge_pattern(4,5-8,p1)`, `hip_thrust_pattern(3,8-12,p2)`, `knee_flexion_isolation(3,10-15,p3)`, `calf_plantarflexion(3,10-15,p4,opt)`, `core_anti_rotation(3,10-15,p3)`
- D5: `horizontal_push(3,8-12,p2)`, `elbow_extension_isolation(3,8-15,p3)`, `elbow_flexion_isolation(3,8-15,p3)`, `lateral_raise_pattern(3,12-20,p4,opt)`, `core_flexion(3,10-20,p4,opt)`

## 6.2 6-day intermediate template
- Push A: `horizontal_push(4,6-10,p1)`, `vertical_push(3,8-12,p2)`, `elbow_extension_isolation(3,8-15,p3)`, `lateral_raise_pattern(3,12-20,p4,opt)`
- Pull A: `vertical_pull(4,6-10,p1)`, `horizontal_pull(3,8-12,p2)`, `elbow_flexion_isolation(3,8-15,p3)`, `rear_delt_raise_pattern(3,12-20,p4,opt)`
- Legs A: `squat_pattern(4,5-8,p1)`, `hinge_pattern(3,6-10,p1)`, `calf_plantarflexion(3,10-15,p4,opt)`, `core_anti_extension(3,10-15,p3)`
- Push B: `horizontal_push(3,8-12,p1)`, `vertical_push(3,8-12,p2)`, `elbow_extension_isolation(3,10-15,p3)`, `lateral_raise_pattern(3,12-20,p4,opt)`
- Pull B: `horizontal_pull(4,6-10,p1)`, `vertical_pull(3,8-12,p2)`, `elbow_flexion_isolation(3,10-15,p3)`, `rear_delt_raise_pattern(3,12-20,p4,opt)`
- Legs B: `hip_thrust_pattern(4,6-10,p1)`, `lunge_pattern(3,8-12,p2)`, `knee_flexion_isolation(3,10-15,p3)`, `core_anti_rotation(3,10-15,p3)`

Beginner rule: same slots, accessory sets -1 (floor 2).

---

## 7) Set prescription/log linkage and substitution rules
- `set_log.set_prescription_id` links directly to planned set when `set_source='prescribed'`.
- Added sets use `set_source='added'` and null `set_prescription_id`.
- Substitution creates new `exercise_instance` linked via `substituted_from_exercise_instance_id`; existing logs stay immutable and linked to their originating instance.
- Offline-created rows must include `clientSetLogId` and/or `clientItemId` for dedupe.

---

## 8) Nutrition safety bounds (unchanged MVP)
- Hard reject: calories `<1200` or `>5000`, deficit `>30%`, surplus `>20%`, protein `<0.8 g/kg` or `>2.4 g/kg`.
- Warn but allow: deficit `20-30%`, surplus `15-20%`, protein `0.8-1.2` or `2.2-2.4 g/kg`.
- Missing required anthropometrics => require manual maintenance estimate.

---

## 9) Acceptance tests (expanded)
1. Equipment item full-replace persists and removes omitted items.
2. `PUT /equipment/profiles/{id}/items` idempotent replay behavior is correct.
3. Body weight uniqueness on `(user_id, logged_on)` enforced.
4. Weekly trend matches 7-day window formula.
5. SetLog uniqueness and prescribed-link constraints enforced.
6. Added sets accepted without `set_prescription_id` and never collide with prescribed unique key.
7. Skip/delete behaviors persist across sync and replay.
8. RLS denies cross-user access on child tables (`exercise_instance`,`set_prescription`,`set_log`).
9. `completed_outside_app` flow behaves as terminal status and excluded from in-app set logging.
10. Session statuses include `not_started` and `deleted`; missed-workout logic references `not_started` only.
11. Generator deterministically selects same exercise catalog rows for same input.
12. Plan-generation output matches required shape (`planVersion`,`workoutDays`,`exerciseInstances`,`setPrescriptions`,`explanations`,`warnings`,`infeasibleErrors`).
13. 5-day and 6-day template slot order and trim priorities are deterministic.
14. Regeneration preserves in-progress session binding and only rebinds eligible not-started future sessions.
15. Mutation ID hash mismatch returns `409 MUTATION_ID_REUSE_CONFLICT` with stored operation metadata.

## 5.8 Local workout draft sync contract
Local draft object (per session):
```json
{
  "draftId":"uuid",
  "workoutSessionId":"uuid",
  "planVersionId":"uuid",
  "baseSessionVersion":12,
  "operations":[
    {"opId":"uuid","seq":1,"type":"set_log_create|set_log_update|set_log_skip|set_log_delete|substitution|session_complete|session_partial|session_abandon","payload":{},"clientTs":"ISO-8601"}
  ],
  "lastSyncedSeq":0
}
```
Rules:
- Queue ordering is strictly ascending `seq`; server rejects gaps/duplicates with `409 MUTATION_SEQUENCE_CONFLICT`.
- Retry is exponential backoff (1s,2s,4s,8s, max 5 attempts) with same `mutationId`/`opId` (idempotent).
- Conflict: if `baseSessionVersion` stale, server returns `409 VERSION_CONFLICT` + latest session projection and accepted `lastAppliedSeq`.
- Client/server ID reconciliation: response returns `idMap` for local IDs (`clientSetLogId`,`clientItemId`,`draftId`) to canonical UUIDs.
- Apply is atomic per op; partial success within one op is forbidden.
