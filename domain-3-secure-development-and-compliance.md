# Domain 3 — Implement Secure Software Development and Compliance (25–30%)

> GH-100 GitHub Enterprise Administrator — Study Notes
> The single heaviest domain on the exam.

---

## Why this domain matters

This domain is worth **25–30% of the GH-100 exam** — roughly **27%** on average, which makes it the **single largest scoring opportunity** on the test. More questions come from here than from any other domain, and they tend to be the *scenario* style ("An org owner wants X without breaking Y — what do you configure?") rather than simple recall. If you only have time to master one domain deeply, master this one. The material clusters into three big buckets: (1) **policies, rulesets, and audit logging** (governance), (2) **GitHub Advanced Security / GHAS features** (Dependabot, secret scanning, code scanning), and (3) **API access and app integrations** (PATs, GitHub Apps vs OAuth Apps, rate limits, approval policies). Expect the exam to repeatedly test the *boundaries* between similar-sounding features — that is where points are won and lost.

---

## Big-picture analogy

Think of securing a GitHub Enterprise like running a **modern airport with layered security** (defense-in-depth):

- **Enterprise/org/repo policies** are the **airport rules and federal regulations** — set centrally, cascade down to every gate, and a local gate agent can be *more* strict but never *less* strict than the federal floor.
- **Rulesets & branch protection** are the **boarding gates and jet-bridge checks** — nobody reaches the plane (the protected branch) without passing required checks.
- **GHAS (Dependabot / secret scanning / code scanning)** are the **three different scanners**: the baggage X-ray (dependency/Dependabot — checks what you *bring in*), the metal detector at the checkpoint (secret scanning / push protection — stops dangerous items *leaving with you* into the repo), and the behavioral threat analysis (CodeQL code scanning — inspects *your own code* for dangerous patterns).
- **Audit log + streaming** is the **CCTV recording everything**, with the footage streamed off-site to a vault (S3/Splunk) so it can't be tampered with.
- **PATs, GitHub Apps, OAuth Apps** are the **credentials, vendor badges, and visitor passes** — short-lived, least-privilege, and revocable, with a security desk (org owner) that approves or denies who gets in.

Keep this airport image in your head; almost every exam scenario maps onto one of these layers.

---

## 1. Configure security policies and rulesets

### 1.1 Define organization and enterprise policies (the policy cascade)

**Plain English:** GitHub governance flows **top-down** through three tiers: **Enterprise → Organization → Repository**. An enterprise owner sets the *baseline* (the floor). Each organization can tighten — but never loosen — what the enterprise mandated. Repos inherit org settings. Many enterprise policies offer a value like **"Enabled," "Disabled," or "No policy"** — where *No policy* means "let the organizations below decide for themselves." Once a higher tier *enforces* a setting, lower tiers cannot override it.

Examples of policies governed this way: repository creation, repository visibility (can orgs create public repos?), repository deletion/transfer, forking of private/internal repos, member-can-create-pages, default branch name, GitHub Actions usage, and Advanced Security enablement.

**Example:**
An enterprise owner sets **"Repository visibility → members can only create private repositories."** Org A would *like* to allow public repos, but cannot — the enterprise floor wins. Meanwhile the enterprise sets **base permissions to "No policy,"** so each org independently chooses Read/Write/None for its members.

```text
Enterprise policy: "Members can create repositories" = Private only
   └── Org "payments"  → can further restrict to "no new repos at all" ✅ (more strict OK)
   └── Org "marketing" → CANNOT allow public repos ❌ (less strict blocked)
```

**Analogy:** Think **federal → state → city law**. Federal law (enterprise) sets the national minimum. A state (org) can pass *stricter* laws but can't legalize what the feds banned. A city (repo) operates inside both. "No policy" at the federal level = "states' rights" — each state decides.

---

### 1.2 Rulesets vs classic branch protection

**Plain English:** **Rulesets** are the modern, layerable successor to **classic branch protection rules**. Key differences:

- **Layering:** You can have *multiple* rulesets apply to the same branch simultaneously, and they **combine** (the most restrictive union wins). Classic branch protection rules do **not** layer cleanly — for a given branch the most specific single rule applied.
- **Scope / targeting:** Rulesets can be created at the **organization level** and target **multiple repositories at once** (by name pattern, by `~ALL`, by repository property, or custom lists). Classic branch protection is **per-repository only**.
- **Branch *and* tag targeting:** Rulesets protect **branches AND tags** (and even pushes). Classic protection covered branches only.
- **Enforcement levels:** Rulesets have three states — **Active** (enforced), **Evaluate** (logged/"dry-run" — see what *would* be blocked without blocking; **org-level / GHE feature**), and **Disabled** (off). Classic branch protection is just on/off.
- **Bypass lists:** Rulesets have an explicit **bypass list** (specific roles/teams/apps that may skip the rules), with granularity like "Always allow" vs "allow for pull requests only." Classic protection only had a coarse "do not allow administrators to bypass / include administrators" toggle.
- **Visibility:** A user with read access can *see* which rulesets apply to a branch (transparency), even if they can't edit them.

> [!IMPORTANT]
> Rulesets do **not** automatically delete or replace classic branch protection. Both can coexist on the same repo, and **both are evaluated** — the effective protection is the **combination** (most restrictive). This coexistence is a classic exam trap.

**Example — an org-level ruleset targeting all repos:**

```yaml
# Conceptual representation of an org ruleset (configured in UI / REST API)
name: "Require PR review on default branch"
target: branch
enforcement: active            # active | evaluate | disabled
conditions:
  ref_name:
    include: ["~DEFAULT_BRANCH"]   # every repo's default branch
  repository_name:
    include: ["~ALL"]              # all repos in the org
rules:
  - type: pull_request
    parameters:
      required_approving_review_count: 2
      dismiss_stale_reviews_on_push: true
      require_code_owner_review: true
  - type: required_status_checks
    parameters:
      required_status_checks:
        - context: "build"
  - type: required_signatures        # require signed commits
  - type: deletion                    # block branch deletion
  - type: non_fast_forward            # block force pushes
bypass_actors:
  - actor_type: Team
    actor_id: 42                      # "release-engineering" team
    bypass_mode: pull_request         # may bypass only via PR
```

