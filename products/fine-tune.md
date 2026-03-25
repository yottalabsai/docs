---
hidden: true
icon: empire
---

# Fine-Tune

In the following guide, we'll learn how to use the Yotta AI fine-tuning CLI tool to fine-tune a Llama 3 7B model on a testing dataset.

## Install the CLI

To get started, install the Yotta Python CLI:

{% code overflow="wrap" fullWidth="false" %}
```sh
pip install --upgrade yotta
```
{% endcode %}

## Authentication

The API Key can be configured via setting the `YOTTA_API_KEY` environment variable by running the following command:

```sh
export YOTTA_API_KEY=xxxxx
```

## Uploading Data

To upload your data, run the following command. Remember to replace `PATH_TO_DATA_FILE` with the path to your dataset.

```
yotta files upload {PATH_TO_DATA_FILE}
```

## Start Fine-Tuning Job

You now can start the fine-tuning job based on your training file and model. The command line is:&#x20;

```
yotta fine-tuning start --training-file {FILE_ID} --model {MODEL_NAME} --wandb-api-key {WANDB_API_KEY}
```

## Monitor Jobs

You can use the following command line to get the progress of a job:

```
yotta fine-tuning list-events {FINE-TUNING_ID}
```

## Other Useful CLI Commands

```sh
# list all available commands
yotta --help

# check which models are available.
yotta models list

# check your jsonl file
yotta files check test.jsonl

# upload your jsonl file
yotta files upload test.jsonl

# list your uploaded files
yotta files list

# retrieve progress updates about the finetune job
yotta fine-tuning retrieve ft-01b7df2b-122a-4b84-9838-4d84200dc7ac

# download your fine-tuned model using the uuid of your fine-tuning job
yotta fine-tuning download ft-01b7df2b-122a-4b84-9838-4d84200dc7ac 
```

## **Pricing**
