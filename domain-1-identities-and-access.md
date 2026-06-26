# Domain 1 — Manage GitHub Identities and Access (15–20%)

> **Exam:** GH-100 GitHub Enterprise Administrator · **Domain weight:** 15–20%
> **Scope:** GitHub Enterprise Cloud (GHEC) and GitHub Enterprise Server (GHES)

---

## Why this domain matters

Identity and access is the *front door* of every GitHub Enterprise deployment. Before anyone writes a line of code, an administrator has to answer two questions: **"Who are you?"** (authentication) and **"What are you allowed to do?"** (authorization). Getting this wrong is how source code leaks, how ex-employees keep access after they've left, and how a contractor accidentally force-pushes to `main`. This domain tests whether you can wire GitHub up to a corporate identity provider, enforce strong authentication, provision and *de*-provision users automatically, and lay out the layered permission model so the right people have exactly the right access — no more, no less. Expect the exam to lean heavily on the subtle distinctions between SAML, SCIM, and team sync, and between the various account types and roles.

---

## Big-picture analogy

Think of GitHub Enterprise as a **secure corporate office building**:

- **Authentication (SAML / 2FA)** is the **security desk in the lobby** — it checks your badge and confirms you are who you claim to be. It does *not* decide which rooms you can enter.
- **Authorization (roles & permissions)** is the **set of keycards** that open specific doors. The lobby let you in; your keycard decides whether you can enter the server room (Admin), the print room (Write), or just the reception area (Read).
- **SCIM** is the **HR system wired to the badge office** — when HR hires someone, a badge is automatically printed; when HR fires someone, the badge is automatically deactivated. No human walks down to the badge office.
- **Team synchronization** is the **building directory** — it maps "everyone in the Marketing department" (an IdP group) to "everyone who can enter the Marketing floor" (a GitHub team), and keeps the two in sync.
- **Managed users (EMU)** are **employees who only exist on the corporate badge system** — they cannot use a personal badge from another building to get in, and their badge only works in *this* building.

Keep this building in your head; almost every exam question maps onto it.

---

## 1. Manage user identities and authentication

### 1.1 Managed users (EMU) vs personal accounts

GitHub supports two fundamentally different account models for an enterprise.

**Personal accounts (standard GHEC):** Users bring their *own* GitHub.com account (e.g. `octocat`). The enterprise *invites* that existing identity into its organizations. The person owns the account, can use it for open-source work, can belong to many organizations and other enterprises, and keeps the account when they leave the company. SAML SSO is layered *on top* to gate access.

**Managed user accounts — Enterprise Managed Users (EMU):** GitHub *provisions* the account on the enterprise's behalf from the identity provider. The username looks like `octocat_acme` (a `_shortcode` suffix is appended). The enterprise — not the user — owns and controls the account. Managed users:
- Can **only** be members of *their* enterprise; they cannot join outside organizations or collaborate publicly the way standard accounts can.
- Can **only** sign in through the configured IdP — there is no separate GitHub.com password.
- Are **provisioned and deprovisioned via SCIM** from the IdP; the user can't create the account themselves.
- Have **limited interaction with the wider GitHub.com community** (e.g. cannot push to public repos outside the enterprise, restricted ability to comment/star outside).

You choose **one model per enterprise** at setup time, and it is effectively a permanent architectural decision (you cannot trivially flip an existing standard GHEC enterprise into EMU).

> **Example**
> Acme Corp wants tight control: every developer account must die the moment HR offboards the person, and developers must not be able to use their work account to star random open-source repos. Acme chooses **EMU**. Alice's account is created as `alice_acme`, she signs in only via Entra ID, and when she's terminated in the HR system, SCIM deactivates `alice_acme` automatically.
> Contrast: Globex uses **standard GHEC**. Their developers already have `@globex`-flavored personal GitHub accounts used for open-source contributions; Globex just enforces SAML so those accounts must authenticate through Okta to access Globex orgs.