**Enforcement-level cheat:**

| Level | Behavior |
|---|---|
| **Active** | Rules enforced; violations blocked. |
| **Evaluate** | Rules **not** enforced, but violations are **logged** so you can preview impact before going live (a "dry run"). |
| **Disabled** | Ruleset is off entirely. |

**Analogy:** Classic branch protection = **one bouncer at one club door** checking IDs his own way. Rulesets = a **corporate security policy** the parent company pushes to *every* club at once, where **multiple policies stack** (dress code AND age AND guest-list), with a **VIP bypass list**, and an **"evaluate" mode** that's like running the metal detector but only *flagging* people instead of stopping them — so you can see who *would* have been turned away before you flip it to live.

---

### 1.3 Strengthen enterprise security posture and data protection

**Plain English:** Beyond rulesets, the enterprise/org owner has a toolbox to harden the whole estate:

- **Security overview (dashboard):** An org/enterprise-level dashboard aggregating GHAS alerts — Dependabot, secret scanning, and code scanning — across all repos. Shows coverage ("which repos have features enabled"), open/closed alert trends, and risk. This is your **single pane of glass** for security posture.
- **Push protection:** A secret-scanning feature that **blocks a `git push`** when it detects a supported secret in the commits, *before* the secret ever lands in the repo history. Can be enabled org-wide.
- **IP allow lists:** Restrict which **source IP addresses** can access an organization's private/internal resources. Supports CIDR ranges, and can be configured to **include GitHub App installations' IPs** automatically.
- **Default workflow (Actions) permissions:** Set the default `GITHUB_TOKEN` permission to **read-only** (recommended) instead of read/write, and control whether workflows can **approve pull requests** or create PRs. Enforceable at enterprise/org level.
- **Other hardening:** require **two-factor authentication (2FA)** for all members, **SSO/SAML** + SCIM for provisioning, **verified domains**, and restricting which **Actions** can run (allow GitHub-owned / verified-creator / specific allowlist).

**Example — enforce least-privilege `GITHUB_TOKEN`:**

```yaml
# Per-workflow override, but the org default should already be read-only
permissions:
  contents: read          # default everything to read
  pull-requests: write    # grant write only where needed
```

Org setting: **Settings → Actions → General → Workflow permissions → "Read repository contents and packages permissions"** (read-only default), and **uncheck** "Allow GitHub Actions to create and approve pull requests."

**Example — IP allow list entry:**

```text
203.0.113.0/24    "HQ office"           enabled
198.51.100.42/32  "VPN egress"          enabled
☑ Enable IP allow list configuration for installed GitHub Apps
```

**Analogy:** The **security operations center (SOC)** of the airport. The **security overview dashboard** is the wall of monitors showing every checkpoint's status at a glance. **IP allow lists** are the perimeter fence — only badge-holders coming from approved entrances get in. **Default read-only workflow permissions** is giving every robot worker a key that only *opens* doors, not one that *changes the locks* — you hand out the master key only to the few who truly need it.

---

### 1.4 Implement audit logging and reporting

**Plain English:** The **enterprise/org audit log** records *who did what, when, and from where* across the account — security-relevant events like permission changes, repo deletions, member additions, secret-scanning alerts, OAuth app authorizations, and (optionally) **Git events** (clone, fetch, push). Key concepts:

- **Event types** are namespaced like `repo.create`, `org.add_member`, `org.update_member`, `business.add_admin`, `protected_branch.policy_override`, `secret_scanning_alert.create`, `oauth_application.create`, `git.clone`, `git.push`.
- **Retention:** In the GitHub-hosted web UI / API, audit log events are queryable for a **limited window** (commonly **up to ~6 months / 180 days** by search; Git events have a *shorter* retention than non-Git events). For **long-term / compliance retention** you must use **audit log streaming**, which sends events to external storage you control with **indefinite** retention.
- **Audit log streaming targets:** **Amazon S3**, **Azure Blob Storage**, **Azure Event Hubs**, **Google Cloud Storage (GCS)**, **Splunk**, and **Datadog**. Streaming delivers events as they occur (near-real-time), and you can **pause/resume** a stream.
- **Access & export:** View in the UI, query via the **REST/GraphQL audit log API**, or export to **CSV/JSON**.

> [!WARNING]
> The web UI/API search window is **finite**. If the exam asks how to keep audit data for **years** (compliance), the answer is **audit log streaming** to S3/Azure/Splunk/etc. — *not* relying on the built-in search.

**Example — query with the gh CLI / API filter:**

```bash
# Find who deleted repositories in the last week (org audit log)
gh api "/orgs/ACME/audit-log?phrase=action:repo.destroy&include=all" \
  --jq '.[] | {actor, repo, created_at}'

# Common audit-log filter phrases:
#   action:org.add_member          (someone joined)
#   action:org.update_member       (role changed)
#   action:repo.create
#   action:protected_branch.*      (branch protection changes)
#   actor:octocat                  (everything a user did)
#   created:>2026-06-01            (date range)
```

**Example — enabling streaming to S3 (conceptual):**

```text
Enterprise → Settings → Audit log → Log streaming → Configure stream → Amazon S3
  Bucket:        acme-github-audit-logs
  Auth:          OIDC (recommended) or access keys
  Region:        us-east-1
  ✅ Stream Git events (clone/fetch/push)
→ "Check endpoint"  → Save
```

