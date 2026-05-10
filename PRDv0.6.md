# Hypertrophy & Muscle Gain Coach
## Product Requirements Document — v0.6

**Status:** Build-ready *after* §0 pre-build gates pass. Until then, this is a hypothesis document.
**Primary audience:** Product owner, designer, engineer, technical reviewer
**Primary user:** Beginner hypertrophy learner (intermediates are secondary; see §2)
**MVP platform:** Mobile-first React Native / Expo, Supabase/Postgres backend
**Honest scope estimate:** Core-1 is a 3-month build for a 2–3 person team. Anything that doesn't fit that envelope gets cut, not deferred.

**What changed from v0.5 (high level):**
- Pre-build research gates promoted to §0; build is blocked until they pass.
- Primary persona narrowed to beginner; intermediate moved to secondary.
- Nutrition removed from MVP entirely (was Core-2; now v1.1).
- Per-day equipment toggle removed from MVP.
- Exercise seed shrunk from 150–200 to ~80.
- Success metrics restructured: "Why" tap rate is no longer a primary input metric (replaced by composite tap × acceptance + trust survey).
- Algorithm gaps fixed: cross-slot exclusion, unfillable-slot fallback, plan regen during in-flight session, pain-location → muscle-group mapping.
- Progression rules made noise-tolerant; deload rule made explicit.
- Scoring weights, time estimates, and thresholds explicitly marked as v0/calibration-pending with a tuning step.

---

## 0. Pre-Build Gates (NEW)

This PRD is written ahead of validated user research. **No production code is written until all four gates pass.** A two-week research sprint runs first.

