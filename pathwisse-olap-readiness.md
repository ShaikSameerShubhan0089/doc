# Pathwisse LMS OLAP Readiness Report

> Repository inspected: `pathwisse-talent-development-system` (this repo).
> Target architecture: Supabase Postgres (OLTP) → ETL/CDC → ClickHouse/OLAP → **`lumina-analytics-service`** (Metrics Gateway) → LMS Analytics UI.
> Date of inspection: 2026-05-26. No code in this repo was modified.
>
> Tags used below:
> - **[FOUND]** — verified in this repo
> - **[INFERRED]** — derived from code/schema but not directly named
> - **[GATEWAY]** — belongs to the future external `lumina-analytics-service`, not this repo

---

## 1. Executive Summary

**Verdict: This LMS repo is *moderately ready* for OLAP integration from the frontend/product side. It is closer to ready than most LMS apps at this stage, but it is not OLAP-shaped yet.**

What's already strong:

- A dedicated `src/hooks/analytics/` layer and a `src/components/analytics/` chart kit already exist. Three analytics pages are wired today: `AdminAnalytics`, `PlatformAnalytics`, `InstructorDashboard`.
- All dashboard aggregation is **already pushed server-side** via Postgres `SECURITY DEFINER` RPCs (`get_platform_stats`, `get_organization_stats`, `get_instructor_stats`, `get_enrollment_trend`, etc.). That is the natural cut-over surface to the Metrics Gateway — replacing `supabase.rpc(...)` calls with `metricsApi.getX(...)` is a low-risk swap.
- TanStack Query is used everywhere with sane cache tiers (`CACHE_TIMES.STANDARD`, `LONG`, `SHORT` in `src/lib/queryClient.ts`) and centralized query keys (`src/lib/queryKeys.ts`). The cache layer is gateway-ready.
- Tenant scoping is driven by `profile.organization_id` from `useAuth`, not user-editable params — good baseline.
- An engagement events pipeline exists: `engagement_events` table + `track_engagement_event` RPC + `useEngagementTracking()`. This is the right primitive to feed the warehouse, but emission coverage is thin (see §13).

What blocks straight integration:

- There is **no `src/services/metricsApi.ts`** and no Metrics Gateway client. The frontend talks to Supabase directly for analytics today.
- **`AdminDashboard.tsx`** still does **4 sequential `count: exact` queries** (`profiles`, `user_roles`, `courses`, then `enrollments` IN courseIds) and **filters invitations client-side** after pulling the full RPC result — these are OLAP candidates, not OLTP queries.
- **`useInstructorAnalytics`** has two fall-through paths that fetch raw `enrollments` rows and bucket them in JavaScript (`enrollmentTrend`, `progressDistribution`). These need to move to the gateway.
- Event coverage is incomplete: no `lesson_started`, `lesson_completed`, `course_started`, `course_completed`, `certificate_issued`, `video_played`. Several of these are required for v1 OLAP metrics.
- No "at-risk learners" UI exists despite the supporting RPC (`get_instructor_students_at_risk`) being deployed.
- Role guards exist (`ProtectedRoute` + `allowedRoles`) but no `RoleGuard` is wrapped around analytics-specific buttons inside admin sub-pages; gateway access control must be enforced server-side regardless.

**Bottom line:** A Metrics Gateway client + 6 hooks + 4 page files would get Tenant Admin Analytics v1 onto OLAP without disturbing OLTP behavior. See §16 for ticket plan.

---

## 2. Repo Architecture

| Concern | Found in repo |
|---|---|
| Frontend framework | React 18 + Vite 5 + TypeScript 5 (`src/App.tsx`, `vite.config.ts`) |
| Routing | `react-router-dom` v6 (`BrowserRouter` in `src/App.tsx`), lazy-loaded pages via `lazyRetry()` |
| State management | TanStack Query v5 (`src/lib/queryClient.ts`) + React context (`AuthProvider` in `src/hooks/useAuth.tsx`). No Redux/Zustand. |
| Data fetching | `supabase-js` v2 via singleton at `src/integrations/supabase/client.ts`. Edge functions invoked via `supabase.functions.invoke(...)`. |
| Auth | Supabase Auth (`signInWithPassword`, `signUp`, OAuth-style password reset). Session bootstrap in `useAuth.tsx`. JWT lives in `localStorage`. |
| Role model | Enum `app_role`: `platform_owner | tenant_admin | instructor | student`. Roles stored in `public.user_roles` (separate table — good). Helper RPC `has_role(_user_id, _role, _org_id)` enforced in RLS. Frontend reads via `get_user_context` RPC. |
| Route protection | `src/components/ProtectedRoute.tsx` + `allowedRoles` prop in `src/App.tsx`. Four role areas: `/platform`, `/admin`, `/instructor`, `/dashboard`. |
| Admin/tenant dashboard | `src/pages/admin/AdminDashboard.tsx` (overview), `src/pages/admin/AdminAnalytics.tsx` (charts), `src/pages/platform/PlatformAnalytics.tsx` (global). |
| API routes (server) | None in the React app. Backend lives in `supabase/functions/*` (~40 Deno edge functions). |
| Env contract | `.env` has `VITE_SUPABASE_URL`, `VITE_SUPABASE_PROJECT_ID`, `VITE_GATEWAY_URL` (already points at `api-gateway-production-edc4.up.railway.app`), `VITE_VOICE_WS_URL`. **`VITE_GATEWAY_URL` is the natural base for the Metrics Gateway** (or a new `VITE_METRICS_API_URL`). |
| Cache tiers | `src/lib/queryClient.ts` exports `CACHE_TIMES.{SHORT, STANDARD, LONG}`. Already used by analytics hooks. |
| Query keys | Centralized in `src/lib/queryKeys.ts` with `queryKeys.analytics.{admin,instructor,platform}.*` namespaces. Gateway hooks should extend the same tree. |

---

## 3. Supabase Usage Map

Below is the full inventory of `supabase.from(...)`, `supabase.rpc(...)`, `supabase.storage`, `supabase.auth`, `supabase.functions.invoke(...)` callsites surfaced by `rg`. ~21 files use Supabase. Auth/storage rows are summarized; only OLTP/analytics-relevant rows are individually classified for the migration plan.

### 3a. Analytics-relevant (candidates for Metrics Gateway)

