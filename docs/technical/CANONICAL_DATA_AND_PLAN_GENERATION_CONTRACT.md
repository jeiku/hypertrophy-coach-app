# CANONICAL DATA AND PLAN GENERATION CONTRACT (FINAL MVP BUILD CONTRACT)

**Status:** Not ready for implementation until checklist in §9 is fully complete.  
**Source of truth:** `PRDv0.6.md`  
**Revision date:** 2026-05-10

---

## 0) Scope and implementation boundary

This contract defines MVP-only behavior for:
- canonical data model
- API request/response schemas
- idempotency/concurrency semantics
- ownership and Supabase RLS behavior
- deterministic generation and recovery rules
- acceptance tests and readiness checklist

Out of scope for MVP: nutrition entities, nutrition APIs, per-day equipment calendar logic, historical body-weight trend storage (`BodyWeightLog`, see §2.16).

---

## 1) Canonical enums (MVP)

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
- `exercise_preference_type`: `preferred | disliked | neutral`
- `limitation_type`: `mobility_limit | pain_history | injury_history | medical_restriction | equipment_constraint | other`
- `limitation_severity`: `low | moderate | high | hard_block`
- `limitation_status`: `active | resolved | archived`

---

## 2) Data model (MVP canonical)

All PKs are UUIDv7. All user-owned mutable tables must expose DB `version_token` and API `versionToken`.

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
- `version_token` (int not null default 1)

### 2.2 `goal_plan`
- `id` (pk)
- `user_id` (fk)
- `goal_type`
- `days_per_week` (int, 2..6)
- `preferred_weekdays` (`week_day[] not null`)  
  - allowed values: subset of enum `week_day`, unique items only, lowercase only.  
  - validation: cardinality must equal `days_per_week`.  
  - default when omitted: first `days_per_week` weekdays from Monday (`[monday..]`).
- `session_length_min` (int, 20..120)
- `focus_muscles` (`text[] not null default '{}'`)
- `is_active` (boolean not null default true)
- `version_token` (int not null default 1)

Generator usage of `preferred_weekdays`:
- primary day assignment and session spacing seed.
- recovery logic/regeneration preserves preferred weekdays when feasible.
- if conflict with availability constraints, generator returns warning in `validation_warnings_json` and applies nearest feasible day mapping.

### 2.3 `equipment_profile`
- `id` (pk), `user_id` (fk), `name`, `is_active`
- `version_token` (int not null default 1)

### 2.4 `equipment_profile_item`
- `id` (pk), `user_id` (fk), `equipment_profile_id` (fk), `equipment_key`
- uniqueness: `unique(equipment_profile_id, equipment_key)`

### 2.5 `exercise_catalog` (seed-owned)
Read-only to clients; service-role seed ownership.

### 2.6 `user_exercise_preference`
- includes `version_token` (int not null default 1)

### 2.7 `user_limitation`
- fields retained; add `version_token` (int not null default 1)

### 2.8 `notification_preference`
- `workout_reminder_minutes_before` (int not null **default 30**; allowed 5..1440)
- includes `version_token` (int not null default 1)

### 2.9 `plan_version`
- fields retained; add `version_token` (int not null default 1)

### 2.10 `workout_day`, `exercise_instance`, `set_prescription`
- `set_prescription` replaces `target_rpe` with:
  - `target_rir_min` (numeric(3,1) nullable)
  - `target_rir_max` (numeric(3,1) nullable)
  - validation: when either exists, both required and `0 <= min <= max <= 5`.

### 2.11 `workout_session`
- includes `version_token` (int not null default 1)

### 2.12 `set_log`
- replace `rpe` with `rir` (`numeric(3,1) nullable`, allowed `0..5`)
- `client_mutation_id` unique scope: `unique(user_id, workout_session_id, client_mutation_id)`

### 2.13 `post_workout_check_in`
- include `version_token` (int not null default 1)

### 2.14 `recommendation`
- include `version_token` (int not null default 1)

