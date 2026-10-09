# Duo Leveling — Database Migration Specification

Status: implementation blueprint, PostgreSQL/Supabase, v1.0  
Source: Technical Product Specification, October 2026

## 1. Decisions and invariants

- Use `auth.users.id` as the player identifier; application tables never accept a client-selected owner as authority.
- Personal rows use `owner_user_id`; shared rows use `duo_id`. These are mutually exclusive on goals and quests.
- A duo has exactly two active members at most. A duo can temporarily have one active member while an invitation is pending or after a member leaves.
- RLS is enabled on every user-facing table. Security-definer functions are narrowly scoped, set a fixed `search_path`, and are not executable by `public`.
- XP ledger entries are append-only. Corrections use a compensating negative entry referencing the original; cached totals and levels update in the same transaction.
- Financial amounts are nonnegative integer minor units plus an ISO 4217 code. Transfer direction is represented by paired account entries linked by a transfer id; transfers never enter income/expense totals.
- All timestamps are `timestamptz` in UTC. Scheduling stores an IANA time-zone name.
- History (XP, rating, finance, security events) is retained on account deletion by anonymizing identity references where required by the chosen retention policy. Hard-delete policy requires a separate product/legal decision.

## 2. Migration order

1. `0001_identity_and_duos.sql`: extensions, enums, profiles, duos, memberships, invitations, sharing policies, membership and invitation RPCs, RLS.
2. `0002_goals_and_quests.sql`: vision, goals, milestones, quest definitions/instances, confirmations, evidence, RLS and completion RPC.
3. `0003_progression_and_ratings.sql`: versioned rules, XP ledger/caches, attributes/observations, snapshots, achievements, award and calculation functions.
4. `0004_finance_and_audit.sql`: accounts, categories, transactions/links, budgets, financial goals, balances, system events, reviews, notifications, idempotency, RLS.
5. `0005_seed_v1_rules.sql`: initial seven attributes, rule versions, and achievement definitions. Seed data is immutable after release; edits require a new version.

Every migration is transactional, re-runnable only through migration tooling, and reviewed against the RLS matrix in §8 before deployment.

## 3. Shared conventions

```sql
create extension if not exists pgcrypto;
create schema if not exists private;
create type public.owner_kind as enum ('user', 'duo');
create type public.membership_status as enum ('active', 'left', 'removed');
create type public.resource_permission as enum ('read', 'summary', 'write');
create type public.goal_status as enum ('draft', 'active', 'paused', 'completed', 'cancelled');
create type public.quest_status as enum ('draft', 'scheduled', 'active', 'pending_confirmation', 'completed', 'paused', 'skipped', 'overdue', 'cancelled');
create type public.quest_difficulty as enum ('trivial', 'easy', 'standard', 'hard', 'epic');
create type public.transaction_kind as enum ('income', 'expense', 'transfer', 'refund', 'adjustment');
create type public.account_kind as enum ('asset', 'liability', 'cash', 'investment', 'other');
```

All tables below have `id uuid primary key default gen_random_uuid()` unless a composite key is specified, and `created_at timestamptz not null default now()` where applicable. Add `updated_at` and a trigger for mutable entities. FK deletion defaults to `restrict`; only explicitly ephemeral child rows may cascade.

## 4. Identity and duo tables

