# CANONICAL DATA AND PLAN GENERATION CONTRACT (FINAL MVP BUILD CONTRACT)

**Status:** Draft contract revision for implementation readiness (see §10 gates).  
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

Out of scope for MVP: nutrition entities/APIs, per-day equipment calendar logic, historical body-weight trend storage (`body_weight_log`, post-MVP).

---

## 1) Canonical enums (DB)

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

## 2) Canonical data model (implementation-ready)

Conventions:
- IDs are `uuid` (UUIDv7 generated in application/service layer).
- Timestamps are `timestamptz` in UTC.
- API uses camelCase; DB uses snake_case.
- Mutable user-owned entities expose `version_token` in DB and `versionToken` in API.
- `updated_at` set by trigger on every successful write.

### 2.1 `user_profile`
**Purpose:** User onboarding/profile data and scheduling timezone.

Columns:
- `id uuid pk`
- `user_id uuid not null unique fk auth.users(id) on delete cascade`
- `birth_date date null`
- `sex sex null`
- `height_cm numeric(5,2) null check (height_cm between 100 and 260)`
- `body_weight numeric(6,2) null check (body_weight between 25 and 400)`
- `body_weight_unit weight_unit null`
- `experience_level experience_level null`
- `timezone text not null default 'UTC'`
- `onboarding_completed_at timestamptz null`
- `created_at timestamptz not null default now()`
- `updated_at timestamptz not null default now()`
- `version_token integer not null default 1 check (version_token >= 1)`

Constraints/indexes:
- `check ((body_weight is null and body_weight_unit is null) or (body_weight is not null and body_weight_unit is not null))`
- index on `(user_id)` unique already covers lookup.

Ownership/versioning:
- owner is `user_id`.
- any mutable update increments `version_token +1`.

API mapping exposed: `bodyWeightUnit`, `experienceLevel`, `onboardingCompletedAt`, `versionToken`.

### 2.2 `goal_plan`
**Purpose:** User goal and weekly scheduling preferences.

Columns:
- `id uuid pk`
- `user_id uuid not null fk auth.users(id) on delete cascade`
- `goal_type goal_type not null`
- `days_per_week smallint not null check (days_per_week between 2 and 6)`
- `preferred_weekdays week_day[] not null`
- `session_length_min smallint not null check (session_length_min between 20 and 120)`
- `focus_muscles text[] not null default '{}'`
- `is_active boolean not null default true`
- `created_at timestamptz not null default now()`
- `updated_at timestamptz not null default now()`
- `version_token integer not null default 1 check (version_token >= 1)`

Constraints/indexes:
- partial unique index: one active goal per user: `unique (user_id) where is_active = true`
- check: `cardinality(preferred_weekdays) = days_per_week`
- check: `preferred_weekdays` values unique (trigger validation).
- index on `(user_id, is_active)`.

Versioning: update increments token.

### 2.3 `equipment_profile`
**Purpose:** Active equipment context used by generation.

Columns:
- `id uuid pk`
- `user_id uuid not null fk auth.users(id) on delete cascade`
- `name text not null check (char_length(name) between 1 and 80)`
- `is_active boolean not null default true`
- `created_at timestamptz not null default now()`
- `updated_at timestamptz not null default now()`
- `version_token integer not null default 1 check (version_token >= 1)`

Constraints/indexes:
- partial unique: one active profile per user: `unique (user_id) where is_active = true`
- index `(user_id, is_active)`.

Versioning:
- direct updates increment token.
- create/update/delete in `equipment_profile_item` also increments parent profile token in same transaction.

### 2.4 `equipment_catalog` (canonical taxonomy)
**Purpose:** Source of valid `equipment_key` values for MVP.

Columns:
- `equipment_key text pk` (e.g., `barbell`, `dumbbell`, `bench_flat`, `cable_machine`)
- `display_name text not null`
- `is_active boolean not null default true`
- `sort_order integer not null default 0`
- `created_at timestamptz not null default now()`

