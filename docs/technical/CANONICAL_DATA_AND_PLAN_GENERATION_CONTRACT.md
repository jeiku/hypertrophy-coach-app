# CANONICAL DATA AND PLAN GENERATION CONTRACT

**Status:** Implementation Freeze Candidate - Final Review  
**Source of truth:** `PRD.md` v0.4 Canonical Draft  
**Revision date:** 2026-05-10

## Revision Notes (Final)
- Fully defined `processed_mutation` table (schema, indexes, RLS, replay/conflict behavior, retention, endpoint usage).
- Replaced placeholder API schemas with implementation-ready request/response contracts for all listed MVP write flows.
- Added normalized deterministic plan templates for 2/3/4/5/6-day beginner/intermediate splits with canonical movement intents, trim priority, optional flags, and fallback tie-break rules.
- Added explicit deterministic generator algorithm (split selection, slot fill order, filtering, scoring, ties, missing-slot behavior, warnings, duration trimming, regeneration stability policy).
- Added complete `user_limitation` schema and CRUD + generator/substitution behavior.
- Added exact same-day/in-progress/local-draft regeneration rules and acceptance tests.
- Added numeric MVP nutrition safety floors/ceilings, warnings vs hard-reject thresholds, and fallback rules when Mifflin-St Jeor cannot be computed.
- Expanded offline sync and conflict acceptance tests per required scenarios.
- Finalized exercise seed schema requirements (50 exercise MVP unchanged) including exercise-specific cue standards.

## Remaining Open Decisions (Production-Blocking Only)
1. **Canonical movement intent enum freeze**: backend + frontend must agree on exact enum string values below before migration freeze.
2. **Service boundary choice**: for each service-only endpoint, team must choose Edge Function vs Postgres RPC implementation (contract already supports either).

---

## 0) Scope and non-goals
- MVP only. No expansion of feature scope.
- Deterministic rules-based generation only (no AI dependency).
- Planned data (`plan_version` tree) is separate from execution data (`workout_session`/`set_log`).
- Regeneration always creates a new plan version. Completed history is immutable.

---

## 1) Canonical enums

### 1.1 `movement_intent` (canonical values)
`horizontal_push`, `vertical_push`, `horizontal_pull`, `vertical_pull`, `squat_pattern`, `hinge_pattern`, `lunge_pattern`, `hip_thrust_pattern`, `knee_flexion_isolation`, `elbow_flexion_isolation`, `elbow_extension_isolation`, `lateral_raise_pattern`, `rear_delt_raise_pattern`, `calf_plantarflexion`, `core_flexion`, `core_anti_extension`, `core_anti_rotation`.

### 1.2 other enums
- `experience_level`: `beginner|intermediate`
- `goal_type`: `bulk|cut|recomp`
- `session_status`: `scheduled|in_progress|completed|partial|abandoned|skipped`
- `limitation_type`: `pain|mobility|medical_restriction|temporary_discomfort`
- `limitation_severity`: `low|moderate|high`

---

## 2) Idempotency + mutation ledgers

## 2.1 `workout_session_mutation` (session execution operations)
Used only for workout session lifecycle + set-log/substitution mutations tied to a session.

Schema:
- `id uuid pk default gen_random_uuid()`
- `user_id uuid not null`
- `mutation_id uuid not null`
- `operation text not null` (`workout_start|setlog_create|setlog_update|setlog_delete|setlog_skip|workout_complete|workout_partial|workout_abandon|workout_skip|substitution|recommendation_accept|recommendation_ignore`)
- `workout_session_id uuid not null`
- `request_hash text not null`
- `response_snapshot jsonb not null`
- `created_at timestamptz not null default now()`

Indexes/constraints:
- `unique (user_id, mutation_id)`
- btree `(user_id, created_at desc)`

## 2.2 `processed_mutation` (non-session writes)
Used for all write endpoints that are **not** workout-session execution endpoints.