| File | Function/Component | Table / RPC | Op | Filters | Tenant-safe? | Analytics relevance | Recommendation |
|---|---|---|---|---|---|---|---|
| `src/hooks/analytics/useAdminAnalytics.ts` | `useAdminAnalytics` | rpc `get_organization_stats` | rpc | `org_id=profile.organization_id` | Yes (RPC is `SECURITY DEFINER` + org-scoped) | **High** | **Replace with Metrics Gateway** `GET /metrics/tenant/summary` |
| same | same | rpc `get_enrollment_trend` | rpc | `org_id`, `start_date`, `end_date` | Yes | High | **Replace** with `GET /metrics/tenant/enrollments/trend` |
| same | same | rpc `get_course_performance` | rpc | `org_id`, `max_courses=5` | Yes | High | **Replace** with `GET /metrics/tenant/courses/performance` |
| same | same | rpc `get_progress_distribution` | rpc | `org_id` | Yes | High | **Replace** (extend `…/learners/progress` or new `/distribution`) |
| `src/hooks/analytics/usePlatformAnalytics.ts` | `usePlatformAnalytics` | rpc `get_platform_stats` | rpc | (none, platform-owner only) | RLS-gated | High | **Replace later** (`GET /metrics/platform/summary` — `[GATEWAY]`, post-MVP) |
| same | same | rpc `get_platform_growth_trend` | rpc | `start_date`, `end_date` | RLS | High | Replace later |
| same | same | rpc `get_tenant_activity` | rpc | `max_tenants=6` | RLS | High | Replace later |
| same | same | rpc `get_role_distribution` | rpc | — | RLS | Medium | Replace later |
| same | same | rpc `get_category_distribution` | rpc | — | RLS | Medium | Replace later |
| `src/hooks/analytics/useInstructorAnalytics.ts` | `useInstructorAnalytics` | rpc `get_instructor_stats` | rpc | `instructor_user_id=auth.user.id` | Yes | High | Replace later (`/metrics/instructor/summary` — `[GATEWAY]`, post-MVP) |
| same | same | rpc `get_instructor_course_performance` | rpc | `instructor_user_id` | Yes | High | Replace later |
| same | same | rpc `get_instructor_recent_activity` | rpc | `instructor_user_id`, `limit_count=5` | Yes | Medium | Replace later |
| same | same (trend fallback) | `courses` + `instructor_assignments` + `enrollments` | select | `instructor_id`, `IN(course_ids)`, `gte(enrolled_at, startDate)` | Yes | High but **risky**: client-side bucketing | **Replace now** (move to gateway / new RPC) |
| same | same (distribution fallback) | `courses` + `instructor_assignments` + `enrollments` | select | `instructor_id`, `IN(course_ids)` | Yes | High but **risky**: fetches every enrollment row | **Replace now** |
| `src/hooks/analytics/useAdminDashboard.ts` | `useAdminDashboard` | `profiles`, `user_roles`, `courses` (×2), `enrollments` | 4× `count: exact, head: true` + select-then-IN | `organization_id`, `IN(courseIds)` | Yes | High | **Replace** with `GET /metrics/tenant/summary` (counts already included) |
| `src/pages/admin/AdminDashboard.tsx` (lines 84-117) | inline `useQuery` | rpc `get_invitations_for_admin`, `semesters` | rpc + select | client-side filter on `organization_id` | Yes but **pulls all invitations across orgs to filter in JS** | Medium | Add `?organization_id=` server arg or move to gateway summary card |
| `src/pages/admin/AdminDashboard.tsx` (line 200) | export builder | rpc `get_invitations_for_admin` | rpc | — | Same | Medium | Same as above |
| `src/hooks/usePrefetch.ts` (181, 194) | prefetch | rpc `get_organization_stats`, `get_platform_stats` | rpc | — | Yes | High | Keep — adapt to call gateway prefetch instead, once swap is done |
| `src/hooks/useLearningAnalytics.ts` | `useLearningSummary` | rpc `get_learning_summary` (via edge fn `generate-learning-summary`) | rpc/edge | `_user_id=auth.uid`, `_period` | Yes | High (learner-side) | Future `GET /metrics/user/:userId/progress` |
| `src/hooks/useEngagement.ts` (144, 159) | `useEngagementTracking` | rpc `track_engagement_event` | rpc (write) | `_user_id=profile.id` | Yes | **Source of warehouse events** | **Keep** as the OLTP event sink; CDC into warehouse |
| `src/hooks/useEnrollment.ts` | `useEnrollment` | `enrollments` (select+insert) | select/insert | `user_id` | Yes | Low (transactional) | Keep |
| `src/hooks/useProgress.ts` | `useProgress` | `lesson_progress` (with `lessons!inner.modules!inner`) | select | `user_id`, `lessons.modules.course_id` | Yes | Low (transactional) | Keep |
| `src/pages/instructor/InstructorStudents.tsx` (56-57) | inline | `courses`, `instructor_assignments` | select | `instructor_id` | Yes | Medium | Will be subsumed by `GET /metrics/tenant/learners/progress?instructor_id=…` post-MVP |

### 3b. Non-analytics (transactional / leave as-is)

Auth flows (`signInWithPassword`, `signUp`, `resetPasswordForEmail`, `getSession`, `onAuthStateChange`, `signOut`): `useAuth.tsx`, `Auth.tsx`, `AcceptInvite.tsx`, `ResetPassword.tsx`, `PasswordChangeForm.tsx`, `Onboarding.tsx`.

Storage uploads (`supabase.storage.from(...).upload/getPublicUrl`): `VideoUploader.tsx`, `ThumbnailUploader.tsx`, `PdfUploader.tsx`, `InstructorSettings.tsx`, `Settings.tsx`, `UploadZone.tsx` (voice), `CharterCard.tsx`.

Content CRUD: `LessonEditor.tsx`, `ModuleList.tsx`, `CourseEditor.tsx`, `AdminOnboardingManager.tsx`, `QuizPlayer.tsx` (`quiz_attempts` insert).

Edge function calls: `analyze-behavior`, `generate-nudge`, `generate-insights`, `generate-learning-summary`, `recommend-content`, `recommend-path`, `customize-roadmap`, `generate-quiz`, `generate-stage-project`, `grade-assignment`, `ai-tutor`, `start-voice-session`, etc.

XP/rewards: `add_user_xp` RPC called from `useSprintTimer.tsx`, `useCareerRoadmap.ts`, `useProgress.ts`.

User management: `admin_create_user` RPC from `UserManager.tsx`.

---

## 4. LMS Domain Model Inferred From Code

Inferred from `src/integrations/supabase/types.ts` (implied by usage), `useAuth.tsx`, the RPC source code shipped with the schema, and the hook layer.