Rules:
- seed-owned, service-role writes only.
- `equipment_profile_item.equipment_key` must exist in this table.

### 2.5 `equipment_profile_item`
**Purpose:** Junction of user equipment profile and allowed equipment keys.

Columns:
- `id uuid pk`
- `user_id uuid not null fk auth.users(id) on delete cascade`
- `equipment_profile_id uuid not null fk equipment_profile(id) on delete cascade`
- `equipment_key text not null fk equipment_catalog(equipment_key)`
- `created_at timestamptz not null default now()`

Constraints/indexes:
- `unique (equipment_profile_id, equipment_key)`
- index `(user_id, equipment_profile_id)`

Ownership:
- `user_id` must equal parent `equipment_profile.user_id` (trigger-enforced).

### 2.6 `exercise_catalog` (seed-owned)
**Purpose:** Canonical exercises used in generation and substitutions.

Columns:
- `id uuid pk`
- `exercise_key text not null unique`
- `name text not null`
- `primary_muscles text[] not null`
- `secondary_muscles text[] not null default '{}'`
- `movement_pattern text not null`
- `equipment_keys text[] not null`
- `experience_level_min experience_level not null`
- `is_unilateral boolean not null default false`
- `is_active boolean not null default true`
- `metadata_json jsonb not null default '{}'::jsonb`
- `created_at timestamptz not null default now()`
- `updated_at timestamptz not null default now()`

Constraints/indexes:
- GIN indexes on `primary_muscles`, `equipment_keys`.
- check arrays non-empty where required.

Seed count contract:
- MVP ships **~80 curated exercises**; exact count is product-controlled seed decision and may vary slightly.

### 2.7 `user_exercise_preference`
**Purpose:** User likes/dislikes for exercise personalization.

Columns:
- `id uuid pk`
- `user_id uuid not null fk auth.users(id) on delete cascade`
- `exercise_catalog_id uuid not null fk exercise_catalog(id)`
- `preference_type exercise_preference_type not null`
- `notes text null check (char_length(notes) <= 500)`
- `created_at timestamptz not null default now()`
- `updated_at timestamptz not null default now()`
- `version_token integer not null default 1`

Constraints/indexes:
- `unique (user_id, exercise_catalog_id)`
- index `(user_id, preference_type)`.

### 2.8 `user_limitation`
**Purpose:** Safety/constraint inputs affecting planning and recommendations.

Columns:
- `id uuid pk`
- `user_id uuid not null fk auth.users(id) on delete cascade`
- `limitation_type limitation_type not null`
- `severity limitation_severity not null`
- `status limitation_status not null default 'active'`
- `title text not null check (char_length(title) between 1 and 120)`
- `details text null check (char_length(details) <= 1000)`
- `affected_muscles text[] not null default '{}'`
- `affected_equipment_keys text[] not null default '{}'`
- `resolved_at timestamptz null`
- `created_at timestamptz not null default now()`
- `updated_at timestamptz not null default now()`
- `version_token integer not null default 1`

Constraints:
- `check ((status = 'resolved') = (resolved_at is not null))`
- index `(user_id, status)`.

### 2.9 `notification_preference`
**Purpose:** Workout reminders and quiet-hours behavior.

Columns:
- `id uuid pk`
- `user_id uuid not null unique fk auth.users(id) on delete cascade`
- `push_opt_in boolean not null default true`
- `workout_reminder_enabled boolean not null default true`
- `workout_reminder_minutes_before integer not null default 30 check (workout_reminder_minutes_before between 5 and 1440)`
- `quiet_hours_enabled boolean not null default true`
- `quiet_hours_start_local time not null default '21:00:00'`
- `quiet_hours_end_local time not null default '08:00:00'`
- `timezone text not null` (must match `user_profile.timezone` on write)
- `created_at timestamptz not null default now()`
- `updated_at timestamptz not null default now()`
- `version_token integer not null default 1`

Rules:
- quiet-hours window may wrap midnight (default 9pm–8am local).
- scheduler must evaluate quiet hours in user timezone.

