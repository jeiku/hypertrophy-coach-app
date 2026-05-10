# Hypertrophy & Muscle Gain Coach
## Product Requirements Document - v0.4 Canonical Draft

**Status:** Canonical planning draft  
**Primary audience:** Product owner, designer, engineer, Codex/technical reviewer  
**Primary user:** Beginner-to-intermediate hypertrophy learner  
**MVP platform:** Mobile-first React Native / Expo app with Supabase/Postgres backend

---

## 1. Product Summary

Hypertrophy & Muscle Gain Coach is a mobile-first training and nutrition app for beginner-to-intermediate users who want a clear, guided path to building muscle without manually designing programs or managing spreadsheets.

The app creates a weekly hypertrophy plan based on the user’s schedule, equipment availability by training day, experience level, goal, and preferences. Users log workouts, receive simple progression recommendations, track lightweight nutrition adherence, and review weekly progress.

**Primary promise:** Know what to do today, understand why, and make steady progress with a plan that adapts to real life.

---

## 2. Target User & Positioning

### 2.1 Primary persona

**Early/intermediate hypertrophy learner**

This user has either recently started lifting or has trained inconsistently and wants better structure. They are not an expert in programming, volume landmarks, periodization, or nutrition math. They want to build muscle, learn along the way, and avoid overcomplicating the process.

They need:

- A simple weekly plan
- Clear workout instructions
- Easy set logging
- Simple progression guidance
- Equipment-aware exercise choices
- Lightweight nutrition support
- Non-shaming recovery from missed workouts

### 2.2 Secondary persona

**Beginner lifter**

This user needs conservative defaults, simple exercise choices, and minimal jargon. They should be able to use the app without understanding RIR, weekly set targets, or training theory.

### 2.3 Non-primary persona

**Advanced bodybuilding/programming expert**

Advanced users may still use the app, but MVP should not be optimized around advanced periodization, detailed volume landmarks, mesocycle design, or full custom programming.

### 2.4 Product positioning

The app should feel like a guided coach, not a complex spreadsheet.

The differentiator is not merely “equipment-aware planning,” because competitors already support equipment-aware workout generation. The MVP differentiation should be framed as:

> A beginner/intermediate-friendly hypertrophy coach that combines day-specific equipment planning, explainable progression, lite nutrition adherence, and real-life recovery flows in one simple daily experience.

---

## 3. Goals, Success Metrics & Business Model

### 3.1 MVP product goals

- Help users complete workouts consistently.
- Help users understand what to progress next.
- Reduce confusion around exercise selection when equipment changes by day.
- Support nutrition adherence without requiring detailed food tracking at launch.
- Validate that users trust simple, explainable recommendations.

### 3.2 MVP success metrics

Primary MVP success metrics:

- Day 7 retention
- Day 30 retention
- Average workouts completed per user per week
- Percent of users completing at least 2 workouts per week
- Percent of scheduled workouts started
- Percent of started workouts completed or intentionally marked partial
- Recommendation acceptance rate
- Weekly review viewed rate
- Lite nutrition logging days per active user per week

Secondary metrics:

- Exercise substitution rate
- Missed-workout recovery action rate
- Workout draft recovery rate after app close/offline interruption
- Plan edit rate
- User-reported confidence/trust score after first week

### 3.3 Business model draft

MVP should validate product value before overbuilding monetization.

Recommended model:

- Free trial or free starter tier
- Paid subscription for continued adaptive planning, progression recommendations, and weekly review
- No ads in MVP
- No marketplace or coach platform in MVP

Monetization is not part of MVP Core implementation unless required for launch.

---

## 4. Competitive Context

### 4.1 Competitor categories

Relevant competitors include:

- General adaptive workout apps, such as Fitbod
- Hypertrophy-specific apps, such as RP Hypertrophy
- Manual workout trackers, such as Strong-style logging apps
- Spreadsheet/program templates
- Nutrition trackers, such as MacroFactor/MyFitnessPal-style apps