| Domain object | Tables (FOUND) | Key fields used in code | Relationships | Analytics usefulness |
|---|---|---|---|---|
| **User / Profile** | `profiles` | `id, email, full_name, avatar_url, organization_id, onboarding_completed, career_goal, gender, role, department, batch, section` | `profiles.id = auth.users.id`; `profiles.organization_id → organizations.id` | High (dim_user) |
| **Organization (tenant)** | `organizations` | `id, name` | Parent of profiles, courses, user_roles | High (tenant key) |
| **Role / Membership** | `user_roles` | `user_id, role (app_role), organization_id` | M:M through `user_id` | Medium (dim_role for RBAC analytics) |
| **Course** | `courses` | `id, title, organization_id, instructor_id, category, is_published, difficulty_level, duration_hours, thumbnail_url` | `→ organizations`, `→ profiles (instructor)`; M:1 with modules | High (dim_course) |
| **Instructor assignment** | `instructor_assignments` | `instructor_id, subject_id (→ courses.id)` | Many-to-many courses↔instructors | Medium |
| **Module** | `modules` | `id, course_id, title, order_index` | `→ courses` | Medium |
| **Lesson** | `lessons` | `id, module_id, title, order_index` | `→ modules` | High (drop-off lessons) |
| **Enrollment** | `enrollments` | `id, user_id, course_id, enrolled_at, completed_at, progress_percentage` | `→ profiles, → courses` | **Critical** (fact_enrollment) |
| **Lesson progress** | `lesson_progress` | `user_id, lesson_id, completed, completed_at, progress_seconds, last_accessed_at` | `→ profiles, → lessons` | **Critical** (fact_lesson_progress) |
| **Assignment** | `assignments` | `id, lesson_id, stage_id, title, max_points, due_date, is_published` | `→ lessons` or `→ skill_stages` | High |
| **Assignment submission** | `assignment_submissions` | `id, assignment_id, user_id, grade, feedback, submitted_at, graded_at` | `→ assignments, → profiles` | High (fact_submission, grade dist) |
| **Quiz attempts** | `quiz_attempts` | inserted from `QuizPlayer.tsx`; `started_at`, score | `→ lessons` (inferred) | High (assessment_pass_rate) |
| **Skill / Stage** | `skills`, `skill_stages`, `user_stage_progress`, `user_skill_assessments` | `skill_id, stage_id, score, completed_at` | Skill roadmap subsystem | High (skill mastery) |
| **Learning goals** | `learning_goals` | `user_id, target_value, current_value, progress_percentage, status, completed_at` | `→ profiles` | Medium |
| **Engagement event** | `engagement_events` | `user_id, event_type, event_data jsonb, source, session_id, created_at` | `→ profiles` | **Critical** (fact_event — feeds warehouse) |
| **Behavior pattern** | `behavior_patterns` (inferred from `useEngagement.ts` `BehaviorPattern` interface) | derived | — | Medium |
| **Nudge** | `scheduled_nudges` (inferred) | nudge_type, status, channel | — | Medium (engagement loop QA) |
| **XP / Experience** | `user_experience`, `xp_transactions` | `total_xp, current_level, coins_balance, lifetime_coins_earned` | `→ profiles` | Medium (gamification) |
| **Certificate** | **[INFERRED — not directly found]** No `certificates` table referenced in src/ scan. Likely derived from `enrollments.completed_at` today. | — | — | **Gap** — needs DB confirmation |
| **Payments / subscriptions** | **[NOT FOUND]** No `payments`, `subscriptions`, `stripe_*` references in src/. `payments--enable_stripe_payments` is available as a deferred tool but not enabled. | — | — | N/A for v1 |
| **AI usage** | Implicit only — `LOVABLE_API_KEY` + `GEMINI_API_KEY` used by edge functions (`ai-tutor`, `generate-quiz`, `generate-insights`, etc.). No `ai_usage` log table found in src scan. | — | — | **Gap** — would need event tracking if AI cost dashboards are wanted |
| **Notifications** | `notifications`, `notification_preferences` (referenced in `create_role_notification` RPC) | — | — | Low |
| **Invitations** | `invitations` | `email, role, organization_id, expires_at, accepted_at, token_hash` | `→ organizations` | Medium |
| **Semesters** | `semesters` | `id, organization_id, is_active` | `→ organizations` | Medium |
| **Tickets / Support** | `platform_tickets` | submitter, status | — | Low |

---

## 5. Existing Analytics and Dashboard Logic

Three live analytics surfaces today:

### 5a. `src/pages/admin/AdminAnalytics.tsx` — Tenant Admin Analytics
- Date range selector (`7d/30d/90d/1y`).
- 4 stat cards: Total Enrollments, Completion Rate, Active Learners, Avg. Progress (all from `get_organization_stats`).
- AreaChart: Enrollment Trend (`get_enrollment_trend`).
- BarChart: Course Completion Rates (`get_course_performance`).
- DonutChart: Progress Distribution (`get_progress_distribution`).
- BarChart: Top Courses by Enrollment (same `coursePerformance` data, sliced).

### 5b. `src/pages/admin/AdminDashboard.tsx` — Tenant Admin Overview
- 4 dashboard cards from `useAdminDashboard`: Total Users / Instructors / Courses / Enrollments (4 raw counts).
- Pending Invitations Card (from `get_invitations_for_admin`, **filtered client-side**).
- Active Semesters Card (`semesters.select.eq(organization_id)`).
- Hardcoded recent activity list (`recentActivity` is mocked, comment says "in production, fetch from audit_logs table" — `audit_logs` table **[NOT FOUND]**).

### 5c. `src/pages/platform/PlatformAnalytics.tsx` — Platform Owner Analytics
- 4 stat cards: Total Users, Total Courses, Organizations, Completion Rate.
- AreaChart: Growth Trend (`get_platform_growth_trend`).
- BarCharts: Tenant Activity, Role Distribution, Category Distribution.

### 5d. `src/pages/instructor/InstructorDashboard.tsx` (uses `useInstructorAnalytics`)
- 5 stat fields: total_students, completion_rate, avg_progress, pending_grades, total_courses.
- Course performance bar chart.
- Recent enrollment activity feed.
- Enrollment trend + progress distribution — **computed in JS from raw enrollment rows** (perf risk; see §7).

### 5e. `src/pages/dashboard/StudentDashboard.tsx` / `StudentInsights.tsx`
- Personal progress, XP, learning summary (via `useLearningAnalytics`).
- Not in v1 OLAP scope; lives on personal `lesson_progress`/`user_experience` (OLTP-friendly).

### Existing metrics catalog (extracted from code)

| Metric | Where calculated | Source RPC/tables | Formula | Safe? | Move to OLAP? |
|---|---|---|---|---|---|
| Total enrollments (org) | `get_organization_stats` | `enrollments WHERE course IN org_courses` | `COUNT(*)` | Yes | Yes |
| Completed enrollments | `get_organization_stats` | same | `COUNT(*) FILTER (completed_at IS NOT NULL)` | Yes | Yes |
| Completion rate (org) | `get_organization_stats` | same | `completed/total * 100` (int) | Yes | Yes |
| Avg progress (org) | `get_organization_stats` | same | `AVG(progress_percentage)` rounded int | Yes | Yes |
| Active learners (org) | `get_organization_stats` | `lesson_progress WHERE last_accessed_at >= NOW()-7d` | `COUNT(DISTINCT user_id)` — **NOT scoped to org** | **Warning** — bug-ish: counts platform-wide active users for any org caller. RLS may hide rows in practice, but the filter should still be explicit. | Yes (fix in gateway) |
| Enrollment trend (daily) | `get_enrollment_trend` | `enrollments` + `generate_series` | per-day `COUNT(id)` | Yes | Yes |
| Course performance (top N) | `get_course_performance` | `courses LEFT JOIN enrollments` | per-course completion% + avg progress | Yes | Yes |
| Progress distribution | `get_progress_distribution` | `enrollments` bucketed 0-25/26-50/51-75/76-100 | bucket counts | Yes | Yes |
| Total users / courses / orgs (platform) | `get_platform_stats` | `profiles, courses, organizations, enrollments` | `COUNT(*)` | Yes | Yes |
| Growth trend (platform) | `get_platform_growth_trend` | `profiles` + `generate_series` | per-day `COUNT(id)` on profiles | Yes | Yes |
| Tenant activity (top N) | `get_tenant_activity` | `organizations LEFT JOIN profiles/courses` | distinct user/course counts | Yes | Yes |
| Role/category distribution | `get_role_distribution` / `get_category_distribution` | `user_roles` / `courses` | `GROUP BY` | Yes | Yes |
| Instructor stats | `get_instructor_stats` | `courses + enrollments + lesson_progress + assignment_submissions` | CTEs incl. avg per-student progress | Yes | Yes |
| Instructor course performance | `get_instructor_course_performance` | same | per-course totals | Yes | Yes |
| At-risk students | `get_instructor_students_at_risk` | `enrollments, lesson_progress, assignment_submissions, quiz_attempts, engagement_events` | rule-based: missing assignments, avg_grade, last activity | Yes (rules are reasonable) | **Yes — should also surface in Tenant Admin v1** |
| Avg session/learning time | `get_learning_summary` | `user_path_progress, user_stage_progress` | `SUM(time_spent_minutes)` over period | Yes | Yes (user-scoped) |

---

## 6. Tenant Isolation Review

### 6a. Safe (tenant_id/org_id/user_id explicit)