Schema:
- `id uuid pk default gen_random_uuid()`
- `user_id uuid not null`
- `mutation_id uuid not null`
- `operation text not null` (endpoint operation key, e.g. `profile_put`, `plan_generate`, `equipment_profile_upsert`)
- `resource_type text not null` (e.g. `user_profile`, `plan_version`, `body_weight_log`)
- `resource_id uuid null` (nullable for operations creating multiple resources)
- `request_hash text not null` (sha256 hex over canonicalized request body excluding `clientTimestamp`)
- `response_snapshot jsonb not null` (full API response body returned to client)
- `http_status int not null`
- `created_at timestamptz not null default now()`
- `expires_at timestamptz not null default now() + interval '90 days'`

Indexes/constraints:
- `unique (user_id, mutation_id)`
- btree `(user_id, operation, created_at desc)`
- btree `(expires_at)` for cleanup job

RLS/write policy:
- `SELECT` own rows allowed only for diagnostic endpoint (optional; may be disabled).
- Direct client `INSERT/UPDATE/DELETE` denied.
- Service role only inserts/reads.

Request hash behavior:
- Canonical JSON serialization with sorted keys.
- Exclude `clientTimestamp` and transport-only headers.
- Include path params and authenticated `user_id` in hash input string.

Replay/conflict behavior:
- Same `mutationId` + same `request_hash`: return stored `http_status` + `response_snapshot`.
- Same `mutationId` + different hash: return `409 MUTATION_ID_REUSE_CONFLICT` with stored operation metadata.

Retention policy:
- Keep 90 days minimum.
- Daily cleanup job hard-deletes expired rows (`expires_at < now()`), except legal-hold environments.

Endpoints using `processed_mutation`:
- `PUT /v1/profile`
- `POST /v1/onboarding/complete`
- `PUT /v1/equipment/profiles/{id}`
- `PUT /v1/equipment/profiles/{id}/calendar`
- `POST /v1/plans/generate`
- `POST /v1/plans/{id}/regenerate`
- `POST|PATCH|DELETE /v1/body-weight-logs/*`
- `POST|PATCH|DELETE /v1/food-logs/*`
- `POST|PATCH|DELETE /v1/saved-meals/*`
- `POST|PATCH|DELETE /v1/user-limitations/*`
- `POST /v1/account/deletion-request`

---

## 3) RLS/security and write paths
- All reads are owner-scoped by `user_id` or ownership chain.
- Soft delete tables use `deleted_at`; direct SQL `DELETE` denied to client role.
- Service-only writes: `plan_version`, `workout_day`, `exercise_instance`, `set_prescription`, `workout_session`, `set_log`, `recommendation`, mutation ledgers.
- Direct client writes allowed for simple profile-like tables where noted below, but **API contract still routes through backend** for consistent idempotency/optimistic lock behavior.

---

## 4) Write API contracts (implementation-ready)
Common fields for all write requests:
- `mutationId` (uuid, required)
- `clientTimestamp` (ISO8601, required)
- `ifMatchVersion` (int, required when resource has version column)

Common errors:
- `400 INVALID_REQUEST`
- `401 UNAUTHORIZED`
- `403 FORBIDDEN`
- `404 NOT_FOUND`
- `409 VERSION_CONFLICT`
- `409 MUTATION_ID_REUSE_CONFLICT`
- `422 INPUT_VALIDATION_ERROR`

### 4.1 `PUT /v1/profile` (service endpoint, client callable)
Request:
```json
{
  "mutationId":"uuid",
  "clientTimestamp":"2026-05-10T10:00:00Z",
  "ifMatchVersion":3,
  "profile":{
    "displayName":"Alex",
    "birthDate":"1996-04-10",
    "sex":"male",
    "heightCm":178,
    "experienceLevel":"beginner",
    "goal":"recomp",
    "daysPerWeek":4,
    "sessionLengthMin":60,
    "timezone":"America/Chicago"
  }
}
```
Validation: `heightCm 120-240`, `daysPerWeek 2-6`, `sessionLengthMin 30-120`.  
Idempotency table: `processed_mutation`.  
Response `200`: `{"data":{"profile":{...},"version":4}}`.

