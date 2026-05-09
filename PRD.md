# Hypertrophy & Muscle Gain Coach

## Product Requirements Document (PRD) - v0.2 Working Draft

**Date:** May 09, 2026
**Status:** Working draft
**Source baseline:** BRD v0.1, January 02, 2026
**Purpose:** Convert the original BRD into a developer-ready PRD that can eventually be reviewed by Codex or handed to an engineer.

---

## 1. Product Summary

A science-informed hypertrophy training and nutrition tracking app that creates and adapts a weekly workout plan based on the user's goal, experience level, schedule, and equipment availability by day.

The app tracks workout performance, recovery signals, and nutrition adherence, then provides explainable recommendations for progressive overload, exercise substitutions, set/volume changes, and nutrition consistency.

**Primary promise:** Stay on track for muscle growth with a plan that fits your life and evolves with your results.

---

## 2. Product Positioning

This app is not just a workout logger. It is a guided hypertrophy coach for beginner-to-intermediate users who want a clear path to building muscle without needing to become experts first.

The target user is likely not an advanced bodybuilding expert. They are more likely:

* A beginner who wants to start lifting correctly
* An early intermediate lifter who has trained before but wants better structure
* Someone who wants to build muscle but does not know how to design a program
* Someone who wants guidance without manually managing spreadsheets, volume landmarks, progression rules, and nutrition math
* Someone who wants to learn the reasoning behind recommendations over time
* Someone who may train with mixed equipment depending on the day

The product should feel like a coach that teaches while guiding. It should provide enough structure for users to trust the plan, while still giving them control when life, equipment, fatigue, or preferences change.

The product should be more flexible than a static program, more structured than a generic workout tracker, and less overwhelming than a fully manual nutrition/training spreadsheet.

### 2.1 Product experience goal

The user should feel:

* “I know what to do today.”
* “I understand why the app is recommending this.”
* “I can change the plan without breaking everything.”
* “I am learning how hypertrophy training works as I go.”
* “This app adapts to my real life, not just an ideal gym schedule.”

### 2.2 Product experience anti-goals

The app should not feel like:

* A complex bodybuilding spreadsheet
* A generic workout list with no logic
* A black-box AI coach that changes things without explanation
* A punishment system when users miss workouts
* A medical, rehabilitation, or injury-treatment app
* A platform only advanced lifters can understand

---

## 3. MVP Product Principles

### 3.1 Guided simplicity first

The app should make the next action obvious. Most users should not need to understand programming theory, weekly set targets, periodization, or RIR before starting.

The product can expose deeper concepts gradually, but the default experience should be simple and guided.

### 3.2 Teach while coaching

The app should explain key concepts at the moment they matter.

Examples:

* Explain progressive overload when the app recommends adding reps or load.
* Explain RIR only when the user enables it or sees it for the first time.
* Explain deloads only when fatigue/performance signals suggest one.
* Explain volume only when recommending adding or removing sets.

### 3.3 User control first

The app may suggest changes, but the user should remain in control. Users can edit exercises, sets, reps, schedule, equipment, nutrition mode, goals, and reminders.

### 3.4 Explain every recommendation

Every recommendation should include a short explanation using user-understandable language.

Example:

> You hit the top of the target rep range for all sets last week, so this app recommends increasing load by 5 lb next time.

### 3.5 Avoid fake precision

The app should not pretend to know more than it does. Recovery, soreness, and RIR are subjective. Recommendations should be framed as suggestions, not absolute instructions.

### 3.6 Beginner safety defaults

Beginner plans should prioritize consistency, conservative progression, manageable volume, simple exercise selection, and form-first guidance.

### 3.7 Intermediate-friendly depth

Intermediate users should have access to more granular hypertrophy tools, including optional RIR, specialization muscles, higher volume ranges, exercise locks, and progression preferences.

However, these tools should not be required to use the app successfully.

### 3.8 Progressive disclosure

Advanced controls should be available but not forced into the default flow.

Examples of progressive disclosure:

* Hide RIR by default for beginners.
* Show simple effort labels before numeric RIR.
* Keep volume explanations plain-language unless the user opens “learn more.”
* Allow intermediate users to expand details on weekly sets, movement coverage, and progression rules.

### 3.9 MVP must remain buildable

MVP should avoid overbuilding complex AI autonomy, photo analysis, social features, wearable integrations, and advanced periodization until the core loop works.

---

## 4. Target Users / Personas

### 4.1 Beginner lifter

**Profile:** New to structured lifting or returning after a long break.
**Needs:** Simple workouts, conservative progressions, clear explanations, habit-building.
**Risks:** Overtraining, poor form, confusion, dropping off after soreness or missed days.
**MVP priority:** High.

Expected product behavior:

* Keep workouts simple and confidence-building.
* Avoid excessive terminology.
* Avoid aggressive volume or failure-based training.
* Prefer simple exercise substitutions.
* Explain what to do today more than why every advanced detail matters.

### 4.2 Early/intermediate hypertrophy learner

**Profile:** Has some lifting experience and wants muscle growth, but does not fully know how to structure training, progress over time, or adjust when life gets in the way.
**Needs:** A clear plan, progression guidance, workout logging, nutrition targets, and explanations that help them learn.
**Risks:** Program hopping, inconsistent progression, doing too much too soon, not eating enough protein/calories, overcomplicating details.
**MVP priority:** Very high. This is likely the primary target user.

Expected product behavior:

* Provide a structured weekly plan.
* Explain progression decisions.
* Offer RIR as optional, not mandatory.
* Help the user understand why load, reps, sets, or exercises are changing.
* Provide enough flexibility to edit the plan without making the user feel like they broke the program.

### 4.3 Hybrid equipment user

**Profile:** Trains in different environments depending on the day: gym, home, dumbbells, bodyweight, machines, or mixed equipment.
**Needs:** Equipment calendar, day-specific exercise selection, easy substitutions.
**Risks:** Plans becoming unusable when equipment changes.
**MVP priority:** Very high because this is a key differentiator.

Expected product behavior:

* Respect day-level equipment availability.
* Make substitutions easy and explainable.
* Avoid assuming every user has full gym access every day.
* Preserve the training intent even when equipment changes.

### 4.4 Advanced lifter / bodybuilding expert

**Profile:** Experienced lifter who already understands programming, progression, volume, intensity, and nutrition.
**Needs:** Highly customizable controls, detailed analytics, advanced progression preferences, periodization.
**Risks:** May find MVP too simple.
**MVP priority:** Low to medium.

MVP stance:

* The app should not be optimized primarily for advanced experts.
* Advanced users may still benefit from tracking and customization.
* Advanced features should not make the beginner/intermediate experience harder.

---

## 5. MVP Scope & Cut Line

### 5.1 MVP strategy

The MVP should prove the core product loop:

1. User completes onboarding.
2. App generates a usable hypertrophy plan based on schedule and equipment by day.
3. User completes and logs workouts.
4. App gives simple, explainable progression recommendations.
5. User logs enough nutrition to support their goal.
6. App summarizes progress weekly and suggests small next steps.

The MVP should not attempt to build every possible fitness feature. The first version should prove that a beginner-to-intermediate user can trust the app to guide training and nutrition without needing to design a program manually.

The core MVP differentiator is:

> A guided hypertrophy plan that adapts to the user's schedule, equipment availability by day, logged performance, and nutrition adherence.

---

### 5.2 MVP must-have

These features are required for the first useful version of the product.

#### Account and profile

* Account creation/login
* User profile with height, weight, experience level, units, and goal
* Goal selection: bulk, cut, recomp
* Basic privacy/account settings

#### Onboarding

* Experience level selection
* Goal selection
* Body metrics input
* Training schedule setup: 2 to 6 days per week
* Day-level equipment calendar
* Basic preferences/limitations
* Nutrition mode selection
* Plan review before generation

#### Program generation

* Rules-based weekly plan generator
* Split selection based on days/week and experience level
* Exercise selection constrained by equipment available that day
* Movement-pattern coverage checks
* Beginner/intermediate starting volume defaults
* Plain-language “why this plan” explanation

#### Workout plan and editor

* Weekly plan screen
* Workout detail screen
* Ability to edit exercises, sets, reps, rest time, and training days
* Ability to change equipment for a specific day
* Basic exercise substitution flow
* Plan version history
* Revert to previous plan version

#### Workout logging

* Start workout
* Log weight and reps for each set
* Optional RIR field
* Skip set/exercise
* Add set
* Copy previous set
* Finish workout
* Post-workout check-in: fatigue, soreness, mood, optional notes
* Preserve in-progress workout data

#### Progression engine

* Double-progression logic
* Add reps recommendation
* Add load recommendation
* Hold load recommendation
* Reduce load recommendation when performance/pain suggests caution
* Conservative beginner guardrails
* Recommendation explanation
* User can accept, modify, or ignore recommendations

#### Nutrition

* Initial calorie and macro targets
* Daily macro dashboard
* Strict mode food search/logging through at least one provider or provider abstraction
* Lite mode quick-add entries or saved meals
* Edit/delete food logs
* Saved meals/favorites

#### Today and weekly review

* Today dashboard showing workout, nutrition progress, and next best action
* Weekly review showing workout adherence, nutrition adherence, weight trend, and recommendations
* Basic progress trends: body weight, workouts completed, recent improvements

#### Edge-case handling

* Missed workout recovery flow
* Partial workout handling
* Equipment change revalidation
* Goal change handling
* Pain/discomfort flag behavior
* Nutrition search failure fallback
* Offline or interrupted workout preservation, at least locally
* No-shame recovery language

#### Notifications

* Basic configurable workout reminders
* Basic configurable nutrition reminders
* Quiet hours
* Max reminders/day
* Notification permission denied handling

---

### 5.3 MVP should-have

These features are valuable for MVP but can be cut if they slow down the first build too much.

#### Training should-have

* Rest timer
* Exercise notes
* Warm-up set marker
* Estimated workout duration
* Basic PR detection
* Muscle-group volume summary
* Lock exercise/day controls
* Substitution reason tracking

#### Nutrition should-have

* Multiple food database providers through abstraction layer
* Custom food creation
* Macro shortcut creation
* Meal templates
* Recent foods
* Copy yesterday’s meal

#### Coaching should-have

* “Learn more” expandable explanations
* Beginner-friendly concept cards
* Recommendation history
* Simple readiness categories: good, caution, poor
* Weekly coach summary

#### UX should-have

* Empty states for all main screens
* Friendly onboarding progress indicator
* In-app reminder fallback if push notifications are disabled
* Basic onboarding edit/review screen

---

### 5.4 Post-MVP / V1 candidates

These features should not block MVP. They are good candidates after the core loop is proven.

#### Training V1 candidates

* Barcode-style exercise QR scanning, if ever needed
* Custom exercise creation
* Exercise demo videos
* Advanced exercise instruction library
* Advanced periodization blocks
* Body-part specialization phases
* Multiple progression models beyond double progression
* Advanced volume landmarks
* Advanced fatigue/readiness scoring
* Wearable recovery integrations
* Apple Health / Google Fit integration
* Calendar app integration
* Coach/admin portal

#### Nutrition V1 candidates

* Barcode scanning for packaged foods
* Paid nutrition database provider
* Meal plan generation
* Recipe builder
* Grocery lists
* Restaurant meal estimation
* Nutrition label photo parsing
* Automated target adjustments based on weight trend

#### AI V1 candidates

* AI-generated plan explanations beyond templated rules
* AI chat coach
* AI auto-apply mode
* AI-generated substitutions with user approval
* AI weekly narrative summaries
* Natural-language food logging
* Natural-language workout edits

#### Product V1 candidates

* Subscriptions/payments
* Referral system
* Social/community features
* Public profile
* Challenges
* Achievements/badges beyond simple progress highlights
* Data export dashboards

---

### 5.5 Explicitly not in MVP

The following should not be built in MVP unless the strategy changes.

* Photo storage
* Physique photo comparison
* Computer vision form checks
* Medical diagnosis
* Injury rehabilitation programming
* Eating disorder coaching
* Extreme weight-loss planning
* Fully autonomous AI plan changes
* Automatic deloads without user confirmation
* Public social feed
* Coach marketplace
* Advanced bodybuilding contest-prep tools
* Complex macro cycling
* Complex periodization models
* Wearable-driven readiness scoring
* Multi-user/team coaching

---

### 5.6 MVP cut-line by product loop

| Product loop    | MVP requirement                                      | Cut for later                                        |
| --------------- | ---------------------------------------------------- | ---------------------------------------------------- |
| Onboarding      | Collect enough data to generate first plan           | Deep preference quiz, advanced goal modeling         |
| Plan generation | Rules-based plan using schedule + equipment calendar | AI-created periodized blocks                         |
| Workout logging | Log sets, reps, weight, optional RIR                 | Advanced analytics during workout                    |
| Progression     | Double progression with explanations                 | Multiple progression systems                         |
| Substitution    | Same intent + equipment-compatible replacements      | AI-generated novel substitutions                     |
| Nutrition       | Macro targets + strict/lite logging                  | Meal plans, barcode, recipes, grocery lists          |
| Coaching        | Recommendation cards + weekly review                 | Full AI chat coach                                   |
| Progress        | Adherence, weight trend, recent PRs                  | Advanced dashboards and predictive analytics         |
| Notifications   | Configurable reminders                               | Behavioral automation campaigns                      |
| Offline/data    | Preserve workout logs locally                        | Full multi-device offline-first system if too costly |

---

### 5.7 MVP risk areas

The largest MVP risks are:

1. **Scope creep**

   * The app can easily become a full fitness platform. MVP should stay focused on the guided training/nutrition loop.

2. **Program generator complexity**

   * The generator should start rules-based, not AI-magical. Codex should review whether the rules are implementable.

3. **Nutrition database complexity**

   * Strict nutrition logging can become expensive and difficult. Lite mode is important because it gives users a fallback.

4. **Offline sync complexity**

   * MVP must preserve in-progress workout logs, but full multi-device conflict resolution may be simplified initially.

5. **Too much advanced terminology**

   * The app should teach concepts gradually. Beginner/intermediate users should not feel like they need to understand every training variable.

6. **Unsafe recommendations**

   * The app must avoid medical advice, extreme dieting, aggressive progression after pain, or pretending subjective recovery data is precise.

---

### 5.8 MVP release definition

The MVP is ready when a user can complete this full path without manual developer intervention:

1. Create an account.
2. Complete onboarding.
3. Generate a weekly plan based on schedule and equipment by day.
4. Start and complete at least one workout.
5. Log sets with weight and reps.
6. Receive a simple progression recommendation after enough data exists.
7. Log nutrition in either strict or lite mode.
8. View Today dashboard.
9. Complete a weekly review.
10. Edit the plan and preserve prior logs.
11. Recover from at least one missed or partial workout.

---

### 5.9 Codex implementation boundary

When Codex reviews or eventually implements this product, it should treat the MVP must-have list as the implementation target.

Codex should not implement post-MVP features unless explicitly asked.

Codex should flag any MVP feature that is too vague, too large, or dependent on unresolved product decisions.

Codex should separate:

* Required MVP functionality
* Nice-to-have functionality
* Technical prerequisites
* Future architecture hooks
* Features that should be deferred

---

## 6. Core User Journey

### 6.1 Onboarding

**Goal:** Create a usable weekly training and nutrition plan in 3 to 5 minutes.

Required onboarding inputs:

1. Experience level

   * Beginner
   * Intermediate

2. Primary goal

   * Bulk
   * Cut
   * Recomp

3. Current body metrics

   * Height
   * Weight
   * Optional waist
   * Optional neck/hip

4. Training schedule

   * Days per week: 2 to 6
   * Preferred training days
   * Session length

5. Equipment calendar

   * Equipment available for each selected training day
   * Example: Monday = gym, Wednesday = dumbbells only, Friday = bodyweight only

6. Preferences and constraints

   * Disliked exercises
   * Injuries/limitations
   * Focus muscles
   * Exercises user wants to keep or avoid

7. Nutrition mode

   * Strict
   * Lite

8. Reminder preferences

   * Workout reminders
   * Nutrition reminders
   * Quiet hours

Output:

* Weekly workout plan
* First workout ready to start
* Calorie and macro targets
* Explanation of why the plan was created

---

## 7. MVP User Flows & Screen Requirements

### 7.1 Purpose of this section