| Query | File | Filter |
|---|---|---|
| `get_organization_stats(org_id)` | `useAdminAnalytics.ts` | RPC takes `org_id` from `profile.organization_id` |
| `get_enrollment_trend(org_id, …)` | same | same |
| `get_course_performance(org_id, …)` | same | same |
| `get_progress_distribution(org_id)` | same | same |
| `useAdminDashboard` counts | `useAdminDashboard.ts` | `.eq('organization_id', organizationId)` on `profiles`, `user_roles`, `courses` |
| `useEnrollment` | `useEnrollment.ts` | `.eq('user_id', user.id)` |
| `useProgress` | `useProgress.ts` | `.eq('user_id', user.id)` |
| `track_engagement_event(_user_id, …)` | `useEngagement.ts` | passes `profile.id` |

### 6b. Risky — may leak cross-tenant or rely on RLS alone

| Query | File | Risk | Recommendation |
|---|---|---|---|
| `get_invitations_for_admin()` + JS `.filter(inv.organization_id === organizationId)` | `AdminDashboard.tsx:90` | The RPC returns invitations across all orgs that the caller can see (platform_owner sees everything). For tenant_admin, RLS scopes correctly, but the JS filter is the **only** layer applied. If a tenant_admin ever gains visibility to another org's invitations, this silently passes them. | Add `p_organization_id` arg to RPC or call a gateway endpoint that re-asserts the org. |
| `get_organization_stats` "active_learners" CTE | `public.get_organization_stats` (DB) | `active_users` CTE counts distinct `user_id` from `lesson_progress` globally — not joined to `org_courses`. Caller cannot leak data via RLS, but the **number is wrong** for an org that is not the platform. | Fix RPC to scope by `org_courses` (DB migration, not in this repo to fix). |
| `get_instructor_recent_activity` joins `profiles` for student name/avatar | DB RPC | Acceptable as long as instructor is gated by `instructor_id`. Confirmed. | Keep |
| Raw `enrollments` fetch in `useInstructorAnalytics` (trend + distribution fallbacks) | `useInstructorAnalytics.ts:135-200` | Filters via `IN(courseIds)` after computing `courseIds` from `instructor_id`. Safe as long as RLS on `enrollments` is correct. Currently relies on Postgres RLS. | Move to gateway; do not rely solely on RLS for analytics. |
| `usePrefetch.ts` calls `get_platform_stats` | `usePrefetch.ts:194` | Will fail for non-platform-owners (RLS) but is invoked without a role guard. Should be wrapped in `if (role === 'platform_owner')`. | Minor — silent failure is fine, but waste of round-trip. |
| `audit_logs` table referenced in a comment (`AdminDashboard.tsx:119`) | comment only | Table **[NOT FOUND]** — recent activity is mocked. | Add table or remove the placeholder. |

### 6c. Frontend-only filtering (should not be trusted)

- `AdminDashboard.tsx` filters invitations client-side. Already flagged above.
- No other cases found.

---

## 7. Performance Risks in Current Dashboard/Data Queries

| Risk | File / Function | Why | Short-term fix | OLAP replacement |
|---|---|---|---|---|
| **4 sequential `count: exact, head: true` queries** | `useAdminDashboard.ts` (`profiles`, `user_roles`, `courses`, `enrollments`) | Sequential `await`s — first paint blocks on the slowest. `enrollments` count also requires a prior `courses.select('id')` round-trip. | Run with `Promise.all`; or hit `get_organization_stats` (already exists, just under-used). | `GET /metrics/tenant/summary` — single call. |
| **Client-side bucketing of every enrollment row** | `useInstructorAnalytics.ts:146-171` (enrollment trend) | Pulls every `enrollments.enrolled_at` for instructor's courses, then groups by day in JS. O(N) memory in the browser; grows linearly with enrollments. | Move bucketing into a `get_instructor_enrollment_trend(instructor_id, start, end)` RPC. | `GET /metrics/tenant/enrollments/trend?instructor_id=…` |
| **Client-side progress distribution** | `useInstructorAnalytics.ts:196-211` | Pulls `progress_percentage` for every enrollment. | Add a `get_instructor_progress_distribution(instructor_id)` RPC. | Gateway extension of `…/learners/progress`. |
| **Pulls full invitation list per tenant load** | `AdminDashboard.tsx:84-97` | Re-pulls everything on every visit; JS filter pattern. | Scope at RPC level. | Subsume in `/metrics/tenant/summary`. |
| **Mocked recent activity** | `AdminDashboard.tsx:119-…` | Not a perf risk today but blocks v1 OLAP "Activity Feed" UI. | — | `GET /metrics/tenant/activity?limit=N` (post-MVP). |
| **`usePrefetch` may prefetch RPCs the user can't read** | `usePrefetch.ts:181, 194` | RLS will deny → wasted round-trip. | Role-gate the prefetch. | Same role-gate at gateway client. |
| **`get_instructor_stats` CTE chain** | DB RPC, called from frontend on every dashboard load | Walks `enrollments × lesson_progress × assignments × submissions × quiz_attempts × engagement_events`. Acceptable now; scales poorly. | Cache 5min (already done via `CACHE_TIMES.STANDARD`). | Move to materialized view in warehouse. |
| **Repeated dashboard queries across role pages** | `usePrefetch.ts` warms several RPCs on route change | Defensible while latency is good, but multiplies cost as data grows. | — | Gateway can absorb with its own cache (Redis / CDN). |
| **Fetch-all `lesson_progress` for student dashboard** | `useProgress.ts:46-58` | `select('lesson_id, completed, progress_seconds').eq('user_id', user.id)` — fine for one user, expensive on power users with thousands of lessons. | Add LIMIT or course-scoped path (already partially done). | Out of v1 scope (learner-private). |
| **No pagination on `useInstructorAnalytics` activity** | RPC capped at 5; OK | — | — | Keep. |

---

## 8. Tenant Admin Analytics v1 Proposal

Page-level proposal (uses the **existing** routing/components):

```
/admin                              (existing — AdminDashboard.tsx)
/admin/analytics                    (existing — AdminAnalytics.tsx) → re-wire to Metrics Gateway
/admin/analytics/courses            (new) → Course Analytics deep-dive (per-course detail)
/admin/analytics/learners           (new) → Learner Progress + At-Risk table
/admin/analytics/engagement         (new) → Engagement chart (daily/weekly active)
/admin/analytics/reports            (new, optional v1.5) → Export CSV/PDF
```

All four `/admin/analytics/*` children are gated by the existing `<ProtectedRoute allowedRoles={['tenant_admin']}>` on `/admin`. Add a `<RoleGuard role="tenant_admin">` wrapper inside each page for double-defense (the gateway must also enforce server-side; never trust the frontend filter).

### Components per page

| Page | Components (reuse `src/components/analytics/*` where possible) |
|---|---|
| `/admin/analytics` (overview) | `<SummaryCards />` (replaces 4 `StatCard`s), `<EnrollmentTrendChart />`, `<CoursePerformanceTable />`, `<ProgressDonut />`, `<DateRangeFilter />` |
| `/admin/analytics/courses` | `<CourseFilter />`, `<CoursePerformanceTable />` (sortable, paginated), per-course detail drawer |
| `/admin/analytics/learners` | `<LearnerProgressTable />`, `<AtRiskLearnersTable />` (new — backed by `get_instructor_students_at_risk` adapted to org scope, or a new gateway endpoint), filters by course/cohort |
| `/admin/analytics/engagement` | `<EngagementChart />` (daily/weekly active users), `<SessionDurationChart />` (post-MVP), `<DropOffLessonsTable />` |
| `/admin/analytics/reports` | Export buttons (CSV/PDF) calling gateway export endpoints |

