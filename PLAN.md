# DetectForge: 12-Hour Hackathon Plan (team of 4)

> **One-liner:** DetectForge turns a threat description into a **validated Sigma detection rule**.
> An open-weight LLM drafts the rule, we test it against labeled attack and benign Windows logs,
> and the model refines it until it catches the attack without false positives.
> Packaged as an **Agent Skill** that follows the Agent Skills Open Standard.

**Challenge:** DigitalOcean x MLH, Hacktoberfest 2026 (open-source / open-weight AI)

## How we meet the challenge rules

| Rule | Our answer |
|---|---|
| Open-weight AI is an important part | The open-weight model writes and refines every rule. Without it there is no product. |
| Public GitHub repo + open-source license | This repo, MIT |
| Agent Skill complies with the Open Standard | `skills/detectforge/SKILL.md`, checked with `agentskills validate` in CI |
| Original harness (bonus) | Our own draft → validate → test → refine loop with failure feedback |

## How it works

```
threat description
      │
      ▼
 open-weight model ──draft──► pySigma syntax check ──► evaluator (attack + benign logs)
      ▲                                                        │
      └──────── feedback: "missed event X" / "FP on event Y" ◄─┘   (max N rounds)
                                                               │
                                                               ▼
                                                  final rule + TP/FP report
```

- **Model:** open-weight model on NVIDIA NIM (OpenAI-compatible API); local **Ollama** (`qwen2.5-coder`) as the fallback
- **Rule language:** Sigma only (YARA is a stretch goal)
- **Logs:** Windows Security / Sysmon EVTX from public datasets (EVTX-ATTACK-SAMPLES, OTRF Security-Datasets). **We do not generate attacks ourselves.**
- **Scope:** 5–8 ATT&CK techniques, for example PowerShell download cradle, suspicious service creation, LSASS access, scheduled-task persistence, rundll32 abuse

## Team lanes

| Person | Lane | Owns | Load |
|---|---|---|---|
| **A** | Forge loop (core) | `forge/llm.py`, `forge/loop.py`: prompts, NIM/Ollama client, draft → validate → refine loop, retries | Heaviest |
| **B** | Evaluator | `forge/evaluate.py`: Sigma rule → matcher over EVTX events, TP/FP counts, failed-event extraction | Heavy |
| **C** | Data + skill | `corpus/`: labeled attack/benign events per technique; `skills/detectforge/` (SKILL.md, references, scripts); CLI | Medium |
| **D** | Benchmark + demo | `bench/`: metrics across techniques (one-shot vs refined), README, slides, 3-min video, submission | Medium |

**Interfaces (agree these at H0, then everyone can stub the others):**

```python
# B provides
evaluate(rule_yaml: str, technique: str) -> EvalResult
# EvalResult: tp, fp, fn, missed_events: list[dict], false_positive_events: list[dict], error: str | None

# A provides
forge(description: str, technique: str, max_rounds: int = 4) -> ForgeResult
# ForgeResult: rule_yaml, rounds: list[EvalResult], final: EvalResult, model: str

# C provides
load_corpus(technique: str) -> {"attack": list[dict], "benign": list[dict]}
```

## Hour-by-hour

### H0–H1: Kickoff (everyone)
- Read this plan, agree on the interfaces above, create branches
- Everyone sets up: Python 3.11+, `pip install pysigma openai python-evtx`, NIM key in `.env`
- C confirms which dataset files we are using; D sets up the repo board (issues per lane)

### Person A: Forge loop
| Hour | Task |
|---|---|
| H1–H2 | NIM client (OpenAI SDK, base URL + key from `.env`); first draft prompt that returns **only** Sigma YAML |
| H2–H3 | pySigma syntax validation; on parse error, feed the error back and retry (max 2) |
| H3–H4 | Wire up `forge()` against a **stub** `evaluate()` until B's lands |
| H4–H6 | Refine loop: turn `missed_events` and `false_positive_events` into short feedback prompts; stop when FN=0 and FP=0 or after `max_rounds` |
| H6 | ✅ Checkpoint 2: `forge()` runs end-to-end on 2 techniques with B's real evaluator |
| H6–H8 | Ollama fallback (`qwen2.5-coder`), provider switch flag, log every round to JSON for D |
| H8–H9 | Tune prompts with D's metrics; guard against over-broad rules (penalise FP in feedback) |
| H9–H10.5 | Bug fixes, help C with the skill script |
| H10.5–H12 | Code freeze → demo dry-run → submit |

