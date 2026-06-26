# Domain 4 — Manage GitHub Actions (20–25%)

> **GH-100 · GitHub Enterprise Administrator** — study notes for the second-largest exam domain. Focus is on *administering* Actions at scale (policies, runners, secrets), not on writing pipelines as a developer.

## Why this domain matters

GitHub Actions is GitHub's built-in CI/CD and automation engine. As an **enterprise administrator**, you are not the person writing every workflow — you are the person who decides **who is allowed to run what, on which machines, with which secrets, under which policies**. Mistakes here are expensive in two directions: too restrictive and developers can't ship; too permissive and you've handed untrusted code access to your network, your cloud accounts, and your long-lived credentials. The exam tests whether you can configure Actions so teams move fast *and* the blast radius stays contained. Expect questions on policy inheritance (enterprise → org → repo), the GitHub-hosted vs self-hosted tradeoff, runner groups, `GITHUB_TOKEN` permissions, secret precedence, and OIDC. Roughly a fifth to a quarter of the exam lives here, so it is worth over-preparing.

## Big-picture analogy

Think of GitHub Actions as a **factory with assembly lines**.

- A **workflow** is the blueprint for one assembly line — what triggers it to start and what stations it has.
- **Jobs** are the work cells; **steps** are the individual operations at each cell.
- **Runners** are the **workers** on the floor. You can use **GitHub-hosted temps** (show up clean, do the job, vanish — you pay per shift, never maintain them) or **self-hosted full-time staff** (your own employees on your own premises — full control and access to your private tooling, but you feed, secure, and maintain them).
- **Runner groups** are **departments** that decide which lines (orgs/repos) a pool of workers is allowed to staff.
- **Secrets** are the **keys to the safes** — handed out on a need-to-know basis, with the local branch manager's keys overriding the corporate master set.
- **Policies** are the **factory rulebook** posted by head office (enterprise), refined by each plant manager (org), and applied on each line (repo) — a line can be stricter than head office but never looser.
- **OIDC** is a **temporary visitor badge**: instead of giving a contractor a permanent master key to keep in their pocket, the front desk issues a badge that works for ten minutes and expires on its own.

---

## Objective 1 — Configure workflows and reusable components

### 1.1 Workflow anatomy (admin refresher)

A workflow is a YAML file in `.github/workflows/` in a repository. Its moving parts:

- **Events / triggers** (`on:`) — what starts the run: `push`, `pull_request`, `schedule` (cron), `workflow_dispatch` (manual), `workflow_call` (called by another workflow), `repository_dispatch` (external API).
- **Jobs** (`jobs:`) — units that run in parallel by default; sequence them with `needs:`. Each job runs on a fresh runner.
- **Steps** (`steps:`) — ordered actions or shell commands inside a job; they share the job's filesystem and runner.
- **Runners** (`runs-on:`) — the machine that executes the job, selected by label (e.g. `ubuntu-latest`, or a self-hosted label).

```yaml
name: CI
on:
  push:
    branches: [ main ]
  pull_request:
jobs:
  build:
    runs-on: ubuntu-latest          # which worker
    steps:
      - uses: actions/checkout@v4    # a reusable action
      - run: npm ci && npm test      # a shell step
```

**Example:** A push to `main` fires the `push` event, which queues the `build` job onto an `ubuntu-latest` GitHub-hosted runner; the runner checks out code and runs the tests, then is destroyed.

**Analogy:** The `on:` block is the **doorbell wiring** — it decides which button (push, PR, schedule, manual) makes the line start moving; jobs are the **work cells** and steps are the **operations** performed in each.

> As an admin you rarely author these, but you must recognize the structure to reason about *where* policy and secrets apply (repo file → job → step) and *which* runner a job will land on.

### 1.2 Reusing actions and workflows across the enterprise

Duplication is the enemy of consistency. GitHub gives you several distinct mechanisms — knowing **which one to recommend** is a classic exam point.

#### Reusable workflows (`workflow_call`)

