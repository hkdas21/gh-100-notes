# Domain 5 — Monitor and Optimize GitHub Usage (10–15%)

> **GH-100 GitHub Enterprise Administrator** · Study Notes
> Domain weight: **10–15%** of the exam
> Focus: knowing *where the usage data lives*, *how to read it*, and *how to cut cost without breaking developer workflows*.

---

## Why this domain matters

As an enterprise administrator you are accountable for two things that pull in opposite directions: **visibility** (knowing who is doing what, whether the platform is healthy, and whether your security investments are actually being used) and **economy** (making sure the organization is not silently burning money on idle license seats, expensive runner minutes, or runaway storage). GitHub Enterprise is a **metered, partly per-seat platform** — almost everything generates either an audit event, a usage record, or a billing line. This domain tests whether you can turn that firehose of telemetry into decisions: *open a support ticket or fix it yourself? reclaim a seat or leave it? switch a job to Linux runners or pay the macOS premium?* The exam rewards administrators who treat GitHub like a utility bill they actively manage rather than a flat subscription they ignore.

---

## Big-picture analogy

🔌 **Think of yourself as the facilities manager of a large office building with a smart electricity meter.**

- The **audit log** is the building's **security camera + access-card log** — it tells you *who entered which room and when*, but it does not by itself lower your power bill.
- The **usage/billing reports** are the **electricity meter** — they tell you *how many kilowatt-hours* each tenant burned, and some appliances (the macOS "data center HVAC") cost **10× per minute** of what the Linux "LED lights" cost.
- **License seats** are like **assigned parking spaces** — you pay for each one whether or not the car ever shows up. Dormant users are empty reserved spaces you're still renting.
- **Spending limits** are the **circuit breaker**; **budget alerts** are the **"you're nearing your limit" text message**. One *stops the power*, the other *just warns you*.
- **GitHub Support** is the **utility company / electrician** — you call them only after you've checked your own fuse box (diagnostics, support bundle) and confirmed the problem is upstream.

Keep this meter-and-fuse-box image in mind; every objective below maps to one part of it.

---

## Objective 1 — Monitor enterprise usage and activity

### 1A. Analyze audit logs and API usage

**Plain English.** The **enterprise audit log** is the authoritative, time-stamped record of administrative and security-relevant events across all your organizations: who changed a repo's visibility, who added an outside collaborator, who modified a branch protection rule, who created a personal access token, who changed billing settings. You search it with a **key:value query syntax**, you can filter by actor / action / time, and — critically for compliance — you can **stream** it to external storage because the in-product UI retains only a limited window.

The **REST/GraphQL API** is metered by **rate limits**. Every API response carries `X-RateLimit-*` headers telling you your ceiling, what's left, and when the window resets. Monitoring these lets you detect a misbehaving integration *before* it gets throttled (HTTP `403`/`429`).

**Key audit-log event categories** (the `action` namespace prefix tells you the category):

| Category prefix | What it covers |
|---|---|
| `org.*` | Org membership changes, ownership, default settings |
| `repo.*` | Repo create/delete, visibility change, transfer |
| `team.*` | Team creation, membership, repo access grants |
| `protected_branch.*` | Branch protection rule changes |
| `business.*` / `enterprise.*` | Enterprise-level policy and billing changes |
| `oauth_authorization.*`, `personal_access_token.*` | Token/credential lifecycle |
| `git.*` (GHES/streamed) | Git pushes/clones/fetches (Git events, often only via streaming) |
| `actions.*`, `workflows.*` | Workflow runs and Actions policy changes |

**Audit log search syntax** uses qualifiers joined implicitly with AND:

```text
# All actions by a specific user in their org/enterprise audit log
actor:octocat

# Repo visibility changes that made something public, last 7 days
action:repo.access created:>2026-06-19

# Who deleted repositories
action:repo.destroy

# Outside-collaborator additions to a given repo
action:repo.add_member repo:my-org/payments-service

# Token creation events within a date range
action:personal_access_token.create created:2026-06-01..2026-06-26
```

