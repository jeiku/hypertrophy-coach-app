# CANONICAL DATA AND PLAN GENERATION CONTRACT

**Status:** Reconciliation Draft (PRD v0.6 alignment in review; **not implementation-ready**)  
**Source of truth:** `PRDv0.6.md`  
**Revision date:** 2026-05-10

---

## 0) Scope, gates, and blocking status

- This contract supersedes prior v0.4 assumptions and is scoped to MVP behavior defined in PRD v0.6 only.
- Production implementation is explicitly blocked until PRD §0 pre-build gates are passed and this reconciliation is reviewed/approved.
- This document is design-contract only. No code, migrations, API handlers, tests, or seed artifacts are created by this reconciliation.
- Deterministic rules-based generation remains the only MVP generation mode.
- Planned structures (`plan_version` tree) remain separate from execution structures (`workout_session`, `set_log`, check-ins, recommendations).
- Regeneration creates new plan versions; execution history remains immutable and non-migrated.

---

## 1) MVP boundaries (explicitly in / out)

### 1.1 In MVP
- Onboarding/profile inputs required for generation: training days/week, session duration target, focus muscles, active equipment profile, experience level, and other PRD-v0.6-required generator inputs.
- Profile demographic/body metrics retained for future expansion: age, sex, height, body weight.
- Single active equipment profile used across all training days.
- Exercise catalog (~80 curated exercises, exact lock pending).
- Draft plan generation, user acceptance/promotion, plan regeneration safeguards.
- Post-workout check-ins, progression recommendations, missed-workout recovery, offline draft sync.

### 1.2 Deferred from MVP (v1.1+)
- Nutrition tracking and related data entities/APIs (FoodLog, SavedMeal, macro/calorie targets, maintenance overrides, nutrition safety bounds).
- Per-day equipment calendar behavior.
- Weekly review notifications.

### 1.3 Constraint notes
- Goal type (`goal_type`) is captured as profile/program metadata in MVP but **does not alter MVP generator output rules**.
- Age/sex/height/body weight are stored but **do not introduce nutrition logic into MVP generation**.

---

## 2) Canonical enums and value sets

### 2.1 Core enums
- `experience_level`: `beginner | intermediate`
- `goal_type`: `bulk | cut | recomp` (recorded only; no generator branching in MVP)
- `session_status`: `not_started | in_progress | completed | partial | abandoned | skipped | completed_outside_app | deleted`
- `sex`: `male | female`
- `weight_unit`: `kg | lb`
- `set_source`: `prescribed | added`
- `set_state`: `active | skipped | deleted`

### 2.2 Plan lifecycle enums
- `plan_version_status`: `draft | active | archived | reverted`

### 2.3 Check-in and recommendation enums
- `soreness_level`: `none | mild | moderate | high`
- `pain_type`: `sharp | dull_aching | joint | nerve_like | other`
- `fatigue_level`: `low | normal | high`
- `pain_location`:
  - `shoulder`, `elbow`, `wrist`, `lower_back`, `upper_back`, `hip`, `knee`, `ankle`, `other`
- `recommendation_type`:
  - `add_reps | add_load | hold_load | reduce_load | substitute_exercise | suggest_deload`
- `recommendation_status`:
  - `pending | accepted | ignored | dismissed | expired`
- `recommendation_ignored_reason`:
  - `too_aggressive | wrong_exercise | not_today | disagree_with_reasoning | other | no_reason_given`

### 2.4 Movement-intent taxonomy
- MVP generator must use PRD v0.6 §8.3 movement-intent slot taxonomy.
- If implementation keeps a more granular internal enum, an explicit non-lossy mapping table to PRD §8.3 slots is required and must be reviewed before implementation.

---

## 3) Canonical data model (MVP reconciled)

> Notes:
> - Below is the logical contract shape (not migration SQL).
> - Removed from MVP: nutrition tables/APIs and per-day equipment calendar table/API.

### 3.1 `user_profile`
- Holds identity-owned profile data, including:
  - demographic/body metrics: age (or birth date), sex, height, body weight + preferred unit
  - training preferences and timezone
  - onboarding completion metadata
- Constraint: one profile per auth user.
- Privacy: user-owned read/write only.

### 3.2 `goal_plan`
- Holds user-level planning preferences including:
  - `goal_type` (recorded metadata)
  - `days_per_week`, `session_length_min`, `focus_muscles`
- Explicitly removed fields from MVP:
  - calorie/macro targets
  - maintenance calorie override
- Constraint: one active goal plan per user.

### 3.3 `equipment_profile`
- User-owned named profiles with active flag.
- Exactly one active equipment profile per user used for generation.

### 3.4 `equipment_profile_item`
- Equipment inventory rows under a profile.
- Ownership-chain enforcement: item user must match parent profile user.