A whole workflow file that other workflows **call** as a single unit. The called workflow defines `on: workflow_call` with typed `inputs`, `secrets`, and `outputs`. The caller references it with `uses:` at the **job** level.

```yaml
# .github/workflows/reusable-deploy.yml  (the callee)
on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
    secrets:
      deploy_token:
        required: true
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying to ${{ inputs.environment }}"
```

```yaml
# .github/workflows/app.yml  (the caller)
jobs:
  call-deploy:
    uses: my-org/ci-workflows/.github/workflows/reusable-deploy.yml@v2
    with:
      environment: production
    secrets:
      deploy_token: ${{ secrets.DEPLOY_TOKEN }}
```

- Called at **job level** (`uses:` directly under a job, not under steps).
- Can be stored in a **separate repo** and shared org-wide (the repo must allow access from the calling org/repos under Actions settings).
- Pin to a ref (`@v2`, a SHA, or a branch) for stability/security.
- Secrets are passed explicitly or with `secrets: inherit`.

**Example:** Twenty microservice repos each have a 6-line caller that invokes the org's `reusable-deploy.yml@v2`. When the deploy logic changes, you update one file and bump the tag — all 20 repos pick it up.

**Analogy:** A reusable workflow is a **franchise operations manual** — each store runs the *entire* documented procedure end-to-end; corporate edits the manual once and every franchise follows the new version.

#### Composite actions

An **action** (not a workflow) that bundles multiple steps into one reusable step, defined in an `action.yml` with `runs.using: "composite"`. Referenced inside a job's `steps:` with `uses:`.

```yaml
# my-org/setup-action/action.yml
name: 'Setup Toolchain'
inputs:
  node-version:
    default: '20'
runs:
  using: "composite"
  steps:
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
    - run: npm ci
      shell: bash        # shell is REQUIRED for run steps in composite actions
```

**Example:** Every repo's "install dependencies and set up Node" boilerplate (5+ steps) collapses to a single `- uses: my-org/setup-action@v1` line in each job.

**Analogy:** A composite action is a **pre-assembled tool kit** you drop into one station on the line — it does several operations, but it is *one item on a step list*, not a whole separate line.

> **Reusable workflow vs composite action — the #1 exam trap:**
> - Reusable workflow = called at **job** level, can define its own `runs-on`/jobs/environments, can use secrets/environments. Think *whole pipeline*.
> - Composite action = used at **step** level inside someone else's job, runs on *that job's* runner, **cannot** define jobs or use environment protection. Think *one bundled step*.

#### Organization workflow templates (the `.github` repo)

Put **starter workflow files** plus a `.json` metadata file in a `workflow-templates` folder of a public-or-internal repo named **`.github`** in your organization. They then appear under the org's **Actions → New workflow** "By <org>" section so developers can scaffold standardized pipelines.

```
.github/                         (a repo literally named ".github")
└── workflow-templates/
    ├── node-ci.yml              # the starter workflow
    └── node-ci.properties.json  # name, description, iconName, categories
```

**Example:** Platform team ships a `node-ci.yml` template; new repos click "New workflow → Node CI by Contoso" and get the blessed pipeline pre-filled instead of copy-pasting from a wiki.

**Analogy:** Workflow templates are the **"new document" gallery** — pre-formatted starting documents people pick from, then edit. (Contrast with reusable workflows, which are *called live* and stay centrally owned.)

#### Required workflows

Configured at the **organization** level, a **required workflow** must run and **pass** on pull requests in the targeted repositories before merge — the org enforces it, the repo can't skip it. (GitHub has been consolidating this capability into **repository rulesets**, but the exam-relevant idea is org-enforced, mandatory CI.)

**Example:** Security mandates a license-scan workflow on every repo. Marked as a required workflow at org level, it appears as a required status check no developer can bypass.

**Analogy:** A required workflow is the **mandatory safety inspection** every product must pass before leaving the factory — not optional, not skippable by the line operator.

#### Sharing actions from an internal repository

Custom actions live in their own repos. To share inside the enterprise without going public, host the action in an **internal** repository and enable **"Accessible from repositories in the organization/enterprise"** under that repo's **Settings → Actions → General**. Other repos then `uses: my-org/my-action@v1`.

