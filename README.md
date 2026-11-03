# SWE-Gemma — Kaggle Gemma 4 Developer Agent Competition

An **autonomous software-engineering agent** built for the [Kaggle Gemma 4 Developer Agent Competition](https://www.kaggle.com/competitions/gemma-4-developer-agent). SWE-Gemma works inside a hermetic Docker sandbox to resolve GitHub-issue-style tickets by producing **minimal, correct code patches**, verified by `pytest` in a clean container.

Built declaratively on **Google ADK (Agent Development Kit)**, powered by the **Gemma 4 31B** quantized model (`gemma-4-31b-it-qat-w4a16-ct`) served via vLLM across 4× NVIDIA L4 GPUs.

---

## What It Does

Given a bug report / issue ticket inside `/workspace`, the agent:

1. **Reads the task** — extracts failing tests, error messages, and affected symbols.
2. **Explores** — uses a dedicated read-only code-explorer sub-agent backed by pre-built code-graph embeddings (`search_similar_code`, `get_code_neighbors`, `get_code_subgraph`).
3. **Finds the root cause** — classifies failures (logic / missing import / assertion / signature / edge-case / dependency).
4. **Patches minimally** — prefers targeted `edit_file` over `write_file`, keeping original style.
5. **Verifies** — runs the narrowest failing test first, expanding outward (method → file → directory → suite).
6. **Submits** — cleans `/workspace` and calls `submit_patch()`.

**Success metric:** a patch *only* counts if `pytest` exits `0` in a fresh verification container. Test files are reset by the harness, so the agent must fix real library code.

---

## Architecture

The agent is a **declarative ADK config** — no imperative agent code. The whole agent is defined in YAML and packaged into `submission.zip`.

```
submission/
├── agent.yaml                    # Root agent config (LlmAgent)
├── configs/
│   └── sampling.yaml             # Generation / thinking parameters
├── prompts/
│   ├── system.md                 # Main SWE agent system prompt
│   └── explorer.md               # Read-only exploration sub-agent prompt
├── sub_agents/
│   └── code_explorer.yaml        # Code-explorer sub-agent (AgentTool)
└── skills/
    ├── repo_navigation/          # Progressive narrowing search strategy
    ├── debugging/                # Error classification & root-cause isolation
    └── test_driven_repair/       # Narrow → wide pytest workflow
```

### Root Agent (`agent.yaml`)

```yaml
agent_class: LlmAgent
name: swegemma_swe_agent
model: gemma-4-31b-it-qat-w4a16-ct
instruction: !include prompts/system.md
tools: [run_command, read_file, edit_file, write_file,
        search_similar_code, get_code_neighbors, get_code_subgraph,
        get_status, submit_patch,
        agent_tool: {config_path: sub_agents/code_explorer.yaml, skip_summarization: true}]
generate_content_config: !include configs/sampling.yaml
skills: [skills/repo_navigation, skills/debugging, skills/test_driven_repair]
```

Key design choices:

- **Single model rule** — every agent declares the same base model, avoiding `ParticipantVisibleError`.
- **Sandboxed tools** — all tools are bound via `SwegemmaContext.create_tools()`; `get_status` and `submit_patch` are **free** (don't count toward the tool-call budget).
- **Context-window optimization** — the code-explorer `AgentTool` uses `skip_summarization: true`: the sub-agent's full findings pass to the parent while intermediate `read_file` noise stays out of the parent's context.
- **Hermetic verification** — two phase, two air-gapped containers: the agent sandbox (Container A) and a clean verification sandbox (Container B, `network_mode="none"`), so patches are tested against a pristine baseline.

### Sampling Config (`configs/sampling.yaml`)

```yaml
temperature: 0.1          # Deterministic, low creativity
top_p: 0.95
top_k: 40
max_output_tokens: 16384  # Half of the 32k context window
thinking_config:
  thinking_budget: 2048   # Preserve context for multi-file edits
  thinking_level: "LOW"
  include_thoughts: false
```

### Code Explorer Sub-Agent (`sub_agents/code_explorer.yaml`)

- **Read-only** — never writes or edits; delegated via `AgentTool` for isolated context.
- **Cheap** — low thinking budget (`1024`), `include_contents: none`.
- **Graph-backed** — prefers `search_similar_code` → `get_code_neighbors` → `get_code_subgraph` over raw `grep`/`find`.

---

## The Prompt (system.md)

The heart of the agent is a disciplined 7-step workflow:

1. **Read Task** — extract failing tests and hints.
2. **Progressive Exploration** — code-explorer sub-agent first, then graph tools.
3. **Locate Root Cause** — form a single, testable hypothesis.
4. **Minimal Patch** — smallest change, preserve style.
5. **Targeted Verification Ladder** — narrowest test → full file → directory → suite.
6. **Final Check** — `get_status()`, clean `/workspace`, `submit_patch()`.
7. **Budget Discipline** — poll `get_status()` every 8–10 calls; if <15 calls remain, submit the best patch.

Budget semantics: `get_status` / `submit_patch` are free; everything else counts (~50 call budget, 60-minute wall clock).

---

## Skills

| Skill | Purpose |
|-------|---------|
| **repo_navigation** | Survey (`find -maxdepth 3`) → semantic search → graph traversal → targeted read; grep fallbacks |
| **debugging** | 6 error classes, stack-trace analysis, root-cause isolation, targeted testing ladder, stop after 3 identical failures |
| **test_driven_repair** | Find relevant tests → narrow test (`::test_method -x -q`) → expand → minimal patch → re-verify; never modify tests |

---

## Building the Submission

`submission.zip` is the packaged agent (8 files, ~8.3 KB). Build and verify with:

```bash
python build_zip.py       # Builds submission.zip + runs post-build asserts
python final_report.py    # Independent verification checklist + workspace summary
```

### Verification checks

-  `!include` path — uses `prompts/explorer.md`, **no** `../` path traversal (the `!include` sandbox blocks `..`)
-  `system.md` — no redundant `{hints}` template variable (the harness injects hints as a separate message)
-  `thinking_budget: 2048` — matches build/report assertions
-  `agent.yaml` at zip root (`MissingRootConfigError` otherwise)

> Note: the repo also contains a stale `gemma-4-developer-agent.zip` (18.8 MB, corrupt `BadZipFile`) — not referenced by the build and safe to delete.

---

##  Getting Started

### Prerequisites
- Python 3.12+ (ADK).
- A running Gemma 4 31B endpoint (vLLM, quantized INT4) on your inference infra.
- Sandbox docker images (`competition/docker/`) for real evaluation.

### Run the build
```bash
git clone <this-repo>
cd kaggle-gemma-4-developer-agent
python build_zip.py        # produce submission.zip
python final_report.py     # verify the artifact
```

---

## Known Blockers

- **Windows HTTPS egress** — this repo was originally built on Windows, where all HTTPS submission paths (Kaggle CLI, GitHub API, `curl`) failed with a Schannel `SEC_E_NO_CREDENTIALS` TLS error. The `submission.zip` artifact itself is valid; only the *upload path* was affected. On a clean environment (WSL, CI, or Linux) standard uploads work.
- **Submission blocked from this machine** — use WSL, a CI runner, or the Kaggle web UI to submit.

---

## Repository Layout

```
kaggle-gemma-4-developer-agent/
├── submission/                # Declarative ADK agent → packaged into submission.zip
├── competition/               # Reference harness & evaluation infra (read-only)
│   ├── HARNESS_README.md      # Full 654-line harness technical reference
│   └── docker/                # Dockerfile.sandbox / public, imp.py & telnetlib.py shims
├── build_zip.py               # Zip build script with verification asserts
├── final_report.py            # Post-build verification + workspace summary
├── submission.zip             # Built artifact (8 files, ~8.3 KB)
└── *.py                       # Ad-hoc Kaggle/GitHub submission helper scripts
```