**Analogy:** The **airport's CCTV system**. The control room can rewind the **last few months** of footage on its local recorder (the UI search window — and the busy *baggage-belt cameras* (Git events) get overwritten *sooner* because they generate so much footage). But for legal/compliance you **stream every frame off-site to a tamper-proof vault** (S3/Splunk) where it's kept forever and an attacker who breaches the airport can't erase it.

---

## 2. Enable repository security features (GitHub Advanced Security — GHAS)

> **Licensing note:** On public repos, dependency graph, Dependabot alerts/updates, secret scanning alerts, and code scanning are **free**. On **private/internal** repos, **secret scanning** and **code scanning** require a **GitHub Advanced Security (GHAS)** license; dependency graph + Dependabot are available without GHAS. (As of GitHub's 2025 repackaging, secret protection and code security are also sold as standalone "GitHub Secret Protection" and "GitHub Code Security" products — but for the exam, know them as the GHAS feature set.)

### 2.1 Dependency graph, Dependabot alerts, security updates, and version updates

These four are **distinct** and constantly confused on the exam. Learn the ladder:

**(a) Dependency graph** — *Plain English:* GitHub parses your manifest/lockfiles (`package-lock.json`, `requirements.txt`, `go.mod`, `pom.xml`, etc.) to build a graph of **direct and transitive dependencies**. It's the **foundation** — every Dependabot feature depends on it being enabled.

**(b) Dependabot alerts** — *Plain English:* When a dependency in your graph matches a known vulnerability in the **GitHub Advisory Database (GHSA)**, GitHub raises an **alert**. Alerts *notify* you — they do **not** change code.

**(c) Dependabot security updates** — *Plain English:* Automatically opens **pull requests that bump a vulnerable dependency to the minimum patched version** in response to an alert. **Reactive** — triggered by a vulnerability.

**(d) Dependabot version updates** — *Plain English:* On a **schedule you define** in `.github/dependabot.yml`, opens PRs to keep dependencies **up to date**, regardless of whether there's a vulnerability. **Proactive / hygiene.** Requires the config file; security updates do **not** require the config file.

> [!IMPORTANT]
> **Security updates = reactive (vuln-driven), no config file required.**
> **Version updates = proactive (schedule-driven), REQUIRES `dependabot.yml`.**

**Example — `.github/dependabot.yml` (version updates):**

```yaml
version: 2
updates:
  - package-ecosystem: "npm"          # which manifest
    directory: "/"                    # where it lives
    schedule:
      interval: "weekly"              # daily | weekly | monthly
      day: "monday"
    open-pull-requests-limit: 5
    labels: ["dependencies", "javascript"]
    groups:                           # batch related updates into one PR
      dev-dependencies:
        dependency-type: "development"
    ignore:
      - dependency-name: "lodash"
        versions: ["4.x"]             # pin / skip specific versions

  - package-ecosystem: "github-actions"   # keep Actions pinned & current
    directory: "/"
    schedule:
      interval: "weekly"

  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "monthly"
```

**Analogy:** Your home's **medicine cabinet**.
- **Dependency graph** = the *inventory list* of every medicine you own (including the ones hidden behind others — transitive deps).
- **Dependabot alerts** = a *recall notice* in the mail saying "Brand X aspirin is contaminated."
- **Security updates** = a service that, on getting the recall, *automatically ships you the fixed bottle* (reactive PR).
- **Version updates** = a *subscription* that refreshes your whole cabinet on a schedule so nothing expires (proactive PRs), recall or not.

---

### 2.2 Secret scanning + push protection

**Plain English:** **Secret scanning** inspects repository contents (and history) for **credentials** — API keys, tokens, connection strings — using a library of **partner patterns** (AWS, Azure, Stripe, Google, etc., maintained jointly with the providers) plus any **custom patterns** you define with regex. When a known *partner* secret is found, GitHub can notify the partner so they can **auto-revoke** it.

- **Validity checks:** For supported providers, secret scanning can ping the provider to report whether a leaked token is **active/valid** vs already revoked — so you triage real risk first.
- **Custom patterns:** Define your own regex (e.g., an internal token format `ACME_[A-Z0-9]{32}`) at repo/org/enterprise level, with a **dry run** to test against existing content before going live.
- **Push protection:** A *preventive* layer that **blocks the `git push`** the moment a recognized secret is detected in the outgoing commits — stopping the leak **before** it enters history. A developer can **bypass** with a reason (e.g., "it's a test value," "used in tests," "will fix later"), and that bypass is **audited/alerted** to admins.

> [!IMPORTANT]
> **Secret scanning (detection) = finds secrets *already committed* — reactive/after-the-fact.**
> **Push protection (prevention) = stops the secret *at push time* — proactive/before-the-fact.**
> Push protection is *built on top of* secret scanning; you can run secret scanning without push protection, and turning on push protection org-wide is a best practice.

**Example — custom pattern (conceptual config):**

```text
Name:               ACME internal token
Secret format:      ACME_[A-Za-z0-9]{32}
Before secret:      (^|[^A-Za-z0-9])
After secret:       ([^A-Za-z0-9]|$)
[ Run dry run ]  →  reviews matches across the org before enabling
```

**Example — what a developer sees when push protection blocks them:**

```text
$ git push
remote: error: GH013: Repository rule violations found for refs/heads/main
remote: —— Push cannot contain secrets ————————————————————————
remote:   AWS Access Key ID detected in config/settings.py:42
remote:   To push, remove the secret, or bypass with a reason at:
remote:   https://github.com/ACME/app/security/secret-scanning/unblock-secret/...
remote: ————————————————————————————————————————————————————————
! [remote rejected] main -> main (push declined due to repository rule violations)
```

**Analogy:** The **airport checkpoint metal detector**.
- **Secret scanning** = a guard reviewing *yesterday's* X-ray archive and finding a knife that already got through — "uh oh, we need to confiscate it now" (reactive).
- **Push protection** = the **metal detector at the gate** that *beeps and stops you before you board* (proactive). You can argue your way past ("it's a plastic toy") — but that **override is logged** and reported to the supervisor.
- **Validity checks** = the guard radioing the issuing bank to ask "is this stolen credit card still active?" so they chase the live threats first.

---

### 2.3 Code scanning with CodeQL

**Plain English:** **Code scanning** performs **static analysis (SAST)** on your source to find vulnerabilities and coding errors (SQL injection, XSS, path traversal, hard-coded crypto misuse, etc.). GitHub's first-party engine is **CodeQL**, which treats code as a database you can query. Two setup modes:

- **Default setup:** One-click (or org-wide bulk) enablement. GitHub auto-detects languages, picks a query suite, and manages the analysis — **no workflow file to maintain**. Best for most teams.
- **Advanced setup:** GitHub generates a **`codeql.yml` Actions workflow** you fully control — custom build steps (for compiled languages), custom query packs, specific `paths`/`paths-ignore`, matrix of languages, scheduled scans, etc.

- **Third-party SARIF:** Code scanning is engine-agnostic via the **SARIF** format. You can run *any* third-party SAST tool (ESLint, Semgrep, Snyk, etc.) and **upload its SARIF** results so they appear in the same **Security → Code scanning alerts** UI alongside CodeQL.

**Example — advanced setup workflow:**

```yaml
# .github/workflows/codeql.yml
name: "CodeQL"
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: "0 3 * * 1"          # weekly deep scan, Monday 03:00 UTC
jobs:
  analyze:
    runs-on: ubuntu-latest
    permissions:
      security-events: write     # required to upload results
      contents: read
    strategy:
      matrix:
        language: ["javascript-typescript", "python"]
    steps:
      - uses: actions/checkout@v4
      - uses: github/codeql-action/init@v3
        with:
          languages: ${{ matrix.language }}
          queries: security-extended    # broader query suite
          # config-file: ./.github/codeql/codeql-config.yml
      - uses: github/codeql-action/autobuild@v3
      - uses: github/codeql-action/analyze@v3
```

```yaml
# .github/codeql/codeql-config.yml  (optional, referenced above)
name: "ACME CodeQL config"
queries:
  - uses: security-and-quality
paths-ignore:
  - "**/test/**"
  - "third_party/**"
```

```bash
# Upload third-party SARIF results into the Security tab
gh api /repos/ACME/app/code-scanning/sarifs \
  -f commit_sha="$GITHUB_SHA" -f ref="refs/heads/main" \
  -f sarif="$(gzip -c results.sarif | base64 -w0)"
```

**Analogy:** **CodeQL** is the airport's **behavioral threat-detection officer** who watches *how you act* — not what you carry — and flags suspicious patterns in your own conduct (your code). **Default setup** is hiring the standard, trained officer who follows the proven playbook; **advanced setup** is writing your own custom interrogation script. **SARIF upload** is letting *outside* security contractors file their reports into the *same* central incident system so everything shows on one board.

---

### 2.4 Security advisories (private vulnerability reporting, GHSA, CVEs)

**Plain English:** A **repository security advisory** lets maintainers **privately discuss, fix, and disclose** a vulnerability in *their own* project. Mechanics:

- **Private collaboration:** Maintainers open an advisory draft in a **private space**, optionally spin up a **temporary private fork** to develop and review the fix without tipping off attackers, then **publish** once a patch is ready.
- **Private vulnerability reporting (PVR):** A repo setting that gives security researchers a **private channel** to report a vuln directly to maintainers (instead of a public issue that discloses it to the world). Maintainers enable it per-repo or org-wide.
- **GHSA & CVE:** Published advisories get a **GHSA ID** (GitHub Security Advisory) and maintainers can **request a CVE ID** (GitHub is a CVE Numbering Authority). Published advisories flow into the **GitHub Advisory Database**, which is exactly what **Dependabot alerts** consume — closing the loop.
- **Credit:** Advisories can credit the reporting researcher.

**Example flow:**

```text
1. Researcher uses Private Vulnerability Reporting → submits "auth bypass in /login"
2. Maintainer opens a draft advisory (private), opens a temporary private fork
3. Fix developed + reviewed privately; severity scored (CVSS), CWE tagged
4. Maintainer requests a CVE  →  advisory gets GHSA-xxxx-xxxx-xxxx + CVE-2026-NNNNN
5. Patch released → advisory PUBLISHED
6. Advisory enters GitHub Advisory Database → downstream users get Dependabot alerts
```

**Analogy:** A **responsible disclosure "war room."** Private vulnerability reporting is the **confidential tip line** (instead of someone shouting the flaw across the public terminal). The draft advisory + temporary private fork is the **sealed room** where you build the fix behind closed doors so adversaries don't pounce. Publishing with a **GHSA/CVE** is the official **press release with a tracking number** — and that number is what *everyone else's* recall system (Dependabot) keys off of.

---

### 2.5 Define and implement a security response plan

**Plain English:** GHAS surfaces alerts; a **response plan** defines what humans do with them. Core elements an admin/org owner establishes:

- **Triage:** Classify each alert by **severity** (Critical/High/Medium/Low, often via CVSS) and **exploitability/validity**. Prioritize **valid, critical, internet-reachable** issues first. Use the **security overview** to see the whole queue.
- **Remediation SLAs:** The org sets **time-to-fix targets** by severity (e.g., *Critical = 7 days, High = 30 days, Medium = 90 days*). These numbers are **org-defined**, not a GitHub constant — don't memorize a specific value as if GitHub mandates it.
- **Dismiss vs fix:** **Fixing** removes the vulnerability (merge the Dependabot PR, rotate the leaked secret, patch the code). **Dismissing** closes the alert *without* a code change and **requires a reason** — e.g., *"false positive," "used in tests," "won't fix / risk accepted," "no bandwidth."* Dismissals are auditable. Secret-scanning alerts should generally be **revoked/rotated**, not merely dismissed.
- **Ownership & automation:** Assign owners (CODEOWNERS, security team), require reviews via **rulesets**, and route alerts to ticketing (e.g., stream to a SIEM / open Jira).

**Example — org-defined SLA matrix:**

| Severity | Acknowledge | Remediate (SLA) | Default action |
|---|---|---|---|
| Critical | < 24 h | **7 days** | Patch/rotate immediately; hotfix if exploited |
| High | < 48 h | **30 days** | Schedule fix this sprint |
| Medium | < 1 wk | **90 days** | Backlog with due date |
| Low | best effort | next cycle | Batch with hygiene updates |

> [!NOTE]
> Those numbers are an *example policy*. The exam-correct point is **"the org defines the SLAs and enforces triage,"** not any specific number.

**Analogy:** A hospital **emergency-room triage protocol**. Incoming alerts are patients; the nurse (triage) sorts by how critical and how *real* the condition is. **SLAs** are the hospital's own "a heart attack must be seen in X minutes" targets. **Fixing** is treating the patient; **dismissing with a reason** is formally discharging someone with a documented "not actually sick / risk accepted" note in the chart — never just ignoring them off the books.

---

## 3. Manage API access and integrations

### 3.1 Personal access tokens: classic vs fine-grained

**Plain English:** PATs let a **user** authenticate to the GitHub API/Git over HTTPS as themselves.

- **Classic PATs** use broad, coarse **scopes** (e.g., `repo` grants access to **all** repos the user can touch — public *and* private). They can be set to **never expire**. Powerful but blunt.
- **Fine-grained PATs** (the recommended default) are scoped to **specific repositories** (or all repos in **one** org), use granular **permissions** (e.g., *Contents: Read, Issues: Write, Pull requests: Read*) instead of scopes, **must have an expiration**, and are subject to **organization approval policies** — an org owner can require that fine-grained PATs targeting the org be **approved** before they work, and can **restrict or block** PAT access entirely.

> [!IMPORTANT]
> A **fine-grained PAT belongs to a single resource owner** (one user or one org). To hit two orgs you need two tokens. Classic PATs span everything the user can access. Org PAT approval/restriction policies apply most fully to **fine-grained** PATs.

#### Comparison table: classic PAT vs fine-grained PAT

| Aspect | **Classic PAT** | **Fine-grained PAT** |
|---|---|---|
| Access model | Broad **scopes** (`repo`, `admin:org`, `workflow`…) | Granular **permissions** (Contents, Issues, PRs… each Read/Write/None) |
| Repo targeting | **All** repos the user can access | **Selected** repos, or all repos in **one** org/account |
| Resource owner | The user (spans many orgs) | **A single** user or org |
| Expiration | Optional — **can be "no expiration"** | **Required** (must expire) |
| Org control | Limited (can disallow access broadly) | **Approval policy** + allow/restrict/block per org |
| Org approval before use | No (works immediately) | **Yes** — org can require admin approval |
| SSO/SAML | Must be authorized for SSO orgs | Honors org policy directly |
| Recommended? | Legacy / niche | **Yes — least privilege default** |

**Example — least-privilege fine-grained token:**

```text
Resource owner:  ACME (org)
Repository access: Only select repositories → [ "acme/billing-service" ]
Repository permissions:
   Contents:        Read-only
   Pull requests:   Read and write
   Metadata:        Read-only (mandatory)
Expiration:        30 days
→ If ACME requires approval, token is "pending" until an owner approves.
```

```bash
# Using a PAT with the CLI / git
export GH_TOKEN="github_pat_11ABC..."     # fine-grained tokens start with github_pat_
gh api /repos/acme/billing-service/issues
# classic tokens start with ghp_
```

**Analogy:** A classic PAT is a **building master key** — one key opens *every* door you're allowed into, and it **never expires**. A fine-grained PAT is a **programmed hotel keycard** — coded for *room 412 only*, *expires Friday*, and the **front desk (org owner) must activate it** before it works. You'd never hand a contractor the master key when a one-room, time-limited card does the job.

---

### 3.2 Rate limits for PATs and GitHub Apps

**Plain English:** GitHub throttles API usage to protect the platform.

- **Authenticated REST API (user/PAT):** **5,000 requests per hour** per user (this is the well-known baseline). Unauthenticated requests are far lower (**60/hour** by IP).
- **GitHub Apps:** Get **higher** limits than a single user. A GitHub App installation's limit **scales with the size of the org/installation** (more repos and users → a higher ceiling), and **GitHub Enterprise Cloud** installations get a substantially higher floor (commonly cited around **15,000 requests/hour** for Apps installed on an enterprise org). This is a major reason to build integrations as **Apps, not PATs**.
- **Primary vs secondary rate limits:**
  - **Primary** = the per-hour request *quota* (the 5,000/15,000 numbers). When exhausted you get **HTTP 403/429** with `X-RateLimit-Remaining: 0` and `X-RateLimit-Reset`.
  - **Secondary** = anti-abuse limits on *bursty/concurrent* behavior — too many requests too fast, too many concurrent requests, too much content creation in a short window — even if you haven't hit the hourly primary quota. Honor `Retry-After` and back off.
- **GraphQL** uses a separate **point-based** budget (also commonly **5,000 points/hour** for users).

**Example — reading the headers:**

```bash
$ gh api -i /rate_limit | grep -i ratelimit
X-RateLimit-Limit: 5000
X-RateLimit-Remaining: 4983
X-RateLimit-Reset: 1750972800     # epoch when the window resets

# Best practice: check Remaining, sleep until Reset; on secondary limits honor Retry-After
```

**Analogy:** A theme park.
- **Primary limit** = your **daily ride wristband** good for 5,000 rides (15,000 if you're a VIP tour group = a GitHub App). Run out, and you wait for the park to reset at the top of the hour.
- **Secondary limit** = the **"no cutting / no sprinting / no shoving"** crowd-control rules — break those by hammering the gate even *with* rides left on your band, and a guard makes you **wait a few minutes** (`Retry-After`) regardless of your remaining count.
- **Apps get a bigger band** because they're the official tour operator serving a whole group, not one walk-in guest.

---

### 3.3 GitHub Apps vs OAuth Apps

**Plain English:** Both let external software integrate with GitHub, but they differ fundamentally:

- **OAuth App:** Authorizes **as the user** who granted it. It inherits **that user's** access and uses broad OAuth **scopes** (like classic PAT scopes). Tokens are typically **long-lived**. If the user has access to 200 repos, the OAuth App effectively does too. Coarse and user-bound.
- **GitHub App:** Is its **own identity** — you **install** it on specific orgs/repos and grant **fine-grained permissions** to *just those repos*. It authenticates with **short-lived installation tokens** (expire ~1 hour) and can also act **on behalf of a user** when needed. It has **higher rate limits**, can subscribe to **webhooks**, and least-privilege is the default. **GitHub strongly recommends GitHub Apps** for new integrations.
- **Acts-as:** OAuth App = **acts as the user** (always). GitHub App = **acts as the app/installation** (default), or **acts as a user** (optional user-to-server flow) — your choice per action.

#### Comparison table: GitHub App vs OAuth App

| Aspect | **GitHub App** | **OAuth App** |
|---|---|---|
| Identity | **Own identity** (a bot/installation) | Acts **as the authorizing user** |
| Granted via | **Installation** on selected orgs/repos | **User authorization** (consent screen) |
| Permission model | **Fine-grained**, per-repo | Broad **scopes** (user-wide) |
| Token lifetime | **Short-lived** installation tokens (~1 h) + JWT | Typically **long-lived** user token |
| Repo scope | Only repos it's **installed on** | **All** repos the user can access |
| Rate limits | **Higher**, scales with installation size | Same as the **user** (5,000/h) |
| Acts as | **App** (or optionally a user) | **User** only |
| Webhooks | First-class, per-app | More limited |
| Admin control | **Install approval** (owner approves) | **OAuth App access restrictions** (allow-list) |
| GitHub recommends | **✅ Yes** (preferred) | Legacy / specific cases |

**Example — GitHub App auth flow (server-to-server):**

```text
1. App authenticates with a signed JWT (using its private key)  → proves "I am app #12345"
2. App requests an INSTALLATION access token for org ACME
3. GitHub returns a token valid ~1 hour, scoped to ACME's granted repos & permissions
4. App calls the API with that short-lived token; it expires automatically
```

```bash
# Conceptual: mint an installation token (after building the JWT)
curl -s -X POST \
  -H "Authorization: Bearer $APP_JWT" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/app/installations/$INSTALLATION_ID/access_tokens
# → { "token": "ghs_...", "expires_at": "2026-06-26T19:00:00Z", "permissions": {...} }
# Installation tokens start with ghs_
```

**Analogy:** An **OAuth App is a colleague borrowing your badge** — security sees *you* walking around, and it opens every door *you* can open; if you can reach 200 rooms, so can they, indefinitely. A **GitHub App is a bonded vendor with its own company badge** — facilities (the org owner) **installs** it and grants access to *only* the three rooms it services, with a badge that **auto-expires every hour** and re-issues itself. The vendor acts as *itself*, not impersonating you — and when it truly needs to act for you, it briefly "acts on your behalf" with explicit consent.

---

### 3.4 Approve or deny app usage based on policy

**Plain English:** Org owners gate which third-party software touches org data:

- **OAuth App access restrictions:** When enabled for an org, **previously unapproved OAuth Apps are blocked** from accessing org resources until an **owner approves** them. Members can **request** approval; owners **allow/deny**. This is the org's allow-list for OAuth Apps.
- **GitHub App installation approval:** Members can **request** to install a GitHub App; an **owner must approve** the installation (and review the permissions it requests) before it's active. Owners can also **uninstall**/suspend an app and review its granted permissions/repos.
- **PAT policies (tie-in):** Owners can **require approval for fine-grained PATs**, **restrict** what PATs can do, and **block** classic PATs from accessing the org — the same "approve or deny" governance applied to user tokens.
- **Actions/Marketplace:** Similarly, owners can restrict which **GitHub Actions** are allowed (GitHub-owned, verified creators, or an explicit allow-list).

**Example — request/approve flow:**

```text
Member: tries to install "AcmeBot" GitHub App on 4 repos
   → GitHub: "Installation request sent to org owners for approval"
Owner:  Org → Settings → Third-party Access → GitHub Apps / OAuth app policy
   → reviews requested permissions (Contents: write, PRs: write)
   → [ Approve ]  or  [ Deny ]
```

```text
Org → Settings → Third-party Access:
  • OAuth app policy:            ☑ "Require approval of OAuth Apps"  (restrictions ON)
  • Personal access tokens:
        Fine-grained: ☑ Require approval
        Classic:      ☑ Restrict access (block classic PATs)
  • GitHub Apps:                 owners approve installation requests
```

**Analogy:** The **vendor-management / security desk** at the airport. No contractor, OAuth visitor, or borrowed badge gets past the perimeter until the **security manager reviews exactly which doors they're asking for and signs off**. "Require approval" flips the desk from *"default-allow, ask forgiveness"* to *"default-deny, must get permission first"* — the safer posture the exam favors.

---

## GHAS features at a glance — what each protects against

| GHAS feature | What it inspects | Protects against | Reactive / Proactive |
|---|---|---|---|
| **Dependency graph** | Manifests/lockfiles | (Foundation) maps direct + transitive deps | — |
| **Dependabot alerts** | Deps vs GitHub Advisory DB | Using **known-vulnerable** dependencies | Reactive (notify) |
| **Dependabot security updates** | Vulnerable deps | Same — **auto-PR to patched version** | Reactive (fix) |
| **Dependabot version updates** | All deps on a schedule | **Drift / outdated** dependencies | Proactive (hygiene) |
| **Secret scanning** | Repo content + history | **Committed credentials** (keys/tokens) | Reactive (detect) |
| **Push protection** | Outgoing pushes | Secrets **entering** the repo at all | Proactive (prevent) |
| **Code scanning (CodeQL)** | Your source code (SAST) | **Vulns in your own code** (SQLi, XSS…) | Proactive (analyze) |
| **Security advisories / PVR** | Reported vulns in your project | **Public disclosure before a fix** exists | Process |

---

## ⚠️ Exam traps

> [!CAUTION]
> These are the distinctions the exam loves to blur. Burn them in.

- **Rulesets vs classic branch protection:** Rulesets **layer** (multiple apply and combine, most-restrictive wins), can be created **at the org level targeting many repos**, protect **branches AND tags AND pushes**, and have **Active / Evaluate / Disabled** enforcement plus **bypass lists**. Classic protection is **per-repo, single-rule, branches-only, on/off**. They **coexist** — enabling rulesets does **not** remove classic rules, and both are enforced together. **Evaluate** = dry-run logging without blocking.

- **Dependabot alerts vs security updates vs version updates:** *Alerts* = notify only. *Security updates* = **reactive** PRs triggered by a vulnerability, **no config file needed**. *Version updates* = **proactive** scheduled PRs, **require `.github/dependabot.yml`**. If a question says "keep dependencies current on a schedule," it's **version updates**; "auto-fix a vulnerability," it's **security updates."

- **Secret scanning vs push protection:** Secret scanning **detects secrets already in the repo/history** (after the fact). Push protection **blocks the push** so the secret never lands (before the fact). Push protection is *built on* secret scanning. Bypasses to push protection are **logged/alerted**.

- **Dependency graph is the prerequisite:** Dependabot alerts/updates need the **dependency graph enabled**. If alerts "aren't appearing," check the graph first.

- **GitHub App vs OAuth App:** OAuth App **acts as the user** with broad **scopes** and **long-lived** tokens across **all** the user's repos. GitHub App is its **own identity**, **installed** on selected repos with **fine-grained** permissions, **short-lived (~1h) installation tokens**, and **higher rate limits**. **GitHub recommends Apps.** If the scenario stresses least privilege, per-repo scope, expiring tokens, or higher rate limits → **GitHub App**.

- **Fine-grained vs classic PAT:** Classic = broad **scopes**, can **never expire**, spans all the user's repos. Fine-grained = granular **permissions**, **must expire**, scoped to **one** owner/selected repos, subject to **org approval policy**. "Require approval before a token works" applies to **fine-grained** PATs.

- **Audit log retention:** The built-in UI/API search only covers a **limited window** (commonly up to ~**180 days / 6 months**; **Git events retained for a shorter period**). For **long-term/compliance retention**, you **must** use **audit log streaming** to S3 / Azure Blob / GCS / Splunk / Datadog. Don't pick "increase the UI retention setting" — that's not the mechanism.

- **Primary vs secondary rate limits:** Primary = the **per-hour quota** (5,000 user / ~15,000 enterprise App). Secondary = **anti-abuse** limits on **bursts/concurrency**, which can trip **even with quota remaining** — honor `Retry-After`.

- **"No policy" at enterprise level** means *"organizations decide,"* **not** *"disabled."* Higher tiers can only make things **more** restrictive, never less.

- **Dismiss vs fix:** Dismissing closes an alert **without** changing code and **requires a reason**; it does **not** remediate. Secrets should be **rotated/revoked**, not just dismissed.

---

## 📋 Quick reference

| Topic | Key fact to remember |
|---|---|
| Policy cascade | **Enterprise → Org → Repo**; lower tiers can only tighten; "No policy" = orgs decide |
| Ruleset enforcement | **Active** (enforce) / **Evaluate** (dry-run log) / **Disabled** |
| Ruleset superpower | Org-level, **multi-repo**, layers, targets **branches + tags + pushes**, bypass lists |
| Branch protection | Per-repo, single-rule, branches only — **coexists** with rulesets |
| Security overview | Single-pane dashboard for Dependabot + secret + code scanning across repos |
| Push protection | **Blocks the push**; bypass needs a reason and is **audited** |
| Default `GITHUB_TOKEN` | Set org default to **read-only**; disallow PR create/approve |
| IP allow list | Restrict org access by **CIDR**; can include GitHub App IPs |
| Audit log search window | Up to **~180 days**; **Git events shorter** |
| Long-term audit retention | **Audit log streaming** → S3 / Azure Blob / Event Hubs / GCS / Splunk / Datadog |
| Dependency graph | **Prerequisite** for all Dependabot features |
| Dependabot security updates | **Reactive**, vuln-driven, **no config file** required |
| Dependabot version updates | **Proactive**, scheduled, **requires `dependabot.yml`** |
| Secret scanning | Detects committed secrets; **validity checks**; custom patterns w/ dry run |
| Code scanning | CodeQL SAST; **default** (managed) vs **advanced** (`codeql.yml`); **SARIF** upload for 3rd-party |
| Security advisory | Private fix via **PVR** + temp private fork → **GHSA/CVE** → feeds Advisory DB |
| Classic PAT | Broad **scopes**, **can never expire**, all user repos |
| Fine-grained PAT | Granular **permissions**, **must expire**, one owner, **org approval** |
| Auth REST rate limit | **5,000 req/hour** (user/PAT); **60/hour** unauthenticated |
| GitHub App rate limit | **Higher**, scales w/ install; **~15,000/hr** on enterprise orgs |
| Primary vs secondary | Primary = hourly quota; Secondary = burst/concurrency anti-abuse (`Retry-After`) |
| GitHub App | Own identity, **installed**, fine-grained, **short-lived (~1h)** tokens, **recommended** |
| OAuth App | Acts **as user**, broad scopes, **long-lived**, all user repos |
| App governance | **OAuth app access restrictions** + **GitHub App install approval** (default-deny) |
| Token prefixes | `ghp_` classic PAT · `github_pat_` fine-grained · `ghs_` installation · `gho_` OAuth |

---

## 🧠 Self-check

Test yourself before peeking. Aim to explain *why*, not just the answer.

**Q1.** An enterprise sets repository visibility to "private only." Org A's owner wants to allow public repos. Can they? Why or why not?

<details><summary>Answer</summary>
<b>No.</b> Policies cascade <b>Enterprise → Org → Repo</b>, and lower tiers can only be <i>more</i> restrictive, never less. The enterprise floor (private only) binds Org A. To allow public repos, the change must be made at the enterprise level (or set to "No policy" so orgs decide).
</details>

**Q2.** You want to require 2 PR reviews and signed commits on the default branch of **all 80 repos** in an org, see the impact before enforcing, and let the release team bypass. What do you use and in what mode first?

<details><summary>Answer</summary>
An <b>organization-level ruleset</b> targeting <code>~ALL</code> repos and <code>~DEFAULT_BRANCH</code>, with <code>pull_request</code> (2 approvals) and <code>required_signatures</code> rules, plus a <b>bypass list</b> for the release team. Set enforcement to <b>Evaluate</b> first (dry-run logging) to preview impact, then switch to <b>Active</b>. Classic branch protection couldn't target all repos at once.
</details>

**Q3.** A developer's `git push` is rejected because it contains an AWS key. Which feature did that, and how does it differ from secret scanning?

<details><summary>Answer</summary>
<b>Push protection</b> — it blocks the secret <b>before</b> it enters the repo (proactive/preventive). Plain <b>secret scanning</b> detects secrets <b>already committed</b> (reactive). Push protection is built on top of secret scanning; the developer can bypass with a reason, which is <b>audited/alerted</b>.
</details>

**Q4.** What's the difference between Dependabot **security updates** and **version updates**, and which one needs a config file?

<details><summary>Answer</summary>
<b>Security updates</b> = <b>reactive</b> PRs that bump a dependency to a patched version in response to a vulnerability alert; <b>no config file required</b>. <b>Version updates</b> = <b>proactive</b> PRs on a <b>schedule</b> you define; they <b>require <code>.github/dependabot.yml</code></b>. Both rely on the dependency graph being enabled.
</details>

**Q5.** Compliance requires keeping audit events for 7 years. The org owner says "just increase the audit log retention in settings." Correct?

<details><summary>Answer</summary>
<b>No.</b> The built-in UI/API audit log search only covers a limited window (commonly up to ~<b>180 days</b>, with <b>Git events even shorter</b>). For multi-year/compliance retention you must enable <b>audit log streaming</b> to external storage you control — <b>S3, Azure Blob/Event Hubs, GCS, Splunk, or Datadog</b> — which retains indefinitely.
</details>

**Q6.** A third-party CI bot needs API access to exactly 3 of an org's 50 repos, with tokens that expire automatically and a high rate limit. PAT, OAuth App, or GitHub App?

<details><summary>Answer</summary>
A <b>GitHub App</b>. It's installed on only those 3 repos with fine-grained permissions, uses <b>short-lived (~1h) installation tokens</b>, and gets <b>higher rate limits</b> that scale with the installation. A classic PAT/OAuth App would act as a user with broad access to all 50 repos and long-lived tokens.
</details>

**Q7.** Name three differences between a **classic PAT** and a **fine-grained PAT**.

<details><summary>Answer</summary>
Any three: (1) classic uses broad <b>scopes</b> vs fine-grained <b>per-permission</b> grants; (2) classic <b>can never expire</b> vs fine-grained <b>must expire</b>; (3) classic spans <b>all the user's repos</b> vs fine-grained is scoped to <b>one owner / selected repos</b>; (4) fine-grained is subject to <b>org approval policy</b> before it works.
</details>

**Q8.** Your app has 4,900 of 5,000 hourly requests remaining but starts getting throttled after firing hundreds of requests in a few seconds. Which limit, and what do you do?

<details><summary>Answer</summary>
The <b>secondary rate limit</b> (anti-abuse on bursts/concurrency), which can trip even with primary quota remaining. <b>Back off</b> and honor the <code>Retry-After</code> header; reduce concurrency and add delays. The <b>primary</b> limit (the 5,000/hour quota) is separate.
</details>

**Q9.** A member wants to install a GitHub App and authorize an OAuth App, but both are blocked pending approval. What two org settings cause this, and what posture do they represent?

<details><summary>Answer</summary>
<b>GitHub App installation approval</b> (owners must approve installs/permissions) and <b>OAuth App access restrictions</b> (unapproved OAuth Apps are blocked until an owner allows them). Together they implement a <b>default-deny</b> third-party access posture — the safer stance.
</details>

**Q10.** A maintainer discovers a critical auth-bypass bug. They want to build and review the fix privately, then publish with a tracking ID that downstream users' Dependabot will pick up. Walk the path.

<details><summary>Answer</summary>
Use a <b>repository security advisory</b>: draft it privately, optionally create a <b>temporary private fork</b> to develop/review the fix without public exposure, score severity, then <b>request a CVE</b> (GitHub is a CNA) so it gets a <b>GHSA + CVE ID</b>. Release the patch and <b>publish</b> the advisory — it enters the <b>GitHub Advisory Database</b>, which <b>Dependabot alerts</b> consume to warn downstream users. <b>Private vulnerability reporting (PVR)</b> is the inbound channel researchers use to report such bugs privately.
</details>

---

> **Final tip:** When a question feels ambiguous, choose the answer that is **least privilege, most auditable, and most centrally enforced** — that's almost always the GitHub-recommended (and exam-correct) posture. Master the *boundaries* in the Exam traps section and you'll bank the ~27% this domain is worth.
