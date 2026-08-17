# Curriculum-Industry Skill Feature Store Using Feast

> **Assignment:** MLOps – Curriculum-Industry Skill Feature Store Using Feast  
> **Important:** The dataset included here is an **illustrative starter dataset** because the previous-activity dataset was not included with the assignment document. Replace `data/skill_gap_raw.csv` with your own previous-activity dataset before final submission and update the student details/results below.

## Student Details

- **Name:** Prem Sagar
- **Register Number:** `231FA04G39`
- **Section:** `03`

## Problem Statement

The curriculum-industry skill-gap problem is to identify which technical skills are sufficiently covered by an academic curriculum and which skills have stronger industry demand but lower curriculum coverage. The project converts the skill-gap dataset into a Feast feature store so the same engineered features can be retrieved consistently for model training and online prediction.

## Dataset

The starter dataset contains **20 skills**. It is intentionally small so the complete Feast workflow can run locally.

### Columns

| Column | Meaning |
|---|---|
| `skill_id` | Unique identifier for each skill; Feast entity key |
| `skill_name` | Skill name |
| `curriculum_coverage` | Curriculum coverage score from 0–100 |
| `industry_demand` | Industry demand score from 0–100 |
| `job_posting_frequency` | Relative job-posting frequency score from 0–100 |
| `practical_exposure` | Practical/project exposure score from 0–100 |
| `event_timestamp` | Timestamp used by Feast for point-in-time retrieval |
| `high_gap` | Target label: 1 when the calculated gap is high, otherwise 0 |

### Target

`high_gap` is the machine-learning target. It is created from the engineered skill-gap score using a threshold of 20.

### How entries were created

The values are synthetic/illustrative starter values designed to represent curriculum coverage, industry demand, job-posting frequency, and practical exposure. They should be replaced with the student's own previous-activity skill-gap dataset for final submission.

## Feature Engineering

The feature pipeline creates these Feast features:

| Feast feature | Meaning |
|---|---|
| `curriculum_coverage` | Curriculum coverage percentage/score |
| `industry_demand` | Industry demand score |
| `job_posting_frequency` | Relative job-posting frequency |
| `practical_exposure` | Practical exposure score |
| `skill_gap_score` | `industry_demand - curriculum_coverage` |
| `industry_pressure` | Average of industry demand and job-posting frequency |
| `readiness_score` | Average of curriculum coverage and practical exposure |

### Example feature calculation

For a skill with curriculum coverage 60 and industry demand 85:

`skill_gap_score = 85 - 60 = 25`

A positive score indicates that industry demand is higher than curriculum coverage.

## Feast Architecture

```text
Original Dataset (CSV)
        ↓
Feature Engineering
        ↓
Parquet Offline Data
        ↓
Feast FeatureView
        ↓
   ┌────────────────────────┐
   ↓                        ↓
Historical Features     Materialization
   ↓                        ↓
Model Training          SQLite Online Store
                            ↓
                     Online Retrieval
                            ↓
                        Prediction
```

## Implementation

### Entity

The Feast entity is `skill`, with `skill_id` as its join key. It uniquely identifies a curriculum/industry skill record.

### Data source

`FileSource` points to the engineered Parquet file in `data/skill_gap_features.parquet`. The local Feast provider uses the file-based offline store and SQLite as the online store.

### FeatureView

`skill_gap_features` contains the engineered numeric features listed above. The FeatureView has a one-day TTL and uses `event_timestamp` for point-in-time correctness.

### Historical retrieval

`src/historical_features.py` creates an entity dataframe containing `skill_id` and `event_timestamp`, then calls `get_historical_features()` to retrieve the features needed for model training.

### Model

`src/train_model.py` trains a simple `LogisticRegression` classifier using the historical Feast features and the `high_gap` target. Accuracy is printed and saved to `outputs/model_metrics.json`.

### Online retrieval

After `feast materialize`, `src/online_prediction.py` calls `get_online_features()` for selected `skill_id` values and feeds the returned feature vector to the trained model.

## Commands

### 1. Create environment

```bash
python -m venv .venv
```

Windows:

```bash
.venv\Scripts\activate
```

Linux/macOS:

```bash
source .venv/bin/activate
```

### 2. Install dependencies

```bash
pip install -r requirements.txt
```

### 3. Prepare the feature dataset

```bash
python src/prepare_data.py
```

This creates `data/skill_gap_features.parquet`.

### 4. Register Feast objects

From the repository root:

```bash
feast -c feature_repo apply
```

`feast apply` scans the feature repository, validates/registers the entity and FeatureView, and prepares the local feature-store infrastructure. Feast documentation describes `apply` as the step that publishes feature definitions and configures the required infrastructure. urlFeast documentation – apply and architecturehttps://docs.feast.dev/getting-started/components/overview

