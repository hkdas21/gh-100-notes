# Domain 2 — Administer GitHub Enterprise Environment (10–15%)

> **GH-100 study notes.** This domain is about *running* GitHub Enterprise day-to-day: helping users, setting healthy developer standards, and picking + paying for the right deployment. It pairs technical know-how (support bundles, licensing math) with judgment calls (which deployment, where the admin/Support line sits).

---

## Why this domain matters

As a GitHub Enterprise administrator you are the operational owner of the platform. When something breaks, a developer cannot push, or finance asks "why is the bill going up?", you are the first responder. This domain tests three intertwined skills:

1. **Supporting people** — triaging issues you can fix yourself versus escalating to GitHub Support with the right diagnostic evidence (support bundles).
2. **Setting guardrails** — recommending sane developer standards (branching, code review, releases) so thousands of engineers work consistently and safely.
3. **Choosing & paying for the platform** — knowing the four enterprise deployment shapes, when each is appropriate (data residency, isolation, control), and how the hybrid **per-seat + metered** billing model works.

Get this wrong and you either over-escalate trivial tickets, leave repos unprotected, pick a deployment that violates a data-residency law, or blow the budget on unmonitored Actions minutes. It's only 10–15% of the exam, but it's the "you actually run this thing" domain.

---

## Big-picture analogy

Think of yourself as the **facilities manager of a large office building**.

- **Supporting users** = you fix the jammed printer and reset the badge reader yourself (admin-resolvable), but you call the *power company* when the whole block loses electricity (GitHub Support). When you call them, you bring the meter readings and fault logs (a **support bundle**), not just "the lights are off."
- **Developer standards** = you post the building rules: which doors are emergency-only (protected branches), who must sign off on renovations (CODEOWNERS / required reviews), and how you label each floor's renovation (semantic version releases).
- **Deployment & licensing** = deciding whether the company **leases space in GitHub's managed tower** (GitHub-hosted: GHEC) or **builds its own building on company land** (self-hosted: GHES) — and then paying both a fixed **per-desk rent** (per-user license) plus **metered utilities** (Actions minutes, storage, Codespaces).

Keep that building in mind: every concept below maps onto "fix it yourself vs call the utility," "post the rules," or "lease vs build + rent + metered utilities."

---

## 1. Support GitHub Enterprise users and stakeholders

### 1.1 Admin can resolve vs. needs GitHub Support

The core judgment: **is this a configuration / permission / usage issue inside *your* control plane, or is it a fault in GitHub's platform, code, or infrastructure?** The former is yours; the latter is a ticket to GitHub Support.

**Admin can resolve (inside your control):**
- A user can't see a repo → fix org/team membership or repository permissions.
- A developer can't push to `main` → branch protection / ruleset requires a PR or review; adjust the rule or the user's role.
- SSO login loop / SAML mismatch → fix the IdP attribute mapping or the SAML configuration.
- "I ran out of Actions minutes" → raise the spending limit or org policy.
- New hire has no access → add them to the EMU directory group / org.
- A workflow fails because a self-hosted runner is offline → restart the runner.