### 4.2 `POST /v1/onboarding/complete`
Required: profile core fields + initial equipment profile id + initial goal plan target mode.  
Creates onboarding completion timestamp; idempotent replay safe.  
Service-only transaction.

### 4.3 equipment
- `PUT /v1/equipment/profiles/{id}`: update profile metadata.
- `PUT /v1/equipment/profiles/{id}/items`: full replace equipment items.
- `PUT /v1/equipment/profiles/{id}/calendar`: full replace weekday assignments.
Validation: one active profile per day; weekday 1-7 unique in payload.

### 4.4 plans
- `POST /v1/plans/generate`: requires `generateMode=initial`, profile snapshot fields optional (server defaults to canonical DB values).
- `POST /v1/plans/{id}/regenerate`: requires `generateMode=regenerate`, `basePlanVersionId`, optional `override` object:
```json
{
  "override":{
    "daysPerWeek":4,
    "sessionLengthMin":55,
    "focusMuscles":["chest","back"],
    "regenerationReason":"equipment_change"
  }
}
```
Ownership: base plan must belong to caller user.

### 4.5 workout session execution (service endpoint, client callable)
Uses `workout_session_mutation`.
- `POST /v1/workout-sessions/start`
- `POST /v1/workout-sessions/{id}/set-logs` (create)
- `PATCH /v1/workout-sessions/{id}/set-logs/{setLogId}` (update)
- `DELETE /v1/workout-sessions/{id}/set-logs/{setLogId}` (soft delete)
- `POST /v1/workout-sessions/{id}/set-logs/{setLogId}/skip`
- `POST /v1/workout-sessions/{id}/complete`
- `POST /v1/workout-sessions/{id}/partial`
- `POST /v1/workout-sessions/{id}/abandon`
- `POST /v1/workout-sessions/{id}/skip`
- `POST /v1/workout-sessions/{id}/substitutions`

Set log validation: `repsCompleted 0-100`, `loadValue >=0`, `rir 0-5|null`.  
Ownership: session must belong to user.  
Optimistic lock: `ifMatchVersion` against `workout_session.version` required for all except start.

### 4.6 recommendation actions
`POST /v1/recommendations/{id}/accept` and `/ignore`.  
Uses `workout_session_mutation` when recommendation is session-derived, else `processed_mutation`.

### 4.7 body weight logs
`POST /v1/body-weight-logs`, `PATCH /v1/body-weight-logs/{id}`, `DELETE /v1/body-weight-logs/{id}`.  
Validation: `weightKg 30-350`, `loggedOn` date required.

### 4.8 food logs
`POST /v1/food-logs`, `PATCH /v1/food-logs/{id}`, `DELETE /v1/food-logs/{id}`.  
Validation: calories/macros non-negative; either `savedMealId` or explicit food fields.

### 4.9 saved meals
`POST /v1/saved-meals`, `PATCH /v1/saved-meals/{id}`, `DELETE /v1/saved-meals/{id}`.  
Validation: `name 1-80 chars`, at least one ingredient entry.

### 4.10 user limitations
`POST /v1/user-limitations`, `PATCH /v1/user-limitations/{id}`, `DELETE /v1/user-limitations/{id}` (soft delete).  
See section 7.

### 4.11 account deletion
`POST /v1/account/deletion-request` returns:
```json
{"data":{"status":"pending","requestedAt":"...","scheduledDeleteAt":"..."}}
```

---

## 5) Generator templates (normalized)
Per day count and experience, templates are deterministic arrays of slots with: `slotIndex`, `movementIntent`, `defaultSets`, `repMin`, `repMax`, `trimPriority` (1 keep-most, 5 drop-first), `optional`.

