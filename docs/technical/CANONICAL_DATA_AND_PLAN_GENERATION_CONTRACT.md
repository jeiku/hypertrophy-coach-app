# CANONICAL DATA AND PLAN GENERATION CONTRACT (FINAL MVP BUILD CONTRACT)

**Status:** Draft contract revision; eligible for engineering estimation and architecture review only. Architecture-freeze readiness is not complete and cannot be marked complete while the contract §9 open decisions inherited from the PRD / PRD open decisions remain unresolved. Production implementation remains blocked until §10.C.1 (PRD §0 pre-build gates passed and signed off) is true and §10.B is fully complete.  
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
- `movement_intent`: `knee_dominant_squat | hip_hinge | horizontal_push | vertical_push | horizontal_pull | vertical_pull | single_leg | core_anti_extension | core_anti_rotation | arm_elbow_flexion | arm_elbow_extension | lateral_deltoid | rear_deltoid | calf_plantarflexion`

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
- `analytics_opt_out boolean not null default false`
- `show_rir_field boolean not null default false`
- `pending_deletion_at timestamptz null`
- `hard_delete_by timestamptz null`
- `onboarding_completed_at timestamptz null`
- `created_at timestamptz not null default now()`
- `updated_at timestamptz not null default now()`
- `version_token integer not null default 1 check (version_token >= 1)`

Constraints/indexes:
- `check ((body_weight is null and body_weight_unit is null) or (body_weight is not null and body_weight_unit is not null))`
- index on `(user_id)` unique already covers lookup.
- check (`pending_deletion_at is null and hard_delete_by is null`) or (`pending_deletion_at is not null and hard_delete_by is not null and hard_delete_by >= pending_deletion_at`).

Ownership/versioning:
- owner is `user_id`.
- any mutable update increments `version_token +1`.

API mapping exposed: `bodyWeightUnit`, `experienceLevel`, `analyticsOptOut`, `showRirField`, `onboardingCompletedAt`, `versionToken`.

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
- `movement_intents movement_intent[] not null`
- `suppression_tags text[] not null default '{}'`
- `primary_muscles text[] not null`
- `secondary_muscles text[] not null default '{}'`
- `movement_pattern text not null`
- `equipment_keys text[] not null`
- `difficulty smallint not null check (difficulty between 1 and 5)`
- `fatigue_cost smallint not null check (fatigue_cost between 1 and 5)`
- `tempo_seconds smallint not null check (tempo_seconds between 1 and 20)`
- `rest_seconds smallint not null check (rest_seconds between 15 and 600)`
- `warmup_overhead smallint not null check (warmup_overhead between 0 and 900)`
- `setup_overhead smallint not null check (setup_overhead between 0 and 900)`
- `cue_setup text not null check (char_length(cue_setup) between 10 and 500)`
- `cue_execution text not null check (char_length(cue_execution) between 10 and 500)`
- `cue_common_mistake text not null check (char_length(cue_common_mistake) between 10 and 500)`
- `cue_safety_note text null check (cue_safety_note is null or char_length(cue_safety_note) between 10 and 500)`
- `experience_level_min experience_level not null`
- `experience_levels_allowed experience_level[] not null default '{beginner,intermediate}'`
- `is_unilateral boolean not null default false`
- `is_active boolean not null default true`
- `metadata_json jsonb not null default '{}'::jsonb`
- `created_at timestamptz not null default now()`
- `updated_at timestamptz not null default now()`

Constraints/indexes:
- GIN indexes on `movement_intents`, `suppression_tags`, `primary_muscles`, `equipment_keys`.
- `suppression_tags` values must come from canonical domain: `chest`,`shoulders`,`back_compound_pull`,`triceps_overhead_loaded`,`biceps`,`triceps`,`loaded_grip_pull`,`loaded_pressing`,`loaded_pulling`,`posterior_chain_loaded`,`spinal_loading`,`rows`,`pull_ups`,`loaded_carries`,`squat_pattern`,`hinge_pattern`,`lunge_pattern`,`leg_extension`,`leg_press`,`calf`,`loaded_standing`,`horizontal_pull`.
- checks: `cardinality(movement_intents)>=1`, `cardinality(primary_muscles)>=1`, `cardinality(equipment_keys)>=1`, `cardinality(experience_levels_allowed)>=1`, and `experience_level_min = any(experience_levels_allowed)`.
- all cue/timing columns required (non-null) for seeded active exercises.

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
- `missed_workout_follow_up_enabled boolean not null default true`
- `recommendation_pending_review_enabled boolean not null default true`
- `streak_milestone_enabled boolean not null default false`
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

### 2.9.A `trust_survey_response`
**Purpose:** Canonical persistence for week-indexed trust survey responses.

Columns:
- `id uuid pk`
- `user_id uuid not null fk auth.users(id) on delete cascade`
- `week_index integer not null check (week_index >= 1)`
- `trust_rating integer not null check (trust_rating between 1 and 5)`
- `submitted_at timestamptz not null`
- `created_at timestamptz not null default now()`
- `updated_at timestamptz not null default now()`
- `version_token integer not null default 1 check (version_token >= 1)`

Constraints/indexes:
- `unique (user_id, week_index)` (no duplicate response for same user/week in MVP; update semantics are explicit overwrite through endpoint upsert path only).
- index `(user_id, submitted_at desc)`.

Versioning/concurrency:
- endpoint/RPC writes require current `version_token`; successful upsert increments token exactly `+1` whether insert-or-update path.

### 2.10 `plan_version`
**Purpose:** Immutable generated plan snapshot metadata + lifecycle.

Columns:
- `id uuid pk`
- `user_id uuid not null fk auth.users(id) on delete cascade`
- `goal_plan_id uuid not null fk goal_plan(id)`
- `equipment_profile_id uuid not null fk equipment_profile(id)`
- `status plan_version_status not null default 'draft'`
- `generation_reason text not null` (`initial|regenerate|recovery|revert_clone` values via check)
- `generation_trigger text null` (`manual_regenerate|equipment_change|missed_workout_recovery|schedule_change|focus_change|other` values via check; optional specificity for generation/regeneration trigger)
- `source_plan_version_id uuid null fk plan_version(id)`
- `validation_warnings_json jsonb not null default '[]'::jsonb`
- `plan_explanation_json jsonb not null default '{}'::jsonb`
- `movement_coverage_summary_json jsonb not null default '{}'::jsonb`
- `accepted_at timestamptz null`
- `archived_at timestamptz null`
- `created_at timestamptz not null default now()`
- `updated_at timestamptz not null default now()`
- `version_token integer not null default 1`

Constraints/indexes:
- one current active-like plan version per user: partial unique `(user_id) where status in ('active','reverted')`
- index `(user_id, status, created_at desc)`
- active plan lookup semantics MUST include statuses `active` and `reverted`.

Behavior:
- workouts tied to a plan version remain tied permanently.
- regeneration creates new row; no mutation of historical planned workout structure.
- a reverted plan (`status='reverted'`) is treated as the user's current active plan and MUST be created from `source_plan_version_id`.

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
- `selection_reason_json jsonb not null default '{}'::jsonb`
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
- `set_type text not null check (set_type in ('working','warmup'))`
- `set_index smallint not null check (set_index >= 1)`
- `set_number smallint generated always as (set_index) stored`
- `target_rep_min smallint not null check (target_rep_min between 1 and 50)`
- `target_rep_max smallint not null check (target_rep_max between target_rep_min and 60)`
- `load_value numeric(6,2) null`
- `load_unit weight_unit null`
- `target_rir_min numeric(3,1) null`
- `target_rir_max numeric(3,1) null`
- `rest_seconds smallint null check (rest_seconds between 15 and 600)`
- `created_at timestamptz not null default now()`

Constraints:
- `unique (exercise_instance_id, set_index)`
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
- `set_log_index smallint not null check (set_log_index >= 1)`
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
- `unique (workout_session_id, exercise_instance_id, set_log_index)`
- index `(workout_session_id, exercise_instance_id)`
- if `set_source='prescribed'` then `set_prescription_id is not null`; if `set_source='added'` then `set_prescription_id is null`.
- prescribed set logs must reference a `set_prescription` whose `exercise_instance_id` equals set log `exercise_instance_id` and belongs to same `workout_session.plan_version_id` chain.
- added set logs are allowed for user edits but cannot be marked prescribed by trigger rewrite/validation.

### 2.16 `post_workout_check_in`
**Purpose:** Post-session recovery/safety check aligned with PRD.

Columns:
- `id uuid pk`
- `user_id uuid not null fk auth.users(id) on delete cascade`
- `workout_session_id uuid not null unique fk workout_session(id) on delete cascade`
- `soreness_level soreness_level not null`
- `soreness_location pain_location[] null`
- `pain_flag boolean not null default false`
- `pain_location pain_location[] not null default '{}'`
- `pain_type pain_type null`
- `pain_notes text null check (char_length(pain_notes) <= 1000)`
- `fatigue_level fatigue_level null`
- `form_breakdown boolean null`
- `created_at timestamptz not null default now()`
- `updated_at timestamptz not null default now()`
- `version_token integer not null default 1`

Constraints:
- `soreness_level` default persisted value is `mild` when client skips soreness input.
- skipped optional fields are represented as `null` (`fatigue_level`, `form_breakdown`, `soreness_location`).
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
- `ignored_reason_text text null check (char_length(ignored_reason_text) <= 500)`
- `created_at timestamptz not null default now()`
- `updated_at timestamptz not null default now()`
- `version_token integer not null default 1`

Constraints/indexes:
- index `(user_id, status, created_at desc)`
- check `ignored_reason is null unless status='ignored'`.

### 2.18 `exercise_substitution` (future-plan and session-level persistence)
**Purpose:** Persist substitution decisions without rewriting completed history.

Columns:
- `id uuid pk default gen_random_uuid()`
- `user_id uuid not null references auth.users(id)`
- `target_context text not null check (target_context in ('future_plan','session_override'))`
- `plan_version_id uuid not null references plan_version(id)`
- `workout_day_id uuid null references workout_day(id)`
- `workout_session_id uuid null references workout_session(id)`
- `original_exercise_instance_id uuid not null references exercise_instance(id)`
- `replacement_exercise_catalog_id uuid not null references exercise_catalog(id)`
- `replacement_exercise_instance_id uuid null references exercise_instance(id)`
- `retired_from_future_view boolean not null default false`
- `session_override_payload jsonb null`
- `reason text null check (reason is null or char_length(reason) between 1 and 500)`
- `created_at timestamptz not null default now()`
- `updated_at timestamptz not null default now()`
- `version_token integer not null default 1`