```sql
create table public.profiles (
  id uuid primary key references auth.users(id) on delete restrict,
  display_name text not null check (length(trim(display_name)) between 1 and 80),
  timezone text not null default 'UTC', locale text not null default 'en',
  created_at timestamptz not null default now(), updated_at timestamptz not null default now()
);
create table public.duos (
  id uuid primary key default gen_random_uuid(), name text not null check (length(trim(name)) between 1 and 100),
  created_by uuid not null references public.profiles(id) on delete restrict,
  status text not null default 'active' check (status in ('active','archived')),
  created_at timestamptz not null default now()
);
create table public.duo_members (
  duo_id uuid not null references public.duos(id) on delete restrict,
  user_id uuid not null references public.profiles(id) on delete restrict,
  status public.membership_status not null default 'active', joined_at timestamptz not null default now(),
  left_at timestamptz, primary key (duo_id,user_id),
  check ((status='active' and left_at is null) or (status<>'active'))
);
create unique index duo_one_active_duo_per_user on public.duo_members(user_id) where status='active';
create table public.duo_invitations (
  id uuid primary key default gen_random_uuid(), duo_id uuid not null references public.duos(id) on delete restrict,
  invited_by uuid not null references public.profiles(id) on delete restrict,
  invitee_email_hash bytea, token_hash bytea not null unique, expires_at timestamptz not null,
  accepted_at timestamptz, revoked_at timestamptz, created_at timestamptz not null default now(),
  check (expires_at > created_at), check (accepted_at is null or revoked_at is null)
);
create table public.sharing_policies (
  id uuid primary key default gen_random_uuid(), owner_user_id uuid not null references public.profiles(id) on delete restrict,
  duo_id uuid not null references public.duos(id) on delete restrict, resource_type text not null,
  resource_id uuid, permission public.resource_permission not null,
  created_at timestamptz not null default now(), revoked_at timestamptz,
  check (resource_type in ('character_level','life_rating','financial_summary','goal','quest','note'))
);
create unique index sharing_policy_active_unique on public.sharing_policies
  (owner_user_id,duo_id,resource_type,coalesce(resource_id,'00000000-0000-0000-0000-000000000000'::uuid)) where revoked_at is null;
```

Add a deferred constraint trigger on `duo_members` that locks the duo row and rejects a third active member. RPCs `create_duo()`, `issue_duo_invitation(email, expires_at)`, `accept_duo_invitation(token)`, `leave_duo(duo_id)`, and `revoke_sharing(policy_id)` derive the actor from `auth.uid()`, hash tokens with a cryptographically secure random value before persistence, enforce single use/expiry, and write `system_events`. Invitation email delivery is outside SQL. Do not expose invitation token hashes.

## 5. Goals and quests

