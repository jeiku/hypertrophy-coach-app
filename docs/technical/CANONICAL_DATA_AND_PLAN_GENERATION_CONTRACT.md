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

Out of scope for MVP: nutrition entities, nutrition APIs, per-day equipment calendar logic.

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
- `focus_muscles` (text[] not null, non-empty)
- unique partial: one active goal plan per user (`is_active=true`)
- `is_active` (boolean not null default true)

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

### 2.6 `user_exercise_preference` (NEW MVP blocker resolved)
- `id` (pk)
- `user_id` (fk)
- `exercise_id` (fk -> exercise_catalog.id)
- `preference_type` (`preferred|disliked|neutral`)
- `source` (text default `user_explicit`)
- `notes` (text nullable, private, analytics-excluded)
- uniqueness: `unique(user_id, exercise_id)`
- generator use:
  - preferred: positive score boost during slot ranking
  - disliked: penalty; if no alternatives and constraints fail, substitution candidate set built from same movement intent/equipment compatibility
  - neutral: no bias

### 2.7 `user_limitation` (NEW MVP blocker resolved)
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
- privacy: only owner + service role
- eligibility effect:
  - `hard_block`: exercise ineligible if catalog movement intent intersects affected intents OR pain map denies location
  - `high`: strong penalty and substitution-first
  - `moderate|low`: score penalty only

### 2.8 `plan_version`
- `id`, `user_id`, `goal_plan_id`
- `based_on_plan_version_id` (nullable)
- `version_number` (int)
- `status` (`draft|active|archived|reverted`)
- `generate_mode` (`initial|regenerate|revert_clone`)
- `input_snapshot_json`, `output_snapshot_json`, `validation_warnings_json`
- uniqueness: one active plan per user (`unique(user_id) where status='active'`)

### 2.9 `workout_day`, `exercise_instance`, `set_prescription`
- `workout_day`: links to `plan_version`, holds day order/schedule metadata
- `exercise_instance`: links to `workout_day`, references `exercise_catalog`, stores slot metadata and trim priority
- `set_prescription` fields:
  - `id`, `user_id`, `exercise_instance_id` fk, `set_order`, `target_reps_min`, `target_reps_max`, `target_rpe` nullable, `set_type` text default `working`
  - uniqueness: `unique(exercise_instance_id, set_order)`

### 2.10 `workout_session`
- `id`, `user_id`, `plan_version_id`, `workout_day_id`
- `scheduled_for_date` (date)
- `status` enum
- `started_at`, `finished_at` nullable
- `version_token` (int default 1)

### 2.11 `set_log` (fully specified)
- `id` (pk)
- `user_id` (fk)
- `workout_session_id` (fk)
- `exercise_instance_id` (fk)
- `set_prescription_id` (fk nullable)
- `set_source` (`prescribed|added`)
- `set_state` (`active|skipped|deleted`)
- `reps` (int nullable when skipped)
- `load` (numeric(6,2) nullable)
- `rpe` (numeric(3,1) nullable)
- `client_mutation_id` (text not null)
- `logged_at` (timestamptz default now())

Rules:
- `set_prescription_id` nullable **only** when `set_source='added'`; required for prescribed/skipped prescribed sets.
- Added set linkage: must always include `exercise_instance_id`; may optionally include nearest preceding prescribed set for UI context only (not required in DB).
- idempotency uniqueness scope: `unique(user_id, workout_session_id, client_mutation_id)`.
- duplicate replay response: HTTP 200 with original persisted resource and `idempotentReplay=true`.
- skipped prescribed set behavior: create `set_log` with `set_source='prescribed'`, `set_state='skipped'`, matching `set_prescription_id`, null reps/load allowed.

### 2.12 `post_workout_check_in`
- `id`, `user_id`, `workout_session_id` (unique)
- `soreness_level`
- `soreness_locations` pain_location[] nullable
- `pain_flag` boolean
- `pain_locations` pain_location[] required when `pain_flag=true`
- `pain_type` required when `pain_flag=true`
- `pain_notes` text nullable (analytics-excluded)
- `fatigue_level` nullable
- `form_breakdown` boolean nullable