**Example:** You build `contoso/deploy-action`, mark it internal-accessible, and 50 repos reference it without publishing to the public Marketplace.

**Analogy:** An internal action repo is the **company's private parts catalog** — any in-house line can order from it, but outside vendors can't see it.

### 1.3 Apply organizational policies for GitHub Actions

Policies set **what kinds of Actions code may run** and **how much power it gets by default**. They flow **enterprise → organization → repository**, and each lower level may be *more* restrictive but not *less*.

#### Allow / disallow Actions and choose which actions are permitted

Under **Settings → Actions → General → Policies**, pick one of:

- **Disable** Actions entirely.
- **Allow enterprise/org actions only** (only actions authored inside your tenant).
- **Allow all actions and reusable workflows** (open).
- **Allow `<owner>` actions and reusable workflows, and select non-`<owner>` actions** — an **allow list**.

The allow list supports:
- **Allow actions created by GitHub** (the `actions/*` and `github/*` first-party actions).
- **Allow Marketplace actions by verified creators** (the verified badge).
- **Specific patterns**: `actions/*`, `aws-actions/*`, or a precise `monalisa/my-action@v1`.

```text
# Example allow-list patterns
actions/*
github/*
docker/login-action@*
my-org/*
```

**Example:** Finance org allows `actions/*`, verified-creator actions, and `my-org/*`, but blocks everything else — so an engineer cannot pull a random unvetted `someuser/sketchy-action@main` into a pipeline.

**Analogy:** The allow list is the **approved-vendor list** in procurement: you can buy from these suppliers and no one else, no matter how convenient the off-list option looks.

#### Fork pull request policies

PRs from **forks** (often from outside contributors) are dangerous because they run *attacker-influenced* code. Admin controls:

- **Require approval for first-time / all outside contributors** before fork PR workflows run (default: first-time contributors require approval).
- Whether fork PRs can **read secrets** and whether `GITHUB_TOKEN` gets **write** permissions on fork PRs (default: secrets withheld, token read-only for forks).
- On private repos: whether to **run workflows from fork PRs** at all, and **send write tokens / secrets** to them.

**Example:** A public open-source repo requires manual approval before any fork PR workflow runs, preventing a drive-by PR from executing crypto-mining or exfiltration code on your runners.

**Analogy:** Fork PR approval is the **visitor sign-in desk** — outsiders' "deliveries" (code) sit in the lobby until a staff member vouches for them before they're allowed onto the floor.

#### Default `GITHUB_TOKEN` permissions

Each workflow run gets an automatic, short-lived **`GITHUB_TOKEN`** to act on the repo. Admins set the **default** at enterprise/org/repo:

- **Permissive (read/write)** — legacy default; the token can write to the repo.
- **Restricted (read-only)** — recommended; workflows must opt back into specific scopes.

Workflows can then narrow further with a `permissions:` block:

```yaml
permissions:
  contents: read
  pull-requests: write   # grant only what this job needs
```

Also configurable: whether `GITHUB_TOKEN` (and Actions generally) may **create or approve pull requests**.

**Example:** Org sets the default to **read-only**. A workflow that needs to push a tag explicitly declares `permissions: contents: write`, making the elevated access visible and auditable.

**Analogy:** Default token permissions are **default door access on a new badge** — issue badges that open *nothing* by default and add specific doors on request, rather than handing every new hire a master key.

#### Artifact and log retention

Admins set how long **artifacts** and **workflow run logs** are retained (default 90 days; configurable per repo/org/enterprise, e.g. 1–90 days for public, up to 400 for private/enterprise artifacts). Shorter retention reduces storage cost and data exposure; longer aids audits/debugging.

**Example:** Compliance requires keeping build artifacts 180 days; the admin raises the enterprise artifact retention from the 90-day default accordingly.

**Analogy:** Retention settings are the **shredding schedule** — how long you keep the paperwork before it's automatically destroyed.

---

## Objective 2 — Manage runners

### 2.1 GitHub-hosted vs self-hosted runners