```sql
create table public.life_visions (
  id uuid primary key default gen_random_uuid(), owner_type public.owner_kind not null,
  owner_user_id uuid references public.profiles(id) on delete restrict,
  duo_id uuid references public.duos(id) on delete restrict,
  horizon smallint not null check (horizon in (1,3,5)), description text not null,
  visibility text not null default 'private' check (visibility in ('private','shared')), created_at timestamptz not null default now(),
  check ((owner_type='user' and owner_user_id is not null and duo_id is null) or (owner_type='duo' and duo_id is not null and owner_user_id is null))
);
create table public.goals (
  id uuid primary key default gen_random_uuid(), owner_type public.owner_kind not null,
  owner_user_id uuid references public.profiles(id) on delete restrict, duo_id uuid references public.duos(id) on delete restrict,
  parent_goal_id uuid references public.goals(id) on delete restrict, vision_id uuid references public.life_visions(id) on delete restrict,
  goal_type text not null check (goal_type in ('outcome','habit','milestone','financial_target','skill')),
  title text not null check (length(trim(title)) between 1 and 200), description text,
  baseline_value numeric, target_value numeric, current_value numeric, unit text,
  direction text check (direction in ('increase','decrease','milestones')),
  target_date date, timezone text not null default 'UTC', status public.goal_status not null default 'draft',
  priority smallint not null default 3 check (priority between 1 and 5), progress_method text not null,
  required_dependencies boolean not null default true, created_at timestamptz not null default now(), updated_at timestamptz not null default now(),
  check ((owner_type='user' and owner_user_id is not null and duo_id is null) or (owner_type='duo' and duo_id is not null and owner_user_id is null)),
  check (direction is distinct from 'increase' or target_value > baseline_value),
  check (direction is distinct from 'decrease' or target_value < baseline_value)
);
create table public.goal_milestones (
  id uuid primary key default gen_random_uuid(), goal_id uuid not null references public.goals(id) on delete restrict,
  title text not null, target_date date, weight numeric not null check(weight>0), status public.goal_status not null default 'active',
  depends_on uuid references public.goal_milestones(id) on delete restrict, completed_at timestamptz,
  unique(goal_id,id)
);
create table public.quests (
  id uuid primary key default gen_random_uuid(), owner_type public.owner_kind not null,
  owner_user_id uuid references public.profiles(id) on delete restrict, duo_id uuid references public.duos(id) on delete restrict,
  goal_id uuid references public.goals(id) on delete restrict, assignee_id uuid references public.profiles(id) on delete restrict,
  title text not null check(length(trim(title)) between 1 and 200), description text,
  difficulty public.quest_difficulty not null, base_xp smallint not null check(base_xp in (5,15,30,60,100)),
  milestone_bonus smallint not null default 0 check(milestone_bonus between 0 and 50), reward_xp smallint not null check(reward_xp between 5 and 150),
  requires_evidence boolean not null default false, requires_both boolean not null default false, allow_single_completion boolean not null default false,
  schedule_rule jsonb, schedule_timezone text not null default 'UTC', status public.quest_status not null default 'draft',
  created_by uuid not null references public.profiles(id) on delete restrict, created_at timestamptz not null default now(), updated_at timestamptz not null default now(),
  check ((owner_type='user' and owner_user_id is not null and duo_id is null) or (owner_type='duo' and duo_id is not null and owner_user_id is null))
);
create table public.quest_instances (
  id uuid primary key default gen_random_uuid(), quest_id uuid not null references public.quests(id) on delete restrict,
  occurrence_key text not null, scheduled_for timestamptz, due_at timestamptz, status public.quest_status not null default 'scheduled',
  completed_by uuid references public.profiles(id) on delete restrict, completed_at timestamptz,
  evidence_required boolean not null default false, created_at timestamptz not null default now(),
  unique(quest_id,occurrence_key), check ((status='completed')=(completed_at is not null))
);
create table public.quest_confirmations (
  quest_instance_id uuid not null references public.quest_instances(id) on delete restrict,
  user_id uuid not null references public.profiles(id) on delete restrict,
  confirmation_status text not null check(confirmation_status in ('confirmed','rejected')),
  confirmed_at timestamptz not null default now(), primary key(quest_instance_id,user_id)
);
create table public.quest_evidence (
  id uuid primary key default gen_random_uuid(), quest_instance_id uuid not null references public.quest_instances(id) on delete restrict,
  submitted_by uuid not null references public.profiles(id) on delete restrict, evidence_type text not null,
  storage_key text not null, created_at timestamptz not null default now(), unique(storage_key)
);
create table public.goal_dependencies (
  goal_id uuid not null references public.goals(id) on delete restrict,
  depends_on_goal_id uuid not null references public.goals(id) on delete restrict,
  mandatory boolean not null default true, primary key(goal_id,depends_on_goal_id), check(goal_id<>depends_on_goal_id)
);
```

Add indexes on all FK columns, `(owner_user_id,status)`, `(duo_id,status)`, quest due/schedule timestamps, and goal target dates. A trigger verifies every referenced goal/vision and assignee belongs to the same owner scope and that duo writes are from active members. `complete_quest_instance(instance_id, idempotency_key)` locks the instance, checks actor/assignment/state/evidence/required confirmations, transitions state once, appends XP and event rows atomically, and returns the original result on a retry.

## 6. Progression, ratings, and configuration