### 2.13 `recommendation`
- `id`, `user_id`, `workout_session_id` nullable, `exercise_instance_id` nullable
- `type`, `status`
- `title`, `user_facing_reason`, `educational_context`
- `trigger_factors` text[]
- `inputs_used_json` jsonb
- `suggested_change_json` jsonb
- `ignored_reason` enum nullable
- `other_reason_text` text nullable (private, analytics-excluded)
- `version_token` int default 1

---

## 3) API contracts (implementation-ready)

All endpoints require Bearer auth unless marked Anonymous. Errors use `{ code, message, fieldErrors? }`. Validation errors return 422. Auth failures return 401. Ownership violations return 403. Not found returns 404. Conflict/version issues return 409. 

### 3.1 Equipment profile + items
- `POST /v1/equipment-profiles`
  - body: `{ name: string, isActive?: boolean }`
  - response 201: `{ profile: { id,userId,name,isActive,createdAt,updatedAt } }`
  - side effect: if `isActive=true`, other profiles set inactive transactionally.
- `PATCH /v1/equipment-profiles/{id}`
  - body: `{ name?: string, isActive?: boolean, versionToken: number }`
  - idempotency: same payload/version replay returns 200 unchanged.
- `DELETE /v1/equipment-profiles/{id}`
  - soft delete; reject if only active profile without replacement.
- `POST /v1/equipment-profiles/{id}/items`
  - body: `{ equipmentKey: string }`
  - response 201 `{ item }`, 409 on duplicate equipment key in same profile.
- `DELETE /v1/equipment-profiles/{id}/items/{itemId}` soft delete.

### 3.2 User exercise preferences
- `PUT /v1/users/me/exercise-preferences/{exerciseId}`
  - body: `{ preferenceType: 'preferred'|'disliked'|'neutral', notes?: string, clientMutationId: string }`
  - idempotency: unique `(user, endpoint target, clientMutationId)`; replay returns prior 200 with `idempotentReplay=true`.
- `GET /v1/users/me/exercise-preferences`
  - response `{ items:[...] }`
- side effects: generator ranking cache invalidation for current user.

### 3.3 User limitations
- `POST /v1/users/me/limitations`
  - body: `{ limitationType, painLocations?:[], affectedMovementIntents?:[], severity, notes?:string, startedOn?:date }`
- `PATCH /v1/users/me/limitations/{id}`
  - body includes mutable fields + `versionToken`
- `POST /v1/users/me/limitations/{id}/resolve`
  - body `{ resolvedOn?:date, versionToken:number }`
- side effects: exercise eligibility cache invalidation.

### 3.4 Plans: generate / accept / regenerate / revert
- `POST /v1/plans/generate`
  - body: `{ goalPlanId, equipmentProfileId, clientRequestId }`
  - idempotency by `clientRequestId` 24h TTL; replay returns same draft plan.
- `POST /v1/plans/{planVersionId}/accept`
  - body: `{ expectedCurrentActivePlanVersionId?:uuid, versionToken:number }`
  - transactional side effect: promote draft->active, archive prior active.
- `POST /v1/plans/{planVersionId}/regenerate`
  - body: `{ reason:'manual'|'missed_workout_recovery', protectInProgress:true, clientRequestId }`
  - response new draft.
- `POST /v1/plans/{planVersionId}/revert`
  - body: `{ versionToken:number, clientRequestId }`
  - side effect: new active `revert_clone`, prior active archived.

### 3.5 Workout session lifecycle
- `POST /v1/workout-sessions/start` body `{ workoutDayId, scheduledForDate, clientRequestId }`
- `POST /v1/workout-sessions/{id}/resume` body `{ versionToken }`
- `POST /v1/workout-sessions/{id}/finish` body `{ versionToken, finishedAt? }`
- `POST /v1/workout-sessions/{id}/partial` body `{ versionToken, reason?:string }`
- `POST /v1/workout-sessions/{id}/skip` body `{ versionToken, reason?:string }`
- `POST /v1/workout-sessions/{id}/completed-outside-app` body `{ versionToken, notes?:string }`
- concurrency: optimistic lock on `versionToken`; server increments on status change.

### 3.6 Set logging / added sets / skipped sets
- `POST /v1/workout-sessions/{id}/set-logs`
  - body:
    - prescribed log: `{ exerciseInstanceId, setPrescriptionId, reps, load?, rpe?, clientMutationId }`
    - added set: `{ exerciseInstanceId, setSource:'added', reps, load?, rpe?, clientMutationId }`
    - skipped prescribed: `{ exerciseInstanceId, setPrescriptionId, setState:'skipped', clientMutationId }`
  - validation:
    - `setPrescriptionId` required unless added set.
    - exercise/session ownership chain must match.
  - duplicate replay: 200 + original row + `idempotentReplay=true`.

