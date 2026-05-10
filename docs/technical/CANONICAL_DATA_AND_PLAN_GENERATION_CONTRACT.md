# CANONICAL DATA + GENERATION + PROGRESSION CONTRACT (MVP)

**Status:** Implementation-ready (Final)  
**Source of truth:** `PRDv0.6.md`  
**Revision date:** 2026-05-10

---

## 0) Scope and boundary

This contract is the executable specification for MVP database schema, generator, progression/recommendation engine, substitution behavior, set logging semantics, sync/recovery, API surfaces, RLS/ownership, and acceptance tests.

**Explicitly out of MVP:**
- Nutrition entities/APIs.
- Historical body-weight trend storage (`BodyWeightLog` or equivalent).

Only `user_profile.body_weight` (current value) is in MVP.

---

## 1) Canonical enums

- `experience_level`: `beginner | intermediate`
- `goal_type`: `bulk | cut | recomp`
- `sex`: `male | female`
- `weight_unit`: `kg | lb`
- `week_day`: `monday | tuesday | wednesday | thursday | friday | saturday | sunday`
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

---

## 2) Data model (typed, no vague `metadata_json`)

All IDs: UUIDv7. API camelCase, DB snake_case. Mutable rows include `version_token int >=1`.

### 2.1 `exercise_catalog` (seed-owned; service role writes)
Required columns:
- `id uuid pk`
- `exercise_key text unique not null`
- `name text not null`
- `movement_intents text[] not null` (non-empty; values from §3.1)
- `primary_muscles text[] not null` (non-empty)
- `secondary_muscles text[] not null default '{}'`
- `equipment_required boolean not null default true`
- `equipment_keys text[] not null` (empty only when `equipment_required=false`)
- `difficulty experience_level not null` (minimum experience level)
- `eligible_beginner boolean not null`
- `eligible_intermediate boolean not null`
- `fatigue_cost smallint not null check (fatigue_cost between 1 and 5)`
- `setup_complexity smallint not null check (setup_complexity between 1 and 5)`
- `tempo_seconds smallint not null check (tempo_seconds between 1 and 20)`
- `rest_seconds smallint not null check (rest_seconds between 15 and 300)`
- `warmup_overhead_seconds smallint not null check (warmup_overhead_seconds between 0 and 600)`
- `setup_overhead_seconds smallint not null check (setup_overhead_seconds between 0 and 600)`
- `cues_setup text not null check (char_length(cues_setup) between 10 and 500)`
- `cues_execution text not null check (char_length(cues_execution) between 10 and 500)`
- `cues_common_mistake text not null check (char_length(cues_common_mistake) between 10 and 500)`
- `cues_safety_note text not null check (char_length(cues_safety_note) between 10 and 500)`
- `substitution_tags text[] not null default '{}'`
- `is_compound boolean not null default false`
- `is_active boolean not null default true`

**Removed:** ambiguous `metadata_json`.

### 2.2 Other core tables
`user_profile`, `goal_plan`, `equipment_profile`, `equipment_profile_item`, `plan_version`, `workout_day`, `exercise_instance`, `set_prescription`, `workout_session`, `set_log`, `post_workout_check_in`, `recommendation` remain as canonical user-owned structure with FK ownership invariants and version tokens.

For `set_log` and `recommendation`, semantics are fixed in §§5 and 4.

---

## 3) Deterministic plan generator contract

### 3.1 Movement-intent slot taxonomy
Allowed movement intents:
`horizontal_push, vertical_push, horizontal_pull, vertical_pull, knee_dominant, hip_hinge, squat_pattern, unilateral_lower, core_anti_extension, core_anti_rotation, arms_biceps, arms_triceps, calves, rear_delt, lateral_delt`.

### 3.2 Split templates by days/week
- **2-day:** Full A / Full B
- **3-day:** Upper / Lower / Full
- **4-day:** Upper A / Lower A / Upper B / Lower B
- **5-day:** Push / Pull / Legs / Upper Hypertrophy / Lower Hypertrophy
- **6-day:** Push A / Pull A / Legs A / Push B / Pull B / Legs B

Experience modulation:
- beginner: fewer compounds/day and lower accessory density.
- intermediate: full template density.

### 3.3 Slot assignment algorithm (deterministic)
1. Build candidate pool by eligibility/equipment/safety rules.
2. Instantiate split template slots in canonical order.
3. Fill compounds first, then accessories.
4. For each slot, score candidates (see §3.10), select max score; break ties using §3.11.
5. Enforce cross-slot exclusion and compound reuse limits.
6. Run duration estimator; if over target, cut accessories by §3.8.
7. Emit plan-level explanations and warnings.