```sql
create table public.algorithm_versions (
  id text primary key, kind text not null check(kind in ('progression','rating','duo_progression')),
  configuration jsonb not null, effective_at timestamptz not null, created_at timestamptz not null default now()
);
create table public.character_profiles (
  user_id uuid primary key references public.profiles(id) on delete restrict,
  level smallint not null default 1 check(level between 1 and 99), total_xp bigint not null default 0 check(total_xp>=0),
  display_title text, mastery_completed_at timestamptz, created_at timestamptz not null default now()
);
create table public.duo_progress (
  duo_id uuid primary key references public.duos(id) on delete restrict,
  level smallint not null default 1 check(level between 1 and 99), total_xp bigint not null default 0 check(total_xp>=0), created_at timestamptz not null default now()
);
create table public.xp_ledger (
  id uuid primary key default gen_random_uuid(), recipient_type public.owner_kind not null,
  recipient_user_id uuid references public.profiles(id) on delete restrict, recipient_duo_id uuid references public.duos(id) on delete restrict,
  source_type text not null, source_id uuid not null, amount integer not null check(amount<>0), reason text not null,
  rule_version text not null references public.algorithm_versions(id) on delete restrict,
  reverses_id uuid references public.xp_ledger(id) on delete restrict, idempotency_key text,
  created_at timestamptz not null default now(),
  check ((recipient_type='user' and recipient_user_id is not null and recipient_duo_id is null) or (recipient_type='duo' and recipient_duo_id is not null and recipient_user_id is null))
);
create unique index xp_quest_source_once on public.xp_ledger(recipient_type,source_type,source_id) where source_type='quest_instance' and amount>0;
create unique index xp_idempotency_once on public.xp_ledger(idempotency_key) where idempotency_key is not null;
create table public.attribute_definitions (
  id uuid primary key default gen_random_uuid(), slug text not null unique, name text not null, definition text not null,
  active boolean not null default true
);
create table public.attribute_indicators (
  id uuid primary key default gen_random_uuid(), attribute_id uuid not null references public.attribute_definitions(id) on delete restrict,
  slug text not null, calculation_type text not null, weight numeric not null check(weight>0), configuration jsonb not null default '{}',
  evidence_policy text not null default 'self_report', active boolean not null default true, unique(attribute_id,slug)
);
create table public.indicator_observations (
  id uuid primary key default gen_random_uuid(), user_id uuid not null references public.profiles(id) on delete restrict,
  indicator_id uuid not null references public.attribute_indicators(id) on delete restrict,
  value numeric not null, source text not null, evidence_ref text, observed_at timestamptz not null,
  created_at timestamptz not null default now()
);
create table public.rating_snapshots (
  id uuid primary key default gen_random_uuid(), user_id uuid not null references public.profiles(id) on delete restrict,
  overall_score numeric(6,3), confidence_status text not null check(confidence_status in ('unassessed','provisional','supported')),
  assessed_domain_count smallint not null check(assessed_domain_count between 0 and 7), algorithm_version text not null references public.algorithm_versions(id) on delete restrict,
  explanation jsonb not null, calculated_at timestamptz not null default now()
);
create table public.rating_domain_snapshots (
  snapshot_id uuid not null references public.rating_snapshots(id) on delete restrict,
  attribute_id uuid not null references public.attribute_definitions(id) on delete restrict,
  raw_score numeric(6,3), display_score smallint, confidence_status text not null,
  explanation jsonb not null default '{}', primary key(snapshot_id,attribute_id)
);
create table public.achievements (
  id uuid primary key default gen_random_uuid(), slug text not null, name text not null, criteria_version text not null,
  configuration jsonb not null, unique(slug,criteria_version)
);
create table public.achievement_unlocks (
  id uuid primary key default gen_random_uuid(), achievement_id uuid not null references public.achievements(id) on delete restrict,
  recipient_type public.owner_kind not null, recipient_user_id uuid references public.profiles(id) on delete restrict,
  recipient_duo_id uuid references public.duos(id) on delete restrict, unlocked_at timestamptz not null default now(),
  check ((recipient_type='user' and recipient_user_id is not null and recipient_duo_id is null) or (recipient_type='duo' and recipient_duo_id is not null and recipient_user_id is null))
);
create unique index achievement_unlock_user_once on public.achievement_unlocks(achievement_id,recipient_user_id) where recipient_user_id is not null;
create unique index achievement_unlock_duo_once on public.achievement_unlocks(achievement_id,recipient_duo_id) where recipient_duo_id is not null;
```

Ledger inserts are service-role-only through award functions. `award_quest_xp(instance_id)` derives recipient(s), stored reward and rule version from persisted rows; clients cannot pass amount or recipient. A trigger or function in the same transaction updates cached total and level using the referenced progression version. XP corrections require a privileged audited function and a compensating entry.