Constraints/indexes:
- check `((target_context='future_plan' and workout_day_id is not null and workout_session_id is null and replacement_exercise_instance_id is not null and retired_from_future_view=true) or (target_context='session_override' and workout_session_id is not null and workout_day_id is null and replacement_exercise_instance_id is null))`
- index `(user_id, target_context, created_at desc)`
- unique `(user_id, target_context, original_exercise_instance_id, coalesce(workout_session_id,'00000000-0000-0000-0000-000000000000'::uuid))`

Rules:
- Future-plan substitution creates replacement `exercise_instance` + `set_prescription` rows on the same `workout_day_id`; original exercise instance is preserved for history/audit and hidden from future views by substitution-aware query predicate (`retired_from_future_view=true`).
- Session-level substitution stores only an override record and optional session-local payload; it does not mutate plan structure tables.
- Completed workout history and completed set logs are immutable and never rewritten by substitution endpoints/RPCs.

---

## 2.A) Plan Generator Contract (deterministic, implementation-ready target; MVP constraints enforced)

### 2.A.1 Movement-intent slot taxonomy
`movement_intent` enum domain used by generator and catalog:
- `knee_dominant_squat`
- `hip_hinge`
- `horizontal_push`
- `vertical_push`
- `horizontal_pull`
- `vertical_pull`
- `single_leg`
- `core_anti_extension`
- `core_anti_rotation`
- `arm_elbow_flexion`
- `arm_elbow_extension`
- `lateral_deltoid`
- `rear_deltoid`
- `calf_plantarflexion`

Generator slot object schema:
- `slotId text not null`
- `dayIndex smallint not null`
- `slotIndex smallint not null`
- `movementIntent movement_intent not null`
- `isAccessory boolean not null`
- `targetSets smallint not null`
- `targetRepMin smallint not null`
- `targetRepMax smallint not null`
- `targetRirMin numeric(3,1) null`
- `targetRirMax numeric(3,1) null`

PRD slot taxonomy mapping (non-lossy; authoritative for generator/output APIs):

| Internal movementIntent | PRD slot taxonomy | isolationSubTag |
|---|---|---|
| knee_dominant_squat | knee_dominant_squat | null |
| hip_hinge | hip_hinge | null |
| horizontal_push | horizontal_push | null |
| vertical_push | vertical_push | null |
| horizontal_pull | horizontal_pull | null |
| vertical_pull | vertical_pull | null |
| single_leg | lunge_unilateral_leg | null |
| core_anti_extension | isolation | abs |
| core_anti_rotation | isolation | abs |
| arm_elbow_flexion | isolation | biceps |
| arm_elbow_extension | isolation | triceps |
| lateral_deltoid | isolation | side_delt |
| rear_deltoid | isolation | rear_delt |
| calf_plantarflexion | isolation | calf |

Contract rule: every internal `movementIntent` value MUST map to exactly one PRD slot taxonomy row above, and every emitted slot/reason object must include both `movementIntent` and `prdSlotTaxonomy` fields.

### 2.A.2 Split templates by days/week (PRD rules + deterministic implementation expansion of PRD)
- 2 days/week: Full Body A/B (PRD-exact slot count and composition).
  - Each session has exactly 6 slots: `knee_dominant_squat`,`hip_hinge`,`horizontal_push`,`horizontal_pull`,(`vertical_push` OR `vertical_pull`), and exactly 1 isolation slot.
  - Deterministic implementation expansion of PRD: Day A uses `vertical_push`; Day B uses `vertical_pull`.
- 3 days/week: Full Body A/B/C (PRD-exact structure; deterministic weekly rotation).
  - Each session preserves the same 6-slot full-body structure as 2-day (`knee_dominant_squat`,`hip_hinge`,`horizontal_push`,`horizontal_pull`, one vertical pattern, one isolation).
  - Deterministic implementation expansion of PRD: rotate which vertical pattern leads and which knee-dominant pattern variation leads across A/B/C while preserving the same slot counts.
- 4 days/week: Upper/Lower (PRD-exact slot count and composition).
  - Upper day: `horizontal_push`,`vertical_push`,`horizontal_pull`,`vertical_pull`, plus exactly 2 isolation slots.
  - Lower day: `knee_dominant_squat`,`hip_hinge`,`single_leg`, plus exactly 1 isolation slot.
- 5 days/week: Upper/Lower + focus day (PRD-exact).
  - Days 1-4 use the exact 4-day Upper/Lower structure above.
  - Day 5 focus day has exactly 4 slots: exactly 1 compound slot + exactly 3 isolation slots aligned to stated `focus_muscles`.
- 6 days/week, beginner: Full Body A/B/C repeated twice with reduced per-session volume (PRD-exact high-level rule).
  - Deterministic implementation expansion of PRD is allowed only if each session preserves the Full Body slot structure and does not inflate PRD slot counts/volume.
- 6 days/week, intermediate: PPL x2 (PRD-exact high-level rule).
  - Deterministic implementation expansion of PRD for Push/Pull/Legs day-by-day slot assignment is allowed only if it preserves PRD movement coverage and does not inflate PRD slot counts/volume.
Deterministic template resolution keys: `(daysPerWeek, experienceLevel)`.
MVP `goalType` is recorded-only metadata and MUST NOT affect template choice, slot counts, volume targets, exercise selection, progression, or recommendation decisions.
### 2.A.3 Initial weekly volume targets (PRD-exact muscle-by-muscle tables)
Generator SHALL use the PRD v0.6 explicit beginner/intermediate muscle-by-muscle weekly set ranges.
Canonical constants (direct sets/week):
- beginner: chest 4-8, back 6-10, quads 4-8, hamstrings/glutes 4-8, shoulders 4-8, biceps 4-8, triceps 4-8, calves/core 0-6 (optional).
- intermediate: chest 8-14, back 10-16, quads 8-14, hamstrings/glutes 8-14, shoulders 8-14, biceps 6-12, triceps 6-12, calves/core 4-10 (optional).
- no history -> initialize at low end of PRD range.
- generator target -> midpoint of PRD range.
- progression volume increases are gated: minimum 14 consecutive days of adherence consistency and no high-soreness paired signal and no pain flag.
If implementation stores hamstrings and glutes separately, or calves and core separately, that storage is an internal mapping only and MUST preserve the combined PRD range above without increasing required volume.
### 2.A.4 Duration estimation and accessory-cut priority (PRD-exact)
Core duration estimate per session uses exercise-specific values:
`sum_over_exercises(sum_over_sets(target_reps * tempo_seconds + rest_seconds) + warmup_overhead + setup_overhead)`.
No fixed global `avgRepSeconds` constant is permitted as the core estimation rule.
Threshold: run cut policy when estimated duration `> target_session_minutes * 1.15`.
Accessory-cut priority order:
1) Tertiary isolations, specifically calf/forearm-style optional isolation work.
2) Duplicate movement intents within a session.
3) Reduce isolation set counts from 3 to 2 (floor 2).
4) Reduce compound-accessory set counts from 4 to 3 (floor 3 for accessory compounds; primary compounds floor 3 always).
5) Emit validation warning: `Session length target may be too short — consider 10 more minutes or fewer training days.`
Focus-muscle protection (PRD-exact): isolation sets matching the user's focus muscle skip step 1 and are demoted to step 3 only.
Never cut the primary compound for a slot.
### 2.A.5 Hard disqualifiers
Candidate exercise is ineligible if any true:
- inactive in `exercise_catalog`.
- required equipment not in active equipment profile.
- user limitation with severity `hard_block` intersects movement intent, muscle, or equipment.
- active pain-location suppression mapping (§2.B.1.A) or active hard-block limitation contraindicates movement intent/muscle/equipment.

`disliked` is NOT a hard disqualifier; it remains a scoring penalty in §2.A.6 and may be relaxed only via §2.A.7 fallback.
Experience-level eligibility filter uses `experience_levels_allowed` as the canonical machine-testable allowlist (with `experience_level_min` retained for scoring/tie-break context). Exercises where the trainee level is not in `experience_levels_allowed` are initially ineligible; if no alternative eligible candidate exists for the slot, this eligibility gate is relaxable in unfillable-slot fallback before slot-drop (PRD-exact relax-only-when-no-alternative behavior).

### 2.A.6 Selection rule precedence, scoring weights, tie-breakers (PRD-exact rule precedence + tie-breakers)
Scoring components:
- Movement-intent match `+40`
- Same primary muscle `+25`
- Beginner-friendly `+15`
- Focus muscle `+10`
- User preferred `+10`
- Prior successful history `+8`
- Low setup complexity when short session `+5`
- High fatigue cost on back-to-back day `-10`
- Recently skipped 3+ times `-20`
- User disliked `-30`
Rule precedence (PRD-exact):
1. safety/pain
2. equipment availability
3. locked user choice
4. schedule/session length
5. movement coverage
6. experience-level volume cap
7. disliked
8. focus muscles
9. preferred
10. variety
Tie-breakers (PRD-exact, applied only after precedence/scoring leaves a tie):
1. lower setup complexity
2. more beginner-friendly
3. better prior adherence
4. lower fatigue cost
5. stable variety / avoid weekly novelty
### 2.A.7 Cross-slot exclusion, variety enforcement, unfillable-slot fallback (PRD-exact + additional safety guard)
Cross-slot exclusion in same day:
- cannot assign same `exercise_catalog_id` to >1 slot.
Weekly primary-compound exclusion (PRD-exact):
- across a week, the same primary compound cannot appear in more than two slots total.
- if top-ranked exercise is excluded by this weekly primary-compound rule, assign the next-highest-ranked eligible exercise.
Additional safety guard (does not replace PRD weekly primary-compound exclusion):
- cannot assign two hinge-intent heavy spinal-loading lifts if both fatigue_cost >=4 on same day.
Variety rule: across A/B sessions in the same week, at least one exercise per movement intent must differ. If A/B days are identical for any movement intent, rerun selection for that intent using the second-highest-ranked exercise for the most-overused intent.
Unfillable fallback order when max score `< 0` after filters/exclusions:
1) ignore disliked penalty.
2) ignore preferred boost.
3) if still `< 0` or pool empty: drop slot and emit validation warning.
Nearest-intent substitution and slot rewrite are NOT allowed in this fallback path unless PRD explicitly permits for that slot class.
### 2.A.8 Calibration checkpoint (pre-launch, PRD-exact)
Pre-launch generator calibration gate:
- run generator on 20 representative profiles.
- coach review outputs against PRD acceptance rubric.
- if pass rate `<85%`, tune scoring weights and rerun until threshold met.
This is a release-readiness calibration checkpoint; it is not replaced by runtime after-two-sessions auto-adjustment.

