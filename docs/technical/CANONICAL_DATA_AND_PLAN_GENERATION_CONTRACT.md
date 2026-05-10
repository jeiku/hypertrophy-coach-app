# CANONICAL DATA AND PLAN GENERATION CONTRACT (FINAL MVP BUILD CONTRACT)

**Status:** Implementation-ready pending checklist sign-off (PRD v0.6 aligned)  
**Source of truth:** `PRDv0.6.md`  
**Revision date:** 2026-05-10

---

## 0) Scope and implementation boundary

This contract defines MVP-only, implementation-ready behavior for:
- canonical data model
- API request/response schemas
- idempotency/concurrency semantics
- ownership and Supabase RLS behavior
- deterministic generation and recovery rules
- acceptance tests and readiness checklist

Out of scope for MVP: nutrition entities, nutrition APIs, per-day equipment calendar logic, historical body-weight trend storage (`BodyWeightLog`, see §2.14).

---

## 1) Canonical enums (MVP)

- `experience_level`: `beginner | intermediate`
- `goal_type`: `bulk | cut | recomp`
- `sex`: `male | female`
- `weight_unit`: `kg | lb`
- `session_status`: `not_started | in_progress | completed | partial | abandoned | skipped | completed_outside_app | deleted`
- `set_source`: `prescribed | added`
- `set_state`: `active | skipped | deleted`
- `plan_version_status`: `draft | active | archived | reverted`
- `soreness_level`: `none | mild | moderate | high`
- `pain_type`: `sharp | dull_aching | joint | nerve_like | other`
- `fatigue_level`: `low | normal | high`
- `pain_location`: `shoulder | elbow | wrist | lower_back | upper_back | hip | knee | ankle | other`
- `recommendation_type`: `add_reps | add_load | hold_load | reduce_load | substitute_exercise | suggest_deload`
- `recommendation_status`: `pending | accepted | ignored | dismissed | expired`
- `recommendation_ignored_reason`: `too_aggressive | wrong_exercise | not_today | disagree_with_reasoning | other | no_reason_given`
- `exercise_preference_type`: `preferred | disliked | neutral`
- `limitation_type`: `mobility_limit | pain_history | injury_history | medical_restriction | equipment_constraint | other`
- `limitation_severity`: `low | moderate | high | hard_block`
- `limitation_status`: `active | resolved | archived`

---

## 2) Data model (MVP canonical)

All PKs are UUIDv7. All tables include `created_at timestamptz not null default now()` and `updated_at timestamptz not null default now()` unless noted.

### 2.1 `user_profile`
- `id` (pk)
- `user_id` (uuid, unique, fk -> auth.users.id)
- `birth_date` (date, nullable)
- `sex` (enum, nullable)
- `height_cm` (numeric(5,2), nullable)
- `body_weight` (numeric(6,2), nullable)
- `body_weight_unit` (`kg|lb`, nullable)
- `experience_level` (enum, nullable)
- `timezone` (text, required)
- `onboarding_completed_at` (timestamptz, nullable)

### 2.2 `goal_plan`
- `id` (pk)
- `user_id` (fk)
- `goal_type`
- `days_per_week` (int, 2..6)
- `session_length_min` (int, 20..120)
- `focus_muscles` (`text[] not null default '{}'`)  
  - optional onboarding input; empty array is valid for MVP.
- unique partial: one active goal plan per user (`is_active=true`)
- `is_active` (boolean not null default true)

Generator behavior when `focus_muscles=[]`:
- no focus-muscle boost is applied
- no focus-muscle protection is applied during accessory cuts
- balanced default volume targets are used

### 2.3 `equipment_profile`
- `id` (pk)
- `user_id` (fk)
- `name` (text, 1..80)
- `is_active` (boolean)
- uniqueness: one active profile per user (`unique (user_id) where is_active=true`)

### 2.4 `equipment_profile_item`
- `id` (pk)
- `user_id` (fk)
- `equipment_profile_id` (fk)
- `equipment_key` (text)
- uniqueness: `unique(equipment_profile_id, equipment_key)`
- ownership-chain rule: `equipment_profile.user_id == equipment_profile_item.user_id`

