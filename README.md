# News Temporal Extraction and Summarization

A modular NLP pipeline for extracting temporal expressions from news articles,
normalizing them into structured TimeML-style dates, resolving event
dependencies, and generating a chronological summary.

Unlike a plain article summarizer, this project focuses on the timeline behind a
news story. It identifies phrases such as `three days ago`, `within two hours of
the explosion`, `the following morning`, and `five days later`, resolves them
against verified anchor dates, then produces a date-aware event summary.

## Highlights

- BERT + BiLSTM + CRF named-entity recognizer for temporal expression extraction.
- Rule-first TimeML normalization before any LLM call.
- LLM validation layer for missed or partially captured temporal spans.
- Dependency graph construction for relative expressions that depend on earlier
  events.
- Topological layer execution so later dates are resolved only after their
  anchors are verified.
- Anchor registry with schema validation to avoid poisoning the pipeline with
  invalid LLM output.
- Conflict resolution, deduplication, fallback handling, and chronological
  summary generation.
- Interactive terminal app, command-line execution, and importable Python API.

## Problem It Solves

News articles often describe events with relative time expressions:

```text
A bomb blast occurred three days ago. Rescue workers were deployed within two
hours of the explosion. The investigation will start after 5 days of the bomb
blast.
```

A normal summarizer may preserve these phrases, but it does not understand the
underlying timeline. This project extracts the temporal expressions, resolves
relative dates against verified anchors, and emits a structured timeline that can
be used for downstream summaries, dashboards, or event intelligence systems.

## Architecture

```text
Input paragraph
      |
      v
Sentence splitter
      |
      v
BERT + BiLSTM + CRF NER
      |
      v
LLM NER validator
      |
      v
Rule-first TimeML normalizer
      |
      v
Dependency graph builder
      |
      v
Topological layer executor
      |
      v
Anchor registry + conflict resolver
      |
      v
Chronological summary
```

## Pipeline Stages

| Stage | Module | Purpose |
| --- | --- | --- |
| 1 | `core/predict.py` | Loads the NER model and extracts temporal entities from each sentence. |
| 1.5 | `llm/llm_ner_validator.py` | Uses an LLM to recover missed spans and fix partial spans such as `two days` -> `two days later`. |
| 2 | `pipeline/stages/rule_normalize.py` | Applies deterministic TimeML-style rules to every entity before any LLM arithmetic. |
| 3 | `pipeline/stages/graph_builder.py` | Builds a dependency graph for expressions that depend on previous events. |
| 4 | `pipeline/stages/graph_builder.py` | Assigns topological layers so dependencies are resolved in order. |
| 5 | `pipeline/stages/layer_executor.py` | Resolves each layer using fast-path rules or LLM date arithmetic with verified anchors only. |
| 6 | `pipeline/stages/conflict_resolver.py` | Deduplicates entities and flags conflicting date resolutions. |
| 7 | `llm/llm_summarize.py` | Produces a chronological bullet summary from the resolved timeline. |

## Repository Structure

```text
.
|-- app.py
|-- requirements.txt
|-- README.md
|-- core
|   |-- __init__.py
|   |-- anchor_registry.py
|   |-- model.py
|   `-- predict.py
|-- llm
|   |-- __init__.py
|   |-- llm_ner_validator.py
|   `-- llm_summarize.py
`-- pipeline
    |-- __init__.py
    |-- pipeline.py
    `-- stages
        |-- __init__.py
        |-- conflict_resolver.py
        |-- graph_builder.py
        |-- layer_executor.py
        `-- rule_normalize.py