### 3.5 `plan_version`
- Required fields:
  - `id`, `user_id`, `goal_plan_id`, `based_on_plan_version_id?`, `version_number`
  - `status: draft|active|archived|reverted`
  - `generate_mode` (`initial|regenerate|revert_clone`)
  - generator input/output snapshots
  - validation warnings
- Lifecycle rules:
  - new generation saves as `draft`
  - user acceptance promotes one `draft` to `active`
  - exactly one `active` per user at all times
  - prior active transitions to `archived` when replaced
  - revert creates a new `active` clone marked `reverted` lineage (`generate_mode=revert_clone`, link via `based_on_plan_version_id`)

### 3.6 `workout_day`, `exercise_instance`, `set_prescription`
- Planned program tree under `plan_version`.
- `exercise_instance` stores movement-intent slot assignment and trim priority metadata.
- No day-specific equipment override references.

### 3.7 `workout_session`, `set_log`
- Execution layer remains append-only for logs.
- `set_log` idempotency key uses `clientMutationId` contract (server alias `mutationId` accepted).
- History immutability:
  - Completed/partial/abandoned/skipped/completed_outside_app sessions are immutable.
  - In-progress sessions stay attached to original `plan_version_id`.
  - SetLogs are never rewritten/migrated by regeneration.

### 3.8 `post_workout_check_in`
- Replaces old score model with:
  - `sorenessLevel: none|mild|moderate|high`
  - `sorenessLocation: pain_location[]?`
  - `painFlag: boolean`
  - `painLocation: pain_location[]` (required if `painFlag=true`)
  - `painType: sharp|dull_aching|joint|nerve_like|other` (required if `painFlag=true`)
  - `painNotes: string?` (**never sent to analytics**)
  - `fatigueLevel: low|normal|high?`
  - `formBreakdown: boolean?`
- Ownership and privacy controls required for all check-in data.

### 3.9 `recommendation`
- Required fields:
  - `id, userId, type, status, title, userFacingReason, educationalContext, triggerFactors[], inputsUsed, targetEntityType, targetEntityId, suggestedChange, ignoredReason?, createdAt, updatedAt`
- Explicitly removed from MVP schema:
  - AI/source provenance fields.
- “Why?” visibility:
  - every recommendation surface must expose `userFacingReason + triggerFactors + key inputsUsed`.
- Ignore/dismiss feedback:
  - optional `ignoredReason` enum
  - optional free-text “other” note, stored privately and never sent to analytics.

### 3.10 Removed/deferred entities
- No MVP `food_log`, `saved_meal`, nutrition target tables, or nutrition safety tables.
- No MVP `equipment_calendar_entry` (or equivalent weekday equipment mapping table).

---

## 4) Exercise catalog and seed contract

- Old fixed “50 exercises” contract is removed.
- New requirement: curated catalog of approximately 80 exercises per PRD v0.6; **exact count must be locked before Build Week 1**.
- Exercise catalog record must include:
  - `name`
  - `movement_intent` or `movement_intent[]` (per chosen schema strategy)
  - `primary_muscles[]`
  - `secondary_muscles[]`
  - `equipment_required[]`
  - `difficulty`
  - `fatigue_cost`
  - `tempo_seconds`
  - `rest_seconds`
  - `warmup_overhead`
  - `setup_overhead`
  - `cues` object with:
    - `setup`
    - `execution`
    - `common_mistake`
    - `safety_note?`
- Prior `instruction_cues` / `safety_cues`-only shape is deprecated.
- Pre-build acceptance must verify:
  - equipment-profile coverage across common MVP profiles
  - timing field completeness
  - cue structure completeness.

---

## 5) Plan version lifecycle and APIs

### 5.1 Lifecycle invariants
- Generated plans are created as `draft`.
- User action is required to accept/promote a draft.
- Exactly one active plan per user enforced transactionally and by DB uniqueness invariant.
- Revert behavior creates a new active clone; it does not mutate historical versions.
- Workout logs remain attached to original sessions/plan versions (never migrated).

### 5.2 Contracted endpoints (logical)
- `POST /plans/generate` → creates `draft` plan version.
- `POST /plans/{planVersionId}/accept` → promotes draft to active, archives prior active, returns new state/version token.
- `POST /plans/{planVersionId}/revert` → creates new active clone from prior version, returns clone metadata.
- All endpoints must be idempotent with conflict-safe responses.

---

## 6) Onboarding sample plan before account creation

### 6.1 Required behavior
- Sample plan generation must be available pre-account (per PRD v0.6).
- Must not be silently replaced with account-required generation.