### 2.10 `plan_version`
**Purpose:** Immutable generated plan snapshot metadata + lifecycle.

Columns:
- `id uuid pk`
- `user_id uuid not null fk auth.users(id) on delete cascade`
- `goal_plan_id uuid not null fk goal_plan(id)`
- `equipment_profile_id uuid not null fk equipment_profile(id)`
- `status plan_version_status not null default 'draft'`
- `generation_reason text not null` (`initial|regenerate|recovery|revert_clone` values via check)
- `source_plan_version_id uuid null fk plan_version(id)`
- `validation_warnings_json jsonb not null default '[]'::jsonb`
- `accepted_at timestamptz null`
- `archived_at timestamptz null`
- `created_at timestamptz not null default now()`
- `updated_at timestamptz not null default now()`
- `version_token integer not null default 1`

Constraints/indexes:
- one active plan version per user: partial unique `(user_id) where status='active'`
- index `(user_id, status, created_at desc)`

Behavior:
- workouts tied to a plan version remain tied permanently.
- regeneration creates new row; no mutation of historical planned workout structure.

### 2.11 `workout_day`
**Purpose:** Planned days inside a plan version.

Columns:
- `id uuid pk`
- `user_id uuid not null fk auth.users(id) on delete cascade`
- `plan_version_id uuid not null fk plan_version(id) on delete cascade`
- `day_index smallint not null check (day_index between 1 and 14)`
- `week_day week_day not null`
- `title text not null`
- `estimated_duration_min smallint not null check (estimated_duration_min between 20 and 120)`
- `created_at timestamptz not null default now()`

Constraints/indexes:
- `unique (plan_version_id, day_index)`
- index `(user_id, plan_version_id)`.

### 2.12 `exercise_instance`
**Purpose:** Planned exercise entries for each workout day.

Columns:
- `id uuid pk`
- `user_id uuid not null fk auth.users(id) on delete cascade`
- `workout_day_id uuid not null fk workout_day(id) on delete cascade`
- `exercise_catalog_id uuid not null fk exercise_catalog(id)`
- `sequence_no smallint not null check (sequence_no >= 1)`
- `notes text null`
- `created_at timestamptz not null default now()`

Constraints:
- `unique (workout_day_id, sequence_no)`
- index `(user_id, workout_day_id)`.

### 2.13 `set_prescription`
**Purpose:** Planned sets for each exercise instance.

Columns:
- `id uuid pk`
- `user_id uuid not null fk auth.users(id) on delete cascade`
- `exercise_instance_id uuid not null fk exercise_instance(id) on delete cascade`
- `set_number smallint not null check (set_number >= 1)`
- `rep_min smallint not null check (rep_min between 1 and 50)`
- `rep_max smallint not null check (rep_max between rep_min and 60)`
- `load_value numeric(6,2) null`
- `load_unit weight_unit null`
- `target_rir_min numeric(3,1) null`
- `target_rir_max numeric(3,1) null`
- `rest_seconds smallint null check (rest_seconds between 15 and 600)`
- `created_at timestamptz not null default now()`

Constraints:
- `unique (exercise_instance_id, set_number)`
- `check ((target_rir_min is null and target_rir_max is null) or (target_rir_min is not null and target_rir_max is not null and target_rir_min >= 0 and target_rir_max <= 5 and target_rir_min <= target_rir_max))`

### 2.14 `workout_session`
**Purpose:** User execution state for a scheduled workout day/date.

Columns:
- `id uuid pk`
- `user_id uuid not null fk auth.users(id) on delete cascade`
- `plan_version_id uuid not null fk plan_version(id)`
- `workout_day_id uuid not null fk workout_day(id)`
- `scheduled_for_date date not null`
- `status session_status not null default 'not_started'`
- `started_at timestamptz null`
- `ended_at timestamptz null`
- `notes text null`
- `created_at timestamptz not null default now()`
- `updated_at timestamptz not null default now()`
- `version_token integer not null default 1`

