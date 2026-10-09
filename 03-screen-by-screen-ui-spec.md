# Duo Leveling — Screen-by-Screen UI Specification

Status: product interaction blueprint, web-first responsive MVP. Visual tokens follow the source specification: Obsidian `#0B0D12`, Graphite `#171B24`, System Blue `#63A9FF`, Arcane Violet `#AD91FF`.

## 1. Global shell and conventions

### Information architecture

Primary routes: `/system`, `/character`, `/quests`, `/roadmap`, `/duo`, `/finance`, `/analytics`, `/achievements`, `/settings`. Nested routes may show detail pages, but quest completion and common edits use side panels/dialogs and preserve current context.

### Desktop layout

- Fixed left navigation rail with brand/system status, route links, current user's character identity and settings entry.
- Main content has a constrained readable width; dashboard and analytics may use wider grids.
- Right-side context panel is reserved for selected quest/goal detail, alerts, or activity context.
- Header includes page title, date/week context, notifications, and quick-create action.

### Mobile layout

- Compact top bar with page title and notification affordance.
- Bottom navigation for System, Quests, Roadmap, Duo; secondary pages under a labeled More menu.
- Panels become full-height sheets; critical actions remain reachable without horizontal scrolling.
- Finance tables become stacked transaction rows with amount, currency, category, and date clearly labeled.

### Shared interaction rules

- Every screen has loading skeletons, empty states with an actionable next step, recoverable error states, and permission-denied states that do not disclose whether a private record exists.
- Use keyboard-operable controls, visible focus, semantic headings, accessible names, sufficient contrast, and non-color status cues. Respect `prefers-reduced-motion` and the in-app setting.
- Use optimistic updates only for low-risk presentation state. Quest completion, finance mutations, sharing changes, and ratings wait for server confirmation.
- Confirm destructive or privacy-impacting changes and explain what access changes. Re-fetch sensitive content after permission updates.
- Avoid sensitive quest titles, finance amounts, or notes in browser notifications/analytics unless the user explicitly opts in.

## 2. System Dashboard — `/system`

### Layout and components

1. System header: `SYSTEM ONLINE`, local date, greeting, system alerts.
2. Player status card: character name/portrait, level, current XP and next-level progress bar, level threshold explanation.
3. Life Rating card: score or `Unassessed`, confidence label, assessed domains count, delta from previous snapshot, “Why did this change?” link.
4. Daily quests: due today and overdue/rescheduled groups, reward, difficulty, goal link, completion action.
5. Duo status: duo level/XP and shared-visible next milestone; if no duo, invitation/setup card.
6. Upcoming goals/milestones and deadlines.
7. Recent system events, filtered to data visible to current user.
8. Quick actions: Create quest, Record transaction, Add goal, Start review. Hide finance action when finance is disabled.

### States and behavior

- First use: onboarding continuation card; no invented rating, XP, or “day 1” facts.
- No daily quests: show a short add/schedule action and today's planned total.
- Unassessed rating: explain how to establish an optional baseline.
- Completion: open confirmation sheet if evidence/partner confirmation is required; otherwise call completion RPC, show returned XP and level transition, refresh cards and event feed.
- API failure: retain unsent form input and show retry; never display XP before server confirmation.
- No duo: personal dashboard remains fully usable and does not block onboarding.

## 3. Character Profile — `/character`

### Layout and components

- Character header with avatar, display name/title, level, total XP, next threshold, and optional cosmetic controls.
- Seven attribute cards (Mind, Discipline, Health, Career, Finance, Relationships, Creativity), each with rating/unassessed status, trend, confidence, and explanation affordance.
- Overall Life Rating panel with provisional/supported status and assessed-domain count.
- Historical rating/XP progression chart with accessible data table alternative and selectable time range.
- Achievement/title collection and recent milestone timeline.
- Mastery checklist visible at level 99; explain four-domain requirement and review of remaining domains.

### Interaction and states