### 3.4 Cross-slot exclusion
Within one workout day, selected exercises cannot share same `exercise_key`. Also cannot select two exercises where primary movement intent is identical unless template explicitly marks duplicate intent slot (none in MVP templates).

### 3.5 Weekly compound reuse limits
For `is_compound=true`: max appearances/week by `exercise_key`:
- beginner: 2
- intermediate: 3

### 3.6 A/B variety enforcement
For paired A/B days, at least 50% of compound slots on B day must differ from A day `exercise_key` while preserving same movement intent family.

### 3.7 Unfillable-slot fallback order
If no candidate for a slot:
1) same movement intent, looser secondary muscle match  
2) adjacent intent family map (e.g., vertical_pull→horizontal_pull)  
3) bodyweight alternative if equipment missing  
4) drop lowest-priority accessory slot and emit warning

### 3.8 Session duration + accessory-cut
Estimated seconds = sum per set `(tempo_seconds * target_reps_midpoint + rest_seconds)` + exercise `warmup_overhead_seconds + setup_overhead_seconds`.
If estimated duration > `goal_plan.session_length_min` by >5 min, cut accessories in order:
`arms_biceps, arms_triceps, calves, rear_delt, lateral_delt, core_*`.

### 3.9 Focus-muscle protection + initial volume targets
If `goal_plan.focus_muscles` contains muscle M, generator must preserve minimum weekly direct sets:
- beginner: 8
- intermediate: 10
before any accessory cuts touching M.

Initial weekly direct-set targets:
- major muscle groups: beginner 8–12, intermediate 10–16
- minor groups: beginner 4–8, intermediate 6–12

### 3.10 Eligibility + scoring weights
Candidate eligible iff:
- difficulty compatible with user experience (`eligible_beginner`/`eligible_intermediate`)
- equipment satisfiable by active profile (or `equipment_required=false`)
- not blocked by active hard limitations/pain suppression

Score = weighted sum:
- movement-intent match: 40
- focus-muscle match: 20
- novelty vs last active plan: 10
- lower fatigue cost when session near duration limit: 10
- lower setup complexity when session near duration limit: 10
- user preference (liked/disliked): 10

### 3.11 Deterministic tie-breakers
If equal score, choose in order:
1) lowest `fatigue_cost`  
2) lowest `setup_complexity`  
3) lexicographically smallest `exercise_key`

### 3.12 Calibration checkpoint
After generation, validator checks:
- split shape matches days/week template
- weekly set targets within ranges
- estimated duration each day <= target+5
- focus muscles protected
- no exclusion/reuse violations

### 3.13 Plan explanation + warning output
`plan_version.validation_warnings_json` schema:
```json
[{"code":"UNFILLABLE_SLOT","severity":"warning","message":"string","workoutDayId":"uuid|null","movementIntent":"string|null"}]
```
`plan_version.explanation_json` (new required typed jsonb):
```json
{"splitType":"upper_lower_4","focusMusclesApplied":["chest"],"durationAdjustments":[{"workoutDayId":"uuid","cutIntents":["arms_biceps"]}]}
```

---

## 4) Progression + recommendation engine contract

### 4.1 Double progression
For each prescribed set range `[rep_min, rep_max]`:
- if user reaches `rep_max` for all active prescribed sets in two consecutive exposures at same load and RIR >= target min, recommend `add_load`.
- else if below top-rep condition and load tolerated, recommend `add_reps` until reaching `rep_max`.

### 4.2 Recommendation precedence
Order of evaluation: safety suppression > deload > substitute > reduce_load > hold_load > add_reps > add_load.

### 4.3 Threshold logic
- **top-rep condition:** all prescribed active sets at `performed_reps >= rep_max`.
- **hold threshold:** mixed performance not meeting top-rep and no safety/fatigue flags.
- **reduce threshold:** any two consecutive sessions with `performed_reps < rep_min` on >=50% sets OR RIR collapse to 0 with form breakdown.
- **deload threshold:** high fatigue OR moderate/high soreness for same region for 3 consecutive check-ins OR pain_flag true with joint/nerve_like.

### 4.4 Pain/safety + soreness/fatigue handling
If pain location intersects exercise primary muscles or limitation region:
- suppress `add_load` and `add_reps`.
- prefer `substitute_exercise` (same intent, lower joint stress) or `suggest_deload`.