### 4.2 Competitive implications

Because existing products already offer equipment selection, exercise substitutions, workout logging, and hypertrophy-oriented guidance, this app should not claim day-level equipment alone as the entire differentiator.

MVP should compete by being:

- Easier for beginner/intermediate users
- Less overwhelming than advanced hypertrophy tools
- More adaptive than static templates
- More structured than generic workout trackers
- More forgiving around missed workouts and partial completion
- More practical for users who split time between home and gym
- Nutrition-aware without requiring strict logging on day one

### 4.3 MVP positioning statement

> For beginner-to-intermediate lifters who want to build muscle but do not want to design their own program, this app provides a guided daily workout and lite nutrition loop that adapts to schedule, equipment, and progress while explaining recommendations in plain language.

---

## 5. MVP Scope

### 5.1 MVP Core

MVP Core is the first shippable product loop.

MVP Core must support:

1. Account creation/login
2. Onboarding
3. Weekly plan generation using schedule and day-level equipment
4. Today dashboard
5. Workout player
6. Durable set logging
7. Partial/missed workout handling
8. Basic exercise substitutions
9. Rules-based progression recommendations
10. Lite nutrition tracking
11. Weekly review
12. Basic privacy/account controls

### 5.2 MVP Extended

MVP Extended can follow after the Core loop is validated.

MVP Extended includes:

- Strict food database search
- Custom food creation
- Multiple food providers
- Recommendation modify flow
- Full plan revert UI
- Detailed muscle-volume summaries
- Rest timer
- Warm-up set marker
- More advanced analytics
- Robust multi-device conflict UX

### 5.3 Explicitly not in MVP

- AI chat coach
- Autonomous AI plan changes
- Barcode scanning
- Wearable integrations
- Computer vision/form checks
- Photo storage or body photo comparison
- Advanced periodization blocks
- Coach marketplace
- Social feed/community
- Admin/support access to individual user data
- Exercise video library
- Medical diagnosis or injury rehabilitation programming

### 5.4 Exercise instruction MVP decision

Because the primary user includes beginners, MVP must include **basic written exercise instructions and safety cues** for seeded exercises.

MVP does not require videos, but each exercise should include:

- Setup cue
- Execution cue
- Common mistake cue
- Safety note if relevant

---

## 6. User Experience Requirements

### 6.1 App navigation

Recommended MVP tabs:

1. Today
2. Plan
3. Workout History / Progress
4. Nutrition
5. Settings

The Today tab should answer:

> What should I do today, and why?

### 6.2 Onboarding flow

Onboarding should be shortened to avoid overwhelming the target user.

MVP onboarding screens:

1. Welcome / value proposition
2. Account creation/login
3. Goal and experience
4. Schedule and session length
5. Equipment by training day
6. Preferences/limitations
7. Nutrition mode and initial target setup
8. Review and generate plan

### 6.3 Onboarding required inputs

Required:

- Experience level: beginner or intermediate
- Goal: bulk, cut, recomp
- Weight
- Height
- Age
- Sex, only if using calculated calorie targets
- Days per week: 2 to 6
- Preferred training days
- Session length target
- Equipment profile per training day
- Nutrition preference: lite mode for MVP Core

Optional:

- Disliked exercises
- Preferred exercises
- Focus muscles
- Movement limitations

Not MVP Core:

- Waist/neck/hip measurements
- Detailed body composition estimates
- Physique photo uploads

### 6.4 Today dashboard

Today must show:

- Scheduled workout or rest-day state
- Start/resume workout action
- Lite nutrition progress
- Next best action coach card
- Missed/partial workout recovery action if needed

States:

- No plan yet
- Training day, workout not started
- Workout in progress
- Workout completed
- Rest day
- Missed workout
- Offline/local draft exists

### 6.5 Workout player

Workout player must support:

- Workout overview
- Exercise list
- Basic instructions/safety cues
- Planned sets and target rep range
- Previous performance if available
- Log weight and reps
- Optional RIR hidden by default for beginners
- Skip set/exercise
- Add set
- Substitute exercise
- Finish as completed or partial

### 6.6 Weekly review

Weekly review must show:

- Workouts scheduled vs completed
- Missed or partial workouts
- Basic weight trend if enough data exists
- Lite nutrition adherence
- Progression recommendations
- Next-week plan changes requiring user approval

### 6.7 Accessibility, localization, device support, and time zone

MVP requirements:

- Phone-first layouts
- Usable on common tablet screen sizes, but tablet-specific UX is not required
- Text scaling support
- Sufficient color contrast
- Screen-reader-friendly labels for core actions
- Store user time zone
- Schedule workouts/reminders using the user’s local time zone
- English only for MVP

Post-MVP:

- Localization
- Tablet-optimized layout
- More robust accessibility QA

---

## 7. Edge Cases & State Handling

### 7.1 Missed workout

When a scheduled workout is missed, the app should offer:

- Move to next available day
- Skip and continue plan
- Replace next workout
- Mark completed outside app
- Regenerate rest of week

Rules:

- Do not shame the user.
- Do not double future volume automatically.
- If missed workouts repeat, suggest fewer weekly training days or shorter sessions.

### 7.2 Partial workout

A partial workout is valid data.

Rules:

- Completed sets count.
- Uncompleted planned sets do not count as failed attempts.
- Repeated partial workouts due to time should suggest shorter sessions.

### 7.3 Soreness vs pain

The app must distinguish normal muscle soreness from pain/discomfort.

Soreness examples:

- General muscle soreness
- Expected post-workout fatigue
- Delayed soreness that improves normally

Pain/discomfort examples:

- Sharp pain
- Joint pain
- Nerve-like pain
- Worsening pain during movement
- Pain that changes form
- Pain that persists or feels abnormal

Rules:

- Soreness may influence volume/progression conservatively.
- Pain/discomfort should suppress load increases and suggest substitution or professional guidance if persistent.
- The app must not diagnose or treat injuries.

### 7.4 Equipment change

Changing equipment for a future training day should trigger revalidation of upcoming workouts.

Options:

- Substitute incompatible exercises
- Regenerate affected workout
- Regenerate rest of week
- Keep plan and warn user

Completed workouts must not be rewritten.

### 7.5 Offline/interrupted workout

Workout draft data must be saved locally after every meaningful action.

If app closes or network fails:

- User can resume the workout.
- Completed set logs are preserved.
- Sync retries when network returns.
- Completed set logs are never silently discarded.

---

## 8. Program Generator Requirements

### 8.1 Generator principle

MVP generator must be deterministic and rules-based. AI is not required for MVP Core.

The generator should output:

- PlanVersion
- WorkoutDay templates
- ExerciseInstances
- Plan explanation
- Validation warnings
- Movement coverage summary

### 8.2 Generator inputs

Inputs:

- Experience level
- Goal
- Days per week
- Preferred days
- Session length
- Equipment per training day
- Preferences
- Limitations
- Focus muscles
- Prior plan context if regenerating

### 8.3 Split selection

| Days/week | Beginner default | Intermediate default |
|---:|---|---|
| 2 | Full Body A/B | Full Body A/B |
| 3 | Full Body A/B/C | Full Body A/B/C |
| 4 | Upper/Lower | Upper/Lower |
| 5 | Upper/Lower + Full/Focus | Upper/Lower + Full/Focus |
| 6 | Conservative PPL or Full Body rotation | PPL x 2 |

### 8.4 Generator algorithm

The MVP generator should follow this deterministic sequence:

1. Validate required inputs.
2. Select split template from days/week and experience.
3. Create workout-day templates.
4. Assign movement-intent slots per workout.
5. Set initial weekly volume targets by experience.
6. For each slot, filter eligible exercises.
7. Score eligible exercises.
8. Assign highest-ranked exercise.
9. Validate weekly movement coverage.
10. Estimate session duration.
11. Reduce lower-priority accessory work if sessions are too long.
12. Create plan explanation and warnings.
13. Save as PlanVersion.

### 8.5 Initial volume targets

These are conservative starting targets, not advanced volume landmarks.

Beginner starting direct sets/week:

| Muscle group | Sets/week |
|---|---:|
| Chest | 4–8 |
| Back | 6–10 |
| Quads | 4–8 |
| Hamstrings/glutes | 4–8 |
| Shoulders | 4–8 |
| Biceps | 2–6 |
| Triceps | 2–6 |
| Calves/core | Optional |

Intermediate starting direct sets/week:

| Muscle group | Sets/week |
|---|---:|
| Chest | 6–12 |
| Back | 8–14 |
| Quads | 6–12 |
| Hamstrings/glutes | 6–12 |
| Shoulders | 6–12 |
| Biceps | 4–8 |
| Triceps | 4–8 |
| Calves/core | Optional |

Rules:

- Start near the low end.
- Increase volume only after adherence and recovery are acceptable.
- Do not present these numbers as precise science.

### 8.6 Rule precedence

Apply constraints in this order:

1. Safety/pain limitation
2. Equipment availability
3. Locked user choice
4. Schedule/session length
5. Major movement coverage
6. Experience-level volume cap
7. Disliked exercises
8. Focus muscles
9. Preferred exercises
10. Variety/tie-breakers

### 8.7 Exercise scoring

After hard filters are applied, score candidates.

Hard disqualifiers:

- Requires unavailable equipment
- Contraindicated by pain/limitation
- Not appropriate for experience level unless no alternative exists

Suggested scoring:

| Factor | Score |
|---|---:|
| Same movement intent | +40 |
| Same primary muscle | +25 |
| Beginner-friendly, for beginner user | +15 |
| Matches focus muscle | +10 |
| User preferred | +10 |
| Prior successful history | +8 |
| Low setup complexity when session is short | +5 |
| High fatigue cost on back-to-back training day | -10 |
| User disliked | -30 |
| Recently skipped repeatedly | -20 |

Tie-breakers:

1. Lower setup complexity
2. More beginner-friendly
3. Better prior adherence
4. Lower fatigue cost
5. Stable variety, avoid unnecessary novelty

### 8.8 Substitution algorithm

When substituting an exercise:

1. Filter by safety/pain limitations.
2. Filter by equipment available that day.
3. Prioritize same movement intent.
4. Prioritize same primary muscle.
5. Prefer similar difficulty and fatigue cost.
6. Prefer exercises with prior successful history.
7. Avoid disliked or repeatedly skipped exercises.

Substitution must preserve:

- Movement intent where possible
- Target muscle
- Planned set count where reasonable
- Target rep range where reasonable

### 8.9 Substitution load handling

Do not blindly copy load from the old exercise.

Load selection rules:

1. If user has history with the substitute exercise, suggest the last successful load that reached at least the minimum target reps.
2. If no history exists, leave load blank or suggest “choose a conservative starting load.”
3. Explain that the user should choose a weight they can perform near the lower end of the rep range with controlled form.
4. For bodyweight substitutions, use bodyweight/assisted/variation level instead of numeric load.

---

## 9. Progression Requirements

### 9.1 Progression model

MVP uses double progression.

For an exercise with 3 sets of 8–12 reps:

- Add reps until all planned working sets reach 12 reps.
- Then suggest a conservative load increase if fatigue/pain checks are acceptable.
- After load increases, reps may drop back toward the lower end of the range.

Example:

- Session 1: 50 × 10, 9, 8 → hold load, try to add reps
- Session 2: 50 × 12, 11, 10 → hold load, try to add reps
- Session 3: 50 × 12, 12, 12, no pain, normal fatigue → suggest small load increase
- Session 4: 55 × 9, 8, 8 → hold new load