- **GitHub-hosted runners**: VMs GitHub provisions on demand, **fresh per job**, pre-loaded with common tools, then discarded. GitHub patches and maintains them; you pay per-minute (free tier for public repos). Great default; limited custom software, runs in GitHub's network.
- **Self-hosted runners**: machines **you** own (physical, VM, container, cloud). You install the runner agent, maintain the OS/tools, and provide network access (e.g. to on-prem databases). Full control; **you** own security and upkeep.

> ⚠️ **Self-hosted runners are strongly discouraged on PUBLIC repositories.** A fork PR could execute untrusted code on your machine, and because self-hosted runners are **not** guaranteed ephemeral, malware could persist between jobs and reach your internal network. Use ephemeral/GitHub-hosted runners for public repos.

#### Comparison table — GitHub-hosted vs self-hosted

| Dimension | GitHub-hosted | Self-hosted |
|---|---|---|
| **Provisioning** | On-demand, automatic | You provision & register |
| **Lifecycle** | Ephemeral, clean per job | Persistent by default (can be made ephemeral) |
| **Maintenance/patching** | GitHub | **You** |
| **Cost model** | Per-minute (free for public) | Your hardware/cloud cost; no Actions minute charge |
| **Custom software/hardware (GPU, large disk)** | Limited | Full control |
| **Access to private/on-prem networks** | No (except Azure private networking) | Yes |
| **Security responsibility** | GitHub | **You** |
| **Public repo use** | Safe (ephemeral) | **Risky — discouraged** |
| **Scaling** | Automatic | You scale (e.g. ARC autoscaling) |

**Example:** A job needing a licensed proprietary compiler and access to an on-prem Oracle DB must use a **self-hosted** runner; a standard `npm test` job should use a **GitHub-hosted** one.

**Analogy:** GitHub-hosted = **hiring temps through an agency** (turn up trained, leave, no HR overhead). Self-hosted = **full-time employees on your premises** (full control and building access, but you handle payroll, training, and security).

### 2.2 Runner groups

**Runner groups** organize **self-hosted** (and GitHub-hosted, at enterprise level) runners and control **which organizations/repositories** and **which workflows** may use them. They exist at the **organization** or **enterprise** level.

Controls per group:
- Which **organizations** (enterprise group) or **repositories** (org group) can access the runners — all, or a selected list.
- Whether **public repositories** may use the group (off by default for safety).
- Restrict to **selected workflows** (pin a group to specific reusable-workflow refs).

**Example:** A `production-deploy` runner group contains hardened runners with prod network access; it's restricted to the `payments` repo and only to the `deploy.yml@v3` workflow, so no other repo or workflow can land a job on those machines.

**Analogy:** A runner group is a **secured department with a badge reader** — only employees from approved teams (orgs/repos) and only those running an authorized procedure (workflow) can enter and use that department's equipment.

> **Exam point:** Runner-group access lists scope **who may use the runners**, not what the runners can reach. Default deny — new groups don't auto-grant public repos.

### 2.3 IP allow lists and networking

- **IP allow lists** (enterprise/org): restrict which source IPs may access your GitHub resources. If you enable this **and** use GitHub-hosted runners, you must **allow GitHub Actions' runner IP ranges** (or enable the "GitHub Actions" toggle that auto-handles them), or runners get blocked.
- **Azure private networking for GitHub-hosted runners**: lets GitHub-hosted runners join a **virtual network (VNET) in your Azure subscription**, so they can reach **private resources** (e.g. a private database, internal package feed) while *still being GitHub-managed and ephemeral* — the best-of-both option that avoids self-hosting just for network reach.
- **Self-hosted runner networking/proxy**: self-hosted runners need **outbound HTTPS** to GitHub; configure HTTP proxy via `HTTP_PROXY`, `HTTPS_PROXY`, `NO_PROXY` env vars (or a `.env` file). Allow-list GitHub's domains/IPs on egress firewalls.

```bash
# Self-hosted runner behind a corporate proxy
export HTTPS_PROXY=http://proxy.contoso.local:8080
export NO_PROXY=.internal.contoso.com
./run.sh
```