```

## Core Design Ideas

### Rule-first normalization

Every extracted entity is passed through the deterministic rule engine first.
Clear expressions such as `yesterday`, `next week`, `three days ago`, `at 9 PM`,
or `every Monday morning` are resolved without involving an LLM.

### LLMs only handle bounded tasks

The LLM is not allowed to freely guess dates. It is used for two controlled
tasks:

1. Finding missed or partially captured temporal expressions.
2. Performing coreference/date arithmetic only with already verified anchor
   dates.

### Verified anchor registry

The `AnchorRegistry` stores resolved dates by entity ID and sentence index. Each
write is validated as a real `YYYY-MM-DD` date before it becomes available to
later layers.

### Dependency-aware execution

Expressions like `two hours later` or `after 5 days of the bomb blast` depend on
earlier events. The graph builder connects those expressions to their anchor
events and the layer executor resolves them in topological order.

## Tech Stack

- Python
- PyTorch
- Hugging Face Transformers
- BERT
- BiLSTM
- CRF
- python-dateutil
- Ollama for optional local LLM stages

## Installation

Use Python 3.10 or newer.

```bash
git clone https://github.com/Rammanideep-ghantasala/News_Article_Summarizer.git
cd News_Article_Summarizer

python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

On Windows PowerShell:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

## Model Checkpoint

The NER model checkpoint is expected at the project root:

```text
tempeval3_ner_final.pt
```

The original README references this Hugging Face model location:

```text
https://huggingface.co/ManojBudim/temporal_extraction
```

Download the checkpoint and place it in the repository root before running the
pipeline.

## Optional LLM Setup

LLM-backed stages use Ollama. The pipeline still runs with fallback behavior when
Ollama is unavailable, but Stage 1.5 validation, LLM-based date arithmetic, and
LLM summarization work best with a local model installed.

Install Ollama, then pull a model:

```bash
ollama pull mistral
```

The default LLM model name used by the app is `mistral`.

## Usage

### Interactive terminal

```bash
python app.py
```

With custom model and LLM names:

```bash
python app.py --model tempeval3_ner_final.pt --llm mistral
```

Interactive commands:

| Command | Action |
| --- | --- |
| `<paragraph>` | Run the complete pipeline on pasted text. |
| `demo 0` to `demo 4` | Run one of the built-in demo articles. |
| `json` | Toggle JSON output mode. |
| `quit` | Exit the app. |

### Run one paragraph from the command line

```bash
python -m pipeline.pipeline --text "The earthquake struck yesterday. Rescue teams arrived three hours later."
```

Return compact JSON output:

```bash
python -m pipeline.pipeline --text "The wildfire started Monday. It was contained five days later." --json
```

### Run a built-in demo

```bash
python -m pipeline.pipeline --demo 0
```

Available demos:

| Demo | Scenario |
| --- | --- |
| 0 | Blast, rescue, and investigation timeline. |
| 1 | Wildfire progression and containment. |
| 2 | Political crisis with mixed temporal dependencies. |
| 3 | Flood response with dense temporal chain. |
| 4 | Warehouse fire with investigation and report timeline. |

## Python API

```python
from core.predict import load_model
from pipeline.pipeline import run_pipeline

load_model("tempeval3_ner_final.pt")

result = run_pipeline(
    "The wildfire started Monday. Firefighters arrived three hours later.",
    llm_model="mistral",
)

print(result["summary"])

for entity in result["entities"]:
    print(entity["text"], entity["absolute_date"], entity["method"])
```

## Output Format

`run_pipeline()` returns a dictionary:

```python
{
    "paragraph": "original input paragraph",
    "sentences": ["sentence 1", "sentence 2"],
    "entities": [
        {
            "entity_id": 0,
            "sentence_idx": 0,
            "text": "three days ago",
            "type": "DATE",
            "value": "2026-04-15",
            "absolute_date": "2026-04-15",
            "end_date": None,
            "confidence": 1.0,
            "method": "rule",
            "conflict_flag": False
        }
    ],
    "summary": "- Event happened three days ago (2026-04-15).",
    "timeline": []
}
```

Entity fields:

