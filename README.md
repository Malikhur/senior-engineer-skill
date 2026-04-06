# Senior Engineer Skill

![Agent Skills Compatible](https://img.shields.io/badge/Agent%20Skills-Compatible-brightgreen) ![AI Supported](https://img.shields.io/badge/AI-Cursor%20%7C%20Claude%20%7C%20Copilot%20%7C%20Windsurf%20%7C%20Codex-blue) ![Engineering Quality](https://img.shields.io/badge/Engineering-Senior%20Level-orange)

> A collection of skills that make AI coding agents write code like a senior software engineer. Stops the AI from generating junior-level, insecure, untested, poorly-architected slop.

---

## What is this?

**Senior Engineer Skill** is a collection of composable AI agent skills modeled after the wildly successful — but instead of frontend design, it focuses on **senior backend and fullstack software engineering excellence**.

Every AI coding agent you use — Claude Code, Cursor, GitHub Copilot, Windsurf, Codex, or any agent that reads `SKILL.md` files — will instantly adopt senior-engineer habits:
- Write code that passes security review
- Design systems that scale
- Write tests that actually catch bugs
- Build APIs that developers love
- Ship production-ready code from the first draft

---

## Install

```bash
npx skills add https://github.com/Malikhur/senior-engineer-skill
```

Or clone and reference manually:

```bash
git clone https://github.com/Malikhur/senior-engineer-skill.git
```

Then point your agent to the `skills/` directory or drop the relevant `SKILL.md` files into your project.

---

## Skills

| Skill | Description |
|---|---|
| [architect-skill](skills/architect-skill/SKILL.md) | System design, Clean Architecture, DDD, microservices vs monolith decisions, scalability patterns |
| [code-quality-skill](skills/code-quality-skill/SKILL.md) | SOLID principles, design patterns, code smells, naming conventions, clean code enforcement |
| [testing-skill](skills/testing-skill/SKILL.md) | TDD, test pyramid, edge case generation, mocking discipline, property-based testing |
| [security-skill](skills/security-skill/SKILL.md) | OWASP Top 10, input validation, auth/authz patterns, secrets management, CORS/CSP |
| [performance-skill](skills/performance-skill/SKILL.md) | Big-O awareness, N+1 prevention, caching strategies, async patterns, profiling-first optimization |
| [devops-skill](skills/devops-skill/SKILL.md) | CI/CD pipelines, IaC, container best practices, observability, GitOps, graceful shutdown |
| [database-skill](skills/database-skill/SKILL.md) | Schema design, zero-downtime migrations, indexing strategy, query optimization, transactions |
| [api-design-skill](skills/api-design-skill/SKILL.md) | REST maturity model, HTTP semantics, status codes, pagination, versioning, RFC 7807 errors |
| [code-review-skill](skills/code-review-skill/SKILL.md) | PR description quality, Conventional Commits, review checklists, constructive feedback style |
| [output-skill](skills/output-skill/SKILL.md) | Full-output enforcement — bans `// TODO`, `// ...`, and all lazy placeholder patterns |

---

## Settings (Configurable Dials)

The `architect-skill` exposes configurable dials at the top of its SKILL.md. Edit them to match your project's needs:

| Dial | Default | Range | Description |
|---|---|---|---|
| **ARCHITECTURE_COMPLEXITY** | `6` | 1–10 | `1` = Simple scripts/monolith · `10` = Full distributed systems with event sourcing |
| **STRICTNESS_LEVEL** | `7` | 1–10 | `1` = Loose guidelines · `10` = Zero-tolerance, every rule enforced |
| **VERBOSITY** | `5` | 1–10 | `1` = Minimal comments/docs · `10` = Exhaustive documentation, JSDoc on everything |

Example: For a startup MVP, set `ARCHITECTURE_COMPLEXITY: 3, STRICTNESS_LEVEL: 5`. For a regulated fintech system, set `ARCHITECTURE_COMPLEXITY: 8, STRICTNESS_LEVEL: 9`.

---

## Common Questions

**Does this work with my language/framework?**
Yes. Every skill is language-agnostic and framework-agnostic. The rules describe *engineering decisions*, not syntax.

**Can I use just one skill instead of all of them?**
Absolutely. Drop a single `SKILL.md` into your project root or reference it explicitly in your agent's context.

**Will this slow down my AI agent?**
No. These are instruction files, not runtime overhead. They change the *quality* of output, not the speed.

**Does this conflict with other instruction files?**
Skills compose well. If you have a `.cursorrules` or `CLAUDE.md` already, the skills add specificity on top of your existing instructions.

**Which agents support SKILL.md?**
Claude Code, Cursor, GitHub Copilot (via `.github/copilot-instructions.md`), Windsurf, Codex, and any agent with configurable instruction loading. The `skills/llms.txt` index enables automatic discovery.

---

## Framework-Agnostic by Design

These skills work whether you're writing Python, TypeScript, Go, Rust, Java, or anything else. The rules are about:
- *How* you structure code, not which library you import
- *What* you test, not which test framework you use
- *Why* you make architecture decisions, not which cloud you deploy to

---

## Contribute

1. Fork this repo
2. Create a new skill directory under `skills/your-skill-name/`
3. Add a `SKILL.md` with YAML front matter, direct imperative voice, Banned Patterns, Bias Correction, and a Pre-flight Checklist
4. Update `skills/llms.txt` with a one-line description
5. Open a PR

Skills should be **minimum 1500 words**, dense, and actionable. Study the existing SKILL.md files for conventions.

---

## Related

- [taste-skill](https://github.com/Leonxlnx/taste-skill) — The original skills repo for frontend design (7k+ stars)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/) — Security reference used in security-skill
- [Conventional Commits](https://www.conventionalcommits.org/) — Commit standard used in code-review-skill
- [RFC 7807](https://www.rfc-editor.org/rfc/rfc7807) — Problem Details standard used in api-design-skill