Constraints/indexes:
- `unique (user_id, workout_day_id, scheduled_for_date)`
- index `(user_id, status, scheduled_for_date desc)`

### 2.15 `set_log`
**Purpose:** Per-set performance logs for session history.

Columns:
- `id uuid pk`
- `user_id uuid not null fk auth.users(id) on delete cascade`
- `workout_session_id uuid not null fk workout_session(id) on delete cascade`
- `exercise_instance_id uuid not null fk exercise_instance(id)`
- `set_prescription_id uuid null fk set_prescription(id)`
- `set_source set_source not null`
- `set_state set_state not null default 'active'`
- `performed_reps smallint null check (performed_reps between 0 and 100)`
- `performed_load numeric(6,2) null`
- `load_unit weight_unit null`
- `rir numeric(3,1) null check (rir between 0 and 5)`
- `duration_seconds integer null check (duration_seconds between 0 and 7200)`
- `notes text null`
- `client_mutation_id text not null`
- `created_at timestamptz not null default now()`
- `updated_at timestamptz not null default now()`

Constraints/indexes:
- `unique (user_id, workout_session_id, client_mutation_id)`
- index `(workout_session_id, exercise_instance_id)`

### 2.16 `post_workout_check_in`
**Purpose:** Post-session recovery/safety check aligned with PRD.

Columns:
- `id uuid pk`
- `user_id uuid not null fk auth.users(id) on delete cascade`
- `workout_session_id uuid not null unique fk workout_session(id) on delete cascade`
- `soreness_level soreness_level not null`
- `soreness_location pain_location[] not null default '{}'`
- `pain_flag boolean not null default false`
- `pain_location pain_location[] not null default '{}'`
- `pain_type pain_type null`
- `pain_notes text null check (char_length(pain_notes) <= 1000)`
- `fatigue_level fatigue_level not null`
- `form_breakdown boolean not null default false`
- `created_at timestamptz not null default now()`
- `updated_at timestamptz not null default now()`
- `version_token integer not null default 1`

Constraints:
- if `pain_flag=false`: `pain_location` empty AND `pain_type` null.
- if `pain_flag=true`: at least one `pain_location` and non-null `pain_type`.

### 2.17 `recommendation`
**Purpose:** Actionable adaptation recommendations generated from training signals.

Columns:
- `id uuid pk`
- `user_id uuid not null fk auth.users(id) on delete cascade`
- `plan_version_id uuid not null fk plan_version(id)`
- `workout_session_id uuid null fk workout_session(id)`
- `type recommendation_type not null`
- `status recommendation_status not null default 'pending'`
- `title text not null check (char_length(title) between 1 and 120)`
- `user_facing_reason text not null`
- `educational_context text null`
- `trigger_factors jsonb not null default '{}'::jsonb`
- `inputs_used jsonb not null default '{}'::jsonb`
- `target_entity_type text not null check (target_entity_type in ('plan_version','workout_day','exercise_instance','set_prescription'))`
- `target_entity_id uuid not null`
- `suggested_change jsonb not null`
- `ignored_reason recommendation_ignored_reason null`
- `created_at timestamptz not null default now()`
- `updated_at timestamptz not null default now()`
- `version_token integer not null default 1`

Constraints/indexes:
- index `(user_id, status, created_at desc)`
- check `ignored_reason is null unless status='ignored'`.

---

## 3) API contract (full endpoints)

Common:
- Auth: `Authorization: Bearer <jwt>` unless marked public.
- `Content-Type: application/json` for bodies.
- optimistic concurrency: mutable endpoints require `versionToken` in body (except create/start where noted).
- idempotent mutating endpoints require `Idempotency-Key` header.
- common errors: `400`, `401`, `403`, `404`, `409 VERSION_CONFLICT`, `422`, `429`, `500`.

