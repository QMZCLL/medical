# Medical Workspace Learning Docs Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create a complete Chinese learning-documentation suite for the medical workspace.

**Architecture:** Documentation is organized into four sections: overview, `medix-agent-swarm`, `MediX-R1`, and tool handbook. Each document is self-contained but links conceptually to the runtime or training/evaluation lifecycle.

**Tech Stack:** Markdown, local source inspection, shell verification with `find`, `wc`, and `rg`.

## Global Constraints

- Write Chinese documentation for newcomers.
- Use current source facts as authoritative.
- Separate README claims from source-confirmed behavior.
- Do not modify business source code.
- Do not expose secrets or credentials.
- Root directory is not a git repository, so commit steps are replaced by verification evidence.

---

## File Structure

- Create: `docs/learning/00-project-overview.md`
- Create: `docs/learning/01-learning-roadmap.md`
- Create: `docs/learning/02-source-vs-readme-facts.md`
- Create: `docs/learning/03-run-and-environment.md`
- Create: `docs/learning/10-agent-swarm-architecture.md`
- Create: `docs/learning/11-request-lifecycle.md`
- Create: `docs/learning/12-swarm-routing-and-task-decomposition.md`
- Create: `docs/learning/13-agent-loop-and-function-calling.md`
- Create: `docs/learning/14-skills-system.md`
- Create: `docs/learning/15-memory-system.md`
- Create: `docs/learning/16-knowledge-base-rag.md`
- Create: `docs/learning/17-deep-research-workflow.md`
- Create: `docs/learning/18-constraints-and-safety.md`
- Create: `docs/learning/19-agent-swarm-current-issues.md`
- Create: `docs/learning/20-medix-r1-training-overview.md`
- Create: `docs/learning/21-training-data-and-config.md`
- Create: `docs/learning/22-reward-function.md`
- Create: `docs/learning/23-verl-training-lifecycle.md`
- Create: `docs/learning/24-evaluation-pipeline.md`
- Create: `docs/learning/25-results-and-training-report.md`
- Create: `docs/learning/30-tools-overview.md`
- Create: `docs/learning/31-llm-and-function-calling-tools.md`
- Create: `docs/learning/32-rag-and-vector-db-tools.md`
- Create: `docs/learning/33-memory-tools.md`
- Create: `docs/learning/34-web-research-tools.md`
- Create: `docs/learning/35-training-tools.md`
- Create: `docs/learning/36-evaluation-tools.md`
- Create: `docs/learning/37-debugging-and-next-steps.md`

## Tasks

### Task 1: Overview Docs

**Files:**
- Create: `docs/learning/00-project-overview.md`
- Create: `docs/learning/01-learning-roadmap.md`
- Create: `docs/learning/02-source-vs-readme-facts.md`
- Create: `docs/learning/03-run-and-environment.md`

**Produces:** Newcomer entry points and project-wide source-fact boundaries.

- [ ] Write the four overview docs.
- [ ] Verify the files exist.
- [ ] Scan for placeholders.

### Task 2: medix-agent-swarm Docs

**Files:**
- Create: `docs/learning/10-agent-swarm-architecture.md`
- Create: `docs/learning/11-request-lifecycle.md`
- Create: `docs/learning/12-swarm-routing-and-task-decomposition.md`
- Create: `docs/learning/13-agent-loop-and-function-calling.md`
- Create: `docs/learning/14-skills-system.md`
- Create: `docs/learning/15-memory-system.md`
- Create: `docs/learning/16-knowledge-base-rag.md`
- Create: `docs/learning/17-deep-research-workflow.md`
- Create: `docs/learning/18-constraints-and-safety.md`
- Create: `docs/learning/19-agent-swarm-current-issues.md`

**Produces:** Runtime and architecture learning docs for the medical Agent system.

- [ ] Write the ten Agent Swarm docs.
- [ ] Verify the files exist.
- [ ] Scan for placeholders.

### Task 3: MediX-R1 Docs

**Files:**
- Create: `docs/learning/20-medix-r1-training-overview.md`
- Create: `docs/learning/21-training-data-and-config.md`
- Create: `docs/learning/22-reward-function.md`
- Create: `docs/learning/23-verl-training-lifecycle.md`
- Create: `docs/learning/24-evaluation-pipeline.md`
- Create: `docs/learning/25-results-and-training-report.md`

**Produces:** Training and evaluation learning docs for MediX-R1.

- [ ] Write the six MediX-R1 docs.
- [ ] Verify the files exist.
- [ ] Scan for placeholders.

### Task 4: Tool Handbook Docs

**Files:**
- Create: `docs/learning/30-tools-overview.md`
- Create: `docs/learning/31-llm-and-function-calling-tools.md`
- Create: `docs/learning/32-rag-and-vector-db-tools.md`
- Create: `docs/learning/33-memory-tools.md`
- Create: `docs/learning/34-web-research-tools.md`
- Create: `docs/learning/35-training-tools.md`
- Create: `docs/learning/36-evaluation-tools.md`
- Create: `docs/learning/37-debugging-and-next-steps.md`

**Produces:** Detailed tool explanations tied to actual project usage.

- [ ] Write the eight tool handbook docs.
- [ ] Verify the files exist.
- [ ] Scan for placeholders.

### Task 5: Final Verification

**Files:**
- Inspect: `docs/learning/*.md`
- Inspect: `docs/superpowers/specs/2026-07-07-medical-learning-docs-design.md`
- Inspect: `docs/superpowers/plans/2026-07-07-medical-learning-docs.md`

**Produces:** Evidence that the full documentation request is complete.

- [ ] Run `find docs/learning -maxdepth 1 -type f -name '*.md' | sort`.
- [ ] Run `find docs/learning -maxdepth 1 -type f -name '*.md' | wc -l` and expect `28`.
- [ ] Run a placeholder scan on `docs/learning`.
- [ ] Run targeted scans for `medix-agent-swarm`, `MediX-R1`, `Milvus`, `Mem0`, `vLLM`, `Ray`, `FSDP`, and `function calling`.
- [ ] Report that git commit was skipped because root is not a git repository.