### 2.A.9 Substitution algorithm and load handling (PRD-exact MVP)
Substitution trigger: unavailable equipment, pain suppression, user skip-request per exercise.
Substitution scoring (PRD-exact):
- Preserve movement intent: `+40`
- Preserve primary muscle: `+25`
- Similar difficulty/fatigue cost within one tier: `+10`
- Prior successful history: `+8`
- Apply standard disliked/repeatedly-skipped penalties.
Algorithm:
1) find candidate pool with same movement_intent.
2) score using the substitution scoring above and applicable penalties.
3) apply §2.A.7 fallback only when required.
4) choose deterministic top candidate by §2.A.6 tie-breaker order.
5) preserve planned set count and rep range when possible.
Load handling:
- if user has history with substitute and last successful set met minimum target reps, suggest that last successful load.
- if no history, leave load blank and attach conservative-start prompt.
- bodyweight substitutions: prescribe bodyweight/assisted/variation level.
- machine/non-comparable load scales: leave load blank.
MVP forbids carrying load unchanged by default and forbids bilateral→unilateral conversion heuristics unless introduced as explicitly labeled post-MVP/open decision.
### 2.A.10 Generator outputs
Persistence contract: generator explanations/reasons are persisted and never API-transient-only. `plan_version.plan_explanation_json`, `plan_version.movement_coverage_summary_json`, and `exercise_instance.selection_reason_json` are required storage for generated outputs and for any future-plan substitution replacement exercise instances.

Plan-level explanation output schema (`plan_version.plan_explanation_json` companion API field `planExplanation`):
- `templateChosen`
- `equipmentConstraintsApplied[]`
- `limitationSuppressions[]`
- `durationAdjustments[]`
- `substitutionsApplied[]`
Validation warning object schema:
- `code text`
- `severity enum(info|warning|blocking)`
- `message text`
- `slotId text null`
Movement coverage summary output (`plan_version.movement_coverage_summary_json` companion API field `movementCoverageSummary`):
- per movement intent: `targetSets`, `assignedSets`, `coverageStatus enum(full|partial|missing)`.

Per-exercise selection reason output (`exercise_instance.selection_reason_json` companion API field `selectionReason`):
- required keys for every generated exercise instance: `movementIntent`, `prdSlotTaxonomy`, `scoringFactors`, `constraintsApplied`, `fallbackApplied`, `fallbackStatus`, `userFacingExplanation`.
- `scoringFactors` includes weighted components and tie-break resolution details used for that exercise choice.
- `constraintsApplied` includes pain/limitation/equipment filters actually applied (or empty list when none).
- `fallbackStatus` explicitly states whether normal selection, fallback step #, or unfilled-slot pathway was used.

## 2.B) Progression and Recommendation Rules (PRD-exact precedence/threshold contract)

### 2.B.1 Recommendation precedence (strict order)
`pain/safety -> reduce -> hold -> increase -> default hold/add reps`.


### 2.B.1.A Pain-location suppression mapping (machine-testable)

| painLocation | suppressedMuscles | suppressedMovements | substitutionRequiredForEligibleExercises | progressionSuppressed |
|---|---|---|---|---|
| shoulder | [chest,shoulders,back_compound_pull,triceps_overhead_loaded] | [horizontal_push,vertical_push] | true | true |
| elbow | [biceps,triceps,loaded_grip_pull] | [arm_elbow_flexion,arm_elbow_extension] | true | true |
| wrist | [loaded_pressing,loaded_pulling] | [] | true | true |
| lower_back | [posterior_chain_loaded,spinal_loading,loaded_standing] | [hip_hinge] | true | true |
| upper_back | [rows,pull_ups,loaded_carries] | [] | true | true |
| hip | [squat_pattern,hinge_pattern,lunge_pattern] | [knee_dominant_squat,hip_hinge,single_leg] | true | true |
| knee | [squat_pattern,lunge_pattern,leg_extension,leg_press] | [knee_dominant_squat,single_leg] | true | true |
| ankle | [squat_pattern,lunge_pattern,calf,loaded_standing] | [calf_plantarflexion] | true | true |
| other | [] | [] | false | false |

Application rules: mapping is applied to exercise eligibility filtering, substitution candidate retrieval, and progression recommendation suppression. Tag-based suppression is authoritative when PRD specifies tags/muscle groups; movement-based suppression is used only where PRD suppresses the whole movement class. `other` does not auto-suppress but emits review prompt.

### 2.B.2 Load increase threshold
All conditions required:
- all planned working sets completed.
- top-rep condition true: either all sets at top rep OR all sets within 1 rep of top and prior comparable session also met this condition.
- no pain flag.
- fatigue not high.
- no form breakdown.

RIR-enabled behavior:
- 1–3 RIR supports normal load increase when other increase conditions pass.
- 0 RIR paired with high fatigue forces hold recommendation.
- 4+ RIR can still support load increase; explanation must note effort was easier than target.

### 2.B.3 Hold threshold
Suggest hold when any are true (and reduce / safety did not fire):
- user did not reach top reps on most sets.
- user recently increased load and reps dropped but stayed in range.
- fatigue high.
- logging incomplete.
- RIR missing and readiness unclear.

### 2.B.4 Reduce threshold
Suggest reduce when any are true (and safety did not fire):
- reps below minimum target on 2+ working sets.
- total reps drop approximately 20% at the same load across comparable sessions.
- user reports form breakdown.

Pain is handled by precedence step 1 and is not duplicated in reduce-threshold evaluation.

### 2.B.5 Deload threshold and application
Suggest deload when 3 or more of these 5 signals appear within a trailing 7-day window:
1. performance decreases across 2 comparable exposures.
2. fatigue high in 2 or more recent sessions.
3. soreness high in 2 or more recent sessions.
4. 2 or more missed workouts after prior consistency.
5. pain or joint discomfort reported in this window.

Deload is recommendation-only, never auto-applied, and requires explicit user confirmation.

### 2.B.5 Structured reason object and Why? requirements (PRD)
Every recommendation object MUST include:
- `recommendationType`
- `triggerFactors`
- `userFacingReason`
- `educationalContext`
UI contract: always expose `userFacingReason` first; `educationalContext` is shown on deeper tap.
Additional machine-readable fields are additive only and MUST NOT replace these four required fields.

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
- Success `201`: returns draft `planVersion`, nested `workoutDays/exercises/sets`, `planExplanation`, `movementCoverageSummary`, per-exercise `selectionReason`, `validationWarnings`, `versionToken`.
- Idempotency: same user + key + semantically equal body returns same response.
- Transaction: create `plan_version` + children atomically.

2) `POST /v1/plans/{planVersionId}/accept`
- Headers: `Idempotency-Key` required
- Body: `{ "versionToken": 3 }`
- Success `200`: activated planVersion with incremented token, including `planExplanation`, `movementCoverageSummary`, and nested per-exercise `selectionReason`.
- Transaction: archive prior active plan (if any), set current active.

3) `POST /v1/plans/{planVersionId}/regenerate`
- Headers: `Idempotency-Key` required
- Body: `{ "versionToken": 4, "reason":"regenerate", "generationTrigger":"equipment_change" }`
- Success `201`: new draft planVersion (new id), prior plan untouched; response includes `planExplanation`, `movementCoverageSummary`, and nested per-exercise `selectionReason`.

4) `POST /v1/plans/{planVersionId}/revert`
- Headers: `Idempotency-Key` required
- Body: `{ "versionToken": 6 }`
- Success `201`: new active clone with status `reverted`; response includes `planExplanation`, `movementCoverageSummary`, and nested per-exercise `selectionReason`.

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
- Body: `{ "versionToken":2, "pushOptIn":true, "workoutReminderEnabled":true, "workoutReminderMinutesBefore":30, "missedWorkoutFollowUpEnabled":true, "recommendationPendingReviewEnabled":true, "streakMilestoneEnabled":false, "quietHoursEnabled":true, "quietHoursStartLocal":"21:00:00", "quietHoursEndLocal":"08:00:00", "timezone":"America/New_York" }`
- Success `200` with token+1.

### 3.8 Recommendation actions
MVP bounded schema requirement for recommendation explainability fields:
- `triggerFactors` object allowlist keys: `progression_signal`, `pain_safety_signal`, `deload_signal`, `hold_signal`, `reduce_signal`, `substitution_signal`, `missed_workout_recovery_signal`.
- Each key maps to `{ active:boolean, source:string, threshold?:string, windowDays?:number }`; additional keys forbidden in MVP v1.
- `inputsUsed` allowlist arrays: `recent_performance`, `post_workout_checkins`, `missed_workout_count`, `equipment_availability`, `limitation_flags`, `goal_schedule_context`.
- Unknown keys must fail validation with `422 INVALID_RECOMMENDATION_SCHEMA`; future extensibility via explicit versioned schema upgrade only.
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
- Success `200`: transient `samplePlan` response plus local-persistence contract metadata `{ localSamplePlanTtlDays:7, localSamplePlanKeyScope:"device_install" }`.
- Errors: `429` (10 requests/IP/hour), `422` validation.
- Persistence: no user-owned backend writes and no email/user-owned record creation before account creation; client saves sample plan locally for 7 days keyed to device/local install only.
- Reopen behavior: reopening within 7 days MUST show restore prompt text-equivalent to “Save your plan?” and allow restore/discard.
- Expiry behavior: after 7 days from local save timestamp, local sample plan MUST be discarded and restore prompt MUST NOT be shown.
- Analytics: only allowlisted `onboarding_abandoned_with_plan` may emit pre-account-creation and payload must exclude prohibited PII/sensitive fields from §8.1.

### 3.11 Local draft recovery
24) `POST /v1/sync/local-drafts/recover`
- Headers: `Idempotency-Key` required
- Body supports either existing `workoutSessionId` path or creation context path; includes local set logs/check-in snapshots.
- Success `200`: recovered session + duplicates + conflict resolutions.
- Conflicts: `409` with codes `SESSION_ALREADY_COMPLETED`, `PLAN_VERSION_MISMATCH`, `WORKOUT_DAY_MISMATCH`, `STALE_VERSION_TOKEN`.