> **Analogy**
> A **personal account** is like an employee who owns their own car and uses it both for work *and* personal trips — the company can require a parking permit (SAML) to enter the lot, but the car is theirs forever. An **EMU account** is a **company fleet vehicle**: issued by the company, usable only for company business, and handed back (deactivated) the day employment ends.

---

### 1.2 Configure and enforce SAML SSO and 2FA

**SAML SSO** delegates *authentication* to an external Identity Provider (IdP). GitHub becomes the "Service Provider," and when a user tries to access enterprise resources, GitHub redirects them to the IdP to prove their identity. Where you configure it differs by product:

| Product | Where SAML is configured | Scope |
|---|---|---|
| **GHEC (standard)** | At the **organization** level *or* the **enterprise** level | Per-org, or enterprise-wide cascading to all orgs |
| **GHEC (EMU)** | At the **enterprise** level only | Mandatory — IdP is the *only* way in |
| **GHES** | **Globally**, in the Management Console / appliance settings | Whole appliance |

On GHEC, enforcing SAML at the org removes any member whose identity can't be linked to the IdP. Configuring it at the **enterprise** level pushes the policy down to every org.

**Two-factor authentication (2FA):** A second proof-of-identity (TOTP app, security key, etc.). Enforcement scope:
- **GHEC standard:** enforce 2FA **per organization** (Org settings → Authentication security → "Require two-factor authentication"). Enabling it **removes any member or outside collaborator who doesn't have 2FA enabled** — and they must be re-invited after turning it on. You can also require it enterprise-wide.
- **GHES:** 2FA is a **global** appliance policy; if SAML/CAS/LDAP external auth is used, 2FA may be enforced by the IdP rather than GitHub itself.
- **EMU:** 2FA is generally handled by the IdP, since the IdP is the sole authentication path.

> **Note:** With **SAML alone**, if you remove a user from the IdP they can no longer *sign in*, but their GitHub membership/seat is **not automatically removed** — there's no auto-deprovisioning. That gap is exactly what SCIM closes (see 1.3).

> **Example**
> An admin opens the Acme organization settings, enables "Require two-factor authentication for everyone in the organization," and clicks save. Three outside collaborators who never set up 2FA are **immediately removed** and must be re-invited once they enable it. Separately, the admin configures SAML pointing at `https://login.microsoftonline.com/...` so every member must authenticate through Entra ID.

> **Analogy**
> SAML is **outsourcing the lobby security desk to a trusted third-party guard company** (your IdP) — GitHub trusts whoever that guard waves through. 2FA is **requiring two IDs at the desk** (a badge *and* a fingerprint). Turning on the 2FA requirement is like the guard suddenly enforcing the fingerprint rule and **escorting out anyone already inside who can't provide one**.

---

### 1.3 SCIM and team synchronization

These two are the most-tested pair in the domain because they sound similar but do completely different jobs.

#### SCIM (System for Cross-domain Identity Management)

SCIM is the **provisioning** protocol: it lets the IdP **create, update, and deactivate** GitHub accounts/memberships automatically. The IdP pushes user lifecycle events to GitHub's SCIM endpoint.

- **Requires SAML to be configured first.** SCIM rides on top of an existing SAML trust; you cannot do SCIM provisioning without SAML authentication in place.
- Solves the **deprovisioning gap**: when HR disables a user in the IdP, SCIM deactivates the GitHub account/membership and **frees the license seat** — automatically.
- On **EMU**, SCIM is *mandatory* and is how accounts come into existence at all.
- On **standard GHEC**, SCIM provisioning is supported with specific IdPs (Entra ID, Okta, etc.) at the organization level.

#### Team synchronization (team sync)

Team sync **maps IdP groups to GitHub teams**, so GitHub team membership automatically mirrors the IdP group. Add someone to the "Backend Engineers" group in Entra ID, and they're automatically added to the linked GitHub team (and removed when the group removes them).

- Manages **which team you're in**, *not whether your account exists*.
- Typically available with **Entra ID** and **Okta** on GHEC; on GHES it relies on the configured IdP.
- Team membership confers repository access via the team's repo permissions, so team sync indirectly drives *authorization*.