This section defines the MVP user experience at the screen and flow level. The goal is to make the product understandable for a beginner-to-intermediate user and specific enough for designers, developers, or Codex to review.

The MVP should prioritize the following user loops:

1. Onboard and generate a plan
2. See what to do today
3. Complete and log a workout
4. Log nutrition simply
5. Review weekly progress
6. Accept, modify, or ignore recommendations
7. Edit the plan when life changes

---

### 7.2 MVP information architecture

Primary navigation should be simple and likely include 4 to 5 main areas.

Recommended MVP tabs:

1. **Today**

   * Daily workout, nutrition summary, reminders, next best action
2. **Plan**

   * Weekly workout plan, plan editor, exercise substitutions, equipment calendar
3. **Log** or **Nutrition**

   * Food logging, saved meals, daily macro progress
4. **Progress**

   * Weekly review, weight trend, workout adherence, PRs, recommendation history
5. **Settings**

   * Profile, goals, equipment, reminders, units, privacy, account

MVP should avoid too many tabs or advanced dashboards. The default experience should answer:

> What should I do today, and why?

---

### 7.3 Global UX requirements

#### Required global behaviors

* User should always be able to return to Today/Home.
* User should always be able to edit the current plan from the Plan area.
* User should be able to undo or revert meaningful plan changes.
* User should see plain-language explanations for recommendations.
* User should not be punished or shamed for missed workouts or incomplete logging.
* App should save workout progress continuously during a live workout.
* App should support offline workout logging in MVP if technically feasible.

#### Tone requirements

The app should use supportive, coach-like language.

Good examples:

* “Let’s keep the same weight and try to add 1 rep next time.”
* “You missed this workout. Want to move it, skip it, or adjust the week?”
* “This change keeps your plan balanced.”

Avoid:

* “You failed.”
* “You are behind.”
* “Bad workout.”
* Overly technical language without explanation.

---

### 7.4 First launch / welcome flow

#### Screen: Welcome

Purpose:

* Explain the product promise quickly.
* Set expectation that the app will create a personalized training and nutrition plan.

Primary content:

* App name/logo
* Short value proposition
* Primary CTA: “Build my plan”
* Secondary CTA: “Log in”

Example copy:

> Build muscle with a plan that adapts to your schedule, equipment, progress, and nutrition.

Required actions:

* Start onboarding
* Log in to existing account

Acceptance criteria:

* User can start onboarding from the welcome screen.
* User can log into an existing account.
* User understands the app is for guided hypertrophy training and nutrition.

---

### 7.5 Account creation flow

#### Screen: Create account

Purpose:

* Create a user account so plans, workouts, nutrition, and progress can sync.

Required fields:

* Email
* Password

Optional MVP options:

* Google sign-in
* Apple sign-in, especially if mobile/iOS is targeted

UX notes:

* Account creation should be low-friction.
* If the product supports a demo/local mode later, it can be post-MVP.

Acceptance criteria:

* User can create an account with email/password.
* User sees clear validation errors for invalid email or weak password.
* User can recover from account creation errors.
* User can proceed to onboarding after account creation.

---

### 7.6 Onboarding flow

The onboarding flow should take approximately 3 to 5 minutes. It should collect only the information needed to generate the first plan.

#### Screen 1: Experience level

Purpose:

* Set training defaults, safety rules, terminology, and progression aggressiveness.

Options:

* Beginner
* Intermediate

Helper text:

* Beginner: “New to structured lifting or returning after a long break.”
* Intermediate: “You have lifted before and want better structure and progression.”

Acceptance criteria:

* User must select an experience level.
* Selection influences plan defaults and terminology.

---

#### Screen 2: Goal

Purpose:

* Set training/nutrition framing.

Options:

* Build muscle while gaining weight / Bulk
* Build muscle while maintaining or slowly changing weight / Recomp
* Keep or build muscle while losing fat / Cut

Required inputs:

* Goal type
* Current weight
* Optional target weight or rate of change

Acceptance criteria:

* User can choose bulk, recomp, or cut.
* App stores current weight.
* App can calculate initial nutrition targets later in onboarding.

---

#### Screen 3: Body metrics

Purpose:

* Collect enough information for nutrition targets and progress tracking.

Required fields:

* Height
* Weight, if not already collected

Optional fields:

* Age or age range
* Waist
* Neck
* Hip

UX notes:

* Explain that measurements are optional and can be added later.
* Avoid making the app feel like a body-composition judgment tool.

Acceptance criteria:

* User can complete the screen with required fields only.
* User can skip optional measurements.
* User can choose units.

---

#### Screen 4: Training schedule

Purpose:

* Define how many days the user wants to train and which days are available.

Required fields:

* Days per week: 2 to 6
* Preferred training days
* Session length target

Session length options:

* 30 minutes
* 45 minutes
* 60 minutes
* 75+ minutes

Acceptance criteria:

* User can select 2 to 6 training days.
* User can choose specific days of the week.
* User can choose a target session length.
* App warns gently if schedule and session length may be difficult for the selected goal.

---

#### Screen 5: Equipment calendar

Purpose:

* Capture equipment availability per training day.

Required fields:

* Equipment available for each selected training day

Default equipment profiles:

* Full gym
* Dumbbells only
* Bodyweight only
* Bands only
* Barbell only
* Custom / mixed equipment

UX notes:

* This screen is a key differentiator and should be simple.
* User should be able to copy one day’s equipment to other days.
* User should be able to edit equipment later.

Acceptance criteria:

* User can assign different equipment to different training days.
* User can copy equipment settings across days.
* App uses day-level equipment during plan generation.

---

#### Screen 6: Preferences and limitations

Purpose:

* Prevent bad exercise matches and support better personalization.

Inputs:

* Disliked exercises
* Preferred exercises
* Focus muscles
* Injuries/limitations
* Movements to avoid

Focus muscle options:

* Chest
* Back
* Shoulders
* Arms
* Glutes
* Legs
* Core
* No specific focus

UX notes:

* Keep this skippable.
* Make clear this is not medical advice.
* For injuries, use language like “limitations” and “movements to avoid,” not diagnosis/treatment.

Acceptance criteria:

* User can skip this screen.
* User can add at least one disliked exercise or limitation.
* Generator respects limitations where possible.

---

#### Screen 7: Nutrition mode

Purpose:

* Choose how detailed nutrition tracking should be.

Options:

* Strict tracking
* Lite tracking

Strict helper text:

> Search foods and track calories, protein, carbs, and fat more precisely.

Lite helper text:

> Use saved meals, quick protein entries, and macro shortcuts to stay consistent without logging every detail.

Acceptance criteria:

* User can select strict or lite mode.
* User can switch modes later.
* App generates calorie and macro targets either way.

---

#### Screen 8: Reminder preferences

Purpose:

* Set workout and nutrition reminder defaults without being annoying.

Inputs:

* Enable workout reminders: yes/no
* Enable nutrition reminders: yes/no
* Preferred reminder time
* Quiet hours
* Max reminders per day

Acceptance criteria:

* User can skip reminders.
* User can enable reminders.
* App stores quiet hours and reminder preferences.

---

#### Screen 9: Review inputs

Purpose:

* Let user confirm onboarding choices before plan generation.

Content:

* Goal summary
* Training days
* Equipment by day
* Experience level
* Nutrition mode
* Focus muscles or limitations

Primary CTA:

* “Generate my plan”

Acceptance criteria:

* User can review selections.
* User can go back and edit previous steps.
* User can generate a plan.

---

### 7.7 Plan generation/loading state

#### Screen: Generating plan

Purpose:

* Give the user confidence while the app generates the initial plan.

Content examples:

* “Building your weekly plan…”
* “Matching exercises to your equipment…”
* “Balancing muscle groups…”
* “Setting starting volume…”

Acceptance criteria:

* User sees a loading state while plan generation runs.
* If generation succeeds, user is taken to the Plan Ready screen.
* If generation fails, user sees a clear retry path.

---

### 7.8 Plan Ready / first plan review flow

#### Screen: Your plan is ready

Purpose:

* Show the generated plan and explain why it was created.

Required content:

* Weekly split name
* Training days
* Estimated session durations
* Equipment used by day
* Muscle/movement coverage summary
* Nutrition target summary
* Plain-language “why this plan” explanation

Primary actions:

* Start first workout
* View full plan
* Edit plan

Acceptance criteria:

* User can see the generated weekly plan.
* User can understand why this plan was selected.
* User can start the first workout.
* User can edit the plan before starting.

---

### 7.9 Today dashboard

#### Screen: Today

Purpose:

* Give the user a simple daily command center.

Required content:

* Today’s workout, if scheduled
* Rest day message, if no workout
* Nutrition progress summary
* Current macro/protein progress
* Next best action coach card
* Reminder or streak/adherence context, without shaming

Possible states:

1. Training day, workout not started
2. Training day, workout in progress
3. Training day, workout completed
4. Rest day
5. Missed workout from previous day
6. No plan generated yet
7. Offline mode

Primary actions:

* Start workout
* Resume workout
* Log food
* View plan
* Review recommendation

Acceptance criteria:

* User can immediately see what to do today.
* User can start or resume today’s workout.
* User can log nutrition from Today.
* User can recover from missed workout state.

---

### 7.10 Workout player flow

#### Screen: Workout overview

Purpose:

* Preview the workout before starting.

Required content:

* Workout name
* Estimated duration
* Exercise list
* Sets and rep ranges
* Equipment needed
* Last performance summary, if available

Actions:

* Start workout
* Substitute exercise
* Edit workout

Acceptance criteria:

* User can review workout before starting.
* User can start workout.
* User can substitute an exercise before starting.

---

#### Screen: Live workout logging

Purpose:

* Let user log sets quickly without friction.

Required content per exercise:

* Exercise name
* Target sets
* Target rep range
* Previous performance
* Current set inputs
* Rest timer
* Notes
* Substitute button

Required set fields:

* Weight or resistance
* Reps

Optional fields:

* RIR
* Notes
* Pain/discomfort flag

Core actions:

* Log set
* Copy previous set
* Add set
* Skip set
* Mark warm-up set
* Start/skip rest timer
* Substitute exercise
* Finish workout

Acceptance criteria:

* User can log all planned sets.
* User can skip a set.
* User can add an extra set.
* User can copy prior set values.
* Workout progress is saved during the session.
* User can close and resume an in-progress workout.

---

#### Screen: Exercise substitution during workout

Purpose:

* Let user replace an exercise without breaking the program.

Required content:

* Original exercise
* Reason prompt, optional
* Ranked substitutions
* Equipment-compatible options
* Explanation of why substitution is similar

Substitution reason options:

* Equipment unavailable
* Pain/discomfort
* Too hard
* Too easy
* Machine/equipment occupied
* Preference
* Time constraint
* Other

Acceptance criteria:

* User can substitute an exercise during a workout.
* App prioritizes substitutions with the same movement intent and compatible equipment.
* App preserves sets/reps where reasonable.
* App stores substitution reason if provided.

---

#### Screen: Workout complete

Purpose:

* Summarize the workout and collect recovery data.

Required content:

* Exercises completed
* Sets completed
* PRs or improvements
* Skipped exercises/sets
* Estimated volume summary, optional

Required actions:

* Finish workout
* Complete quick check-in

Post-workout check-in:

* Fatigue rating
* Soreness rating
* Mood rating
* Optional notes
* Optional pain/discomfort follow-up

Acceptance criteria:

* User can complete workout.
* User can submit or skip check-in.
* App saves workout logs.
* App uses workout data for future recommendations.

---

### 7.11 Missed workout flow

#### Screen/state: Missed workout

Purpose:

* Help the user recover without guilt or overcompensation.

Trigger:

* Scheduled workout day passes without completion.

Options:

* Move workout to next available day
* Skip and continue plan
* Replace next workout with missed workout
* Regenerate rest of week

UX notes:

* Avoid shaming language.
* Do not automatically double future volume.
* Explain impact of each choice simply.

Acceptance criteria:

* User sees missed workout recovery options.
* App does not automatically add missed volume to the next workout.
* App updates the plan based on selected recovery option.
* App avoids creating high-fatigue back-to-back workouts where possible.

---

### 7.12 Plan tab / weekly plan screen

#### Screen: Weekly plan

Purpose:

* Let user view, understand, and edit the current plan.

Required content:

* Week view
* Training days
* Workout names
* Exercise list per day
* Equipment profile per day
* Estimated session duration
* Weekly muscle/movement coverage summary

Actions:

* Start workout
* Edit workout
* Change day
* Change equipment for day
* Lock exercise or day
* Regenerate plan
* View plan history

Acceptance criteria:

* User can view the weekly plan.
* User can edit exercises, sets, reps, and schedule.
* User can change equipment for a specific day.
* User can regenerate the plan after meaningful changes.
* User can revert to a prior plan version.

---

### 7.13 Plan editor flow

#### Screen: Edit plan/workout

Purpose:

* Give the user control without requiring advanced programming knowledge.

Editable fields:

* Training day
* Exercise
* Exercise order
* Sets
* Rep range
* Rest time
* Equipment profile
* Notes
* Lock/unlock exercise

UX requirements:

* App should warn if an edit creates an obvious imbalance or equipment conflict.
* App should explain warnings in plain language.
* App should allow the user to proceed when safe.
* App should store user edits as planning signals.

Example warning:

> Removing this row leaves your plan with no back exercise this week. You can continue, or choose a replacement pull exercise.

Acceptance criteria:

* User can edit workout details.
* App validates obvious conflicts.
* App stores edit history.
* User can undo/revert meaningful changes.

---

### 7.14 Recommendation flow

#### Screen/component: Recommendation card

Purpose:

* Present adaptive coaching suggestions in a way the user understands and controls.

Recommendation types:

* Add load
* Add reps
* Hold load
* Reduce load
* Add set
* Remove set
* Substitute exercise
* Deload suggestion
* Nutrition target adjustment
* Schedule adjustment

Required content:

* Recommendation title
* Suggested change
* Why this is recommended
* Inputs used
* Confidence/caution note, if relevant

Actions:

* Accept
* Modify
* Ignore
* Learn more

Acceptance criteria:

* User can accept, modify, or ignore each recommendation.
* App explains why the recommendation exists.
* Accepted recommendations update the plan or targets.
* Ignored recommendations are logged as feedback.
* Modified recommendations are stored as user preference signals.

---

### 7.15 Nutrition dashboard flow

#### Screen: Nutrition dashboard

Purpose:

* Help user understand daily nutrition progress without overwhelming them.

Required content:

* Calories target and remaining
* Protein target and remaining
* Carbs target and remaining
* Fat target and remaining
* Meals logged today
* Quick-add actions
* Nutrition mode indicator: strict or lite

Primary actions:

* Search food
* Quick add
* Add saved meal
* Edit food log
* Switch nutrition mode

Acceptance criteria:

* User can view daily macro targets.
* User can log food from the dashboard.
* Dashboard updates after each log.
* User can edit or delete logged items.

---

### 7.16 Strict food logging flow

#### Screen: Food search

Purpose:

* Let strict-mode users search a food database and log precise entries.

Required content:

* Search input
* Food results
* Serving size selector
* Nutrition preview
* Meal category

Actions:

* Search
* Select food
* Adjust serving
* Add to log
* Save as favorite

Error states:

* No results found
* Provider unavailable
* Missing nutrition data

Acceptance criteria:

* User can search for a food.
* User can select a result.
* User can adjust serving size.
* User can add food to daily log.
* App handles no-result and provider-error states.

---

### 7.17 Lite nutrition logging flow

#### Screen: Lite quick add

Purpose:

* Let users track nutrition adherence quickly without detailed food search.

Logging options:

* Add saved meal
* Add protein serving
* Add carb serving
* Add fat serving
* Add custom macro shortcut
* Add calories only, if allowed

Example saved meals:

* Protein shake
* Usual breakfast
* Greek yogurt
* Chicken/rice meal

Acceptance criteria:

* User can create a saved meal or macro shortcut.
* User can quick-add a saved item.
* User can edit quick-add values.
* Dashboard updates immediately after quick add.

---

### 7.18 Weekly review flow

#### Screen: Weekly review

Purpose:

* Summarize progress and guide next week.

Required content:

* Workouts scheduled vs completed
* Nutrition adherence
* Weight trend
* PRs/improvements
* Missed workouts
* Recovery trend
* Recommendations for next week

Primary actions:

