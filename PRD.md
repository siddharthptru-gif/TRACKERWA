# TRACKER 360 — Product Requirements Document (v2.0)
## JEE/NEET Preparation Operating System — Deterministic Edition

**Status:** Approved → Implemented in this build
**Core law:** *No AI API. No fake progress. No fake sync. Every number is computed. Every recommendation is explainable.*

---

# PART A — PRODUCT FOUNDATION

## 1. Product Vision

Tracker 360 is a **deterministic preparation-management system** for JEE/NEET students. It understands the student's batch structure (subjects → chapters → lectures → DPPs → PYQs → tests → revisions), their real life (school hours, coaching, travel, sleep, weekly rhythms, events) and their actual performance, and it continuously produces a **realistic, editable, explainable** preparation plan.

It is not a todo app, not a calendar, not an AI chatbot. It is the student's **preparation command center** that answers, at any moment, with real data:

> *"Meri preparation exactly kaha hai, kitna bacha hai, kitna time lagega, kab karna hai, aur ab kya karun?"*

Eight operating questions the product must always answer instantly:
1. What should I do today? → **Planner**
2. What should I do *right now*? → **NOW engine**
3. What is pending and how long will it take? → **Backlog engine (type-wise, time-weighted)**
4. What should I revise? → **Revision engine (configurable spaced repetition)**
5. Can I realistically finish the backlog? → **Backlog Killer (duration × mode feasibility math)**
6. Am I falling behind? → **Preparation Health (Excellent→Critical with reasons)**
7. What if I miss today? → **Missed-day recovery redistribution**
8. Why am I seeing this task? → **Explainability ("WHY") on every generated task**

## 2. Non-Negotiable Product Principles

| # | Principle | Enforcement |
|---|-----------|-------------|
| P1 | **No AI API** for core functionality | All intelligence = deterministic rule engine, scoring, constraint scheduling, spaced repetition |
| P2 | **No fake progress** | Completion states come only from the student or an authorized integration; everything carries a `source` |
| P3 | **No fake sync** | "Synced with PW" never displayed unless authorized sync occurred; manual mode is first-class |
| P4 | **No impossible schedules** | Every plan respects available minutes, fixed timetable blocks, and event overrides |
| P5 | **Student in control** | Every task editable: move, split, postpone, complete, skip, resize; AUTO / HYBRID / MANUAL modes |
| P6 | **Partial completion is real** | Lectures/DPPs tracked at percentage level; 12/20 ≠ complete |
| P7 | **Current batch protection** | Backlog clearing never cannibalizes running classes, revision, or minimum practice |
| P8 | **Explainability** | Every recommendation ships with a computed *reason* ("revision due today + chapter accuracy 48%") |
| P9 | **Manual mode is not a crippled mode** | Manual/CSV/JSON import is a complete path, not an apology |
| P10 | **Honesty in UI** | "Planner recommendation", never "AI recommendation" |

## 3. Target Users & Student Modes

JEE (Main / Advanced / Main+Advanced) and NEET aspirants. **Student type changes planner behavior deterministically:**

| Mode | Schedule reality | Planner behavior |
|------|------------------|------------------|
| **Regular School** | Fixed school block + travel | School hours hard-blocked; study fitted into evening slots; lower daily caps |
| **Dummy School** | Flexible days | More study slots, higher caps, still anti-unrealistic guards (max 11h planned) |
| **Dropper** | Full-time prep | Aggressive revision/practice/test allocation; syllabus-completion-first ordering |
| **College + Prep** | Semi-fixed | College blocks fixed; weekend-heavy distribution |
| **Self-study** | Fully flexible | Student defines every slot; planner fills custom slots only |

## 4. Authentication & Identity

**Production path (documented, see §15):** Firebase Authentication → *Sign in with Google* → profile at `users/{uid}`. No passwords (Google's, PW's, or ours) are ever stored.

**This build:** single-workspace student profile with the identical normalized shape (`name, exam, classStatus, studentType, targetYear, batch, teachers, dailyTarget, preferences`). The auth boundary is a provider seam — swapping the demo profile provider for Firebase Auth changes zero engine code.

## 5. Architecture Overview

