# SMS Spam Classification Pipeline

This project builds and evaluates a text classifier that labels SMS messages as `ham` or `spam`. DVC manages the reproducible pipeline and its generated data and model artifacts; DVCLive records evaluation metrics and parameters.

## Pipeline

Run the pipeline from the `DVC-end2end` project root. The stages are declared in `dvc.yaml` and execute in this order:

1. **Data ingestion** (`src/data_ingestion.py`) reads `experiments/spam.csv`, renames its `v1` and `v2` columns to `target` and `text`, and creates train/test splits. The split size is controlled by `data_ingestion.test_size`.
2. **Text preprocessing** (`src/data_preprocessing.py`) encodes labels, removes duplicate rows, lowercases and tokenizes message text, removes stopwords and punctuation, and stems tokens.
3. **Feature engineering** (`src/feature_engineering.py`) fits a TF-IDF vectorizer on the training messages and transforms both splits. `feature_engineering.max_features` sets the vocabulary limit.
4. **Model building** (`src/model_building.py`) trains a `RandomForestClassifier`. Its tree count and random seed are controlled by `model_building.n_estimators` and `model_building.random_state`.
5. **Model evaluation** (`src/model_evaluation.py`) evaluates the held-out test split and writes accuracy, precision, recall, and ROC AUC metrics.

## Setup

Use Python 3.12 or a compatible Python version. From this directory, create and activate a virtual environment.

Windows PowerShell:

```powershell
py -3.12 -m venv .venv
.\.venv\Scripts\Activate.ps1
```

macOS or Linux:

```sh
python3 -m venv .venv
source .venv/bin/activate
```

Install the runtime and DVC packages:

```sh
python -m pip install --upgrade pip
python -m pip install pandas scikit-learn nltk PyYAML dvclive dvc
```

The first preprocessing run downloads the NLTK `stopwords`, `punkt`, and `punkt_tab` data packages, so network access is needed once. Run commands with the project root as the current directory so relative paths such as `experiments/spam.csv` and `params.yaml` resolve correctly.

## Run

To execute only stages whose inputs or parameters have changed:

```sh
dvc repro
```

To run the stages directly in order:

```sh
python src/data_ingestion.py
python src/data_preprocessing.py
python src/feature_engineering.py
python src/model_building.py
python src/model_evaluation.py
```

Useful DVC commands:

```sh
dvc dag
dvc status
dvc metrics show
```

## Parameters

Edit `params.yaml` to change the current defaults:

| Parameter | Default | Purpose |
| --- | ---: | --- |
| `data_ingestion.test_size` | `0.25` | Fraction of input rows assigned to the test split |
| `feature_engineering.max_features` | `40` | Maximum number of TF-IDF features |
| `model_building.n_estimators` | `24` | Number of trees in the random forest |
| `model_building.random_state` | `2` | Random seed for model training |

After changing a parameter, run `dvc repro` to rebuild affected stages.

## Outputs

Generated data and model directories are ignored by Git and managed as DVC outputs where declared in `dvc.yaml`.

| Path | Contents |
| --- | --- |
| `data/raw/` | Ingested `train.csv` and `test.csv` splits |
| `data/interim/` | Preprocessed train and test CSV files |
| `data/processed/` | TF-IDF train and test feature CSV files |
| `models/model.pkl` | Trained random forest model |
| `reports/metrics.json` | Accuracy, precision, recall, and ROC AUC |
| `dvclive/` | Logged metrics, parameters, and metric plots |
| `logs/` | Per-stage execution logs |

## DVC Remote Storage

The repository has a default S3 remote configured in `.dvc/config`. Local pipeline runs do not need S3 access when the required data is present locally. To pull from or push to the remote, install the S3-enabled DVC package and configure AWS credentials outside the repository:

```sh
python -m pip install "dvc[s3]"
```

Do not commit AWS credentials or other secrets. The evaluation script writes local DVCLive metrics without automatically saving a DVC experiment. To record DVC experiments against this S3-configured repository, install the S3-enabled package and configure AWS credentials first, then use `dvc exp run`.

## Project Layout

```text
DVC-end2end/
├── data/                 # DVC-managed pipeline data (generated)
├── dvclive/              # DVCLive metrics, params, and plots
├── experiments/
│   ├── mynotebook.ipynb  # Notebook-based exploration
│   └── spam.csv          # Source SMS dataset used by ingestion
├── models/               # Trained model artifact (generated)
├── reports/              # Evaluation metrics (generated)
├── src/                  # Pipeline stage implementations
├── dvc.yaml              # DVC stage definitions
└── params.yaml           # Pipeline parameters
```
