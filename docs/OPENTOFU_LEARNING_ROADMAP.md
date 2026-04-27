# OpenTofu Core: Learning Roadmap

This roadmap is for developers who already use OpenTofu (or similar tools) and want to **understand most of the core codebase** in a deliberate order. It aligns with how the project documents itself and how execution actually flows through the tree.

**Companion docs in this repo (read these first, in order):**

1. [docs/README.md](./README.md) — index of core documentation  
2. [docs/architecture.md](./architecture.md) — **start here** for request flow  
3. [docs/resource-instance-change-lifecycle.md](./resource-instance-change-lifecycle.md) — plan/apply from the provider RPC perspective  
4. [docs/planning-behaviors.md](./planning-behaviors.md) — planning edge cases  
5. [docs/diagnostics.md](./diagnostics.md) — errors and warnings  
6. [docs/plugin-protocol/](./plugin-protocol/) — gRPC/protobuf wire protocol (if you touch plugins or SDK boundaries)  
7. [contributing/DEVELOPING.md](../contributing/DEVELOPING.md) — build, test, debug  
8. [rfc/README.md](../rfc/README.md) — design evolution and future direction  

---

## Phase 0 — Prerequisites (flexible time)

| Topic | Why it matters in this repo |
|--------|-----------------------------|
| **Go** (modules, interfaces, concurrency, testing) | Almost all of core is Go. Graph walks use concurrency; tests are dense. |
| **HCL v2** | Config is HCL; see `internal/configs` and deferred `hcl.Body` / `hcl.Expression` in [architecture.md](./architecture.md). |
| **`cty`** ([zclconf/go-cty](https://github.com/zclconf/go-cty)) | Language values at runtime; providers receive `cty.Value`. |
| **gRPC / protobuf (basics)** | Provider protocol; see `internal/plugin`, `internal/plugin6`, `docs/plugin-protocol/`. |
| **Directed acyclic graphs (DAGs)** | Plan/apply ordering; `internal/dag`, transforms in `internal/tofu`. |

**Outcome:** You can build (`go build ./cmd/tofu`), run targeted tests (`go test ./internal/<pkg>`), and set `TF_LOG=trace` while running a small config.

---

## Phase 1 — User model and entrypoint (about 1–2 weeks)

**Goal:** Map CLI verbs to packages.

| Step | What to do |
|------|------------|
| 1.1 | Read [website/docs/intro/core-workflow.mdx](../website/docs/intro/core-workflow.mdx) (or [opentofu.org](https://opentofu.org/docs/) intro) so *init / validate / plan / apply / destroy* are crisp in your head. |
| 1.2 | Trace **command dispatch**: `cmd/tofu` → `internal/command` (see [architecture.md § CLI](./architecture.md#cli-command-package)). |
| 1.3 | Pick **one** command (e.g. `plan`) and read the command type, how it builds `backend.Operation`, and how it hands off to a backend. |
| 1.4 | Skim **`internal/backend`** and **`internal/backend/init`**: what is a backend vs “enhanced” backend, and when `local.Local` wraps others ([architecture.md § Backends](./architecture.md#backends)). |

**Packages:** `cmd/tofu`, `internal/command`, `internal/backend`, `internal/backend/local`, `internal/states/statemgr`.

**Exercise:** Add a log line or breakpoint in the `plan` path from `main` through to where `tofu.Context` is created (next phase).

---

## Phase 2 — Configuration loading (about 1–2 weeks)

**Goal:** Know how `.tf` / `.tofu` become `configs.Config`.

| Step | What to do |
|------|------------|
| 2.1 | Read **`internal/configs`** types: modules, resources, providers, variables. |
| 2.2 | Follow **`internal/configs/configload`**: `Loader` and module installation vs reload ([architecture.md § Configuration Loader](./architecture.md#configuration-loader)). |
| 2.3 | Touch **`internal/initwd`** and **`internal/getmodules`** if you care about `init` and module sources. |
| 2.4 | Skim **`internal/addrs`**: addresses for modules, resources, providers — used everywhere downstream. |

**Exercise:** Run `tofu init` on a tiny root + child module; while debugging, watch `configs.Config` assemble.

---

## Phase 3 — State (about 1 week)

**Goal:** Understand the bridge between remote APIs and the in-memory model.

| Step | What to do |
|------|------------|
| 3.1 | Read **`internal/states`** (`State`, instances, modules). |
| 3.2 | Read **`internal/states/statemgr`**: `Full`, filesystem vs remote backends. |
| 3.3 | Optional: **`internal/states/statefile`** serialization; **`internal/command/clistate`** if you care about CLI state subcommands. |

**Exercise:** Inspect a local `.tfstate` (or remote state after apply) and map JSON keys to `states` types.

---

## Phase 4 — The execution engine: `internal/tofu` (about 3–5 weeks)

**Goal:** This is the **largest** slice of “understanding the codebase.” Pace by subgraph.

| Step | What to do |
|------|------------|
| 4.1 | Read **`tofu.Context`**: `Plan`, `Apply`, `Validate`, etc., and how they invoke graph builders ([architecture.md § Graph Builder / Walk](./architecture.md#graph-builder)). |
| 4.2 | **`internal/dag`**: `AcyclicGraph.Walk` — ordering and parallelism contract. |
| 4.3 | **Graph transforms:** search for `GraphTransformer` implementations; read `ConfigTransformer`, `StateTransformer`, `ReferenceTransformer`, `ProviderTransformer` as called out in [architecture.md](./architecture.md#graph-builder). |
| 4.4 | **Graph walk:** `ContextGraphWalker`, `EvalContext` vs `BuiltinEvalContext` ([architecture.md § Graph Walk](./architecture.md#graph-walk)). |
| 4.5 | **Vertex execution:** types implementing `GraphNodeExecutable` / `Execute` — e.g. plannable vs applyable resource instances ([architecture.md § Vertex Evaluation](./architecture.md#vertex-evaluation)). |
| 4.6 | **Dynamic expansion:** `GraphNodeDynamicExpandable`, `count` / `for_each` subgraphs ([architecture.md § Sub-graphs](./architecture.md#sub-graphs)). |
| 4.7 | **`internal/lang`:** `References`, `Scope`, `EvalExpr` / `EvalBlock` — how expressions bind to state ([architecture.md § Expression Evaluation](./architecture.md#expression-evaluation)). |

**Parallel track (newer code):** If your checkout includes it, explore **`internal/engine`** and RFCs below — some execution work is moving or duplicating concepts for clearer architecture.

**Exercise:** Single-file config with two resources and an explicit `depends_on`; draw the plan graph on paper, then confirm edges in trace logs or debugger.

---

## Phase 5 — Plans, providers, and apply artifacts (about 2–3 weeks)

| Step | What to do |
|------|------------|
| 5.1 | **`internal/plans`** and **`plans/planfile`**: what gets serialized for `tofu apply` with a saved plan. |
| 5.2 | **`internal/plugin` / `internal/plugin6`** and **`internal/providers`**: dialing providers, schema, RPC boundaries. |
| 5.3 | Re-read [resource-instance-change-lifecycle.md](./resource-instance-change-lifecycle.md) with the above packages open. |
| 5.4 | **`internal/registry`** and **`internal/getproviders`**: provider installation (ties to `init`). |

**Exercise:** Run `plan` with `-json` or structured logging (per docs) and map output fields to `plans` types.

---

## Phase 6 — Supporting subsystems (pick by interest, 1–2 weeks each)

| Area | Packages / docs |
|------|------------------|
| **Encryption** | `internal/encryption`, [docs/state_encryption.md](./state_encryption.md) |
| **Cloud / TFE-like** | `internal/cloud` |
| **Checks / preconditions** | `internal/checks`, config validation paths |
| **Refactoring (moved, import)** | `internal/refactoring`, `internal/configs` move blocks |
| **Testing framework (`tofu test`)** | `internal/moduletest`, related command wiring |
| **Built-in `terraform`/`tofu` provider** | `internal/builtin/providers` |
| **Legacy SDK** | `internal/legacy` — historical; read only when chasing compatibility |

---

## Phase 7 — Design evolution and “why it looks like this”

| Document | Purpose |
|----------|---------|
| [rfc/20250728-execution-architecture.md](../rfc/20250728-execution-architecture.md) | Pain points and goals for core architecture |
| [rfc/20251001-eval-plan-apply-architecture.md](../rfc/20251001-eval-plan-apply-architecture.md) | Evaluator / planner / applier direction |

**Outcome:** You can read a PR that touches `internal/tofu` or `internal/engine` and infer which subsystem it belongs to.

---

## Suggested weekly rhythm

- **3–5 deep sessions** per week (2–3 hours): read doc → read code → run one test package under debugger.  
- **One “vertical slice”** per week: e.g. “from `tofu plan` flag parsing to first provider PlanResourceChange.”  
- **Avoid** trying to read every `internal/command/testdata` file; use them only when a test fails or you need a minimal repro.

**Rough calendar (part-time):** Phases 0–3 ≈ 1 month; Phase 4 alone ≈ 1 month+; Phases 5–7 ≈ 1–2 months — **~3–5 months** total for solid coverage, depending on prior Go/systems background.

---

## Research papers and academic context

OpenTofu/Terraform-style tools are **industry systems**; there is no single paper “about OpenTofu.” Academic work usually studies **Infrastructure as Code (IaC)** in general (quality, security, testing, adoption). The papers below help you reason about *why* core concerns (idempotence, drift, state, modules, policy) show up in the design.

### Foundational surveys and mapping studies

1. **Rahman, A., Mahdavi-Hezaveh, R., & Williams, L. (2019).** *A systematic mapping study of infrastructure as code research.* **Information and Software Technology**, 108, 65–77.  
   - **DOI:** [10.1016/j.infsof.2018.12.004](https://doi.org/10.1016/j.infsof.2018.12.004)  
   - **Open access preprint:** [arXiv:1807.04872](https://arxiv.org/abs/1807.04872)  
   - **Why read it:** Landscape of IaC research topics (tools, adoption, testing, defects) — good map of *adjacent* problems researchers care about.

2. **Sharma, S., et al. (2017).** *A systematic mapping study on DevOps.* (Often cited alongside IaC work for CI/CD and automation context.)  
   - Useful for understanding where IaC sits in broader “DevOps automation” research. Search the title in your library for the exact venue/edition.

### Quality, defects, and security (IaC scripts and configs)

3. Search for **“Infrastructure as Code smells”** and **“IaC anti-patterns”** in *MSR* / *ICSME* / *EMSE* — there is a growing line of work on static analysis of Ansible/Terraform/Chef-style artifacts (names evolve yearly). Use Rahman (2019)’s bibliography as a seed.

4. **Policy-as-code / compliance** — Often appears as **OPA/Rego** or **Sentinel**-style enforcement in industry; academically, look for *policy enforcement* + *cloud configuration* in recent software-engineering venues (2022–2026).

### Understandability and maintainability (helps when reading huge graphs)

5. **Quéval, P.-J., et al. (2025).** *On the understandability of coupling-related practices in infrastructure-as-code based deployments.* **Information and Software Technology**, 185, 107761.  
   - **DOI:** [10.1016/j.infsof.2025.107761](https://doi.org/10.1016/j.infsof.2025.107761)  
   - **Why read it:** Connects **coupling** in IaC deployments to human comprehension — relevant when you wonder why `internal/tofu` invests heavily in explicit dependencies and graph transforms.

### What is *not* a substitute for papers

- **HCL specification and implementation** ([hashicorp/hcl](https://github.com/hashicorp/hcl)) — normative for syntax; not paper-centric.  
- **OpenTofu / Terraform plugin protocol** — implementation truth is in `docs/plugin-protocol/` and protobufs in-repo.  
- **Classical scheduling / DAG execution** — standard algorithms texts (parallel DAG scheduling, topological order) explain `internal/dag` at a mathematical level if you want deeper theory.

---

## Exporting this roadmap to PDF

This file is Markdown. Common options:

```bash
# If you have pandoc and a LaTeX engine (e.g. BasicTeX / MacTeX, provides pdflatex):
pandoc docs/OPENTOFU_LEARNING_ROADMAP.md -o OPENTOFU_LEARNING_ROADMAP.pdf

# No LaTeX: generate HTML and use the browser’s Print → Save as PDF
pandoc docs/OPENTOFU_LEARNING_ROADMAP.md -o OPENTOFU_LEARNING_ROADMAP.html --standalone \
  --metadata title="OpenTofu Core Learning Roadmap"

# Or open this file in any Markdown preview (VS Code, GitHub) and print to PDF
```

---

## Quick reference — top-level `internal/` packages

| Directory | Role (one line) |
|-----------|------------------|
| `command` | CLI commands, UI, JSON views |
| `backend` | State storage + operation execution wiring |
| `configs` | Parsed configuration model |
| `configload` | Load root + child modules into `configs.Config` |
| `tofu` | Core graph build/walk/eval (heart of plan/apply) |
| `dag` | DAG data structure and walk |
| `states` / `statemgr` | State model and persistence interfaces |
| `lang` | Expression evaluation, functions, references |
| `plans` / `planfile` | Plan objects and serialized plan files |
| `plugin` / `plugin6` | gRPC provider client |
| `addrs` | Address types for modules, resources, providers |
| `registry` / `getproviders` | Registry protocol and provider installation |
| `initwd` | Working directory init concerns |
| `encryption` | State encryption |
| `engine` | Newer execution-related implementation (see RFCs) |
| `legacy` | Compatibility layer for older provider SDK patterns |

---

*This roadmap is a study guide maintained alongside the repo; it is not an official governance document. For contribution rules, see [CONTRIBUTING.md](../CONTRIBUTING.md).*
