# LongMemEval Experiment

This project packages the LongMemEval source code with a Kaggle-ready reproduction notebook:

- `longmemeval-kaggle-reproduce-pipeline.ipynb` runs a smoke-test pipeline on Kaggle.
- `src/` contains the LongMemEval retrieval, generation, and evaluation code.
- `data/custom_history/` keeps the original helper for custom history compilation.

The default notebook run is intentionally small: 20 examples, CPU BM25 retrieval, and API-based generation/evaluation. It is meant to verify that the pipeline, secrets, endpoint, and output format work before running the full benchmark.

## Reference and Attribution

This project is derived from the official LongMemEval repository:

- Original repository: https://github.com/xiaowu0162/LongMemEval
- Original benchmark paper: `LongMemEval: Benchmarking Chat Assistants on Long-Term Interactive Memory`
- Local source baseline used when this project was created: upstream `main` commit `9e0b455`

All original benchmark design, dataset format, baseline code, and research credit belong to the original LongMemEval authors. This repository adds Kaggle-oriented execution, compatibility, and safety adjustments around the original code.

## What Changed

The Kaggle-facing changes are intentionally narrow:

- The notebook is cleared of outputs so it does not store API responses or secrets.
- `run_generation.py` no longer prints API keys in `Namespace(...)` output.
- `run_generation.py` can read `OPENAI_API_KEY` from the environment, so the notebook does not pass secrets as command-line arguments.
- OpenAI-compatible routers are supported through `OPENAI_BASE_URL`, `OPENAI_DEFAULT_HEADERS`, `GEN_MODEL_NAME`, and `METRIC_MODEL_NAME`.
- `evaluate_qa.py` can use a custom metric model alias via `METRIC_MODEL_NAME`.
- `eval_utils.py` uses `np.asarray(..., dtype=float)` instead of removed `np.asfarray`, which keeps retrieval metrics compatible with NumPy 2.x.
- The notebook retrieval summary follows the reporting rule in `run_retrieval.py`: skip abstention items and items without user-side target labels.

## Kaggle Notebook Usage

Open `longmemeval-kaggle-reproduce-pipeline.ipynb` in Kaggle and enable Internet unless you attach the benchmark JSON files as a Kaggle Dataset.

Set Kaggle Secrets:

- `OPENAI_API_KEY`: required for generation and QA evaluation.
- `OPENAI_BASE_URL`: required only when using a non-OpenAI router.
- `OPENAI_ORGANIZATION`: optional.

Default runtime settings:

- `LONGMEMEVAL_DATASET=longmemeval_s_cleaned.json`
- `RUN_FULL=0`
- `N_EXAMPLES=20`
- `RETRIEVER=flat-bm25`
- `GRANULARITY=session`
- `TOPK_CONTEXT=50`
- `GEN_MODEL_NAME=gpt-4o-mini-2024-07-18`
- `GEN_MODEL_ALIAS=gpt-4o-mini`
- `METRIC_MODEL_SHORT=gpt-4o-mini`
- `METRIC_MODEL_NAME=gpt-4o-mini-2024-07-18`

For an OpenAI-compatible router, set the model variables in the notebook configuration cell or as environment variables. Example:

```text
OPENAI_BASE_URL=https://your-router.example/v1
GEN_MODEL_NAME=cx/gpt-5.2
GEN_MODEL_ALIAS=router-gpt-5.2
METRIC_MODEL_SHORT=router-gpt-5.2
METRIC_MODEL_NAME=cx/gpt-5.2
TOKENIZER_BACKEND=openai
MODEL_MAX_LENGTH=128000
```

To run the full benchmark, set:

```text
RUN_FULL=1
N_EXAMPLES=500
```

Full runs cost more because both generation and LLM-as-judge evaluation call an API for every example.

## Data

The notebook downloads the official cleaned LongMemEval files from Hugging Face if they are not attached under `/kaggle/input`:

- `longmemeval_oracle.json`
- `longmemeval_s_cleaned.json`
- `longmemeval_m_cleaned.json`

For cheaper first checks, use `longmemeval_oracle.json`. For the default BM25 RAG smoke test, use `longmemeval_s_cleaned.json`.

## Local Commands

Evaluation-only setup:

```bash
pip install -r requirements-lite.txt
```

Full retrieval/generation support requires the heavier dependencies in `requirements-full.txt` plus the PyTorch/CUDA setup described by the upstream project.

Baseline retrieval:

```bash
cd src/retrieval
bash run_retrieval.sh ../../data/longmemeval_s_cleaned.json flat-bm25 session
```

Retrieval-augmented generation:

```bash
export OPENAI_API_KEY=...
cd src/generation
bash run_generation.sh ../../retrieval_logs/flat-bm25/session/longmemeval_s_cleaned.json_retrievallog_session_flat-bm25 gpt-4o-mini flat-bm25-session 50 json false con
```

QA evaluation:

```bash
export OPENAI_API_KEY=...
cd src/evaluation
python evaluate_qa.py gpt-4o-mini HYPOTHESIS_FILE ../../data/longmemeval_s_cleaned.json
```

## Notes on Comparability

The default notebook is a smoke test, not a paper-comparable run. For paper-style comparison, use the full dataset, the same retrieval configuration, and the same evaluator model as the original setup, typically GPT-4o for QA judging.

Index expansion and time-aware pruning are still available in `src/retrieval` and `src/index_expansion`, but they require additional released cache artifacts from the upstream project.
