# CANONICAL DATA AND PLAN GENERATION CONTRACT (FINAL MVP BUILD CONTRACT)

**Status:** Draft contract revision; implementation-ready for engineering estimation and architecture freeze (see §10 A/B checked items). Production implementation remains gated by §10.C PRD pre-build sign-off.  
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
- `movement_intents text[] not null`
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
- `is_unilateral boolean not null default false`
- `is_active boolean not null default true`
- `metadata_json jsonb not null default '{}'::jsonb`
- `created_at timestamptz not null default now()`
- `updated_at timestamptz not null default now()`

Constraints/indexes:
- GIN indexes on `movement_intents`, `primary_muscles`, `equipment_keys`.
- checks: `cardinality(movement_intents)>=1`, `cardinality(primary_muscles)>=1`, `cardinality(equipment_keys)>=1`.
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

### 2.A.2 Split templates by days/week (PRD-exact)
- 2 days/week: Full Body A/B.
- 3 days/week: Full Body A/B/C.
- 4 days/week: Upper/Lower.
- 5 days/week: Upper/Lower + 1 focus day.
- 6 days/week, beginner: Full Body x6 with conservative volume.
- 6 days/week, intermediate: PPL x2.
Deterministic template resolution keys: `(daysPerWeek, experienceLevel)`.
MVP `goalType` is recorded-only metadata and MUST NOT affect template choice, slot counts, volume targets, exercise selection, progression, or recommendation decisions.

### 2.A.3 Initial weekly volume targets (PRD muscle-by-muscle tables)
Generator SHALL use the PRD v0.6 explicit beginner/intermediate muscle-by-muscle weekly set ranges (no major/minor substitution).
- no history -> initialize at low end of PRD range.
- generator target -> midpoint of PRD range.
- progression volume increases are gated: minimum 14 consecutive days of adherence consistency and no high-soreness paired signal and no pain flag.
Any stored copy of ranges in code/data must remain field-equivalent to PRD tables.

### 2.A.4 Duration estimation and accessory-cut priority (PRD-exact)
Core duration estimate per session uses exercise-specific values:
`sum_over_exercises(sum_over_sets(target_reps * tempo_seconds + rest_seconds) + warmup_overhead + setup_overhead)`.
No fixed global `avgRepSeconds` constant is permitted as the core estimation rule.
Threshold: run cut policy when estimated duration `> target_session_minutes * 1.15`.
Accessory-cut priority is PRD-ordered and MUST preserve PRD focus-muscle protection and PRD set-count reduction floors before any slot drop.

### 2.A.5 Hard disqualifiers
Candidate exercise is ineligible if any true:
- inactive in `exercise_catalog`.
- required equipment not in active equipment profile.
- user limitation with severity `hard_block` intersects movement intent, muscle, or equipment.
- exercise marked disliked with active pain flag for matching location in last 14 days.
- experience level below `experience_level_min`.

### 2.A.6 Selection rule precedence, scoring weights, tie-breakers (PRD-exact)
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
Tie-breakers: MUST follow PRD exact order (no substitutions).

### 2.A.7 Cross-slot exclusion, variety enforcement, unfillable-slot fallback (PRD-exact)
Cross-slot exclusion in same day:
- cannot assign same `exercise_catalog_id` to >1 slot.
- cannot assign two hinge-intent heavy spinal-loading lifts if both fatigue_cost >=4.
Variety enforcement remains as previously specified.
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
Algorithm:
1) find candidate pool with same movement_intent.
2) score per §2.A.6.
3) apply §2.A.7 fallback only when required.
4) choose deterministic top candidate by PRD tie-breaker order.
Load handling:
- if user has history with substitute and last successful set met minimum target reps, suggest that last successful load.
- if no history, leave load blank and attach conservative-start prompt.
- bodyweight substitutions: prescribe bodyweight/assisted/variation level.
- machine/non-comparable load scales: leave load blank.
MVP forbids carrying load unchanged by default and forbids bilateral→unilateral conversion heuristics unless introduced as explicitly labeled post-MVP/open decision.

### 2.A.10 Generator outputs
Plan-level explanation output schema (`plan_version.validation_warnings_json` companion API field `planExplanation`):
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
Movement coverage summary output:
- per movement intent: `targetSets`, `assignedSets`, `coverageStatus enum(full|partial|missing)`.

## 2.B) Progression and Recommendation Rules (PRD-exact precedence/threshold contract)

### 2.B.1 Recommendation precedence (strict order)
`pain/safety -> reduce -> hold -> increase -> default hold/add reps`.