## 7. Finance and operational tables

```sql
create table public.financial_accounts (
  id uuid primary key default gen_random_uuid(), owner_type public.owner_kind not null,
  owner_user_id uuid references public.profiles(id) on delete restrict, duo_id uuid references public.duos(id) on delete restrict,
  name text not null, account_type public.account_kind not null, currency char(3) not null,
  visibility text not null default 'private' check(visibility in ('private','shared')), archived_at timestamptz,
  created_at timestamptz not null default now(),
  check ((owner_type='user' and owner_user_id is not null and duo_id is null) or (owner_type='duo' and duo_id is not null and owner_user_id is null))
);
create table public.financial_categories (
  id uuid primary key default gen_random_uuid(), owner_type public.owner_kind not null,
  owner_user_id uuid references public.profiles(id) on delete restrict, duo_id uuid references public.duos(id) on delete restrict,
  name text not null, category_type text not null check(category_type in ('income','expense','neutral')), archived_at timestamptz,
  check ((owner_type='user' and owner_user_id is not null and duo_id is null) or (owner_type='duo' and duo_id is not null and owner_user_id is null))
);
create table public.financial_transactions (
  id uuid primary key default gen_random_uuid(), account_id uuid not null references public.financial_accounts(id) on delete restrict,
  type public.transaction_kind not null, amount_minor bigint not null check(amount_minor>0), currency char(3) not null,
  category_id uuid references public.financial_categories(id) on delete restrict, occurred_at timestamptz not null,
  description text, transfer_group_id uuid, created_by uuid not null references public.profiles(id) on delete restrict,
  idempotency_key text unique, created_at timestamptz not null default now(), voided_at timestamptz,
  check ((type='transfer')=(transfer_group_id is not null))
);
create table public.transaction_links (
  transaction_id uuid not null references public.financial_transactions(id) on delete restrict,
  related_transaction_id uuid not null references public.financial_transactions(id) on delete restrict,
  relationship_type text not null check(relationship_type in ('refund_of','transfer_pair','adjustment_of')),
  primary key(transaction_id,related_transaction_id), check(transaction_id<>related_transaction_id)
);
create table public.budgets (
  id uuid primary key default gen_random_uuid(), owner_type public.owner_kind not null,
  owner_user_id uuid references public.profiles(id) on delete restrict, duo_id uuid references public.duos(id) on delete restrict,
  category_id uuid references public.financial_categories(id) on delete restrict, period_start date not null,
  period_end date not null, target_amount_minor bigint not null check(target_amount_minor>=0), currency char(3) not null,
  check(period_end>period_start), check ((owner_type='user' and owner_user_id is not null and duo_id is null) or (owner_type='duo' and duo_id is not null and owner_user_id is null))
);
create table public.financial_goals (
  id uuid primary key default gen_random_uuid(), owner_type public.owner_kind not null,
  owner_user_id uuid references public.profiles(id) on delete restrict, duo_id uuid references public.duos(id) on delete restrict,
  goal_type text not null check(goal_type in ('savings','debt_reduction','purchase','business_revenue')),
  target_amount_minor bigint not null check(target_amount_minor>0), currency char(3) not null, target_date date,
  current_amount_minor bigint not null default 0, created_at timestamptz not null default now(),
  check ((owner_type='user' and owner_user_id is not null and duo_id is null) or (owner_type='duo' and duo_id is not null and owner_user_id is null))
);
create table public.account_balance_snapshots (
  id uuid primary key default gen_random_uuid(), account_id uuid not null references public.financial_accounts(id) on delete restrict,
  balance_minor bigint not null, currency char(3) not null, as_of timestamptz not null, created_at timestamptz not null default now(), unique(account_id,as_of)
);
create table public.system_events (
  id uuid primary key default gen_random_uuid(), actor_id uuid references public.profiles(id) on delete set null,
  owner_type public.owner_kind, owner_user_id uuid references public.profiles(id) on delete restrict,
  duo_id uuid references public.duos(id) on delete restrict, event_type text not null, resource_type text not null,
  resource_id uuid, payload jsonb not null default '{}', created_at timestamptz not null default now()
);
create table public.idempotency_keys (
  actor_id uuid not null references public.profiles(id) on delete restrict, key text not null,
  operation text not null, request_hash bytea not null, response jsonb, created_at timestamptz not null default now(),
  expires_at timestamptz not null, primary key(actor_id,key,operation)
);
create table public.notification_preferences (
  user_id uuid primary key references public.profiles(id) on delete restrict, settings jsonb not null default '{}', updated_at timestamptz not null default now()
);
create table public.notifications (
  id uuid primary key default gen_random_uuid(), user_id uuid not null references public.profiles(id) on delete restrict,
  kind text not null, payload jsonb not null default '{}', read_at timestamptz, created_at timestamptz not null default now()
);
create table public.weekly_reviews (
  id uuid primary key default gen_random_uuid(), user_id uuid not null references public.profiles(id) on delete restrict,
  week_start date not null, snapshot jsonb not null, suggestions jsonb not null default '[]', decisions jsonb,
  finalized_at timestamptz, created_at timestamptz not null default now(), unique(user_id,week_start)
);
```