### 6.2 Safe technical flow (selected)
- Use transient anonymous generation request with no persisted user identity/profile rows.
- Server may compute and return sample plan payload; no durable user-linked writes allowed.
- Client stores sample plan locally with device-scoped key.

### 6.3 Local retention contract
- Retain sample plan locally for 7 days from generation timestamp.
- On app return within window, surface “Save your plan?” restore flow.
- Discard automatically after 7 days.
- If user creates account and confirms restore, plan is regenerated/converted into persisted `draft` under authenticated user, with explicit consent.

---

## 7) Generator templates and algorithm (PRD v0.6 aligned)

### 7.1 Split and template selection
- Apply PRD v0.6 §8.4 split selection rules exactly.
- Use single active equipment profile for all days (no weekday resolution).

### 7.2 Slot fill and validation (PRD v0.6 §8.5)
- Cross-slot exclusion: same exercise cannot appear more than once in one workout day.
- Weekly primary compound cap: same primary compound appears in at most two slots across the week.
- A/B variety enforcement for repeating day patterns.
- Unfillable slot fallback order:
  1. relax disliked penalty
  2. relax preferred boost
  3. drop slot **with explicit validation warning**
  - Silent slot dropping is forbidden.
- Validate movement coverage and direct-set targets.
- Record generation warnings in plan validation payload.

### 7.3 Duration estimate and accessory cut
- Duration formula:
  - `est_duration = Σ[ working_sets × (tempo_seconds × target_reps + rest_seconds) + warmup_overhead + setup_overhead ]`
- If estimated duration exceeds `target_duration × 1.15`, apply PRD accessory-cut policy.
- Accessory-cut constraints:
  - follow PRD priority order exactly
  - protect focus-muscle minimum intent
  - never cut the slot’s primary compound.

### 7.4 Calibration gate
- Before ship: run calibration across 20 representative profiles.
- Pass threshold: >=85% coach-review pass rate.
- Failing threshold blocks implementation release.

---

## 8) Pain/soreness suppression and recommendation logic

### 8.1 Pain-location suppression mapping
- Implement PRD §7.3.2 mapping from pain locations to affected muscle groups/movement intents.
- Mapping must drive:
  - progression suppression
  - substitution suggestions
  - recommendation reasons
  - acceptance tests.

### 8.2 Soreness/pain safety outcomes
- `sorenessLevel=high` for a muscle group should trigger hold recommendation for next session involving that group.
- Same pain location across 2+ sessions surfaces: “Consider professional guidance.”
- No diagnosis language or medical claims.

### 8.3 PRD §9 recommendation precedence
- Exactly one recommendation per exercise per session.
- Precedence chain:
  1. pain/safety
  2. reduce
  3. hold
  4. increase
  5. default hold/add reps guidance
- `add load` allowed only when all are true:
  - all planned working sets completed
  - top-rep condition satisfied (noise-tolerant)
  - no pain flag
  - fatigue not high
  - no form breakdown.
- Implement explicit hold/reduce thresholds per PRD.
- Deload logic:
  - suggest deload when >=3 of 5 PRD signals in rolling 7-day window
  - never auto-apply
  - user approval required.
- Full “Why?” rationale must always be visible to user.

---

## 9) Recommendation APIs (MVP)

- `POST /recommendations/{id}/accept`
- `POST /recommendations/{id}/ignore`
- `POST /recommendations/{id}/dismiss`

Contract requirements:
- Ignore/dismiss accept optional:
  - `ignoredReason` enum
  - free-text `otherReasonText` when reason is `other` (private; excluded from analytics)
- Response returns updated recommendation:
  - `status`
  - version/concurrency token
  - updated timestamps.

---

## 10) Missed-workout recovery and regeneration safety

### 10.1 Missed workout detection
- Missed session definition:
  - `scheduled_for_date < local_today`
  - and `status = not_started`.

### 10.2 Today card actions
- Must support:
  - move to next available day
  - skip and continue plan
  - mark completed outside app
  - regenerate rest of week.

### 10.3 Guardrails
- Never double future volume.
- If 3+ misses in 14 days, surface “Consider fewer training days?” prompt.

### 10.4 Regeneration/history immutability
- In-progress sessions never rebound.
- Same-day in-progress sessions never rebound.
- Future not-started sessions may be rebound/regenerated only when safe and not draft-protected.
- Completed/partial/abandoned/skipped/completed_outside_app immutable.
- Workout history and SetLogs are never overwritten.

---

## 11) Offline/local draft sync alignment

- PRD term is `clientMutationId`.
- If API exposes `mutationId`, it is explicitly an alias of `clientMutationId`.
- Local draft model includes:
  - `localDraftId`
  - `planVersionId`
  - `workoutDayId`
  - `workoutSessionId` (after first sync)
  - `startedAt`
  - completed set logs with `clientMutationId`
  - skipped states
  - notes
  - `lastSavedAt`
  - `syncStatus`