RLS/ownership expectations for all endpoints: authenticated user may only read/write rows where `row.user_id = auth.uid()`; cross-user attempts return `403 FORBIDDEN_RESOURCE`; same-user invalid FK chain returns `422 FK_OWNERSHIP_VIOLATION`.
The endpoint schema expansions in §3.12–§3.14 are normative subsections of §3 and define strict implementation-ready schemas for every endpoint listed in §3.1–§3.14, including method/path/auth/headers/request/response/errors/idempotency/versionToken/RLS/transaction notes.



### 3.12 Trust survey
25) `POST /v1/trust-survey`
- Headers: `Idempotency-Key` required
- Body: `{ "versionToken":7, "weekIndex":4, "trustRating":5, "submittedAt":"2026-05-10T12:00:00Z" }`
- Success `200`: `{ "data": { "weekIndex":4, "trustRating":5, "submittedAt":"...", "versionToken":8 } }`
- Behavior: upsert for `(user_id, week_index)` via `rpc_upsert_trust_survey_response`; request `versionToken` must match current `trust_survey_response.version_token` for update path (or current `user_profile.versionToken` on first insert when no row exists). Success increments `trust_survey_response.version_token` by exactly +1 and returns that token.

### 3.13 Account deletion request
26) `POST /v1/account/deletion-request`
- Headers: `Idempotency-Key` required
- Body: `{ "versionToken":8, "requestedAt":"2026-05-10T12:00:00Z" }`
- Success `200`: `{ "data": { "pendingDeletionAt":"...", "hardDeleteBy":"...", "versionToken":9 } }`
- Behavior: sets `user_profile.pending_deletion_at` immediately, computes `hard_delete_by` from policy-configured retention window, and blocks app data access immediately for that user except deletion-status read + auth/session termination flows.
- Legal note: final retention/anonymization text remains PRD/legal blocker in §9.

### 3.14 Endpoint schema expansion
 (authoritative supplement to §3.1–§3.11)
For each endpoint in §3, the following are mandatory and not optional: method/path, auth, required headers, strict request schema, strict success schema, error codes, idempotency behavior, versionToken behavior, transaction notes, RLS ownership checks.

#### 3.14.1 Shared request/response envelopes
- Request headers for authenticated mutable endpoints: `Authorization`, `Content-Type: application/json`, `Idempotency-Key`.
- Success envelope: `{ "data": <resource>, "meta": { "requestId": "uuid", "idempotencyReplay": false } }`.
- Error envelope: `{ "error": { "code": "...", "message": "...", "details": {...} } }`.

#### 3.14.2 Full schema obligations by endpoint group
- Plans endpoints (§3.1): body must include all required IDs, optional fields explicitly nullable; response must include planVersion with workoutDays[] -> exercises[] -> setPrescriptions[] and `planExplanation`, `validationWarnings`, `movementCoverageSummary`.
- Workout session endpoints (§3.2): response includes `status`, `versionToken`, `startedAt`, `endedAt`, `scheduledForDate`.
- Set logs endpoint (§3.3): entries require `clientMutationId`, `exerciseInstanceId`, `setSource`, `setLogIndex`, reps/load/RIR fields; `setPrescriptionId` required only for prescribed sets.
- Recovery endpoint (§3.4): includes action enum, optional targetDate, and explicit `versionToken`; returns both updated session and optional created planVersion summary.
- Limitations endpoints (§3.5): enforce typed arrays for affected muscles/equipment, versionToken on patch/resolve.
- Post-workout check-in (§3.6): full schema required with pain conditional validation.
- Notification endpoints (§3.7): patch requires versionToken and timezone string.
- Recommendation actions (§3.8): include versionToken; ignore requires `ignoredReason` enum.
- Equipment item deletion (§3.9): requires `equipmentProfileVersionToken`; returns incremented parent token.
- Public sample plan (§3.10): no auth, no user-owned writes, rate limits.
- Local draft recovery (§3.11): strict local draft schema in §3.15.

#### 3.14.3 Endpoint-level idempotency and versionToken behavior
- `POST /v1/plans/generate`: idempotent by key+body; no versionToken required.
- `POST /v1/plans/{id}/accept|regenerate|revert`: require versionToken; stale -> `409 VERSION_CONFLICT`.
- `POST /v1/workout-sessions/start`: idempotent create by key+day+date.
- `POST /v1/workout-sessions/{id}/resume|finish|mark-partial|skip|complete-outside-app`: require versionToken; duplicate key replay returns same transition result.
- `POST /v1/workout-sessions/{id}/set-logs`: dedupe by `(user_id, workout_session_id, client_mutation_id)`; replay returns prior row IDs.
- `PUT /v1/workout-sessions/{id}/post-workout-check-in`: idempotent upsert semantics by session.
- `PATCH /v1/notification-preferences`, limitation patch/resolve, recommendation actions: optimistic concurrency required.

### 3.15 Local draft recovery schema and conflict matrix
 (authoritative local-draft payload schema and conflict matrix)
Local draft payload schema:
- `localDraftId string not null`
- `planVersionId uuid not null`
- `workoutDayId uuid not null`
- `workoutSessionId uuid null`
- `startedAt timestamptz not null`
- `completedSetLogs[]` each with `clientMutationId`, `exerciseInstanceId`, `setSource`, `setLogIndex`, `setPrescriptionId?`, `performedReps?`, `performedLoad?`, `loadUnit?`, `rir?`, `setState`.
- `skippedExercises[]` with `exerciseInstanceId`, `reason?`
- `skippedSets[]` with `exerciseInstanceId`, `setLogIndex`, `reason?`
- `notes string null`
- `lastSavedAt timestamptz not null`
- `syncStatus enum('not_synced','partially_synced','synced','conflicted')`
- `sessionVersionToken integer null`

Recovery path A (existing workoutSessionId):
1) validate ownership and session open state.
2) validate `planVersionId/workoutDayId` chain equals existing session.
3) apply idempotent set log upserts by `clientMutationId`.
4) return `duplicateClientMutationIds[]` and accepted IDs.

Recovery path B (creation context before first sync):
1) `scheduledForDate` is required when `workoutSessionId` is null; server interprets date in `user_profile.timezone` and persists canonical UTC boundaries accordingly.
2) if `workoutSessionId` is provided and `scheduledForDate` is also provided, `scheduledForDate` MUST equal existing `workout_session.scheduled_for_date`; mismatch returns `409 LOCAL_DRAFT_SCHEDULED_DATE_CONFLICT` with deterministic payload `{ expectedScheduledForDate, providedScheduledForDate }`.
3) create session from `planVersionId+workoutDayId+scheduledForDate` in transaction using uniqueness guard `(user_id, workout_day_id, scheduled_for_date)` to prevent duplicate session creation during recovery replay.
2) insert recovered set logs/check-in snapshot.
3) return created `workoutSessionId` and server `versionToken`.

Conflict handling codes:
- `SESSION_ALREADY_COMPLETED`: do not append logs; return latest server session snapshot.
- `PLAN_VERSION_MISMATCH`: reject with expected/current IDs.
- `WORKOUT_DAY_MISMATCH`: reject with expected/current IDs.
- `STALE_VERSION_TOKEN`: reject with `currentVersionToken`.
- `DUPLICATE_CLIENT_MUTATION_ID`: non-fatal replay list in success payload.
- `PARTIAL_SYNC_CONFLICT`: return merged summary + unresolved item list.
- `LOCAL_DRAFT_SCHEDULED_DATE_CONFLICT`: deterministic mismatch error for creation-context payload vs existing session date.

### 3.16 Missing MVP endpoint contracts
 (authoritative supplement to §3)

#### 3.16.1 User profile / onboarding
- `PUT /v1/user-profile`
  - Auth: required; owner row only.
  - Headers: `Authorization`, `Content-Type: application/json`, `Idempotency-Key`.
  - Request: `{ versionToken?: number, birthDate?: date|null, sex?: enum|null, heightCm?: number|null, bodyWeight?: number|null, bodyWeightUnit?: kg|lb|null, experienceLevel?: beginner|intermediate|null, timezone: string, onboardingCompletedAt?: timestamptz|null, analyticsOptOut?: boolean, showRirField?: boolean }`
  - Success: `{ data: { ...userProfile, versionToken:number } }`
  - Errors: `400 INVALID_JSON`, `401 UNAUTHORIZED`, `403 FORBIDDEN_RESOURCE`, `409 VERSION_CONFLICT`, `422 VALIDATION_ERROR|UNDERAGE_NOT_ALLOWED|ONBOARDING_INCOMPLETE_REQUIREMENTS`, `429 RATE_LIMITED`.
  - Onboarding completion rule: setting `onboardingCompletedAt` requires non-null `bodyWeight`, `bodyWeightUnit`, `heightCm`, (`birthDate` proving age >=18 at save time), `experienceLevel`, active `goalPlan` (`goalType`,`daysPerWeek`,`preferredWeekdays`,`sessionLengthMin`), and active equipment profile with >=1 equipment item.
  - Age gate: server rejects under-18 users at onboarding completion and account-save with `422 UNDERAGE_NOT_ALLOWED`; `birthDate` may remain null only while onboarding draft is incomplete. Sex remains optional in MVP.
  - Idempotency/versioning: requires `Idempotency-Key`; create path may omit versionToken, update path requires exact current token.
  - Transaction notes: single-row upsert on `user_profile`; if onboarding completion requested, server validates goal/equipment preconditions in same transaction snapshot.
  - RLS: only `user_profile.user_id=auth.uid()` row.

#### 3.16.2 Goal plan CRUD
- `POST /v1/goal-plans`, `PATCH /v1/goal-plans/{id}` with strict fields from §2.2.
- `POST /v1/goal-plans`
  - Auth/headers: auth required + `Idempotency-Key`.
  - Request: `{ goalType, daysPerWeek, preferredWeekdays[], sessionLengthMin, focusMuscles?:string[], isActive?:boolean }`.
  - Success `201`: `{ data:{ id, userId, goalType, daysPerWeek, preferredWeekdays, sessionLengthMin, focusMuscles, isActive, versionToken:1 } }`.
  - Errors: `409 ACTIVE_GOAL_ALREADY_EXISTS` (if `isActive=true` and no archive flag path), `422 INVALID_WEEKDAY_CARDINALITY|INVALID_RANGE`, `403 FORBIDDEN_RESOURCE`.
  - Idempotency: key+body replay returns same created row.