### 2.B.1.A Pain-location suppression mapping (machine-testable)

| painLocation | suppressedMuscles | suppressedMovements | substitutionRequiredForEligibleExercises | progressionSuppressed |
|---|---|---|---|---|
| shoulder | [chest,shoulders,back_compound_pull,triceps_overhead_loaded] | [horizontal_push,vertical_push,horizontal_pull,vertical_pull_overhead_loaded] | true | true |
| elbow | [biceps,triceps,loaded_grip_pull] | [horizontal_pull,vertical_pull,arm_elbow_flexion,arm_elbow_extension] | true | true |
| wrist | [loaded_pressing,loaded_pulling] | [horizontal_push,vertical_push,horizontal_pull,vertical_pull] | true | true |
| lower_back | [posterior_chain_loaded,spinal_loading] | [hip_hinge,knee_dominant_squat,loaded_standing] | true | true |
| upper_back | [rows,pull_ups,loaded_carries] | [horizontal_pull,vertical_pull,loaded_carry] | true | true |
| hip | [squat_pattern,hinge_pattern,lunge_pattern] | [knee_dominant_squat,hip_hinge,single_leg] | true | true |
| knee | [squat_pattern,lunge_pattern,leg_extension,leg_press] | [knee_dominant_squat,single_leg] | true | true |
| ankle | [squat_pattern,lunge_pattern,calf,loaded_standing] | [knee_dominant_squat,single_leg,calf_plantarflexion,loaded_standing] | true | true |
| other | [] | [] | false | false |

Application rules: mapping is applied to exercise eligibility filtering, substitution candidate retrieval, and progression recommendation suppression; `other` does not auto-suppress but emits review prompt.

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

RLS/ownership expectations for all endpoints: authenticated user may only read/write rows where `row.user_id = auth.uid()`; cross-user attempts return `403 FORBIDDEN_RESOURCE`; same-user invalid FK chain returns `422 FK_OWNERSHIP_VIOLATION`.
`§3.A`, `§3.B`, and `§3.C` are integrated normative subsections of §3 (not appendices) and define the strict implementation-ready schemas for every endpoint listed in §3.1–§3.11 and §3.C.1–§3.C.8, including method/path/auth/headers/request/response/errors/idempotency/versionToken/RLS/transaction notes.

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
- [x] All MVP schemas fully defined (columns/types/nullability/defaults/enums/FKs/uniques/checks/indexes/ownership/versioning/API mapping).
- [x] All endpoints include method/path/auth/headers/request-success-errors/versionToken/idempotency/RLS/transaction notes.
- [x] Regeneration and local-draft safety matrix fully specified.

### B) Architecture-freeze ready
- [x] RLS/FK/trigger/RPC enforcement finalized with explicit per-table permissions and domain errors.
- [x] Acceptance tests mapped to high-risk flows and invariants.
- [x] PRD open decisions either resolved or explicitly listed as blockers.

### C) Production implementation ready
- [ ] PRD §0 pre-build gates are passed and signed off; production implementation remains blocked until this is true.
- [ ] Migration DDL generated exactly from this contract.
- [ ] RPCs implement all state transitions and idempotency semantics.
- [ ] Integration/e2e suite passes all §6 tests in CI.
- [x] No placeholder language remains.


## 11) Patch Summary
- Sections amended: §1 status, §2.6 (`cue_safety_note` nullability), §2.16 (`soreness_location` nullability + API consistency), §3 ownership/error invariants and normative integration statement, §5.A explicit per-table RLS matrix, §10 readiness checklist.
- Sections added: §2.18 `exercise_substitution` persistence model (future-plan + session-level substitution with FK/ownership/versioning/immutability semantics).
- Sections moved: none; supplemental sections §3.A/§3.B/§3.C, §5.A, and §6.A are explicitly integrated into parent normative sections by wording updates.
- Sections deleted: none. No implementation-critical schema/API/RLS/sync/state-machine/generator/progression/acceptance-test content was removed.
- Remaining blockers before production implementation readiness: §10.C items (migration generation, RPC implementation, CI e2e pass, PRD pre-build gates) remain execution tasks and are intentionally unchecked.

## 3.A) Endpoint schema expansion (authoritative supplement to §3.1–§3.11)
For each endpoint in §3, the following are mandatory and not optional: method/path, auth, required headers, strict request schema, strict success schema, error codes, idempotency behavior, versionToken behavior, transaction notes, RLS ownership checks.

