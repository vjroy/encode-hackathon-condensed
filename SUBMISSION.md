# Submission: Gherkas

## Team

- Team name: Gherkas
- Members, one GitHub handle per line: 
Lorcan7274
saimaanav
vjroy
ElieBen-Shlomo
- Repo URL:
https://github.com/ElieBen-Shlomo/encode-hackathon-condensed

## What we built and why

We built an agentic harness on top of Qwen-27B. This is a custom iterative loop (up to 20 iterations) in which
Qwen is given access to local executable tools (python, bash, LibreOffice) and tries to solve the prompt. At
each step, it inspects its own solution and determines if it is correct of needs further IO iterations. We then
 fine tuned Qwen-27B using LORA (through the entire agentic harness run) to get something even more accurate for our use case.

## Models

Inference model: `Qwen/Qwen3.8-27B`, accessed through Tinker.

Sampling used `qwen3_8_medium_reasoning`, 
temperature `0`, 
32768 output tokens per turn,
tools: Python, Bash, LibreOffice in the docker ctr

We also experimented with LoRA fine-tuning, but the submitted inference pipeline uses the base model rather than a fine-tuned checkpoint.

## Scores on the 400

Produced by the shipped evaluator, nothing else:

```sh
uv run evaluate.py --predictions ../predictions.jsonl --all --out ../results.json
```

Paste the `summary` block of `results.json` here and put the file in the repo. `items` must be 400.

```json
{
  "items": 400,
  "graded": 400,
  "missing": 0,
  "errors": 0,
  "pass_rate": 0.8825,
  "cell_accuracy": 0.8231,
  "pass_rate_cell_level": 0.88,
  "pass_rate_sheet_level": 0.888
}
```

## Your run on the 400

  In the repo:

  - `predictions.jsonl`: `predictions.jsonl` — 400 prediction records.
  - `outputs/`: `outputs/` — 400 completed `.xlsx` workbooks.
  - `traces/`: `traces/` — 400 per-task agent traces, containing model calls and tool activity.
  - `run.log`: `run.log` — unedited log from the 400-task agent run.

## Code

The submission pipeline is the Docker image defined in `Dockerfile`. Its entrypoint runs `research/baseline/agent_predict.py`, reads the judge-mounted dataset from `/data` read-only, and writes
`predictions.jsonl`, `outputs/`, `traces/`, and `run.log` to `/out`.

The agent uses `Qwen/Qwen3.8-27B` through Tinker and executes model-written Python, Bash, and LibreOffice commands only inside the container. The Python workbook edits use `openpyxl`; LibreOffice recalculates
formulas before output workbooks are saved.

Required environment variable:

- `TINKER_API_KEY`: API key for Tinker model inference.