**Example:** Security mandates an enterprise IP allow list. To keep CI working, the admin enables the GitHub Actions option so hosted-runner ranges are permitted; for a workflow that must hit a private Azure SQL instance, they configure **Azure private networking** rather than standing up self-hosted runners.

**Analogy:** IP allow lists are the **building's guest-list at the gate**; Azure private networking is **issuing the temp agency workers a badge to your secure wing** so they can reach internal rooms without you having to hire them permanently.

### 2.4 Monitor and troubleshoot runner performance

- **Runner status**: Idle / Active / Offline, shown in Settings → Actions → Runners. Offline = agent not connected.
- **Queue / wait times**: jobs stuck "Queued" usually mean **no runner with matching labels is available** (label typo, all runners busy, or capacity too low). Monitor wait time as a health metric.
- **Labels**: jobs target runners by label (`runs-on: [self-hosted, linux, gpu]`). A mismatched or misspelled label leaves jobs queued forever.
- **Autoscaling with Actions Runner Controller (ARC)**: the official way to run **self-hosted runners on Kubernetes** that **scale up/down** with demand (scale to zero when idle, burst on load). Uses runner scale sets; ideal for elastic, ephemeral self-hosted capacity.
- **`_diag` logs**: the self-hosted runner writes diagnostic logs to the **`_diag`** folder in the runner install directory (`Runner_*.log`, `Worker_*.log`) — first stop for "runner won't connect / job failed on the agent" issues. Job-level debugging can be increased by setting the `ACTIONS_RUNNER_DEBUG` and `ACTIONS_STEP_DEBUG` secrets/vars to `true`.

```bash
# Where to look when a self-hosted runner misbehaves
ls ./_diag/            # Runner_*.log, Worker_*.log
```

**Example:** Jobs sit Queued for 20 minutes. The admin checks runners: all are Active (busy). They deploy **ARC** so the Kubernetes-backed runner scale set bursts from 3 to 30 pods under load, draining the queue, then scales back to zero overnight.

**Analogy:** Monitoring runners is **managing a kitchen during a rush** — `_diag` logs are the **CCTV** showing what each cook did, queue time is the **ticket backlog**, labels are the **station each order is routed to**, and ARC is **calling in extra cooks automatically when tickets pile up** and sending them home when it's quiet.

---

## Objective 3 — Manage encrypted secrets

Secrets are encrypted values (tokens, keys) exposed to workflows as masked variables. They are encrypted before reaching GitHub and only decrypted in the runner at use time.

### 3.1 Scope and access for secrets

Three scopes, with a clear **precedence** when names collide:

| Scope | Defined at | Visible to | Notes |
|---|---|---|---|
| **Organization secret** | Org settings | All repos, or a **selected** repo list, or private+internal repos | Central management; great for shared creds |
| **Repository secret** | Repo settings | That repo's workflows | **Overrides an org secret of the same name** |
| **Environment secret** | Repo → Environments | Only jobs that target that `environment:` | Gated by **environment protection rules** |

**Precedence:** when the same name exists at multiple scopes, the **most specific wins** — **environment secret > repository secret > organization secret**. (Environment-scoped only applies when the job declares that environment; otherwise repo beats org.)

**Environment protection rules** add gates around environment secrets/deployments: **required reviewers** (manual approval), **wait timer**, and **deployment branch restrictions** (only `main` may deploy to `production`). A job only gets an environment's secrets after its rules are satisfied.

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production          # unlocks env secrets AFTER protection rules pass
    steps:
      - run: ./deploy.sh
        env:
          KEY: ${{ secrets.PROD_API_KEY }}   # environment secret