### 2.5 `exercise_catalog` (seed-owned)
- `id` (pk)
- `slug` (text unique)
- `name` (text)
- `movement_intents` (text[] not null)
- `primary_muscles` (text[])
- `secondary_muscles` (text[])
- `equipment_required` (text[])
- `difficulty` (text)
- `fatigue_cost` (int)
- `tempo_seconds` (int)
- `rest_seconds` (int)
- `warmup_overhead_seconds` (int)
- `setup_overhead_seconds` (int)
- `cues_json` (jsonb)
- read-only to clients; service-role seed ownership

### 2.6 `user_exercise_preference`
- `id` (pk)
- `user_id` (fk)
- `exercise_id` (fk -> exercise_catalog.id)
- `preference_type` (`preferred|disliked|neutral`)
- `source` (text default `user_explicit`)
- `notes` (text nullable, private, analytics-excluded)
- uniqueness: `unique(user_id, exercise_id)`

### 2.7 `user_limitation`
- `id` (pk)
- `user_id` (fk)
- `limitation_type` (enum)
- `pain_locations` (`pain_location[]`, nullable)
- `affected_movement_intents` (text[] nullable)
- `severity` (`low|moderate|high|hard_block`)
- `status` (`active|resolved|archived` default `active`)
- `started_on` (date nullable)
- `resolved_on` (date nullable)
- `notes` (text nullable, private, analytics-excluded)

### 2.8 `notification_preference` (MVP core)
- `id` (pk)
- `user_id` (uuid not null unique fk -> auth.users.id)
- `workout_reminder_enabled` (boolean not null default true)
- `workout_reminder_minutes_before` (int not null default 60; allowed 5..1440)
- `missed_workout_followup_enabled` (boolean not null default true)
- `recommendation_pending_review_enabled` (boolean not null default true)
- `streak_milestone_enabled` (boolean not null default false)
- `quiet_hours_start` (time not null default `'21:00:00'`)  
- `quiet_hours_end` (time not null default `'08:00:00'`)
- `timezone` (text not null; IANA zone)
- `created_at`, `updated_at`

### 2.9 `plan_version`
- `id`, `user_id`, `goal_plan_id`
- `based_on_plan_version_id` (nullable)
- `version_number` (int)
- `status` (`draft|active|archived|reverted`)
- `generate_mode` (`initial|regenerate|revert_clone`)
- `input_snapshot_json`, `output_snapshot_json`, `validation_warnings_json`
- uniqueness: one active plan per user (`unique(user_id) where status='active'`)

### 2.10 `workout_day`, `exercise_instance`, `set_prescription`
- `workout_day`: links to `plan_version`, holds day order/schedule metadata
- `exercise_instance`: links to `workout_day`, references `exercise_catalog`, stores slot metadata and trim priority
- `set_prescription` fields:
  - `id`, `user_id`, `exercise_instance_id` fk, `set_order`, `target_reps_min`, `target_reps_max`, `target_rpe` nullable, `set_type` text default `working`
  - uniqueness: `unique(exercise_instance_id, set_order)`

### 2.11 `workout_session`
- `id`, `user_id`, `plan_version_id`, `workout_day_id`
- `scheduled_for_date` (date)
- `status` enum
- `started_at`, `finished_at` nullable
- `version_token` (int default 1)

### 2.12 `set_log`
- `id` (pk)
- `user_id` (fk)
- `workout_session_id` (fk)
- `exercise_instance_id` (fk)
- `set_prescription_id` (fk nullable)
- `set_source` (`prescribed|added`)
- `set_state` (`active|skipped|deleted`)
- `display_order` (numeric(8,3) not null)
- `reps` (int nullable when skipped)
- `load` (numeric(6,2) nullable)
- `rpe` (numeric(3,1) nullable)
- `client_mutation_id` (text not null)
- `logged_at` (timestamptz default now())

Ordering + idempotency rules:
- prescribed sets: `display_order = set_prescription.set_order`
- added sets: client must send `displayOrder`; server persists as submitted (within exercise scope)
- offline sync: server preserves client order within each `exercise_instance_id`
- final rendering order: by `exercise_instance.order`, then `display_order`, then `logged_at` fallback
- `set_prescription_id` nullable only when `set_source='added'`
- uniqueness scope: `unique(user_id, workout_session_id, client_mutation_id)`