- `PATCH /v1/goal-plans/{id}`
  - Request: `{ versionToken:number, goalType?, daysPerWeek?, preferredWeekdays?, sessionLengthMin?, focusMuscles?, isActive? }`.
  - Success `200`: updated row + incremented `versionToken`.
  - Errors: `404 GOAL_PLAN_NOT_FOUND`, `409 VERSION_CONFLICT`, `422 INVALID_WEEKDAY_CARDINALITY|INVALID_RANGE`.
  - Idempotency/versionToken: `Idempotency-Key` required; stale token rejected.
- Active-goal uniqueness enforced transactionally; update/archive prior active in same transaction.
- Transaction/RLS: `user_id=auth.uid()` enforced; toggling active state archives previous active goal in same transaction.

#### 3.16.3 Equipment profile and items
- `POST /v1/equipment-profiles`, `PATCH /v1/equipment-profiles/{id}`, `DELETE /v1/equipment-profiles/{id}` (soft-delete/archive only if referenced).
- `POST /v1/equipment-profiles/{id}/items` request `{ equipmentProfileVersionToken:number, equipmentKey:string }`.
- Delete item endpoint in §3.9 remains authoritative; add endpoint follows same parent token bump semantics.
- `POST /v1/equipment-profiles`: auth + `Idempotency-Key`; success `201` with `versionToken:1`; active uniqueness transaction identical to goal-plan rule.
- `PATCH /v1/equipment-profiles/{id}`: requires `{ versionToken:number, name?:string, isActive?:boolean }`; success `200` token+1; errors `409 VERSION_CONFLICT`, `422 INVALID_NAME`.
- `DELETE /v1/equipment-profiles/{id}`: requires `{ versionToken:number }`; success `200` archived profile token+1; returns `409 PROFILE_IN_USE` if internal hard-delete requested while referenced.
- `POST /v1/equipment-profiles/{id}/items`
  - Headers: `Idempotency-Key` required.
  - Request: `{ equipmentProfileVersionToken:number, equipmentKey:string }`.
  - Success `201`: `{ data:{ itemId, equipmentProfileId, equipmentKey, equipmentProfileVersionToken:number } }` (parent token increments exactly +1).
  - Errors: `404 EQUIPMENT_PROFILE_NOT_FOUND`, `409 VERSION_CONFLICT|DUPLICATE_EQUIPMENT_ITEM`, `422 INVALID_EQUIPMENT_KEY|FK_OWNERSHIP_VIOLATION`.
  - Transaction/RLS: insert item + parent token bump atomic; parent `user_id` ownership validated.

#### 3.16.4 Exercise catalog read APIs
- `GET /v1/exercises?movementIntent=&equipmentKey=&limit=&cursor=` and `GET /v1/exercises/{id}`.
- Read-only through API proxy to seed tables; direct client table access is allowed only read (`is_active=true`) under RLS.
- `GET /v1/exercises`
  - Auth: required in app flow (public unauth read is non-MVP/internal only).
  - Request query: `movementIntent?`, `equipmentKey?`, `limit(1..100, default 25)`, `cursor?`.
  - Success `200`: `{ data:{ items:[...exerciseCatalogProjection], nextCursor?:string|null } }`.
  - Errors: `422 INVALID_QUERY_PARAM`.
- `GET /v1/exercises/{id}`
  - Success `200`: single active exercise projection; `404 EXERCISE_NOT_FOUND` when missing/inactive.

#### 3.16.5 Exercise preference CRUD
- `POST /v1/exercise-preferences`, `PATCH /v1/exercise-preferences/{id}`, `DELETE /v1/exercise-preferences/{id}` using §2.7 schema and versionToken for patch/delete.
- `POST /v1/exercise-preferences`: headers include `Idempotency-Key`; request `{ exerciseCatalogId, preferenceType, notes? }`; success `201` versionToken=1; errors `409 DUPLICATE_PREFERENCE`, `422 INVALID_PREFERENCE`.
- `PATCH /v1/exercise-preferences/{id}`: request `{ versionToken, preferenceType?, notes? }`; success `200` token+1; errors `409 VERSION_CONFLICT`, `404 NOT_FOUND`.
- `DELETE /v1/exercise-preferences/{id}`: request `{ versionToken }`; success `200` tombstone ack; idempotent replay returns same ack; errors `409 VERSION_CONFLICT`.
- RLS/transaction: each mutation scoped to `user_id=auth.uid()`; no cross-user exercise preference reads/writes.

#### 3.16.6 Substitution candidates + apply
- `POST /v1/substitutions/candidates`
  - Request: `{ planVersionId:uuid, workoutDayId:uuid, exerciseInstanceId:uuid, context:'future_plan'|'session_override', painLocation?:pain_location[], unavailableEquipmentKeys?:string[], versionToken:number }`
  - Success: `{ data:{ candidates:[{ exerciseCatalogId:uuid, score:number, whyThisSubstitution:{ movementIntentMatch:boolean, equipmentCompatible:boolean, painSuppressionApplied:boolean, fatigueComparison:string, historyLoadHint?:string|null } }], suppressedByPainMapping:boolean } }`
- `POST /v1/substitutions/apply`
  - Request: `{ targetContext:'future_plan'|'session_override', planVersionId:uuid, workoutDayId?:uuid, workoutSessionId?:uuid, exerciseInstanceId:uuid, replacementExerciseCatalogId:uuid, versionToken:number }`
  - Success: `{ data:{ substitutionId:uuid, mutationType:'replace_future_exercise_instance'|'session_level_override', newExerciseInstanceId?:uuid, preservedOriginalExerciseInstanceId:uuid, newExerciseSelectionReason?:object } }`
  - Mutation rule: future-plan applies by creating replacement `exercise_instance` row (with required `selection_reason_json`) and retiring original from future view; session override writes through `exercise_substitution.session_override_payload` only (no separate `session_exercise_override` entity in MVP). Completed workout history and completed set logs are never rewritten.
  - Headers/auth: both substitution endpoints require auth + `Idempotency-Key`.
  - Errors (both): `404 TARGET_NOT_FOUND`, `409 VERSION_CONFLICT|INVALID_STATE_TRANSITION`, `422 INVALID_SUBSTITUTION_CONTEXT|FK_OWNERSHIP_VIOLATION`.
  - VersionToken: required and validated against plan/session context selected by `context/targetContext`.
  - Transaction notes: candidate computation is read-only snapshot; apply is atomic write transaction touching only allowed target context rows.

#### 3.16.7 Recommendation action ignoredReason other text
- `POST /v1/recommendations/{id}/ignore` request extends to `{ versionToken:number, ignoredReason:enum, ignoredReasonText?:string|null }`; `ignoredReasonText` required when `ignoredReason='other'`.
- Additional validation: if `ignoredReason!='other'`, `ignoredReasonText` must be null/omitted else `422 IGNORED_REASON_TEXT_NOT_ALLOWED`.
- Idempotency/versioning: `Idempotency-Key` required; duplicate request replays prior state transition response, stale version token returns `409 VERSION_CONFLICT`.

#### 3.16.8 Account deletion status/read endpoint
- `GET /v1/account/deletion-status`
  - Purpose: allow app to read deletion pending status after deletion-request path blocks general data APIs.
  - Auth: required.
  - Success `200`: `{ data:{ pendingDeletionAt:timestamptz|null, hardDeleteBy:timestamptz|null, deletionState:'none'|'pending'|'scheduled_hard_delete' } }`.
  - Errors: `401 UNAUTHORIZED` only; endpoint bypasses pending-deletion write/read block by explicit policy exception.
  - VersionToken/idempotency: read-only endpoint; no token/header requirement.

#### 3.16.9 Transaction/RPC notes for endpoint groups
- Workout-session status transitions are RPC-only (`rpc_transition_workout_session`).
- Recommendation accept/ignore/dismiss are RPC-only (`rpc_apply_recommendation_action`).
- Set logs are append/idempotent insert only via RPC (`rpc_append_set_logs`); no client update/delete contract.


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

### 4.1 Explicit transition tables (authoritative state-machine supplement)

#### 4.1.1 PlanVersion transitions
| Source | Target | Trigger | versionToken | Idempotency | Invalid transition |
|---|---|---|---|---|---|
| draft | active | `POST /v1/plans/{id}/accept` | required | required (`Idempotency-Key`) | `422 INVALID_STATE_TRANSITION` |
| active | archived | implicit inside accept/regenerate/revert transaction | required on initiating endpoint | inherited from initiating endpoint | `422 INVALID_STATE_TRANSITION` |
| active | reverted (new clone row) | `POST /v1/plans/{id}/revert` | required | required | `422 INVALID_STATE_TRANSITION` |
| active/draft | draft (new row) | `POST /v1/plans/{id}/regenerate` | required | required | `422 INVALID_STATE_TRANSITION` |

#### 4.1.2 WorkoutSession transitions
| Source | Target | Trigger endpoint/RPC | versionToken | Idempotency | Invalid transition |
|---|---|---|---|---|---|
| not_started | in_progress | `POST /v1/workout-sessions/start` or `/resume` via `rpc_transition_workout_session` | start: none, resume: required | required | `422 INVALID_STATE_TRANSITION` |
| in_progress | completed | `POST /v1/workout-sessions/{id}/finish` | required | required | `422 INVALID_STATE_TRANSITION` |
| in_progress | partial | `POST /v1/workout-sessions/{id}/mark-partial` | required | required | `422 INVALID_STATE_TRANSITION` |
| not_started/in_progress | skipped | `POST /v1/workout-sessions/{id}/skip` | required | required | `422 INVALID_STATE_TRANSITION` |
| not_started/in_progress | completed_outside_app | `POST /v1/workout-sessions/{id}/complete-outside-app` | required | required | `422 INVALID_STATE_TRANSITION` |
| in_progress | abandoned | internal-only non-MVP automation/timeout path (no public endpoint) | internal token check | internal idempotency | `422 INVALID_STATE_TRANSITION` |
| any non-deleted | deleted | internal-only non-MVP retention/admin path (no public endpoint) | internal token check | internal idempotency | `422 INVALID_STATE_TRANSITION` |