Always include: **EmptyState**, **LoadingSkeleton** (the existing `StatCard` already supports `isLoading`), **ErrorBoundary** (reuse `src/components/QueryErrorBoundary.tsx`), and **DateRangeFilter** (already exists as `DateRangeSelector`).

---

## 9. Metrics Catalog (v1)

| Metric | Definition | Current source in repo | Current tables/RPCs | Future Metrics Gateway endpoint | Status |
|---|---|---|---|---|---|
| Total learners | distinct `user_id` with role `student` in org | `useAdminDashboard.totalUsers` (loosely) | `profiles WHERE organization_id` (counts all profiles, not just students) | `GET /metrics/tenant/summary` | **Needs DB confirmation** (filter by `user_roles.role='student'`) |
| Active learners | distinct `user_id` in `lesson_progress` last 7d **in org** | `get_organization_stats.active_learners` | `lesson_progress` (NOT scoped to org — see §6b) | `GET /metrics/tenant/summary` | **Needs backend aggregation fix** before going to OLAP |
| Total courses | count of `courses WHERE organization_id` | `useAdminDashboard.courses` | `courses` | `GET /metrics/tenant/summary` | Ready now |
| Total enrollments | count of `enrollments` where course ∈ org | `get_organization_stats.total_enrollments` | `enrollments` | `GET /metrics/tenant/summary` | Ready now |
| Course completions | `enrollments WHERE completed_at IS NOT NULL` | `get_organization_stats.completed_enrollments` | same | `GET /metrics/tenant/summary` | Ready now |
| Completion rate | completed / total × 100 | `get_organization_stats.completion_rate` | same | `GET /metrics/tenant/summary` | Ready now |
| Average course progress | `AVG(progress_percentage)` | `get_organization_stats.avg_progress` | `enrollments` | `GET /metrics/tenant/summary` | Ready now |
| Lesson completion rate | completed lessons / total lessons (per user / per course) | NOT directly exposed — computed inside `get_instructor_stats` CTE | `lesson_progress, lessons, modules` | `GET /metrics/tenant/learners/progress` (new aggregate) | Needs backend aggregation |
| Assessment pass rate | `quiz_attempts WHERE passed=true / total` | NOT directly exposed | `quiz_attempts` (insert only in `QuizPlayer.tsx`) | `GET /metrics/tenant/engagement` | Needs backend aggregation + event coverage |
| Daily active learners (DAL) | distinct user_id with any event in last 24h | NOT exposed | `engagement_events` (partial coverage) | `GET /metrics/tenant/engagement?granularity=day` | Needs event tracking + aggregation |
| Weekly active learners (WAL) | same, 7d | NOT exposed | same | same with `granularity=week` | Needs event tracking + aggregation |
| At-risk learners | rule-based: missing assignments >3, avg_grade <60, no activity 14d | `get_instructor_students_at_risk` (instructor scope only) | `enrollments + assignment_submissions + lesson_progress + quiz_attempts + engagement_events` | `GET /metrics/tenant/learners/progress?at_risk=true` | **Needs DB confirmation** (org-scope variant) — UI gap in v1 |
| Top courses (by enrollment) | top N by `COUNT(enrollments)` | `get_course_performance` (already used in `AdminAnalytics`) | `courses, enrollments` | `GET /metrics/tenant/courses/performance` | Ready now |
| Drop-off lessons | lessons where `(started − completed)/started` is highest | NOT exposed | `lesson_progress` (needs `lesson_started` event) | `GET /metrics/tenant/engagement` (extension) | **Needs event tracking** |
| Enrollment trend | per-day enrollment count | `get_enrollment_trend` | `enrollments` | `GET /metrics/tenant/enrollments/trend` | Ready now |
| Course completion trend | per-day completion count | NOT exposed | `enrollments.completed_at` | `GET /metrics/tenant/enrollments/trend?type=completions` | Post-MVP |
| Revenue / subscription metrics | — | NOT FOUND | — | — | Post-MVP (no payment integration yet) |
| Certificates issued | — | NOT FOUND | inferred from `completed_at` | — | Needs DB confirmation |

---

## 10. Future Metrics Gateway API Usage

Endpoint contract (provided by the user) and how this repo will consume each:

| Gateway endpoint **[GATEWAY]** | Replaces in this repo | LMS-side hook |
|---|---|---|
| `GET /metrics/tenant/summary?range=30d` | `useAdminDashboard` + `get_organization_stats` half of `useAdminAnalytics` | `useTenantSummary(range)` |
| `GET /metrics/tenant/enrollments/trend?range=30d&granularity=day` | `get_enrollment_trend` | `useEnrollmentTrend(range)` |
| `GET /metrics/tenant/courses/performance?limit=10&sort=completion_rate` | `get_course_performance` | `useCoursePerformance(opts)` |
| `GET /metrics/tenant/learners/progress?range=30d&at_risk=false&page=1` | `get_progress_distribution` + extension | `useLearnerProgress(opts)` |
| `GET /metrics/tenant/engagement?range=30d&granularity=day` | (new — no current source) | `useEngagement(range)` |
| `GET /metrics/course/:courseId/summary` | (new — would replace per-course details currently absent) | `useCourseSummary(courseId)` |
| `GET /metrics/user/:userId/progress` | `get_learning_summary` (via `useLearningAnalytics`) | `useUserProgress(userId)` |

### API client design (`src/services/metricsApi.ts`)

```ts
// Base URL: import.meta.env.VITE_METRICS_API_URL ?? import.meta.env.VITE_GATEWAY_URL
//
// Auth: forward the Supabase session JWT as `Authorization: Bearer <token>`.
// The gateway verifies the JWT against Supabase JWKS and extracts org_id from
// the user_roles join — frontend MUST NOT send org_id as a query param.
//
// Errors: throw MetricsApiError with .status, .code, .traceId.
// Timeouts: 10s default; AbortController per call.
// Retries: 2x on 5xx / network with exponential backoff (250, 500ms).
// Mock fallback: if VITE_METRICS_API_URL is unset OR localStorage.metricsMock === '1',
//                return fixtures from src/services/__fixtures__/metrics/*.json.
```

### Hooks contract

| Hook | Params | Output | Loading | Error | Cache (staleTime) | Endpoint |
|---|---|---|---|---|---|---|
| `useTenantSummary` | `range: DateRange` | `TenantSummaryMetrics` | `isLoading` | `MetricsApiError | null` | `STANDARD` (5min) | `/metrics/tenant/summary` |
| `useEnrollmentTrend` | `range, granularity` | `EnrollmentTrendPoint[]` | `isLoading` | same | `STANDARD` | `/metrics/tenant/enrollments/trend` |
| `useCoursePerformance` | `{ limit, sort, courseFilter? }` | `CoursePerformanceMetric[]` | `isLoading` | same | `LONG` (15min) | `/metrics/tenant/courses/performance` |
| `useLearnerProgress` | `{ range, atRisk?, page, courseId? }` | `Paginated<LearnerProgressMetric>` | `isLoading` | same | `STANDARD` | `/metrics/tenant/learners/progress` |
| `useEngagementMetric` | `range, granularity` | `EngagementMetric[]` | `isLoading` | same | `STANDARD` | `/metrics/tenant/engagement` |
| `useCourseSummary` | `courseId` | `CourseSummary` | `isLoading` | same | `LONG` | `/metrics/course/:id/summary` |
| `useUserProgressMetric` | `userId` | `UserProgressMetric` | `isLoading` | same | `STANDARD` | `/metrics/user/:id/progress` |