```

**Example:** `API_KEY` exists as an org secret (shared default) and as a repo secret in `payments` (its own value). Workflows in `payments` get the **repo** value; every other repo gets the **org** value. The `production` environment additionally requires two reviewers before its `PROD_API_KEY` is released.

**Analogy:** Org/repo/environment secrets are **corporate / branch / vault-room keys**. The branch manager's key (repo) overrides the corporate master (org) for that branch; the vault-room key (environment) only works *after* you pass the guard at the vault door (protection rules).

### 3.2 Configuring secrets at org and repo levels

- **Repo secret:** Repo → Settings → Secrets and variables → Actions → New repository secret.
- **Org secret:** Org → Settings → Secrets and variables → Actions → New organization secret, then set **Repository access**: *All repositories*, *Private and internal*, or **Selected repositories** (least privilege — pick only the repos that need it).

```bash
# GitHub CLI
gh secret set DEPLOY_TOKEN --repo my-org/app --body "<value>"

# Org-level secret limited to selected repos
gh secret set SHARED_REGISTRY_TOKEN --org my-org \
  --visibility selected --repos app,api,worker
```

**Example:** A shared container-registry token is created as an **org secret** with **visibility: selected** and attached only to the three repos that publish images — not exposed to the other 60 repos.

**Analogy:** Limiting an org secret to selected repos is **giving the supply-closet key only to the teams that actually use the closet**, instead of copying it for the whole building.

### 3.3 Integrate third-party vaults and use OIDC

**The problem with stored secrets:** long-lived cloud credentials (AWS access keys, Azure client secrets) sit in GitHub indefinitely. If leaked, they work until someone manually rotates them.

**OIDC (OpenID Connect)** fixes this with **keyless, short-lived auth**. The workflow asks GitHub's OIDC provider for a signed **JWT** describing the run (repo, branch, environment). The cloud provider trusts GitHub as an identity provider, validates the token's claims against a configured trust policy, and hands back a **short-lived access token** — **no long-lived secret is ever stored in GitHub**.

Supported targets include **Azure**, **AWS**, **GCP**, and **HashiCorp Vault** (Vault's JWT/OIDC auth method exchanges the GitHub token for a short-lived Vault token, which then fetches the real secret).

```yaml
# OIDC to a cloud provider — NO stored credentials
permissions:
  id-token: write      # REQUIRED: lets the job request the OIDC JWT
  contents: read
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      # AWS example: assumes a role via OIDC, returns temporary creds
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-oidc
          aws-region: us-east-1
      # Azure example (alternative):
      # - uses: azure/login@v2
      #   with:
      #     client-id: ${{ vars.AZURE_CLIENT_ID }}
      #     tenant-id: ${{ vars.AZURE_TENANT_ID }}
      #     subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}
      - run: aws sts get-caller-identity