* Accept recommendations
* Modify recommendations
* Keep plan unchanged
* Edit schedule/equipment

UX notes:

* The weekly review should reinforce progress and consistency.
* It should not overwhelm the user with advanced analytics.
* It should highlight the next best action.

Acceptance criteria:

* User can view weekly adherence.
* User can view relevant progress trends.
* User can accept or ignore next-week recommendations.
* App can create a new plan version based on accepted changes.

---

### 7.19 Progress screen

#### Screen: Progress

Purpose:

* Show meaningful trends over time.

MVP content:

* Body weight trend
* Workout adherence
* Training volume summary
* Recent PRs
* Exercise performance trend
* Nutrition adherence trend

Optional MVP content:

* Waist trend
* Muscle group volume summary
* Recommendation history

Acceptance criteria:

* User can see progress over time.
* User can view recent improvements.
* User can understand adherence trends.
* User can view at least basic body weight history.

---

### 7.20 Settings screens

#### Screen: Settings

Purpose:

* Let user manage preferences and account data.

Required settings:

* Profile
* Goal
* Training schedule
* Equipment profiles
* Nutrition mode
* Reminder preferences
* Units
* Privacy/data settings
* Account settings

Acceptance criteria:

* User can update profile details.
* User can change goal.
* User can update equipment availability.
* User can switch nutrition modes.
* User can update reminder preferences.
* User can change units.

---

### 7.21 Empty states

MVP should include clear empty states for:

* No plan generated yet
* No workout today
* No food logged today
* No progress data yet
* No recommendations yet
* No saved meals yet
* No search results

Empty states should include a clear next action.

Example:

> No saved meals yet. Create one for foods you eat often so logging is faster next time.

Acceptance criteria:

* Empty states are clear and actionable.
* Empty states avoid making the app feel broken.

---

### 7.22 Error and offline states

Required error/offline states:

* Food database unavailable
* Plan generation failed
* Sync failed
* Workout save conflict
* Network offline
* Login/session expired
* Notification permission denied

Workout logging priority:

* Workout logs should not be lost.
* If offline, save locally and sync later.
* If sync conflict occurs, preserve both records or ask user to resolve.

Acceptance criteria:

* User sees clear error messages.
* User has a retry path where appropriate.
* Workout logs are preserved during offline/error states.
* App does not silently discard user-entered data.

---

### 7.23 MVP screen inventory

| Area       | Screen                    |                   MVP priority |
| ---------- | ------------------------- | -----------------------------: |
| Onboarding | Welcome                   |                      Must-have |
| Onboarding | Create account / login    |                      Must-have |
| Onboarding | Experience level          |                      Must-have |
| Onboarding | Goal                      |                      Must-have |
| Onboarding | Body metrics              |                      Must-have |
| Onboarding | Training schedule         |                      Must-have |
| Onboarding | Equipment calendar        |                      Must-have |
| Onboarding | Preferences/limitations   |                      Must-have |
| Onboarding | Nutrition mode            |                      Must-have |
| Onboarding | Reminder preferences      |                    Should-have |
| Onboarding | Review inputs             |                      Must-have |
| Plan       | Plan ready                |                      Must-have |
| Today      | Today dashboard           |                      Must-have |
| Workout    | Workout overview          |                      Must-have |
| Workout    | Live workout logging      |                      Must-have |
| Workout    | Exercise substitution     |                      Must-have |
| Workout    | Workout complete/check-in |                      Must-have |
| Plan       | Weekly plan               |                      Must-have |
| Plan       | Plan editor               |                      Must-have |
| Coaching   | Recommendation card       |                      Must-have |
| Nutrition  | Nutrition dashboard       |                      Must-have |
| Nutrition  | Strict food search/log    | Must-have if strict mode ships |
| Nutrition  | Lite quick add            |   Must-have if lite mode ships |
| Progress   | Weekly review             |                      Must-have |
| Progress   | Progress trends           |                    Should-have |
| Settings   | Profile/settings          |                      Must-have |
| Settings   | Equipment profiles        |                      Must-have |
| Settings   | Reminder settings         |                    Should-have |
| Settings   | Privacy/account           |                      Must-have |

---

### 7.24 Overall user-flow acceptance criteria

* User can create an account and complete onboarding.
* User can generate a weekly hypertrophy plan.
* User can understand why the plan was generated.
* User can start and complete a workout.
* User can log sets with weight and reps.
* User can optionally log RIR.
* User can substitute an exercise.
* User can complete a post-workout check-in.
* User can log nutrition in the selected mode.
* User can view daily progress from Today.
* User can view weekly progress from Weekly Review.
* User can accept, modify, or ignore recommendations.
* User can edit their plan without losing prior logs.
* User can recover from missed workouts.
* User-entered workout data is not lost during app close, offline state, or sync issue.

---

## 8. Edge Cases & State Handling

### 8.1 Purpose of this section

This section defines how the MVP should behave when user behavior, data, network conditions, or plan assumptions do not follow the ideal path.

The goal is to protect user trust. The app should avoid losing data, avoid making confusing automatic changes, and avoid punishing the user when real life interrupts the plan.

Core principles:

* Preserve user-entered data.
* Explain what happened.
* Give the user a clear next action.
* Avoid shame or guilt language.
* Prefer safe/conservative recommendations.
* Never silently overwrite meaningful user choices.
* Never let plan logic become a black box.

---

### 8.2 Missed workout states

#### Trigger

A scheduled workout day passes without the workout being completed.

#### Possible causes

* User forgot
* User was too busy
* User was sick
* User intentionally skipped
* User trained outside the app
* User started but did not finish

#### Required app behavior

The app should not automatically punish the user, double the next workout, or assume the user failed.

The app should show a missed workout recovery state with options:

1. Move workout to next available day
2. Skip and continue plan
3. Replace next workout with missed workout
4. Mark as completed outside app
5. Regenerate the rest of the week

#### State handling rules

* If the user moves the workout, avoid creating high-fatigue back-to-back sessions where possible.
* If the user skips the workout, do not add all skipped volume to future workouts.
* If the user marks the workout completed outside the app, allow optional notes and estimated effort.
* If multiple workouts are missed, prioritize resuming consistency over making up volume.
* If the user repeatedly misses workouts, suggest reducing weekly training days or session length.

#### Example UX copy

> Looks like this workout was missed. Want to move it, skip it, or adjust the rest of the week?

#### Acceptance criteria

* Given a workout is missed, the user sees recovery options.
* Given a workout is missed, the app does not automatically double future volume.
* Given the user repeatedly misses workouts, the app may suggest a more realistic schedule.
* Given the user marks the workout completed outside the app, the app stores that state separately from a fully logged workout.

---

### 8.3 Partially completed workout

#### Trigger

User starts a workout but does not complete all planned exercises or sets.

#### Possible states

* Workout in progress
* Workout abandoned
* Workout partially completed and intentionally finished
* Workout partially completed due to app close/crash/offline state

#### Required app behavior

The app should preserve all completed set logs and ask the user how to treat the unfinished work.

Options:

1. Resume workout
2. Finish as partial workout
3. Discard uncompleted planned sets only
4. Move remaining exercises to another day, if appropriate

#### Progression rules

* Completed sets should count toward training history.
* Uncompleted sets should not be treated as failed sets unless the user marks them as attempted and failed.
* The progression engine should be cautious when using partial workout data.
* If an exercise was not reached because of time, the app may suggest shorter sessions or reordering.

#### Acceptance criteria

* User can resume an in-progress workout.
* User can finish a workout as partial.
* Completed sets are saved.
* Uncompleted planned sets are not incorrectly treated as poor performance.
* App can use repeated partial completions as a signal that workouts may be too long.

---

### 8.4 Skipped sets and skipped exercises

#### Trigger

User skips a planned set or exercise.

#### Required skip reasons

The app may ask for an optional reason:

* Time constraint
* Too fatigued
* Pain/discomfort
* Equipment unavailable
* Did not like exercise
* Exercise occupied/unavailable
* Too difficult
* Other

#### State handling rules

* A skipped set should be stored as skipped, not deleted.
* A skipped exercise should remain visible in workout history.
* Repeated skips should become planning signals.
* Skips due to pain should suppress progression recommendations for that exercise.
* Skips due to equipment should suggest substitution or equipment-calendar update.
* Skips due to time should suggest shorter workout design if repeated.

#### Acceptance criteria

* User can skip a set.
* User can skip an exercise.
* App stores skipped state and optional reason.
* Repeated skipped exercises can trigger a substitution recommendation.
* Pain-related skips prevent aggressive progression suggestions.

---

### 8.5 User changes equipment midweek

#### Trigger

User changes equipment availability for one or more upcoming training days.

#### Example

User originally planned Thursday as a full-gym day but changes it to dumbbells-only.

#### Required app behavior

The app should re-validate affected workouts and identify incompatible exercises.

Options:

1. Substitute incompatible exercises only
2. Regenerate affected workout
3. Regenerate rest of week
4. Keep plan unchanged and warn user

#### State handling rules

* Do not alter completed workouts.
* Do not overwrite locked exercises without confirmation.
* Preserve movement intent when substituting.
* Store the equipment change as a planning signal.
* Explain what changed.

#### Example UX copy

> Thursday now uses dumbbells only, so 3 exercises need substitutions. Want the app to update just those exercises?

#### Acceptance criteria

* User can change equipment for a specific day.
* App detects exercises incompatible with new equipment.
* App offers substitutions or regeneration.
* Completed workouts remain unchanged.
* Locked exercises are not overwritten without confirmation.

---

### 8.6 User changes schedule midweek

#### Trigger

User changes training days, number of weekly workouts, or session length during an active plan week.

#### Required app behavior

The app should ask whether the change applies to:

* This week only
* Future weeks only
* This week and future weeks

#### State handling rules

* Completed workouts remain attached to their actual completion dates.
* Upcoming workouts may be moved or regenerated.
* Missed workouts should not automatically become extra volume.
* If days/week decreases, prioritize preserving major movement coverage.
* If days/week increases, add volume conservatively.

#### Acceptance criteria

* User can update schedule midweek.
* App asks scope of schedule change.
* App preserves completed workout history.
* App updates future plan days based on user choice.

---

### 8.7 User changes goal

#### Trigger

User changes goal between bulk, cut, and recomp.

#### Required app behavior

The app should explain that goal changes may affect nutrition targets and progression aggressiveness.

Possible changes:

* Calorie target
* Macro targets
* Progression confidence
* Volume recommendations
* Weekly review interpretation

#### State handling rules

* Training plan should not necessarily regenerate immediately.
* Nutrition targets should be recalculated or manually reviewed.
* Progression engine should become more conservative during a cut.
* The app should preserve historical data under the prior goal.
* Create a goal-change event.

#### Example UX copy

> Changing from bulk to cut may lower your calorie target and make progression recommendations more conservative. Your existing workout plan can stay the same unless you want to adjust it.

#### Acceptance criteria

* User can change goal.
* App stores goal history.
* App recalculates or prompts user to review nutrition targets.
* App does not erase prior progress data.
* App adjusts future recommendation logic based on new goal.

---

### 8.8 User reports pain or injury limitation

#### Trigger

User reports pain during a workout, in a check-in, or in settings/preferences.

#### Required app behavior

The app should avoid medical diagnosis and avoid giving rehabilitation instructions.

The app may:

* Suggest stopping or avoiding the painful movement
* Recommend substituting the exercise
* Recommend reducing load or intensity
* Recommend consulting a qualified professional for persistent or severe pain
* Store the limitation for future planning

#### State handling rules

* Pain flag should block load-increase recommendations for that exercise.
* Repeated pain flags should trigger a substitution suggestion.
* Pain-related limitations should be respected by the generator.
* App should not claim to treat or diagnose injuries.

#### Example UX copy

> Since you reported discomfort, the app will avoid progressing this exercise for now. Consider choosing a substitute or getting professional guidance if pain continues.

#### Acceptance criteria

* User can report pain/discomfort.
* App stores pain flag.
* App avoids recommending progression for flagged exercise.
* App can recommend substitution.
* App does not provide diagnosis or medical treatment advice.

---

### 8.9 User repeatedly rejects recommendations

#### Trigger

User repeatedly ignores or rejects the same recommendation type.

Examples:

* User keeps rejecting load increases.
* User keeps rejecting added sets.
* User keeps rejecting deload suggestions.
* User keeps rejecting a substitution.

#### Required app behavior

The app should treat repeated rejection as feedback, not user error.

State handling rules:

* Store ignored/rejected recommendations.
* Reduce frequency of similar suggestions.
* Ask if the user wants to adjust preferences.
* Avoid repeatedly showing the same recommendation without new evidence.

#### Example UX copy

> You have skipped this suggestion a few times. Want to keep this exercise the same for now?

#### Acceptance criteria

* Ignored recommendations are logged.
* Repeated ignored recommendations influence future suggestions.
* User can mute or reduce similar recommendations.

---

### 8.10 User edits plan in a way that creates imbalance

#### Trigger

User removes or changes exercises such that the plan no longer covers key movement patterns or muscle groups.

Examples:

* Removes all pulling exercises
* Removes all lower-body work
* Adds too much direct arm work
* Changes every leg day to upper body

#### Required app behavior

The app should warn the user in plain language but still preserve user control unless the change creates a safety or data-integrity issue.

State handling rules:

* Show the impact of the edit.
* Offer a suggested replacement.
* Allow user to continue if they choose.
* Store the edit as a user preference signal.

Example warning:

> This removes your only back exercise for the week. For balanced training, consider adding another pull exercise.

#### Acceptance criteria

* App detects major plan imbalances.
* App warns the user before saving.
* App suggests a fix.
* User can still proceed when appropriate.

---

### 8.11 Exercise database has no valid substitution

#### Trigger

The app cannot find a suitable substitution for a movement intent and equipment profile.

#### Required app behavior

The app should clearly explain the limitation and provide the best fallback options.

Fallback hierarchy:

1. Same movement intent and same primary muscle
2. Same primary muscle with different movement pattern
3. Similar training effect with available equipment
4. Bodyweight fallback
5. Ask user to change equipment or manually choose

#### Acceptance criteria

* App handles no-perfect-substitution state.
* App explains why options are limited.
* App does not silently assign an unrelated exercise.
* User can manually choose or change equipment.

---

### 8.12 Nutrition search returns no results

#### Trigger

User searches for a food and the provider returns no useful result.

#### Required app behavior

Offer fallback actions:

* Try another search term
* Create custom food
* Quick add calories/macros
* Use saved meal
* Search another provider, if available

#### Acceptance criteria

* User sees a no-results state.
* User can recover without abandoning logging.
* User can create a custom entry or quick macro entry.

---

### 8.13 Nutrition provider unavailable

#### Trigger

USDA, Open Food Facts, or another provider is unavailable, slow, or returns an error.

#### Required app behavior

The app should not block all nutrition logging.

Fallback actions:

* Use recently logged foods
* Use saved meals
* Use quick-add macros
* Create custom entry
* Retry search

State handling rules:

* Show provider error in user-friendly language.
* Do not expose raw API errors to the user.
* Log provider failure for debugging.
* Preserve any food entry the user was editing.

#### Example UX copy

> Food search is unavailable right now. You can still use saved meals or quick-add macros.

#### Acceptance criteria

* User can still log nutrition when provider search fails.
* App shows a clear fallback path.
* App does not discard partially entered food data.

---

### 8.14 Incomplete nutrition day

#### Trigger

User logs some but not all nutrition for the day.

#### Required app behavior

The app should avoid assuming the user failed their targets.

State handling rules:

* Distinguish “not logged” from “not consumed.”
* Weekly nutrition adherence should be based on logged data with caveats.
* Reminders can encourage logging but should not shame.
* Lite mode should accept approximate adherence.

#### Acceptance criteria

* App distinguishes missing data from missed targets.
* App does not over-interpret incomplete logs.
* Weekly review reflects uncertainty when nutrition data is incomplete.

---

### 8.15 Weight trend conflicts with goal

#### Trigger

User's logged body weight trend does not align with selected goal over multiple check-ins.

Examples:

* Bulk goal but weight is flat or decreasing
* Cut goal but weight is increasing
* Recomp goal but weight changes faster than intended

#### Required app behavior

The app may suggest reviewing calorie targets or adherence.

State handling rules:

* Do not change calorie targets automatically without user approval.
* Require enough weigh-ins before identifying a trend.
* Explain that daily fluctuations are normal.
* Focus on trend over time, not single weigh-ins.

#### Acceptance criteria

* App requires multiple data points before trend-based nutrition recommendations.
* App explains weight trend recommendations.
* User can accept, modify, or ignore target changes.

---

### 8.16 Offline workout logging

#### Trigger

User starts or continues workout without network connection.

#### Required app behavior

Workout logging should continue offline if technically feasible.

State handling rules:

* Save workout data locally.
* Show offline indicator.
* Queue sync when connection returns.
* Do not block set logging.
* Do not lose set logs on app close.

#### Acceptance criteria

* User can log workout data offline.
* App clearly indicates offline state.
* Data syncs when connection returns.
* User-entered workout data is preserved.

---

### 8.17 Sync conflict

#### Trigger

Local workout, plan, or nutrition data conflicts with server data.

Examples:

* User edits workout on two devices.
* User logs offline, then edits same workout on another device.
* Plan regenerates while offline changes exist.

#### Required app behavior

The app should avoid silently overwriting user data.

Conflict resolution priority:

1. Preserve completed workout logs.
2. Preserve user-created food logs.
3. Preserve latest explicit user edits.
4. If conflict cannot be resolved safely, ask user.

Possible UX actions:

* Keep local version
* Keep server version
* Merge when safe
* Show both versions for manual resolution

#### Acceptance criteria

* App detects sync conflicts.
* App does not silently discard workout logs.
* App resolves simple conflicts automatically where safe.
* App asks user for manual resolution when needed.

---

### 8.18 Plan generation failure

#### Trigger

Generator cannot create a valid plan.

Possible causes:

* Too few training days for selected constraints
* Equipment too limited
* Too many disliked/blocked exercises
* Conflicting limitations
* Internal generator error

#### Required app behavior

The app should explain the issue and offer recovery paths.

Recovery options:

* Relax equipment constraints
* Remove some disliked exercises
* Reduce focus muscles
* Increase session length
* Choose simpler plan
* Contact support/report issue
* Retry generation

#### Acceptance criteria

* User sees a clear failure message.
* User understands what input may be blocking generation.
* User can revise inputs and retry.
* App does not leave user stuck in loading state.

---

### 8.19 Notification permission denied

#### Trigger

User declines push notification permission.

#### Required app behavior

The app should continue functioning normally.

State handling rules:

* Do not repeatedly nag for permission.
* Show reminders inside the app instead.
* Let user enable notifications later in settings.

#### Acceptance criteria

* App works without push permission.
* User can enable reminders later.
* User is not repeatedly interrupted by permission prompts.

---

### 8.20 User deletes data or account

#### Trigger

User requests data export, data deletion, or account deletion.

#### Required app behavior

The app should provide clear account/data controls.

MVP minimum:

* Delete account request
* Delete local/user data associated with account where required
* Privacy explanation
* Confirmation before destructive actions

Possible post-MVP:

* Full data export
* Download workout history
* Download nutrition logs

#### Acceptance criteria

* User can request account deletion.
* App confirms before destructive action.
* App explains what data will be deleted.
* App complies with applicable privacy requirements for target market.

---

### 8.21 Dangerous or inappropriate user goals

#### Trigger

User enters or implies unsafe goals or behaviors.

Examples:

* Extreme weight-loss target
* Very low calorie target
* Training through severe pain
* Excessive training frequency with high fatigue
* Requests for medical/injury treatment advice

#### Required app behavior

The app should set boundaries and use conservative, safety-oriented messaging.

State handling rules:

* Do not support extreme or unsafe target recommendations.
* Encourage consulting a qualified professional for medical issues, severe pain, eating disorder concerns, or extreme goals.
* Use safe defaults and refuse unsafe automation.

#### Acceptance criteria

* App can detect obviously unsafe targets or inputs.
* App does not generate extreme recommendations.
* App provides safe, non-diagnostic guidance.

---

### 8.22 Stale plan state

#### Trigger

User has not opened the app or logged workouts for a meaningful period.

Example thresholds:

* 7+ days inactive
* 14+ days inactive
* 30+ days inactive

#### Required app behavior

The app should not assume the old plan is still appropriate.

State handling rules:

* Welcome user back supportively.
* Ask whether schedule, equipment, goal, or fitness level changed.
* Offer to resume, adjust, or regenerate plan.
* Avoid aggressive progression after long inactivity.

#### Example UX copy

> Welcome back. Want to continue your old plan or refresh it based on your current schedule?

#### Acceptance criteria

* App handles returning users.
* App does not blindly continue aggressive progression after inactivity.
* User can resume or refresh plan.

---

### 8.23 Duplicate or accidental logs

#### Trigger

User accidentally logs the same set, workout, food, or weigh-in multiple times.

#### Required app behavior

The app should make duplicate data easy to correct.

State handling rules:

* Allow edit/delete for user logs.
* Consider duplicate detection for identical entries in a short time window.
* Confirm before deleting completed workouts.
* Preserve audit trail where needed.

#### Acceptance criteria

* User can edit or delete accidental logs.
* App prevents or warns about obvious duplicates where feasible.
* Deleted logs are handled safely.

---

### 8.24 State handling summary table

| Edge case                | App response                               | Must preserve                         |
| ------------------------ | ------------------------------------------ | ------------------------------------- |
| Missed workout           | Offer move/skip/replace/regenerate options | Plan history and adherence context    |
| Partial workout          | Let user resume or finish as partial       | Completed set logs                    |
| Skipped set/exercise     | Store skipped state and reason             | Workout history                       |
| Equipment change         | Revalidate upcoming workouts               | Completed workouts and locked choices |
| Schedule change          | Ask scope: this week/future/both           | Completed workout dates               |
| Goal change              | Recalculate/review targets                 | Historical goal/progress data         |
| Pain flag                | Avoid progression and suggest substitution | Pain context and exercise history     |
| Rejected recommendations | Reduce repeated suggestions                | User feedback signals                 |
| Plan imbalance edit      | Warn and suggest fix                       | User control and edit history         |
| No food results          | Offer custom/quick-add fallback            | Partial food entry                    |
| Provider unavailable     | Allow saved meals/quick-add                | Nutrition logging ability             |
| Offline workout          | Save locally and sync later                | All set logs                          |
| Sync conflict            | Merge or ask user                          | User-entered logs                     |
| Generation failure       | Explain and offer recovery                 | Onboarding inputs                     |
| Notifications denied     | Use in-app reminders                       | App functionality                     |
| Long inactivity          | Offer resume or refresh                    | Historical data                       |
| Duplicate logs           | Allow edit/delete                          | Data integrity                        |

---

### 8.25 Overall edge-case acceptance criteria

* User-entered workout logs are not lost due to app close, offline use, sync issues, or partial completion.
* User-entered nutrition logs are not lost due to provider errors or network issues.
* The app distinguishes skipped, missed, incomplete, and failed workout data.
* The app does not automatically apply major plan changes without user approval.
* The app explains recovery options when a plan cannot be followed as expected.
* The app avoids shaming language for missed workouts or incomplete nutrition logs.
* The app stores user edits, skips, substitutions, and ignored recommendations as future planning signals.
* The app handles equipment, schedule, and goal changes without corrupting prior plan history.
* The app avoids unsafe training or nutrition recommendations.
* The app provides a clear path forward from every major error state.

---

## 9. Program Generator Requirements

### 7.1 Generator purpose

The program generator creates a weekly hypertrophy plan that is:

* Matched to the user's experience level
* Matched to the user's goal: bulk, cut, or recomp
* Matched to the user's available training days
* Matched to the user's equipment availability by day
* Balanced across major movement patterns and muscle groups
* Conservative enough to be sustainable
* Editable by the user at any time
* Explainable in plain language

The MVP generator should be primarily rules-based. AI may help with explanations, summaries, or suggestions, but the core training logic should be deterministic and auditable.

---

### 7.2 Generator inputs

The generator must consider the following inputs.

#### User profile inputs

* Experience level: beginner or intermediate
* Goal type: bulk, cut, or recomp
* Current body weight
* Optional age or age range
* Optional sex, only if needed for nutrition formulas or future calculations
* Training units: pounds or kilograms

#### Schedule inputs

* Number of training days per week: 2 to 6
* Preferred training days
* Session length target
* Days the user cannot train
* Whether the user wants workouts evenly distributed when possible

#### Equipment inputs

* Equipment available per training day
* Equipment profiles, such as:

  * Full gym
  * Dumbbells only
  * Bodyweight only
  * Barbell only
  * Bands only
  * Mixed home gym
* Available load increments, if known

  * Example: dumbbells increase by 5 lb
  * Example: adjustable dumbbells increase by 2.5 lb

#### Preference and limitation inputs

* Disliked exercises
* Preferred exercises
* Injuries or limitations
* Pain/discomfort flags from prior sessions
* Focus muscles
* Exercises locked by the user
* Days locked by the user
* Maximum desired session length

#### Historical inputs, if available

* Last completed workout per exercise
* Recent set logs
* Recent skipped exercises
* Recent substitutions
* Recovery check-ins
* Adherence over the last 1 to 4 weeks
* Prior accepted or rejected recommendations

---

### 7.3 Generator outputs

The generator must output:

* Active weekly plan
* Plan version number
* Workout days
* Exercises per workout
* Exercise order
* Planned sets
* Target rep ranges
* Optional RIR guidance
* Rest guidance
* Estimated session duration
* Movement intent coverage
* Muscle-group volume summary
* Explanation of why the plan was generated
* Warnings or notes about constraints

Example explanation:

> This plan uses a 4-day upper/lower split because you selected 4 training days, intermediate experience, and a hypertrophy goal. Monday and Thursday use gym-based exercises, while Tuesday uses dumbbell-only substitutions because your equipment calendar says you train at home that day.

---

### 7.4 MVP split selection rules

The MVP should use predictable default templates. These defaults can later become more personalized.

| Days / week | Beginner default                                         | Intermediate default                                  | Notes                                                          |
| ----------- | -------------------------------------------------------- | ----------------------------------------------------- | -------------------------------------------------------------- |
| 2 days      | Full Body A/B                                            | Full Body A/B                                         | Prioritize major movement coverage and simplicity.             |
| 3 days      | Full Body A/B/C                                          | Full Body A/B/C or Upper/Lower/Full                   | Use full body by default unless user prefers split training.   |
| 4 days      | Upper/Lower                                              | Upper/Lower                                           | Most straightforward hypertrophy default.                      |
| 5 days      | Upper/Lower + Full or Focus Day                          | Upper/Lower/Pull/Push/Legs or Upper/Lower + Focus Day | Avoid overly complex plans for beginners.                      |
| 6 days      | Full Body rotation or simple PPL x 2 only if appropriate | PPL x 2                                               | Beginner 6-day plans should be conservative and lower fatigue. |

#### Split selection rules

* If user selects 2 or 3 days, default to full-body templates.
* If user selects 4 days, default to upper/lower.
* If user selects 5 days, use upper/lower plus a focus day unless the user is intermediate and explicitly prefers PPL-style training.
* If user selects 6 days, use PPL x 2 for intermediate users by default.
* If beginner selects 6 days, show a caution message and generate a conservative plan with lower per-session volume.
* If schedule has back-to-back training days, avoid placing two high-fatigue lower-body sessions consecutively.

---

### 7.5 Weekly muscle volume targets - MVP draft

The app should track weekly hard sets by muscle group. For MVP, a hard set means a planned working set for a primary muscle group.

#### Beginner starting targets

| Muscle group        | Starting weekly hard sets |
| ------------------- | ------------------------: |
| Chest               |                    6 to 8 |
| Back                |                   6 to 10 |
| Quads               |                    6 to 8 |
| Hamstrings / glutes |                    4 to 8 |
| Shoulders / delts   |                    4 to 8 |
| Biceps              |                    2 to 6 |
| Triceps             |                    2 to 6 |
| Calves              |                    0 to 4 |
| Core                |                    0 to 4 |

#### Intermediate starting targets

| Muscle group        | Starting weekly hard sets |
| ------------------- | ------------------------: |
| Chest               |                   8 to 12 |
| Back                |                   8 to 14 |
| Quads               |                   8 to 12 |
| Hamstrings / glutes |                   6 to 12 |
| Shoulders / delts   |                   6 to 12 |
| Biceps              |                    4 to 8 |
| Triceps             |                    4 to 8 |
| Calves              |                    2 to 8 |
| Core                |                    0 to 6 |

#### Volume rules

* Beginners should start at the low-to-middle end of the range.
* Intermediate users may start near the middle of the range.
* Focus muscles may receive a small volume bump.
* The generator must avoid exceeding the user's session length target where possible.
* Volume should increase gradually, not automatically jump to high-volume training.
* The app should track indirect volume separately from direct volume where feasible.

  * Example: bench press counts primarily toward chest and secondarily toward triceps/front delts.

---

### 7.6 Movement coverage requirements

Each weekly plan should cover the following movement or muscle intents unless the user has limitations or equipment constraints.

#### Required major movement intents

* Horizontal push
* Horizontal pull
* Knee-dominant lower body
* Hip hinge

#### Strongly recommended intents

* Vertical pull or lat-focused pull
* Vertical push or shoulder press
* Lateral delt isolation
* Rear delt / upper-back isolation
* Direct biceps
* Direct triceps

#### Optional intents

* Calves
* Core
* Forearms / grip
* Adductors / abductors
* Neck

#### Coverage rules

* Every plan should include at least one push pattern, one pull pattern, one knee-dominant pattern, and one hinge pattern per week unless blocked by user limitations.
* Intermediate plans should generally train most major muscle groups at least twice per week when schedule allows.
* Beginner plans may use fewer exercises and simpler coverage.
* If a movement intent is missing because of equipment or injury constraints, the plan explanation must say so.

---

### 7.7 Exercise metadata requirements

Each exercise in the exercise database should include enough metadata for planning and substitution.

Required exercise fields:

* Exercise ID
* Exercise name
* Primary muscle groups
* Secondary muscle groups
* Movement intent
* Equipment required
* Difficulty level
* Beginner-friendly flag
* Substitution group
* Unilateral or bilateral flag
* Compound or isolation flag
* Default rep range
* Default set range
* Default rest time
* Setup complexity
* Fatigue cost: low, medium, or high
* Joint stress notes, if applicable
* Contraindication tags, if applicable

Example movement intents:

* horizontal_push
* vertical_push
* horizontal_pull
* vertical_pull
* knee_dominant
* hip_hinge
* chest_isolation
* quad_isolation
* hamstring_isolation
* lateral_delt
* rear_delt
* biceps_isolation
* triceps_isolation
* calf_raise
* core_flexion
* core_anti_extension

---

### 7.8 Equipment constraint logic

The generator must only assign exercises compatible with the equipment profile for that specific training day.

#### Equipment matching rules

* If the day is marked full gym, all standard gym exercises are eligible unless blocked by preferences or limitations.
* If the day is marked dumbbells only, only dumbbell/bodyweight exercises are eligible.
* If the day is marked bodyweight only, only bodyweight exercises are eligible.
* If the day is marked bands only, only band/bodyweight exercises are eligible.
* If an exercise requires equipment not available that day, it must not be assigned.

#### Constraint fallback rules

If no ideal exercise exists for a movement intent:

1. Select the closest available exercise in the same movement intent.
2. If unavailable, select from the same primary muscle group.
3. If unavailable, select a bodyweight or low-equipment fallback.
4. If still unavailable, flag the gap and explain the limitation.

Example:

> Tuesday is bodyweight-only, so the app selected push-ups instead of dumbbell bench press for the horizontal press slot.

---

### 7.9 Exercise substitution requirements

Users must be able to substitute an exercise from the workout player or plan editor.

Substitution options should be ranked by:

1. Same movement intent
2. Same primary muscle
3. Compatible equipment
4. Similar difficulty
5. Similar fatigue cost
6. User preference history
7. Prior performance history

When substituting, the app should preserve:

* Movement intent
* Target muscle
* Planned set count when reasonable
* Target rep range when reasonable
* Training purpose of the original slot

The app should not blindly preserve load from the prior exercise because different exercises use different loading patterns.

---

### 7.10 Exercise ordering rules

Workout exercise order should generally follow:

1. High-skill or high-fatigue compound lifts
2. Secondary compound lifts
3. Machine or dumbbell accessories
4. Isolation exercises
5. Calves/core/finishers