**Needs GitHub Support (GitHub's responsibility):**
- A platform-wide outage (github.com or GHEC is down for everyone) — check **githubstatus.com** first.
- A reproducible **bug in GHES** or in a GitHub feature (e.g., Actions service returns 500s, a UI feature is broken for all users).
- **GHES upgrade failures**, replication/cluster faults, data corruption, or unexpected behavior after an upgrade.
- License/billing discrepancies you cannot reconcile in the admin UI.
- Anything requiring GitHub to inspect their own infrastructure or source code.

> **Rule of thumb:** if the fix is a *setting, role, policy, or membership* you can click in the admin/Management Console, it's **yours**. If it requires GitHub to touch *their* code or infrastructure, it's **Support**. Always check **githubstatus.com** before opening a ticket for "everything is down."

**Example.** A developer reports "I can't merge my PR — it says a review is required, but I'm the only one on the project." You inspect the repo's **ruleset** and see *Require a pull request before merging → Require 1 approval*. That's a config you own: you adjust the ruleset (or add a reviewer). **No ticket needed.** Contrast: every developer in the company reports `git push` hanging with 500 errors at the same minute, and githubstatus.com shows "Git Operations — degraded." That's a **Support / status-page** matter, not a setting you can fix.

**Analogy.** You're triage nurse. A splinter you remove yourself (permission fix). A heart attack you escalate to the ER with the patient's full chart (support bundle to GitHub Support). Bringing a splinter to the ER wastes everyone's time; trying to do heart surgery yourself is worse.

---

### 1.2 Generate support bundles and diagnostics (GHES)

When you *do* escalate a **GitHub Enterprise Server (GHES)** issue, Support needs evidence. GHES produces **diagnostic bundles** — compressed archives of logs, configuration, and system state — so Support can debug without direct access to your appliance.

| Artifact | What it is | How to get it | Time window |
|---|---|---|---|
| **Support bundle** | `.tgz` of logs + config + diagnostics for the appliance | Management Console → **Support** → *Download support bundle*, or SSH `ghe-support-bundle` | ~last **2 days** of logs |
| **Extended bundle** | Larger bundle covering more history | `ghe-support-bundle -x` | up to ~**7 days** of logs |
| **Cluster support bundle** | Bundle spanning **all nodes** in a clustered / HA deployment | `ghe-cluster-support-bundle` (optionally `-x` for extended) | per-node logs aggregated |
| **Diagnostics** | Lighter snapshot of system info / health (no full logs) | `ghe-diagnostics`, or Management Console diagnostics export | point-in-time |

**Key commands (run over SSH on port 122 to the GHES admin shell):**

```bash
# Generate a standard support bundle (writes a .tgz you then download)
ssh -p 122 admin@HOSTNAME -- 'ghe-support-bundle -o' > support-bundle.tgz

# Extended bundle = more days of logs (use when the issue is older than ~2 days)
ssh -p 122 admin@HOSTNAME -- 'ghe-support-bundle -x -o' > support-bundle-extended.tgz

# Cluster-wide bundle (HA / clustering): collect from every node at once
ssh -p 122 admin@HOSTNAME -- 'ghe-cluster-support-bundle -o' > cluster-bundle.tgz

# Lightweight diagnostics snapshot
ssh -p 122 admin@HOSTNAME -- 'ghe-diagnostics' > diagnostics.txt

# Upload an EXISTING bundle straight to GitHub Support (needs outbound access)
ghe-support-upload -f /path/to/support-bundle.tgz
# Or associate the upload with a ticket number:
ghe-support-upload -t TICKET_ID -f support-bundle.tgz
```

**Three ways to get the bundle to Support:**
1. **Management Console UI** — *Settings → Support → Download support bundle*, then attach it to your ticket on the support portal.
2. **`ghe-support-bundle`** over SSH, download the `.tgz`, upload via the support portal.
3. **`ghe-support-upload`** — pushes a bundle directly from the appliance to GitHub (handy when the file is large or the appliance has outbound connectivity but you don't want to download a multi-GB file locally).

> **Privacy note:** support bundles contain logs and configuration that may include sensitive data. Generate and share them deliberately, only with GitHub Support, and prefer the encrypted upload path. The **Management Console** is the web admin UI (on port **8443**) where you configure the appliance, view health, and download bundles — distinct from the SSH **admin shell** (port **122**) where the `ghe-*` commands live.

**Example.** After a failed GHES upgrade, repository services won't start. You SSH in and run `ghe-cluster-support-bundle -o > cluster.tgz` (it's an HA pair, so you want all nodes), then `ghe-support-upload -t 123456 -f cluster.tgz` to attach it to your open ticket. Support reads the replication logs in the bundle and pinpoints a stuck migration — something you could never have diagnosed from the UI alone.

**Analogy.** The support bundle is the **flight data recorder ("black box")** you hand investigators after an incident — it has the full instrument logs. `ghe-diagnostics` is the quick **dashboard photo** ("temp gauge is in the red") you snap for a fast read. You bring the black box to the manufacturer (GitHub Support); you glance at the dashboard yourself.

---

## 2. Recommend standards for developer processes

Your job isn't to *write* every team's code — it's to **set platform-wide guardrails** so work is consistent, reviewed, and traceable. GitHub gives you the enforcement tools (rulesets, CODEOWNERS, Releases); you recommend the *policy*.

### 2.1 Branching strategy

Recommend a branching model that matches the org's release cadence:

- **Trunk-based development** — everyone integrates into a single `main` (trunk) frequently via short-lived feature branches; releases cut from trunk. Favors CI/CD, fast feedback, fewer long-lived branches. GitHub's own recommended **GitHub Flow** is essentially this: branch → PR → review → merge → deploy.
- **GitFlow** — long-lived `main` + `develop`, plus `feature/*`, `release/*`, and `hotfix/*` branches. More ceremony; suits scheduled/versioned releases and teams that ship infrequently.

**Example.** A SaaS team deploying many times a day adopts **trunk-based / GitHub Flow**: feature branches live hours, merge to `main`, auto-deploy. A firmware team shipping quarterly uses **GitFlow** with `release/2026.Q3` stabilization branches and `hotfix/*` for emergency patches.

**Analogy.** Trunk-based is a **busy single-lane road with frequent on-ramps** — keep merging in, keep traffic flowing. GitFlow is a **rail yard with dedicated staging tracks** — assemble each train (release) on its own siding before sending it out.

### 2.2 Code review & required reviews

Recommend **pull requests** as the unit of change, with **required reviews** enforced by branch protection or a **ruleset**. Combine with **CODEOWNERS** so the right people are auto-requested.

- Require N approvals before merge; dismiss stale approvals on new commits.
- Require **status checks** (CI) to pass.
- Require review from **Code Owners** for sensitive paths.

**`CODEOWNERS`** (lives in `.github/`, repo root, or `docs/`):

```gitignore
# Lines map path patterns → owners (auto-requested as reviewers)
*                       @org/platform-leads
/infra/                 @org/sre-team
/payments/**            @org/payments-team @alice
*.tf                    @org/cloud-infra
/.github/workflows/     @org/devops
```

**Analogy.** CODEOWNERS is the **building's sign-off matrix**: any change to the electrical floor needs the licensed electrician's signature; any change to plumbing needs the plumber's. The doors won't "merge" without the right specialist's stamp.

### 2.3 Protected branches & rulesets

**Branch protection rules** and the newer, more flexible **rulesets** enforce the standards above. Rulesets can target branches/tags, be layered, scoped org-wide, and run in **Evaluate** (dry-run) mode.

Common ruleset/protection controls:
- Require a pull request before merging (+ required approvals).
- Require status checks to pass; require branches up to date.
- Require signed commits; require linear history.
- Block force pushes; restrict who can push/delete.
- Require deployments to succeed; require code scanning results.

**Example.** An org-level **ruleset** targets `main` and `release/*` across all repos: require 2 approvals, require the `build` and `security-scan` checks, require Code Owner review, block force pushes. New repos inherit it automatically — no per-repo setup.

**Analogy.** Rulesets are the **building code enforced at the city level** rather than apartment-by-apartment: every unit must have the smoke detector and the fire door, whether or not the tenant remembers to install one.

### 2.4 Releases & semantic versioning

Standardize how software ships: **Git tags → GitHub Releases** with **Semantic Versioning (SemVer)** `MAJOR.MINOR.PATCH`.

- **MAJOR** — breaking changes (`2.0.0`).
- **MINOR** — backward-compatible features (`1.4.0`).
- **PATCH** — backward-compatible bug fixes (`1.4.3`).

```bash
git tag -a v1.4.0 -m "Add export API"
git push origin v1.4.0
# Then create a GitHub Release from the tag (UI or gh CLI):
gh release create v1.4.0 --generate-notes --title "v1.4.0"
```

**GitHub Releases** attach binaries/assets, auto-generate release notes from merged PRs, and give consumers a stable download point.

**Example.** A library at `v1.4.3` adds a new optional parameter (backward compatible) → release `v1.5.0`. It later removes a deprecated function (breaking) → `v2.0.0`, with auto-generated notes listing every PR since `v1.5.0`.

**Analogy.** SemVer is the **floor-renovation labeling system**: repainting a wall is a *patch* (cosmetic, safe), adding a new conference room is a *minor* (new capability, nothing breaks), and knocking down a load-bearing wall is a *major* (tenants must change how they move around). Anyone reading the label instantly knows how disruptive the change is.

---

## 3. Manage deployment and licensing

### 3.1 GitHub Enterprise deployment scenarios

There are two hosting models — **GitHub-hosted (cloud, GHEC)** and **self-hosted (GHES)** — and within GHEC, three account/data shapes. Memorize the four shapes and *when* to choose each.

1. **GHEC with personal accounts** — Enterprise on github.com where members use their **own personal GitHub accounts** (the ones they also use for open source). The enterprise *invites* existing accounts into orgs. Users own their account and can also participate in public github.com.

2. **GHEC with EMU (Enterprise Managed Users)** — GitHub **provisions and fully controls** user accounts via your **IdP (SCIM/SAML)**. Usernames are namespaced (e.g., `alice_acme`). Accounts are **managed**: users *cannot* use them for personal/public work, can only belong to your enterprise, and outbound collaboration is restricted. Maximum identity control and offboarding.

3. **GHEC with Data Residency + EMU** — runs on **`*.ghe.com`** (a dedicated, regional GHEC instance) so that **data is stored in a chosen region** (e.g., EU). Still cloud-hosted by GitHub, still uses **EMU** identities. For organizations with **data-residency / sovereignty** requirements that still want GitHub to operate the platform.

4. **GHES — GitHub Enterprise Server (self-hosted)** — you run the GHES **virtual appliance** on your **own infrastructure** (on-prem datacenter or your own cloud/VPC). *You* own upgrades, backups, scaling, and the network. Maximum **isolation and control**; can run air-gapped. You administer via the **Management Console** + admin shell, and generate support bundles when you need GitHub Support.

> **Tip:** "EMU = GitHub manages the *identities*." "Data Residency = GitHub stores the *data* in your region." "GHES = *you* manage everything." They're independent ideas that combine (Data Residency *includes* EMU).

#### Deployment comparison table

| | **GHEC + personal accounts** | **GHEC + EMU** | **GHEC + Data Residency + EMU** | **GHES (self-hosted)** |
|---|---|---|---|---|
| **Hosting** | GitHub cloud (github.com) | GitHub cloud (github.com) | GitHub cloud, regional (`*.ghe.com`) | **Your** infrastructure (appliance) |
| **Who creates/owns accounts** | User owns personal account; enterprise invites | **GitHub provisions via IdP (SCIM)**; enterprise owns | **GitHub provisions via IdP (SCIM)**; enterprise owns | You create on the appliance (or via SAML/SCIM you run) |
| **Username** | User's real github.com handle | Namespaced `user_shortcode` | Namespaced `user_shortcode` | Local to your appliance |
| **Data location** | GitHub's default (US) | GitHub's default (US) | **Chosen region** (e.g., EU) | **Wherever you host it** (on-prem / your cloud) |
| **Can users access public github.com with this account** | **Yes** (it's their personal account) | **No** (managed account is locked to the enterprise) | **No** (managed account) | N/A (separate instance; no public access from the appliance account) |
| **Identity / offboarding control** | Lower (user owns the account) | **High** (suspend in IdP → access gone) | **High** (IdP-driven) | High (you control the appliance) |
| **Who runs upgrades/backups** | GitHub | GitHub | GitHub | **You** |
| **Best for** | Teams already on github.com, open-source-friendly, low overhead | Strict identity control, no personal-account mixing, central deprovisioning | Identity control **+ regulatory data-residency** needs, still cloud-managed | **Air-gapped / maximum isolation**, full infrastructure control, on-prem mandates |

**Example.** A US startup with engineers already active in open source picks **GHEC + personal accounts** — minimal friction, devs keep their handles. A regulated EU bank that must keep source in-region but doesn't want to run servers picks **GHEC + Data Residency + EMU** (`bank.ghe.com`, EU data, IdP-managed identities). A defense contractor on an isolated network with no internet egress runs **GHES** in their own datacenter.

**Analogy.** Housing choices:
- **GHEC + personal accounts** = renting a managed apartment but using your *own* name on the mailbox (you also have a life outside the building).
- **GHEC + EMU** = a **corporate apartment** the company assigns you; your badge only works here and HR can revoke it instantly.
- **GHEC + Data Residency + EMU** = the same corporate apartment, but the building is required to physically sit **in your country**.
- **GHES** = the company **builds and owns the whole building** on its own land — total control, but they also fix the plumbing and pay the property taxes.

---

### 3.2 Licensing and billing models

GitHub Enterprise billing is a **hybrid**: a fixed **per-user license** *plus* **metered (consumption) usage**.

**Per-user (per-seat) — the GitHub Enterprise license:**
- You buy a number of **GitHub Enterprise seats**. Each *unique person* who is a member of the enterprise consumes **one** license, no matter how many orgs they're in.
- The same license covers a user across **GHEC and GHES together** when joined via **GitHub Connect** (see below) — one human = one seat across both.
- **GitHub Advanced Security (GHAS)** is also licensed **per active committer (seat)** for the security features (code scanning, secret scanning, etc.). (GHAS has been repackaged into **GitHub Secret Protection** and **GitHub Code Security** products, still committer/seat-based.)

**Metered / consumption (pay for what you use):**
- **GitHub Actions** — billed by **compute minutes** (with multipliers for larger/non-Linux runners) plus storage for artifacts; each plan includes a free monthly allotment, overage is metered.
- **GitHub Packages** — billed by **storage (GB)** and **data transfer/bandwidth**, with an included allowance.
- **GitHub Codespaces** — billed by **compute hours** (core-hours) **and storage (GB-month)**.
- **Secret scanning / GHAS seats** — note: the *GHAS capability* is **per-seat**, but it's commonly grouped with "consumption" because you add/remove security seats over time. Be careful on the exam (see traps).
- **Large File Storage (Git LFS)** — additional **data packs** for storage and bandwidth.

> **Spending limits & budgets:** metered usage is governed by a **spending limit**. Default is often **$0** (no overage past the included amount) until an admin raises it. You set budgets/alerts in the enterprise/organization billing settings to avoid surprise bills.

**GitHub Connect — sharing one license across GHEC + GHES:**
**GitHub Connect** links a **GHES** instance to **GHEC**. Benefits:
- **License sync / unified seats** — a user with an account on both GHEC and GHES counts as **one** consumed license (you upload/sync the license between them rather than buying two).
- Optionally share features like **server-to-cloud Actions**, **Dependabot/dependency insights**, and **unified search/contributions**.

```text
Billing model summary
┌───────────────────────────┬──────────────────────────────────────────┐
│ Per-USER (fixed seat)      │ Per-USAGE (metered/consumption)            │
├───────────────────────────┼──────────────────────────────────────────┤
│ • Enterprise license seat  │ • Actions minutes (+ runner multipliers)   │
│ • GHAS / Code Security /   │ • Packages storage + bandwidth             │
│   Secret Protection seats  │ • Codespaces compute hours + storage       │
│   (per active committer)   │ • Git LFS data packs                       │
│ 1 human = 1 seat across    │ Governed by a SPENDING LIMIT (default $0)  │
│ GHEC+GHES via GitHub Connect│ Included allowance, then metered overage  │
└───────────────────────────┴──────────────────────────────────────────┘
```

**Example.** Acme buys **500 Enterprise seats**. A developer who belongs to 4 orgs still uses **1** seat. They also enable Actions; the org gets an included minute allowance, and Acme sets a **$2,000/month** Actions spending limit so overage is capped. Security team enables **GHAS** for 120 active committers → 120 security seats. Via **GitHub Connect**, a user who exists on both their GHES appliance and GHEC counts once.

**Analogy.** It's a **gym membership + utilities** bill. The **per-seat license** is the flat monthly membership — one card per member, swiping into any room counts as one membership. The **metered usage** is the electricity/water — Actions minutes, storage, and Codespaces hours are the meters spinning while you use the equipment. **GitHub Connect** is the family plan that lets one membership work at both the downtown branch (GHEC) and the suburban branch (GHES) without paying twice.

---

### 3.3 Monitor license usage and consumption

You must be able to answer "how many seats are consumed, by whom, and how much metered usage are we burning?"

**Where to look:**
- **Enterprise → Settings → Billing and licensing** — shows **consumed licenses vs. purchased**, GHAS seats, and metered usage (Actions, Packages, Storage, Codespaces).
- **Enterprise licensing → "Consumed licenses"** — per-user breakdown of who is using a seat, their account, and whether their GHEC and GHES identities are linked.
- **Download the license usage report (CSV/JSON)** — exportable list of all licensed users; essential for reconciling seats (e.g., finding duplicate identities not linked across GHEC/GHES that are double-consuming).
- **For GHES**, you upload the `.ghl` **license file** and can sync usage to GHEC via **GitHub Connect**; the consumed-licenses view then de-duplicates users across both.
- **Metered usage / billing pages** — current Actions minutes, Packages GB, Codespaces hours against limits; you can export **usage reports** and set **budgets/alerts**.

```bash
# Example: pull consumed-license data programmatically (GHEC) via the API
gh api /enterprises/{enterprise}/consumed-licenses --paginate

# Inspect Actions/Packages/Codespaces billing for the enterprise
gh api /enterprises/{enterprise}/settings/billing/actions
gh api /enterprises/{enterprise}/settings/billing/packages
gh api /enterprises/{enterprise}/settings/billing/shared-storage
```

**Example.** Finance says "we bought 500 seats but the bill implies 540." You open **Consumed licenses**, **download the CSV**, and discover ~40 developers have a GHES account *and* a separate GHEC account that aren't linked. You enable/repair **GitHub Connect license sync**, the duplicates collapse, and consumption drops back under 500 — no extra seats purchased.

**Analogy.** This is the **building's badge-access + utility-meter report**. The consumed-licenses CSV is the **badge log** (who actually walked in this month), letting you spot the contractor who was issued *two* badges and is being double-counted. The metered usage dashboards are the **utility meters** on the wall — you read them monthly so the electricity bill never surprises you.

---

## ⚠️ Exam traps

- **EMU ≠ Data Residency.** EMU is about **who controls the identities** (GitHub provisions managed accounts via your IdP; they can't be used for personal/public work). **Data Residency** is about **where the data physically lives** (a regional `*.ghe.com` instance). Data Residency *uses* EMU, but EMU alone does **not** move your data to another region.
- **Managed users can't touch public github.com.** With **EMU**, accounts are locked to the enterprise — no contributing to open source, no personal repos. With **personal accounts** in GHEC, users *can* still use github.com publicly. A classic "which deployment lets devs also do open source?" question → **personal accounts**.
- **Per-seat vs metered.** The **Enterprise license** and **GHAS/security seats** are **per-user**. **Actions minutes, Packages storage, Codespaces hours, LFS** are **metered/consumption**. Don't say "Actions is per-seat" or "the Enterprise license is metered."
- **One human = one seat.** A user in many orgs still consumes **one** license. Double-counting usually means **unlinked GHEC/GHES identities** — fix with **GitHub Connect license sync**, don't buy more seats.
- **Admin vs Support boundary.** Permission/role/policy/SSO-mapping fixes = **admin**. Platform outage, **GHES bug/upgrade failure**, infra/replication faults = **GitHub Support**. Check **githubstatus.com** before escalating "everything's down."
- **Support bundle vs diagnostics.** A **support bundle** (`ghe-support-bundle`) = full logs + config (the black box); **`ghe-diagnostics`** = a lighter health snapshot. Older issue (>~2 days) → use the **extended** bundle (`-x`). HA/cluster → **`ghe-cluster-support-bundle`**. Send via portal or **`ghe-support-upload`**.
- **Management Console (port 8443) ≠ admin shell (port 122).** The Console is the web UI for appliance config and downloading bundles; the `ghe-*` commands run in the SSH admin shell.
- **Branch protection vs rulesets.** Rulesets are the newer, layerable, org-scopable mechanism (with an **Evaluate/dry-run** mode); branch protection rules are the classic per-branch version. Both enforce required reviews/checks.
- **Spending limit defaults to $0.** Overage beyond the included allowance is blocked until an admin raises the limit — a "why did our builds stop running?" gotcha.

---

## 📋 Quick reference

| Topic | Key fact |
|---|---|
| **Admin-fixable** | Permissions, roles, org/team membership, ruleset/branch protection, SAML/SCIM mapping, spending limits |
| **Support-needed** | Platform outage, GHES bug, upgrade/replication failure, data corruption, infra-level issues |
| **First step on outage** | Check **githubstatus.com** before opening a ticket |
| **Support bundle (standard)** | `ghe-support-bundle -o` (~2 days of logs) |
| **Support bundle (extended)** | `ghe-support-bundle -x -o` (~7 days) |
| **Cluster bundle** | `ghe-cluster-support-bundle -o` (all HA nodes) |
| **Quick health snapshot** | `ghe-diagnostics` |
| **Upload to Support** | `ghe-support-upload -t TICKET -f bundle.tgz`, or Management Console / support portal |
| **Admin shell / Console ports** | SSH admin shell **122**, Management Console **8443** |
| **Branching models** | Trunk-based / GitHub Flow (fast CD) vs GitFlow (scheduled releases) |
| **Enforcement** | Rulesets / branch protection: required reviews, status checks, signed commits, no force-push |
| **Reviewer routing** | `CODEOWNERS` auto-requests path owners |
| **Versioning** | SemVer `MAJOR.MINOR.PATCH`; ship via Git tag → GitHub Release |
| **Deployments** | GHEC + personal accounts / GHEC + EMU / GHEC + Data Residency + EMU / GHES |
| **EMU** | GitHub-managed identities via IdP/SCIM; locked to enterprise; no public github.com |
| **Data Residency** | Regional `*.ghe.com`, data stored in chosen region, uses EMU |
| **GHES** | Self-hosted appliance, you own upgrades/backups, max isolation |
| **Per-seat billing** | Enterprise license + GHAS/Code Security/Secret Protection seats; 1 human = 1 seat |
| **Metered billing** | Actions minutes, Packages storage/bandwidth, Codespaces hours/storage, LFS data packs |
| **One license, both platforms** | **GitHub Connect** license sync across GHEC + GHES |
| **Spending limit default** | Often **$0** (overage blocked until raised) |
| **Monitor licenses** | Enterprise → Billing & licensing → **Consumed licenses**; download usage **CSV** |

---

## 🧠 Self-check

**1.** A developer can't push to `main`; the error says a pull request review is required. Is this an admin fix or a GitHub Support ticket, and why?

<details>
<summary>Answer</summary>
<strong>Admin fix.</strong> It's a branch protection rule / ruleset (<em>require a pull request / required approvals</em>) — a setting you own. Adjust the ruleset or have the change go through a PR. No platform fault, so no Support ticket.
</details>

**2.** GHES had a failed upgrade and repository services won't start across your two-node HA pair. Which exact command collects diagnostics, and how do you get it to Support?

<details>
<summary>Answer</summary>
Use <code>ghe-cluster-support-bundle -o</code> (it spans all HA nodes; add <code>-x</code> for extended history). Send it via the support portal attachment <em>or</em> push it directly with <code>ghe-support-upload -t TICKET_ID -f cluster.tgz</code>.
</details>

**3.** What's the difference between a support bundle and `ghe-diagnostics`?

<details>
<summary>Answer</summary>
A <strong>support bundle</strong> is a full archive of logs + configuration + system state (the "black box," ~2 days standard / ~7 with <code>-x</code>) used by Support to debug. <strong><code>ghe-diagnostics</code></strong> is a lighter point-in-time health/system snapshot — a quick read, not a full log dump.
</details>

**4.** An EU bank must keep its source code stored in the EU but does not want to operate any servers. It also wants central, IdP-driven identity control with no personal-account mixing. Which deployment?

<details>
<summary>Answer</summary>
<strong>GHEC with Data Residency + EMU</strong> (a regional <code>*.ghe.com</code> instance). Data stays in the EU region, GitHub still operates the platform (no servers for the bank to run), and EMU gives IdP-managed identities locked to the enterprise.
</details>

**5.** True or false: with Enterprise Managed Users (EMU), members can also use their accounts to contribute to open-source projects on public github.com.

<details>
<summary>Answer</summary>
<strong>False.</strong> EMU accounts are managed and locked to the enterprise — they cannot be used for personal or public github.com activity. If devs need that, you'd use <strong>GHEC with personal accounts</strong> instead.
</details>

**6.** Classify each as per-seat or metered: (a) GitHub Enterprise license, (b) Actions minutes, (c) GHAS/Code Security, (d) Codespaces compute hours, (e) Packages storage.

<details>
<summary>Answer</summary>
<strong>Per-seat:</strong> (a) Enterprise license, (c) GHAS / Code Security / Secret Protection (per active committer). <strong>Metered/consumption:</strong> (b) Actions minutes, (d) Codespaces hours, (e) Packages storage (+ bandwidth).
</details>

**7.** Finance says you're consuming 540 licenses but only bought 500, and headcount hasn't grown. Most likely cause and fix?

<details>
<summary>Answer</summary>
Users likely have <strong>separate, unlinked GHEC and GHES identities</strong> being counted twice. Enable/repair <strong>GitHub Connect</strong> license sync so each human collapses to <strong>one</strong> seat. Verify via the <strong>Consumed licenses</strong> view and the downloadable license-usage CSV — don't buy more seats.
</details>

**8.** Your Actions builds suddenly stopped running mid-month with no outage reported. What's the most likely administrative cause?

<details>
<summary>Answer</summary>
You hit the included Actions minute allowance and the <strong>spending limit</strong> (often defaulted to <strong>$0</strong>) blocked overage. Raise the spending limit / budget in enterprise billing settings to resume metered usage.
</details>

---

*End of Domain 2 study notes — Administer GitHub Enterprise Environment (10–15%).*
