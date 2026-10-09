# Duo Leveling — Progression Engine Specification

Status: deterministic domain contract, version `progression-v1` and `rating-v1`.

## 1. Authority and invariants

The server is authoritative for state transitions, reward values, levels, ratings, and snapshots. The client may request an action and display the returned explanation; it never submits a trusted XP amount or score. All rules are versioned and persisted with their resulting ledger/snapshot rows. A retry with the same idempotency key returns the original result.

Separate ledgers and caches exist for a player and a duo. Personal XP never enters Duo XP and vice versa. No negative XP is applied for missed quests. Quest reward configuration freezes when a quest/instance is approved; edits after completion are audited and cannot rewrite prior awards.

## 2. Character and Duo levels

For level `L` in `[1,98]`, XP from level `L` to `L+1` is:

```text
T(L) = 100 + 25 * (L - 1)
C(1) = 0
C(L) = (L - 1) * (200 + 25 * (L - 2)) / 2
```

`C(L)` is cumulative XP to reach level `L`. Equivalent implementation: sum `T(k)` for `k=1..L-1`. Level is the greatest `L` where `C(L) <= total_xp`, capped at 99. At level 99, displayed progress is complete; XP can continue accruing in the ledger, but level stays 99. Mastery is a distinct state: level 99 plus the final checklist. The same curve applies to Duo Level.

Expected checkpoints: `C(1)=0`, `C(2)=100`, `C(5)=550`, `C(10)=2,025`, `C(20)=7,025`, `C(50)=35,525`, `C(75)=84,175`, `C(98)=126,175`, `C(99)=128,700`.

```text
level_for(total_xp):
  reject total_xp < 0
  for L from 99 down to 1:
    if total_xp >= C(L): return L
```

The cache update and ledger insert occur under one transaction/row lock. Cache reconciliation can recompute total from the signed ledger, then recalculate the level. The XP ledger remains the source of truth.

## 3. Quest XP awards

| Difficulty | Base XP |
|---|---:|
| Trivial | 5 |
| Easy | 15 |
| Standard | 30 |
| Hard | 60 |
| Epic | 100 |

At creation/approval, validate difficulty and approved milestone bonus `M ∈ [0,50]`, then persist `reward = min(150, base[difficulty] + M)`. This is the immutable amount for that quest version. UI effort estimates are guidance, not a hidden formula. Repeated identical quest patterns may be flagged or capped under a published configuration; never silently alter already earned rewards.

Completion transaction:

1. Authenticate caller and lock the instance and quest rows.
2. Confirm instance is eligible and not already completed/cancelled/skipped; verify caller is owner/assignee or an authorized active duo member.
3. Validate required evidence and required confirmation from each configured participant. If another confirmation is needed, set `pending_confirmation`, record the actor and return without awarding XP.
4. Set completion state once. Use a unique positive ledger source per `quest_instance_id` as a database backstop.
5. Insert the XP ledger row using stored reward and active rule version, update cache/level atomically, unlock eligible achievements, and write an audit event.
6. Persist the idempotent response. A duplicate request returns that result and creates no new award.

For duo quests, the award recipient is the duo only. Confirming completion twice does not create a second award. A one-person completion is allowed only if the approved mission configuration explicitly allows it. Each scheduled recurrence has a unique instance/occurrence key and can earn one reward only.

Reversals create a linked signed compensating ledger entry, with actor, reason, source, and version; the original entry is never deleted. A reversal may not drive cached XP below zero.

## 4. Life Rating

Seven domains: Mind, Discipline, Health, Career, Finance, Relationships, Creativity. Indicators have a calculation type, source, weight `w > 0`, configuration, and evidence policy. Missing or invalid evidence is excluded; an unassessed domain is not zero.

For proportion indicators, clamp `p` to `[0,1]` and calculate `I=100*p`. Other indicator types use their versioned rubric and must produce a score in `[0,100]`. For a domain with valid indicators:

```text
D = sum(w * I) / sum(w)
display_rating = clamp(round_half_up(1 + 0.98 * D), 1, 99)
```

Store `D` at decimal precision; round only for display. Domain score is unassessed if it has no valid indicators. Overall score is the arithmetic mean of available domain raw scores, converted with the same linear map for display. Confidence is `unassessed` when no domains are assessed, `supported` when all seven are assessed, and `provisional` otherwise. Always show assessed domain count and explain contributing evidence and algorithm version.

Life Rating is independent from XP and quest completion. A quest completion changes a rating only if it produces valid indicator evidence under the stated indicator rule. Never automatically reduce rating for missed tasks. Health and relationship domains are self-development tools, not diagnoses or objective judgments of worth.

Rating recomputation creates a new immutable snapshot when input evidence changes or at the daily snapshot boundary. Significant milestones can force an additional snapshot. Historical snapshots preserve the exact algorithm version and input/evidence references. A retroactive algorithm change requires a new version and an explicit recomputation job; never overwrite history.

## 5. Goal and finance calculations

Increasing target: `clamp(100 * (V-B)/(T-B), 0, 100)` where `T>B`. Decreasing target (e.g. debt): `clamp(100 * (B-V)/(B-T), 0, 100)` where `T<B`. Milestone progress is completed milestone weight divided by total weight. Prevent completion while mandatory dependencies remain incomplete unless an authorized user changes the definition, records the reason, and accepts the resulting audit entry.

Financial arithmetic uses integer minor units and currency-specific exponents. Group reports by currency unless a user explicitly selects a supported FX rate/source/date. Cash flow is income less expenses; transfers are excluded. Savings rate is `100*(income-expenses)/income`, or `N/A` if income is zero. Refunds are linked to source expenses and reduce the relevant expense total; adjustments are explicitly classified and explained. No binary floating-point arithmetic.

## 6. Level 99 mastery and duo unlocks

Player mastery requires level 99 and an approved transparent checklist documenting sustained development in at least four of seven attributes, with the other domains reviewed rather than failed. Wealth, body measurements, relationship status, and a particular career outcome are never universal prerequisites.

Duo level unlocks are cosmetic/organizational only: 1 First Connection; 5 Coordinated; 10 Building Together; 20 United Vision; 35 Established; 50 Long-Term Partnership; 75 Legacy Builders; 99 Legendary Duo. An unlock never grants access to private data or authority over a partner.

## 7. Configuration and explanations

Every award and snapshot stores a stable `rule_version`. Configuration includes thresholds, difficulty base values, max quest reward, rating conversion/rounding, indicator definitions, and Duo milestones. Published versions are immutable. UI/API responses include: old/new state, source activity, calculation inputs, rule version, and event timestamp. This explanation is used in dashboard notifications and rating detail screens.

## 8. Unit-test contract

Implement tests in the domain package and integration tests for trusted database/RPC behavior. Required cases:

- Every threshold checkpoint above; one XP below/at/above each sample boundary; XP at level 99; negative input rejection.
- Base reward for every difficulty; bonus lower/upper bounds; cap at 150; frozen reward after edit.
- Concurrent completion requests, retries with same idempotency key, different duplicate keys, invalid state, wrong actor, missing evidence, rejected confirmation, and required second confirmation.
- One award per occurrence; two distinct recurring instances award separately; one duo award despite both partners confirming; personal/duo recipient separation.
- Reversal is append-only, idempotent, auditable, and cannot reduce total below zero.
- Indicator clamp; weighted mean; `round_half_up` tie cases; missing indicator exclusion; domain and overall unassessed/provisional/supported statuses; score bounds 1–99; historical algorithm version retained.
- Increasing and decreasing goal progress at baseline, midpoint, target and beyond; zero denominator rejection; dependency block and authorized override audit.
- Transfer exclusion, equal paired entries, refund limits, currency exponent handling, zero-income savings rate, and absence of implicit FX conversion.