```
Integration Layer ───── BatchIntegrationProvider / LearningPlatformIntegration
   ├── ManualTracker (first-class, complete)
   ├── ImportProvider (JSON/CSV)
   └── PWIntegration (authorized OAuth/API deep-link seam — activates only when official)
        ↓  Normalized Learning Data (Course/Subject/Chapter/Lecture/DPP/PYQ/Test/Revision/Completion, each with `source`)
Tracker Engine ──── lecture %, DPP q-counts, revision states, backlog time math, prep meters, health
        ↓
Planner Engine ──── priority scoring · constraint scheduler · capacity resolver · replanner · recovery
        ↓
Experience Layer ── dashboard, NOW button, promise, study mode, control centers, analytics, docs
```

**Stack (this build):** Next.js 16 (App Router, RSC + Server Actions), PostgreSQL + Drizzle ORM, Tailwind v4, Recharts, lucide-react. **Zero external AI calls.**

---

# PART B — FEATURE SPECIFICATION (what this build implements)

## 6. Onboarding & First-Time Flow
12-step flow → profile → exam → class → student type → batch (select template / manual / import) → teachers → preparation status (bulk) → backlog entry → timetable → daily target → **Activate Planner** with a real summary (prep %, backlogs, revision due, hours). New/empty workspace shows "Configure your preparation" — never fake stats.

## 7. Daily Dashboard
Greeting + identity strip (exam, class, student type, batch), streak, **Today's Plan**, planned/completed/remaining time, promise ring, backlog cards **split by type**, subject progress, **Preparation Health with reasons**, quick actions (Plan My Day, What Should I Do Now?, Replan, Add Backlog, Log Practice), weekly announcements, and explainable insights.

## 8. "WHAT SHOULD I DO NOW?" (NOW Engine)
Deterministic selection of **exactly one task** given: current time, remaining capacity today, pending priorities, deadlines, weakness flags, and task fit. Output = task + estimated minutes + computed **reason** + [Study Now] → Study Mode. Never "whatever is next" — always the highest-scored feasible item, and it tells you why.

## 9. Planner Engine (deterministic core)
**Inputs:** date, exam date, batch progress, chapter weightage, lecture durations **× playback speed**, DPP states, tests ±3 days, revision due/overdue, backlog ages, weak chapters, miss history, timetable availability, custom daily hours per weekday, events, planning style, custom priority order.
**Pipeline:** capacity resolution → candidate generation → priority scoring (`urgency × importance × weakness × overdue × exam-relevance × dependency`, configurable weights/tiers CRITICAL/HIGH/MEDIUM/LOW) → greedy constraint fill (no overflow; ≥1 task guaranteed) → persist with `reason` + full explainability fields.
**Plan My Day** (fresh generation) · **Replan My Day** (same-day regeneration with *reduced* hours: protects current lecture → due revision → DPP → test, drops low-priority last, states exactly what moved and why).
**Every task** carries `{id,type,title,subject,chapter,estimatedMinutes,priority,dueDate,source,status,reason,dependencies}` and is editable (complete, skip, postpone +1d…+3d, split in halves, resize, delete).

## 10. Lecture Tracking & Library Workflow
Per-lecture: duration, **watched %**, watched minutes, notes link, DPP link, status (Not Started/Started/Partial/Completed/Skipped). Manual verification flow after "Open Class on PW" (safe external deep link `pw.live`, no scraping, no passwords): **Completed / Partial (%: 10–100 + custom) / Not completed / Continue later**. Remaining time = `(1−watched%)×duration`, planned at `duration_remaining / preferredSpeed`, but completion is **never** inferred from speed — only from actual recorded state.

## 11. Backlog Engine (type-wise, time-weighted)
Never one number. Independent queues, each with counts **and minutes**:
- **Lectures** (speed-adjusted remaining) · **DPPs** (remaining questions × 3m) · **Tests** (overdue × duration) · **Revisions** (due/overdue × 45m) · **PYQs** (remaining × 2.5m) · **Notes** (pending notes × 30m) — per subject, per chapter, oldest-first, per-teacher grouping ready.
Dashboard and /backlog always show the true breakdown + total estimated time (e.g., *34 lectures · 32h 10m*).

## 12. BACKLOG KILLER
Inputs: duration (7/14/21/30/custom days) × mode (**Safe** ×1.0 / **Balanced** ×1.5 / **Aggressive** ×2.2 dose). Output: per-day typed distribution table, daily backlog-hours, protected current-batch/revision/practice reserves, feasibility verdict, one-click apply → tasks written to planner with sources and reasons.

## 13. Revision Engine
Configurable intervals (default **R1=1, R2=3, R3=7, R4=15, R5=30, R6=60 days** — editable in Settings). Post-revision **difficulty feedback** drives the next interval deterministically:
**Forgot → +1 day · Weak → +2 days · Average → standard interval · Good → interval ×1.3 · Excellent → interval ×1.6.**
States: Upcoming / Due Today / Overdue / Completed / Skipped. Poor linked-test performance can also push a chapter back into early stages.