---

## 11. Recommended Frontend File Structure

Aligned with existing conventions (`src/hooks/analytics/*`, `src/components/analytics/*`, page-per-route, central `queryKeys` and `queryClient`).

```
src/
  services/
    metricsApi.ts                       (new) — fetch client w/ auth, retries, error class
    metricsApi.types.ts                 (new) — TS interfaces (see §12)
    __fixtures__/metrics/
      tenant-summary.json               (new) — local-dev mock
      enrollment-trend.json
      courses-performance.json
      learners-progress.json
      engagement.json
  hooks/analytics/
    useTenantSummary.ts                 (new)
    useEnrollmentTrend.ts               (new)
    useCoursePerformance.ts             (new — supersedes part of useAdminAnalytics)
    useLearnerProgress.ts               (new)
    useEngagementMetric.ts              (new)
    useCourseSummary.ts                 (new)
    useUserProgressMetric.ts            (new)
    useAdminAnalytics.ts                (KEEP, deprecate in 2-step migration)
    useAdminDashboard.ts                (REWIRE to useTenantSummary)
    usePlatformAnalytics.ts             (KEEP — post-MVP swap)
    useInstructorAnalytics.ts           (KEEP — post-MVP swap)
  components/analytics/
    SummaryCards.tsx                    (new — composed of existing StatCard)
    EnrollmentTrendChart.tsx            (new — wraps AreaChartCard)
    CoursePerformanceTable.tsx          (new — sortable table)
    LearnerProgressTable.tsx            (new)
    AtRiskLearnersTable.tsx             (new — flag, badge, drill-in)
    EngagementChart.tsx                 (new — wraps AreaChartCard)
    ExportMenu.tsx                      (new — placeholder, post-MVP)
    StatCard.tsx                        (existing — keep)
    AreaChartCard.tsx                   (existing — keep)
    BarChartCard.tsx                    (existing — keep)
    DonutChartCard.tsx                  (existing — keep)
    DateRangeSelector.tsx               (existing — keep)
    ActivityFeed.tsx                    (existing — keep)
  pages/admin/analytics/
    AnalyticsOverview.tsx               (new — replaces AdminAnalytics.tsx, behind a feature flag during migration)
    AnalyticsCourses.tsx                (new)
    AnalyticsLearners.tsx               (new)
    AnalyticsEngagement.tsx             (new)
    AnalyticsReports.tsx                (new, optional v1.5)
  lib/
    queryKeys.ts                        (extend with queryKeys.metrics.* tree)
    queryClient.ts                      (unchanged; reuse CACHE_TIMES)
```

Route registration (in `src/App.tsx` — additive, no existing routes removed):

```tsx
<Route path="/admin" element={<ProtectedRoute allowedRoles={['tenant_admin']}><DashboardLayout /></ProtectedRoute>}>
  …existing…
  <Route path="analytics" element={<AnalyticsOverview />} />            {/* was AdminAnalytics */}
  <Route path="analytics/courses" element={<AnalyticsCourses />} />
  <Route path="analytics/learners" element={<AnalyticsLearners />} />
  <Route path="analytics/engagement" element={<AnalyticsEngagement />} />
  <Route path="analytics/reports" element={<AnalyticsReports />} />
</Route>
```

---

## 12. TypeScript Data Contracts

```ts
// src/services/metricsApi.types.ts

export type DateRange = '7d' | '30d' | '90d' | '1y' | 'custom';

export interface DateRangeFilter {
  range: DateRange;
  from?: string;   // ISO, required when range==='custom'
  to?: string;
}

export interface TenantSummaryMetrics {
  organizationId: string;
  range: DateRange;
  totals: {
    learners: number;            // students in org
    activeLearners: number;      // distinct user_id with event in window
    instructors: number;
    courses: number;
    enrollments: number;
    completedEnrollments: number;
  };
  rates: {
    completionRate: number;          // 0-100
    avgCourseProgress: number;       // 0-100
    lessonCompletionRate: number;    // 0-100
    assessmentPassRate?: number;     // 0-100, optional v1
  };
  asOf: string; // ISO timestamp of latest source data
}

export interface EnrollmentTrendPoint {
  date: string;        // YYYY-MM-DD
  enrollments: number;
  completions?: number;
}

export interface CoursePerformanceMetric {
  courseId: string;
  courseTitle: string;
  totalEnrollments: number;
  completedEnrollments: number;
  completionRate: number;     // 0-100
  avgProgress: number;        // 0-100
  avgAssessmentScore?: number;
}

export type RiskLevel = 'low' | 'medium' | 'high';

export interface LearnerProgressMetric {
  userId: string;
  fullName: string | null;
  avatarUrl: string | null;
  courseId: string;
  courseTitle: string;
  progressPercentage: number;
  lastActivityAt: string | null;
  missingSubmissions: number;
  averageGrade: number | null;
  riskLevel: RiskLevel;
  reasons: string[];
}

export interface EngagementMetric {
  date: string;
  dailyActiveLearners: number;
  weeklyActiveLearners?: number;
  sessions: number;
  avgSessionMinutes: number;
}

export interface CourseSummary {
  courseId: string;
  courseTitle: string;
  totals: { enrollments: number; completions: number; assignments: number };
  rates: { completionRate: number; passRate: number; avgProgress: number };
  dropOffLessons: Array<{ lessonId: string; lessonTitle: string; dropOffRate: number }>;
}

export interface UserProgressMetric {
  userId: string;
  totals: { coursesEnrolled: number; coursesCompleted: number; lessonsCompleted: number; xp: number };
  recent: { lastActivityAt: string | null; sessionsLast7d: number; minutesLast7d: number };
  perCourse: Array<{ courseId: string; courseTitle: string; progressPercentage: number; completedAt: string | null }>;
}

export interface Paginated<T> {
  rows: T[];
  page: number;
  pageSize: number;
  total: number;
}

export class MetricsApiError extends Error {
  constructor(
    message: string,
    public readonly status: number,
    public readonly code?: string,
    public readonly traceId?: string,
  ) {
    super(message);
    this.name = 'MetricsApiError';
  }
}
```

---

## 13. Event Tracking Gaps

Currently emitted (verified in `src/hooks/useEngagement.ts` / `useSprintTimer.tsx`):

- `session_start`, `session_end` (auto on `useEngagementTracking` mount/unmount, 3s deferred)
- `content_view`, `content_complete` (convenience methods exposed; **call sites not enforced**)
- `assessment_start`, `assessment_complete` (same — convenience methods)
- `streak_update`, `milestone_reached`, `goal_progress`, `inactivity_detected` (declared, **no callers found** in the React app)
- `nudge_sent`, `nudge_opened`, `nudge_acted`, `achievement_unlocked` (declared, callers in edge functions / nudge flow only)

Missing or weakly emitted (for v1 OLAP):