> **Audit log streaming.** Because the UI keeps audit data for only a **bounded retention period** (and Git events especially are not retained long-term in the UI), you configure **audit log streaming** to an external sink — **Amazon S3, Azure Blob Storage, Azure Event Hubs, Google Cloud Storage, Splunk**, etc. Streaming gives you **indefinite, long-term retention** and lets your SIEM run historical analysis. *This is the canonical answer when an exam question says "we need audit data older than the UI shows / for compliance / for long-term analysis."*

**Reading rate-limit headers.** A representative response:

```text
$ curl -sI -H "Authorization: Bearer $TOKEN" https://api.github.com/user

HTTP/2 200
x-ratelimit-limit: 5000          # total requests allowed in the window
x-ratelimit-remaining: 4987      # requests left before throttling
x-ratelimit-used: 13             # requests consumed so far
x-ratelimit-reset: 1750960800    # Unix epoch when the window resets
x-ratelimit-resource: core       # which bucket (core, search, graphql, ...)
```

When `remaining` hits `0`, further calls return **HTTP 403** (primary rate limit) or **429** (secondary/abuse limit). Well-behaved clients read `x-ratelimit-reset` and back off until that time. You can also query the live status with `GET /rate_limit` (which does **not** count against your limit).

> **Example.** Security asks "did anyone make our `payments-service` repo public last week?" You run `action:repo.access repo:my-org/payments-service created:>2026-06-19` in the enterprise audit log, find the actor and timestamp in seconds, and (separately) confirm a Splunk dashboard fed by **audit log streaming** already captured it for the compliance archive.

> 🧭 **Analogy.** The audit log is your building's **keycard reader history** — every door swipe is logged with *who, which door, what time*. The in-product UI is the **last 90 days taped to the security desk**; **streaming** is shipping every swipe to **off-site cold storage** so auditors can review last *year's* swipes. Rate-limit headers are the **revolving-door counter**: it tells you how many more people can spin through before the door locks (`reset`) and reopens.

---

### 1B. Distinguish admin responsibilities vs. GitHub Support, and generate diagnostics

**Plain English.** Not every problem is GitHub's problem. As the admin you **triage first**: check the **GitHub status page**, your **own audit logs and configuration**, recent policy/permission changes, runner health, and (on **GitHub Enterprise Server**) the **Management Console / monitor dashboard**. You escalate to **GitHub Support** only for things you cannot see or fix: suspected platform bugs, data corruption, performance issues rooted in the appliance/cloud service, or anything needing GitHub-side investigation. When you escalate GHES issues, you attach a **support bundle**.

**What the admin investigates first (self-service):**
- Is it GitHub.com itself? → check **githubstatus.com**.
- Did a recent **policy / permission / branch-protection** change cause it? → audit log.
- Is a workflow failing due to **quota / spending limit / runner availability**? → billing + Actions settings.
- On **GHES**: CPU/memory/disk/replication health → **Management Console → Monitor dashboard**, `ghe-*` CLI utilities.

**When to open a Support ticket:**
- Reproducible product **bugs** or unexpected platform behavior.
- **Outages/performance** that the status page or your diagnostics can't explain.
- Account/billing actions only GitHub can perform.
- GHES appliance failures (upgrade failures, storage, replication, restore).

**Generating diagnostics (GHES):**
- **Support bundle** — a downloadable archive of **logs, configuration, and diagnostics** for the appliance. Generate from the **Management Console** or via SSH with `ghe-support-bundle`. For larger/extended captures use an **extended bundle** (`ghe-support-bundle -t`), or upload directly to GitHub with `ghe-support-upload`.
- The **`ghe-diagnostics`** / cluster status utilities and the **Monitor dashboard** provide real-time health (CPU, RAM, disk, queues).

