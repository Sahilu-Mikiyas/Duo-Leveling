# Duo Leveling — Implementation Roadmap

Status: agreed working plan  
Product reference: Technical Product Specification v1.0, October 2026

## Product direction

Build Duo Leveling as an individual progression system connected to an explicitly shared two-person workspace. Each person owns their character, goals, quests, XP, ratings, and finances. Duo features operate only on shared records and explicitly shared summaries.

## MVP scope

### Include in the first usable release

- Individual accounts and profiles.
- Life vision and measurable personal goals.
- Quest creation, scheduling, recurring instances, and completion.
- Server-authoritative XP ledger and character levels.
- Seven optional attribute assessments and explainable rating snapshots.
- Secure duo invitation and membership for exactly two people.
- Shared goals and missions, with explicit completion confirmation rules.
- Granular sharing controls and a recent system event history.
- Responsive dashboard, character profile, Quest Board, Roadmap, Duo Hub, and Settings screens.
- Auditable state changes, idempotent mutations, and tested privacy boundaries.

### Defer until after the core privacy model is proven

- Personal financial transactions, budgets, reports, and Duo Treasury.
- Bank integrations, payment credentials, public profiles, leaderboards, social features.
- AI-generated plans that apply changes automatically.
- Production-scale multi-tenant operations beyond the initial product model.

Finance remains part of the longer-term product plan. Its ownership and permission boundaries should be anticipated in the architecture, but finance UI and transaction storage are not part of the first usable release.

## Delivery phases and exit criteria

### Phase 0 — Decisions and project setup

**Deliverables**

- Confirm MVP scope and the deferred features above.
- Choose Supabase project, frontend hosting, deployment environments, and domain strategy.
- Decide invitation delivery and acceptance flow.
- Define staging versus production secrets and access.
- Convert the three technical specs into implementation-ready issues or small work packages.

**Exit criteria**

- Hosting and environment choices are documented.
- No unresolved decision blocks app scaffolding or the identity/privacy model.
- Secrets and production credentials are not committed to the repository.

### Phase 1 — Foundation and privacy

**Deliverables**

- React, TypeScript, Vite application shell with responsive navigation and design tokens.
- Supabase Auth integration and profile bootstrap.
- Initial database migrations for profiles, duos, membership, and invitations.
- Duo creation, secure expiring invitation, acceptance, leave/disconnect lifecycle.
- Row-Level Security on all client-accessible tables.
- Permission helper functions, audit events, and privacy-focused error states.
- CI checks for formatting, type safety, and build health.

**Exit criteria**

- Two separate accounts can form a duo.
- A duo cannot have more than two active members; a user cannot join multiple active duos.
- Invitation expiry, revocation, recipient validation, and one-time use are enforced.
- A partner cannot access unshared personal data under direct table/API access.
- Permission revocation affects subsequent reads.

### Phase 2 — Life vision, goals, and quests

**Deliverables**

- Resumable onboarding: vision, initial goals, character preferences, optional baseline, suggested quests.
- Goal hierarchy and measurable progress calculations.
- Quest definition and instance model, including recurrence and time zone.
- List, calendar, and kanban Quest Board views.
- Quest detail and completion flows, evidence options, history, and audit trail.
- Weekly planning and review drafts; suggested changes require user approval.

**Exit criteria**

- A user can create a vision, measurable goal, milestone, and associated quests.
- Recurring instances have unique identities and do not create unbounded missed history.
- Goal dependencies prevent completion until resolved or explicitly changed with an audit record.
- Completing a quest changes state once and never causes XP to be awarded twice.

### Phase 3 — Progression engine

**Deliverables**

- Versioned progression configuration and immutable XP ledger.
- Server-side quest reward calculation and atomic completion award.
- Character and Duo level caches derived from ledger totals.
- Seven attribute definitions, observations, confidence states, and rating snapshots.
- Rating explanation UI, historical view, and algorithm version display.
- Achievements and Level 99 mastery checklist.

**Exit criteria**