### 2.13 `post_workout_check_in`
- `id`, `user_id`, `workout_session_id` (unique)
- `soreness_level`
- `soreness_locations` pain_location[] nullable
- `pain_flag` boolean
- `pain_locations` pain_location[] required when `pain_flag=true`
- `pain_type` required when `pain_flag=true`
- `pain_notes` text nullable (analytics-excluded)
- `fatigue_level` nullable
- `form_breakdown` boolean nullable

### 2.14 `recommendation`
- `id`, `user_id`, `workout_session_id` nullable, `exercise_instance_id` nullable
- `type`, `status`
- `title`, `user_facing_reason`, `educational_context`
- `trigger_factors` text[]
- `inputs_used_json` jsonb
- `suggested_change_json` jsonb
- `ignored_reason` enum nullable
- `other_reason_text` text nullable (private, analytics-excluded)
- `version_token` int default 1

### 2.15 `BodyWeightLog` scope note
- `BodyWeightLog` is **post-MVP / v1.1** unless PRD changes.
- MVP stores only `body_weight` and `body_weight_unit` on `user_profile`.

---

## 3) API contracts (implementation-ready)

### 3.0 Common behavior
- Auth: Bearer JWT required for all `/v1/*` endpoints.
- Error schema (all non-2xx):
```json
{ "error": { "code": "string", "message": "string", "fieldErrors": [{"field":"string","message":"string"}] } }
```
- Status codes: `401` auth missing/invalid, `403` ownership denied, `404` not found, `409` version conflict/idempotency conflict, `422` validation.
- Idempotent replay success includes:
```json
{ "meta": { "idempotentReplay": true } }
```
- Version-token conflict includes:
```json
{ "error": { "code":"VERSION_CONFLICT", "message":"Stale versionToken", "currentVersionToken": 7 } }
```

### 3.1 Auth scope
MVP auth is Supabase Auth, **email/password only**. Apple/Google sign-in is fast-follow post-MVP and can only be enabled after explicit post-G3 approval.

### 3.2 Equipment profiles
- `POST /v1/equipment-profiles` (idempotency key required: `Idempotency-Key`)
  - body: `{ "name": "Home", "isActive": true }`
  - 201:
```json
{ "data": { "id":"uuid","userId":"uuid","name":"Home","isActive":true,"createdAt":"iso","updatedAt":"iso" } }
```
- `PATCH /v1/equipment-profiles/{id}`
  - body: `{ "name?":"Gym", "isActive?":false, "versionToken": 3 }`
- `DELETE /v1/equipment-profiles/{id}`
  - body: `{ "versionToken": 3 }`
- `POST /v1/equipment-profiles/{id}/items`
  - body: `{ "equipmentKey": "barbell" }`
  - 201:
```json
{ "data": { "id":"uuid","equipmentProfileId":"uuid","equipmentKey":"barbell","createdAt":"iso","updatedAt":"iso" } }
```
- `DELETE /v1/equipment-profiles/{id}/items/{itemId}`

Ownership: owner only across all endpoints.

### 3.3 User exercise preferences
- `PUT /v1/users/me/exercise-preferences/{exerciseId}`
  - body: `{ "preferenceType":"preferred", "notes?":"string", "clientMutationId":"string" }`
  - success 200:
```json
{ "data": { "id":"uuid","exerciseId":"uuid","preferenceType":"preferred","notes":null,"updatedAt":"iso" }, "meta": { "idempotentReplay": false } }
```
- `GET /v1/users/me/exercise-preferences`
  - 200:
```json
{ "data": [{ "id":"uuid","exerciseId":"uuid","preferenceType":"preferred","notes":null,"updatedAt":"iso" }] }
```

### 3.4 User limitations
- `POST /v1/users/me/limitations`
  - body: `{ "limitationType":"pain_history","painLocations":["knee"],"affectedMovementIntents":["squat"],"severity":"moderate","notes?":"string","startedOn?":"YYYY-MM-DD" }`
- `PATCH /v1/users/me/limitations/{id}`
  - body: `{ "severity?":"high", "status?":"active", "notes?":"string", "versionToken":2 }`
- `POST /v1/users/me/limitations/{id}/resolve`
  - body: `{ "resolvedOn?":"YYYY-MM-DD", "versionToken":2 }`

### 3.5 Notification preferences
- `GET /v1/users/me/notification-preferences`
  - 200:
```json
{ "data": { "id":"uuid","userId":"uuid","workoutReminderEnabled":true,"workoutReminderMinutesBefore":60,"missedWorkoutFollowupEnabled":true,"recommendationPendingReviewEnabled":true,"streakMilestoneEnabled":false,"quietHoursStart":"21:00:00","quietHoursEnd":"08:00:00","timezone":"America/New_York","createdAt":"iso","updatedAt":"iso" } }
```
- `PATCH /v1/users/me/notification-preferences`
  - body (partial allowed):
```json
{ "workoutReminderEnabled?":true,"workoutReminderMinutesBefore?":45,"missedWorkoutFollowupEnabled?":true,"recommendationPendingReviewEnabled?":false,"streakMilestoneEnabled?":false,"quietHoursStart?":"21:00:00","quietHoursEnd?":"08:00:00","timezone?":"America/New_York","versionToken":4 }
```
  - 200 returns same shape as GET.

### 3.6 Plans: generate / accept / regenerate / revert
- `POST /v1/plans/generate` (Idempotency-Key required)
  - body: `{ "goalPlanId":"uuid", "equipmentProfileId":"uuid", "clientRequestId":"string" }`
- `POST /v1/plans/{planVersionId}/accept`
  - body: `{ "expectedCurrentActivePlanVersionId?":"uuid", "versionToken":2 }`
- `POST /v1/plans/{planVersionId}/regenerate` (Idempotency-Key required)
  - body: `{ "reason":"manual|missed_workout_recovery", "protectInProgress":true, "clientRequestId":"string" }`
- `POST /v1/plans/{planVersionId}/revert` (Idempotency-Key required)
  - body: `{ "versionToken":4, "clientRequestId":"string" }`

### 3.7 Workout session lifecycle
- `POST /v1/workout-sessions/start` body `{ "workoutDayId":"uuid", "scheduledForDate":"YYYY-MM-DD", "clientRequestId":"string" }`
- `POST /v1/workout-sessions/{id}/resume` body `{ "versionToken":2 }`
- `POST /v1/workout-sessions/{id}/finish` body `{ "versionToken":2, "finishedAt?":"iso" }`
- `POST /v1/workout-sessions/{id}/partial` body `{ "versionToken":2, "reason?":"string" }`
- `POST /v1/workout-sessions/{id}/skip` body `{ "versionToken":2, "reason?":"string" }`
- `POST /v1/workout-sessions/{id}/completed-outside-app` body `{ "versionToken":2, "notes?":"string" }`

### 3.8 Set logging / added sets / skipped sets
- `POST /v1/workout-sessions/{id}/set-logs`
  - prescribed:
```json
{ "exerciseInstanceId":"uuid", "setPrescriptionId":"uuid", "displayOrder":2, "reps":8, "load?":100, "rpe?":8.5, "clientMutationId":"string" }
```
  - added:
```json
{ "exerciseInstanceId":"uuid", "setSource":"added", "displayOrder":2.5, "reps":12, "load?":25, "rpe?":9, "clientMutationId":"string" }
```
  - skipped:
```json
{ "exerciseInstanceId":"uuid", "setPrescriptionId":"uuid", "setState":"skipped", "displayOrder":3, "clientMutationId":"string" }
```
  - 200/201:
```json
{ "data": { "id":"uuid","workoutSessionId":"uuid","exerciseInstanceId":"uuid","setPrescriptionId":null,"setSource":"added","setState":"active","displayOrder":2.5,"reps":12,"load":25,"rpe":9,"loggedAt":"iso" }, "meta": { "idempotentReplay": false } }
```

### 3.9 Exercise substitutions
- `GET /v1/workout-sessions/{id}/substitution-candidates?exerciseInstanceId={exerciseInstanceId}`
  - 200:
```json
{ "data": { "workoutSessionId":"uuid","exerciseInstanceId":"uuid","generatedAt":"iso","candidates":[{ "exerciseId":"uuid","name":"Incline Dumbbell Press","movementIntentMatch":true,"primaryMuscleMatch":true,"equipmentFeasible":true,"difficultyMatch":"similar","fatigueCostMatch":"similar","score":87,"scoreBreakdown":{"movementIntent":30,"primaryMuscle":25,"equipment":15,"difficulty":8,"fatigueCost":6,"preference":3,"limitation":0},"userFacingReason":"Keeps horizontal press intent while fitting your available dumbbells.","educationalContext":"You will still train chest/triceps with slightly more stabilization demand.","suggestedLoadHandling":{"type":"history_based","lastSuccessfulLoad":32.5,"lastSuccessfulReps":10,"guidance":"Start at your last successful load that hit minimum reps."} }] } }
```
- Candidate algorithm constraints:
  - preserve movement intent
  - preserve primary muscle
  - prefer similar difficulty and fatigue cost
  - enforce equipment feasibility
  - enforce pain/limitation mapping
  - apply preference constants in §4
  - return explainable reason fields (`userFacingReason`, `educationalContext`)