```bash
# On a GHES appliance (admin shell)
ghe-support-bundle -o /tmp/support-bundle.tgz      # standard bundle
ghe-support-bundle -t                               # extended (more history)
ghe-support-upload                                  # send straight to GitHub Support
```

> **Example.** Users report slow Git pushes on GHES. You *first* open the **Monitor dashboard**, see disk I/O saturation and a growing repository-maintenance queue — admin-fixable. If instead you saw nothing abnormal and the issue persisted, you'd generate a **support bundle** with `ghe-support-bundle` and open a ticket so GitHub can dig into the internals.

> 🧭 **Analogy.** You're the **building superintendent**. A flickering light? You check your own **fuse box** (audit log, monitor dashboard) before calling the **power company** (GitHub Support). When you *do* call, you don't say "it's broken" — you hand over the **diagnostic readout** (support bundle) so the utility's engineers don't start from zero.

---

### 1C. Evaluate enterprise usage patterns (adoption, activity, underutilized features)

**Plain English.** Beyond raw events, leadership wants to know whether the platform is **actually being adopted** and whether the features you pay for are being used. GitHub exposes this through **organization & enterprise insights / Insights dashboards**, **Copilot usage metrics**, **Actions usage**, **security feature adoption (Advanced Security / GHAS)**, and **dormant-user** reporting. The administrator's job is to spot **underutilization** (paid-for seats or features nobody touches) and **over- or under-activity** that signals a problem or an opportunity.

**What to look at:**
- **Adoption** — active vs. provisioned users, new repos/PRs over time, org member growth.
- **GitHub Copilot usage** — seats assigned vs. **active seats**, acceptance rates; the **Copilot metrics/usage** views and reports reveal seats that were assigned but never used (reclaim candidates).
- **Actions usage** — minutes consumed by org/repo, by runner type — to find heavy spenders and inefficient workflows.
- **Security feature adoption** — how many repos have **code scanning, secret scanning, Dependabot** enabled; GHAS active-committer counts.
- **Dormant users** — accounts with **no activity for a defined period** (GHES surfaces a **dormant users** list; default dormancy threshold is commonly **90 days**). These are prime license-reclamation targets.

> **Example.** The enterprise Insights view shows **500 Copilot seats assigned but only 310 active** in the last 28 days, and the dormant-users report lists **40 accounts idle 90+ days**. You recommend reclaiming the 190 unused Copilot seats and suspending/deprovisioning the 40 dormant users — real recurring savings with zero impact on active developers.

> 🧭 **Analogy.** This is the **gym-membership audit**. You're paying for 500 memberships (seats), but the turnstile data (insights) shows only 310 members actually swiped in this month. The 190 no-shows are money on the floor — cancel them. The treadmills nobody uses (disabled security features) are equipment you bought but never plugged in.

---

## Objective 2 — Optimize cost and performance

### 2A. Interpret usage reports for metered products

**Plain English.** GitHub bills some things **per seat** (GitHub Enterprise/Copilot user licenses — a flat per-user cost) and other things **per unit of consumption** ("metered" products: **Actions minutes, storage GB for Packages & Actions artifacts, Codespaces compute & storage, Git LFS, Copilot premium requests** on some plans). The **enhanced billing platform** consolidates this and lets you **export a usage report as CSV**, set **budgets**, and configure **spending limits/alerts**. The single most testable nuance: **Actions minutes are multiplied by an OS factor** — the same job costs far more on macOS than on Linux.

**The OS minute multipliers (memorize these — standard rates for GitHub-hosted standard runners):**

| Runner OS | Per-minute multiplier | Reading |
|---|---|---|
| **Linux (Ubuntu)** | **1×** | Baseline / cheapest |
| **Windows** | **2×** | Twice the per-minute cost of Linux |
| **macOS** | **10×** | Ten times the per-minute cost of Linux |

So a 10-minute job = **10 billed minutes on Linux**, **20 on Windows**, **100 on macOS**. **Larger runners** (more vCPUs/RAM) bill at higher per-minute rates *and may not receive the bundled free minutes*. Self-hosted runners don't consume GitHub-hosted Actions minutes (you pay for your own infrastructure instead).