- Level thresholds match the documented curve, including edge cases and level cap.
- Duplicate, concurrent, and retried completions cannot create duplicate awards.
- XP reversals are compensating ledger entries with a reason; prior history is preserved.
- Missing assessment data remains unassessed rather than becoming zero.
- Rating changes cite their evidence and calculation version.
- Duo XP cannot flow into individual character progression or vice versa.

### Phase 4 — Shared duo experience

**Deliverables**

- Duo Hub with only shared-visible status.
- Shared goals, missions, milestones, and planning log.
- Configurable participant confirmation and one-person completion rules.
- Explicit sharing for character level, Life Rating, and approved summaries.
- Duo milestones and achievements.

**Exit criteria**

- Both members can collaborate on shared items within authorized fields.
- Personal quest titles, goals, ratings, notes, and finance details stay private by default.
- Sharing one category does not grant access to other categories.
- Duo unlocks never change privacy settings or grant control over a partner.

### Phase 5 — Finance

**Deliverables**

- Personal accounts, categories, transactions, budgets, financial goals, and reports.
- Currency-aware arithmetic and linked transfer/refund behavior.
- Explicitly shared aggregate summaries.
- Duo Treasury with separate account ownership and explicit authorization.
- Finance export and retention/deletion flows.

**Exit criteria**

- Personal finance rows cannot be read through duo endpoints or summaries.
- Summary permission reveals aggregates only, not underlying transactions.
- Transfers do not count as income or expenses; refunds link to original expenses.
- Arithmetic uses integer minor units and never assumes a single currency or implicit FX rate.
- Concurrent retries cannot duplicate a transaction.

### Phase 6 — Pilot readiness

**Deliverables**

- Accessibility and reduced-motion review.
- Responsive polish and loading, empty, offline, error, and conflict states.
- Data export, account deletion, backups, and restoration procedure.
- Privacy review of notifications, analytics, logs, and storage access.
- Pilot feedback and prioritized follow-up work.

**Exit criteria**

- Both partners can use the app for several weeks without unclear progression or privacy behavior.
- Every rating, level change, and permission change has an understandable explanation.
- Backup restoration, account deletion, invitation expiry, time-zone boundaries, and interrupted mutations have documented handling.

## Implementation approach

- Build a modular monolith; do not introduce microservices or a message broker for the two-person MVP.
- Keep authoritative progression, rating, goal calculations, and permission checks in domain/server modules rather than trusting the browser.
- Treat migrations as the source of database evolution; include ownership, indexes, constraints, RLS, and deletion behavior in each change.
- Use idempotency keys, database constraints, transactions, and row locks for sensitive mutations.
- Work in vertical slices: implement one end-to-end user capability, check its acceptance criteria, review failure and privacy behavior, then continue.
- Keep private Storage objects inaccessible by public URL; issue short-lived authorized links only after a permission check.
- Avoid sending private finance amounts, task titles, or notes to analytics or notifications by default.

## First end-to-end user journey

1. Create an account and profile.
2. Define a life vision and one measurable personal goal.
3. Approve a few suggested quests for that goal.
4. Complete one quest and receive a server-confirmed XP award.
5. Invite a partner and create a shared mission.
6. View individual and duo progress while private records remain inaccessible to the partner.

## First implementation slice

Start with **Phase 1: Foundation and privacy**, but keep it narrow:

1. Inspect repository state and add project conventions.
2. Scaffold the frontend shell and configuration.
3. Set up database migrations and Auth/profile bootstrap.
4. Implement duo invitation and membership lifecycle.
5. Add RLS policies and verify the two-account access matrix.
6. Deliver a responsive shell with sign-in, onboarding entry, and a minimal authenticated home.

Do not start quests, ratings, or finance until the foundation exit criteria are met.

## Open decisions to resolve during Phase 0

- Supabase organization/project and deployment region.
- Frontend hosting provider and domain.
- Whether invitations are email-only or accepted by secure link.
- Whether duo connection happens during onboarding or can be postponed (default: postponable).
- Whether profile display names are visible to the invited partner before acceptance (default: no).