### 9.2 Recommendation types

MVP Core supports:

- Add reps
- Add load
- Hold load
- Reduce load
- Substitute exercise
- Suggest deload, user approval required

MVP Extended:

- Add set
- Remove set
- Modify recommendation before accepting

### 9.3 Load increase threshold

Suggest load increase when all are true:

- All planned working sets were completed.
- All working sets reached the top of the target rep range.
- No pain/discomfort flag was logged.
- Fatigue rating is not high.
- Form was not marked as breakdown/limited.

If RIR is available:

- 1–3 RIR supports normal load increase.
- 0 RIR plus high fatigue suggests holding.
- 4+ RIR may support load increase but should explain effort may have been easier than target.

### 9.4 Hold threshold

Suggest holding load when any are true:

- User did not reach top reps on all sets.
- User recently increased load and reps dropped but remain within range.
- Fatigue is high.
- Logging is incomplete.
- RIR is missing and performance readiness is unclear.

### 9.5 Reduce threshold

Suggest reducing load when any are true:

- User falls below minimum rep target on 2+ working sets.
- Total reps drop by roughly 20% at the same load across comparable sessions.
- User reports pain/discomfort.
- User reports form breakdown.

### 9.6 Deload threshold

Suggest deload only when multiple signals occur together:

- Performance decreases across 2 comparable exposures.
- Fatigue is high across 2+ recent sessions.
- Soreness is high across 2+ recent sessions.
- Multiple workouts are missed after prior consistency.
- Pain or joint discomfort is reported.

Rules:

- Never auto-apply deload.
- Require user confirmation.
- Explain why deload is suggested.

---

## 10. Nutrition Requirements

### 10.1 MVP Core nutrition

Lite nutrition is MVP Core.

MVP Core supports:

- Calorie target
- Protein target
- Carbs/fat targets if user wants them
- Saved meals
- Quick-add macros
- Edit/delete nutrition logs
- Daily nutrition summary
- Weekly adherence summary

Strict food search is MVP Extended.

### 10.2 Calorie formula

MVP default uses Mifflin-St Jeor when required inputs exist.

Activity multipliers:

| Activity | Multiplier |
|---|---:|
| Sedentary | 1.2 |
| Lightly active | 1.375 |
| Moderately active | 1.55 |
| Very active | 1.725 |
| Extremely active | 1.9 |

Goal adjustment:

| Goal | Adjustment |
|---|---:|
| Bulk | +150 to +300 kcal/day |
| Recomp | Maintenance or slight adjustment |
| Cut | -250 to -500 kcal/day |

### 10.3 Protein target

Default protein target:

- Approximately 1.8 g/kg/day

Allowed range:

- 1.6–2.2 g/kg/day depending on goal, preference, and calorie level

### 10.4 Override precedence

Nutrition target precedence:

1. User manual target
2. User-accepted recommendation
3. System-calculated target
4. Prior target carried forward

### 10.5 Safety

The app must not support extreme calorie targets.

If calculated values appear unsafe:

- Show a warning
- Suggest safer defaults
- Allow manual review
- Avoid giving medical advice

---

## 11. Data Model

### 11.1 Data model principles

- Planned data and completed data must be separate.
- Plan changes must not overwrite workout history.
- User-entered logs must be preserved.
- Recommendations must be auditable.
- MVP should not pre-bake AI into required schemas.

### 11.2 Core entities

Core MVP entities:

- User
- UserProfile
- GoalPlan
- EquipmentProfile
- EquipmentCalendarEntry
- Exercise
- UserExercisePreference
- PlanVersion
- WorkoutDay
- ExerciseInstance
- SetPrescription
- WorkoutSession
- SetLog
- PostWorkoutCheckIn
- Recommendation
- FoodLog
- SavedMeal
- NotificationPreference