> ⚠️ **Free-minute caveat:** the monthly **included/free Actions minutes** and storage that come with a plan apply to **standard GitHub-hosted runners**; the OS multiplier still applies when drawing down those minutes, and **larger runners are billed from the first minute** (no free allotment).

**Metered products and what drives their cost:**

| Metered product | Billing unit | What drives the cost | Cheaper-direction lever |
|---|---|---|---|
| **GitHub Actions** | Minutes (× OS multiplier) | Minutes used × **OS factor** (Linux 1× / Win 2× / macOS 10×); larger runners cost more/min | Prefer **Linux**, cache deps, fail fast, concurrency limits |
| **Actions/Packages storage** | GB-month | Artifacts, build outputs, container/package versions, **artifact retention** | Lower **retention days**, prune old packages/artifacts |
| **GitHub Packages** | GB storage + data transfer | Stored package versions and egress | Delete stale versions, dedupe images |
| **Codespaces** | Compute (core-hours) + storage (GB-month) | Machine size × active hours + persisted volume | Smaller machines, **idle timeout**, prebuild wisely, delete unused |
| **Git LFS** | Storage + bandwidth | Large binary files + downloads | Archive/limit large assets |
| **GitHub Copilot** | **Per seat** (mostly) | Number of **assigned** seats (per user/month) | **Reclaim unused seats** |
| **GitHub Advanced Security / GHAS** | **Per unique active committer** | Distinct committers to repos with GHAS on | Enable only where needed; watch committer count |
| **Codespaces / Copilot premium requests** | Metered (where applicable) | Premium model requests over included quota | Set/monitor budget |

**The usage report (enhanced billing platform).** From enterprise/org billing settings you can **download a CSV usage report** and filter by product, repository, SKU, cost center, and date. Columns let you see **which repos/orgs/SKUs consume the most**, broken down by **runner type and OS** for Actions. This CSV is the foundation of any chargeback or optimization analysis.

```text
# Illustrative columns from a usage report CSV (enhanced billing platform)
Date, Product, SKU, Quantity, Unit Type, Repository, Organization, Cost Center, ...
2026-06-10, Actions, "Compute - macOS",  100, minutes, my-org/ios-app,   my-org, Mobile
2026-06-10, Actions, "Compute - Linux",  420, minutes, my-org/api,       my-org, Platform
2026-06-10, Shared Storage, "Storage GB", 35, GB,       my-org/artifacts, my-org, Platform
```