Indexes: every FK, `(account_id,occurred_at)`, `(user_id,week_start)`, unread notifications `(user_id,created_at) where read_at is null`, events `(owner_user_id,created_at)` and `(duo_id,created_at)`, and active invitation expiry. Add a deferred transfer validator requiring exactly two linked entries for a transfer group, same currency and equal amount, in distinct accounts. Refunds reference an original transaction and cannot exceed its unrefunded amount. Financial summaries group by currency; no implicit FX conversion.

## 8. RLS and RPC policy matrix

Enable RLS on every table above; revoke broad `anon`/`authenticated` table writes before granting policy-scoped access. Shared helper functions `private.is_active_duo_member(duo_id)`, `private.can_access_user_resource(owner_user_id, resource_type, resource_id, permission)`, and `private.can_access_account(account_id, write)` use `auth.uid()` and current non-revoked policy rows.

| Data | Select | Insert/update/delete |
|---|---|---|
| Profile | own; partner only fields explicitly shared through a view | own display prefs only; identity changes through auth |
| Duo, membership | active members, limited columns | membership RPC only; no direct client role/status writes |
| Invitations | inviter and intended accepted user, without token hash | issue/revoke RPC only |
| Personal goals/quests/vision | owner only | owner; duo rows require active membership and shared scope |
| Quest instance/evidence | owner/assignee; shared duo members under quest policy | authorized assignee/member via RPC; evidence storage private |
| XP/rating/achievements | owner; partner only through explicit view/policy | service functions only |
| Personal finance | account owner only unless explicit `summary`/`read` policy | account owner only; duo account active members per treasury permission |
| Audit/reviews/notifications | actor/owner; events scoped to authorized resources | trusted functions only, except own review decisions/read markers |

RLS predicates must join back to the owning resource and verify active membership at request time. Never authorize from a client-supplied `duo_id` alone. For summary sharing, expose a view/RPC that returns only approved aggregates and currency-period dimensions; do not grant access to transaction rows. Revocation affects the next database request. Storage bucket is private; object paths include owner scope and are served only through a server-validated signed URL.

## 9. Required acceptance checks before implementing migrations

- Test as anonymous, owner A, partner B, and service role for each resource class.
- A cannot read B's unshared goal, quest title, rating, observation, account, transaction, or evidence object.
- Summary permission reveals only the shared aggregate; `read` permission is explicit and revocable.
- A duo cannot get a third active member; one user cannot be active in two duos.
- Expired, revoked, wrong-recipient, and reused invitations fail without changing membership.
- Concurrent completion and finance retries create a single result/ledger award.
- Transfers reconcile and are excluded from cash flow; unsupported currency conversion is never guessed.
- Financial/progression history cannot be cascade-deleted through parent deletion.