Additional rules:

* Avoid placing grip-intensive pulling exercises before exercises where grip failure would interfere, unless intended.
* Avoid placing high-fatigue lower-body compounds at the end of a session by default.
* For beginners, prioritize simpler and safer exercises earlier in the workout.
* For focus muscles, place the focus movement earlier when appropriate.

---

### 7.11 Session length estimation

The generator should estimate session duration using:

* Number of exercises
* Number of sets
* Rest time per exercise
* Setup complexity
* Warm-up assumption

MVP estimation formula can be simple:

* Working set time estimate: 45 to 75 seconds
* Rest time: exercise default rest seconds
* Exercise transition/setup time: 1 to 3 minutes depending on setup complexity
* Warm-up buffer: 5 to 10 minutes

If generated workout exceeds the user's session length target by more than 10 minutes, the app should reduce lower-priority accessory work before removing major movement coverage.

---

### 7.12 Plan validation checks

Before saving a generated plan, the app should validate:

* Every workout has at least one exercise.
* Every exercise is compatible with that day's equipment.
* Weekly plan includes required movement coverage unless explicitly blocked.
* Weekly hard sets do not exceed safe starting ranges for the user's experience level.
* No locked user preference was overwritten.
* No disliked exercise was selected unless there is no reasonable alternative.
* Estimated session duration is within a reasonable range.
* Consecutive-day fatigue is not obviously excessive.

If validation fails, the app should either revise the plan or show a clear warning.

---

### 7.13 Plan regeneration behavior

The app may regenerate a plan when:

* User changes equipment availability
* User changes training days
* User changes session length
* User changes goal
* User changes experience level
* User adds or removes limitations
* User requests a new plan
* Weekly review recommends meaningful changes

Regeneration rules:

* Do not overwrite locked exercises or locked days.
* Preserve user edits where possible.
* Create a new PlanVersion.
* Store change log.
* Allow one-tap revert to prior plan version.
* Explain major changes.

---

### 7.14 Generator acceptance criteria

* Given a user completes onboarding, when the generator runs, then it creates a weekly plan with workouts matching the selected number of training days.
* Given a user has different equipment on different days, when the generator assigns exercises, then each exercise must match that day's equipment profile.
* Given a user dislikes an exercise, when alternatives exist, then the generator does not select the disliked exercise.
* Given a user has a locked exercise, when the plan regenerates, then the locked exercise remains unless the user unlocks it.
* Given a plan is generated, when the user views it, then the app shows a plain-language explanation of why the plan was created.
* Given the generated plan exceeds the user's session length target, then the app should reduce lower-priority accessory work before removing major movement coverage.
* Given a movement pattern cannot be fulfilled due to equipment or limitation constraints, then the app should explain the gap.

---

## 10. Progression Engine Requirements

### 8.1 Progression engine purpose

The progression engine recommends how future workouts should change based on logged performance, target rep ranges, recovery signals, and user adherence.

The MVP progression engine should be conservative, explainable, and user-approved by default.

The engine may recommend:

* Add reps
* Add load
* Hold load
* Reduce load
* Add a set
* Remove a set
* Substitute an exercise
* Suggest a deload
* Keep the current plan unchanged

---

### 8.2 MVP progression model: double progression

The default MVP progression method is double progression.

For each exercise, the app gives:

* Planned sets
* Target rep range
* Starting load or resistance
* Optional RIR target

The user keeps the same load until they can complete all planned working sets at the top of the target rep range with acceptable effort and form.

Example for 3 sets of 8 to 12 reps:

* Session 1: 50 lb x 10, 9, 8 → hold load
* Session 2: 50 lb x 12, 11, 10 → hold load
* Session 3: 50 lb x 12, 12, 12 → suggest load increase
* Session 4: 55 lb x 9, 8, 8 → hold new load

---

### 8.3 Exercise-level progression rules

#### Rule A: Add reps

If the user completes all planned sets but does not reach the top of the rep range, recommend trying to add reps next time.

Example:

> You completed all sets but have not reached the top of the rep range yet. Try to add 1 rep next time while keeping the same weight.

#### Rule B: Add load

If the user completes all planned sets at or above the top of the target rep range, recommend a load increase next time.

Before suggesting load increase, check:

* User completed all planned sets.
* User reached the top of the rep range on all planned working sets.
* RIR, if logged, was not extremely high or extremely low in a way that contradicts the recommendation.
* No pain flag was logged.
* Recent fatigue check-in is not high.

#### Rule C: Hold load

Recommend holding load when:

* User is still progressing within the rep range.
* User missed the top of the rep range.
* User reported high fatigue.
* User recently increased load and reps dropped as expected.
* User had inconsistent logging.

#### Rule D: Reduce load

Suggest reducing load when:

* User fails to reach the minimum rep target across multiple sets.
* Performance drops sharply compared with prior sessions.
* User reports pain or form breakdown.
* User repeatedly logs very low reps with the same load.

#### Rule E: Substitute exercise

Suggest substitution when:

* User repeatedly skips the exercise.
* User repeatedly substitutes it manually.
* User reports pain or discomfort.
* Equipment is no longer available.
* User marks exercise as disliked.

---

### 8.4 Load increase rules

Load jumps should be based on equipment type and available increments.

Default MVP load jumps:

| Exercise type                 |                                Default suggested increase |
| ----------------------------- | --------------------------------------------------------: |
| Dumbbell upper-body isolation | Smallest available jump, usually 2.5 to 5 lb per dumbbell |
| Dumbbell compound upper-body  |        Smallest available jump, usually 5 lb per dumbbell |
| Barbell upper-body            |                                          5 to 10 lb total |
| Barbell lower-body            |                      10 lb total, conservative by default |
| Machine/cable                 |               Smallest available plate or stack increment |
| Bodyweight                    |       Add reps first, then harder variation or added load |

Rules:

* If available increment data is known, use it.
* If not known, use conservative defaults.
* For beginners, prefer the smallest practical increase.
* For isolation exercises, use smaller jumps than compound exercises.
* If load increase would likely push the user below the minimum rep range, recommend holding load instead.

---

### 8.5 RIR handling

RIR is optional and should never block workout completion.

#### Beginner RIR behavior

* Do not require RIR.
* Use simple effort language instead.
* Example: easy, moderate, hard, too hard.
* If RIR is shown, include education.

#### Intermediate RIR behavior

* Allow optional RIR targets.
* Allow optional RIR logging per set.
* Default hypertrophy guidance may use roughly 0 to 3 RIR, depending on exercise and user preference.
* Compound lifts should generally avoid repeated all-out failure by default.
* Isolation lifts may allow closer-to-failure guidance when appropriate.

#### RIR recommendation logic

* If user logs very high RIR while hitting top reps, app may suggest increasing load.
* If user logs 0 RIR repeatedly with performance decline or high fatigue, app may suggest holding load or reducing volume.
* If RIR is missing, app should rely on reps, load, trend, and check-in data.

---

### 8.6 Readiness and recovery signals

The progression engine may use the following recovery signals:

* Fatigue rating
* Soreness rating
* Mood rating
* Sleep rating, optional
* Pain/discomfort flag
* Missed workouts
* Performance trend

MVP should avoid complex readiness scoring unless it is explainable.

Recommended MVP readiness categories:

* Good: normal or improving performance, low-to-moderate fatigue
* Caution: high soreness/fatigue or inconsistent recent training
* Poor: pain flag, sharp performance drop, repeated high fatigue

Readiness should modify recommendations, not fully control them.

Example:

> You reached the top of the rep range, but your fatigue rating was high. The app recommends holding this weight for one more session before increasing.

---

### 8.7 Volume adjustment rules

Volume changes should be slower than load or rep changes.

#### Add-set suggestion rules

The app may suggest adding one set for a muscle group when:

* User has completed most planned workouts for at least 2 to 4 weeks.
* Recovery check-ins are acceptable.
* Performance is stable or improving.
* The muscle group is a selected focus area or appears under-dosed.
* Current weekly volume is below the user's target range.

#### Remove-set suggestion rules

The app may suggest removing one set when:

* User repeatedly misses workouts because sessions are too long.
* User repeatedly skips the same later-session accessory exercises.
* Fatigue or soreness is consistently high.
* Performance is declining across multiple exercises for the same muscle group.
* User is in a cut and recovery appears limited.

#### Volume cap rules

* Beginner weekly set increases should be conservative.
* Do not increase more than 1 to 2 sets per muscle group per week.
* Do not increase volume if adherence is poor.
* Do not increase volume and load aggressively at the same time for the same muscle group.
* Require user confirmation before meaningful volume increases.

---

### 8.8 Goal-specific progression behavior

#### Bulk

* More willing to progress load, reps, or volume if recovery is good.
* Nutrition adherence should influence confidence in progression recommendations.
* If weight is not trending upward over multiple weeks, nutrition coaching may be prioritized.

#### Cut

* More conservative with volume increases.
* Maintaining strength/performance can be treated as success.
* If fatigue is high and calories are low, recommend holding load or reducing volume rather than pushing progression.

#### Recomp

* Moderate progression expectations.
* Prioritize consistency, adherence, and gradual performance improvement.
* Avoid aggressive volume changes.

---

### 8.9 Deload suggestion rules

The app may suggest a deload when multiple signals occur together.

Possible deload triggers:

* Performance decreases for the same exercise across 2 or more exposures.
* Several exercises for the same muscle group decline in the same week.
* Fatigue or soreness ratings are repeatedly high.
* User reports poor sleep or poor recovery.
* User misses multiple workouts after a period of high adherence.
* User has trained consistently for several weeks without any lower-stress week.
* User reports pain or joint discomfort.

MVP deload behavior:

* Never auto-apply deload by default.
* Explain why deload is suggested.
* Let user accept, modify, or ignore.
* Deload may reduce volume, load, or both.
* Default deload should last one week.

Example:

> Your performance has dropped for two sessions and your fatigue ratings were high this week. Consider a lighter week to recover before pushing progression again.

---

### 8.10 Missed workout handling

When a workout is missed, the app should ask what the user wants to do.

Options:

* Move workout to next available day
* Skip workout and continue plan
* Replace next workout with missed workout
* Regenerate the rest of the week

Rules:

* Do not automatically punish or overload the user for missing a workout.
* Do not double session volume by default.
* Avoid placing two high-fatigue workouts back-to-back.
* If multiple workouts are missed, prioritize getting the user back on schedule.

---

### 8.11 Recommendation card requirements

Each progression recommendation should include:

* Recommendation type
* Specific suggested change
* Reason
* Inputs used
* Risk or caution note, if relevant
* Accept button
* Modify button
* Ignore button

Example:

> **Recommendation:** Increase dumbbell bench press to 55 lb next time.
> **Why:** You completed all 3 sets at the top of the 8 to 12 rep range last session.
> **Inputs used:** Last workout performance, target rep range, progression rule.
> **Action:** Accept / Modify / Ignore

---

### 8.12 Progression audit trail

Every accepted recommendation should create a change event.

Change event should store:

* User ID
* Plan version ID
* Workout day ID
* Exercise instance ID, if applicable
* Recommendation type
* Old value
* New value
* Inputs used
* User action: accepted, modified, ignored, reverted
* Timestamp

This is important for explainability, debugging, and future personalization.

---

### 8.13 Progression acceptance criteria

* Given a user completes all sets below the top of the rep range, when the progression engine runs, then it recommends holding load and trying to add reps.
* Given a user completes all planned sets at the top of the rep range, when fatigue and pain signals are acceptable, then it recommends a conservative load increase.
* Given a user logs high fatigue after reaching the top of the rep range, then the app may recommend holding load for one more session instead of increasing.
* Given a user repeatedly skips an exercise, then the app may recommend a substitution.
* Given a user reports pain on an exercise, then the app should not recommend increasing load for that exercise.
* Given multiple performance and recovery signals are negative, then the app may suggest a deload but must require user confirmation.
* Given a user misses a workout, then the app offers rescheduling choices instead of automatically doubling future workload.
* Given the app recommends a change, then the recommendation includes a plain-language explanation.
* Given the user accepts a progression recommendation, then the change is stored in the plan version history or recommendation audit trail.

---

## 11. Workout Player Requirements

### 9.1 Workout screen must show

* Workout name
* Estimated duration
* Exercise list
* Sets and target rep ranges
* Rest guidance
* Last performance for each exercise
* Optional RIR target or education
* Substitution option
* Notes field

### 9.2 Set logging fields

Required:

* Weight or resistance level
* Reps
* Completed / skipped state

Optional:

* RIR
* Notes
* Pain/discomfort flag

### 9.3 Set states

A set can be:

* Planned
* Completed
* Skipped
* Edited
* Warm-up

### 9.4 Workout completion

At workout completion, app should:

* Save all logs
* Show summary
* Ask quick check-in
* Highlight PRs or improvements
* Show next recommendation if available

---

## 12. Nutrition Requirements

### 10.1 Nutrition modes

Strict mode:

* User logs foods through search
* User can adjust serving sizes
* App tracks calories, protein, carbs, and fat
* User can save meals and favorites

Lite mode:

* User logs macro-serving shortcuts or meal templates
* App prioritizes adherence and simplicity over perfect precision
* User can create reusable meals like “protein shake,” “breakfast,” or “usual lunch”

### 10.2 Macro targets

The app should calculate initial calorie and macro targets using user inputs, then allow users to edit targets manually.

Minimum macro targets to store:

* Calories
* Protein grams
* Carbs grams
* Fat grams

### 10.3 Nutrition dashboard

Dashboard should show:

* Calories remaining
* Protein remaining
* Carbs remaining
* Fat remaining
* Progress toward daily target
* Logged meals
* Quick-add buttons

### 10.4 Nutrition reminders

Nutrition reminders should be conditional, not spammy.

Examples:

* If protein is low by evening and reminders are enabled, send reminder.
* If no food has been logged by a configured time, send reminder.
* Do not exceed max push limit.
* Do not send during quiet hours.

---

## 13. AI / Recommendation Behavior

### 11.1 Default mode

The default mode is suggestion-only.

The app can recommend:

* Add reps
* Add load
* Hold load
* Reduce load
* Substitute exercise
* Add set
* Remove set
* Deload
* Adjust schedule
* Adjust calorie/macro targets

### 11.2 Recommendation card format

Each recommendation should include:

* Recommendation title
* Specific suggested change
* Reason
* Inputs used
* Confidence level or caution note
* Accept button
* Modify button
* Ignore button

Example:

> **Recommendation:** Increase dumbbell bench press to 55 lb next time.
> **Why:** You completed all 3 sets at the top of the 8 to 12 rep range last session.
> **Inputs used:** Last workout performance, target rep range, progression rule.
> **Action:** Accept / Modify / Ignore

### 11.3 Auto-apply mode

Auto-apply is not default.

If enabled later, auto-apply must:

* Stay within safe bounds
* Show change log
* Allow one-tap revert
* Never auto-apply major schedule changes
* Never auto-apply deload without confirmation in MVP
* Never override locked user choices

---

## 14. Plan Editing Requirements

Users can edit:

* Training days
* Workout order
* Exercise selection
* Sets
* Rep range
* Rest time
* Exercise notes
* Equipment for a day
* Focus muscles

When a user edits something, the app should optionally ask why:

* Equipment unavailable
* Pain/discomfort
* Too hard
* Too easy
* Time constraint
* Preference/dislike
* Fatigue
* Other

These edit reasons should become future planning signals.

---

## 15. Data Model + API Requirements

### 15.1 Purpose of this section

This section defines the MVP data model and API requirements at a product/technical specification level. It is not intended to prescribe a specific database, backend framework, or exact implementation, but it should be detailed enough for Codex or an engineer to review for feasibility, missing entities, and implementation risks.

The data model should support:

* Account/profile management
* Onboarding inputs
* Day-level equipment availability
* Rules-based workout plan generation
* Plan version history and revert
* Live workout logging
* Exercise substitutions
* Progression recommendations
* Nutrition logging
* Weekly review
* Notifications/reminders
* Offline or interrupted workout preservation
* User control, auditability, and explainability

---

### 15.2 Data model principles

#### 15.2.1 Preserve user-entered data

Workout logs, food logs, check-ins, body metrics, and user edits should not be overwritten by plan regeneration, sync, or recommendation logic.

#### 15.2.2 Separate planned data from completed data