**Budgets vs. spending limits vs. alerts:**
- **Spending limit** = a **hard (or configurable) ceiling** on metered overage. Once reached, the metered service **stops** (e.g., Actions jobs won't run) — it's the **circuit breaker**.
- **Budget** = a target you define (per product / cost center) on the enhanced billing platform, with **alert thresholds** (e.g., notify at 75%/90%/100%). A budget **alert just notifies**; it does not necessarily stop usage unless paired with a stop/limit behavior.

> **Example.** Finance asks why the Actions bill doubled. You download the **usage report CSV**, pivot by SKU, and see the `ios-app` repo ran its full test matrix on **macOS** runners for every PR — 100 billed minutes per 10-minute job. You move the non-Apple-specific jobs to **Linux**, keeping only the required signing/build steps on macOS, and set a **budget alert at 90%** for the Mobile cost center so finance is warned before the next spike.

> 🧭 **Analogy.** The usage report is your **itemized phone bill**. Per-seat licenses are the **flat monthly line rental** (you pay even if you never call). Metered Actions minutes are **per-minute call charges**, and calling on macOS is like **roaming internationally — 10× the rate**. A **spending limit** is the **prepaid cap that cuts the line**; a **budget alert** is the **"you've used 90% of your data" text** that lets the call continue.

---

### 2B. Recommend strategies for license and resource optimization

**Plain English.** Optimization is two buckets: **license** (per-seat) and **resource** (metered). For licenses, the win is **reclaiming seats nobody uses** (dormant users, unused Copilot/GHAS seats). For resources, the wins are **using cheaper runners, doing less work, and capping the blast radius** of runaway consumption. None of these should degrade the experience of *active* developers — that's the balance the exam wants you to strike.

**License optimization:**
- **Reclaim dormant seats** — identify dormant users (e.g., 90+ days inactive), then **suspend/deprovision** them; for SAML/SCIM enterprises, deprovision via your IdP so the seat is freed.
- **Right-size Copilot/GHAS** — pull unused **Copilot seats**; for **GHAS**, remember billing is by **unique active committer**, so disabling GHAS on dead repos or limiting scope reduces committer count.
- **Audit before renewal** — use Insights/usage reports to true-up seat counts before the billing cycle.

**Resource (metered) optimization:**

| Lever | Why it saves |
|---|---|
| **Prefer Linux runners** | Linux = 1× vs Windows 2× / macOS 10×. Move everything not strictly OS-specific to Linux. |
| **Right-size / use larger runners judiciously** | Bigger runners cost more per minute and skip free minutes — use only for genuinely heavy jobs. |
| **Cache dependencies** (`actions/cache`, package caches) | Cuts minutes spent re-downloading/building. |
| **Concurrency limits** (`concurrency:` groups) | Auto-cancels superseded runs (e.g., new push to same PR) so you don't pay for stale jobs. |
| **Scheduled-workflow hygiene** | Disable/space out `schedule:` crons that run on idle/forked repos; they silently accrue minutes. |
| **Fail fast / path & branch filters** | Don't run the whole matrix on doc-only changes; `paths:` filters and `fail-fast` stop waste early. |
| **Lower artifact/package retention** | Shrinks storage GB-month charges. |
| **Codespaces idle timeout & smaller machines** | Stops paying for idle compute; default to the smallest machine that works. |
| **Set spending limits** | Hard cap prevents a misconfigured workflow from bankrupting the budget. |
| **Self-hosted runners for steady heavy load** | Trade GitHub-hosted per-minute cost for owned infra where it's cheaper at scale. |

> **Example.** A team's CI runs a 30-job matrix on **Windows** for every commit, no caching, no concurrency control. You: (1) move 24 OS-agnostic jobs to **Linux** (2× → 1×), (2) add `actions/cache` for npm, (3) add a `concurrency` group keyed on the PR ref so new pushes cancel in-flight runs, and (4) add `paths-ignore` for docs. Result: large drop in billed minutes with identical coverage — and a **spending limit** as a safety net.

> 🧭 **Analogy.** This is **fleet fuel management**. Reclaiming dormant seats = **selling the company cars sitting unused in the lot**. Preferring Linux = **driving the fuel-efficient sedan instead of the gas-guzzling truck** for routine trips. Caching = **not re-buying the same parts every trip**. Concurrency limits = **not sending two trucks to the same destination**. The spending limit is the **fuel card's hard monthly cap** so one driver can't drain the account.

---

## 💰 Cost-optimization checklist

- [ ] **Default to Linux runners**; reserve **Windows (2×)** and **macOS (10×)** only for OS-specific jobs.
- [ ] **Cache dependencies** (`actions/cache`, language package caches) to shrink build minutes.
- [ ] Add a **`concurrency:` group** per workflow/PR so superseded runs auto-cancel.
- [ ] Apply **`paths:` / `paths-ignore:` and branch filters** so trivial changes don't trigger full matrices.
- [ ] Use **`fail-fast`** and split long jobs so failures stop spending early.
- [ ] **Audit `schedule:` (cron) workflows** — disable ones running on idle/fork repos.
- [ ] **Lower artifact & package retention**; prune old artifacts, packages, container images.
- [ ] **Right-size runners** — use **larger runners** only for genuinely heavy jobs (they cost more/min and skip free minutes).
- [ ] Set **Codespaces idle timeouts** and default to the **smallest viable machine**.
- [ ] **Set spending limits** (circuit breaker) and **budget alerts** (early warning) per product/cost center.
- [ ] **Reclaim dormant user seats** (e.g., 90+ days inactive) and **unused Copilot seats**.
- [ ] **Scope GHAS** to repos that need it (billed per **unique active committer**).
- [ ] **Download the usage report CSV** regularly; pivot by **SKU / OS / repo / cost center** to find hotspots.
- [ ] Consider **self-hosted runners** for steady, heavy, predictable workloads.

---

## ⚠️ Exam traps

> **Trap 1 — Audit log *retention* vs *streaming*.**
> The in-product audit log UI has **limited retention** (and Git events especially are not kept long-term). For **compliance / long-term / historical analysis**, the correct answer is **audit log streaming** to external storage (S3, Azure, Splunk, GCS, Event Hubs) — *not* "increase UI retention" (you can't arbitrarily extend it).

> **Trap 2 — Per-seat vs metered billing.**
> **Copilot and GitHub Enterprise user licenses are per-seat** (flat, paid even if idle). **Actions, storage, Codespaces, Packages, LFS are metered** (pay per unit). A "reclaim a seat" question is a **license** problem; a "reduce minutes/GB" question is a **metered** problem. Don't apply a spending limit to fix unused *seats*.

> **Trap 3 — OS minute multipliers.**
> **Linux 1× · Windows 2× · macOS 10×.** A 10-minute macOS job bills **100 minutes**. If a question asks the cheapest way to cut Actions cost with no functional change, the answer is almost always **move to Linux**.

> **Trap 4 — Spending limit vs budget alert.**
> A **spending limit** is a **hard stop** (service halts at the cap — the circuit breaker). A **budget alert** only **notifies** at thresholds; usage continues. If the requirement is "*must not exceed* $X," that's a **spending limit**; if it's "*warn me when approaching* $X," that's a **budget/alert**.

> **Trap 5 — Admin vs GitHub Support boundary.**
> **You** check the status page, audit logs, config, and (GHES) the Monitor dashboard **first**. **GitHub Support** handles platform bugs/outages/appliance failures, and you give them a **support bundle** (`ghe-support-bundle`). Don't open a ticket for something your own audit log or a recent policy change explains.

> **Trap 6 — Larger runners & free minutes.**
> Included/free monthly minutes apply to **standard runners**; **larger runners are billed from minute one** with no free allotment — and self-hosted runners don't draw GitHub-hosted minutes at all.

---

## 📋 Quick reference

| Concept | Key fact / answer |
|---|---|
| **Audit log search** | `key:value` qualifiers: `actor:`, `action:`, `repo:`, `org:`, `created:` (e.g., `action:repo.destroy created:>2026-06-01`) |
| **Long-term audit retention** | **Audit log streaming** → S3 / Azure Blob / Event Hubs / GCS / Splunk |
| **Rate-limit headers** | `X-RateLimit-Limit / -Remaining / -Used / -Reset / -Resource`; check live via `GET /rate_limit` (free) |
| **Throttled responses** | `403` (primary) / `429` (secondary) — back off until `X-RateLimit-Reset` |
| **GHES diagnostics** | **Monitor dashboard** + **support bundle** (`ghe-support-bundle`, `ghe-support-upload`) |
| **Status check** | githubstatus.com before opening a ticket |
| **OS multipliers** | **Linux 1× · Windows 2× · macOS 10×** |
| **Metered products** | Actions minutes, Actions/Packages **storage GB**, Codespaces, Git LFS, premium requests |
| **Per-seat products** | GitHub Enterprise user licenses, **Copilot** seats |
| **GHAS billing** | Per **unique active committer** |
| **Usage report** | **CSV export** from enhanced billing platform; pivot by SKU/OS/repo/cost center |
| **Hard cap** | **Spending limit** (service stops) |
| **Early warning** | **Budget alert** (notify only) |
| **Dormant user** | No activity for a set period (commonly **90 days** default on GHES) → reclaim seat |
| **Cheapest runner win** | Move OS-agnostic jobs to **Linux**; cache deps; concurrency limits |
| **Copilot waste** | Assigned seats with **0 active usage** → reclaim |

---

## 🧠 Self-check

**Q1.** Compliance needs audit data going back two years, but the GitHub UI only shows a limited window. What do you configure?

<details><summary>Answer</summary>

**Audit log streaming** to an external store (Amazon S3, Azure Blob Storage, Azure Event Hubs, Google Cloud Storage, or Splunk). The UI's retention is bounded; streaming provides indefinite long-term retention for compliance and historical analysis.
</details>

**Q2.** A 10-minute integration job runs on a GitHub-hosted **macOS** runner. How many minutes are billed, and how could you cut it cheaply?

<details><summary>Answer</summary>

**100 billed minutes** (macOS multiplier = **10×**). If the job isn't Apple-platform-specific, move it to **Linux (1×)** → 10 billed minutes, a 90% reduction with no functional change.
</details>

**Q3.** Finance says "we must **never** exceed our Actions budget this month." Spending limit or budget alert?

<details><summary>Answer</summary>

**Spending limit** — it's the hard stop (circuit breaker) that halts metered usage at the cap. A **budget alert** only notifies and would let usage continue past the threshold.
</details>

**Q4.** Users report errors from an integration with `403` responses and an `X-RateLimit-Remaining: 0` header. What's happening and what's the fix?

<details><summary>Answer</summary>

The integration hit its **primary rate limit**. It should **back off until the `X-RateLimit-Reset` time** (Unix epoch), then resume — ideally reading the headers proactively and spacing requests. Use `GET /rate_limit` to inspect status without consuming quota.
</details>

**Q5.** You find 190 Copilot seats assigned but with zero active usage in the last 28 days. Is this a metered or per-seat problem, and what do you do?

<details><summary>Answer</summary>

**Per-seat** (Copilot is licensed per user). **Reclaim/unassign the 190 unused seats** — recurring savings with no impact on active developers. A spending limit would not help here because the cost is the seats, not metered consumption.
</details>

**Q6.** Git pushes are slow on your GHES appliance. Walk through the right escalation order.

<details><summary>Answer</summary>

1) Check **githubstatus.com** (if applicable) and the **Management Console → Monitor dashboard** for CPU/disk/memory/queue health. 2) Review recent **config/policy changes** and audit logs. 3) If it's admin-fixable (e.g., disk saturation), resolve it yourself. 4) If not, **generate a support bundle** (`ghe-support-bundle` / `ghe-support-upload`) and **open a GitHub Support ticket**.
</details>

**Q7.** Name three resource-optimization levers for GitHub Actions that don't reduce test coverage.

<details><summary>Answer</summary>

Any three of: **prefer Linux runners** (1× vs 2×/10×), **cache dependencies** (`actions/cache`), **concurrency groups** to auto-cancel superseded runs, **`paths`/branch filters** so trivial changes skip full matrices, **lower artifact retention**, **right-size runners** (avoid larger runners except for heavy jobs), and **self-hosted runners** for steady heavy load.
</details>

**Q8.** How is **GitHub Advanced Security (GHAS)** billed, and what optimization follows from that?

<details><summary>Answer</summary>

GHAS is billed per **unique active committer** to repositories where it's enabled. Optimization: **enable GHAS only on repos that need it** and disable it on dead/archived repos to reduce the active-committer count.
</details>

---

> **One-line takeaway:** Treat GitHub like a metered utility — *read the meter* (usage reports + audit logs), *fix your own fuse box before calling the utility* (diagnostics vs. Support), and *cut the bill without dimming the lights* (Linux runners, reclaimed seats, spending limits).