### 4.5 Ignore-feedback handling
Ignored recommendation increments `ignore_count` (derived metric). If same recommendation type ignored 2 consecutive times on same target, next cycle downgrades aggressiveness by one step (`add_load`→`add_reps`, `add_reps`→`hold_load`).

### 4.6 Required reason object schema
`recommendation.reason_json` required:
```json
{
  "triggerFactors":["TOP_REP_HIT","PAIN_FLAG","HIGH_FATIGUE","LOW_PERFORMANCE","FORM_BREAKDOWN","IGNORED_RECENT"],
  "inputsUsed":["set_logs","set_prescription","post_workout_check_in","user_limitation","recent_recommendation_history"],
  "metrics":{"topRepHit":true,"sessionsEvaluated":2,"underRepRate":0.0},
  "userFacingReason":"template-rendered string",
  "educationalContext":"required non-empty string"
}
```
Allowed enum values are exactly as above.

`userFacingReason` must be generated from templates keyed by `recommendation_type` and include at least one numeric or categorical signal (e.g., rep range reached, fatigue high).

---

## 5) SetPrescription / SetLog exact semantics

1. `set_source='prescribed'` => `set_prescription_id` **required** and must belong to same `exercise_instance_id` and `workout_session.plan_version_id` chain.
2. `set_source='added'` => `set_prescription_id` **must be null**.
3. `set_log_index int not null` defines ordering. Read order: `set_log_index asc`, then `created_at asc`.
4. Active uniqueness: only one `set_log` with `set_state='active'` per (`workout_session_id`, `set_prescription_id`) where `set_prescription_id is not null`.
5. Edit behavior: editing existing logged set creates new row superseding prior row (`supersedes_set_log_id uuid null`), prior row state set to `deleted`.
6. Delete/skip behavior:
   - skip prescribed set => row with `set_state='skipped'`, prescribed source required.
   - delete added set => mark row `set_state='deleted'`.
7. Replay/idempotency: unique (`user_id`,`workout_session_id`,`client_mutation_id`) with payload hash check; same key different payload => `409 IDEMPOTENCY_KEY_REUSED_WITH_DIFFERENT_PAYLOAD`.
8. RPC validation enforces same-user chain across session, day, plan version, exercise instance, set prescription.

---

## 6) API contract (concrete MVP surfaces)

All endpoints require JWT auth except explicitly public. Mutations require `Idempotency-Key` and `versionToken` when updating existing mutable entity.

### 6.1 Profile and onboarding
- `GET /v1/profile` -> `{id,userId,bodyWeight,bodyWeightUnit,experienceLevel,timezone,versionToken,...}`
- `PATCH /v1/profile` request includes `versionToken` and mutable fields; `409 VERSION_CONFLICT` on stale token.
- `POST /v1/onboarding/complete` request `{profileVersionToken,goalPlan,equipmentProfile}` atomic transaction creating/updating required entities and setting `onboardingCompletedAt`.

### 6.2 Goal plan
- `POST /v1/goal-plan`
- `GET /v1/goal-plan/active`
- `PATCH /v1/goal-plan/{id}`

### 6.3 Equipment profile + items
- `POST /v1/equipment-profiles`
- `GET /v1/equipment-profiles/active`
- `PATCH /v1/equipment-profiles/{id}`
- `GET /v1/equipment-profiles/{id}/items`
- `POST /v1/equipment-profiles/{id}/items` `{equipmentKey,versionToken}`
- `DELETE /v1/equipment-profiles/{id}/items/{itemId}` `{versionToken}`

### 6.4 Exercise catalog
- `GET /v1/exercises?movementIntent=&equipmentKey=&difficulty=`
- `GET /v1/exercises/{exerciseId}` full typed fields including cues/timing/scoring metadata.

### 6.5 Plan surfaces
- `POST /v1/plans/generate`
- `GET /v1/plans/active`
- `GET /v1/plans/{planVersionId}`
- `GET /v1/plans/{planVersionId}/detail` (expanded days/exercises/sets)

### 6.6 Today dashboard
- `GET /v1/today` returns `{date,activePlanVersionId,todayWorkoutSession|null,nextWorkoutDay,recommendationsPendingCount}`.

### 6.7 Session + set logs + recovery
- `POST /v1/workout-sessions/start`
- `POST /v1/workout-sessions/{id}/set-logs/batch`
- `POST /v1/workout-sessions/{id}/finish|skip|mark-partial|complete-outside-app`
- `POST /v1/sync/local-drafts/recover` (full schema in §7)