### 2.15 Version token contract
- Any successful update/state transition on mutable entities increments `version_token` by exactly +1 in same transaction.
- Any request with stale `versionToken` returns `409 VERSION_CONFLICT` with current token.

### 2.16 `BodyWeightLog` scope note
- `BodyWeightLog` is post-MVP/v1.1.  
- Cross-reference target: PRD section covering trend/history (not onboarding profile fields).

---

## 3) API contract requirements (global)

Every endpoint below must define:
- request body
- success response body
- error responses/status codes
- idempotency requirements
- versionToken behavior
- ownership/RLS expectation

Common error payload:
```json
{ "error": { "code": "string", "message": "string", "fieldErrors": [{"field":"string","message":"string"}] } }
```

`409 VERSION_CONFLICT` payload:
```json
{ "error": { "code":"VERSION_CONFLICT", "message":"Stale versionToken", "currentVersionToken": 7 } }
```

Naming rule: DB uses snake_case, API uses camelCase.

---

## 4) Endpoint matrix (implementation-ready summary)

### 4.1 Plans (`/v1/plans/*`)
- `POST /generate` (idempotent by `Idempotency-Key` + `clientRequestId`)  
  request includes `goalPlanId`, `equipmentProfileId`, `preferredWeekdays?` override; response returns draft `planVersion` with `versionToken`, warnings, schedule using preferred weekdays.
- `POST /{planVersionId}/accept` requires `versionToken`; returns activated plan with incremented token.
- `POST /{planVersionId}/regenerate` idempotent; requires `versionToken`; returns new draft and recovery warnings.
- `POST /{planVersionId}/revert` idempotent; requires `versionToken`; returns reverted active clone.

### 4.2 Workout sessions (`/v1/workout-sessions/*`)
- `start/resume/finish/partial/skip/completed-outside-app` each has explicit request body and returns session with incremented `versionToken`.
- all lifecycle transitions require `versionToken` except initial `start`.
- state-machine invalid transitions return `422 INVALID_STATE_TRANSITION`.

### 4.3 Set logs
- `POST /{id}/set-logs` supports prescribed/added/skipped; `rir` nullable; idempotent by `clientMutationId` uniqueness.

### 4.4 Missed-workout recovery
- all recovery actions require `versionToken`; idempotent via `Idempotency-Key` + `clientRequestId`; return updated session + any new planVersion references.

### 4.5 User limitations
- create/patch/resolve full schemas include `versionToken` on mutable operations.

### 4.6 Post-workout check-ins
- upsert requires `versionToken` and returns updated record with token.

### 4.7 Notification preferences
- default reminder 30 minutes; patch requires `versionToken`; response returns `versionToken`.

### 4.8 Recommendations actions
- accept/ignore/dismiss require `versionToken`; increment on success.

### 4.9 Equipment profile item deletion
- `DELETE /v1/equipment-profiles/{id}/items/{itemId}` requires parent profile `versionToken`; returns updated profile token.

### 4.10 Anonymous sample-plan generation
- `POST /v1/public/sample-plan/generate` (unauthenticated).
- request: onboarding-like inputs including `goalType`, `daysPerWeek`, `preferredWeekdays`, `sessionLengthMin`, `equipmentKeys`, `experienceLevel`.
- response: `samplePlan` only (no userId, no persistent planVersionId).
- rate limit: 10 requests/IP/hour.
- persistence: no account row writes; optional ephemeral cache ≤24h for abuse control only.
- client storage requirement: store response locally for 7 days with local timestamp.
- no conflict with authenticated generation: authenticated `/v1/plans/generate` always creates proper draft planVersion.

### 4.11 Local draft recovery
- `POST /v1/sync/local-drafts/recover` (idempotent).
- body must include either:
  - `workoutSessionId`, or
  - creation context: `planVersionId`, `workoutDayId`, `scheduledForDate`, `startedAt`, optional `notes`.
- supports skipped exercises/sets and partial completion flags.
- duplicate `clientMutationId` treated as replay and returned in `duplicates`.
- conflict response includes typed conflicts: `SESSION_ALREADY_COMPLETED`, `PLAN_VERSION_MISMATCH`, `WORKOUT_DAY_MISMATCH`, `STALE_VERSION_TOKEN`.