### 3.1 Plans
1) `POST /v1/plans/generate`
- Auth: required
- Headers: `Idempotency-Key` required
- Body example: `{ "goalPlanId":"...", "equipmentProfileId":"...", "clientRequestId":"...", "preferredWeekdays":["monday","wednesday","friday"] }`
- Success `201`: returns draft `planVersion`, nested `workoutDays/exercises/sets`, `validationWarnings`, `versionToken`.
- Idempotency: same user + key + semantically equal body returns same response.
- Transaction: create `plan_version` + children atomically.

2) `POST /v1/plans/{planVersionId}/accept`
- Headers: `Idempotency-Key` required
- Body: `{ "versionToken": 3 }`
- Success `200`: activated planVersion with incremented token.
- Transaction: archive prior active plan (if any), set current active.

3) `POST /v1/plans/{planVersionId}/regenerate`
- Headers: `Idempotency-Key` required
- Body: `{ "versionToken": 4, "reason":"equipment_change" }`
- Success `201`: new draft planVersion (new id), prior plan untouched.

4) `POST /v1/plans/{planVersionId}/revert`
- Headers: `Idempotency-Key` required
- Body: `{ "versionToken": 6 }`
- Success `201`: new active clone with status `reverted`.

### 3.2 Workout sessions
5) `POST /v1/workout-sessions/start`
- Headers: `Idempotency-Key` required
- Body: `{ "planVersionId":"...", "workoutDayId":"...", "scheduledForDate":"2026-05-10" }`
- Success `201`: session `{status:"in_progress", versionToken:1}`

6) `POST /v1/workout-sessions/{id}/resume`
- Body: `{ "versionToken": 2 }` -> `200` session `in_progress` token+1.

7) `POST /v1/workout-sessions/{id}/finish`
- Body: `{ "versionToken": 4, "endedAt":"..." }` -> `200` status `completed`.

8) `POST /v1/workout-sessions/{id}/mark-partial`
- Body: `{ "versionToken": 3, "notes":"..." }` -> status `partial`.

9) `POST /v1/workout-sessions/{id}/skip`
- Body: `{ "versionToken": 2, "reason":"time_conflict" }` -> status `skipped`.

10) `POST /v1/workout-sessions/{id}/complete-outside-app`
- Body: `{ "versionToken": 2, "notes":"Ran outdoors" }` -> status `completed_outside_app`.

Invalid transitions: `422 INVALID_STATE_TRANSITION`.

### 3.3 Set logs
11) `POST /v1/workout-sessions/{id}/set-logs`
- Headers: `Idempotency-Key` required
- Body: `{ "versionToken": 5, "entries":[{"clientMutationId":"abc-1","exerciseInstanceId":"...","setPrescriptionId":"...","setSource":"prescribed","setState":"active","performedReps":10,"performedLoad":60,"loadUnit":"kg","rir":2.0}] }`
- Success `200`: created/acknowledged set logs, duplicates array, session token+1.
- Duplicate `clientMutationId`: replayed as success.

### 3.4 Missed-workout recovery
12) `POST /v1/workout-sessions/{id}/recover-missed`
- Headers: `Idempotency-Key` required
- Body: `{ "versionToken": 4, "action":"reschedule|regenerate|skip_with_recovery", "targetDate":"2026-05-11" }`
- Success `200`: updated session and any new `planVersionId` created.

### 3.5 User limitations
13) `POST /v1/user-limitations` Body includes type/severity/title/details -> `201` + token=1.
14) `PATCH /v1/user-limitations/{id}` Body includes `versionToken` + patch fields -> `200` + token+1.
15) `POST /v1/user-limitations/{id}/resolve` Body `{versionToken}` -> `200` status resolved.

### 3.6 Post-workout check-ins
16) `PUT /v1/workout-sessions/{id}/post-workout-check-in`
- Headers: `Idempotency-Key` required
- Body: `{ "versionToken":1, "sorenessLevel":"moderate", "sorenessLocation":["lower_back"], "painFlag":true, "painLocation":["lower_back"], "painType":"dull_aching", "painNotes":"Lingering after RDL", "fatigueLevel":"high", "formBreakdown":true }`
- Success `200`: full check-in with incremented token.