### 3.A.1 Shared request/response envelopes
- Request headers for authenticated mutable endpoints: `Authorization`, `Content-Type: application/json`, `Idempotency-Key`.
- Success envelope: `{ "data": <resource>, "meta": { "requestId": "uuid", "idempotencyReplay": false } }`.
- Error envelope: `{ "error": { "code": "...", "message": "...", "details": {...} } }`.

### 3.A.2 Full schema obligations by endpoint group
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
- Local draft recovery (§3.11): strict local draft schema in §3.B.

### 3.A.3 Endpoint-level idempotency and versionToken behavior
- `POST /v1/plans/generate`: idempotent by key+body; no versionToken required.
- `POST /v1/plans/{id}/accept|regenerate|revert`: require versionToken; stale -> `409 VERSION_CONFLICT`.
- `POST /v1/workout-sessions/start`: idempotent create by key+day+date.
- `POST /v1/workout-sessions/{id}/resume|finish|mark-partial|skip|complete-outside-app`: require versionToken; duplicate key replay returns same transition result.
- `POST /v1/workout-sessions/{id}/set-logs`: dedupe by `(user_id, workout_session_id, client_mutation_id)`; replay returns prior row IDs.
- `PUT /v1/workout-sessions/{id}/post-workout-check-in`: idempotent upsert semantics by session.
- `PATCH /v1/notification-preferences`, limitation patch/resolve, recommendation actions: optimistic concurrency required.

## 3.B) Local draft recovery (full schema and conflict matrix)
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
1) create session from `planVersionId+workoutDayId+scheduledForDate(derived)` in transaction.
2) insert recovered set logs/check-in snapshot.
3) return created `workoutSessionId` and server `versionToken`.

Conflict handling codes:
- `SESSION_ALREADY_COMPLETED`: do not append logs; return latest server session snapshot.
- `PLAN_VERSION_MISMATCH`: reject with expected/current IDs.
- `WORKOUT_DAY_MISMATCH`: reject with expected/current IDs.
- `STALE_VERSION_TOKEN`: reject with `currentVersionToken`.
- `DUPLICATE_CLIENT_MUTATION_ID`: non-fatal replay list in success payload.
- `PARTIAL_SYNC_CONFLICT`: return merged summary + unresolved item list.

## 5.A) RLS/FK/trigger/RPC enforcement expansion (per-table matrix required)
Per-table RLS matrix (explicit):
- `user_profile`: client select ✅ insert ✅ update ✅ delete ❌; RPC-only writes ❌; service-role-only writes ❌; policy `USING (user_id=auth.uid()) WITH CHECK (user_id=auth.uid())`.
- `goal_plan`: select ✅ insert ✅ update ✅ delete ✅(soft-delete); RPC-only writes ❌; service-only ❌; same policy.
- `equipment_profile`: select ✅ insert ✅ update ✅ delete ✅(archive); RPC-only writes ❌; service-only ❌; same policy.
- `equipment_profile_item`: select ✅ insert ❌ update ❌ delete ❌; RPC-only writes ✅ (`rpc_add/remove_equipment_item`); service-only ❌; read policy `USING (user_id=auth.uid())`.
- `exercise_catalog`: select ✅(only `is_active=true`) insert ❌ update ❌ delete ❌; RPC-only writes ❌; service-only ✅.
- `user_exercise_preference`: client CRUD ✅ via ownership policy.
- `user_limitation`: client select ✅ insert ✅ update ✅ delete ❌(resolve instead); resolve is RPC-only ✅.
- `notification_preference`: select ✅ insert ✅ update ✅ delete ❌.
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
- `rpc_regenerate_plan(plan_version_id uuid, version_token int, reason text) -> plan_version_payload`.
- `rpc_revert_plan(plan_version_id uuid, version_token int) -> plan_version_payload`.
- `rpc_transition_workout_session(session_id uuid, action text, version_token int, payload jsonb) -> workout_session`.
- `rpc_append_set_logs(session_id uuid, version_token int, entries jsonb) -> set_log_result`.
- `rpc_recover_local_draft(payload jsonb) -> recovery_result`.
- `rpc_apply_recommendation_action(recommendation_id uuid, action text, version_token int, ignored_reason text) -> recommendation`.

Trigger names/responsibilities:
- `trg_set_updated_at_*`: set `updated_at=now()`.
- `trg_bump_version_token_*`: increment `version_token` on successful mutable update.
- `trg_enforce_user_ownership_chain`: assert child-parent `user_id` equality.
- `trg_set_log_prescribed_chain_guard`: validate prescribed set references same exercise/session/plan chain.
- `trg_equipment_profile_item_parent_token_bump`: bump parent `equipment_profile.version_token` +1 on insert/update/delete.