#### 4.1.3 Recommendation transitions
| Source | Target | Trigger | versionToken | Idempotency | Invalid transition |
|---|---|---|---|---|---|
| pending | accepted | `POST /v1/recommendations/{id}/accept` via `rpc_apply_recommendation_action` | required | required | `422 INVALID_STATE_TRANSITION` |
| pending | ignored | `POST /v1/recommendations/{id}/ignore` | required | required | `422 INVALID_STATE_TRANSITION` |
| pending | dismissed | `POST /v1/recommendations/{id}/dismiss` | required | required | `422 INVALID_STATE_TRANSITION` |
| pending | expired | internal scheduler only (no public endpoint) | n/a | n/a | `422 INVALID_STATE_TRANSITION` |

#### 4.1.4 Sync/local draft transitions
| Source | Target | Trigger | versionToken | Idempotency | Invalid transition |
|---|---|---|---|---|---|
| not_synced | partially_synced | `POST /v1/sync/local-drafts/recover` with partial apply | session token required if session exists | required | `409 PARTIAL_SYNC_CONFLICT` |
| not_synced/partially_synced | synced | `POST /v1/sync/local-drafts/recover` full apply | required when session exists | required | `409 STALE_VERSION_TOKEN` |
| not_synced/partially_synced | conflicted | server conflict detection during recover | required when session exists | required | `409 PLAN_VERSION_MISMATCH|WORKOUT_DAY_MISMATCH|LOCAL_DRAFT_SCHEDULED_DATE_CONFLICT` |
| conflicted | synced | client retries recover with corrected payload | required | required | `409 PARTIAL_SYNC_CONFLICT` |

---

## 5) RLS / FK / trigger / RPC enforcement

Implementation rules by table:
- Client `select`: all user-owned tables where `user_id=auth.uid()`.
- Client `insert/update/delete`: only on user-owned mutable tables listed in §2, with column restrictions via RPC where specified.
- No direct client write policies for service-role-owned/planned-structure tables: `exercise_catalog`, `equipment_catalog`, and planned structure tables (`plan_version`, `workout_day`, `exercise_instance`, `set_prescription`) except via approved RPC/service role paths.
- Owner-scoped client reads are allowed where explicitly listed in §5.1 (including plan/workout structure rows), and seed catalog reads are allowed only for active rows (`is_active=true`).

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
### 5.1 RLS/FK/trigger/RPC enforcement expansion
 RLS/FK/trigger/RPC enforcement expansion (per-table matrix required)
Per-table RLS matrix (explicit):
- `user_profile`: client select ✅ insert ✅ update ✅ delete ❌; RPC-only writes ❌; service-role-only writes ❌; policy `USING (user_id=auth.uid()) WITH CHECK (user_id=auth.uid())`.
- `goal_plan`: select ✅ insert ✅ update ✅ delete ✅(soft-delete); RPC-only writes ❌; service-only ❌; same policy.
- `equipment_profile`: select ✅ insert ✅ update ✅ delete ✅(archive); RPC-only writes ❌; service-only ❌; same policy.
- `equipment_profile_item`: select ✅ insert ❌ update ❌ delete ❌; RPC-only writes ✅ (`rpc_add/remove_equipment_item`); service-only ❌; read policy `USING (user_id=auth.uid())`.
- `exercise_catalog`: select ✅(only `is_active=true`) insert ❌ update ❌ delete ❌; RPC-only writes ❌; service-only ✅.
- `user_exercise_preference`: client CRUD ✅ via ownership policy.
- `user_limitation`: client select ✅ insert ✅ update ✅ delete ❌(resolve instead); resolve is RPC-only ✅.
- `notification_preference`: select ✅ insert ✅ update ✅ delete ❌.
- `trust_survey_response`: select ✅ insert ❌ update ❌ delete ❌; RPC-only writes ✅ (`rpc_upsert_trust_survey_response`).
- account deletion state source of truth: `user_profile.pending_deletion_at` + `user_profile.hard_delete_by`; once pending is set, all user-owned table policies additionally require `pending_deletion_at is null` for app reads/writes (except dedicated deletion-status/account endpoints).
- `plan_version`: select ✅ insert ❌ update ❌ delete ❌; RPC-only writes ✅ (`rpc_generate/accept/regenerate/revert_plan`); service-only direct writes ✅.
- `workout_day`: select ✅ insert ❌ update ❌ delete ❌; RPC-only/service-only writes ✅.
- `exercise_instance`: select ✅ insert ❌ update ❌ delete ❌; RPC-only/service-only writes ✅ (including future-plan substitution apply).
- `set_prescription`: select ✅ insert ❌ update ❌ delete ❌; RPC-only/service-only writes ✅.
- `workout_session`: select ✅ insert ❌ update ❌ delete ❌; RPC-only writes ✅ (`rpc_transition_workout_session`,`rpc_recover_local_draft` create path).
- `set_log`: select ✅ insert ❌ update ❌ delete ❌; RPC-only append writes ✅ (`rpc_append_set_logs`,`rpc_recover_local_draft` replay path).
- `post_workout_check_in`: select ✅ insert ❌ update ❌ delete ❌; RPC-only upsert ✅.
- `recommendation`: select ✅ insert ❌ update ❌ delete ❌; RPC-only writes ✅ (`rpc_apply_recommendation_action`).
- `exercise_substitution`: select ✅ insert ❌ update ❌ delete ❌; RPC-only writes ✅ (`rpc_apply_substitution`); service-only direct writes ❌.

Domain errors: cross-user access always `403 FORBIDDEN_RESOURCE`; same-user invalid FK chain always `422 FK_OWNERSHIP_VIOLATION`.

Named RPC list and contracts:
- `rpc_generate_plan(goal_plan_id uuid, equipment_profile_id uuid, preferred_weekdays week_day[]) -> plan_version_payload` errors: `422 GENERATION_UNFILLABLE`, `403`.
- `rpc_accept_plan(plan_version_id uuid, version_token int) -> plan_version` errors: `409 VERSION_CONFLICT`.
- `rpc_regenerate_plan(plan_version_id uuid, version_token int, reason text, generation_trigger text) -> plan_version_payload`.
- `rpc_revert_plan(plan_version_id uuid, version_token int) -> plan_version_payload`.
- `rpc_transition_workout_session(session_id uuid, action text, version_token int, payload jsonb) -> workout_session`.
- `rpc_append_set_logs(session_id uuid, version_token int, entries jsonb) -> set_log_result`.
- `rpc_recover_local_draft(payload jsonb) -> recovery_result`.
- `rpc_apply_recommendation_action(recommendation_id uuid, action text, version_token int, ignored_reason text, ignored_reason_text text null) -> recommendation`.
- `rpc_apply_substitution(target_context text, plan_version_id uuid, workout_day_id uuid, workout_session_id uuid, exercise_instance_id uuid, replacement_exercise_catalog_id uuid, version_token int, idempotency_key text) -> substitution_result`; verifies ownership, validates FK chain to selected context, requires matching version token on mutable parent (`plan_version` for future_plan or `workout_session` for session_override), idempotent on `(user_id,idempotency_key,exercise_instance_id,replacement_exercise_catalog_id)`, errors: `403 FORBIDDEN_RESOURCE`, `409 VERSION_CONFLICT`, `422 FK_OWNERSHIP_VIOLATION|INVALID_SUBSTITUTION_TARGET|PAIN_SUPPRESSION_CONFLICT`.
- `rpc_add_equipment_item(equipment_profile_id uuid, equipment_profile_version_token int, equipment_key text, idempotency_key text) -> equipment_profile_item_result`; validates ownership + `equipment_key` exists and active, bumps parent token +1, idempotent replay returns existing row, errors: `409 VERSION_CONFLICT`, `422 INVALID_EQUIPMENT_KEY|DUPLICATE_EQUIPMENT_ITEM`.
- `rpc_remove_equipment_item(equipment_profile_id uuid, equipment_profile_item_id uuid, equipment_profile_version_token int, idempotency_key text) -> equipment_profile_item_result`; validates ownership chain/profile membership, enforces non-negative remaining item rules from generator preconditions, bumps parent token +1, idempotent delete replay acknowledged, errors: `409 VERSION_CONFLICT`, `422 FK_OWNERSHIP_VIOLATION|MIN_EQUIPMENT_VIOLATION`.
- `rpc_upsert_post_workout_check_in(workout_session_id uuid, version_token int, payload jsonb, idempotency_key text) -> post_workout_check_in`; validates ownership + session mutability, upsert-by-session semantics with token increment exactly +1, same key+payload replay safe, errors: `409 VERSION_CONFLICT`, `422 INVALID_STATE_TRANSITION|CHECKIN_SCHEMA_INVALID`.
- `rpc_upsert_trust_survey_response(week_index int, trust_rating int, submitted_at timestamptz, version_token int, idempotency_key text) -> trust_survey_response`; validates `(user_id,week_index)` ownership semantics, enforces required optimistic concurrency token behavior from §3.12, errors: `409 VERSION_CONFLICT`, `422 TRUST_SURVEY_SCHEMA_INVALID`.
- `rpc_request_account_deletion(version_token int, idempotency_key text) -> deletion_request_result`; server-only mutation of `user_profile.pending_deletion_at` and `user_profile.hard_delete_by`, enforces ownership and policy gates, same-key replay safe, errors: `409 VERSION_CONFLICT`, `422 INVALID_STATE_TRANSITION|ACCOUNT_DELETION_POLICY_BLOCKED`.

Trigger names/responsibilities:
- `trg_set_updated_at_*`: set `updated_at=now()`.
- `trg_bump_version_token_*`: increment `version_token` on successful mutable update.
- `trg_enforce_user_ownership_chain`: assert child-parent `user_id` equality.
- `trg_set_log_prescribed_chain_guard`: validate prescribed set references same exercise/session/plan chain.
- `trg_equipment_profile_item_parent_token_bump`: bump parent `equipment_profile.version_token` +1 on insert/update/delete.

FK chain validation examples:
- set log prescribed chain: `set_log.workout_session_id -> workout_session.workout_day_id -> exercise_instance.workout_day_id`, and `set_log.set_prescription_id -> set_prescription.exercise_instance_id == set_log.exercise_instance_id`.
- same-user wrong chain returns `422 FK_OWNERSHIP_VIOLATION` even when user owns both rows independently.