### 3.7 Post-workout check-ins
- `PUT /v1/workout-sessions/{id}/check-in`
  - body `{ sorenessLevel, sorenessLocations?, painFlag, painLocations?, painType?, painNotes?, fatigueLevel?, formBreakdown?, versionToken }`
  - validation: pain fields required iff `painFlag=true`.
  - side effects: recommendation generation job enqueue.

### 3.8 Missed-workout recovery actions
- `POST /v1/workout-sessions/{id}/recovery/move-next`
- `POST /v1/workout-sessions/{id}/recovery/skip-continue`
- `POST /v1/workout-sessions/{id}/recovery/completed-outside-app`
- `POST /v1/workout-sessions/{id}/recovery/regenerate-week`
- body `{ versionToken, clientRequestId }` each; all idempotent by clientRequestId.

### 3.9 Exercise substitutions
- `POST /v1/workout-sessions/{id}/substitutions`
  - body `{ exerciseInstanceId, replacementExerciseId, reason:'preference'|'limitation'|'pain'|'equipment', versionToken, clientRequestId }`
  - side effects: replacement must preserve movement intent slot + equipment feasibility.

### 3.10 Recommendation actions
- `POST /v1/recommendations/{id}/accept` body `{ versionToken }`
- `POST /v1/recommendations/{id}/ignore` body `{ versionToken, ignoredReason?, otherReasonText? }`
- `POST /v1/recommendations/{id}/dismiss` body `{ versionToken, ignoredReason?, otherReasonText? }`
- concurrency: optimistic lock on recommendation `version_token`.

### 3.11 Local draft sync recovery
- `POST /v1/sync/local-drafts/recover`
  - body `{ localDraftId, workoutSessionId?, setLogs:[...], skippedSets:[...], clientSyncBatchId }`
  - idempotency by `(user, clientSyncBatchId)`.
  - response includes accepted logs, duplicates, conflicts.

---

## 4) Generator behavior integration for preferences and limitations

Scoring per candidate exercise for a slot:
- base score from PRD template fit
- `+PREFERRED_BOOST` for `preferred`
- `-DISLIKED_PENALTY` for `disliked`
- limitation penalties/exclusions applied after preference
- ineligible if limitation severity `hard_block` or pain-map disallow
- substitution fallback chooses same movement intent, same day equipment, nearest fatigue cost

Machine constants (MVP):
- `PREFERRED_BOOST = +12`
- `DISLIKED_PENALTY = -18`
- `LIMITATION_LOW_PENALTY = -8`
- `LIMITATION_MODERATE_PENALTY = -16`
- `LIMITATION_HIGH_PENALTY = -30`

---

## 5) Supabase RLS matrix (table-by-table)

Legend: Owner = `auth.uid() = user_id`.

- `user_profile`: select owner; insert owner only; update owner; delete soft-delete owner; service-role may hard-delete for legal deletion.
- `goal_plan`: select/insert/update/delete owner; service-role may activate/archive transactionally.
- `equipment_profile`: select/insert/update/delete owner; unique-active enforced DB-side.
- `equipment_profile_item`: select/insert/update/delete owner; insert/update require parent profile owner match.
- `exercise_catalog`: select all authenticated; insert/update/delete service-role only.
- `user_exercise_preference`: select/insert/update/delete owner only; fk exercise must exist.
- `user_limitation`: select/insert/update/delete owner only; notes private; service-role read for generation only.
- `plan_version`: select owner; insert/update service-role only; delete forbidden to clients (archive instead).
- `workout_day`: select owner; insert/update/delete service-role only.
- `exercise_instance`: select owner; insert/update/delete service-role only.
- `set_prescription`: select owner; insert/update/delete service-role only.
- `workout_session`: select owner; insert service-role and owner-start endpoint; update owner constrained by state machine; delete soft-delete owner when `not_started` only.
- `set_log`: select owner; insert owner/service-role; update/delete forbidden except service-role maintenance for GDPR redaction envelope.
- `post_workout_check_in`: select/insert/update owner; delete soft-delete owner.
- `recommendation`: select owner; update owner for status transitions only; insert service-role; delete owner dismiss->soft-delete.