#### Key differences at a glance

| | **SAML SSO** | **SCIM** | **Team sync** |
|---|---|---|---|
| Job | Authentication (sign-in) | Provisioning / deprovisioning accounts & seats | Mapping IdP **groups → GitHub teams** |
| Prerequisite | The IdP | **SAML must be configured first** | SAML (and often SCIM) |
| Auto-deprovision? | ❌ No (sign-in blocked, seat remains) | ✅ Yes (account deactivated, seat freed) | Removes from *team* only |
| Manages account existence? | No | **Yes** | No |
| Manages team membership? | No | No | **Yes** |

> **Example**
> Acme uses **SAML** (login via Okta) + **SCIM** (Okta provisions/deprovisions accounts) + **team sync** (Okta groups map to GitHub teams).
> 1. New hire Bob is added to Okta → **SCIM** creates his GitHub account.
> 2. Bob is put in the Okta group "Payments-Devs" → **team sync** adds him to the GitHub `payments-devs` team → he inherits Write on the payments repos.
> 3. Bob leaves; HR disables him in Okta → **SCIM** deactivates his GitHub account and frees the seat. Had they used SAML *only*, Bob couldn't log in but would still occupy a seat and remain a listed member.

> **Analogy**
> - **SCIM** = the **HR-to-badge-office wire** that prints and shreds badges automatically.
> - **Team sync** = the **building directory** that says "everyone in Marketing gets the Marketing-floor keycard" and keeps that list current.
> - **SAML** = the **guard at the desk**. The guard can stop you entering, but the guard does **not** cancel your badge or update the directory — that's SCIM's and team sync's job.

---

### 1.4 Choosing and configuring identity providers

GitHub Enterprise integrates with the major enterprise IdPs. Support varies slightly by feature (SAML vs SCIM vs team sync) and by product (GHEC vs GHES, EMU vs standard).

| IdP | SAML SSO | SCIM provisioning | Team sync | Notes |
|---|---|---|---|---|
| **Microsoft Entra ID** (Azure AD) | ✅ | ✅ | ✅ | Broadest support; common with EMU |
| **Okta** | ✅ | ✅ | ✅ | Broadest support; common with EMU |
| **OneLogin** | ✅ | ✅ (varies) | — | SAML widely supported |
| **PingFederate / PingOne** | ✅ | varies | — | Common in large enterprises |
| **Other SAML 2.0 IdPs** | ✅ (generic) | depends | depends | Any compliant SAML 2.0 IdP can do SSO |

**Selection guidance:**
- If you need **full automation** (auto-provision + team sync), pick **Entra ID or Okta** — they have the deepest GitHub integrations, including EMU support.
- For **EMU**, your IdP choice is constrained to those supporting EMU SCIM provisioning (notably **Entra ID** and **Okta**).
- For **GHES**, you may also use **LDAP** or **CAS** for built-in authentication, in addition to SAML.

