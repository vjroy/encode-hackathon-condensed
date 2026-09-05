# Submission: Gherkas

## Team

- Team name: Gherkas
- Members, one GitHub handle per line:
  - Lorcan7274
  - saimaanav
  - vjroy
  - ElieBen-Shlomo
- Repo URL:
https://github.com/ElieBen-Shlomo/encode-hackathon-condensed

## What we built and why

We built an agentic spreadsheet-editing harness around `Qwen/Qwen3.8-27B`. The entrypoint is `agent_predict.py`. For each task, the model receives the instruction, a compact workbook summary, the target answer range, and relevant source cells. It works in a loop of up to 20 turns, choosing one structured action at a time: inspect workbook data, run Python with openpyxl, run Bash, or recalculate formulas with LibreOffice. After an edit, the harness returns the current answer cells and relevant source data so the model can review its work and make a repair if needed.

We chose this approach because many sheet-level SpreadsheetBench tasks involve filtering, sorting, copying formatting, or writing thousands of cells. Asking a model to return every final value in one JSON response often runs out of space or produces invalid output; asking it to write a compact spreadsheet operation works better. This performed well on large transformations, but it can still fail when the model misreads a business rule or makes small boundary, formula, formatting, or case-sensitivity mistakes. We also ran a separate LoRA experiment on successful agent traces, but it did not improve measured spreadsheet accuracy, so the submitted pipeline uses the base model.

## Models

Inference model: `Qwen/Qwen3.8-27B`, accessed through Tinker.

Sampling used `qwen3_8_medium_reasoning`, 
temperature `0`, 
32768 output tokens per turn,
tools: Python, Bash, LibreOffice 

LoRA fine-tuning was an experiment and is not called by the submitted inference pipeline.

The separate LoRA experiment used 1,722 training action examples and 354 validation examples reconstructed from evaluated-correct agent traces. Golden workbook values were not included in prompts or training examples; golden workbooks were used only to identify successful traces. It used rank 16, learning rate 0.0001, batch size 4, two epochs (862 optimizer steps), and Tinker-managed compute.

## Scores on the 400

Produced by the shipped evaluator, nothing else:

```sh
uv run evaluate.py --predictions ../predictions.jsonl --all --out ../results.json
```

Results: **88%** pass rate

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

## Things to look at

- `agent/`: agent tool protocol, workbook inspection and editing, verification, and local Python/Bash execution.
- `research/baseline/agent_predict.py`: the Tinker-backed agent task loop used for the submitted run.
- `research/training/build_agent_sft_data.py`: construction of supervised fine-tuning examples from evaluated-correct agent traces.
- `research/training/train_lora.py` and `research/config/qwen_lora.yaml`: the separate LoRA experiment and its hyperparameters (rank 16, learning rate 0.0001, batch size 4, two epochs).
- `lora.log`: streamed progress log from the separate LoRA experiment. The experiment did not outperform the base-model agent, so its checkpoint is not used for submission.