| Field | Description |
| --- | --- |
| `entity_id` | Unique ID assigned during pipeline execution. |
| `sentence_idx` | Index of the source sentence. |
| `text` | Original temporal expression span. |
| `type` | Entity type: `DATE`, `TIME`, `DURATION`, `SET`, or `EVENT`. |
| `value` | Normalized TimeML-style value. |
| `absolute_date` | Resolved date in `YYYY-MM-DD`, or `RECURRING` for repeated events. |
| `end_date` | End date for duration expressions, when available. |
| `confidence` | Confidence score assigned by rule, LLM, or fallback path. |
| `method` | Resolution source: `rule`, `llm`, or `fallback`. |
| `conflict_flag` | Whether conflicting resolutions were detected. |

## Example

Input:

```text
A major fire broke out in the warehouse three days ago. Rescue teams arrived two
hours later and controlled the situation by midnight. The investigation began the
following morning and continued for four days.
```

Expected behavior:

1. Extracts `three days ago`, `two hours later`, `midnight`,
   `the following morning`, and `four days`.
2. Resolves the first date using rules.
3. Uses that verified date as an anchor for later relative expressions.
4. Produces a chronological event summary.

## Module Guide

### `core/model.py`

Defines the neural NER model:

- `bert-base-uncased` encoder
- two-layer bidirectional LSTM
- dropout
- linear classifier
- CRF decoding layer

### `core/predict.py`

Loads the checkpoint, tokenizes sentences, runs inference, and groups BIO labels
into temporal entity spans.

### `core/anchor_registry.py`

Stores resolved dates and validates every write. Invalid dates are rejected so
later stages do not build on bad anchors.

### `pipeline/stages/rule_normalize.py`

Contains the deterministic rule engine for:

- relative dates
- weekdays and months
- durations
- time-of-day expressions
- recurring schedules
- cross-dependency hints

### `pipeline/stages/graph_builder.py`

Builds a dependency graph from anchor-dependent entities and assigns layer
numbers using topological sorting.

### `pipeline/stages/layer_executor.py`

Executes each dependency layer. Rule-resolved layers use the fast path. Mixed
layers call the LLM only with verified anchor dates from the registry.

### `pipeline/stages/conflict_resolver.py`

Merges duplicate entities, keeps the highest-confidence result, and flags date
conflicts.

### `llm/llm_ner_validator.py`

Checks whether the NER model missed or partially captured temporal spans. It is
especially useful for phrases where context matters, such as `within two hours of
the explosion`.

### `llm/llm_summarize.py`

Builds a compact timeline and asks the LLM to generate chronological news-style
bullet points. It includes fallback summary generation if the LLM is unavailable.

## Reliability Features

- Fast path for fully rule-resolved layers.
- LLM schema validation for JSON outputs.
- Fallback behavior when Ollama is missing or an LLM call fails.
- Date format validation before writing to the anchor registry.
- Conflict detection for duplicate spans with different resolved dates.
- Chronological sorting of dated and recurring entities.

## Current Limitations

- The project currently runs as a terminal/CLI pipeline, not a web API.
- The model checkpoint is required but is not included in this repository.
- LLM behavior depends on the local Ollama model used.
- The graph builder uses sentence-level heuristics to infer anchor events.
- There is no automated test suite in the repository yet.

## Roadmap

- Add unit tests for rule normalization, graph building, and conflict handling.
- Add a FastAPI service around `run_pipeline()`.
- Add Docker support for reproducible local execution.
- Add benchmark scripts for temporal extraction accuracy and latency.
- Improve anchor selection with semantic event matching.
- Add structured export formats such as CSV, JSONL, and timeline visualizations.

## Skills Demonstrated

`Python` `PyTorch` `Transformers` `NLP` `NER` `TimeML` `LLM Workflows`
`Pipeline Design` `Graph Algorithms` `Topological Sorting` `Data Validation`
`Fallback Handling` `Prompt Engineering` `Temporal Reasoning`