### 3.7 Notification preferences
17) `GET /v1/notification-preferences` -> `200` full preference object.
18) `PATCH /v1/notification-preferences`
- Body: `{ "versionToken":2, "pushOptIn":true, "workoutReminderEnabled":true, "workoutReminderMinutesBefore":30, "quietHoursEnabled":true, "quietHoursStartLocal":"21:00:00", "quietHoursEndLocal":"08:00:00", "timezone":"America/New_York" }`
- Success `200` with token+1.

### 3.8 Recommendation actions
19) `POST /v1/recommendations/{id}/accept` Body `{ "versionToken":2 }` -> `200` status accepted.
20) `POST /v1/recommendations/{id}/ignore` Body `{ "versionToken":2, "ignoredReason":"not_today" }` -> `200` ignored.
21) `POST /v1/recommendations/{id}/dismiss` Body `{ "versionToken":2 }` -> `200` dismissed.

### 3.9 Equipment profile item deletion
22) `DELETE /v1/equipment-profiles/{id}/items/{itemId}`
- Headers: `Idempotency-Key` required
- Body: `{ "equipmentProfileVersionToken": 5 }`
- Success `200`: `{ "deletedItemId":"...", "equipmentProfileVersionToken": 6 }`.

### 3.10 Anonymous sample-plan generation
23) `POST /v1/public/sample-plan/generate`
- Auth: none
- Headers: `Idempotency-Key` optional
- Body: onboarding-like fields (goalType, daysPerWeek, preferredWeekdays, sessionLengthMin, equipmentKeys, experienceLevel)
- Success `200`: transient `samplePlan` only.
- Errors: `429` (10 requests/IP/hour), `422` validation.
- Persistence: no user-owned writes.

### 3.11 Local draft recovery
24) `POST /v1/sync/local-drafts/recover`
- Headers: `Idempotency-Key` required
- Body supports either existing `workoutSessionId` path or creation context path; includes local set logs/check-in snapshots.
- Success `200`: recovered session + duplicates + conflict resolutions.
- Conflicts: `409` with codes `SESSION_ALREADY_COMPLETED`, `PLAN_VERSION_MISMATCH`, `WORKOUT_DAY_MISMATCH`, `STALE_VERSION_TOKEN`.

RLS/ownership expectations for all endpoints: authenticated user may only read/write rows where `row.user_id = auth.uid()`; cross-user attempts return `403`.

---

## 4) Regeneration and state-safety matrix

When regenerating plan or changing equipment profile:
- `not_started`: if scheduled today, keep existing session unless user explicitly chooses replace; future not_started sessions may remap to new plan version.
- `in_progress`: remains attached to original `plan_version_id`; never migrated.
- `completed`: immutable history; never overwritten.
- `partial`: immutable execution history; may spawn follow-up recovery session, original unchanged.
- `skipped`: preserved; recovery may generate new session on new plan version.
- `completed_outside_app`: preserved immutable.
- `abandoned`: preserved; may create replacement session.
- `deleted`: soft-deleted history retained; excluded from future scheduling.
- unsynced local draft before first sync: block destructive ops; require explicit choice `recover | partial | discard`.
- unsynced local draft after partial sync: same block; server returns merge options + detected conflicts.

Hard guarantees:
1. Completed history never overwritten.
2. In-progress sessions remain on original plan version.
3. Unsynced drafts block destructive regeneration/equipment change until user choice recorded.
4. Same-day `not_started` behavior explicit: default keep unless explicit replace.
5. Regeneration always creates new `plan_version` rows.

---

## 5) RLS / FK / trigger / RPC enforcement

Implementation rules by table:
- Client `select`: all user-owned tables where `user_id=auth.uid()`.
- Client `insert/update/delete`: only on user-owned mutable tables listed in §2, with column restrictions via RPC where specified.
- Service-role-only writes: `exercise_catalog`, `equipment_catalog`, and planned structure tables (`plan_version`, `workout_day`, `exercise_instance`, `set_prescription`) except via approved RPC.