- Selecting an attribute opens a detail sheet showing indicators, weights, source/evidence policy, last update, algorithm version, and exact calculation.
- Empty profile prompts user to set display preferences and optionally assess domains; no attribute is defaulted to zero.
- Share controls deep-link to Settings and name exactly which category will be shared.
- Chart includes loading, no-history, error, and reduced-motion accessible states.

## 4. Quest Board — `/quests`

### Primary views

- List: grouped by status/date, supports sorting and bulk reschedule where safe.
- Calendar: week/month, local time zone, unscheduled queue.
- Kanban: Draft, Scheduled, Active, Pending confirmation, Completed; overdue/skipped/cancelled accessible via filters rather than misleading columns.

### Filters and creation

Filters: personal/duo, cadence, attribute/domain, priority, difficulty, status, date range, parent goal, assignee. Persist filters in URL/query state.

Create flow: scope/owner → objective and goal → difficulty/reward preview → due/recurrence and time zone → evidence/confirmation rules → review and save. Reward preview is labeled as a server-calculated estimate and becomes frozen at approval.

### Quest detail panel

Objective, owner/scope, assignee, parent goal, difficulty/reward, due date/time zone, recurrence rule, subtasks, evidence, completion history, permitted collaborators, and edit audit. Show private information only if authorized.

### Completion flow

- Ordinary personal quest: one explicit Complete action, then server confirmation and XP result.
- Required evidence: attach from private storage or choose permitted evidence type before submitting.
- Duo confirmation: show who must confirm and whether state is pending; no duo XP until all required conditions pass.
- Recurring quest: explain that this scheduled occurrence is being completed; never create bonus instances client-side.
- Duplicate request: show completed result, do not animate or award a second time.

## 5. Life Roadmap — `/roadmap`

### Layout

- Vision selector for 1-, 3-, and 5-year horizons.
- Expandable hierarchy: Vision → Annual Goal → Quarterly Milestone → Monthly Goal → Weekly Target → Daily Quest.
- Goal detail pane: target outcome, baseline/current/target, unit, target date/time zone, owner, status, progress method, dependencies, sharing, related quests, and history.
- Timeline view and nested outline view; mobile defaults to outline.

### Interactions/states

- Add goal at a chosen hierarchy level; forms adapt fields by goal type.
- Numeric goals show calculated progress and formula; milestone goals show weighted checkpoints.
- A required incomplete dependency blocks full completion; explain the dependency and provide a link to it.
- An authorized definition change requires reason, preview of effect, and audit confirmation.
- Cycle detection prevents circular dependencies. Empty roadmap guides the user through creating a vision and a measurable first goal.

## 6. Duo Hub — `/duo`

### Layout

- Duo identity and level/XP, shared milestone unlocks, and duo event feed.
- Partner cards show only each person's explicitly shared status; privacy badges link to current sharing choices.
- Shared goals, missions, milestones, decision/planning log, and upcoming dates.
- Optional Duo Treasury summary only if the viewer has explicit permission.

### States and behavior

- No duo: create duo or accept invitation; personal use remains available.
- Invitation pending/expired: show safe state and resend/revoke action to authorized inviter.
- One partner has not shared a category: show “Not shared,” not an inferred score or private field.
- Shared mission completion shows confirmation requirements and duo reward state.
- Leaving/disconnecting requires a clear impact summary; private records remain owned by their original owner.
- Duo milestones do not unlock private data or alter either member's individual permission settings.

## 7. Treasury — `/finance`

### Sections

1. Overview: cash flow, savings rate (clearly labeled net-cash-flow-based), account balances/net worth only where included and current, and financial goals.
2. Transactions: searchable/filterable ledger with income, expenses, transfers, refunds, adjustments; currency always visible.
3. Budgets: category limits, period, actual-versus-target and favorable/unfavorable explanation.
4. Goals: savings, debt reduction, purchases, business revenue; actual versus target.
5. Reports: currency-separated trends and period comparison.
6. Duo Treasury: separate section reached only when explicit duo treasury permission exists.

### Forms and protections