- `POST /v1/workout-sessions/{id}/substitutions`
  - body:
```json
{ "exerciseInstanceId":"uuid", "replacementExerciseId":"uuid", "replacementSource":"candidate_list|manual_override", "candidateListVersion":"string", "manualOverrideReason?":"string", "reason":"preference|limitation|pain|equipment", "versionToken":3, "clientRequestId":"string" }
```
  - validation:
    - `replacementSource='candidate_list'` requires replacement exists in latest server candidate list for that exercise instance.
    - `replacementSource='manual_override'` requires `manualOverrideReason`.

Load-handling rules in response payload:
- if substitute has user history hitting minimum reps: suggest last successful load
- if no history: `suggestedLoadHandling.type='no_history_blank_load'` + conservative start guidance
- bodyweight substitutions: guidance uses variation/assistance level (no numeric load)
- machine substitutions with non-comparable stacks: blank load guidance

### 3.10 Post-workout check-ins
- `PUT /v1/workout-sessions/{id}/check-in`
  - body `{ "sorenessLevel":"mild","sorenessLocations?":[],"painFlag":false,"painLocations?":[],"painType?":null,"painNotes?":"","fatigueLevel?":"normal","formBreakdown?":false,"versionToken":2 }`

### 3.11 Missed-workout recovery actions
- `POST /v1/workout-sessions/{id}/recovery/move-next`
- `POST /v1/workout-sessions/{id}/recovery/skip-continue`
- `POST /v1/workout-sessions/{id}/recovery/completed-outside-app`
- `POST /v1/workout-sessions/{id}/recovery/regenerate-week`
- body for all: `{ "versionToken":3, "clientRequestId":"string" }`

### 3.12 Recommendations
- `POST /v1/recommendations/{id}/accept` body `{ "versionToken":2 }`
- `POST /v1/recommendations/{id}/ignore` body `{ "versionToken":2, "ignoredReason?":"too_aggressive", "otherReasonText?":"string" }`
- `POST /v1/recommendations/{id}/dismiss` body `{ "versionToken":2, "ignoredReason?":"wrong_exercise", "otherReasonText?":"string" }`

### 3.13 Local draft sync recovery
- `POST /v1/sync/local-drafts/recover` (Idempotency-Key required)
  - body:
```json
{ "localDraftId":"string", "workoutSessionId?":"uuid", "setLogs":[{ "exerciseInstanceId":"uuid", "setPrescriptionId?":"uuid", "setSource":"prescribed|added", "setState":"active|skipped", "displayOrder":1, "reps?":8, "load?":100, "rpe?":8, "clientMutationId":"string", "loggedAtClient":"iso" }], "clientSyncBatchId":"string" }
```
  - response:
```json
{ "data": { "accepted":["clientMutationId1"], "duplicates":["clientMutationId2"], "conflicts":[] } }
```

---

## 4) Generator behavior integration

Scoring per candidate exercise for a slot:
- base score from PRD template fit
- `+PREFERRED_BOOST` for `preferred`
- `-DISLIKED_PENALTY` for `disliked`
- limitation penalties/exclusions applied after preference
- ineligible if limitation severity `hard_block` or pain-map disallow
- substitution fallback chooses same movement intent, same day equipment, nearest fatigue cost

Machine constants (MVP, PRD-aligned):
- `PREFERRED_BOOST = +10`
- `DISLIKED_PENALTY = -30`
- `LIMITATION_LOW_PENALTY = -8`
- `LIMITATION_MODERATE_PENALTY = -16`
- `LIMITATION_HIGH_PENALTY = -30`

Future calibration proposals may adjust constants post-MVP only after PRD update + experiment sign-off.

---

## 5) Supabase RLS matrix

Owner = `auth.uid() = user_id`.