A planned workout is not the same thing as a completed workout.

The model should distinguish:

* Planned exercises and sets
* Actual workout sessions
* Actual set logs
* Skipped sets/exercises
* Partial workouts
* Completed workouts

#### 15.2.3 Treat plan changes as versioned events

A generated or edited plan should produce a new `PlanVersion` or a meaningful plan change event. Users should be able to understand what changed and revert when appropriate.

#### 15.2.4 Store recommendation reasoning

Recommendations should store the inputs used and the reason shown to the user. This is important for explainability, debugging, future personalization, and user trust.

#### 15.2.5 Keep AI optional and auditable

AI-generated text or suggestions should be stored separately from deterministic plan rules where possible. The app should know whether a recommendation came from rules, AI, or a hybrid process.

#### 15.2.6 Support progressive disclosure

The data model may store advanced fields, such as RIR or volume targets, but the UI does not need to show them to every user by default.

---

### 15.3 Core entity relationship overview

High-level relationship map:

* `User` has one `UserProfile`
* `User` has many `GoalPlan` records over time
* `User` has many `BodyMetricLog` records
* `User` has many `EquipmentProfile` records
* `User` has many `EquipmentCalendarEntry` records
* `User` has many `PlanVersion` records
* `PlanVersion` has many `WorkoutDay` records
* `WorkoutDay` has many `ExerciseInstance` records
* `ExerciseInstance` references one `Exercise`
* `User` has many `WorkoutSession` records
* `WorkoutSession` has many `SetLog` records
* `WorkoutSession` may have one `PostWorkoutCheckIn`
* `User` has many `FoodLog` records
* `User` has many `SavedMeal` records
* `User` has many `Recommendation` records
* `Recommendation` may create one or more `PlanChangeEvent` records if accepted

---

### 15.4 User and profile entities

#### 15.4.1 User

Represents the authenticated account.

Suggested fields:

* `id`
* `email`
* `passwordHash` or external auth reference
* `authProvider`
* `createdAt`
* `updatedAt`
* `lastLoginAt`
* `accountStatus`
* `deletedAt`, nullable

Possible `accountStatus` values:

* `active`
* `pending_verification`
* `disabled`
* `deleted`

MVP notes:

* Authentication provider can be chosen later.
* The PRD does not require a specific auth implementation.
* If using third-party auth, avoid duplicating sensitive auth data unnecessarily.

---

#### 15.4.2 UserProfile

Stores user-level preferences and baseline information.

Suggested fields:

* `id`
* `userId`
* `displayName`, optional
* `height`
* `heightUnit`
* `currentWeight`
* `weightUnit`
* `experienceLevel`
* `dateOfBirth` or `ageRange`, optional
* `sex`, optional and only if needed for formulas
* `unitsPreference`
* `trainingPreferenceNotes`
* `createdAt`
* `updatedAt`

Possible `experienceLevel` values:

* `beginner`
* `intermediate`

MVP notes:

* Avoid collecting unnecessary sensitive data.
* If sex/age are used for calorie calculations, explain why they are requested.
* Allow users to manually override nutrition targets.

---

### 15.5 Goal and body metrics entities

#### 15.5.1 GoalPlan

Represents the user's goal configuration for a period of time.

Suggested fields:

* `id`
* `userId`
* `goalType`
* `startDate`
* `endDate`, nullable
* `startingWeight`
* `targetWeight`, optional
* `targetRateOfChange`, optional
* `calorieTarget`
* `proteinTargetGrams`
* `carbTargetGrams`
* `fatTargetGrams`
* `targetSource`
* `isActive`
* `createdAt`
* `updatedAt`

Possible `goalType` values:

* `bulk`
* `cut`
* `recomp`

Possible `targetSource` values:

* `system_calculated`
* `user_manual`
* `recommendation_adjusted`

Rules:

* Goal changes should create a new `GoalPlan` or goal-change event rather than overwriting historical goals.
* Prior workout and nutrition data should remain associated with the goal active at the time.

---

#### 15.5.2 BodyMetricLog

Stores weigh-ins and optional measurements over time.

Suggested fields:

* `id`
* `userId`
* `loggedAt`
* `weight`
* `weightUnit`
* `waist`, optional
* `neck`, optional
* `hip`, optional
* `measurementUnit`, optional
* `source`
* `notes`, optional
* `createdAt`
* `updatedAt`

Possible `source` values:

* `manual`
* `imported`
* `estimated`

Rules:

* The app should use trends over multiple body metric logs, not a single weigh-in.
* User can edit/delete accidental entries.

---

### 15.6 Equipment entities

#### 15.6.1 EquipmentProfile

Reusable set of equipment available to the user.

Suggested fields:

* `id`
* `userId`
* `name`
* `equipmentItems`
* `loadIncrementRules`, optional
* `isDefault`
* `createdAt`
* `updatedAt`

Example `equipmentItems`:

* `full_gym`
* `barbell`
* `dumbbells`
* `adjustable_dumbbells`
* `bench`
* `pullup_bar`
* `cables`
* `machines`
* `bands`
* `bodyweight_only`

Example `loadIncrementRules`:

* Dumbbells increase by 5 lb each
* Barbell increases by 5 or 10 lb total
* Cable stack increases by 10 lb

---

#### 15.6.2 EquipmentCalendarEntry

Maps equipment availability to a training day.

Suggested fields:

* `id`
* `userId`
* `dayOfWeek`
* `equipmentProfileId`
* `appliesFromDate`, optional
* `appliesToDate`, optional
* `createdAt`
* `updatedAt`

Possible `dayOfWeek` values:

* `monday`
* `tuesday`
* `wednesday`
* `thursday`
* `friday`
* `saturday`
* `sunday`

Rules:

* Day-level equipment should drive exercise eligibility.
* Changing equipment for a day should trigger revalidation of affected future workouts.
* Completed workout history should not be rewritten if equipment availability changes later.

---

### 15.7 Exercise database entities

#### 15.7.1 Exercise

Represents an exercise that can be assigned, logged, or substituted.

Suggested fields:

* `id`
* `name`
* `slug`
* `primaryMuscles`
* `secondaryMuscles`
* `movementIntent`
* `equipmentRequired`
* `difficultyLevel`
* `isBeginnerFriendly`
* `isUnilateral`
* `isCompound`
* `substitutionGroup`
* `defaultRepMin`
* `defaultRepMax`
* `defaultSetsMin`
* `defaultSetsMax`
* `defaultRestSeconds`
* `setupComplexity`
* `fatigueCost`
* `jointStressNotes`, optional
* `contraindicationTags`, optional
* `instructions`, optional for MVP
* `videoUrl`, post-MVP optional
* `createdAt`
* `updatedAt`

Possible `movementIntent` values:

* `horizontal_push`
* `vertical_push`
* `horizontal_pull`
* `vertical_pull`
* `knee_dominant`
* `hip_hinge`
* `chest_isolation`
* `quad_isolation`
* `hamstring_isolation`
* `lateral_delt`
* `rear_delt`
* `biceps_isolation`
* `triceps_isolation`
* `calf_raise`
* `core_flexion`
* `core_anti_extension`
* `other`

Possible `difficultyLevel` values:

* `beginner`
* `intermediate`
* `advanced`

Possible `setupComplexity` values:

* `low`
* `medium`
* `high`

Possible `fatigueCost` values:

* `low`
* `medium`
* `high`

MVP notes:

* Exercise metadata quality is critical to the program generator.
* The initial exercise database should be small enough to maintain but large enough to support common equipment profiles.
* MVP should prioritize common, reliable exercises over a huge exercise library.

---

#### 15.7.2 UserExercisePreference

Stores user-specific exercise preferences and avoidances.

Suggested fields:

* `id`
* `userId`
* `exerciseId`
* `preferenceType`
* `reason`, optional
* `createdAt`
* `updatedAt`

Possible `preferenceType` values:

* `preferred`
* `disliked`
* `avoid_due_to_pain`
* `avoid_due_to_equipment`
* `locked_in_plan`

Rules:

* Disliked/avoided exercises should be avoided when reasonable substitutions exist.
* Pain-related avoidances should be treated more strictly than general dislikes.

---

### 15.8 Plan entities

#### 15.8.1 PlanVersion

Represents a generated or edited version of the user's active training plan.

Suggested fields:

* `id`
* `userId`
* `versionNumber`
* `status`
* `source`
* `reasonCreated`
* `goalPlanId`
* `generatedFromPlanVersionId`, nullable
* `startedAt`, optional
* `endedAt`, optional
* `createdAt`
* `updatedAt`

Possible `status` values:

* `draft`
* `active`
* `archived`
* `reverted`

Possible `source` values:

* `onboarding_generator`
* `weekly_review`
* `user_edit`
* `recommendation_acceptance`
* `equipment_change`
* `schedule_change`
* `manual_regeneration`

Rules:

* Only one plan version should be active at a time per user.
* Reverting should activate a prior plan version or create a new version derived from the prior one.
* Completed workout logs should remain attached to actual sessions, not erased when plan changes.

---

#### 15.8.2 WorkoutDay

Represents a planned workout day within a plan version.

Suggested fields:

* `id`
* `planVersionId`
* `dayOfWeek`
* `scheduledDate`, optional for specific week instances
* `workoutName`
* `splitType`
* `equipmentProfileId`
* `estimatedDurationMinutes`
* `sortOrder`
* `isLocked`
* `createdAt`
* `updatedAt`

Possible `splitType` values:

* `full_body`
* `upper`
* `lower`
* `push`
* `pull`
* `legs`
* `focus`
* `custom`

Rules:

* WorkoutDay is planned data, not proof a workout happened.
* Actual performance belongs to `WorkoutSession` and `SetLog`.

---

#### 15.8.3 ExerciseInstance

Represents a planned exercise slot inside a workout day.

Suggested fields:

* `id`
* `workoutDayId`
* `exerciseId`
* `sortOrder`
* `plannedSets`
* `targetRepMin`
* `targetRepMax`
* `targetRirMin`, optional
* `targetRirMax`, optional
* `restSeconds`
* `movementIntentOverride`, optional
* `notes`, optional
* `isLocked`
* `source`
* `createdAt`
* `updatedAt`

Possible `source` values:

* `generator`
* `user_edit`
* `substitution`
* `recommendation`

Rules:

* ExerciseInstance should preserve the intent of the exercise slot.
* If the exercise changes, the app should know whether it was a user edit, substitution, or generated change.

---

#### 15.8.4 PlanChangeEvent

Stores meaningful changes to a plan.

Suggested fields:

* `id`
* `userId`
* `planVersionId`
* `changeType`
* `entityType`
* `entityId`
* `oldValue`, structured object or JSON
* `newValue`, structured object or JSON
* `reason`
* `source`
* `createdAt`

Possible `changeType` values:

* `exercise_added`
* `exercise_removed`
* `exercise_substituted`
* `sets_changed`
* `rep_range_changed`
* `rest_changed`
* `day_moved`
* `equipment_changed`
* `plan_regenerated`
* `plan_reverted`
* `recommendation_applied`

Possible `source` values:

* `user`
* `system`
* `recommendation`
* `ai`

Rules:

* Major plan edits should create change events.
* Change events support auditability, explanations, and revert behavior.

---

### 15.9 Workout logging entities

#### 15.9.1 WorkoutSession

Represents an actual workout attempt by the user.

Suggested fields:

* `id`
* `userId`
* `planVersionId`, nullable
* `workoutDayId`, nullable
* `startedAt`
* `completedAt`, nullable
* `status`
* `completionType`
* `durationSeconds`, optional
* `source`
* `notes`, optional
* `createdAt`
* `updatedAt`

Possible `status` values:

* `not_started`
* `in_progress`
* `completed`
* `partial`
* `abandoned`
* `deleted`

Possible `completionType` values:

* `fully_logged`
* `partial_logged`
* `completed_outside_app`
* `skipped`

Rules:

* A workout can be partial and still valid.
* A workout completed outside the app should not be treated the same as a fully logged workout.
* Workout sessions should be preserved if the app closes or goes offline.

---

#### 15.9.2 SetLog

Represents an actual set log from a workout session.

Suggested fields:

* `id`
* `userId`
* `workoutSessionId`
* `exerciseInstanceId`, nullable
* `exerciseId`
* `setNumber`
* `setType`
* `weight`
* `weightUnit`
* `reps`
* `rir`, optional
* `status`
* `skipReason`, optional
* `painFlag`, optional
* `notes`, optional
* `loggedAt`
* `createdAt`
* `updatedAt`

Possible `setType` values:

* `working`
* `warmup`
* `drop_set`
* `backoff`

MVP can support only:

* `working`
* `warmup`

Possible `status` values:

* `completed`
* `skipped`
* `attempted_failed`
* `deleted`

Rules:

* Skipped sets should be stored, not deleted.
* Failed attempts should be distinguishable from skipped sets.
* RIR is optional.
* Pain flags should influence progression logic.

---

#### 15.9.3 PostWorkoutCheckIn

Stores quick recovery and subjective state after a workout.

Suggested fields:

* `id`
* `userId`
* `workoutSessionId`
* `fatigueRating`
* `sorenessRating`
* `moodRating`
* `sleepRating`, optional
* `painReported`
* `notes`, optional
* `createdAt`
* `updatedAt`

Rating scale:

* MVP should use a simple 1 to 5 scale or labeled scale.
* The UI should explain the scale clearly.

Rules:

* Check-in should be skippable.
* Missing check-in data should not block recommendations.

---

### 15.10 Recommendation entities

#### 15.10.1 Recommendation

Represents an app-generated coaching suggestion.

Suggested fields:

* `id`
* `userId`
* `type`
* `status`
* `title`
* `summary`
* `reasonText`
* `inputsUsed`, structured object or JSON
* `targetEntityType`
* `targetEntityId`
* `suggestedChange`, structured object or JSON
* `riskNote`, optional
* `source`
* `confidenceLevel`, optional
* `expiresAt`, optional
* `createdAt`
* `updatedAt`

Possible `type` values:

* `add_reps`
* `add_load`
* `hold_load`
* `reduce_load`
* `add_set`
* `remove_set`
* `substitute_exercise`
* `deload`
* `schedule_adjustment`
* `nutrition_target_adjustment`
* `reminder_adjustment`

Possible `status` values:

* `pending`
* `accepted`
* `modified`
* `ignored`
* `dismissed`
* `expired`
* `reverted`

Possible `source` values:

* `rules_engine`
* `ai`
* `hybrid`

Rules:

* Recommendation reason must be shown to the user.
* Accepted recommendations should create a change event if they alter a plan or target.
* Ignored recommendations should be stored as feedback.

---

#### 15.10.2 RecommendationAction

Stores user action taken on a recommendation.

Suggested fields:

* `id`
* `recommendationId`
* `userId`
* `actionType`
* `modifiedChange`, optional structured object or JSON
* `reason`, optional
* `createdAt`

Possible `actionType` values:

* `accepted`
* `modified`
* `ignored`
* `dismissed`
* `reverted`

---

### 15.11 Nutrition entities

#### 15.11.1 FoodLog

Represents a food or macro entry logged by the user.

Suggested fields:

* `id`
* `userId`
* `loggedDate`
* `loggedAt`
* `mealType`
* `entrySource`
* `provider`
* `providerFoodId`, optional
* `foodName`
* `brandName`, optional
* `servingQuantity`
* `servingUnit`
* `calories`
* `proteinGrams`
* `carbGrams`
* `fatGrams`
* `fiberGrams`, optional
* `notes`, optional
* `createdAt`
* `updatedAt`

Possible `mealType` values:

* `breakfast`
* `lunch`
* `dinner`
* `snack`
* `other`

Possible `entrySource` values:

* `food_search`
* `saved_meal`
* `quick_add`
* `custom_food`
* `copied_entry`

Possible `provider` values:

* `usda_fooddata_central`
* `open_food_facts`
* `manual`
* `internal_saved_meal`

Rules:

* Nutrition logs should be editable and deletable.
* Missing nutrition data should be handled gracefully.
* Incomplete nutrition days should not be interpreted as definitely failed targets.

---

#### 15.11.2 SavedMeal

Represents a reusable meal or macro shortcut.

Suggested fields:

* `id`
* `userId`
* `name`
* `description`, optional
* `calories`
* `proteinGrams`
* `carbGrams`
* `fatGrams`
* `items`, optional structured object or JSON
* `isMacroShortcut`
* `createdAt`
* `updatedAt`

