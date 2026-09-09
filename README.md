# Gherkers — SpreadsheetBench agent

An agentic spreadsheet-editing pipeline for [SpreadsheetBench Verified](https://huggingface.co/datasets/KAKA22/SpreadsheetBench): 400 real Excel-forum tasks where the input is a workbook plus a plain-English instruction, and the output is the workbook with the answer filled in.

Instead of asking a model to emit every final cell value as JSON, we give it tools and let it edit the workbook: inspect, write Python with `openpyxl`, run Bash, recalculate with LibreOffice, then look at what it actually produced and repair it.

**88.25% pass rate on all 400 tasks** with base `Qwen/Qwen3.8-27B` (no fine-tuning), scored by the shipped evaluator.

| Metric | Value |
|---|---|
| `pass_rate` | **0.8825** |
| `cell_accuracy` | 0.8231 |
| `pass_rate_cell_level` (275 tasks) | 0.880 |
| `pass_rate_sheet_level` (125 tasks) | 0.888 |
| graded / missing / errors | 400 / 0 / 0 |

For reference, one-shot prompting on the same 400: DeepSeek-V3.2 55.8%, Qwen3.8-27B 59.0%, Gemini 3.7 Flash 68.3%.

## How it works

`research/baseline/agent_predict.py` runs one agent loop per task, up to 20 turns. Each turn the model picks a single structured action:

| Action | What it does |
|---|---|
| `inspect_workbook` | Sheet names, dimensions, and an answer-aware digest of the workbook |
| `inspect_range` | Read values, formulas, and optionally styles for a range |
| `run_python` | Execute `openpyxl` code against a copy of the output workbook |
| `run_bash` | Shell commands inside the sandbox |
| `recalculate_workbook` | Headless LibreOffice recalculation — the same engine the grader uses |
| `assert_sorted`, `assert_blank` | Cheap deterministic post-conditions on a range |
| `finish` | Declare the task done; blocked if the output workbook is unreadable |

Three things make the loop converge rather than drift:

- **Answer-aware digest** (`agent/digest.py`). Small sheets are dumped whole; large sheets show the head plus a window around the answer range and state explicitly what was omitted. The model's own code still sees the full file.
- **Review after edit** (`agent/verify.py`). After every declared edit the harness replays a workbook diff, an answer/source snapshot, and auto-recalculates formula cells in the graded range, so the model reviews real post-edit state instead of its intent.
- **Task-shaped playbooks** (`agent/skills.py`). Short offline checklists for filter / sort / aggregate / delete / conditional edits, injected only when the task matches.

In the submitted pipeline, everything the model writes executes only inside the Docker container.

### Why these sampling settings

The settings in `research/config/qwen.yaml` come from a 26-configuration study on a fixed stratified 100-task dev split, scored with the shipped evaluator and compared with paired bootstraps over task ids (`research/experiments/representation_ablation.md`):

- `renderer: qwen3_8_medium_reasoning` — thinking off costs 22 points (CI [13, 31]); low/medium/adaptive land within a point of each other. `xhigh` matches medium at 2.5x the latency.
- `max_tokens: 32768` — every extra failure at an 8192-token cap was a reply truncated *while still thinking*. The Qwen3.8 renderers prefill the think tag, so a cut-off completion is reasoning text, not an answer. At 32k, nothing truncated.
- `fallback_renderers: auto` — a reply that hits the cap is re-sampled one thinking level lower (medium → low → off) instead of being handed to the JSON parser.
- `temperature: 0` — competition rule.

### LoRA experiment (not shipped)

We built 1,722 training and 354 validation action examples from evaluated-correct agent traces and fine-tuned with rank 16, lr 1e-4, batch size 4, 2 epochs (862 steps). Golden workbooks were used only to *select* successful traces — never in prompts or training examples. It did not beat the base-model agent, so the submitted pipeline samples the base model. Code in `research/training/`, log in `lora.log`, checkpoint commented out in `research/config/qwen.yaml`.

## Run it

### Docker (the submitted pipeline)

```sh
docker build -t gherkers .
docker run --rm \
  -e TINKER_API_KEY=... \
  -v "$PWD/research/data/spreadsheetbench_verified_400:/data:ro" \
  -v "$PWD/out:/out" \
  gherkers
```

Reads the dataset read-only from `/data`, writes `predictions.jsonl`, `outputs/`, `traces/` and `run.log` to `/out`. Extra flags pass through: `docker run ... gherkers --ids 13-1,51-12`.

`TINKER_API_KEY` is the only required environment variable. The Tinker `project_id` is hardcoded in `research/config/qwen.yaml` so the judges' run uses the same project and session setup we did.

### Locally

```sh
cd research
uv sync --extra tinker
uv run data/download.py                 # 15 MB dataset, SHA-256 checked
uv run baseline/agent_predict.py --out-dir submissions/qwen-agent --ids 13-1,51-12
```

Needs LibreOffice for recalculation (`brew install --cask libreoffice` / `apt install libreoffice-calc`; set `SOFFICE=` if `soffice` is elsewhere). Locally, model-written code runs on your host rather than in the container.

### Score

```sh
cd research
uv run evaluate.py --predictions ../predictions.jsonl --all --out ../results.json
uv run evaluate.py --oracle    # golden vs golden, must be 1.0
```

Only the cells in `answer_position` on `answer_sheet` are compared, after recalculation, with the official normalisation (numbers to 2 dp, dates as Excel serials, empty string equals empty cell). `--all` counts every task and treats a missing prediction as a failure — that is how the judges score.

### Tests

```sh
cd research && uv run pytest tests -q
```

Also run on every push and PR by `.github/workflows/tests.yml`.

## Repo layout

| Path | What |
|---|---|
| `agent/` | Tool protocol, workbook digest and inspection, verification, sandboxed Python/Bash execution |
| `research/baseline/agent_predict.py` | The Tinker-backed agent loop used for the submitted run |
| `research/baseline/llm_predict.py`, `tinker_predict.py` | One-shot baselines (OpenRouter / Tinker) |
| `research/config/qwen.yaml` | Inference config for the submitted pipeline |
| `research/config/qwen_lora.yaml`, `research/training/` | The LoRA experiment |
| `research/evaluate.py` | The shipped evaluator |
| `research/experiments/` | Representation study, per-run scoreboard, reading probes |
| `research/tests/` | Fast test suite: digest, harness (mock model), sandbox, scorer, verification, golden hygiene |
| `datasets/` | Split construction and the train/test id lists |
| `Dockerfile`, `docker_entrypoint.sh` | The submission container |

Artifacts from our full run on the 400 are committed at the repo root: `predictions.jsonl` (400 records), `outputs/` (400 `.xlsx`), `traces/` (400 per-task traces of every model and tool call), `run.log` (unedited), `results.json` (evaluator output).

## More

- [`SUBMISSION.md`](SUBMISSION.md) — the hackathon submission write-up.
- [`research/README.md`](research/README.md) — benchmark, dataset layout, scoring, and baselines in detail.
- [`research/experiments/representation_ablation.md`](research/experiments/representation_ablation.md) — the study behind the sampling settings.

Built at the Ylookup x Encode AI Hackathon, 5–6 September 2026, Encode Hub, Shoreditch. Team **Gherkers**: [@Lorcan7274](https://github.com/Lorcan7274), [@saimaanav](https://github.com/saimaanav), [@vjroy](https://github.com/vjroy), [@ElieBen-Shlomo](https://github.com/ElieBen-Shlomo).