## 14. DPP System
Per DPP: question count, **attempted/correct/incorrect/skipped counts (partial honest)**, accuracy, time, status (Not Started/Started/Partial/Completed/Reattempt/Weak). 12/20 displays as 12/20 — never auto-100%. Practice/Quiz content modes activate only when authorized content exists; otherwise powerful manual tracking.

## 15. Test System & Deterministic Test Analysis
Tests: Chapter/Part/Subject/Full/Mock with Scheduled/Attempted/Missed/Rescheduled; scores, subject splits, attempted/correct/incorrect/unattempted, **mistake taxonomy (Conceptual/Calculation/Silly/Time-mgmt)**, external-test manual entry. Post-test analysis is rules-based: weakest section, accuracy vs target, time-per-question flags, chapter-weakness detection → auto-injects revision+practice into future plans. Overdue tests reschedule intelligently; test backlog has its own catch-up plan.

## 16. Practice, PYQ, Notes, Error Book
Practice sets by source (DPP/Module/Q-Bank/Sheet/Custom) with questions/accuracy/time/difficulty and filters. **PYQ tracker year-wise** (JEE Main / Advanced / NEET per chapter, statuses Not Attempted/Attempted/Correct/Incorrect/Reattempt). Notes states (Not Made/Incomplete/Completed/Revised) for Lecture/Short/Formula/Error notes. **Error Book**: question, subject, chapter, mistake type, correct concept, reattempt date, status — reattempts surface in the planner.

## 17. Timetable Builder & Life Constraints (this build: /timetable)
Time blocks with type (Sleep/School/College/Travel/Meals/PW Classes/Coaching/Homework/Self-study/Revision/Practice/Break/Exercise/Personal), per weekday or daily-recurring, **fixed blocks are never overwritten**; planner computes net availability from blocks. Custom daily hours per weekday (Mon 5h … Sun 10h), weekend/holiday modes, **school-exam mode** (reduces JEE load during, restores after), **custom events** (holiday/sick/travel/function/rest → capacity override → automatic redistribution). Focus-slot mapping: hard problems → high-focus slots, revision → medium, formula → low.

## 18. Study Promise & Study Mode (/study)
Pick any subset of today's tasks → **Today's Promise** (ring, X/Y done). **Study Mode** per task: minimal-distraction screen with subject/chapter, estimated vs remaining (with current watched %), [Open Class on PW], live session timer (start/pause/stop → real minutes into daily metrics), then completion declaration (Completed / Partial % / Not completed / Continue later). Missed promise → no punishment; choose: move tomorrow / next slot / backlog / split / remove — engine re-plans around it.

## 19. Missed-Day Recovery & Robustness
Unfinished tasks never collapse onto tomorrow: redistribution across N future days by priority with current-class protection, user-facing explanation (*"You missed 2 days. We redistributed 7 tasks across the next 6 days."*). Handles zero/huge backlog, single-subject backlog, missed week, exam-week, holidays, low/high availability, duplicates, invalid imports, offline queueing (local-first retry semantics on actions), browser refresh, timezone-safe local-date math, no negative time, no overlapping fixed blocks, idempotent submissions.

## 20. Preparation Meters, Health, Analytics, Goals
**Chapter Prep Meter** = weighted blend {Lectures 25%, DPP 15%, Practice 15%, PYQ 15%, Revision 15%, Tests 10%, Notes 5%} with per-component bars and **manual override** (Weak/Average/Strong/Mastered). Levels: 0 · 1–25 Started · 26–50 Basic · 51–75 Developing · 76–90 Strong · 91–100 Complete.
**Preparation Health** = f(prep %, backlog trend, revision adherence, practice velocity, schedule adherence) → Excellent/Good/Stable/At Risk/Critical **with reasons**.
Analytics: hours/questions/accuracy/backlog trends, subject balance, weekly vs previous-week comparisons, monthly aggregates, streaks (study/lecture/practice/revision). Goals: rank/percentile/score, daily hours, weekly questions, monthly chapters — with pace verdicts. **Preparation Map**: unit-level bars per subject, click-to-expand. Subject comparison surfaces the most-neglected subject from data.

