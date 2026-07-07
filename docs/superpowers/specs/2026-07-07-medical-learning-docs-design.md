# Medical Workspace Learning Docs Design

## Goal

为 `/home/qmz/projects/learning-agent/medical` 编写一套中文学习文档，帮助没有接触过该项目的人全面掌握当前 workspace。文档必须同时覆盖 `medix-agent-swarm` 和 `MediX-R1` 两条主线，并包含项目使用工具的详细讲解。

## Scope

本次交付创建 `docs/learning/` 下的 28 篇 Markdown 文档：

- 总览与学习路线：4 篇
- `medix-agent-swarm` 主线：10 篇
- `MediX-R1` 主线：6 篇
- 工具详解手册：8 篇

文档以当前源码为准，不把 README 中的愿景或过期描述当作已实现事实。遇到 README 与源码不一致时，文档要明确标注。

## Audience

目标读者是项目新学习者，假设他们知道 Python 和大模型基本概念，但不熟悉本项目、医疗 Agent、veRL/Ray/FSDP、Milvus、Mem0、vLLM 等具体工具链。

## Documentation Principles

- 先讲整体定位，再讲运行链路，再进入模块和工具。
- 每篇文档说明：学习目标、源码入口、关键概念、执行流程、注意事项。
- 工具文档说明：工具解决什么问题、项目中哪里使用、输入输出、配置项、常见坑、与其他工具关系。
- 区分“源码已确认事实”和“README 或注释中的愿景/说法”。
- 不泄露 API key、token、私有凭据。
- 不修改业务源码。

## Deliverables

### Overview Docs

1. `00-project-overview.md`
2. `01-learning-roadmap.md`
3. `02-source-vs-readme-facts.md`
4. `03-run-and-environment.md`

### medix-agent-swarm Docs

5. `10-agent-swarm-architecture.md`
6. `11-request-lifecycle.md`
7. `12-swarm-routing-and-task-decomposition.md`
8. `13-agent-loop-and-function-calling.md`
9. `14-skills-system.md`
10. `15-memory-system.md`
11. `16-knowledge-base-rag.md`
12. `17-deep-research-workflow.md`
13. `18-constraints-and-safety.md`
14. `19-agent-swarm-current-issues.md`

### MediX-R1 Docs

15. `20-medix-r1-training-overview.md`
16. `21-training-data-and-config.md`
17. `22-reward-function.md`
18. `23-verl-training-lifecycle.md`
19. `24-evaluation-pipeline.md`
20. `25-results-and-training-report.md`

### Tool Handbook Docs

21. `30-tools-overview.md`
22. `31-llm-and-function-calling-tools.md`
23. `32-rag-and-vector-db-tools.md`
24. `33-memory-tools.md`
25. `34-web-research-tools.md`
26. `35-training-tools.md`
27. `36-evaluation-tools.md`
28. `37-debugging-and-next-steps.md`

## Current Project Facts To Capture

- Root directory is not a git repository.
- Existing Markdown docs are only the three README files under `medix-agent-swarm`, `MediX-R1/training`, and `MediX-R1/eval`.
- `medix-agent-swarm` currently has import blockers: missing `swarm/events.py` and `validation/auto_fixer.py`.
- `medix-agent-swarm/core/llm_client.py` hard-codes an old `/Users/.../swarm` path before importing `config`.
- `MediX-R1/training/run_train.sh` points to `examples/medix-r1_8b_dapo.sh`, while the current tree contains `medix-r1_2b_dapo.sh` and `medix-r1_8b_gspo.sh`.
- Most Agent skills depend on Milvus Lite and SentenceTransformers; DeepResearch additionally depends on LLM, web search, and evidence synthesis.
- The current Milvus knowledge base is a small demo knowledge base with 10 text documents and a local `milvus_lite.db`.

## Verification

Completion requires:

- Exactly 28 learning docs under `docs/learning/`.
- All named files above exist.
- Learning docs contain no placeholder markers.
- Docs mention both `medix-agent-swarm` and `MediX-R1`.
- Tool handbook docs cover LLM/function calling, RAG/vector DB, memory, web research, training, and evaluation tools.
- Verification commands are run and reported.

## Environment Constraint

The brainstorming skill normally asks for a commit after writing the spec. This workspace root is not a git repository, so no commit can be created here. The substitute evidence is file existence checks, counts, placeholder scans, and content scans.