---

## 5) RIR-aligned progression and recommendation logic

- All intensity guidance uses RIR.
- `set_prescription.target_rir_min/max` feed expected effort band.
- `set_log.rir` (nullable) feeds progression and recommendation.
- when `rir` missing, progression falls back to reps/load outcomes only and emits reduced-confidence flag.

---

## 6) RLS, ownership, and FK enforcement

### 6.1 Policy intent by table operation
- Client writable (owner): `user_profile`, `goal_plan`, `equipment_profile`, `equipment_profile_item`, `user_exercise_preference`, `user_limitation`, `notification_preference`, `workout_session` (state transitions via RPC), `set_log` (insert only), `post_workout_check_in`, `recommendation` (action updates only).
- Service-role/RPC-only writes: `exercise_catalog`, `plan_version`, `workout_day`, `exercise_instance`, `set_prescription`.

### 6.2 Enforcement mechanism
- same-user and same-plan invariants enforced by DB constraints + BEFORE triggers on child tables; RPC layer performs mirrored validation and emits domain errors.

Required invariants:
- `workout_session.workout_day_id` belongs to `workout_session.plan_version_id`
- `exercise_instance` belongs to referenced `workout_day`
- `set_prescription` belongs to referenced `exercise_instance`
- `set_log.exercise_instance_id` and `set_log.set_prescription_id` are compatible with session plan/day
- every child `user_id` equals owning parent `user_id`

Violations return `422 FK_OWNERSHIP_VIOLATION` (same-user wrong plan/day) or `403` (cross-user access).

---

## 7) Acceptance tests (must-pass)

Add/retain tests for:
1. preferred weekdays validation/defaults and generator use.
2. plan generate/accept/regenerate/revert full schema and token behavior.
3. RIR-enabled flow (`target_rir_min/max` + logged `rir`) progression/recommendation update.
4. RIR-missing flow fallback and reduced-confidence marker.
5. version token: stale rejection + successful increment for all mutable entities (`equipment_profile`, `user_limitation`, `notification_preference`, `plan_version`, `post_workout_check_in`, `workout_session`, `recommendation`).
6. reminder default = 30 everywhere.
7. workout lifecycle endpoints complete schemas and invalid transition handling.
8. missed-workout recovery action schemas and idempotency.
9. equipment profile item delete updates parent token.
10. anonymous sample-plan endpoint rate-limit/no-persistence behavior.
11. local draft recovery before first server sync (no session id path).
12. local draft recovery after partial sync (existing session id path).
13. duplicate `clientMutationId` replay handling in recovery.
14. recovery conflict payload typing.
15. cross-user FK rejection.
16. same-user wrong-plan/wrong-day FK rejection.

---

## 8) Local draft non-destructive guarantees

If unsynced local draft exists for impacted workout/session, destructive operations must be blocked until user chooses recover/partial/discard.

---

## 9) Implementation readiness checklist (blocking gates)

### A) Contract-ready for engineering estimation
- [ ] PRD §0 pre-build gates complete.
- [ ] Canonical entities/fields and endpoint schemas complete with examples.
- [ ] RIR-only terminology complete.
- [ ] versionToken/version_token consistency complete.

### B) Ready for architecture freeze
- [ ] All PRD §18 open decisions resolved (blocking).
- [ ] RLS + FK enforcement design finalized (constraints/triggers/RPC responsibilities).
- [ ] Anonymous sample-plan architecture + abuse controls signed off.

### C) Ready for production implementation
- [ ] PRD §5.5 content workstream complete before build week 1 (content author identified, reviewer identified, seed exercise cue ownership confirmed).
- [ ] Exercise seed artifacts finalized: exact 84-exercise seed file, movement-intent map, pain map, accessory-cut rules, cue fields.
- [ ] Acceptance test suite in §7 green.

**Verdict rule:** Implementation is only ready when A+B+C are fully checked.