| Event | Where to emit (file/function) | Required payload | Tenant/org/user/course/lesson | v1 / future |
|---|---|---|---|---|
| `lesson_started` | `src/pages/dashboard/LessonPlayer.tsx` on first paint / first `play` | `{ lesson_id, module_id, course_id }` | user_id (auto), course_id (req), lesson_id (req) | **v1** |
| `lesson_completed` | `src/pages/dashboard/LessonPlayer.tsx` on `lesson_progress.completed=true` upsert | `{ lesson_id, course_id, time_spent_seconds }` | same | **v1** |
| `course_started` | `src/hooks/useEnrollment.ts` on successful `enrollments.insert` | `{ course_id, source: 'catalog'|'recommendation'|'admin' }` | course_id (req) | **v1** |
| `course_completed` | server-side trigger on `enrollments.completed_at` set OR client emit in `useProgress` when all lessons done | `{ course_id, completion_pct, duration_days }` | course_id (req) | **v1** |
| `assessment_started` | `src/components/learning/QuizPlayer.tsx` `start()` | `{ quiz_id, lesson_id?, assignment_id? }` | assignment/lesson ids | **v1** |
| `assessment_submitted` | `QuizPlayer.tsx` on insert into `quiz_attempts` | `{ quiz_id, score, max_score, passed, time_spent_seconds }` | same | **v1** |
| `assignment_submitted` | `src/hooks/useAssignmentSubmission.ts` | `{ assignment_id, course_id }` | course_id | **v1** |
| `video_played` | `LessonPlayer.tsx` on `<video>` `play` | `{ lesson_id, course_id, position_seconds }` | lesson/course | future |
| `video_completed` | `LessonPlayer.tsx` on `ended` | `{ lesson_id, course_id, duration_seconds }` | same | future |
| `login_success` | `useAuth.tsx` `signIn` happy path | `{ method: 'password'|'oauth' }` | user_id | **v1** (login funnel) |
| `certificate_issued` | **[INFERRED]** — no certificate flow found; should be emitted from edge function if certificates exist | `{ course_id, certificate_id }` | course_id | future / needs feature confirmation |
| `course_dropped` | optional UX action / inactivity trigger | `{ course_id, reason }` | course_id | future |

All events should already flow through `track_engagement_event(_user_id, _event_type, _event_data, _source, _session_id)` — no schema change needed. Make sure `event_data.course_id` and `event_data.organization_id` are populated so the warehouse can shard cleanly.

---

## 14. Migration Plan

Two-phase migration. Both phases are reversible per query.

### Phase 1 — Tenant Admin (replace now)

| # | File | Today | After |
|---|---|---|---|
| 1 | `src/hooks/analytics/useAdminAnalytics.ts` (stats) | `supabase.rpc('get_organization_stats')` | `useTenantSummary(range)` → `GET /metrics/tenant/summary` |
| 2 | `src/hooks/analytics/useAdminAnalytics.ts` (trend) | `supabase.rpc('get_enrollment_trend')` | `useEnrollmentTrend(range)` |
| 3 | `src/hooks/analytics/useAdminAnalytics.ts` (course perf) | `supabase.rpc('get_course_performance')` | `useCoursePerformance(opts)` |
| 4 | `src/hooks/analytics/useAdminAnalytics.ts` (distribution) | `supabase.rpc('get_progress_distribution')` | `useLearnerProgress({ shape: 'distribution' })` |
| 5 | `src/hooks/analytics/useAdminDashboard.ts` (4 counts) | 4× `from(...).select(count:exact)` | `useTenantSummary(range)` (same data, 1 call) |
| 6 | `src/pages/admin/AdminDashboard.tsx` (invitations card) | `rpc('get_invitations_for_admin')` + JS filter | Gateway `summary.invitations` (folded into summary) **OR** keep as-is but pass `organization_id` arg once added to RPC |
| 7 | `src/hooks/usePrefetch.ts` (admin path) | `rpc('get_organization_stats')` | `metricsApi.getTenantSummary(range)` prefetch |

Replace as a duplicate (delete-after-cutover):

- The "Top Courses by Enrollment" chart in `AdminAnalytics.tsx` is just `coursePerformance` re-sliced. Drop the duplicate and render two views from one fetch.

### Phase 2 — Platform & Instructor (replace later)

| # | File | Action |
|---|---|---|
| 8 | `src/hooks/analytics/usePlatformAnalytics.ts` | Swap all 5 RPCs to `/metrics/platform/*` once gateway exposes them |
| 9 | `src/hooks/analytics/useInstructorAnalytics.ts` (RPCs) | Swap to `/metrics/instructor/*` |
| 10 | `src/hooks/analytics/useInstructorAnalytics.ts` (raw enrollment fallback) | **Delete client-side bucketing**; rely on gateway endpoints |

Keep as-is (transactional, not analytics):

- `useEnrollment`, `useProgress`, `useAuth`, `useAssignmentSubmission`, `useNotifications`, `useRewards`, all storage uploaders, all `supabase.functions.invoke(...)` AI calls.

Suggested order: 1 → 5 → 6 → 2 → 3 → 4 → 7 → (validate) → 8 → 9 → 10.

---

## 15. Security Checklist

- [x] **No service-role key in frontend.** Verified: `src/integrations/supabase/client.ts` reads `VITE_SUPABASE_PUBLISHABLE_KEY` only. Edge functions use `SUPABASE_SERVICE_ROLE_KEY` server-side.
- [ ] **Tenant ID must come from session, not query param.** Today the frontend passes `profile.organization_id` to RPCs — that value is loaded via `get_user_context` RPC (server-verified). When wiring the Metrics Gateway, **do NOT** add `?organizationId=` query params; let the gateway derive `org_id` from the verified JWT (it has `user_id` → `user_roles.organization_id`).
- [x] **Role check before analytics page access.** `<ProtectedRoute allowedRoles={['tenant_admin']}>` wraps `/admin`. Add per-route `<RoleGuard>` inside each new `/admin/analytics/*` page for defense-in-depth.
- [ ] **Metrics Gateway must enforce tenant isolation server-side.** The frontend cannot be trusted with the org filter. Gateway must validate JWT, resolve `organization_id` from `user_roles`, and reject any client-supplied org param mismatch.
- [ ] **Frontend should not trust hidden filters for security.** Already a concrete risk in `AdminDashboard.tsx:90` (client-side invitation filter). Move to RPC-arg or gateway summary.
- [ ] **Avoid exposing PII in analytics tables.** Learner tables (`LearnerProgressTable`, at-risk) show `full_name` + `avatar_url` — acceptable for an admin viewing their own org. Gateway should redact email and DOB unless explicitly requested with audit log.
- [ ] **Avoid raw event logs in UI.** Don't render `engagement_events` rows directly; only aggregates.
- [ ] **CORS lockdown on Metrics Gateway.** The earlier Railway CORS failure (`api-gateway-production-edc4.up.railway.app`) is a reminder: gateway must allow `*.lovable.app`, `app.pathwisse.com`, and localhost dev origins, with credentials forwarded and `OPTIONS` returning before auth middleware runs.
- [ ] **JWT propagation.** The Metrics Gateway client must read the live token via `supabase.auth.getSession()` on every call (not cache it) so revocation/rotation works.
- [ ] **Rate limits.** Per-user + per-org throttling on the gateway (e.g., 60 rpm/user, 600 rpm/org).
- [ ] **`get_organization_stats.active_learners` cross-tenant scope bug** (§6b) — fix in DB before exposing via gateway, otherwise the gateway will inherit the bug.

---

## 16. Recommended Next 10 Engineering Tickets

Scoped strictly to this LMS repo per the open question default.

### TKT-1 — Add Metrics Gateway client skeleton
- **Goal:** Provide a typed, reusable `metricsApi` client with auth, retries, error class, and mock fallback.
- **Files:** `src/services/metricsApi.ts` (new), `src/services/metricsApi.types.ts` (new), `src/services/__fixtures__/metrics/*.json` (new), `.env.example` (add `VITE_METRICS_API_URL`).
- **Acceptance:** `metricsApi.getTenantSummary('30d')` returns mock data when `VITE_METRICS_API_URL` unset; forwards `Authorization: Bearer <jwt>` and a `x-trace-id` when set; throws `MetricsApiError` on non-2xx; retries 2× on 5xx.
- **Risk:** Low.