- `user_profile`, `goal_plan`, `equipment_profile`, `equipment_profile_item`, `user_exercise_preference`, `user_limitation`, `workout_session`, `post_workout_check_in`, `recommendation`: owner CRUD within state-machine constraints.
- `exercise_catalog`, `plan_version`, `workout_day`, `exercise_instance`, `set_prescription`: service-role write, owner read where applicable.
- `set_log`: owner select + insert, no owner update/delete.
- `notification_preference`:
  - owner: select/insert/update/delete own row
  - service-role: read-only and only for notification dispatch jobs

Mandatory FK ownership-chain validations:
- child `user_id` must equal ancestor `user_id`
- cross-user FK writes fail with `foreign_key_ownership_violation`

---

## 6) Regeneration, local drafts, and non-destructive guarantees

Hard rule (MVP): if a **client-only unsynced local draft** exists for a workout/session impacted by plan regeneration, equipment revalidation, or missed-workout recovery regeneration, the client must block destructive action and require explicit user choice:
1. recover/sync draft
2. finish as partial
3. discard draft

Additional guarantees:
- do not silently replace affected workouts while unsynced draft exists
- server-side `in_progress` sessions remain attached to original `planVersionId`
- recovered/synced draft becomes normal `WorkoutSession` + `SetLog` records
- discard requires explicit user action
- discard never deletes already-synced `set_log` rows

---

## 7) Closed blocker resolutions and MVP decisions

1. Exercise catalog lock: 84 exercises for MVP.
2. Movement intent representation: `movement_intents text[]`, primary first.
3. Pain map artifact required: `pain_location_to_muscle_intent_v1.json`.
4. Accessory-cut artifact required: `accessory_cut_priority_v1.json`.
5. Top-rep constants and tie-breaks retained from PRD alignment.
6. Deload trigger retained from PRD alignment.
7. Anonymous sample-plan endpoint controls retained.
8. Auth decision: Supabase email/password only for MVP; Apple/Google fast-follow post-MVP after G3 approval.

---

## 8) Acceptance tests (expanded)

Required tests:
1. User exercise preference CRUD + uniqueness + RLS.
2. Limitation CRUD/privacy + analytics exclusion.
3. Notification preference defaults and PATCH behavior (quiet hours 21:00-08:00; streak milestones default false).
4. Generator preference/limitation scoring using constants in §4.
5. Substitution candidate endpoint ranking/explanations and algorithm constraints.
6. Substitution accept rejects non-candidate replacements unless manual override provided.
7. Substitution load-handling modes (history-based, blank load, bodyweight guidance, non-comparable machine guidance).
8. SetLog uniqueness `(user_id, workout_session_id, client_mutation_id)`.
9. Added set ordering preserved by `display_order`.
10. Offline replay preserves intra-exercise set order.
11. Duplicate idempotent replays do not create duplicate set rows.
12. Skipped prescribed sets retain correct display order.
13. Endpoint schema validation for all APIs in §3.
14. Version token 409 conflict coverage.
15. Idempotency-key behavior for required endpoints.
16. RLS policy enforcement for all tables, including `notification_preference` service-role read-only behavior.
17. Cross-user FK rejection tests for all ownership-chain tables.
18. Unsynced local draft protection on regenerate/equipment revalidation/recovery workflows.
19. Discard local draft does not delete already-synced set logs.
20. BodyWeightLog excluded from MVP implementation scope tests (no MVP dependency on trend history table).

---

## 9) Implementation readiness checklist

- [ ] PRD §0 pre-build gates passed.
- [ ] Machine artifacts present/versioned: movement-intent map, pain map, accessory-cut rules.
- [ ] Exercise seed set fixed at 84 with ownership sign-off.
- [ ] Data model and constraints implemented exactly per §2.
- [ ] API schemas/error contracts/idempotency/version semantics implemented per §3.
- [ ] SetLog ordering + idempotency behavior implemented per §2.12 and §3.8.
- [ ] Substitution candidate flow + explainability implemented per §3.9.
- [ ] Local draft non-destructive workflow implemented per §6.
- [ ] Supabase RLS policies implemented per §5 and validated by automated tests.
- [ ] Analytics pipeline excludes private free-text fields.
- [ ] Auth scope remains email/password only in MVP.
- [ ] BodyWeightLog excluded from MVP unless PRD changes.

**Implementation readiness:** READY TO BUILD once every checklist item above is checked and approved.