- Transaction form: account, type, amount, currency, category, date, description, related transaction for refund, paired destination account for transfer.
- Transfer entry creates both linked sides atomically and is excluded from income/expense.
- Show zero-income savings rate as `N/A`; never divide by zero.
- Currency conversion is absent unless a supported rate and date are explicitly selected.
- Private financial transaction details are never exposed through duo summary permission. Summary sharing returns aggregates only.
- Empty state offers manual account setup; never prompt for bank credentials.
- Errors preserve form values but indicate whether the server saved the transaction; use idempotency to make retry safe.

## 8. Progress Analytics — `/analytics`

- Date range selector and weekly/monthly summaries.
- Quest completion rate, goal measurability/progress, overdue quests rescheduled, review completion, shared-mission participation, rating stability/explainability, and optional user-reported usefulness/stress.
- Separate observed data from suggestions; each suggestion links to evidence and requires user approval before changing plans.
- Exclude private partner data unless shared at the relevant permission level.
- Do not optimize or celebrate screen time, streaks, or volume of tasks as proxies for life improvement.
- Provide an empty state until enough history exists; no fabricated trend line.

## 9. Achievements — `/achievements`

- Collection grouped into personal, duo, milestones, and mastery.
- Locked entries may show non-sensitive criteria; never leak private partner progress through duo achievement descriptions.
- Unlock detail includes criteria version, unlock date, source event, and cosmetic reward.
- Empty collection points to eligible activities without encouraging repetitive low-value task creation.

## 10. Settings — `/settings`

### Sections

- Profile: display name, locale, time zone, portrait/accent/theme.
- Privacy & Sharing: per-category sharing and per-resource permissions, current partner access, revoke action, export caveat for already downloaded data.
- Scoring: published progression/rating versions and explanations; no hidden threshold controls.
- Duo: invitations, membership, leave/disconnect.
- Finance: reporting preferences/currency; account archive controls.
- Notifications: channels, quiet hours, sensitive content preview controls, sound off by default.
- Accessibility: reduced motion, contrast, density, keyboard help.
- Data: export, account deletion request, session controls.

### Permission mutation behavior

Show the exact resource/category and access granted, require explicit save, call server, then invalidate/reload affected views. Revocation must take effect for the next read. Show failure without leaving a stale “saved” state. Permission actions are recorded in audit history.

## 11. First-time onboarding flow

1. **Future** — optional 1/3/5-year vision prompts across career, learning, health, finances, relationships, lifestyle.
2. **Goals** — select 3–5 priorities; measurable outcome, baseline, target/date where appropriate.
3. **Character** — display name, portrait, accent, optional theme; cosmetics never affect scoring.
4. **Baseline** — configure seven attributes and optional assessment; unknown remains unassessed.
5. **First quests** — propose weekly targets and daily actions; user reviews/approves before activating.
6. **Duo** — optional secure expiring invitation; each partner completes their own onboarding and separately chooses sharing.

Persist each step and support resume. Sensitive questions are optional; no income, debt, health, or relationship disclosure is required. Do not grant XP for onboarding claims. Allow skip/postpone of linking and private goals. Target first session: 5–10 minutes.

## 12. Shared dialog, loading, and error patterns

- Confirm dialog: state action, affected record, and irreversible/privacy impact in plain language; default focus on safe cancel for destructive operations.
- Loading: skeletons preserve structure; disable duplicate submissions while retaining an idempotency key.
- Empty: explain why the area is empty and give one relevant next action.
- Error: distinguish permission denial, validation failure, network uncertainty, and server failure. Never expose stack traces or private existence.
- Conflict: for edits made elsewhere, show latest version and ask user to reconcile; do not silently overwrite.
- Offline: allow draft composition only; clearly mark unsynced and prohibit claiming completion/XP until confirmed by server.

## 13. Motion, sound, and visual hierarchy

Use short opacity/transform transitions for panel opening and normal completion. Keep regular completion fast; reserve cinematic level-up animation for a confirmed level transition or mastery unlock. All motion respects reduced-motion preferences. Sound is optional and off by default. Data readability takes priority over decorative effects; financial tables use stable row heights and minimal animation.


