---
title: Connect to Hugging Face Datasets
description: Connect to tabular data on the Hugging Face Hub and organize it into Batches for validation.
hide_feedback_survey: false
hide_title: false
toc_max_heading_level: 2
---

Hugging Face Datasets lets you load datasets from the Hugging Face Hub. In GX, a Data Source identifies the dataset repository, a Data Asset holds materialization options, and a Batch Definition selects a configuration and split. When you request a Batch, you can provide a repository revision so the Batch metadata records which version of the dataset was validated.

This integration uses the pandas execution engine. It is intended for tabular datasets with scalar values supported by pandas. Streaming datasets and complex values such as images, audio, and video are not supported by this workflow.

## Create a Data Source

A Hugging Face Data Source connects GX to a dataset repository. You can provide a Hugging Face token for private or gated repositories; otherwise, public datasets can be accessed without one.

### Prerequisites {#prerequisites-data-source}

- Python installed.
- GX installed.
- The Hugging Face `datasets` package installed.
- A Data Context. The examples below assume that `context` contains your Data Context.

### Procedure {#procedure-data-source}

Create a pandas Hugging Face Data Source with the repository identifier. Set `token` to a token string or a configuration variable reference when the repository requires authentication. The Hub endpoint defaults to `https://huggingface.co`.

```python
hf_datasource = context.data_sources.add_pandas_hugging_face(
    name="hf_wikipedia",
    repository="wikimedia/wikipedia",
    token="${HF_TOKEN}",
)
```

If the dataset is public, omit `token`:

```python
hf_datasource = context.data_sources.add_pandas_hugging_face(
    name="hf_wikipedia",
    repository="wikimedia/wikipedia",
)
```

## Create a Data Asset

A Hugging Face Data Asset represents the data to materialize as a pandas DataFrame. Use it to configure options such as selecting columns. Configuration and split selection are defined by a Batch Definition.

### Prerequisites {#prerequisites-data-asset}

- A Data Context and the Hugging Face `datasets` package.
- A [Hugging Face Data Source](#create-a-data-source).

### Procedure {#procedure-data-asset}

Add a dataset asset to the Data Source. `columns` is optional; omit it to include all columns.

```python
hf_asset = hf_datasource.add_dataset_asset(
    name="wikipedia_data",
    columns=["id", "text"],
    batch_metadata={"pipeline": "wikipedia_quality_checks"},
)
```

`batch_metadata` is optional. Values defined on the Data Asset are included with each Batch created from that asset. The Hugging Face repository, configuration, split, and requested revision are also recorded as Batch metadata.

## Create a Batch Definition

A Batch Definition selects the configuration and split to load from the repository. A split can be a named split such as `train` or a Hugging Face split expression supported by the dataset, such as `train[:10%]`.

### Prerequisites {#prerequisites-batch-definition}

- A Data Context and the Hugging Face `datasets` package.
- A [Hugging Face Data Asset](#create-a-data-asset).

### Procedure {#procedure-batch-definition}

Create a split Batch Definition with a unique name, the dataset configuration, and the split to validate:

```python
train_batch_definition = hf_asset.add_batch_definition_split(
    name="wikipedia_english_train",
    config="20231101.en",
    split="train",
)
```

Retrieve the Batch and pin it to a revision. The revision can be a branch, tag, or commit hash. If omitted, Hugging Face loads the current default revision.

```python
batch = train_batch_definition.get_batch(
    batch_parameters={"revision": "a1f724c09"},
)
```

The Batch metadata preserves its Hugging Face identity, including the repository, configuration, split, and revision used to load the data, together with any metadata defined on the Data Asset. This makes validation results traceable to the dataset version that was checked. Pin a commit hash when you need repeatable validation runs.

You can also define a Batch for an entire configuration without selecting one split:

```python
all_splits_batch_definition = hf_asset.add_batch_definition_whole_config(
    name="wikipedia_english_all",
    config="20231101.en",
)
```

The dataset is loaded through the Hugging Face `datasets` library, which manages its local cache. The GX Batch is materialized as a pandas DataFrame for validation.