### 5.2 RPC/endpoint error-code matrix (implementation-readiness required)
For each named RPC and mutable endpoint, contract requires: allowed caller, ownership checks, versionToken requirement, idempotency behavior, transaction boundary, success mutation, and domain errors.

| Operation | Allowed caller | Ownership/FK checks | versionToken | Idempotency | Transaction boundary | Success mutation | Domain errors (in addition to 400/401/404/429/500) |
|---|---|---|---|---|---|---|---|
| `rpc_generate_plan` / `POST /v1/plans/generate` | Auth user via API (service executes RPC) | `goal_plan.user_id` + `equipment_profile.user_id` == `auth.uid()` | Not required | Required key; replay returns same plan payload | Single tx create `plan_version`,`workout_day`,`exercise_instance`,`set_prescription` | New draft plan + persisted explanation/coverage/selection reasons | `403 FORBIDDEN_RESOURCE`, `422 FK_OWNERSHIP_VIOLATION`, `422 GENERATION_UNFILLABLE` |
| `rpc_accept_plan` / `POST /v1/plans/{id}/accept` | Auth user | plan ownership + active-plan constraints | Required exact | Required key; safe replay | Single tx archive prior active + activate target | Plan status/token update only | `403 FORBIDDEN_RESOURCE`, `409 VERSION_CONFLICT`, `422 INVALID_STATE_TRANSITION` |
| `rpc_regenerate_plan` / `POST /v1/plans/{id}/regenerate` | Auth user | source plan ownership | Required exact source token | Required key; replay returns created plan | Single tx new plan tree create | New draft plan row/tree persisted | `403`, `409 VERSION_CONFLICT`, `422 GENERATION_UNFILLABLE` |
| `rpc_revert_plan` / `POST /v1/plans/{id}/revert` | Auth user | source plan ownership | Required exact source token | Required key | Single tx clone source + activate reverted clone | New reverted-active plan row/tree | `403`, `409 VERSION_CONFLICT`, `422 INVALID_REVERT_SOURCE` |
| `rpc_transition_workout_session` / §3.2 mutators | Auth user | session ownership + status transition validity | Required exact | Required key for all mutators; replay same transition | Single-session state tx | session status/timestamps/token | `403`, `409 VERSION_CONFLICT`, `422 INVALID_STATE_TRANSITION` |
| `rpc_append_set_logs` / `POST .../set-logs` | Auth user | session/exercise/set chain ownership | Required exact | Required key + `clientMutationId` dedupe | Append tx for all entries | new set_log rows + session token bump | `403`, `409 VERSION_CONFLICT`, `422 FK_OWNERSHIP_VIOLATION`, `422 SET_LOG_SCHEMA_INVALID` |
| `rpc_upsert_post_workout_check_in` / `PUT .../post-workout-check-in` | Auth user | session ownership + session mutability | Required exact | Required key; same payload replay safe | Upsert tx per session | check-in upsert + token bump | `403`, `409 VERSION_CONFLICT`, `422 CHECKIN_SCHEMA_INVALID`, `422 INVALID_STATE_TRANSITION` |
| `rpc_apply_recommendation_action` / §3.8 mutators | Auth user | recommendation ownership + pending status rules | Required exact | Required key; replay safe | Single-row update tx | recommendation status + ignored fields | `403`, `409 VERSION_CONFLICT`, `422 INVALID_RECOMMENDATION_ACTION` |
| `rpc_apply_substitution` / `POST /v1/substitutions/apply` | Auth user | plan/day/session/exercise chain ownership | Required exact (`plan_version` or `workout_session`) | Required key scoped as §5.1 | Single tx; future_plan path creates replacement exercise+sets | substitution row + optional replacement exercise instance with `selection_reason_json` | `403`, `409 VERSION_CONFLICT`, `422 FK_OWNERSHIP_VIOLATION`, `422 INVALID_SUBSTITUTION_TARGET`, `422 PAIN_SUPPRESSION_CONFLICT` |
| `rpc_add_equipment_item` / `POST .../items` | Auth user | profile ownership + active key exists | Required exact parent token | Required key; replay returns existing row | Single tx insert item + bump parent token | equipment item create | `403`, `409 VERSION_CONFLICT`, `422 INVALID_EQUIPMENT_KEY`, `422 DUPLICATE_EQUIPMENT_ITEM` |
| `rpc_remove_equipment_item` / `DELETE .../items/{itemId}` | Auth user | profile/item ownership chain | Required exact parent token | Required key; replay acknowledged | Single tx delete/archive item + bump token | equipment item deletion effect | `403`, `409 VERSION_CONFLICT`, `422 FK_OWNERSHIP_VIOLATION`, `422 MIN_EQUIPMENT_VIOLATION` |
| `rpc_recover_local_draft` / `POST /v1/sync/local-drafts/recover` | Auth user | ownership and plan/day/session chain per recovery path | Required when attaching to existing mutable session | Required key | Single recovery tx (create or attach path) | session recovered + set-log replay | `403`, `409 STALE_VERSION_TOKEN`, `409 SESSION_ALREADY_COMPLETED`, `422 PLAN_VERSION_MISMATCH`, `422 WORKOUT_DAY_MISMATCH` |
| `rpc_upsert_trust_survey_response` / `POST /v1/trust-survey` | Auth user | row ownership by `(user_id,week_index)` | Required exact (as §3.12) | Required key | Upsert tx | trust row insert/update + token bump | `403`, `409 VERSION_CONFLICT`, `422 TRUST_SURVEY_SCHEMA_INVALID` |
| `PUT /v1/user-profile` | Auth user | `user_profile.user_id = auth.uid()` only; cross-user denied | Required exact `user_profile.version_token` | Required key; same payload replay returns current row | Single-row update tx | profile fields updated + `version_token` +1 + `updated_at` | `403 FORBIDDEN_RESOURCE`, `409 VERSION_CONFLICT`, `422 USER_PROFILE_SCHEMA_INVALID` |
| `POST /v1/goal-plans` | Auth user | `goal_plan.user_id = auth.uid()`; enforce single active goal invariant | Not required for create | Required key; replay returns created goal row | Single-row insert tx | new goal plan created (`status='active'`) and prior active goal archived in same tx when applicable | `403 FORBIDDEN_RESOURCE`, `422 GOAL_PLAN_SCHEMA_INVALID`, `422 INVALID_STATE_TRANSITION` |
| `PATCH /v1/goal-plans/{id}` | Auth user | target goal ownership + valid status transition chain | Required exact `goal_plan.version_token` | Required key; replay safe | Single-row update tx | goal fields/status update + token bump | `403 FORBIDDEN_RESOURCE`, `409 VERSION_CONFLICT`, `422 GOAL_PLAN_SCHEMA_INVALID`, `422 INVALID_STATE_TRANSITION` |
| `POST /v1/equipment-profiles` | Auth user | `equipment_profile.user_id = auth.uid()` | Not required for create | Required key; replay returns created profile | Single-row insert tx | equipment profile created + default active semantics per §3 | `403 FORBIDDEN_RESOURCE`, `422 EQUIPMENT_PROFILE_SCHEMA_INVALID` |
| `PATCH /v1/equipment-profiles/{id}` | Auth user | profile ownership | Required exact `equipment_profile.version_token` | Required key; replay safe | Single-row update tx | equipment profile update + token bump | `403 FORBIDDEN_RESOURCE`, `409 VERSION_CONFLICT`, `422 EQUIPMENT_PROFILE_SCHEMA_INVALID` |
| `DELETE /v1/equipment-profiles/{id}` | Auth user | profile ownership + ownership chain for child items | Required exact `equipment_profile.version_token` | Required key; replay acknowledged for already-archived target | Single tx archive/delete profile and apply active-profile invariant adjustments | profile archived/deleted per §3 + token bump/invariant upkeep | `403 FORBIDDEN_RESOURCE`, `409 VERSION_CONFLICT`, `422 FK_OWNERSHIP_VIOLATION`, `422 INVALID_STATE_TRANSITION` |
| Exercise preference mutators (`POST/PATCH/DELETE /v1/exercise-preferences...`) | Auth user | preference row ownership + `exercise_catalog` FK validity (`is_active=true`) | POST: not required; PATCH/DELETE: required exact `user_exercise_preference.version_token` | Required key; create replay returns same row; patch/delete replay safe | Single-row tx per mutation | create/update/delete (or archive if soft-delete) preference row + token bump on mutable updates | `403 FORBIDDEN_RESOURCE`, `409 VERSION_CONFLICT`, `422 FK_OWNERSHIP_VIOLATION`, `422 EXERCISE_PREFERENCE_SCHEMA_INVALID` |
| User limitation mutators (`POST /v1/user-limitations`, `PATCH /v1/user-limitations/{id}`, `POST /v1/user-limitations/{id}/resolve`) | Auth user | limitation ownership; resolve path enforces active->resolved transition | POST: not required; PATCH/resolve: required exact `user_limitation.version_token` | Required key for each; replay safe | Single-row tx per mutation (resolve may be RPC path) | limitation created/updated/resolved + token bump | `403 FORBIDDEN_RESOURCE`, `409 VERSION_CONFLICT`, `422 USER_LIMITATION_SCHEMA_INVALID`, `422 INVALID_STATE_TRANSITION` |
| `PATCH /v1/notification-preferences` | Auth user | `notification_preference.user_id = auth.uid()` | Required exact `notification_preference.version_token` | Required key; replay safe | Single-row upsert/update tx | notification preferences updated + token bump | `403 FORBIDDEN_RESOURCE`, `409 VERSION_CONFLICT`, `422 NOTIFICATION_PREFERENCES_SCHEMA_INVALID` |
| `rpc_request_account_deletion` / `POST /v1/account/deletion-request` | Auth user | only own `user_profile` mutable; no direct client write to `pending_deletion_at`/`hard_delete_by` columns | Required exact `user_profile.version_token` | Required key; replay returns same pending-deletion state | Single tx sets deletion timestamps + audit record and activates pending-deletion RLS gating | `user_profile.pending_deletion_at` + `hard_delete_by` server-set, token +1 | `403 FORBIDDEN_RESOURCE`, `409 VERSION_CONFLICT`, `422 INVALID_STATE_TRANSITION`, `422 ACCOUNT_DELETION_POLICY_BLOCKED` |

Migration-generated policy verification evidence remains a later implementation-readiness deliverable in §10.C and is not marked complete in this contract revision.

---

## 6) Acceptance tests mapped to risk