```

Key points:
- Requires **`permissions: id-token: write`** in the workflow.
- The cloud side configures a **trust policy** scoping which repo/branch/environment may assume the role (e.g. `repo:my-org/app:ref:refs/heads/main`) — tighten this to prevent other repos from assuming the role.
- No `aws_access_key_id`/`client_secret` stored as a GitHub secret.

**Example:** Instead of storing a permanent AWS key, the team configures an IAM role trusting GitHub's OIDC provider, scoped to `my-org/app` on `main`. Each deploy mints credentials valid for ~1 hour that expire automatically; a leaked log line is useless minutes later.

**Analogy:** Stored secrets are a **permanent master key mailed to a contractor** — if it's lost, anyone holding it can get in until you change the locks. OIDC is a **self-expiring visitor badge** issued at the front desk for one specific visit; it dies on its own, so a dropped badge is worthless.

---

## ⚠️ Exam traps

> **Reusable workflow vs composite action.** Reusable workflow = referenced at **job** level (`uses:` under a job), is a *whole pipeline*, can define `runs-on`, jobs, environments, and accept `secrets`. Composite action = referenced at **step** level inside a job, bundles steps, runs on the *caller's* runner, **cannot** define jobs or use environments. If a question mentions calling something at job level with `secrets: inherit`, it's a reusable workflow.

> **Runner group scope.** Runner-group access lists control **which orgs/repos/workflows can use** the runners — they do **not** control what the runner can reach on the network. New groups do **not** grant public-repo access by default.

> **`GITHUB_TOKEN` default permissions.** The hardened recommendation is **read-only by default**, with workflows opting into specific scopes via a `permissions:` block. The legacy permissive default is read/write. Setting it restricted does **not** break workflows that declare what they need.

> **Secret precedence.** Most specific wins: **environment > repository > organization**. A repo secret **overrides** an org secret of the same name. Environment secrets only apply when the job sets `environment:` and only **after** protection rules pass.

> **OIDC over stored secrets.** OIDC issues **short-lived** tokens via a trust relationship — nothing long-lived is stored in GitHub. It requires `id-token: write`. Prefer it for cloud auth; reserve stored secrets for systems that can't do OIDC.

> **Self-hosted runners on public repos.** Discouraged/dangerous: fork PRs run untrusted code, and self-hosted runners aren't guaranteed ephemeral, so malware can persist and pivot into your network. Use ephemeral or GitHub-hosted runners for public repos; require approval for fork PR workflows.

> **Policy inheritance direction.** Enterprise → org → repo. Lower levels can be **more** restrictive, never **less**. If the enterprise disables an action, an org/repo cannot re-enable it.

> **Workflow templates vs reusable workflows.** Templates (in the `.github` repo's `workflow-templates/`) are **starting points copied** into a repo. Reusable workflows are **called live** and stay centrally owned. "Standardize the starting point" → template; "centralize and update once" → reusable workflow.

> **`workflow_call` vs `workflow_dispatch`.** `workflow_call` = invoked by *another workflow* (reusable). `workflow_dispatch` = invoked **manually** from the UI/API. Don't confuse them.

> **Azure private networking.** Lets **GitHub-hosted** (still ephemeral, GitHub-managed) runners reach **private** Azure resources — an alternative to self-hosting *just for network access*.

---

## 📋 Quick reference

| Concept | What it is / key fact |
|---|---|
| **`on: workflow_call`** | Marks a workflow as **reusable**; called via `uses:` at job level |
| **`on: workflow_dispatch`** | **Manual** trigger from UI/API |
| **Composite action** | `runs.using: "composite"` in `action.yml`; step-level reuse; `shell:` required on run steps |
| **Workflow template** | Starter file in `.github` repo's `workflow-templates/` + `.properties.json` |
| **Required workflow** | Org-enforced workflow that must pass before merge (now via rulesets) |
| **Internal action sharing** | Host in internal repo, enable Actions access for org/enterprise |
| **Actions policy** | Enterprise→org→repo; disable / org-only / all / **allow list** |
| **Allow-list patterns** | `actions/*`, verified creators, `owner/repo@ref` |
| **`GITHUB_TOKEN` default** | Set restricted (read-only) recommended; narrow with `permissions:` |
| **Fork PR policy** | Require approval for outside contributors; withhold secrets/write token |
| **Artifact/log retention** | Default 90 days; configurable per repo/org/enterprise |
| **GitHub-hosted runner** | Ephemeral, GitHub-maintained, per-minute, safe for public repos |
| **Self-hosted runner** | You own/maintain; network access; **risky on public repos** |
| **Runner group** | Restrict runners by org/repo and by workflow; org/enterprise level |
| **`_diag` logs** | Self-hosted runner diagnostic logs (`Runner_*.log`, `Worker_*.log`) |
| **ARC** | Actions Runner Controller — autoscaling self-hosted runners on Kubernetes |
| **`ACTIONS_STEP_DEBUG`** | Set `true` for verbose step debug logging |
| **Azure private networking** | GitHub-hosted runners reach private Azure VNET resources |
| **Secret scopes** | Org / repo / environment |
| **Secret precedence** | Environment > repository > organization |
| **Org secret visibility** | All / private+internal / **selected repositories** |
| **Environment protection** | Required reviewers, wait timer, branch restrictions |
| **OIDC** | Keyless, short-lived cloud auth; needs `id-token: write`; no stored long-lived secret |
| **OIDC targets** | Azure, AWS, GCP, HashiCorp Vault |

---

## 🧠 Self-check

**1.** A team wants every microservice repo to run the *exact same* deploy pipeline, owned and updated centrally in one place. Reusable workflow or workflow template — and why?

<details><summary>Answer</summary>
A <strong>reusable workflow</strong> (<code>on: workflow_call</code>). It is called live with <code>uses:</code> at job level, so updating the one central file (and bumping its ref) instantly changes behavior for every caller. A <em>template</em> only seeds a starting file that each repo then owns and edits independently.
</details>

**2.** What is the key difference in *where* a reusable workflow vs a composite action is referenced, and what can a reusable workflow do that a composite action cannot?

<details><summary>Answer</summary>
A reusable workflow is referenced at <strong>job</strong> level; a composite action at <strong>step</strong> level. A reusable workflow can define its own jobs, <code>runs-on</code>, and use <strong>environments</strong> and <code>secrets</code>; a composite action runs on the caller's runner and cannot define jobs or use environment protection.
</details>

**3.** The enterprise enables an IP allow list and developers suddenly see GitHub-hosted jobs failing/blocked. What likely fixed-by-admin step was missed?

<details><summary>Answer</summary>
The IP allow list must <strong>permit GitHub Actions runner IP ranges</strong> (enable the GitHub Actions option that auto-allows hosted-runner addresses). Otherwise hosted runners are blocked.
</details>

**4.** A secret named `API_KEY` exists at the org level (value A) and as a repository secret in repo X (value B). A workflow in repo X with no `environment:` reads `secrets.API_KEY`. Which value does it get?

<details><summary>Answer</summary>
Value <strong>B</strong> — the <strong>repository</strong> secret overrides the organization secret of the same name. Precedence: environment > repository > organization.
</details>

**5.** Why is OIDC preferred over storing a long-lived cloud access key as a GitHub secret, and what permission must the workflow declare?

<details><summary>Answer</summary>
OIDC mints <strong>short-lived</strong> tokens via a trust relationship, so <strong>no long-lived credential is stored</strong> in GitHub (nothing to leak or rotate). The job must declare <code>permissions: id-token: write</code> to request the OIDC JWT.
</details>

**6.** Why are self-hosted runners discouraged on public repositories?

<details><summary>Answer</summary>
Fork pull requests can run <strong>untrusted code</strong>, and self-hosted runners are <strong>not guaranteed ephemeral</strong>, so malicious code can persist between jobs and pivot into your internal network. Use ephemeral/GitHub-hosted runners and require approval for fork PR workflows.
</details>

**7.** Jobs are stuck "Queued" indefinitely on a self-hosted setup even though runners appear online. Give two likely causes and the autoscaling tool for elastic self-hosted capacity.

<details><summary>Answer</summary>
Likely causes: a <strong>label mismatch</strong> (no runner matches the requested labels) or <strong>all matching runners are busy</strong> (insufficient capacity). For elastic, demand-based self-hosted capacity on Kubernetes, use <strong>Actions Runner Controller (ARC)</strong> with runner scale sets.
</details>

**8.** An admin wants a shared registry token available to only 3 of 60 repos, managed in one place. How should they configure it?

<details><summary>Answer</summary>
Create an <strong>organization secret</strong> with <strong>visibility: selected repositories</strong> and attach only those 3 repos — central management with least-privilege exposure.
</details>

**9.** Where do you look first when a self-hosted runner won't connect or a job fails on the agent itself, and how do you get verbose step logging?

<details><summary>Answer</summary>
Check the runner's <strong><code>_diag</code></strong> folder (<code>Runner_*.log</code>, <code>Worker_*.log</code>) in the runner install directory. For verbose per-step logs, set the <code>ACTIONS_STEP_DEBUG</code> (and <code>ACTIONS_RUNNER_DEBUG</code>) secret/variable to <code>true</code>.
</details>

---

> **Study tip:** For every Actions admin question, ask three things in order — *(1) Is the code allowed to run?* (policies/allow lists/fork rules), *(2) Where does it run?* (hosted vs self-hosted, runner groups, networking), *(3) What can it touch?* (`GITHUB_TOKEN` permissions, secret scope/precedence, OIDC vs stored secrets). That mental checklist resolves most exam scenarios.