Rules:

* Saved meals support lite tracking.
* User can edit/delete saved meals.
* A logged saved meal should snapshot nutrition values at the time of logging so future edits do not rewrite history.

---

### 15.12 Notification and reminder entities

#### 15.12.1 NotificationPreference

Stores reminder preferences.

Suggested fields:

* `id`
* `userId`
* `workoutRemindersEnabled`
* `nutritionRemindersEnabled`
* `preferredWorkoutReminderTime`
* `preferredNutritionReminderTime`
* `quietHoursStart`
* `quietHoursEnd`
* `maxPushesPerDay`
* `timezone`
* `permissionStatus`
* `createdAt`
* `updatedAt`

Possible `permissionStatus` values:

* `granted`
* `denied`
* `not_requested`
* `unknown`

Rules:

* App must work if notifications are denied.
* Quiet hours and max pushes/day should be respected.

---

#### 15.12.2 NotificationEvent

Stores notification attempts and outcomes.

Suggested fields:

* `id`
* `userId`
* `type`
* `title`
* `body`
* `scheduledFor`
* `sentAt`, nullable
* `status`
* `failureReason`, optional
* `createdAt`

Possible `type` values:

* `workout_reminder`
* `nutrition_reminder`
* `weekly_review`
* `missed_workout_followup`

Possible `status` values:

* `scheduled`
* `sent`
* `skipped_quiet_hours`
* `skipped_max_limit`
* `failed`
* `cancelled`

---

### 15.13 Sync and local persistence requirements

MVP should support at least interrupted workout protection.

#### 15.13.1 Local workout draft

The client should preserve in-progress workout data locally before server confirmation.

Data to preserve:

* Active workout session ID or temporary local ID
* Exercise being logged
* Completed set logs
* Skipped set states
* Notes
* Started time
* Last saved time

Rules:

* If the app closes, user can resume workout.
* If network fails, user can continue logging.
* When network returns, local logs should sync.
* The app should not silently discard local workout data.

#### 15.13.2 Sync metadata

Entities that support offline/interrupted writes should include:

* `createdAt`
* `updatedAt`
* `deletedAt`, nullable where soft delete is useful
* `clientUpdatedAt`, optional
* `syncStatus`, optional on client
* `clientMutationId`, optional

Possible client `syncStatus` values:

* `synced`
* `pending_create`
* `pending_update`
* `pending_delete`
* `conflict`

MVP note:

* Full multi-device offline-first sync can be post-MVP.
* However, workout logs should be protected from loss from the beginning.

---

### 15.14 API design principles

The backend API should support the product loops without forcing the client to implement all business logic.

API principles:

* Use authenticated user-scoped endpoints.
* Validate user inputs on the server.
* Keep plan generation and progression logic server-side or shared in a controlled rules engine.
* Return clear errors that the UI can translate into helpful messages.
* Avoid exposing raw provider/API errors to users.
* Support idempotent writes for workout and food logs where possible.
* Support pagination for history endpoints.
* Support filtering by date range for logs and progress.

API style can be REST, GraphQL, or RPC. MVP examples below use REST-style language for clarity, not as a hard requirement.

---

### 15.15 Auth and profile API requirements

Required capabilities:

* Create account
* Log in
* Log out
* Refresh session/token
* Get current user profile
* Update profile
* Delete account request

Example endpoint groups:

* `POST /auth/signup`
* `POST /auth/login`
* `POST /auth/logout`
* `GET /me`
* `PATCH /me/profile`
* `DELETE /me/account`

Acceptance criteria:

* User can create and access an account.
* User profile can be retrieved after login.
* User can update profile values used by planning.
* Account deletion flow requires confirmation.

---

### 15.16 Onboarding API requirements

Required capabilities:

* Save onboarding progress
* Retrieve onboarding state
* Submit completed onboarding
* Generate initial plan from onboarding

Example endpoint groups:

* `GET /onboarding`
* `PATCH /onboarding`
* `POST /onboarding/complete`
* `POST /plans/generate-initial`

Rules:

* User should not lose onboarding progress if they leave midway.
* Plan generation should validate required onboarding inputs.
* If generation fails, API should return actionable reason codes.

Possible generation failure reason codes:

* `missing_required_inputs`
* `no_valid_exercises_for_equipment`
* `conflicting_limitations`
* `invalid_schedule`
* `internal_generation_error`

---

### 15.17 Equipment API requirements

Required capabilities:

* Create/update/delete equipment profiles
* Assign equipment to training days
* Retrieve equipment calendar
* Revalidate plan after equipment change

Example endpoint groups:

* `GET /equipment-profiles`
* `POST /equipment-profiles`
* `PATCH /equipment-profiles/{id}`
* `DELETE /equipment-profiles/{id}`
* `GET /equipment-calendar`
* `PUT /equipment-calendar`
* `POST /plans/{planVersionId}/revalidate-equipment`

Acceptance criteria:

* User can assign different equipment to different days.
* API can identify exercises incompatible with updated equipment.
* Completed workouts are not modified by equipment changes.

---

### 15.18 Exercise API requirements

Required capabilities:

* Retrieve exercise database
* Search/filter exercises
* Retrieve valid substitutions for a planned exercise
* Save user exercise preferences

Example endpoint groups:

* `GET /exercises`
* `GET /exercises/{id}`
* `GET /exercises/search`
* `GET /exercise-instances/{id}/substitutions`
* `POST /exercise-preferences`
* `PATCH /exercise-preferences/{id}`

Exercise filters should include:

* Equipment
* Movement intent
* Muscle group
* Difficulty
* Beginner-friendly

Acceptance criteria:

* Client can retrieve exercises for plan editing and substitutions.
* Substitution endpoint returns compatible alternatives ranked by relevance.
* User preferences influence future plan generation.

---

### 15.19 Plan API requirements

Required capabilities:

* Retrieve active plan
* Retrieve plan version history
* Generate plan
* Regenerate plan
* Edit plan
* Revert plan
* Validate plan
* Lock/unlock plan elements

Example endpoint groups:

* `GET /plans/active`
* `GET /plans/{planVersionId}`
* `GET /plans/history`
* `POST /plans/generate`
* `POST /plans/{planVersionId}/regenerate`
* `PATCH /plans/{planVersionId}`
* `POST /plans/{planVersionId}/revert`
* `POST /plans/{planVersionId}/validate`
* `POST /exercise-instances/{id}/lock`
* `POST /exercise-instances/{id}/unlock`

Rules:

* Plan edits should produce change events.
* Revert should not delete workout logs.
* Plan validation should detect equipment mismatch, missing movement coverage, and major imbalance.

Acceptance criteria:

* User can retrieve and edit current plan.
* User can view prior plan versions.
* User can revert meaningful changes.
* API preserves workout history across plan changes.

---

### 15.20 Workout logging API requirements

Required capabilities:

* Start workout session
* Resume workout session
* Save set logs
* Skip set/exercise
* Add extra set
* Finish workout
* Save post-workout check-in
* Mark workout completed outside app
* Retrieve workout history

Example endpoint groups:

* `POST /workout-sessions`
* `GET /workout-sessions/{id}`
* `PATCH /workout-sessions/{id}`
* `POST /workout-sessions/{id}/set-logs`
* `PATCH /set-logs/{id}`
* `DELETE /set-logs/{id}`
* `POST /workout-sessions/{id}/finish`
* `POST /workout-sessions/{id}/check-in`
* `GET /workout-history`

Idempotency requirements:

* Set-log creation should support a client-generated mutation ID or temporary ID to avoid duplicates during retries.
* Finishing a workout should be safe to retry.

Acceptance criteria:

* User can start, save, resume, and finish workouts.
* Set logs are preserved during network issues.
* Duplicate logs from retries are minimized.
* Partial workouts are represented correctly.

---

### 15.21 Progression and recommendation API requirements

Required capabilities:

* Generate recommendations
* Retrieve pending recommendations
* Accept recommendation
* Modify recommendation
* Ignore/dismiss recommendation
* Revert accepted recommendation if applicable
* Retrieve recommendation history

Example endpoint groups:

* `POST /recommendations/generate`
* `GET /recommendations/pending`
* `GET /recommendations/history`
* `POST /recommendations/{id}/accept`
* `POST /recommendations/{id}/modify`
* `POST /recommendations/{id}/ignore`
* `POST /recommendations/{id}/revert`

Rules:

* Recommendations should include reason text and inputs used.
* Accepted recommendations should apply changes atomically where possible.
* Major recommendations, such as deloads, should require confirmation.
* Ignored recommendations should influence future suggestions.

Acceptance criteria:

* User can act on recommendations.
* Recommendation actions are stored.
* Accepted recommendations update plans or targets correctly.
* Recommendation history is available for auditability.

---

### 15.22 Nutrition API requirements

Required capabilities:

* Retrieve macro targets
* Update macro targets manually
* Search foods from provider abstraction
* Lookup food details
* Create food log
* Edit/delete food log
* Create/edit/delete saved meals
* Retrieve daily nutrition summary
* Retrieve nutrition history

Example endpoint groups:

* `GET /nutrition/targets`
* `PATCH /nutrition/targets`
* `GET /nutrition/foods/search`
* `GET /nutrition/foods/{provider}/{providerFoodId}`
* `GET /nutrition/logs?date=YYYY-MM-DD`
* `POST /nutrition/logs`
* `PATCH /nutrition/logs/{id}`
* `DELETE /nutrition/logs/{id}`
* `GET /nutrition/saved-meals`
* `POST /nutrition/saved-meals`
* `PATCH /nutrition/saved-meals/{id}`
* `DELETE /nutrition/saved-meals/{id}`
* `GET /nutrition/summary?date=YYYY-MM-DD`

Provider abstraction requirements:

* Food search should use a provider abstraction layer.
* API should normalize provider data into calories, protein, carbs, and fat.
* API should handle provider downtime gracefully.
* API should allow manual/custom entries when provider data is unavailable.

Acceptance criteria:

* User can log nutrition in strict mode.
* User can log nutrition in lite mode.
* User can recover from provider failure.
* Daily macro summary updates after each log.

---

### 15.23 Body metrics and progress API requirements

Required capabilities:

* Create/edit/delete body metric logs
* Retrieve body weight trend
* Retrieve workout adherence
* Retrieve nutrition adherence
* Retrieve PRs/recent improvements
* Retrieve weekly review data

Example endpoint groups:

* `GET /body-metrics`
* `POST /body-metrics`
* `PATCH /body-metrics/{id}`
* `DELETE /body-metrics/{id}`
* `GET /progress/summary`
* `GET /progress/weight-trend`
* `GET /progress/workout-adherence`
* `GET /progress/nutrition-adherence`
* `GET /weekly-review`

Acceptance criteria:

* User can log and edit body metrics.
* Weekly review can retrieve required summary data.
* Progress trends do not over-interpret missing data.

---

### 15.24 Notification API requirements

Required capabilities:

* Retrieve notification preferences
* Update notification preferences
* Schedule/cancel reminders
* Store notification events
* Respect quiet hours and max pushes/day

Example endpoint groups:

* `GET /notification-preferences`
* `PATCH /notification-preferences`
* `GET /notification-events`
* `POST /notification-events/test`, optional internal/dev endpoint

Rules:

* App should work if push permissions are denied.
* Notification logic should be conditional and non-spammy.
* Notification events should be logged for debugging.

---

### 15.25 Error response requirements

API errors should be predictable and user-recoverable.

Recommended error shape:

* `code`
* `message`
* `userMessage`
* `details`, optional
* `retryable`

Example error codes:

* `unauthorized`
* `validation_error`
* `not_found`
* `conflict`
* `provider_unavailable`
* `plan_generation_failed`
* `no_valid_substitution`
* `sync_conflict`
* `rate_limited`
* `internal_error`

Rules:

* Do not expose raw third-party provider errors directly to users.
* UI should show `userMessage` where appropriate.
* Logs can store technical details for debugging.

---

### 15.26 Privacy and security data requirements

MVP should include basic privacy/security expectations.

Requirements:

* Encrypt data in transit.
* Use secure authentication/session handling.
* Do not store unnecessary personal data.
* Do not store photos in MVP.
* Allow account deletion request.
* Protect user workout, nutrition, body metric, and account data.
* Avoid using user data for AI features without clear consent if applicable.
* Separate analytics/product events from sensitive user details where possible.

Data that requires extra care:

* Body weight and measurements
* Nutrition logs
* Goal data
* Pain/injury flags
* Account/auth data

---

### 15.27 Analytics events - MVP draft

Analytics should help evaluate product usage without storing unnecessary sensitive details.

Recommended MVP events:

* `account_created`
* `onboarding_started`
* `onboarding_completed`
* `plan_generated`
* `plan_generation_failed`
* `plan_edited`
* `exercise_substituted`
* `workout_started`
* `set_logged`
* `workout_completed`
* `workout_completed_partial`
* `workout_missed_action_selected`
* `food_logged`
* `saved_meal_created`
* `recommendation_shown`
* `recommendation_accepted`
* `recommendation_modified`
* `recommendation_ignored`
* `weekly_review_viewed`
* `notification_enabled`
* `notification_disabled`

Analytics rules:

* Avoid sending raw food names, body measurements, detailed notes, or pain notes to analytics unless explicitly needed and consented.
* Use aggregate or categorical properties where possible.
* Analytics should support MVP KPIs without compromising privacy.

---

### 15.28 Data model open questions

Codex or an engineer should review the following unresolved decisions:

1. Should `WorkoutDay` represent a recurring weekly template, a scheduled calendar instance, or both?
2. Should plan generation be server-side, client-side, or shared logic?
3. What is the minimum offline support needed for MVP beyond workout draft preservation?
4. Should the MVP support multiple active goal periods or only one active goal at a time?
5. How should plan revert work technically: reactivate older version or create a new copy from an older version?
6. Should exercise instructions/videos be stored in MVP or deferred?
7. How large should the initial exercise database be?
8. Should custom user-created exercises be MVP or post-MVP?
9. How should provider food IDs be normalized across USDA and Open Food Facts?
10. How much recommendation reasoning should be structured data vs plain text?
11. What data should be soft-deleted vs hard-deleted?
12. What sync conflict strategy is realistic for the first build?
13. Which analytics platform will be used, if any?
14. What privacy requirements apply based on launch market?

---

### 15.29 Data/API acceptance criteria

* User profile, goal, schedule, equipment, plan, workout, nutrition, and recommendation data can be stored and retrieved.
* Planned workouts and completed workout logs are stored separately.
* Plan changes are versioned or recorded as change events.
* Workout set logs are not lost if the app is closed, interrupted, or temporarily offline.
* The app can retrieve compatible exercise substitutions based on equipment and movement intent.
* The app can generate, store, and act on recommendations with visible reasoning.
* Nutrition logs can be created through strict food search or lite quick-add flows.
* Nutrition provider failures do not block all nutrition logging.
* Weekly review data can be generated from stored workout, nutrition, body metric, and recommendation data.
* User can edit/delete accidental logs where appropriate.
* Major user-facing API failures return recoverable error messages.
* The data model supports MVP without requiring post-MVP features like barcode scanning, wearables, photo storage, or AI chat.

---

## 16. Implementation Readiness Addendum

### 16.1 Purpose of this addendum

This section resolves or narrows the highest-priority implementation gaps identified during Codex technical review.

The goal is to make the PRD easier to convert into architecture, sprint planning, and eventual implementation tasks without allowing the MVP to expand uncontrollably.

This section should be treated as the bridge between product requirements and engineering planning.

---

### 16.2 MVP Core vs MVP Extended

The MVP is intentionally split into two layers:

1. **MVP Core** — required to prove the product loop and ship a usable first version.
2. **MVP Extended** — valuable fast-follow features that improve quality but should not block the first usable release.

#### MVP Core ship gate

MVP Core must support this complete user path:

1. Create account or authenticate.
2. Complete onboarding.
3. Generate a weekly plan using days/week and day-level equipment.
4. View Today dashboard.
5. Start and complete a workout.
6. Log weight and reps.
7. Preserve interrupted workout data locally.
8. Receive simple progression guidance after enough workout history exists.
9. Use lite nutrition logging.
10. View basic weekly review.
11. Handle a missed or partial workout without data loss.

#### MVP Core includes