FK chain validation examples:
- set log prescribed chain: `set_log.workout_session_id -> workout_session.workout_day_id -> exercise_instance.workout_day_id`, and `set_log.set_prescription_id -> set_prescription.exercise_instance_id == set_log.exercise_instance_id`.
- same-user wrong chain returns `422 FK_OWNERSHIP_VIOLATION` even when user owns both rows independently.

## 6.A) Concrete Given/When/Then acceptance tests
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

## 3.C) Missing MVP endpoint contracts (authoritative supplement to §3)

### 3.C.1 User profile / onboarding
- `PUT /v1/user-profile`
  - Request: `{ versionToken?: number, birthDate?: date|null, sex?: enum|null, heightCm?: number|null, bodyWeight?: number|null, bodyWeightUnit?: kg|lb|null, experienceLevel?: beginner|intermediate|null, timezone: string, onboardingCompletedAt?: timestamptz|null }`
  - Success: `{ data: { ...userProfile, versionToken:number } }`
  - Errors: `422` invalid ranges/combinations, `409 VERSION_CONFLICT`, `403` cross-user.
  - Idempotency/versioning: requires `Idempotency-Key`; create path may omit versionToken, update path requires exact current token.
  - RLS: only `user_profile.user_id=auth.uid()` row.

### 3.C.2 Goal plan CRUD
- `POST /v1/goal-plans`, `PATCH /v1/goal-plans/{id}` with strict fields from §2.2.
- Active-goal uniqueness enforced transactionally; update/archive prior active in same transaction.

### 3.C.3 Equipment profile and items
- `POST /v1/equipment-profiles`, `PATCH /v1/equipment-profiles/{id}`, `DELETE /v1/equipment-profiles/{id}` (soft-delete/archive only if referenced).
- `POST /v1/equipment-profiles/{id}/items` request `{ equipmentProfileVersionToken:number, equipmentKey:string }`.
- Delete item endpoint in §3.9 remains authoritative; add endpoint follows same parent token bump semantics.

### 3.C.4 Exercise catalog read APIs
- `GET /v1/exercises?movementIntent=&equipmentKey=&limit=&cursor=` and `GET /v1/exercises/{id}`.
- Read-only through API proxy to seed tables; direct client table access is allowed only read (`is_active=true`) under RLS.

### 3.C.5 Exercise preference CRUD
- `POST /v1/exercise-preferences`, `PATCH /v1/exercise-preferences/{id}`, `DELETE /v1/exercise-preferences/{id}` using §2.7 schema and versionToken for patch/delete.

### 3.C.6 Substitution candidates + apply
- `POST /v1/substitutions/candidates`
  - Request: `{ planVersionId:uuid, workoutDayId:uuid, exerciseInstanceId:uuid, context:'future_plan'|'scheduled_session', painLocation?:pain_location[], unavailableEquipmentKeys?:string[], versionToken:number }`
  - Success: `{ data:{ candidates:[{ exerciseCatalogId:uuid, score:number, whyThisSubstitution:{ movementIntentMatch:boolean, equipmentCompatible:boolean, painSuppressionApplied:boolean, fatigueComparison:string, historyLoadHint?:string|null } }], suppressedByPainMapping:boolean } }`
- `POST /v1/substitutions/apply`
  - Request: `{ targetContext:'future_plan'|'session_override', planVersionId:uuid, workoutDayId?:uuid, workoutSessionId?:uuid, exerciseInstanceId:uuid, replacementExerciseCatalogId:uuid, versionToken:number }`
  - Success: `{ data:{ substitutionId:uuid, mutationType:'replace_future_exercise_instance'|'session_level_override', newExerciseInstanceId?:uuid, preservedOriginalExerciseInstanceId:uuid } }`
  - Mutation rule: future-plan applies by creating replacement `exercise_instance` row and retiring original from future view; session override writes to `session_exercise_override` view/RPC result only. Completed workout history and completed set logs are never rewritten.

### 3.C.7 Recommendation action ignoredReason other text
- `POST /v1/recommendations/{id}/ignore` request extends to `{ versionToken:number, ignoredReason:enum, ignoredReasonText?:string|null }`; `ignoredReasonText` required when `ignoredReason='other'`.

### 3.C.8 Transaction/RPC notes for endpoint groups
- Workout-session status transitions are RPC-only (`rpc_transition_workout_session`).
- Recommendation accept/ignore/dismiss are RPC-only (`rpc_apply_recommendation_action`).
- Set logs are append/idempotent insert only via RPC (`rpc_append_set_logs`); no client update/delete contract.