### 5. Historical feature retrieval

```bash
python src/historical_features.py
```

The script writes:

`outputs/historical_features.csv`

### 6. Train the model

```bash
python src/train_model.py
```

### 7. Materialize features into SQLite

```bash
feast -c feature_repo materialize-incremental $(date -u +%Y-%m-%dT%H:%M:%S)
```

On Windows PowerShell, use a UTC timestamp such as:

```powershell
feast -c feature_repo materialize-incremental 2026-08-13T06:00:00
```

Materialization loads feature values from the offline store into the online store. urlFeast documentation – materialization and deploymenthttps://docs.feast.dev/v0.41-branch/how-to-guides/feast-snowflake-gcp-aws/deploy-a-feature-store

### 8. Online retrieval and prediction

```bash
python src/online_prediction.py
```

The script writes:

`outputs/online_prediction.json`

## Results

Run the commands above after replacing the starter dataset with your own data. The generated outputs should contain:

- **Historical feature output:** `outputs/historical_features.csv`
- **Model accuracy:** `outputs/model_metrics.json`
- **Online feature output:** `outputs/online_prediction.json`
- **One final prediction:** printed by `src/online_prediction.py`

### Starter-data result format

Because the supplied assignment document does not contain the previous-activity dataset, no factual model accuracy or online prediction is claimed here. The scripts calculate those values when executed on the actual dataset.

## Required Analysis Questions

### 1. What is the entity in your Feast implementation?

The entity is `skill`, identified by the `skill_id` join key. It represents one curriculum/industry skill.

### 2. List the features stored in your FeatureView.

`curriculum_coverage`, `industry_demand`, `job_posting_frequency`, `practical_exposure`, `skill_gap_score`, `industry_pressure`, and `readiness_score`.

### 3. Explain how one feature was calculated.

`skill_gap_score = industry_demand - curriculum_coverage`. For example, if demand is 85 and coverage is 60, the gap score is 25.

### 4. What is the difference between your original dataset and the feature dataset?

The original dataset contains the raw skill-level observations. The feature dataset adds engineered numeric features, standardizes the Feast timestamp/entity structure, and stores the serving/training feature values in Parquet.

### 5. What is the purpose of the offline store?

The offline store keeps historical feature data used for training and historical feature retrieval. In this local implementation, the source data is Parquet.

### 6. What is the purpose of the online store?

The online store keeps the latest materialized feature values for low-latency retrieval during prediction. This project uses SQLite locally.

### 7. What is the purpose of `feast apply`?

It registers/updates the Feast entities and FeatureViews and prepares the configured feature-store infrastructure.

### 8. What does materialization do?

Materialization copies the relevant feature values from the offline store into the online store so they can be retrieved for online inference.

### 9. What is the advantage of retrieving features through Feast instead of manually calculating them separately during training and prediction?

It centralizes feature definitions and retrieval, reducing duplicated feature logic and helping keep training and serving feature computation consistent. Historical retrieval also supports point-in-time-correct feature joins.

### 10. State two limitations of your current dataset.

1. The starter dataset is synthetic and small; it does not represent a statistically complete industry sample.
2. Industry demand and curriculum coverage are simplified scores rather than measurements collected from a large set of verified curriculum and job-posting sources.

### 11. State two ways your feature store could be improved when more curriculum and industry evidence becomes available.

1. Replace synthetic scores with periodically collected curriculum, job-posting, and employer evidence and add source/date metadata.
2. Add time-aware features such as skill-demand trends, regional demand, experience level, and rolling industry-demand statistics.

## Repository Structure

```text
MLOps-Feast-SkillGap/
├── README.md
├── requirements.txt
├── .gitignore
├── data/
│   ├── skill_gap_raw.csv
│   └── skill_gap_features.parquet        # generated
├── feature_repo/
│   ├── feature_store.yaml
│   └── skill_features.py
├── src/
│   ├── prepare_data.py
│   ├── historical_features.py
│   ├── train_model.py
│   └── online_prediction.py
└── outputs/
    ├── historical_features.csv           # generated
    ├── model_metrics.json                # generated
    └── online_prediction.json            # generated
```

## GitHub Submission

Use the required repository name:

`<RegisterNumber>MLOps-Feast-SkillGap`

Example: `231FA04001-MLOps-Feast-SkillGap`

Before submission:

1. Replace the starter CSV with the previous-activity dataset.
2. Fill in name, register number, and section.
3. Run the complete workflow successfully.
4. Commit the source code, README, raw/feature data as allowed by your faculty, and generated evidence.
5. Push the repository to GitHub and submit its link through the faculty Google Form.