### 6.8 Substitution and recommendations
- `POST /v1/substitutions/request` `{workoutSessionId,exerciseInstanceId,reasonCode,painLocation[],versionToken}` -> candidate list with reason objects.
- `POST /v1/substitutions/{requestId}/accept` applies replacement transactionally.
- `POST /v1/recommendations/generate` (server recompute)
- `GET /v1/recommendations?status=pending`
- `POST /v1/recommendations/{id}/accept|ignore|dismiss`

### 6.9 Privacy/account
- `POST /v1/account/delete` soft-delete then purge job schedule; response `{status:"scheduled",purgeAfter:"ISO8601"}`.

### 6.10 Public sample plan + restore support
- `POST /v1/public/sample-plan/generate` (no auth).
- `POST /v1/public/sample-plan/restore-to-user` (auth; converts local sample into persisted draft plan).

Errors are concrete envelope:
```json
{"error":{"code":"VERSION_CONFLICT","message":"...","details":{"currentVersionToken":7}}}
```

---

## 7) Local draft recovery contract

`POST /v1/sync/local-drafts/recover`

### Request path A (existing session)
```json
{"mode":"existing_session","workoutSessionId":"uuid","workoutSessionVersionToken":4,"localSetLogs":[...],"localCheckIn":{...}}
```

### Request path B (creation context)
```json
{"mode":"creation_context","planVersionId":"uuid","workoutDayId":"uuid","scheduledForDate":"2026-05-10","localSetLogs":[...],"localCheckIn":{...}}
```

### Response
```json
{"workoutSession":{"id":"uuid","status":"in_progress","versionToken":5},"mergeResult":{"created":3,"replayed":1,"superseded":1},"conflicts":[]}
```

### Conflict response (409)
```json
{"error":{"code":"RECOVERY_CONFLICT","details":{"reason":"SESSION_ALREADY_COMPLETED|PLAN_VERSION_MISMATCH|WORKOUT_DAY_MISMATCH|STALE_VERSION_TOKEN|MUTATION_PAYLOAD_MISMATCH"}}}
```

Rules:
- local-only added sets merged if chain valid.
- if server session `completed|partial|skipped|deleted`, no automatic mutation; return conflict with allowed actions.
- `planVersionId`/`workoutDayId` mismatch => conflict.
- same `clientMutationId` + different payload => hard conflict `MUTATION_PAYLOAD_MISMATCH`.

---

## 8) Seed validation + acceptance tests (must pass)

1. Deterministic generation: same inputs => byte-identical plan structure/warnings.
2. Split differences by 2/3/4/5/6 days and by experience level.
3. Cross-slot exclusion enforced.
4. Weekly compound reuse limits enforced.
5. A/B variety enforcement enforced.
6. Unfillable fallback path triggers expected warnings.
7. Duration estimator and accessory cuts enforce session cap.
8. Focus-muscle protection preserves minimum direct sets.
9. Pain-location progression suppression blocks load/rep increases.
10. Substitution output includes complete reason object.
11. Progression thresholds: add-load, hold, reduce, deload produce correct recommendation types.
12. Recommendation reason object completeness (`triggerFactors`,`inputsUsed`,`userFacingReason`,`educationalContext`).
13. SetLog semantics: prescribed-set required IDs, added-set null IDs, duplicate active prescribed set rejection, edit supersession, skip/delete rules, stale version token rejection, duplicate `clientMutationId` replay behavior.
14. Local recovery tests for both request modes and all conflict codes.
15. Seed quality tests:
   - every movement intent has >=3 active exercises across each common equipment profile (`full_gym`,`dumbbell_bench`,`bodyweight_only`) or emits build-breaking seed validation failure.
   - every seeded exercise has all required cue/timing/scoring/eligibility fields.
16. API schema tests: no MVP response/request includes AI provenance fields (`aiModel`,`sourceLLM`,`prompt`).
17. RLS/ownership chain tests for all FK-linked writes.

---

## 9) Readiness checklist (complete)

- [x] All MVP schemas concrete, typed, and ownership/versioned.
- [x] Generator fully specified (taxonomy, templates, scoring, ties, duration, fallbacks, warnings, explanations).
- [x] Progression/recommendation engine fully specified.
- [x] SetPrescription/SetLog semantics fully specified.
- [x] API surfaces complete with concrete schemas/errors/idempotency/concurrency notes.
- [x] Local draft recovery fully specified with both request modes and conflict models.
- [x] Acceptance tests cover generator/progression/substitution/recovery and schema constraints.
- [x] Nutrition and historical body-weight logs excluded from MVP.