### Person B: Evaluator
| Hour | Task |
|---|---|
| H1–H2 | Load EVTX → list of flat event dicts (`python-evtx` or pre-converted JSON from the dataset) |
| H2–H4 | Sigma matcher: parse rule with pySigma, support `selection` fields, `contains` / `startswith` / `endswith` / `re` modifiers, `and` / `or` / `not` in `condition` |
| H4 | ✅ Checkpoint 1: `evaluate()` returns correct TP/FP for 2 hand-written Sigma rules |
| H4–H6 | Return `missed_events` and `false_positive_events` (trimmed to key fields so prompts stay small) |
| H6–H8 | Support more modifiers / `1 of selection*` / `all of`; unit tests with known rules from the official Sigma repo |
| H8–H9 | Speed (cache parsed corpus), clear error messages for unsupported rule features |
| H9–H10.5 | Bug fixes from D's benchmark runs |
| H10.5–H12 | Code freeze → demo dry-run → submit |

### Person C: Data + skill
| Hour | Task |
|---|---|
| H1–H3 | Pick 5–8 techniques; pull attack samples from EVTX-ATTACK-SAMPLES; pull benign events (normal logons, service starts, PowerShell use) from OTRF / sample logs |
| H3–H4 | Save as `corpus/<technique>/attack.jsonl` + `benign.jsonl` + `description.txt`; implement `load_corpus()` |
| H4 | ✅ Checkpoint 1: corpus for at least 3 techniques is ready for B and A |
| H4–H6 | Finish remaining techniques; at least 100 benign events per technique so FP numbers mean something |
| H6–H8 | `skills/detectforge/`: `SKILL.md` (frontmatter: name, description, license), `scripts/forge.py`, `references/SIGMA.md` cheatsheet; pass `agentskills validate` |
| H8–H9 | CLI: `detectforge forge "<description>" --technique X` and `detectforge bench` |
| H9–H10.5 | Test the skill inside an agent (Claude Code / OpenCode); fix wording in SKILL.md |
| H10.5–H12 | Code freeze → demo dry-run → submit |

### Person D: Benchmark + demo
| Hour | Task |
|---|---|
| H1–H3 | Repo hygiene: README skeleton, LICENSE, CI (lint, tests, `agentskills validate`); issue board |
| H3–H6 | `bench/run.py`: for each technique run **one-shot** (no refine) vs **forge loop**; record precision, recall, rounds, latency, model |
| H6 | ✅ Checkpoint 2: first benchmark table (even if numbers are bad) |
| H6–H8 | Second model comparison (NIM model vs local Ollama); save example rules to `docs/examples/` |
| H8–H9 | Slides (problem → how it works → live demo → results table → what's next) |
| H9 | ✅ Checkpoint 3: feature freeze candidate, final benchmark run, README results section |
| H9–H10.5 | Record 3-min demo video; pre-record a backup run in case NIM is down |
| H10.5–H12 | Code freeze → demo dry-run → **submit** (repo link, video, description) |

## Checkpoints

| When | What must be true |
|---|---|
| **H4** | B's evaluator works on hand-written rules; C has 3 techniques of data; A drafts valid YAML |
| **H6** | End-to-end: description → refined rule → TP/FP report, on 2 techniques. First benchmark table. **Cut scope here if behind.** |
| **H9** | All techniques run; README has results; skill validates; feature freeze candidate |
| **H10.5** | 🧊 Code freeze. Only fixes for demo-breaking bugs. |

## Scope cuts (in this order, decided at H6)

1. YARA support (already a stretch goal)
2. Ollama vs NIM comparison (keep NIM only, keep Ollama as fallback)
3. Reduce to 4 techniques
4. Drop `1 of` / `all of` support in the matcher

## Demo script (3 min)

1. **Problem (20s):** Writing and tuning detection rules is slow, skilled work.
2. **Live (90s):** "Detect PowerShell downloading and running a remote script" → draft rule → 3 false positives on benign PowerShell → model refines → 4/4 attack events caught, 0 FPs.
3. **Results (40s):** Table showing one-shot vs refined loop across all techniques.
4. **Skill (30s):** The same thing run inside an agent through the DetectForge skill.

## Risks

| Risk | Mitigation |
|---|---|
| Corpus takes too long | Use pre-labeled EVTX-ATTACK-SAMPLES as-is; no hand-crafted logs |
| Sigma matcher is fiddly | B starts on it first; support only the modifiers our techniques need |
| Model writes an over-broad rule to catch everything | The benign corpus punishes it with FPs; feedback says so |
| NIM rate limits / outage | Ollama fallback + pre-recorded demo run |
| Merge conflicts late | One owner per folder; PRs only; no pushes to `main` after H10.5 |

## Git workflow

- Branches: `a/forge-*`, `b/eval-*`, `c/data-*`, `d/bench-*`
- Small PRs → 1 review → squash merge; CI must be green
- Never commit API keys; `.env` is gitignored