RPC-only state transitions:
- workout session status transitions
- plan accept/regenerate/revert
- recommendation accept/ignore/dismiss
- limitation resolve

FK ownership invariants (trigger + constraint checks):
- every child `user_id` must match referenced parent `user_id`.
- session plan/day linkage must match.
- set_log references must belong to same session/day/plan chain.

Supabase policy specifics:
- `USING (user_id = auth.uid())` for select/update/delete on user-owned tables.
- `WITH CHECK (user_id = auth.uid())` for insert/update.
- no direct client policies for service-role-only tables.

Domain errors:
- cross-user access: `403 FORBIDDEN_RESOURCE`.
- same-user wrong-plan/wrong-day chain: `422 FK_OWNERSHIP_VIOLATION`.

Trigger responsibilities:
- set/update `updated_at`.
- increment `version_token` exactly +1 on mutable updates.
- enforce parent-child ownership invariants.
- bump `equipment_profile.version_token` on item create/update/delete.

---

## 6) Acceptance tests mapped to risk

Must-pass tests:
1. Full schema DDL tests for all tables in §2.
2. `version_token` stale write (`409`) and +1 increment on success across all mutable tables.
3. Endpoint contract tests with request/response/error examples for all 24 endpoints.
4. Regeneration safety matrix behavior tests for each session status and unsynced draft scenarios.
5. Local draft recovery conflict and duplicate `clientMutationId` replay tests.
6. RLS tests: cross-user denied (`403`), same-user wrong chain (`422`).
7. FK invariants tests for session/day/plan and set_log references.
8. Notification quiet-hours (9pm–8am local), timezone handling, defaults (`pushOptIn=true`, reminder 30).
9. Equipment taxonomy enforcement (`equipment_key` must exist in `equipment_catalog`) and parent token bump on item mutations.
10. Exercise seed verification: catalog populated with ~80 curated exercises.

---

## 7) Idempotency and concurrency

- Idempotent endpoints require `Idempotency-Key`; server stores hash(user, route, key, request-body-signature) for 24h.
- Replayed identical request returns same status/body.
- Reused key with different body returns `409 IDEMPOTENCY_KEY_REUSED_WITH_DIFFERENT_PAYLOAD`.
- Mutable updates with versioning require matching `versionToken`; stale -> `409 VERSION_CONFLICT` with `currentVersionToken`.

---

## 8) Ownership summary (API camelCase mapping)

All exposed mutable entities include `versionToken` in API: `userProfile`, `goalPlan`, `equipmentProfile`, `userExercisePreference`, `userLimitation`, `notificationPreference`, `planVersion`, `workoutSession`, `postWorkoutCheckIn`, `recommendation`.

---

## 9) Open decisions inherited from PRD

Blocking decisions outside this contract (if unresolved in PRD) must remain flagged:
- exact curated exercise count if product insists on exact number (contract currently uses ~80).
- final allowed recommendation `triggerFactors/inputsUsed` JSON schemas if PRD leaves structure flexible.

---

## 10) Implementation readiness checklist

### A) Contract-complete for engineering estimation
- [ ] All MVP schemas fully defined (columns/types/nullability/defaults/enums/FKs/uniques/checks/indexes/ownership/versioning/API mapping).
- [ ] All endpoints include method/path/auth/headers/request-success-errors/versionToken/idempotency/RLS/transaction notes.
- [ ] Regeneration and local-draft safety matrix fully specified.

### B) Architecture-freeze ready
- [ ] RLS/FK/trigger/RPC enforcement finalized with explicit per-table permissions and domain errors.
- [ ] Acceptance tests mapped to high-risk flows and invariants.
- [ ] PRD open decisions either resolved or explicitly listed as blockers.

### C) Production implementation ready
- [ ] Migration DDL generated exactly from this contract.
- [ ] RPCs implement all state transitions and idempotency semantics.
- [ ] Integration/e2e suite passes all §6 tests in CI.
- [ ] No placeholder language remains.