Must-pass tests:
1. Full schema DDL tests for all tables in §2.
2. `version_token` stale write (`409`) and +1 increment on success across all mutable tables.
3. Endpoint contract tests with request/response/error examples for all MVP endpoints defined in §3.
4. Regeneration safety matrix behavior tests for each session status and unsynced draft scenarios.
5. Local draft recovery conflict and duplicate `clientMutationId` replay tests.
6. RLS tests: cross-user denied (`403`), same-user wrong chain (`422`).
7. FK invariants tests for session/day/plan and set_log references.
8. Notification defaults/quiet-hours tests: workout reminder default on, missed-workout follow-up default on, recommendation-pending review default on, streak milestone default off (opt-in), and every notification type honors timezone + quiet hours (9pm–8am local).
9. Equipment taxonomy enforcement (`equipment_key` must exist in `equipment_catalog`) and parent token bump on item mutations.
10. Exercise seed verification: catalog populated with ~80 curated exercises.
11. Notification preferences cover workout reminder, missed-workout follow-up, recommendation-pending review, streak milestone; quiet-hours suppression enforced in user timezone for each type.
12. Analytics allowlist enforcement rejects non-allowlisted events and blocks payloads containing prohibited PII/sensitive fields.
13. Trust survey persistence + RLS: week-index capture persists rating (1..5), enforces unique `(user_id, week_index)`, user can read own rows only, cross-user read/write blocked.
14. Recommendation `triggerFactors` and `inputsUsed` schema validator accepts allowlisted keys and rejects unknown keys.
15. Substitution endpoints enforce `future_plan` vs `session_override` behavior and guarantee completed workout history/set logs are never rewritten.
16. Onboarding age gate: under-18 onboarding/account-save rejected with `422 UNDERAGE_NOT_ALLOWED`; valid adult onboarding completion succeeds only when all required PRD onboarding fields are present.
17. Pain suppression determinism: shoulder, elbow, wrist, lower_back, upper_back, hip, knee, ankle, and other cases each produce expected suppression outcomes from `suppression_tags` mapping.
18. Account deletion request: setting pending deletion immediately blocks app data access, returns `pendingDeletionAt` and `hardDeleteBy`, and preserves legal-retention open decision flags.
19. Readiness checklist test fails if §10.B marked complete while §9 open decisions remain unresolved.

20. Movement-intent domain enforcement: `exercise_catalog.movement_intents` and pain suppression `suppressedMovements` reject values outside canonical `movement_intent` enum.
21. Hard disqualifier semantics: disliked is scored penalty (not filtered), pain/limitations hard-filter, experience-level filter relaxes only through fallback when alternatives absent.
22. Generator reason persistence: generated plan persists non-empty `plan_explanation_json` and `movement_coverage_summary_json`; every generated `exercise_instance` persists non-empty `selection_reason_json` with required keys.
23. Plan responses: generate/accept/regenerate/revert responses include `planExplanation`, `movementCoverageSummary`, and per-exercise `selectionReason` mapped from persisted DB fields.
24. Future-plan substitution reason persistence: applying future-plan substitution creates replacement `exercise_instance` whose `selection_reason_json` is populated and returned as `newExerciseSelectionReason`.
25. Onboarding-abandoned local sample plan restore-before-expiry: reopen same device/install within 7 days shows “Save your plan?” restore prompt and restores local sample plan without backend user-owned writes.
26. Onboarding-abandoned local sample plan discard-after-expiry: after >7 days, local sample plan is discarded and restore prompt is not shown.
27. Profile RIR toggle persistence: `show_rir_field` defaults false, PATCH/PUT profile updates return `showRirField`, and concurrent writes enforce `409 VERSION_CONFLICT`.
28. Pain suppression mapping precision: shoulder suppresses back compounds via tags (not blanket horizontal/vertical pull movement suppression); lower_back and ankle suppression include `loaded_standing`; `other` suppresses nothing.
### 6.1 Concrete Given/When/Then acceptance tests
 Concrete Given/When/Then acceptance tests
- Given two same-day slots and identical top candidate, when generator assigns slot2, then cross-slot exclusion picks next candidate.
- Given 4-day split and previous workout used exercise X for horizontal_push, when next same-intent day generated, then X not reused unless option pool <2.
- Given missing equipment for vertical_pull and no eligible alternatives, when generating, then slot marked unfilled and warning emitted.
- Given session length cap 45 min and initial estimate 62 min, when duration pass runs, then accessory intents are cut in defined priority order.
- Given pain suppression on shoulder and horizontal_push slot, when substitution runs, then non-shoulder-aggravating substitute selected and load converted per rules.
- Given last two exposures meet add-load threshold, when recommendation engine runs, then add-load recommendation produced.
- Given thresholds partially met, when engine runs, then hold recommendation produced.
- Given repeated misses + form breakdown, when engine runs, then reduce or deload per precedence.
- Given pain location lower_back with hinge movement, when recommendation engine runs, then add-load/add-reps suppressed.
- Given not_started same-day session and regenerate request without explicit replace, when regenerate executes, then original session preserved.
- Given in_progress session and plan regenerate, when operation completes, then session remains on original plan_version_id.
- Given statuses completed/partial/skipped/completed_outside_app, when regenerate or equipment change occurs, then historical states preserved unchanged.
- Given unsynced local draft before first sync, when destructive op attempted, then API blocks with recover/discard requirement.
- Given unsynced draft after partial sync, when recover called, then duplicate clientMutationIds replay as non-fatal and unresolved conflicts returned.
- Given duplicate clientMutationId for same session, when set-logs endpoint called, then server replays prior success with same log identifiers.
- Given stale versionToken, when mutable endpoint called, then `409 VERSION_CONFLICT` with `currentVersionToken`.
- Given user A token attempts user B row, when endpoint called, then `403 FORBIDDEN_RESOURCE`.
- Given same user but plan/day mismatch chain, when set log inserted, then `422 FK_OWNERSHIP_VIOLATION`.
- Given exercise seed load, when validation test runs, then ~80 exercises exist with 6-10 options per movement intent per common equipment profile and all cue/timing fields non-null.


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


## 8.1 MVP analytics and trust-validation contract

Event allowlist (only these events permitted in MVP analytics pipeline):
- `account_created`
- `onboarding_started`
- `onboarding_completed`
- `onboarding_abandoned_with_plan`
- `plan_generated`
- `plan_generation_failed`
- `workout_started`
- `set_logged`
- `workout_completed`
- `workout_completed_partial`
- `exercise_substituted`
- `missed_workout_action_selected`
- `recommendation_shown`
- `recommendation_accepted`
- `recommendation_ignored`
- `recommendation_why_expanded`
- `recommendation_why_then_accepted`
- `trust_survey_submitted`

PII/sensitive boundaries (hard deny, never emitted):
- body weight values
- pain notes
- workout notes
- email/name
- exact age
- exact height
- soreness/pain location specifics

Analytics preference support:
- `analytics_opt_out` is canonical in `user_profile`; if true, emit no product analytics events for that user (including allowlisted events) except strictly required operational/security logs that are outside product analytics pipeline.

Trust survey capture (week-4 metric minimum contract):
- `POST /v1/trust-survey` request `{ versionToken:number, weekIndex:4, trustRating:1..5, submittedAt:timestamptz }` and response with incremented `versionToken`.
- Persist using canonical `trust_survey_response` table contract in §2.9.A (including `updated_at` and `version_token`); request `versionToken` remains required, and successful upsert increments `version_token` exactly +1 for metric-safe concurrency.

## 9) Open decisions inherited from PRD

Blocking decisions outside this contract (if unresolved in PRD) must remain flagged:
- legal/privacy deletion policy and retention text (blocking production data lifecycle implementation).
- final safety disclaimer copy.
- Apple/Google sign-in decision.
- final exercise seed list contents (contract keeps ~80 guidance until finalized list is approved).
- pricing validation.
- WCAG audit cadence and accessibility QA owner assignment.
- §5.5 content author and reviewer confirmation before build week 1.


---

## 10) Implementation readiness checklist

### A) Contract-complete for engineering estimation
- [x] All MVP schemas fully defined (columns/types/nullability/defaults/enums/FKs/uniques/checks/indexes/ownership/versioning/API mapping).
- [x] All endpoints include method/path/auth/headers/request-success-errors/versionToken/idempotency/RLS/transaction notes.
- [x] Regeneration and local-draft safety matrix fully specified.

### B) Architecture-freeze ready
- [x] Contract content finalized for RLS/FK/trigger/RPC enforcement with explicit per-table permissions, named RPC contracts, and complete mutable-endpoint error-code matrix.
- [ ] Migration-generated policy verification evidence produced and reviewed (implementation artifact; not a contract-text gap).
- [x] Acceptance tests mapped to high-risk flows and invariants.
- [ ] All contract §9 open decisions inherited from the PRD / PRD open decisions closed (required before architecture-freeze can be marked complete).
- [x] PRD open decisions explicitly listed as blockers while unresolved.

### C) Production implementation ready
- [ ] PRD §0 pre-build gates are passed and signed off; production implementation remains blocked until this is true.
- [ ] Migration DDL generated exactly from this contract.
- [ ] RPCs implement all state transitions and idempotency semantics.
- [ ] Integration/e2e suite passes all §6 tests in CI.
- [x] Placeholder/meta references to former subsection labels removed or rewritten to current section references where applicable.


## 11) Patch Summary
- Sections patched: §2.6 (`exercise_catalog` eligibility fields/checks), §2.A.5 (experience-level hard-disqualifier/fallback behavior), §4.1.4 (sync/local draft transitions table format validation), §5 (RLS wording consistency), §5.1 (named RPC signatures/contracts list), §5.2 (expanded mutable endpoint + RPC error-code matrix), §6 (trust-survey persistence consistency text), §10.B (readiness checklist accuracy), and this §11 summary.
- Sections deleted: none.
- Remaining blockers before production implementation: PRD §0 pre-build gates must pass/sign-off; all §9 open PRD/contract decisions must be closed; migration DDL and RPC implementations must be generated from this contract; and CI integration/e2e must pass §6 acceptance tests.
- Architecture-freeze remains blocked only by unresolved external/open-decision evidence and implementation artifacts (not by missing contract matrix/RLS/RPC text after this patch).
- Confirmation: this patch preserved existing schema/API/RLS/sync/state-machine/acceptance-test detail and only added/amended targeted consistency and implementation-safety content.