### TKT-2 — `useTenantSummary` hook + extend `queryKeys`
- **Goal:** First Gateway-backed hook; basis for all later swaps.
- **Files:** `src/hooks/analytics/useTenantSummary.ts` (new), `src/lib/queryKeys.ts` (extend), tests next to it.
- **Acceptance:** Hook returns `TenantSummaryMetrics` with `isLoading`/`error`; uses `CACHE_TIMES.STANDARD`; covered by a unit test with mocked client.
- **Risk:** Low.

### TKT-3 — Rewire `AdminDashboard` to `useTenantSummary`
- **Goal:** Replace 4 sequential `count: exact` queries + invitations card with a single gateway call.
- **Files:** `src/pages/admin/AdminDashboard.tsx`, `src/hooks/analytics/useAdminDashboard.ts` (delete or wrap).
- **Acceptance:** Page renders identical numbers; network panel shows 1 gateway call instead of 4 Supabase calls; loading state preserved.
- **Risk:** Medium (visible page).

### TKT-4 — `useEnrollmentTrend` + `useCoursePerformance` hooks
- **Goal:** Complete the data layer needed for `/admin/analytics`.
- **Files:** `src/hooks/analytics/useEnrollmentTrend.ts`, `src/hooks/analytics/useCoursePerformance.ts`.
- **Acceptance:** Both hooks pass `range`/`limit` correctly, parse responses to existing chart shapes, covered by unit tests.
- **Risk:** Low.

### TKT-5 — Rewire `AdminAnalytics` to gateway hooks (Phase 1 cutover)
- **Goal:** Replace 4 RPC calls in `useAdminAnalytics` with gateway hooks behind a feature flag.
- **Files:** `src/pages/admin/AdminAnalytics.tsx` → renamed to `src/pages/admin/analytics/AnalyticsOverview.tsx`; `src/hooks/analytics/useAdminAnalytics.ts` deprecated.
- **Acceptance:** Feature flag `VITE_FEATURE_METRICS_GATEWAY=true` flips data source; numbers match within rounding; can roll back by unsetting the flag.
- **Risk:** Medium.

### TKT-6 — Add `/admin/analytics/learners` page with `useLearnerProgress`
- **Goal:** New learner table + at-risk surface for tenant admins.
- **Files:** `src/pages/admin/analytics/AnalyticsLearners.tsx`, `src/hooks/analytics/useLearnerProgress.ts`, `src/components/analytics/LearnerProgressTable.tsx`, `src/components/analytics/AtRiskLearnersTable.tsx`, route in `src/App.tsx`.
- **Acceptance:** Paginated table with course filter; risk-level chips; empty/loading/error states; protected by `tenant_admin`.
- **Risk:** Medium.

### TKT-7 — Emit `lesson_started`, `lesson_completed`, `course_started`, `course_completed` events
- **Goal:** Plug the four most-missing events to make warehouse-side aggregates trustworthy.
- **Files:** `src/pages/dashboard/LessonPlayer.tsx`, `src/hooks/useEnrollment.ts`, `src/hooks/useProgress.ts`.
- **Acceptance:** Events visible in `engagement_events` with `event_data.course_id` populated; no duplicate emission on remount (use ref-guard like `useEngagementTracking` already does).
- **Risk:** Medium (touches learner flow; gate behind try/catch — engagement is non-critical).

### TKT-8 — Emit `assessment_started`/`assessment_submitted` + `assignment_submitted`
- **Goal:** Assessment funnel and grading metrics.
- **Files:** `src/components/learning/QuizPlayer.tsx`, `src/hooks/useAssignmentSubmission.ts`.
- **Acceptance:** Each event carries `quiz_id`/`assignment_id`, `score` (where known), `lesson_id`/`course_id`; fires once per attempt; covered by integration test.
- **Risk:** Low.

### TKT-9 — Fix client-side invitation filtering in `AdminDashboard`
- **Goal:** Remove cross-tenant filter-in-JS pattern.
- **Files:** `src/pages/admin/AdminDashboard.tsx` (lines 84-117, 200). DB migration (separate ticket if needed) to add `p_organization_id` arg to `get_invitations_for_admin`.
- **Acceptance:** RPC accepts org arg; no client-side `.filter(inv.organization_id === ...)`; same UI behavior.
- **Risk:** Low–Medium (requires migration on the DB side, but `AdminDashboard` change is small).

### TKT-10 — Add gateway-aware `<RoleGuard>` wrappers + `MetricsErrorBoundary` to all `/admin/analytics/*` pages
- **Goal:** Defense-in-depth for the new analytics surface.
- **Files:** `src/components/RoleGuard.tsx` (extend), `src/components/QueryErrorBoundary.tsx` (subclass into `MetricsErrorBoundary`), apply to all new pages.
- **Acceptance:** Non-`tenant_admin` users get a 403-style "Not authorized" view; gateway 401/403 surfaces as a friendly retry/login prompt, not a raw error; existing UX for other pages unchanged.
- **Risk:** Low.

---

## Appendix A — File inventory (analytics-relevant)

```
Pages:        src/pages/admin/AdminAnalytics.tsx
              src/pages/admin/AdminDashboard.tsx
              src/pages/platform/PlatformAnalytics.tsx
              src/pages/platform/PlatformDashboard.tsx
              src/pages/instructor/InstructorDashboard.tsx
              src/pages/dashboard/StudentDashboard.tsx
              src/pages/dashboard/StudentInsights.tsx

Hooks:        src/hooks/analytics/useAdminAnalytics.ts
              src/hooks/analytics/useAdminDashboard.ts
              src/hooks/analytics/useInstructorAnalytics.ts
              src/hooks/analytics/usePlatformAnalytics.ts
              src/hooks/analytics/index.ts
              src/hooks/useLearningAnalytics.ts
              src/hooks/useEngagement.ts
              src/hooks/useEnrollment.ts
              src/hooks/useProgress.ts
              src/hooks/useLearningInsights.ts
              src/hooks/usePrefetch.ts

Components:   src/components/analytics/ActivityFeed.tsx
              src/components/analytics/AreaChartCard.tsx
              src/components/analytics/BarChartCard.tsx
              src/components/analytics/DateRangeSelector.tsx
              src/components/analytics/DonutChartCard.tsx
              src/components/analytics/StatCard.tsx
              src/components/analytics/index.ts

Infra:        src/integrations/supabase/client.ts
              src/lib/queryClient.ts
              src/lib/queryKeys.ts
              src/components/ProtectedRoute.tsx
              src/components/RoleGuard.tsx
              src/components/QueryErrorBoundary.tsx
```

## Appendix B — Analytics RPCs found in DB (FOUND)

`get_platform_stats`, `get_organization_stats`, `get_instructor_stats`, `get_instructor_course_performance`, `get_course_performance`, `get_enrollment_trend`, `get_platform_growth_trend`, `get_progress_distribution`, `get_role_distribution`, `get_category_distribution`, `get_tenant_activity`, `get_learning_summary`, `get_instructor_students_at_risk`, `get_instructor_recent_activity`, `get_instructor_assignments`, `get_invitations_for_admin`, `get_org_profiles_for_admin`, `get_user_context`, `add_user_xp`, `track_engagement_event`, `complete_onboarding`, `admin_create_user`, `grade_submission_with_xp`, `create_role_notification`, `has_role`, `get_user_role`, `get_user_organization`.

All RPCs are `SECURITY DEFINER` with `SET search_path = public` (verified in schema dump).