- Preserve idempotency and conflict handling while maintaining append-only SetLog semantics.

---

## 12) Notification preference contract (MVP)

Included:
- workout reminders
- missed-workout follow-up
- recommendation pending review
- optional streak milestone (if retained in MVP scope)

Deferred:
- weekly review notification fields (v1.1)

Global constraints:
- all notifications honor user timezone and quiet hours.

---

## 13) Security, ownership, and RLS contract

- Remove RLS/policy references for deferred nutrition and per-day equipment tables.
- Keep service-only writes where required for generated structures (plan tree/session scaffolding/log pipelines per architecture).
- Enforce user ownership on:
  - workout notes
  - pain notes
  - post-workout check-ins
  - recommendations
- Cross-user FK linking must be rejected at DB boundary (ownership-chain constraints/triggers).
- Sensitive free-text fields (pain notes, ignored “other” reason) excluded from analytics pipelines.

---

## 14) Acceptance test contract (reconciled)

### 14.1 Removed tests
- Nutrition entities/APIs/RLS tests.
- Per-day equipment calendar tests.
- Fixed exactly-50 exercise seed tests.

### 14.2 Required tests
1. No nutrition MVP tables/APIs exist.
2. No per-day equipment MVP behavior exists.
3. Single active equipment profile drives all training days.
4. Draft plan generation + explicit accept/promotion workflow.
5. Sample-plan local restore/discard at 7-day boundary.
6. Exercise seed schema completeness + pre-build exact-count lock check.
7. Cross-slot exercise exclusion per day.
8. Weekly primary compound repeat limit (<=2).
9. A/B variety enforcement.
10. Unfillable-slot fallback and explicit warning emission.
11. Timing-based duration estimate formula correctness.
12. Accessory-cut priority with focus-muscle protection and primary-compound protection.
13. Pain-location suppression mapping behavior.
14. High soreness -> hold recommendation behavior.
15. Noise-tolerant load-increase gate behavior.
16. Deload threshold rule (>=3/5 signals within 7 days).
17. Recommendation ignore/dismiss with `ignoredReason` handling.
18. In-progress sessions never rebound across regeneration.
19. Completed history and SetLogs never overwritten.
20. RLS/ownership/cross-user FK rejection behavior.

---

## 15) Open decisions blocking implementation

1. **Exact exercise catalog count lock** (target ~80) and final list ownership sign-off before Build Week 1.
2. **Movement-intent representation choice**: single vs multi-intent field shape, plus finalized non-lossy mapping artifact if granular internal enum is retained.
3. **PRD §7.3.2 mapping table artifact**: canonical machine-readable mapping file and reviewer sign-off.
4. **Accessory-cut priority order artifact**: explicit ordered rules text copied from PRD v0.6 into implementation checklist (to avoid interpretation drift).
5. **Noise-tolerant top-rep condition constants**: finalize numeric tolerance bands and edge-case handling from PRD §9.
6. **Deload 5-signal definitions**: finalize exact signal list and thresholds as machine-testable constants.
7. **Anonymous sample-plan endpoint operational constraints**: rate limits, abuse controls, and retention telemetry boundaries.

Until these are resolved and approved with this reconciliation, implementation remains blocked.

---

## 16) Readiness checklist (must pass before implementation)

- [ ] Contract source-of-truth references only PRD v0.6.
- [ ] PRD §0 pre-build gates passed and documented.
- [ ] Nutrition and per-day equipment are fully deferred from MVP contract scope.
- [ ] PlanVersion lifecycle (`draft|active|archived|reverted`) approved.
- [ ] Sample plan pre-account flow + 7-day local retention approved.
- [ ] Generator algorithm aligned to PRD §§8.3/8.4/8.5 with explicit warning behavior.
- [ ] Duration formula + accessory-cut protections approved.
- [ ] Post-workout check-in schema aligned to pain/soreness model.
- [ ] Pain-location suppression mapping and recommendation precedence approved.
- [ ] Recommendation schema/APIs/status lifecycle approved.
- [ ] Missed-workout recovery and regeneration immutability safeguards approved.
- [ ] Offline sync naming (`clientMutationId`/`mutationId` alias) approved.
- [ ] Notification preferences trimmed to MVP scope and quiet-hours/timezone rules.
- [ ] Security/RLS ownership and cross-user FK protections approved.
- [ ] Acceptance test matrix updated to reconciled MVP scope.
- [ ] All open blocking decisions in §15 closed.

**Implementation readiness:** **NOT READY** until every checklist item above is approved and all PRD v0.6 blockers are resolved.