Mandatory FK ownership-chain validations (DB constraints/triggers):
- any child row with `user_id` must equal ancestor chain owner.
- cross-user FK attempts must fail with `foreign_key_ownership_violation`.

Cross-user FK rejection tests required for every child table in acceptance suite.

---

## 6) Closed blocker resolutions

1. **Exercise catalog count + seed ownership:** lock at **84 exercises** for MVP; seed artifact owned by Content + Head Coach; DB seed writes service-role only.
2. **Movement intent representation:** canonical storage = `movement_intents text[]` with mandatory primary intent first; PRD slot map artifact versioned as `movement_intent_map_v1.json`.
3. **PRD §7.3.2 pain mapping machine artifact:** required file `pain_location_to_muscle_intent_v1.json` with semantic version + reviewer sign-off.
4. **Accessory-cut ordered rules artifact:** required file `accessory_cut_priority_v1.json` consumed by generator.
5. **Top-rep condition constants:** top set qualifies when `actual_reps >= target_reps_max - 1`; if rep range width > 3, threshold=`target_reps_max`; ties broken by last two sessions median.
6. **Deload signals constants:** signals = performance drop, soreness high, pain flag, fatigue high, form breakdown; trigger when >=3 signals in rolling 7 days.
7. **Anonymous sample-plan endpoint controls:** 10 req/day/device fingerprint, burst 3/min/IP, hCaptcha after anomaly score threshold, telemetry excludes free text + no durable identity join.

---

## 7) PRD §18 decisions included in readiness checklist

- Legal/privacy deletion text approved and implemented as user-facing policy string.
- Safety disclaimer copy approved by legal + coach reviewer.
- Apple/Google sign-in decision finalized (both enabled in MVP).
- Pricing validation complete against store metadata and backend entitlements.
- WCAG audit owner assigned (Design Ops) with monthly cadence.
- Content author/reviewer identified for exercise cues, safety notes, and education cards.

---

## 8) Acceptance tests (expanded)

Required tests:
1. User exercise preference CRUD + uniqueness (`user_id, exercise_id`) + RLS.
2. Limitation persistence CRUD, privacy, and analytics exclusion for notes.
3. Generator preference/limitation scoring effects and substitution eligibility.
4. SetLog uniqueness scope `(user_id, workout_session_id, client_mutation_id)`.
5. SetLog duplicate replay returns existing row + `idempotentReplay=true`.
6. Added set requires exerciseInstance linkage; prescribed set requires setPrescriptionId.
7. Skipped prescribed set writes `set_state='skipped'` with nullable reps/load.
8. Endpoint schema validation for all APIs in §3 (request + response JSON schema).
9. Concurrency token conflict coverage (`versionToken` 409 paths).
10. Idempotency key behavior coverage (`clientRequestId`, `clientMutationId`, `clientSyncBatchId`).
11. Table-by-table RLS policy enforcement tests from §5.
12. Cross-user FK rejection tests for all ownership-chain tables.
13. Analytics PII exclusion tests (pain notes, limitation notes, ignored otherReasonText excluded).
14. Account deletion + retention behavior (soft-delete timings, legal hold exceptions, hard-delete path).
15. Anonymous sample-plan rate-limit and abuse-control behavior tests.

---

## 9) Implementation readiness checklist

- [ ] PRD §0 pre-build gates passed.
- [ ] All machine artifacts present/versioned: movement-intent map, pain map, accessory-cut rules.
- [ ] Exercise seed set fixed at 84 with approved ownership sign-off.
- [ ] Data model tables and constraints implemented exactly per §2.
- [ ] API schemas and error contracts implemented exactly per §3.
- [ ] SetLog/setPrescription linkage and idempotency behavior implemented per §2.11.
- [ ] Supabase RLS policies implemented per §5 and validated by automated tests.
- [ ] Analytics pipeline excludes all private free-text fields.
- [ ] Account deletion/retention policy implemented and verified.
- [ ] Legal safety disclaimer, social sign-in decision, pricing validation, WCAG owner/cadence, and content ownership recorded.

**Implementation readiness:** READY TO BUILD once every checklist item above is checked and approved.