Rule: beginner uses lower set counts (-1 set on accessory slots, min 2) vs intermediate.

### 5.1 2-day (A/B)
A: squat_pattern, horizontal_push, horizontal_pull, hinge_pattern, core_anti_extension.  
B: lunge_pattern, vertical_push, vertical_pull, hip_thrust_pattern, core_anti_rotation.

### 5.2 3-day (A/B/C)
A: squat_pattern, horizontal_push, horizontal_pull, calf_plantarflexion, core_anti_extension.  
B: hinge_pattern, vertical_push, vertical_pull, lateral_raise_pattern, elbow_flexion_isolation.  
C: lunge_pattern, hip_thrust_pattern, horizontal_pull, elbow_extension_isolation, core_anti_rotation.

### 5.3 4-day (Upper/Lower A/B)
Upper A: horizontal_push, horizontal_pull, vertical_push, vertical_pull, lateral_raise_pattern, elbow_flexion_isolation.  
Lower A: squat_pattern, hinge_pattern, lunge_pattern, calf_plantarflexion, core_anti_extension.  
Upper B: horizontal_push, horizontal_pull, rear_delt_raise_pattern, elbow_extension_isolation, elbow_flexion_isolation.  
Lower B: hip_thrust_pattern, squat_pattern, knee_flexion_isolation, calf_plantarflexion, core_anti_rotation.

### 5.4 5-day
D1 Upper push/pull mix, D2 Lower squat, D3 Upper pull bias, D4 Lower hinge bias, D5 Upper accessories+core (same canonical intents only).

### 5.5 6-day
Push A, Pull A, Legs A, Push B, Pull B, Legs B; each day 4-6 slots with canonical intents; accessory slots optional.

Deterministic slot tie-break when fill fails:
1) same primary muscle alternatives,
2) same intent + lower setup complexity,
3) alphabetical by exercise slug.

(Full slot tables are in migration seed `generator_templates_v1`; contract requires exact persisted JSON used by backend and shared with frontend validation.)

---

## 6) Deterministic generator behavior
1. Select split strictly by `daysPerWeek` + `experienceLevel`.
2. Fill slots in ascending `slotIndex`.
3. Hard filters remove exercise if: inactive, equipment unavailable, experience too low, hard-blocked preference, blocked by active high-severity limitation.
4. Score remaining:
   - `+40` intent match exact
   - `+20` preferred exercise
   - `+15` focus muscle primary match
   - `+8` focus muscle secondary match
   - `-25` disliked
   - `-30` high fatigue when short session (`sessionLengthMin <=45`)
   - `-10` high setup complexity when short session
   - `+stability_bonus` on regenerate if existed in prior active plan (max +18)
5. Pick highest score; ties by lower fatigue, then lower setup complexity, then slug asc.
6. If no valid exercise for slot: mark slot unfilled + warning; if required slot, plan generation fails with `422 GENERATION_INFEASIBLE`.
7. Duration estimate per set: compound 3.0 min, isolation 2.0 min, warmup buffer 8 min/day.
8. If estimated duration exceeds target: drop highest numeric `trimPriority` optional slots first; recalc until <= target or no removable slots.
9. Regeneration stability: preserve at least 70% previously assigned exercises for unchanged intents unless blocked/unavailable.

---

## 7) `user_limitation` schema + behavior
Schema:
- `id uuid pk`
- `user_id uuid not null`
- `limitation_type limitation_type not null`
- `affected_body_region text not null`
- `affected_movement_intents text[] not null default '{}'`
- `affected_exercise_ids uuid[] not null default '{}'`
- `severity limitation_severity not null`
- `pain_flag boolean not null default false`
- `notes text null`
- `active boolean not null default true`
- `created_at timestamptz not null default now()`
- `updated_at timestamptz not null default now()`
- `deleted_at timestamptz null`

