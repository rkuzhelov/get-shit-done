# Requirements Generation Flow

## Entry Points

Two commands can kick off the flow:

- **`/gsd:new-project`** — first-time project initialization
- **`/gsd:new-milestone`** — adding features to an existing project

## End-to-End Execution Order

```
User runs: /gsd:new-project
│
├─ Phase 3-4: Orchestrator — Deep Questioning
│  └─ INPUT:  user answers (freeform conversation)
│  └─ OUTPUT: .planning/PROJECT.md
│
├─ Phase 5: Orchestrator — Workflow Preferences
│  └─ OUTPUT: .planning/config.json
│
├─ Phase 6 [Optional]: Research
│  ├─ 4× gsd-project-researcher (parallel)
│  │  ├─ STACK research    → .planning/research/STACK.md
│  │  ├─ FEATURES research → .planning/research/FEATURES.md
│  │  ├─ ARCHITECTURE      → .planning/research/ARCHITECTURE.md
│  │  └─ PITFALLS          → .planning/research/PITFALLS.md
│  │
│  └─ gsd-research-synthesizer (waits for all 4)
│     └─ OUTPUT: .planning/research/SUMMARY.md
│
├─ Phase 7: Orchestrator — Define Requirements  ⬅ REQUIREMENTS.md created here
│  ├─ INPUT:  research/FEATURES.md (if exists), user conversation
│  └─ OUTPUT: .planning/REQUIREMENTS.md
│             (v1 reqs, v2 reqs, out-of-scope, traceability table)
│
├─ Phase 8: gsd-roadmapper agent
│  ├─ INPUT:  REQUIREMENTS.md, PROJECT.md, SUMMARY.md, config.json
│  └─ OUTPUT: .planning/ROADMAP.md, .planning/STATE.md
│             + updates REQUIREMENTS.md traceability section
│
└─ Phase 9: git commit all .planning/ files
```

## Implementation Approach (Feature → Code)

Once requirements and roadmap exist, features get implemented through this chain:

```
REQUIREMENTS.md  (what to build)
    ↓
ROADMAP.md       (which phase builds what)
    ↓
/gsd:discuss-phase → {phase}-CONTEXT.md  (user's implementation preferences)
    ↓
/gsd:plan-phase
    ├─ gsd-phase-researcher  → {phase}-RESEARCH.md
    ├─ gsd-planner           → {phase}-01-PLAN.md, {phase}-02-PLAN.md, ...
    └─ gsd-plan-checker      → validates coverage, iterates until pass
    ↓
/gsd:execute-phase
    └─ gsd-executor (one per PLAN.md, parallelized by wave)
       └─ OUTPUT: SUMMARY.md per plan + VERIFICATION.md per phase
```

## Key Agents

| Agent | Role | Inputs | Outputs |
|-------|------|--------|---------|
| **Orchestrator** | Coordinates flow, deep questioning | User answers | PROJECT.md, REQUIREMENTS.md |
| **gsd-project-researcher** (×4) | Domain research | PROJECT.md + web | STACK/FEATURES/ARCHITECTURE/PITFALLS.md |
| **gsd-research-synthesizer** | Merge research | 4 research files | SUMMARY.md |
| **gsd-roadmapper** | Map reqs → phases | REQUIREMENTS.md, PROJECT.md, SUMMARY.md | ROADMAP.md, STATE.md, updates REQUIREMENTS.md |
| **gsd-phase-researcher** | Research implementation | ROADMAP phase + CONTEXT | {phase}-RESEARCH.md |
| **gsd-planner** | Break phase into tasks | CONTEXT, RESEARCH, ROADMAP | {phase}-NN-PLAN.md files |
| **gsd-plan-checker** | Validate plans | PLAN files, ROADMAP | Pass/fail + revision notes |
| **gsd-executor** | Execute plans | PLAN.md (is the prompt) | Code + SUMMARY.md + VERIFICATION.md |

## Key Source Files

- **Commands:** `commands/gsd/new-project.md`, `commands/gsd/plan-phase.md`, `commands/gsd/execute-phase.md`
- **Agents:** `agents/gsd-project-researcher.md`, `agents/gsd-research-synthesizer.md`, `agents/gsd-roadmapper.md`, `agents/gsd-planner.md`
- **Templates:** `get-shit-done/templates/requirements.md`, `get-shit-done/templates/roadmap.md`, `get-shit-done/templates/project.md`

## Design Principles

- **Traceability**: every v1 requirement maps to exactly one phase; 100% coverage validated
- **Parallel execution**: 4 researchers run simultaneously; plans execute in waves
- **Fresh context**: each spawned agent gets its own ~200k token window with only what it needs
- **Flexible depth**: Quick (3-5 phases), Standard (5-8), Comprehensive (8-12+) — applied after phase identification, not imposed