| Gate | Method | Pass criterion |
|---|---|---|
| G1: Wedge validation | 30+ semi-structured interviews with self-identified beginner lifters who have tried Fitbod/Strong/Hevy/RP and churned or stalled. | ≥60% spontaneously cite "I don't understand why the app is telling me to do this" or equivalent as a frustration, *without prompting*. |
| G2: Competitor teardown | Two researchers each use Fitbod and RP Hypertrophy daily for 30 days. Document every recommendation and tag whether the rationale was discoverable. | Written teardown delivered. The wedge claim in §2.4 is rewritten to match what the teardown actually shows, not what we assumed. |
| G3: Willingness-to-pay | Survey N≥150 of the target persona. Test $9.99/mo vs $14.99/mo, with and without the explainability promise. | ≥15% indicate they would pay at $9.99/mo with the explainability framing, and the framing produces a measurable lift over the no-framing control. |
| G4: Day-by-day equipment | Quantitative question on the same survey: "In a typical week, do you train at multiple locations (home, gym, travel)?" | Used to confirm or kill the per-day equipment feature for v1.1. (It's already cut from MVP; this gate decides whether to add it later.) |

**If G1 fails:** the explainability wedge is wrong. Stop. Rewrite §2 before continuing.
**If G2 contradicts §2.4:** rewrite §2.4 to reflect reality. Do not ship a positioning that doesn't survive 30 days of using the actual competitors.
**If G3 fails at $9.99/mo:** product economics don't support this scope. Cut further.

This is not a checkbox. The product owner signs off on each gate in writing before build begins.

---

## 1. Product Summary

A mobile-first training app for beginners who want to build muscle but do not want to design their own programs. The app generates a weekly plan from the user's schedule, equipment, and experience, walks them through workouts, and recommends progression — and **explains every recommendation in plain language**.

**Primary promise:** Know what to do today, understand *why* you're doing it, and progress without spreadsheets.

Nutrition is not in MVP. We're not in the calorie-counting business until the training loop is validated.

---

## 2. Target User & Positioning

### 2.1 Primary persona: Beginner hypertrophy learner

Has lifted for 0–18 months, inconsistently. Doesn't know what RIR or volume landmarks are. Wants structure, not theory. Wants to learn while training, not before.

This is the *only* persona we optimize for in v1.

### 2.2 Secondary persona: Intermediate

We don't break the app for intermediates, but we don't tune for them either. Hidden RIR field, unlocked volume cap, ability to see scoring rationale — that's the extent of intermediate accommodation in MVP.

### 2.3 Out of scope

Advanced lifters, periodization geeks, strength sport athletes, rehab patients.

### 2.4 Positioning (subject to G2 rewrite)

**Working hypothesis:** Existing competitors ship adaptive workouts but treat their decision logic as a black box, and beginners either trust the app or fight it. Our differentiator is that *every recommendation comes with a one-tap plain-language reason*.

**The moat is not technical** — competitors could add a "Why?" expander in a sprint. The moat is **trust compounded over time**: a user who learned *why* their last 12 weeks of programming worked has invested in our explanation system in a way they can't easily port to a competitor. Defensibility comes from explanation quality + the corpus of personalized reasons accumulated for each user, not from rules-vs-ML architecture.

### 2.5 Positioning statement

> For beginner lifters who want to build muscle but don't want to design their own program, this app provides a guided daily workout where every recommendation — what to do, what to lift, when to progress — comes with a one-tap plain-language explanation, so users learn while they train.

---

## 3. Goals, Success Metrics & Business Model

### 3.1 MVP product goals

- Help users complete workouts consistently.
- Help users understand what to progress next, and *why*.
- Validate that explainable rule-based recommendations build trust and adherence — measured, not assumed.

### 3.2 Success metrics

**North Star**

- Median workouts completed per active user per week, week 4: **≥ 2.5**

**Primary input metric (replaces v0.5's "Why tap rate")**

- **Trust composite:** users who tap "Why?" *and then accept the recommendation* in the same session, divided by total recommendations shown, week 4: **≥ 20%**.
  - Rationale: tap-rate alone conflates curiosity with distrust. Tap-then-accept measures explanation usefulness.

**Secondary input metrics (4-week targets)**

- Workout completion rate (started → completed): ≥ 80%
- Recommendation acceptance rate (shown → accepted): ≥ 60%
- "Why?" tap rate: monitored, no target. Diagnostic only.

**Trust survey (NEW)**

- In-app survey at week 4: "Do you understand why the app made the recommendations it gave you this week?" Likert 1–5.
- Target: median ≥ 4.0.
- This is the explainability validation, not tap rate.

**Retention (4-week targets)**

- Day 7 retention: ≥ 50%
- Day 30 retention: ≥ 25%
- Percent of users completing ≥ 2 workouts/week: ≥ 40%

**Diagnostic (no targets, monitored)**

- Partial-workout rate, substitution rate, missed-workout-recovery action rate, draft-recovery rate, plan-edit rate.

**Kill condition:** If trust-survey median is < 3.0 at week 4 *or* trust composite < 10%, the wedge is not landing. Stop adding scope and revisit §2.

### 3.3 Business model

- 14-day free trial, no credit card. (v0.5 used 7 days; that's too short to see progression patterns. A user needs at least 4–6 logged sessions before "Why hold vs. progress" surfaces, and 7 days only gives them 2–3.)
- Monthly $9.99 / Annual $59.99 (pending G3).
- No ads, no marketplace, no in-app purchases for content.

---

## 4. Competitive Context

### 4.1 Categories

- General adaptive: Fitbod
- Hypertrophy-specific: RP Hypertrophy
- Manual trackers: Strong, Hevy
- Spreadsheets / templates

(Nutrition trackers omitted because we don't compete with them in MVP.)

### 4.2 What we will not match

| Competitor strength | Their depth | Our position |
|---|---|---|
| Exercise library | Fitbod ~600 | We ship ~80 (see §8.11) |
| Advanced periodization | RP Hypertrophy: full mesocycle design | We do not |
| Strict food database | MacroFactor / MFP: millions of foods | Not in MVP at all |

These are conscious cuts. If users churn citing "not enough exercises," we know to add more before adding new feature surface area.

### 4.3 Honest competitive risk

The "Why?" expander is easy to copy. If a competitor ships it within 90 days of our launch, our moat is whatever trust we built in those 90 days. Plan accordingly: explanation quality has to be excellent at v1, not adequate.

---

## 5. MVP Scope

### 5.0 What this MVP is and is not (NEW)

**Is:**
- A training-only app.
- Beginner-first, intermediate-tolerated.
- Single equipment profile (one set of equipment, applies to every day).
- ~80 curated exercises.
- Deterministic rule-based generator and progression.
- Designed to be built in ~3 months by a 2–3 person team.

**Is not:**
- A nutrition tracker.
- A mesocycle programmer.
- A multi-location app.
- A motivation/social app.
- An AI coach.

If any of those is essential to a stakeholder, raise it now, not after build.

### 5.1 MVP Core (formerly Core-1; the only "Core" in v0.6)

1. Account creation / login
2. Onboarding (training inputs only)
3. Weekly plan generation from schedule + equipment
4. Today dashboard
5. Workout player with written setup/execution/safety cues
6. Durable set logging with offline draft preservation
7. Rules-based progression with **one-tap "Why?" explanations**
8. Basic exercise substitutions *with "Why this substitution?"* — moved up from v0.5 Core-2 because the Today/Player UX in §6.5 commits to it
9. Basic privacy / account controls
10. Missed-workout recovery (move / skip / regenerate-rest-of-week) — moved up from v0.5 Core-2 because the validation cohort *will* miss workouts

§§5–9 above are the smallest scope that delivers the §2.5 promise. Anything smaller doesn't make the promise; anything larger doesn't ship in 3 months.

### 5.2 v1.1 (after MVP validation)

- Lite nutrition (calorie + protein targets, saved meals, common foods, visual portion guides)
- Weekly review tab
- Per-day equipment profile (gated on §0 G4 result)
- Recommendation modify flow
- Detailed muscle-volume summaries
- Rest timer
- Warm-up set marker
- Localization
- Tablet-optimized layout

### 5.3 Explicitly not in MVP or v1.1

- AI chat coach
- Autonomous AI plan changes
- Strict food database / barcode scanning
- Wearable integrations
- Computer vision / form checks
- Photo storage or body comparisons
- Advanced periodization
- Coach marketplace, social feed
- Exercise video library (written cues only)
- Medical / rehab content

### 5.4 Exercise instruction MVP requirement

Each of the ~80 seeded exercises ships with:
- Setup cue
- Execution cue
- Common-mistake cue
- Safety note (when relevant)

Roughly 320 strings authored by a credentialed strength coach and reviewed by a second reviewer for safety. (See §5.5.)

### 5.5 Content workstream (NEW)

This is a real workstream, not a side task.

- **Author:** Contracted strength coach (CSCS or equivalent), ~40 hours over 4 weeks.
- **Reviewer:** Second coach reviews for safety, beginner-readability, accuracy.
- **Editor:** Product writer brings them into the app's plain-language voice.
- **Owner:** Product owner. Tracked alongside engineering.

If we don't have an author lined up before build week 1, build week 1 doesn't start.

---

## 6. User Experience Requirements

### 6.1 Navigation

Three tabs in MVP:

1. **Today**
2. **Plan**
3. **Profile** (settings, account)

(v0.5's "Progress" tab folds into Plan and into post-MVP weekly review.)

### 6.2 Onboarding

7 screens; sample plan generated *before* account creation to anchor commitment.

1. Welcome + goal + experience (combined)
2. Schedule and session length
3. Equipment profile (single, applies to all days)
4. Preferences / limitations
5. (Nutrition mode screen removed)
6. Generate sample plan and review
7. Account creation and save plan

If the user abandons after step 6, see §6.6.

### 6.3 Onboarding inputs

**Required:**
- Experience (beginner / intermediate)
- Goal (bulk / cut / recomp) — kept for future nutrition use; affects *nothing* in MVP generator
- Weight, height, age (for v1.1 nutrition; collected once, not used in MVP rules)
- Sex (optional in MVP since calorie targets are out)
- Days/week (2–6)
- Preferred days
- Session length target
- **Equipment profile (one)**

**Optional:**
- Preferred / disliked exercises
- Focus muscles
- Movement limitations / pain history

**Cut from v0.5:**
- "My equipment varies by day" toggle (now v1.1, gated on G4)
- Body composition estimates / waist-neck-hip
- Physique photos

### 6.4 Today dashboard

States:
- No plan yet
- Training day, not started
- In progress
- Completed
- Rest day
- Missed
- Offline draft exists

Today shows scheduled workout (or rest state), start/resume action, and the **next-best-action coach card with "Why?" expander**. The "Why?" reveals plain-language reasoning derived from §9 rules.

**Coach-card prioritization (NEW — v0.5 was silent):** Pain-flagged recommendation > Missed-workout recovery > Today's workout > Progression notice. Only one card shows; secondary cards live under a "More" affordance.

### 6.5 Workout player

- Workout overview
- Exercise list with cues (§5.4)
- Planned sets and target rep range
- Previous performance if available
- Log weight + reps
- RIR field hidden by default; user toggles in Profile
- Skip set / exercise
- Add set
- **Substitute exercise with "Why this substitution?"** (per §5.1 #8)
- Finish as completed or partial
- Post-workout check-in (§7.3)

### 6.6 First-workout UX (NEW)

The first time a user starts a workout from a brand-new plan:
- A one-screen welcome with the "Why this plan?" plan-level explanation.
- "Tap any exercise to see how to do it" hint.
- "We'll suggest progression after this session."
- The hint dismisses on first set logged. It does not show again.

### 6.7 Onboarding-abandonment behavior (NEW)

If the user generates a sample plan in step 6 and closes the app without account creation:
- Plan is held in local storage for 7 days, keyed to device.
- On next open within 7 days, "Save your plan?" prompt restores it.
- After 7 days the local plan is discarded.
- We do not collect email at any earlier step.

### 6.8 Accessibility, devices, time zone

- iOS 16+, Android 12+, phone-first.
- Text scaling, WCAG 2.1 AA contrast, screen-reader labels for core actions.
- User time zone stored; reminders local.
- English only.

(Tablet UX, localization, and AAA review are v1.1+.)

---

## 7. Edge Cases & State Handling

### 7.1 Missed workout

Options on the Today card the day after a miss:
- Move to next available day
- Skip and continue plan
- Mark completed outside app
- Regenerate rest of week

Rules:
- Do not shame.
- Do not double future volume.
- 3+ misses in 14 days surfaces "Consider fewer training days?"

### 7.2 Partial workout

A partial workout is valid data. Completed sets count; uncompleted planned sets do not count as failed attempts. Repeated partials due to time surface "Consider shorter sessions?"

### 7.3 Soreness vs pain

Distinguished via `PostWorkoutCheckIn` after every completed workout.

#### 7.3.1 Check-in UX

1. **Soreness slider** (none / mild / moderate / high). Default mild. Skippable.
2. **Soreness location** (NEW) — optional multi-select on simple body map. Same enum as pain location.
3. **Pain flag** ("Did anything hurt during this workout?"). Default off.
4. If pain flag on, reveal:
   - Pain location (multi-select body map)
   - Pain type (sharp / dull-aching / joint / nerve-like / other)
   - Pain notes (optional free text)

#### 7.3.2 Pain-location → muscle-group mapping (NEW; was missing in v0.5)

| Pain location | Muscle groups whose progression is suppressed |
|---|---|
| shoulder | chest, shoulders, back (compound pulls), triceps if overhead-loaded |
| elbow | biceps, triceps, all loaded grip/pull work |
| wrist | all loaded pressing and pulling |
| lower_back | hinge (deadlift, RDL, good morning), squat, loaded standing work |
| upper_back | rows, pull-ups, loaded carries |
| hip | squat, hinge, lunge |
| knee | squat, lunge, leg extension, leg press |
| ankle | squat, lunge, calf, loaded standing |
| other | none auto — surface "review your plan" prompt |

This mapping is what §9.2 step 1 uses. It also determines which exercises trigger substitute suggestions.

#### 7.3.3 Rules

- Soreness `high` for a muscle group → next session for that group: suggest hold.
- Pain flag → suppress progression on exercises in mapped groups (per 7.3.2). Trigger substitute suggestion for affected exercises only.
- Same-location pain across 2+ sessions → "Consider professional guidance" notice. We do not diagnose.

### 7.4 Equipment change

Changing equipment in Profile triggers revalidation of *future* workouts only. Options:
- Substitute incompatible exercises
- Regenerate affected day(s)
- Regenerate rest of week
- Keep plan and warn

**In-flight session rule (NEW; was missing in v0.5):** Any `WorkoutSession` already in `in_progress` state references its original `planVersionId` and is unaffected. Equipment changes apply starting with the next not_started session.

### 7.5 Offline / interrupted workout

#### 7.5.1 Local persistence

After every meaningful action (set logged, exercise skipped, etc.) the workout draft is saved locally. App close or network failure: user can resume; completed set logs are preserved; sync retries when network returns. Completed set logs are never silently discarded.

#### 7.5.2 Conflict resolution

| Entity | Strategy |
|---|---|
| `SetLog` | Append-only, idempotent on `clientMutationId`. Server dedupes; ordering is by `setIndex` then server timestamp. |
| `WorkoutSession` | Server is source of truth for status transitions. Client posts intent; server validates. |
| `PlanVersion` | Per-field LWW. No conflict UI in MVP — multi-device plan editing is rare and we don't pay UX cost for it. (v0.5's 60-second conflict-surfacing UI is deferred to v1.1.) |
| `UserProfile` | Per-field LWW. |

All client mutations carry `clientMutationId` (UUID v4).

---

## 8. Program Generator Requirements

### 8.1 Principle

Deterministic, rules-based, auditable. Every output carries a structured `reason` the UI can render in plain language. AI is not in MVP.

### 8.2 Inputs

Experience, goal (recorded but doesn't affect generator output in MVP), days/week, preferred days, session length, equipment profile (single), preferences, limitations, prior plan if regenerating.

### 8.3 Movement-intent slot taxonomy

1. Horizontal push
2. Vertical push
3. Horizontal pull
4. Vertical pull
5. Hip hinge
6. Knee-dominant squat
7. Lunge / unilateral leg
8. Isolation (sub-tag: biceps / triceps / side delt / rear delt / calf / abs)

### 8.4 Split selection

| Days/week | Split | Slots per session |
|---:|---|---|
| 2 | Full Body A/B | 1 squat + 1 hinge + 1 horizontal push + 1 horizontal pull + 1 vertical (push or pull) + 1 isolation |
| 3 | Full Body A/B/C | Same, rotating which vertical and which knee-dominant pattern lead |
| 4 | Upper / Lower | Upper: H-push, V-push, H-pull, V-pull, 2 isolation. Lower: squat, hinge, lunge, 1 isolation |
| 5 | Upper/Lower + 1 focus | U/L as above; 5th day = 1 compound + 3 isolation for stated focus muscle |
| 6 (beginner) | Full Body x 6 | Conservative — full body A/B/C run twice, reduced per-session volume |
| 6 (intermediate) | PPL x 2 | Standard PPL |

### 8.5 Generator algorithm (with v0.5 gaps closed)

1. Validate required inputs.
2. Select split template (§8.4).
3. Create workout-day templates with movement-intent slots.
4. Set initial weekly volume targets (§8.7).
5. For each slot, filter eligible exercises (§8.8.1).
6. Score eligible exercises (§8.8.3).
7. **Assign with cross-slot exclusion (NEW):** Same exercise cannot appear in more than one slot in the same workout day. Across the week, the same primary compound cannot appear in more than two slots total. If the highest-ranked exercise is excluded by these rules, assign the next-highest-ranked.
8. **Variety enforcement (NEW):** Across A/B sessions in the same week, at least one exercise per movement intent must differ. If the algorithm produces identical A and B days, re-run with the second-highest-ranked exercise for the most-overused intent.
9. **Unfillable slot fallback (NEW):** If after filters and exclusions no candidate has score ≥ 0, relax in this order: (a) ignore "disliked" penalty, (b) ignore "preferred" boost, (c) drop the slot and emit a validation warning ("Couldn't fill horizontal push with your equipment + preferences — consider adding a barbell or removing dislikes for bench press, push-up, DB press"). Never silently drop.
10. Validate weekly movement coverage and direct-set targets per muscle.
11. Estimate session duration (§8.6).
12. If estimated duration > target × 1.15, apply accessory-cut (§8.6.1).
13. **Calibration checkpoint (NEW):** Run the generator on 20 representative input profiles before launch. Manually score outputs for sensibility. If <85% pass coach review, tune §8.8.3 weights before ship.
14. Generate plan explanation, validation warnings, coverage summary.
15. Save as `PlanVersion` in `draft`; user accepts to promote to `active`.

### 8.6 Session duration estimation

```
est_duration = sum_over_exercises(
    working_sets × (exercise.tempo_seconds × target_reps + exercise.rest_seconds)
    + exercise.warmup_overhead
    + exercise.setup_overhead
)
```

**Per-exercise fields on the seed list (NEW; replaces v0.5's fixed `avg_rep_time = 4s`):** every seeded exercise has its own `tempo_seconds`, `rest_seconds`, `warmup_overhead`, `setup_overhead`. Calibrated by the §5.5 author based on realistic execution.

#### 8.6.1 Accessory-cut priority

When over budget, cut in order:
1. Tertiary isolations (calf, forearm).
2. Duplicate movement intents within a session.
3. Reduce isolation set counts 3 → 2 (floor 2).
4. Reduce compound-accessory set counts 4 → 3 (floor 3 for accessory; primary compounds are floor 3 always).
5. Validation warning: "Session length target may be too short — consider 10 more minutes or fewer training days."

**Focus-muscle protection (NEW):** isolation sets that match the user's `focus muscle` skip step 1 and are demoted to step 3 only.

Never cut the primary compound for a slot.

### 8.7 Initial volume targets (v0)

Marked v0 — these are starting points subject to adjustment after launch. UI must communicate uncertainty.

**Beginner direct sets/week:**

| Muscle group | Sets/week |
|---|---:|
| Chest | 4–8 |
| Back | 6–10 |
| Quads | 4–8 |
| Hamstrings/glutes | 4–8 |
| Shoulders | 4–8 |
| Biceps | 4–8 |
| Triceps | 4–8 |
| Calves/core | 0–6 (optional) |

**Intermediate:**

| Muscle group | Sets/week |
|---|---:|
| Chest | 8–14 |
| Back | 10–16 |
| Quads | 8–14 |
| Hamstrings/glutes | 8–14 |
| Shoulders | 8–14 |
| Biceps | 6–12 |
| Triceps | 6–12 |
| Calves/core | 4–10 (optional) |

Generator targets the midpoint; users with no history start at the low end. Volume increases only after 2 weeks of consistent adherence with no high-soreness pair or pain flag.

### 8.8 Eligibility, scoring, tie-breakers

#### 8.8.1 Hard disqualifiers

- Requires unavailable equipment.
- Contraindicated by active pain/limitation per §7.3.2 mapping.
- Marked beginner-only and user is intermediate (or vice versa) — only when an alternative exists; relaxed automatically by §8.5 step 9.

#### 8.8.2 Rule precedence

1. Safety / pain (§7.3.2)
2. Equipment availability
3. Locked user choice (preferred / disliked)
4. Schedule / session length (post-cut)
5. Movement coverage
6. Experience-level volume cap
7. Disliked (soft penalty)
8. Focus muscles
9. Preferred
10. Variety

#### 8.8.3 Scoring (v0; calibration-pending per §8.5 step 13)

| Factor | Score |
|---|---:|
| Movement-intent match | +40 |
| Same primary muscle | +25 |
| Beginner-friendly (for beginner user) | +15 |
| Matches focus muscle | +10 |
| User preferred | +10 |
| Prior successful history | +8 |
| Low setup complexity (when session is short) | +5 |
| High fatigue cost on back-to-back day | -10 |
| Recently skipped 3+ times | -20 |
| User disliked | -30 |

These weights are an honest first guess. The §8.5 step 13 calibration may shift them ±10 before launch.

#### 8.8.4 Tie-breakers (within 3 points)

1. Lower setup complexity
2. More beginner-friendly
3. Better prior adherence
4. Lower fatigue cost
5. Stable variety (avoid weekly novelty)

### 8.9 Substitution algorithm

Same eligibility + scoring as generator, biased toward:

1. Preserve movement intent (+40, same as generator).
2. Preserve primary muscle (+25).
3. Similar difficulty / fatigue cost (+10 if within one tier).
4. Prior successful history (+8).
5. Standard penalties (disliked / repeatedly skipped).

Preserves planned set count and rep range when possible.

### 8.10 Substitution load handling

1. If user has history with the substitute, suggest the last successful load that hit at least the minimum target reps.
2. If no history, leave load blank with prompt: "Choose a conservative starting load near the lower end of the rep range with controlled form."
3. For bodyweight subs, use bodyweight/assisted/variation level instead of numeric load.
4. For machines with non-comparable load scales, leave load blank.

### 8.11 Exercise seed list

MVP ships ~80 curated exercises (down from v0.5's 150–200). Coverage: every movement intent in §8.3 has 6–10 options across the common equipment profiles (full home gym, basic home gym, commercial gym, dumbbell-only, bodyweight). Schema:

- name
- movement_intent[]
- primary_muscle, secondary_muscle[]
- equipment_required[]
- difficulty (beginner / intermediate)
- fatigue_cost (low / moderate / high)
- tempo_seconds, rest_seconds, warmup_overhead, setup_overhead
- cues: { setup, execution, common_mistake, safety_note? }

The list is committed in number and schema; specific contents are owned by the §5.5 author and must be locked before build week 1.

---

## 9. Progression Requirements

### 9.1 Model

Double progression with explicit precedence (§9.2). For 3 sets of 8–12:
- Add reps until hitting 12 across all working sets.
- Then suggest a conservative load increase (subject to checks).
- After load increases, reps may drop back toward the lower end.

### 9.2 Recommendation precedence

One recommendation per exercise per session, evaluated in order:

1. Pain/safety (per §7.3.2 mapping) → suggest substitute or hold.
2. Reduce check (§9.5) → suggest reduce.
3. Hold check (§9.4) → suggest hold.
4. Increase check (§9.3) → suggest increase.
5. Default → hold load, try to add reps next session.

### 9.3 Recommendation types in MVP

- Add reps (default)
- Add load
- Hold load
- Reduce load
- Substitute exercise
- Suggest deload (user approval required)

(Add set / Remove set / Modify recommendation are v1.1.)

### 9.4 Load increase threshold (noise-tolerant; revised from v0.5)

Suggest load increase when **all** are true:

- All planned working sets were completed.
- **Top-rep condition:** all sets at top of range *OR* (all sets within 1 rep of top *AND* the prior comparable session also met one of these conditions). This tolerates 12,12,11 → 12,12,12 noise that v0.5 would have stalled.
- No pain flag was logged for this session.
- Fatigue rating not high.
- No form-breakdown flag.

If RIR is enabled:
- 1–3 RIR supports normal load increase.
- 0 RIR + high fatigue → hold.
- 4+ RIR may support load increase but the explanation notes effort was easier than target.

### 9.5 Hold threshold

Suggest hold when **any** are true (and reduce / safety did not fire):
- User did not reach top reps on most sets.
- User recently increased load and reps dropped but stayed in range.
- Fatigue high.
- Logging incomplete.
- RIR missing and readiness unclear.

### 9.6 Reduce threshold

Suggest reduce when **any** are true (and safety did not fire):
- Reps below minimum target on 2+ working sets.
- Total reps drop ~20% at the same load across comparable sessions.
- User reports form breakdown.

(Pain is handled by precedence step 1; not listed here.)

### 9.7 Deload threshold (made explicit; v0.5 was ambiguous)

Suggest deload when **≥3 of these 5 signals** appear within a 7-day window:

1. Performance decreases across 2 comparable exposures.
2. Fatigue high in ≥2 recent sessions.
3. Soreness high in ≥2 recent sessions.
4. ≥2 missed workouts after prior consistency.
5. Pain or joint discomfort reported in this window.

Rules:
- Never auto-apply.
- Require user confirmation.
- Full reasoning visible in "Why?" expander.

### 9.8 Recommendation feedback (NEW)

When a user ignores or dismisses a recommendation, capture an optional `ignoredReason` enum:
- "Too aggressive"
- "Wrong exercise"
- "Not today"
- "Disagree with the reasoning"
- "Other" (free text, never sent to analytics)

Default is "no reason given." This is the data we need to tune §8.8.3 weights and §9 thresholds in v1.1.

### 9.9 Explanation requirement

Every recommendation surfaces a structured `reason` object:

- `recommendationType` (increase / hold / reduce / substitute / deload)
- `triggerFactors` (e.g., `["all_sets_at_top_rep", "no_pain", "fatigue_normal"]`)
- `userFacingReason` (templated: "You hit 12 reps on all 3 sets, no pain reported, fatigue normal — time to try +5 lb.")
- `educationalContext` (optional: "We progress load only after you can complete every working set at the top of the rep range. This is called *double progression*.")

The "Why?" expander shows `userFacingReason` always; tapping again reveals `educationalContext`. Templates are written in plain language and reviewed alongside §5.5 cues.

---

## 10. Nutrition

**Removed from MVP. See §5.2 (v1.1).**

(Was §10 in v0.5. Calorie/protein targets, saved meals, common foods, visual portion guides, and Mifflin-St Jeor calculation all move to v1.1. Body weight, height, age, sex are still collected at onboarding so v1.1 can use them — but they affect nothing in MVP rules.)

---

## 11. Data Model

### 11.1 Principles

- Planned data and completed data separate.
- Plan changes don't overwrite history.
- User-entered logs preserved.
- Recommendations auditable.
- No AI fields in MVP schemas.

### 11.2 Core entities (MVP)

- User
- UserProfile
- GoalPlan (goal field; doesn't affect MVP rules)
- EquipmentProfile (single, per user)
- Exercise (seeded, read-only)
- UserExercisePreference
- PlanVersion
- WorkoutDay
- ExerciseInstance
- SetPrescription
- WorkoutSession
- SetLog
- PostWorkoutCheckIn
- Recommendation
- NotificationPreference

(Removed from v0.5 list: EquipmentCalendarEntry, FoodLog, SavedMeal, MealTemplate, CommonFood. All move to v1.1.)

### 11.3 WorkoutDay vs WorkoutSession

WorkoutDay is a planned template within a PlanVersion (no `scheduledDate`). WorkoutSession is the dated execution instance with `scheduledForDate`, `startedAt`, `completedAt`, `status`, `planVersionId`, `workoutDayId`. Performance lives on WorkoutSession and SetLog, never WorkoutDay.

### 11.4 SetPrescription

Fields: id, exerciseInstanceId, setIndex, setType (working / warmup), targetRepMin, targetRepMax, targetRirMin (optional), targetRirMax (optional). AMRAP, top set/backoff, pyramids, drop sets are v1.1+.

### 11.5 PostWorkoutCheckIn

- id
- workoutSessionId
- sorenessLevel: enum (none / mild / moderate / high)
- **sorenessLocation: enum[]** (NEW; same enum as painLocation; optional)
- painFlag: boolean
- painLocation: enum[] — required if painFlag
- painType: enum (sharp / dull_aching / joint / nerve_like / other) — required if painFlag
- painNotes: string (optional, never sent to analytics)
- fatigueLevel: enum (low / normal / high) — optional
- formBreakdown: boolean — optional
- createdAt

### 11.6 Recommendation

- id
- userId
- type
- status (pending / accepted / ignored / dismissed / expired)
- title
- userFacingReason
- educationalContext (optional)
- triggerFactors[]
- inputsUsed
- targetEntityType
- targetEntityId
- suggestedChange
- **ignoredReason: enum?** (NEW per §9.8)
- createdAt
- updatedAt

No `source: ai` field. Add later if AI is added.

### 11.7 Body metrics

MVP collects body weight only at onboarding (kept current via Profile). Waist/neck/hip are post-MVP.

---

## 12. State Machines

### 12.1 PlanVersion

States: `draft`, `active`, `archived`, `reverted`. One active per user. Revert creates a new active clone from a prior version. Workout logs remain attached to original sessions regardless of plan changes.

### 12.2 WorkoutSession

States: `not_started`, `in_progress`, `completed`, `partial`, `abandoned`, `skipped`, `completed_outside_app`, `deleted` (soft).

**In-flight rule (NEW; closes v0.5 gap):** A session in `in_progress` always references its original `planVersionId`. Plan regeneration creates a new `PlanVersion` but does not migrate active sessions.

### 12.3 Recommendation

States: `pending`, `accepted`, `ignored`, `dismissed`, `expired`. Modify is v1.1.

### 12.4 Sync states

Client states: `synced`, `pending_create`, `pending_update`, `pending_delete`, `conflict`. Per §7.5.2.

---

## 13. API & Architecture

### 13.1 Decisions

| Item | Decision |
|---|---|
| Platform | React Native / Expo |
| Language | TypeScript |
| Backend | Supabase |
| Database | Supabase Postgres |
| Auth | Supabase Auth, email/password. Apple/Google sign-in is fast-follow (per G3 results). |
| Plan generator | Server-side rules engine |
| Nutrition | Not in MVP |
| Analytics | PostHog (decided; was open in v0.5). Self-hosted option deferred. |
| Admin/support | Not in MVP |

### 13.2 API groups

- Auth/profile
- Onboarding
- Equipment
- Exercises
- Plans
- Workout sessions
- Post-workout check-ins
- Recommendations
- Notifications/preferences

(No nutrition API in MVP.)

### 13.3 Plan generation contract

Server in:
- User profile, goal (recorded only), schedule, equipment profile, preferences/limitations, prior plan if regenerating.

Server out:
- PlanVersion
- WorkoutDay templates with movement-intent slots
- ExerciseInstances with structured `reason`
- SetPrescriptions
- Plan-level explanation
- Validation warnings (including unfillable-slot warnings per §8.5 step 9)
- Movement coverage summary

### 13.4 Local workout draft

Saved client-side after every meaningful action. Includes: localDraftId, planVersionId, workoutDayId, workoutSessionId (after first sync), startedAt, completed set logs with clientMutationIds, skipped states, notes, lastSavedAt, syncStatus.

### 13.5 Notifications

| Notification | Trigger | Default |
|---|---|---|
| Workout reminder | 30 min before scheduled time | On |
| Missed-workout follow-up | Day after a missed workout | On |
| Recommendation pending review | Plan-level changes awaiting approval | On |
| Streak milestone | 7/30/90 days | Off (opt-in) |

(Weekly review notification removed; weekly review is v1.1.)

All honor user time zone and quiet hours (default 9pm–8am).

### 13.6 Data deletion

- Account deletion request → enters pending immediately; user data inaccessible.
- Hard-delete or anonymize within 30 days, subject to legal review.
- Soft-deleted workout logs excluded from recommendations.
- Final policy text per §18.

---

## 14. Privacy, Safety & Compliance

### 14.1 Principles

- Collect only what's used. (Notably: weight/height/age stored but not used in MVP — flagged in privacy notice as "for future nutrition features.")
- No photos.
- No precise location.
- Weight, pain notes, workout notes treated as sensitive wellness data.

### 14.2 Safety boundaries

The app does not provide medical diagnosis, injury treatment plans, eating-disorder coaching, extreme weight-loss recommendations, supplement protocols, or training-through-pain.

### 14.3 Minors

Adult-users-only at MVP. DOB gate at onboarding; under-18 blocked. Separate minors policy required for any U18 expansion.

### 14.4 Retention

| Data | Retention |
|---|---|
| Workout logs | Indefinite while active |
| Soft-deleted logs | 30 days then hard-delete |
| Account post-deletion | 30 days then hard-delete or anonymize |
| Analytics | 90 days |
| Pain notes | Treated as health data; never to analytics; hard-deleted with account |
| Auth/session logs | 12 months |

### 14.5 Compliance

- GDPR: data export, account deletion, data minimization.
- CCPA: opt-out of analytics in Settings.
- HIPAA: not applicable; consumer wellness, not a covered entity. App does not market as a medical device.
- Final compliance posture confirmed by legal review (§18).

---

## 15. Analytics

PostHog. Product events only. Never raw values.

### 15.1 Events

- account_created
- onboarding_started, onboarding_completed
- onboarding_abandoned_with_plan (NEW per §6.7)
- plan_generated, plan_generation_failed
- workout_started, set_logged, workout_completed, workout_completed_partial
- exercise_substituted
- missed_workout_action_selected
- recommendation_shown, recommendation_accepted, recommendation_ignored
- recommendation_why_expanded, recommendation_why_then_accepted (NEW for §3.2 trust composite)
- trust_survey_submitted (NEW)

### 15.2 PII boundary

Never to analytics: body weight values, pain notes, workout notes, email/name/exact age/exact height, soreness/pain location specifics. Aggregate counts and event timing only.

---

## 16. AI Policy

AI is not in MVP. MVP rules are template-based throughout.

Future AI candidates (no commitment):
- Natural-language enrichment of explanations (current explanations are templated)
- Chat coach
- Weekly summaries
- Natural-language plan edits

Rules for any future AI:
- Must not auto-apply major plan changes without approval.
- Must not override locked user choices without warning.
- Must not recommend unsafe behavior.
- Must be auditable.
- Must enrich the rules-based reason, not replace it.

---

## 17. First Technical Deliverable

Before production code:

`docs/technical/CANONICAL_DATA_AND_SYNC_CONTRACT.md`

Defines:

1. Core entities and relationships (extends §11).
2. State machines (extends §12).
3. Local workout draft sync behavior (extends §7.5.2 and §12.4).
4. Notification spec (extends §13.5).

The generator algorithm, scoring, and plan generation contract are inlined in §8 of this PRD and not duplicated.

Acceptance:
- Mobile and backend implementation can begin without redefining core entities.
- Planned workouts cannot be confused with completed.
- Generator inputs and outputs are explicit and consistent with §8.
- Workout logs are protected from plan-regeneration side effects.

---

## 18. Open Decisions Before Build

Genuine open decisions that must close before architecture freeze:

1. Confirm legal/privacy deletion policy text and retention windows.
2. Confirm final copy for safety disclaimers (medical guidance, EDs, training-through-pain).
3. Confirm whether Apple/Google sign-in is in MVP or fast-follow (depends on G3).
4. Confirm exercise seed list contents (count and schema committed in §8.11; specific exercises owned by §5.5 author).
5. Confirm pricing ($9.99/mo placeholder, validated via G3).
6. Confirm WCAG audit cadence and accessibility QA owner.
7. Identify §5.5 content author and reviewer before build week 1.

(Removed from v0.5 list: Supabase, Expo/RN, analytics provider — all decided.)

---

## 19. PRD Quality Rules

- Maintain one canonical MVP definition.
- Do not duplicate requirements across sections.
- Move non-MVP ideas to §5.2 or §20.
- Do not add AI fields to MVP schemas.
- Update dependent sections when a decision changes.
- Every recommendation surface must specify how the "Why?" explanation is sourced.
- Numbers presented as decisions (weights, thresholds, durations) must say so; numbers that are guesses must be marked v0/calibration-pending.

---

## 20. Appendix: Post-MVP Backlog

**v1.1 (next):**
- Lite nutrition (calorie + protein, saved meals, common foods, visual guides)
- Weekly review tab
- Per-day equipment (gated on G4)
- Recommendation modify
- Rest timer, warm-up marker
- Localization
- Tablet UX

**Mid-term:**
- Strict food database, custom foods, recent foods
- USDA / Open Food Facts integration
- Barcode scanning
- Exercise videos
- Wearable integrations
- Advanced progression / detailed muscle-volume views

**Backlog tail:**
- Advanced periodization
- AI chat coach
- Natural-language logging
- Coach/admin portal
- Social/community

---

## 21. Worked Example: Generator Output

**Inputs:**
- Experience: intermediate
- Goal: bulk
- Days/week: 4
- Preferred days: Mon/Tue/Thu/Fri
- Session length target: 60 min
- Equipment: full home gym (rack, barbell, plates 5–250 lb, adjustable DB 5–60 lb, cable stack, bench)
- Focus muscle: shoulders
- No limitations

**Output (abbreviated):**

```
PlanVersion id=pv_001, status=draft, splitTemplate=upper_lower

WorkoutDay 1 (Upper A — Mon)
  e1: Barbell Bench Press           slot=horizontal_push     4×6–10
       reason: { dominant: ["movement_intent", "primary_compound"] }
  e2: Standing Overhead Press       slot=vertical_push       4×6–10
       reason: { dominant: ["movement_intent", "focus_muscle_match"] }
  e3: Chest-Supported DB Row        slot=horizontal_pull     4×8–12
       reason: { dominant: ["movement_intent", "low_fatigue_cost"] }
  e4: Lat Pulldown                  slot=vertical_pull       3×8–12
  e5: DB Lateral Raise              slot=isolation_side_delt 4×12–15
       reason: { dominant: ["focus_muscle_match", "isolation_for_focus"] }
  e6: Cable Triceps Pushdown        slot=isolation_triceps   3×10–15
  estimated_duration_min: 58

WorkoutDay 2 (Lower A — Tue)
  ... (squat, hinge, lunge, isolation; details omitted)

WorkoutDay 3 (Upper B — Thu)
  Variety enforcement: H-push and V-push differ from Upper A.
  e1: Incline DB Press              slot=horizontal_push
  e2: Seated DB Shoulder Press      slot=vertical_push
  ...

WorkoutDay 4 (Lower B — Fri)
  ...

Plan-level reason: "4-day Upper/Lower split chosen for intermediate user training 4 days/week.
  Shoulders prioritized via focus muscle: 4 sets of OHP and 8 sets of side delt isolation
  per week (above midpoint of 8–14 range). Vertical and horizontal push/pull balanced
  across A/B days; A and B differ in at least one exercise per intent for variety."

Validation warnings: none
Movement coverage: all 7 compound intents covered; focus muscle 12 sets/week (within 8–14 range)
Cross-slot exclusion: no exercise repeats within a session; no primary compound repeats more than twice in the week.
```

---

**End of v0.6.**
