# GH-100: GitHub Enterprise Administrator — Study Notes

> A complete, exam-aligned study guide for **Exam GH-100: GitHub Enterprise Administrator**.
> Content is mapped to the **official Microsoft Learn blueprint ("Skills measured as of July 2026")**, which restructured the exam into **5 domains**. Each domain lives in its own markdown file, written with plain-language explanations, worked examples, and analogies.

---

## 📌 Exam at a glance

| Field | Detail |
|-------|--------|
| **Exam code** | GH-100 |
| **Title** | GitHub Enterprise Administrator |
| **Maintained by** | GitHub (delivered through Microsoft / Pearson VUE) |
| **Level** | Intermediate |
| **Passing score** | 700 / 1000 |
| **Audience** | System admins, DevOps engineers, application admins, IT pros administering GitHub Enterprise (Cloud **and** Server) |
| **Blueprint version** | Skills measured as of **July 2026** |

> ⚠️ **Heads-up:** The exam changed *significantly* in July 2026 — objectives were added, removed, reworded, and regrouped. Older study material organized around 4 or 6 domains is **out of date**. These notes follow the current **5-domain** structure.

---

## 🗺️ Domains & weights

| # | Domain | Weight | File |
|---|--------|--------|------|
| 1 | Manage GitHub identities and access | **15–20%** | [domain-1-identities-and-access.md](./domain-1-identities-and-access.md) |
| 2 | Administer GitHub Enterprise environment | **10–15%** | [domain-2-enterprise-environment.md](./domain-2-enterprise-environment.md) |
| 3 | Implement secure software development and compliance | **25–30%** ⭐ | [domain-3-secure-development-and-compliance.md](./domain-3-secure-development-and-compliance.md) |
| 4 | Manage GitHub Actions | **20–25%** | [domain-4-github-actions.md](./domain-4-github-actions.md) |
| 5 | Monitor and optimize GitHub usage | **10–15%** | [domain-5-monitor-and-optimize.md](./domain-5-monitor-and-optimize.md) |

⭐ = Heaviest domain. If you're short on time, **start with Domain 3**, then Domain 4.

```
Weight distribution (approx. midpoints)
Domain 3  ███████████████████████████  27%
Domain 4  ██████████████████████       22%
Domain 1  █████████████████            17%
Domain 2  █████████████                12%
Domain 5  █████████████                12%
```

---

## 🧭 How to use these notes

Each domain file follows the same structure so you always know where you are:

1. **Why this domain matters** — the admin's real-world job in one paragraph.
2. **Big-picture analogy** — a mental model for the whole domain.
3. **Objective-by-objective notes** — every skill from the blueprint, each with:
   - a plain-English explanation,
   - a concrete **Example**,
   - an **Analogy** to make it stick,
   - CLI / YAML / config snippets where useful.
4. **⚠️ Exam traps** — the distinctions GitHub loves to test.
5. **Quick-reference table** — cram sheet for the night before.
6. **Self-check questions** — with answers at the bottom.

---

## ✅ Suggested 3-week study plan

| Week | Focus | Files |
|------|-------|-------|
| **Week 1** | Identity, access, and the enterprise environment | Domains 1 & 2 |
| **Week 2** | Security & compliance (the big one) + Actions | Domains 3 & 4 |
| **Week 3** | Monitoring/optimization, full review, practice exams | Domain 5 + review all |

**Daily rhythm:** read a section → do the matching task in a free GitHub org / GitHub Enterprise trial → answer that section's self-check questions.

---

## 🔑 Core mental model for the whole exam

GitHub administration is governance at **three nested scopes**:

```
Enterprise  →  the whole company account (policies cascade DOWN)
  └── Organization  →  a department/team grouping of repos
        └── Repository  →  a single project
```

A recurring exam theme: **a policy set at a higher scope can force, restrict, or allow a lower scope — but a lower scope can never loosen what a higher scope locked.** Keep this hierarchy in your head and half the policy questions answer themselves.

---

## 📚 Official resources

- Exam page & study guide: Microsoft Learn → *GitHub Administration* certification
- Learning paths: *GitHub Administration – Part 1 of 2* and *Part 2 of 2*
- GitHub Docs (Enterprise administration), GitHub Changelog, and `status.github.com`
- Practice assessment on Microsoft Learn

---

*These notes are a community study aid. Always verify against the official GitHub Docs and the Microsoft Learn study guide, since cloud features change frequently.*