## 21. Customization & Configuration (Settings)
Profile/teachers/exam targets · student type · weekly hours · timetable & slots · **planning mode (AUTO/MANUAL/HYBRID — default HYBRID)** · **planning style (Balanced/Backlog/Current/Revision/Practice/Exam)** · custom priority order · priority weights · revision intervals · playback speed (0.5–3x) · study pace (lectures/questions/revision/backlog per day) · motivation ON/OFF · notifications · import/export (**JSON/CSV, templates provided**) · tiered data resets (today / planner / progress / all) with confirmations.

## 22. Docs / Help Center (/docs)
"What is this / How it works / How to use it" for: sign-in model, batch connect & manual import, teachers, timetable & student modes, backlog entry & killer, planner activation & editing, lecture completion & partial %, DPP/test/revision/PYQ/practice tracking, error book, promise & study mode, missed-day recovery, school-exam mode, import/export, data resets, troubleshooting (sync errors, auth expiry, offline), and honest capability notes.

## 23. Admin Panel (/admin)
Global batch template management (subjects/chapters structure), announcements (student-facing banner), documentation entries. Admin cannot touch private student data; admin-write operations guarded by an admin flag in the production ruleset.

## 24. Automated Verification
Deterministic engine test-suite (see §19): planner capacity invariants, priority ordering, revision interval math incl. feedback scaling, backlog time math & killer feasibility, partial-% remaining math, health labeling, recovery redistribution — runnable via `npx tsx scripts/engine.test.ts`.

---

# PART C — TECHNICAL ARCHITECTURE

## 15. Data Layer & Firebase-Ready Mapping

This build persists to PostgreSQL via Drizzle (the sandbox's production database). The normalized model maps 1:1 onto the specified Firebase RTDB structure for the production deployment:

| RTDB node (production) | Postgres table (this build) |
|---|---|
| `users/{uid}` | `users` |
| `studentSettings/{uid}` | `user_settings` |
| `batches/{id}`, `subjects`, `chapters`, `lectures` | `batches`, `subjects`, `chapters`, `lectures` |
| `dpps`, `practice`, `pyqs` | `practice_items` (typed) |
| `tests`, `revisions` | `tests`, `revision_events` |
| `dailyPlans/{uid}/{date}` | `plan_tasks` |
| `studySessions`, `goals`, `analytics`, `notifications` | `study_sessions`, `goals`, `daily_metrics`, `announcements` |
| `timetable`, `customEvents`, `errorBook` | `timetable_blocks`, `custom_events`, `error_book` |

**Security (production Firebase Rules, enforced logic mirrored in server actions here):** unauthenticated ⇒ no access; users read/write only `*/{uid}/**` of their own tree; global templates/admin read-only for students; admin-only writes require `auth.token.admin === true`; type validation on every node; no user may ever write `admins/**`. In this build, every server action re-scopes by the workspace user id server-side.

**PW integration posture:** `LearningPlatformIntegration` interface with `authenticate / getBatches / getBatchDetails / getSubjects / getChapters / getLectures / getDPPs / getTests / getProgress / syncProgress`. `PWIntegration` activates only via official OAuth/API. Until then: `pw_authorized | manual | imported` source tags on all data, safe external class links, and **no credential collection, no scraping, no sync claims**.

## 16. Engine APIs (deterministic)
`generatePlan / replanDay / ensurePlan / nowRecommendation / backlogBreakdown / backlogKiller / applyKillerPlan / chapterPrepBreakdown / preparationHealth / revisionFeedbackIntervals / availableMinutesFor / recoveryRedistribute / explainTask / weeklyReview / importJSON / exportJSON` — pure inputs → stored outputs, all unit-tested.

## 17. UI System
Dark-first premium academic SaaS: Space Grotesk/Inter/JetBrains Mono, emerald signal accent, subject color-coding (Physics sky / Chemistry violet / Maths amber), rings, bars, timelines, prep-map blocks, heatmap-style consistency, command quick-actions, mobile bottom-reachable nav + scroll tabs, desktop sidebar command center. Light-mode-ready token set.

## 18. Performance & Scale
Server components stream per-entity aggregates; client islands only where interactive; idempotent generation keys (`userId+date`); indexes on `(userId,date)`, `(chapterId)`, `(userId,dueDate)`; large-syllabus-safe (500+ lectures × multi-subject) with in-page batching.

## 19. Definition of Done (this build)
All Part-B features above live and interactive on real persistent data · zero AI calls · zero fake sync copy · `scripts/engine.test.ts` green · typecheck/build/health green · 21 routes responsive.

---
*End of PRD v2.0.*