> **Example**
> A company already standardized on **Microsoft 365 / Entra ID**. The admin registers GitHub as an Enterprise Application in Entra ID, configures SAML (entity ID + ACS URL from GitHub's enterprise settings), enables Entra's **SCIM provisioning** with a secret token from GitHub, and turns on **group claims** so team sync can map Entra groups to GitHub teams. One IdP, all three capabilities.

> **Analogy**
> Choosing an IdP is choosing **which national passport authority your building trusts**. Entra ID and Okta are like passports that come with a **full embassy service** (they'll also handle visas/SCIM and group directories). A generic SAML IdP is a passport that proves identity but offers **no extra embassy services** — you'll handle provisioning yourself.

---

### 1.5 Authentication vs authorization (authN vs authZ)

Two words that look alike and are constantly confused — and the exam knows it.

- **Authentication (authN) = "Who are you?"** Proving identity. Handled by SAML SSO, 2FA, passwords, passkeys, the IdP.
- **Authorization (authZ) = "What are you allowed to do?"** Granting permission. Handled by **roles** — enterprise roles, organization roles, team membership, and repository permission levels.

You can be perfectly *authenticated* (the guard let you in) and still be *unauthorized* to do something (your keycard won't open the server room). The two are independent layers.

**The role hierarchy (broad → narrow):**

```
Enterprise
  └─ Enterprise owners / Billing managers / Enterprise members
       └─ Organization
            └─ Org owners / Members / Moderators / Billing managers
                 └─ Team (membership → inherited repo access)
                      └─ Repository
                           └─ Read · Triage · Write · Maintain · Admin
```

Permissions generally **flow downward and accumulate**: an enterprise owner has sweeping control; a repo-level Read user has the least. A user's *effective* permission on a repo is the **highest** level granted across their direct grants and all their team memberships.

> **Example**
> Dana authenticates successfully via SAML + 2FA (**authN passed**). She tries to delete the `core-api` repository and is denied because she only has **Write**, not **Admin** (**authZ failed**). Same person, two separate gates.

> **Analogy**
> **AuthN** is the bouncer checking your ID at the door. **AuthZ** is the VIP wristband that decides whether you can go backstage. Getting past the bouncer does not get you backstage — different check, different gate.

---

## 2. Manage access and permissions

### 2.1 Configure organization roles and repository roles

#### Repository permission levels

GitHub provides **five built-in repository roles**, increasing in power:

| Role | Can do (cumulative) |
|---|---|
| **Read** | Clone/pull, view, open issues & PRs, comment, discuss |
| **Triage** | Read **+** manage issues/PRs (label, assign, close) — **no write to code** |
| **Write** | Triage **+** push code, manage own PRs, create branches |
| **Maintain** | Write **+** manage repo settings *without* destructive/sensitive admin actions (e.g. edit repo description, manage some settings) — **cannot** delete the repo, change visibility, or manage security-critical settings |
| **Admin** | Full control: manage access, change visibility, delete the repo, manage all settings, rulesets, secrets |

> **Maintain vs Admin** is a favorite exam trap: **Maintain** is for project leads who shouldn't have destructive power. **Admin** can delete the repo, change visibility, and manage collaborator access; **Maintain cannot.**

**Custom repository roles** (GHEC, and GHES): start from a base role (e.g. Write) and **add or remove individual permissions** to create a tailored role like "Write + manage webhooks but not edit settings." Created at the **organization** level and assignable on repos.

#### Organization roles

| Org role | Purpose |
|---|---|
| **Owner** | Full administrative control of the org (settings, members, billing, all repos) |
| **Member** | Default role; access to repos via teams/grants, can create repos if allowed |
| **Moderator** | Member **+** moderation powers (block users, manage interaction limits, minimize comments) — *not* full admin |
| **Billing manager** | Manage billing/payment and seats **only** — no access to repos or org content |

**Custom organization roles** (GHEC): bundle a set of organization-level permissions (and/or repo-level base access) into a named role you can assign to members or teams — e.g. an "Org auditor" role that can read audit data without being a full Owner.

> **Example**
> A team lead, Priya, is given **Maintain** on `web-app` so she can manage labels, topics, and merge settings — but she **cannot** delete the repo or flip it to public. The org's security engineer is given a **custom organization role** "Security-Reader" that grants read access to security alerts org-wide without making him an Owner. A finance person is a **Billing manager** — she sees the invoice but can't open a single repo.

> **Analogy**
> Repository roles are **keycard tiers** in one building wing: Read = visitor pass, Triage = receptionist (can shuffle the mail but not edit the files), Write = staff (can edit files), Maintain = floor manager (runs the floor but can't demolish it), Admin = building superintendent (can knock down walls). Custom roles are **bespoke keycards** cut for one specific job. Org roles are **company-wide titles**: Owner = CEO, Billing manager = accountant, Moderator = HR/conduct officer, Member = regular employee.

---

### 2.2 Define and manage enterprise teams

**Enterprise teams** (GHEC) let you manage groups of members **at the enterprise level**, spanning multiple organizations, rather than re-creating the same team in each org. They're useful for granting consistent access and applying policy across orgs, and can be **connected to your IdP groups** so membership is managed centrally.

- Created and managed in **enterprise settings**, by enterprise owners.
- Membership can be **synced from an IdP group** (via team sync), so the same source-of-truth group drives access across every organization.
- Distinct from **organization teams**, which live inside a single org and map directly to that org's repos.

> **Example**
> Acme has 12 organizations. The security team needs read access to security alerts in *all* of them. Instead of creating a "security" team 12 times, the enterprise owner creates **one enterprise team** "Security," links it to the `sec-team` Entra group, and uses it to grant consistent access enterprise-wide. New security hires added to the Entra group flow through automatically.

> **Analogy**
> An **organization team** is a **department roster for one office**. An **enterprise team** is a **company-wide committee** (e.g. "Global Security Council") whose roster is maintained once at HQ and recognized in every branch office.

---

### 2.3 Audit access and permissions

Administrators must be able to **review who has access to what, and what they did**. GitHub provides several tools:

- **Audit log** (org-level and enterprise-level): an immutable record of events — membership changes, permission changes, repo creation/deletion, SSO sign-ins, policy changes, etc. Searchable by actor, action, and time; on GHEC it can be **streamed** to external SIEM/storage (e.g. Splunk, Azure Event Hub, S3) and queried via API.
- **Member / outside collaborator lists**: see who's a member, who's an outside collaborator, and who hasn't enabled 2FA.
- **Repository access reviews**: per-repo "Manage access" shows every person/team and their permission level and *how* they got it (direct vs via team).
- **People → 2FA / SAML status**: filter members by whether they've enabled 2FA or linked a SAML identity.
- **Enterprise "Compliance"/security overview**: aggregate view of security and access posture.

Good practice: schedule **periodic access reviews**, watch for *outside collaborators* with broad access, and confirm SCIM is correctly deprovisioning departed users (so seats are reclaimed).

> **Example**
> During a quarterly review the admin filters the org's people list by **"2FA disabled,"** finds two stale service accounts, and removes them. She then searches the **audit log** for `action:repo.access` to confirm no unexpected permission escalations occurred, and exports the log to Splunk for the compliance team.

> **Analogy**
> The audit log is the building's **CCTV + door-swipe ledger** — every badge swipe, every door opened, time-stamped and tamper-evident. Access reviews are the **periodic guard walk-through**: "Why does this contractor's badge still open the server room?"

---

### 2.4 Manage settings, policies, rulesets, and roles (the policy cascade)

Enterprise governance flows **top-down through three scopes**: **Enterprise → Organization → Repository.** Higher scopes set guardrails that lower scopes operate within.

**The cardinal rule:** a higher scope can **restrict** a lower scope, but a lower scope **cannot loosen** what a higher scope locked down. If the **enterprise** disables a setting (e.g. "members cannot create public repos"), an org owner **cannot re-enable** it for their org. If the enterprise sets a policy to "**No policy / org admins choose**," then orgs are free to decide.

```
Enterprise policy: "Repository creation = private only"   (LOCKED)
        │  cascades down — orgs cannot loosen
        ▼
Org setting: would like to allow public repos             ❌ blocked
        ▼
Repo: cannot be made public                               ❌ blocked
```

Versus:

```
Enterprise policy: "Repository creation = No policy"      (delegated)
        │
        ▼
Org setting: org owner chooses to allow public            ✅ allowed
```

**Rulesets** are the modern way to protect branches and tags. They define rules like *require pull request reviews*, *require status checks*, *block force-pushes*, *require signed commits*, *restrict who can push/delete*. Key properties:
- Defined at **repository** or **organization** level (and managed within enterprise policy).
- **Multiple rulesets stack / layer** — unlike legacy branch protection rules, rulesets are **additive**; the most restrictive combination applies. An org-level ruleset and a repo-level ruleset both apply.
- Support **bypass lists** (specific roles/teams/apps allowed to bypass), and an **evaluate / "active" mode** so you can test a ruleset before enforcing it.
- Rulesets are the successor to **branch protection rules**; they can coexist, with the most restrictive winning.

**Roles** also cascade: enterprise owners can set policies on what org owners may do (e.g. base permissions, allowed repo visibilities, Actions policy), and org settings constrain what repo admins may do.

> **Example**
> The enterprise sets a policy: **2FA required** and **public repositories disabled** enterprise-wide. The Globex org owner wants to open-source a project and tries to allow public repos — the toggle is **greyed out / disabled** because the enterprise locked it. Separately, the platform team publishes an **org-level ruleset** requiring 2 PR approvals and blocking force-push on any branch named `main`; a repo admin adds a **repo-level ruleset** requiring signed commits. On `core-api/main`, **all** of these apply because rulesets stack.

> **Analogy**
> Think **building codes → company policy → team rules.** City building code (enterprise) says "no open flames in the building." A department head (org) can add "no candles on desks either" — making it *stricter* — but **cannot** say "open flames are fine on our floor." Rulesets are like **multiple posted safety signs on the same door**: if one says "authorized personnel only" and another says "hard hat required," you must satisfy **both** to pass.

> ⚠️ The "you can tighten but never loosen a higher-scope policy" rule is one of the **most tested ideas** in this domain. When in doubt, the answer is: *the more restrictive policy wins, and the higher scope sets the ceiling.*

---

## ⚠️ Exam traps

GitHub loves to test the fine distinctions. Memorize these:

- **SAML vs SCIM.** SAML = authentication (sign-in). SCIM = provisioning/deprovisioning (account & seat lifecycle). **SCIM requires SAML first.** SAML alone does **not** auto-remove a user or free their seat — it only blocks sign-in.
- **SCIM vs team sync.** SCIM manages *whether the account exists*; team sync manages *which team the user is in*. Different jobs.
- **EMU vs standard GHEC.** EMU accounts are enterprise-owned, IdP-only, `_shortcode` usernames, can't join outside orgs, and are SCIM-provisioned. Standard accounts are user-owned personal accounts invited into orgs. You pick one model and it's effectively permanent.
- **Maintain vs Admin (repo).** Maintain = manage the repo without destructive power. Admin = can delete the repo, change visibility, manage access. If a question says "should run the project but must NOT be able to delete it or make it public," the answer is **Maintain**.
- **Triage vs Write.** Triage can manage issues/PRs but **cannot push code**. Write can push code.
- **Org roles confusion.** Billing manager sees billing but **no repos**. Moderator moderates but is **not** an Owner. Owner = full control.
- **Policy cascade direction.** Higher scope (enterprise) can **restrict** lower scopes; lower scopes can **never loosen** a locked higher-scope policy. "No policy" = delegated to the org.
- **Rulesets are additive.** Multiple rulesets (org + repo) **stack**; the most restrictive combination wins. This differs from legacy branch protection.
- **2FA enforcement removes non-compliant users.** Turning on the org 2FA requirement **immediately removes** members/outside collaborators without 2FA — they must be re-invited.
- **Where SAML/2FA is configured.** GHEC standard = org *or* enterprise level. GHES = **global** appliance setting. EMU = enterprise level, IdP mandatory.
- **Effective permission = highest grant.** A user's repo access is the *highest* of their direct grant and all team-inherited grants — permissions don't subtract.

---

## 📋 Quick reference

| Concept | One-line summary | Scope / where |
|---|---|---|
| **Personal account** | User-owned GitHub.com identity invited into orgs | GHEC standard |
| **Managed user (EMU)** | Enterprise-owned, IdP-only account (`user_shortcode`) | GHEC EMU |
| **SAML SSO** | Authentication via external IdP | GHEC org/enterprise; GHES global |
| **2FA** | Second factor; enforcing it removes non-compliant users | Org or enterprise (GHEC); global (GHES) |
| **SCIM** | Auto provision/deprovision accounts & seats (needs SAML) | GHEC (EMU mandatory) |
| **Team sync** | Map IdP groups → GitHub teams | GHEC (Entra/Okta) |
| **AuthN** | "Who are you?" — SAML, 2FA | Identity layer |
| **AuthZ** | "What can you do?" — roles/permissions | Access layer |
| **Repo roles** | Read · Triage · Write · Maintain · Admin | Repository |
| **Custom repo role** | Base role + tweaked permissions | Defined at org |
| **Org roles** | Owner · Member · Moderator · Billing manager | Organization |
| **Custom org role** | Bundle of org-level permissions | GHEC |
| **Enterprise team** | Cross-org team managed at enterprise level | Enterprise (GHEC) |
| **Audit log** | Immutable event ledger; streamable to SIEM | Org & enterprise |
| **Policy cascade** | Enterprise → Org → Repo; tighten only, never loosen | All scopes |
| **Rulesets** | Stackable branch/tag protections; most restrictive wins | Org & repo |

**Repo role power ladder:**

```
Read  <  Triage  <  Write  <  Maintain  <  Admin
view     manage     push      manage repo   delete repo,
         issues     code      (non-destructive)  change visibility,
                                              full control
```

---

## 🧠 Self-check

Test yourself before peeking. Answers at the bottom.

1. Your company removed a departing engineer from Okta, but weeks later discovers their GitHub seat is still consuming a license. You only have **SAML** configured. What's missing, and what does it require to work?
2. A team lead must manage repo settings and merge PRs but must **never** be able to delete the repo or make it public. Which repository role do you assign?
3. What is the core difference between **SCIM** and **team synchronization**?
4. The enterprise policy disables public repositories enterprise-wide. An org owner wants to open-source one repo and asks you to enable public repos for just their org. Can you? Why or why not?
5. Name two characteristics that make an **EMU** account different from a standard GHEC personal account.
6. A user is fully signed in via SAML + 2FA but can't merge to a protected branch. Is this an authentication or an authorization problem? Explain in those terms.
7. Two rulesets apply to `main`: an org ruleset requiring 2 reviews and a repo ruleset blocking force-pushes. Which rules are enforced?
8. Where do you configure SAML SSO on **GHES** versus **standard GHEC**?

<details>
<summary><strong>Answer key</strong></summary>

1. **SCIM** is missing. SAML only blocks the departed user's *sign-in* — it does not deprovision the account or free the seat. SCIM auto-deactivates accounts and reclaims seats, and **SCIM requires SAML to be configured first**.
2. **Maintain.** It allows managing the repository and its non-destructive settings, but cannot delete the repo, change visibility, or manage access — those are **Admin** powers.
3. **SCIM** provisions/deprovisions *whether the account exists* (account & seat lifecycle from the IdP). **Team sync** maps *IdP groups to GitHub teams*, controlling *which team a user belongs to* — it does not create or delete accounts.
4. **No.** A higher scope (enterprise) policy that *disables* a setting locks it for all lower scopes. An org cannot **loosen** an enterprise-locked policy — it can only be equal or more restrictive. The enterprise would have to change the policy (or set it to "No policy" to delegate).
5. Any two of: enterprise-*owned* (not user-owned); username has a `_shortcode` suffix; can sign in **only** via the IdP; **SCIM-provisioned/deprovisioned** (can't self-create); **cannot** join outside organizations or fully participate in public GitHub.com.
6. **Authorization (authZ).** Authentication ("who you are") succeeded — SAML + 2FA let them in. The merge block is about "what you're allowed to do," which is governed by roles/branch rules — a separate, independent gate.
7. **Both.** Rulesets are **additive/stack**, so `main` requires 2 reviews **and** blocks force-pushes. The most restrictive combination always applies.
8. **GHES:** SAML is configured **globally** in the appliance's Management Console (whole-appliance authentication). **Standard GHEC:** SAML is configured at the **organization** level or the **enterprise** level (cascading to all orgs).

</details>

---

*Study tip: if you can correctly place every term in this domain onto the "secure office building" analogy — security desk (SAML/2FA), HR-to-badge wire (SCIM), directory (team sync), keycards (roles), and building codes that you can tighten but never loosen (policy cascade) — you'll handle most Domain 1 questions on sight.*
