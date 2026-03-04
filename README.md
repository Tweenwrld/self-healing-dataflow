# Self-Healing Dataflow

A production-ready Apache Airflow pipeline for sentiment analysis on large-scale review datasets with built-in data quality diagnosis and automatic healing.

[![Python](https://img.shields.io/badge/Python-3.8+-blue.svg)](https://www.python.org/)
[![Airflow](https://img.shields.io/badge/Airflow-3.0+-017CEE.svg)](https://airflow.apache.org/)
[![OLLAMA](https://img.shields.io/badge/OLLAMA-llama3.2-000000.svg)](https://ollama.ai/)

## What It Does

This pipeline processes Yelp review data and performs sentiment classification while automatically detecting and fixing common data quality issues:

- **Self-healing ingestion**: Detects malformed, missing, or invalid text fields and applies normalization rules
- **Local LLM inference**: Uses OLLAMA (llama3.2) for privacy-preserving, cost-free sentiment analysis
- **Health monitoring**: Tracks success/healing/degradation rates and reports pipeline health status
- **Batch processing**: Configurable batch sizes with offset-based windowing for controlled execution
- **Comprehensive metrics**: Generates detailed JSON reports with sentiment distributions, confidence scores, and quality statistics

## Quick Start

### Prerequisites

- Python 3.8+
- [OLLAMA](https://ollama.ai/) installed and running
- Apache Airflow 3.0+

### Installation

```bash
# Clone the repository
cd self-healing-dataflow

# Create and activate virtual environment
python -m venv venv
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Initialize Airflow
export AIRFLOW_HOME=$PWD
airflow db migrate
airflow dags reserialize
```

### Run the Pipeline

```bash
# Start OLLAMA (in a separate terminal)
ollama serve
ollama pull llama3.2

# Start Airflow
airflow standalone

# Trigger the pipeline (in another terminal)
source venv/bin/activate
export AIRFLOW_HOME=$PWD
airflow dags unpause self_healing_pipeline
airflow dags trigger self_healing_pipeline --conf '{"batch_size": 100, "offset": 0}'
```

### Check Results

Output files are written to `output/` with naming pattern:

```
sentiment_analysis_summary_<timestamp>_Offset<offset>.json
```

## Key Features

### Self-Healing Rules

The pipeline automatically handles:

- Missing or `null` text fields → placeholder insertion
- Wrong data types → safe string conversion
- Empty/whitespace-only text → placeholder replacement
- Special characters only → safe marker substitution
- Overlong text → truncation with ellipsis

### Health Status Classification

Each run receives a health status based on data quality:

- **HEALTHY**: Pipeline processed data with minimal issues
- **WARNING**: High healing rate (>50%) but no failures
- **DEGRADED**: Some records failed to process properly
- **CRITICAL**: Significant failures (>10% degradation rate)

### Output Metrics

Every run generates comprehensive metrics including:

- Success/healing/degradation rates
- Sentiment distribution (positive/negative/neutral)
- Star rating correlation with sentiment
- Average confidence scores by status
- Per-record healing actions and error types

## Configuration

Key environment variables:

| Variable                | Default                    | Description             |
| ----------------------- | -------------------------- | ----------------------- |
| `PIPELINE_BASE_DIR`   | Current directory          | Project root path       |
| `PIPELINE_INPUT_FILE` | `input/input.json`       | Input data file         |
| `PIPELINE_OUTPUT_DIR` | `output/`                | Results directory       |
| `OLLAMA_HOST`         | `http://localhost:11434` | OLLAMA service endpoint |
| `OLLAMA_MODEL`        | `llama3.2`               | Model for inference     |

Runtime parameters can be passed via `--conf`:

```bash
airflow dags trigger self_healing_pipeline --conf '{
  "batch_size": 500,
  "offset": 1000,
  "ollama_model": "llama3.2"
}'
```

## Architecture

The pipeline implements six Airflow tasks in sequence:

```
load_model → load_reviews → diagnose_and_heal_batch → 
batch_analyze_sentiment → aggregate_results → generate_health_report
```

Each task is isolated, retryable, and logged independently for maximum observability.

## Documentation

For comprehensive technical details, see **[Technical Guide](TECHNICAL_GUIDE.md)**:

- Complete architecture and design decisions
- Detailed setup and configuration instructions
- Full debugging history with root cause analysis
- Operations, troubleshooting, and performance tuning
- Known caveats and improvement roadmap

## Use Cases

- **Data quality experimentation**: Test resilience to real-world messy data
- **Local NLP development**: No external API costs or data privacy concerns
- **Pipeline health monitoring**: Learn observability patterns for production ML workflows
- **Batch processing patterns**: Reference implementation for windowed data processing

## Command Reference

### Common Troubleshooting Commands

```bash
# Check DAG status
airflow dags list

# View all DAG runs and their states
airflow dags list-runs self_healing_pipeline

# Unpause a DAG (required before it can run)
airflow dags unpause self_healing_pipeline

# Check if specific processes are running
ps aux | grep -E "airflow (standalone|scheduler|webserver)" | grep -v grep

# View latest task logs
tail -f logs/dag_id=self_healing_pipeline/run_id=<run_id>/task_id=<task_id>/attempt=1.log
```

## Requirements

See [requirements.txt](requirements.txt) for full dependency list. Core components:

- `apache-airflow>=3.0.6`
- `transformers`
- `torch`
- `ollama>=0.6.0`

## Contributing

This is currently an internal experimentation project. For production use, consider:

- Adding unit/integration tests
- Implementing idempotency controls
- Externalizing configuration to environment profiles
- Adding structured logging and alerting

## License

Internal use only.