* Account/profile basics
* Onboarding
* Equipment calendar
* Small exercise database with high-quality metadata
* Deterministic rules-based plan generator
* Weekly plan screen
* Today dashboard
* Workout player
* Set logging
* Partial workout handling
* Local in-progress workout persistence
* Lite nutrition logging
* Saved meals or macro shortcuts
* Simple recommendation cards
* Accept/ignore recommendation actions
* Basic weekly review
* Basic privacy/account controls

#### MVP Extended fast-follow

MVP Extended can follow after the Core loop works:

* Strict food database search
* Multiple nutrition providers
* Recommendation modify flow
* Full plan revert UI
* Detailed muscle-group volume summaries
* Advanced readiness categories
* Custom foods
* Rest timer
* Warm-up set marker
* More detailed analytics
* More robust multi-device sync conflict UX

#### Post-MVP remains deferred

* Barcode scanning
* Wearables
* AI chat coach
* AI auto-apply
* Advanced periodization
* Exercise demo video library
* Social/community features
* Photo storage or body photo comparison

---

### 16.3 Canonical decision: workout templates vs dated instances

The system should model both recurring templates and actual workout sessions.

#### WorkoutDay = planned template

`WorkoutDay` represents a planned workout inside a `PlanVersion`.

It answers:

* What does the plan expect the user to do?
* Which day of week is this usually assigned to?
* Which exercises are planned?
* What equipment profile does this planned day use?

#### WorkoutSession = dated execution instance

`WorkoutSession` represents an actual workout attempt on a specific date/time.

It answers:

* Did the user start the workout?
* When did they start and finish?
* Which planned workout was it based on?
* Was it completed, partial, abandoned, skipped, or completed outside the app?
* What did the user actually log?

#### Rule

Do not store actual performance directly on `WorkoutDay` or `ExerciseInstance`. Actual performance belongs to `WorkoutSession` and `SetLog`.

This prevents plan edits, reverts, or regenerations from corrupting workout history.

---

### 16.4 Canonical domain state machines

#### 16.4.1 PlanVersion lifecycle

Possible states:

* `draft`
* `active`
* `archived`
* `reverted`

Allowed transitions:

| From     | To       | Trigger                                                |
| -------- | -------- | ------------------------------------------------------ |
| draft    | active   | Initial plan accepted or generated                     |
| active   | archived | New plan version becomes active                        |
| active   | reverted | User reverts away from current version                 |
| archived | active   | User restores prior version by cloning or reactivation |
| reverted | archived | System retains old reverted version for history        |

Recommended MVP rule:

* Revert should create a **new active clone** from a prior version rather than mutating the old version in place.
* This keeps historical plan states auditable.

---

#### 16.4.2 WorkoutSession lifecycle

Possible states:

* `not_started`
* `in_progress`
* `completed`
* `partial`
* `abandoned`
* `skipped`
* `completed_outside_app`
* `deleted`

Allowed transitions:

| From              | To                    | Trigger                                           |
| ----------------- | --------------------- | ------------------------------------------------- |
| not_started       | in_progress           | User starts workout                               |
| in_progress       | completed             | User finishes all or confirms completion          |
| in_progress       | partial               | User finishes without completing all planned work |
| in_progress       | abandoned             | User exits and later chooses not to resume        |
| not_started       | skipped               | User skips scheduled workout                      |
| not_started       | completed_outside_app | User says they trained outside app                |
| completed/partial | deleted               | User deletes accidental session                   |

Rules:

* `partial` is a valid completed state, not a failure.
* `skipped` is not the same as `attempted_failed`.
* Completed set logs remain valid even if the session is partial.
* Deleted sessions should be soft-deleted at least initially to prevent accidental loss.

---

#### 16.4.3 Recommendation lifecycle

Possible states:

* `pending`
* `accepted`
* `modified`
* `ignored`
* `dismissed`
* `expired`
* `reverted`

Allowed transitions:

| From     | To        | Trigger                      |
| -------- | --------- | ---------------------------- |
| pending  | accepted  | User accepts recommendation  |
| pending  | modified  | User accepts with changes    |
| pending  | ignored   | User chooses ignore          |
| pending  | dismissed | User closes/dismisses card   |
| pending  | expired   | Recommendation becomes stale |
| accepted | reverted  | User reverts accepted change |

MVP Core rule:

* Support `accept` and `ignore` first.
* `modify` can be MVP Extended unless implementation is simple.

---

#### 16.4.4 Sync lifecycle

Client-side sync states for locally edited records:

* `synced`
* `pending_create`
* `pending_update`
* `pending_delete`
* `conflict`

MVP Core rule:

* Full multi-device offline-first sync is not required for MVP Core.
* In-progress workout data must still be protected locally.
* If conflicts occur, preserve workout logs first and ask the user only when automatic merge is unsafe.

---

### 16.5 Generator rule precedence matrix

When generating or regenerating a plan, rules should be applied in this order.

| Priority | Rule                         | Behavior                                                             |
| -------: | ---------------------------- | -------------------------------------------------------------------- |
|        1 | Safety / pain limitation     | Avoid movements or exercises flagged for pain/discomfort.            |
|        2 | Equipment availability       | Only assign exercises possible with that day's equipment.            |
|        3 | Locked user choices          | Preserve locked exercises/days unless impossible or unsafe.          |
|        4 | Schedule/session length      | Keep workouts within realistic time limits where possible.           |
|        5 | Required movement coverage   | Preserve major movement patterns across the week.                    |
|        6 | Experience-level volume caps | Keep beginner/intermediate volume within safe starting ranges.       |
|        7 | Disliked exercises           | Avoid disliked exercises when reasonable alternatives exist.         |
|        8 | Focus muscles                | Add small priority only after core balance is satisfied.             |
|        9 | Preferred exercises          | Include where possible without breaking higher-priority constraints. |
|       10 | Variety/tie-breakers         | Use history, simplicity, and fatigue cost to break ties.             |

Conflict examples:

* If an exercise is preferred but violates equipment constraints, do not assign it.
* If an exercise is locked but later flagged for pain, warn the user and recommend substitution.
* If focus muscle volume conflicts with session length, preserve session length and major movement coverage first.

---

### 16.6 Progression numeric thresholds - MVP draft

These thresholds are initial defaults for deterministic MVP behavior. They should be adjustable later based on testing and expert review.

#### 16.6.1 Load increase threshold

Suggest load increase when all are true:

* User completed all planned working sets for the exercise.
* Each working set reached the top of the target rep range.
* No pain flag was logged for that exercise.
* Fatigue rating is not high.
* Performance was not marked as form-limited.

If RIR is logged:

* Load increase is reasonable when final set RIR is roughly 1 to 3.
* If RIR is 0 and fatigue is high, hold load instead.
* If RIR is 4+, app may still suggest load increase but should explain that effort may have been easier than target.

#### 16.6.2 Hold load threshold

Suggest holding load when any are true:

* User completed all sets but did not reach top of rep range on every set.
* User recently increased load and reps dropped but remain within range.
* Fatigue rating is high for the session.
* Recent adherence is inconsistent.
* RIR is missing and the performance signal is not clearly ready for load increase.

#### 16.6.3 Reduce load threshold

Suggest reducing load when any are true:

* User falls below minimum rep range on 2 or more working sets for the same exercise.
* Performance drops by more than roughly 20% total reps at the same load across comparable sessions.
* User logs pain/discomfort.
* User marks form breakdown.

#### 16.6.4 Add set threshold

Suggest adding one set for a muscle group only when all are true:

* User completed at least 80% of scheduled workouts over the last 2 to 4 weeks.
* Recovery check-ins are acceptable.
* No repeated pain flags for that muscle group or pattern.
* Current weekly direct volume is below the target range.
* Session length can still fit the added work.

MVP Core note:

* Add-set recommendations can be delayed until after basic load/reps progression works.

#### 16.6.5 Remove set threshold

Suggest removing one set when any are true:

* User repeatedly leaves workouts partial because of time.
* Same accessory exercise is skipped 2 or more times in recent sessions.
* Fatigue is high across multiple sessions.
* Performance declines across multiple exercises for the same muscle group.
* User is cutting and recovery appears limited.

#### 16.6.6 Deload threshold

Suggest deload when at least 2 to 3 negative signals occur together:

* Performance decreases across 2 comparable exposures.
* Fatigue rating is high in 2 or more recent sessions.
* Soreness rating is high in 2 or more recent sessions.
* User misses multiple workouts after prior consistency.
* Pain or joint discomfort is reported.

MVP rule:

* Never auto-apply deload.
* Always require user confirmation.

---

### 16.7 Nutrition provider contract - MVP draft

MVP Core should ship with lite nutrition first if strict provider integration becomes a delivery risk.

#### Provider priority

Recommended order:

1. Manual / saved meal / quick-add entries
2. One food database provider
3. Additional providers post-MVP

#### Provider abstraction requirements

Each provider integration should normalize to:

* Food name
* Brand, if available
* Serving unit
* Serving quantity
* Calories
* Protein
* Carbs
* Fat
* Source/provider
* Provider food ID
* Confidence or completeness flag

#### Confidence/completeness flags

Possible values:

* `complete_macros`
* `missing_minor_fields`
* `missing_required_macros`
* `estimated`
* `user_entered`

Rules:

* Do not log provider entries missing calories or required macros unless user confirms/manual-corrects.
* User-created entries should be clearly marked as manual.
* Saved meals should snapshot values at logging time.

#### Caching and fallback

MVP draft policy:

* Cache recent food search results where allowed by provider terms.
* Always allow quick-add/manual fallback.
* If provider is unavailable, do not block nutrition logging.

---

### 16.8 Workout logging integrity rules

#### Completed records

* Completed workout sessions should not be overwritten by plan regeneration.
* Completed set logs should remain attached to the original workout session.
* Editing a completed workout should create an update timestamp and optionally an edit event.

#### Deletion policy

MVP recommendation:

* Soft-delete workout sessions and set logs initially.
* Exclude soft-deleted logs from progression calculations.
* Retain enough audit data to recover accidental deletion during a short recovery window.

#### Editing after completion

User can edit completed logs, but:

* The app should mark edited records as edited.
* Progression recommendations should use the latest non-deleted user-confirmed values.
* If a recommendation was already generated from old values, it may need to expire or regenerate.

---

### 16.9 AI and recommendation policy constraints

The app should not generate or recommend:

* Training through severe or worsening pain
* Extreme weight-loss targets
* Very low calorie targets
* Medical diagnosis or injury treatment
* Eating disorder coaching
* Unsafe supplement/drug protocols
* Automatic deloads without confirmation
* Automatic major schedule or volume changes without confirmation
* Changes that override locked user choices without warning

Escalation language:

* For persistent/severe pain, recommend consulting a qualified professional.
* For unsafe nutrition goals, explain that the app cannot support extreme targets and suggest safer goal settings.

---

### 16.10 Privacy and compliance baseline

MVP should assume the app handles sensitive wellness-related data, even if it is not positioned as a medical app.

Baseline requirements:

* Minimize collected data.
* Explain why sensitive data is collected.
* Do not store photos in MVP.
* Do not collect precise location.
* Avoid sending raw food names, body metrics, pain notes, or private notes to analytics.
* Provide account deletion request flow.
* Define retention/deletion timelines before launch.
* Review applicable privacy laws before public release.

Initial launch assumption:

* US-first consumer wellness app.
* Not medical advice.
* Not intended for minors unless a separate minors policy is created.
* GDPR and broader international compliance should be reviewed before EU/UK launch.

Open compliance decisions:

* Exact data deletion SLA
* Export policy
* Whether workout/nutrition data is hard-deleted or anonymized after account deletion
* Whether support/admin tooling can access user data
* Consent language for any AI-based processing

---

### 16.11 Implementation dependency graph

Recommended implementation order:

1. **Domain contracts**

   * Finalize entities, state machines, versioning, and enums.

2. **Exercise metadata foundation**

   * Build/seed the initial exercise database with reliable metadata.

3. **Onboarding data capture**

   * Store profile, goal, schedule, equipment, preferences.

4. **Plan generator**

   * Generate deterministic weekly plan using equipment/day constraints.

5. **Plan viewing/editing**

   * Show plan, allow basic edits, preserve plan version semantics.

6. **Workout player and logging**

   * Start/resume/finish workout, durable set logs, partial handling.

7. **Progression recommendations**

   * Add reps/load/hold recommendations using deterministic rules.

8. **Lite nutrition**

   * Saved meals, quick-add macros, daily macro dashboard.

9. **Weekly review**

   * Summarize adherence, weight trend, progress, and recommendations.

10. **MVP hardening**

* Edge cases, privacy, analytics, error handling, notification basics.

Strict nutrition search should start after the lite loop is stable unless it is needed for the launch strategy.

---

### 16.12 Remaining implementation questions

These questions should be answered before Codex is asked to build.

1. What platform is the first build: web app, mobile app, or mobile-first web app?
2. What tech stack should be used?
3. What auth provider should be used?
4. What database should be used?
5. Should plan generation run server-side or client-side?
6. How many exercises are required in the initial database?
7. Which nutrition formula will generate calorie/macro targets?
8. Is strict nutrition search part of MVP Core or MVP Extended?
9. What is the first food provider if strict search ships?
10. What is the exact account deletion/data retention policy?
11. What analytics provider, if any, will be used?
12. Should support/admin access exist in MVP?

---

## 17. Acceptance Criteria Draft

### 14.1 Onboarding acceptance criteria

* User can complete onboarding in under 5 minutes using required fields only.
* User can select 2 to 6 training days per week.
* User can assign different equipment to different training days.
* App generates a weekly plan after onboarding.
* App explains why the plan was generated.
* User can edit the generated plan before starting.

### 14.2 Workout logging acceptance criteria

* User can start a scheduled workout.
* User can log weight and reps for each planned set.
* User can optionally log RIR.
* User can skip a set.
* User can substitute an exercise.
* Workout logs persist even if the app is closed mid-workout.
* Workout can be completed and saved.

### 14.3 Progression acceptance criteria

* App can identify when an exercise reaches the top of its rep range across all planned sets.
* App can suggest a load increase after successful completion.
* App can suggest holding load when reps are below the top of range.
* App explains each progression recommendation.
* User can accept, modify, or ignore recommendation.

### 14.4 Nutrition acceptance criteria

* User can view daily calorie and macro targets.
* User can search and log a food in strict mode.
* User can quick-add a saved meal or macro shortcut in lite mode.
* User can edit a food log.
* Dashboard updates remaining macros after each log.

---

## 18. Open Decisions for Next PRD Pass

1. What exact tech stack is preferred?
2. Is this mobile-first, web-first, or both?
3. Should MVP require account creation, or allow local/demo mode first?
4. What nutrition target formula should be used initially?
5. What food database API should be used first?
6. Should barcode scanning be MVP or V1?
7. How large should the initial exercise database be?
8. Should users be able to create custom exercises in MVP?
9. Should the app include exercise demo instructions in MVP?
10. Should missed workouts be rescheduled automatically or manually?
11. What exact volume ranges should beginners and intermediates start with?
12. What analytics events are needed for product learning?
13. What data export/delete features are required for privacy?
14. What platform notification system will be used?
15. How should offline sync conflicts be resolved?
16. How much advanced terminology should be visible by default?
17. Which features should be hidden behind progressive disclosure?
18. What should the default education style be: short tooltips, coach cards, onboarding lessons, or contextual explanations?
19. Should the primary target persona be named as “early/intermediate hypertrophy learner” throughout the PRD?
20. What is the minimum guidance needed so users can trust the plan without feeling overwhelmed?

---

## 19. Codex Review Prompt Draft

When this PRD is ready, use this prompt with Codex:

```text
Review this PRD as a senior full-stack engineer and technical product reviewer.

Do not write code.
Do not create implementation files.
Do not build the app yet.

Your task is only to review the PRD and identify anything that would block or confuse implementation.

Focus on:
- missing product requirements
- unclear user flows
- missing states and edge cases
- implied data model issues
- implied API/backend requirements
- authentication and permissions concerns
- offline sync risks
- notification logic gaps
- nutrition database integration risks
- progression engine ambiguity
- AI guardrail gaps
- MVP scope creep

Then provide:
1. prioritized PRD issues
2. suggested edits
3. developer-readiness checklist
4. questions I need to answer before implementation
5. recommended implementation phases, but without writing code
```
