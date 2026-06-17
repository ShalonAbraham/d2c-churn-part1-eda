# Business Memo: What to Investigate Before Launching a Retention Campaign

**To:** Product, Marketing & Customer Support leads
**From:** Data Analytics (Capstone Part 1 — D2C Churn Intelligence)
**Re:** Findings from the customer/order/support data audit, and what to check before spending
retention budget

---

## Bottom line

Before any retention campaign goes live, three things in our own data need a closer look:
**how we currently judge "risk" without a model, where that judgment already fails, and which
behavioural signals are actually trustworthy versus noisy.** We now have evidence on all three.

---

## 1. Our existing manual risk process has real signal — but two specific blind spots

The CRM team already assigns every customer a `manual_priority_bucket` (`high`/`medium`/`low`)
before any data science touches the problem. Checking that judgment against what customers
actually did in the following 60 days:

| CRM-assigned priority | Actual churn rate | Customers |
|---|---|---|
| High | 74.7% | 1,163 |
| Medium | 27.9% | 749 |
| Low | 10.0% | 488 |

This is a strong, monotonic relationship — the team's gut feel is not random, and should not be
discarded once a model exists. But it has two concrete failure modes worth fixing immediately:

- **49 customers labeled "low priority" churned anyway** (e.g. `CUST00069`, `CUST00085`,
  `CUST00157`) — these are silent misses that no campaign would have targeted.
- **294 customers labeled "high priority" did not churn** — retention spend aimed at this group
  (welcome offers, bundle discounts) may be partially wasted on customers who were never actually
  at risk.

**Recommendation:** Before scaling any campaign, cross-reference the manual priority list against
recency and session-activity data (see #2) to catch the missed 49 and deprioritize a portion of
the over-flagged 294, freeing budget for genuinely at-risk customers.

## 2. Recency and web/app disengagement are the two most trustworthy risk signals we have

Churn rate rises from 8.8% for customers who ordered in the last week to **72.3%** for customers
inactive 60+ days — the single strongest pattern in the entire dataset, and one already
observable today for every customer without needing a model. Web/app session count tells a
similar, independent story (66.3% churn at zero sessions in 30 days, down to 20.9% at 11+
sessions) — meaning disengagement shows up in browsing behaviour *before* it shows up in an order
gap, giving the team a few extra weeks of lead time if session data is monitored.

**Recommendation:** Any retention trigger system should be built primarily on recency + session
activity, not order history alone, since session drop-off is the earlier warning.

## 3. Support-ticket volume is a misleading signal on its own — don't flag "complainers" as high-risk by default

Counter to intuition, customers who raise 2+ support tickets in 90 days churn *less* (15.2%) than
customers with zero tickets (50.6%). Investigating why: high-ticket customers are disproportionately
recent, frequent buyers (average recency 26 days, average frequency 3.3 orders in 180 days) — they
complain because they're still engaged, not because they're leaving. The real risk combination is
**inactivity plus a negative-sentiment ticket** (65.4% churn) versus an active customer with a
negative ticket (28.7% churn).

**Recommendation:** Do not route "ticket count" alone into any priority or discount-eligibility
rule. If support signals are used, they should be combined with recency/activity, not used
standalone — otherwise the company risks discounting its most loyal, most vocal customers while
under-prioritizing quiet, disengaging ones.

## 4. A handful of return-heavy customers are a distinct, smaller risk pool

Customers returning over half their orders churn at 75.0% (84 customers, e.g. `CUST00078`,
`CUST00108`, `CUST00213`) — markedly higher than the 47.2% baseline. This is a small but
identifiable group worth a different kind of intervention (product-fit or sizing support, not a
generic discount) before any broad campaign launches.

## 5. Discounting is already widespread — campaign budget should not just mean "more discount"

The median order already carries a 27% discount, and over a quarter of orders exceed 36% off.
Before allocating new retention budget toward discount-based offers, it's worth asking whether
the company is already training a large share of customers to expect a discount on every order —
a dynamic that Part 2's segmentation work should examine directly (discount-sensitive segments
likely already exist within the base).

---

## What to investigate next (in priority order)

1. Reconcile the 49 missed "low priority" customers and the 294 over-flagged "high priority"
   customers against recency/session data before the next campaign cycle.
2. Stand up recency + 30-day session activity as the primary always-on risk signal, ahead of any
   model deployment.
3. Re-audit any existing rule that treats "has a support ticket" as a risk flag — it currently
   does the opposite of what's intended.
4. Pull the return-rate-over-50% cohort into a separate, product-quality-focused outreach rather
   than a standard discount campaign.
5. Quantify current discount exposure before committing fresh budget to discount-led retention
   offers — Part 2 will propose a segment-based budget allocation that should be sized against
   this baseline.