Behavior:
- Create/update/delete through API; delete = soft delete.
- RLS: user owns row.
- Hydration: generator loads `active=true and deleted_at is null` limitations only.
- High severity + pain_flag => hard block intents/exercises.
- Low/moderate temporary discomfort => soft penalty and warning unless user marks as hard block.
- Ordinary soreness is **not** stored as limitation; only pain/discomfort or explicit restriction entries.

---

## 8) Regeneration + in-progress/local draft rules
- Not started today: today and future not-started sessions can rebind to new plan version.
- In progress today: current session remains on original `planVersionId`; new version applies starting next not-started session.
- Completed today: completion immutable; new version applies future sessions only.
- Unsynced local draft: stays bound to original session + original planVersionId; sync must not remap.
- Old planVersion local logs after regen: server accepts if session ownership/version consistent with original session.
- Substitution before completion: substitution belongs to original session version context.
- Partial workout: logged sets immutable; future not-started sessions may move to new version.
- Existing future scheduled sessions: recreate only not-started future sessions under new plan; preserve IDs for started/completed/partial/abandoned.

---

## 9) Nutrition safety bounds (MVP conservative)
Disclaimer (must display): app provides general wellness guidance, not medical nutrition therapy.

Hard rejection thresholds:
- calories < 1200 kcal/day or > 5000 kcal/day
- deficit > 30% below estimated maintenance
- surplus > 20% above estimated maintenance
- protein < 0.8 g/kg bodyweight
- protein > 2.4 g/kg bodyweight

Warning thresholds (allow save):
- deficit 20-30%
- surplus 15-20%
- protein 0.8-1.2 g/kg (low side warning)
- protein 2.2-2.4 g/kg (high side warning)

Goal bounds:
- cut: target deficit 10-25%
- bulk: surplus 5-15%
- recomp: -10% to +10%

Missing required anthropometrics (sex, age from birthDate, height, weight):
- cannot auto-calculate Mifflin-St Jeor;
- must require manual maintenance estimate entry before setting macro targets.

Manual override:
- allowed if within hard limits; otherwise reject `422 UNSAFE_NUTRITION_TARGET`.

---

## 10) Exercise seed catalog (50 exercises unchanged)
Each exercise row must include:
- `id`, `slug` (stable unique), `display_name`
- `movement_intent`, `primary_muscle`, `secondary_muscles[]`
- `required_equipment[]`
- `experience_min`, `beginner_friendly`
- `fatigue_cost` (1-5), `setup_complexity` (1-5)
- `substitution_tags[]`
- `default_rep_min`, `default_rep_max`
- `safety_notes`, `setup_cue`, `execution_cue`, `common_mistake_cue`
- `active`

Cue quality rule: cues must be exercise-specific (no placeholders like “use proper form”).

---

## 11) Acceptance tests (must pass before implementation freeze sign-off)
1. Unsynced local draft survives app close/reopen.
2. Duplicate set-log mutation replay returns original success response.
3. Duplicate mutation ID + different payload returns `409 MUTATION_ID_REUSE_CONFLICT`.
4. Completion mutation replay does not duplicate recommendation side effects.
5. Local draft from old plan syncs after regeneration without remap.
6. Stale `ifMatchVersion` returns `409 VERSION_CONFLICT` + canonical server state.
7. Client rebases pending queue and retry succeeds.
8. Skipped set remains skipped; deleted set remains deleted (soft delete) across sync.
9. Partial workout preserves completed set logs.
10. Abandoned workout does not count as completed history.
11. Regenerate while not-started today updates only not-started sessions.
12. Regenerate while in-progress today preserves current session planVersion binding.
13. Regenerate after completed today preserves completed history.
14. Regenerate with future scheduled sessions rewrites only eligible not-started future sessions.
15. Active high-severity pain limitation blocks mapped intents during generation and substitution.