### 11.3 WorkoutDay vs WorkoutSession

WorkoutDay is a recurring planned template inside a PlanVersion.

WorkoutDay should not include `scheduledDate`.

WorkoutSession is the dated execution instance.

WorkoutSession may include:

- scheduledForDate
- startedAt
- completedAt
- status
- planVersionId
- workoutDayId

Actual performance belongs to WorkoutSession and SetLog, not WorkoutDay.

### 11.4 SetPrescription

Do not model planned sets only as a single integer if it blocks future flexibility.

MVP can use straight sets, but model planned work as SetPrescription records or structured data.

Fields:

- id
- exerciseInstanceId
- setIndex
- setType: working or warmup
- targetRepMin
- targetRepMax
- targetRirMin, optional
- targetRirMax, optional

MVP supports:

- working sets
- optional warmup marker

Post-MVP can support:

- AMRAP
- top set/backoff
- pyramids
- drop sets

### 11.5 Recommendation schema

MVP recommendations are generated by the rules engine.

Fields:

- id
- userId
- type
- status
- title
- reasonText
- inputsUsed
- targetEntityType
- targetEntityId
- suggestedChange
- createdAt
- updatedAt

Do not include `source: ai` in MVP-required schema.

If AI is added later, extend the model with an optional source field or AI metadata table.

### 11.6 Body metrics

MVP Core uses body weight only.

Waist/neck/hip are post-MVP unless a specific feature uses them.

This supports data minimization.

---

## 12. State Machines

### 12.1 PlanVersion states

States:

- draft
- active
- archived
- reverted

Rules:

- One active plan per user.
- Revert creates a new active clone from a prior version.
- Completed workout logs remain attached to original sessions.

### 12.2 WorkoutSession states

States:

- not_started
- in_progress
- completed
- partial
- abandoned
- skipped
- completed_outside_app
- deleted

Rules:

- partial is valid data.
- skipped is not failed.
- deleted should be soft-deleted initially.

### 12.3 Recommendation states

States:

- pending
- accepted
- ignored
- dismissed
- expired
- reverted

MVP Core supports accept and ignore.

Modify is MVP Extended.

### 12.4 Sync states

Client sync states:

- synced
- pending_create
- pending_update
- pending_delete
- conflict

MVP rule:

- Preserve workout drafts locally.
- Use idempotent mutation IDs for retries.
- Never silently discard completed set logs.

---

## 13. API & Architecture Requirements

### 13.1 Architecture defaults

| Decision | MVP default |
|---|---|
| Platform | Mobile-first React Native / Expo |
| Language | TypeScript |
| Backend | Supabase |
| Database | Supabase Postgres |
| Auth | Supabase Auth, email/password first |
| Plan generation | Server-side canonical rules engine |
| Nutrition Core | Lite nutrition first |
| Strict nutrition | MVP Extended |
| Analytics | Minimal privacy-safe events or none |
| Admin/support access | Not in MVP Core |

### 13.2 API groups

MVP API groups:

- Auth/profile
- Onboarding
- Equipment
- Exercises
- Plans
- Workout sessions
- Recommendations
- Lite nutrition
- Progress/weekly review
- Notifications/preferences

### 13.3 Server-side plan generation contract

The server must accept:

- User profile
- Goal
- Schedule
- Equipment calendar
- Preferences/limitations
- Existing plan context for regeneration

The server must return:

- PlanVersion
- WorkoutDay templates
- ExerciseInstances
- SetPrescriptions
- Explanation
- Validation warnings

### 13.4 Local workout draft contract

Client must save local workout draft after every meaningful action.

Draft includes:

- localDraftId
- planVersionId
- workoutDayId
- startedAt
- completed set logs
- skipped states
- notes
- lastSavedAt
- syncStatus

### 13.5 Data deletion policy draft

Technical default:

- User can request account deletion.
- Account enters pending deletion immediately.
- User-owned data becomes inaccessible after confirmation.
- Hard-delete or anonymize within 30 days, subject to legal review.
- Soft-deleted workout logs are excluded from recommendations and progress.

---

## 14. Privacy, Safety & Compliance

### 14.1 Privacy principles

- Collect only necessary data.
- Do not store photos in MVP.
- Do not collect precise location.
- Treat body weight, nutrition logs, workout notes, and pain flags as sensitive wellness data.
- Avoid sending raw food names, body metrics, pain notes, or private notes to analytics.

### 14.2 Safety boundaries

The app must not provide:

- Medical diagnosis
- Injury treatment plans
- Eating disorder coaching
- Extreme weight-loss recommendations
- Unsafe supplement/drug protocols
- Training-through-pain recommendations

### 14.3 Minors policy

MVP should assume adult users only unless a separate minors policy is created.

---

## 15. Analytics

If analytics is included, track product events only.

MVP events:

- account_created
- onboarding_started
- onboarding_completed
- plan_generated
- plan_generation_failed
- workout_started
- set_logged
- workout_completed
- workout_completed_partial
- exercise_substituted
- missed_workout_action_selected
- lite_nutrition_logged
- recommendation_shown
- recommendation_accepted
- recommendation_ignored
- weekly_review_viewed

Do not send sensitive raw values to analytics.

---

## 16. AI Policy

AI is not MVP Core.

MVP Core uses deterministic rules for:

- Plan generation
- Substitutions
- Progression recommendations
- Weekly review summaries

Post-MVP AI candidates:

- Natural-language explanations
- AI chat coach
- AI-generated summaries
- Natural-language plan edits
- AI-assisted substitutions

Rules for future AI:

- AI must not auto-apply major plan changes without approval.
- AI must not override locked choices without warning.
- AI must not recommend unsafe training or nutrition behavior.
- AI recommendations must be auditable.

---

## 17. First Technical Deliverable

Before production code, create:

`docs/technical/CANONICAL_DATA_AND_PLAN_GENERATION_CONTRACT.md`

This contract must define:

1. Core entities and relationships
2. State machines
3. Plan generation input schema
4. Plan generation output schema
5. Validation/warning contract
6. Generator algorithm
7. Exercise scoring and substitution scoring
8. Versioning rules
9. Local workout draft sync behavior
10. Nutrition target formula

Acceptance criteria:

- Mobile and backend implementation can begin without redefining core entities.
- Planned workouts cannot be confused with completed workouts.
- Generator inputs and outputs are explicit.
- Workout logs are protected from plan regeneration side effects.

---

## 18. Open Decisions Before Build

Must decide before architecture freeze:

1. Confirm Supabase as final backend.
2. Confirm Expo/React Native as final frontend.
3. Confirm exact exercise seed list.
4. Confirm legal/privacy deletion policy.
5. Confirm whether analytics provider is used or skipped.
6. Confirm final copy for safety disclaimers.
7. Confirm whether Apple/Google sign-in is MVP or fast-follow.

---

## 19. PRD Quality Rules Going Forward

To keep this document build-ready:

- Maintain one canonical MVP definition.
- Do not duplicate requirements across multiple sections.
- Keep section numbering consistent.
- Preview Markdown before sending to Codex.
- Move non-MVP ideas to appendices or backlog.
- Do not add AI fields to MVP schemas unless AI becomes MVP.
- Update dependent sections when a decision changes.

---

## 20. Appendix: Post-MVP Backlog

Post-MVP candidates:

- Strict food search
- USDA FoodData Central integration
- Open Food Facts integration
- Barcode scanning
- Custom foods
- Recent foods
- Exercise videos
- Wearable integrations
- Advanced progression models
- Advanced periodization
- AI chat coach
- Natural-language logging
- Coach/admin portal
- Social/community features
- Localization
- Tablet-optimized layouts
- Data export dashboards

