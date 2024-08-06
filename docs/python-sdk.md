---
hide:
  - navigation
---


# Refuel SDK Guide

**Refuel SDK** is a Python library to programmatically call Refuel's APIs. This guide describes how to install and use the Refuel SDK.

The SDK is currently an beta release and under rapid development. Please expect some sharp edges. Feedback and bug reports are always welcome! Shoot us an email at `support@refuel.ai` or ping us on our shared Slack.

## Getting Started

### Installation

Install the SDK using a package installer such as pip:

```bash
pip install refuel
```

Installing and using the SDK requires Python 3.6+.

### Initializing a client session

Set the environment variable REFUEL_API_KEY. The SDK will read it during initialization, and use this value when sending requests:

```python
import refuel


# Assuming you've set `REFUEL_API_KEY` in your env,
# init() will pick it up automatically
refuel_client = refuel.init()
```

Alternatively, you can supply the API key as a parameter during initializtion as shared below. In the cloud application, there is a top-level dropdown to select the project you’re working on currently. And this selection powers all the pages downstream (datasets, labeling tasks etc). The SDK allows you to do this by setting the project during initialization as well:

```python
import refuel

options = {
    "api_key": "<YOUR API KEY>",
    "project": "<PROJECT NAME>",
}

refuel_client = refuel.init(**options)
```

Here’s the complete list of initialization options currently supported:


| Option             | Is Required | Default Value | Comments |
| :-------------     | :-----------| :-------------| :------- |
| `api_key`          | Yes         | None          | Used to authenticate all requests to the API server |
| `project`          | Yes         | None          | The name of the project you plan to use for the current session |
| `timeout`          | No          | 60            | Timeout in seconds |
| `max_retries`      | No          | 3             | Max number of retries for failed requests |
| `max_workers`      | No          | Num CPUs (os.cpu_count()) | Max number of concurrent requests to the API server |


## Projects

These functions let you create a new project in your team’s Refuel account, or get information about projects already defined.

### Get Projects

The get_projects API will return a list of all projects that belong to your team:

```python
projects = refuel_client.get_projects()
```

You can also retrieve information about a specific project, either by name or by UUID

```python
project = refuel_client.get_project(project='<PROJECT NAME>')
```

### Create Project

```python
refuel_client.create_project(
  project='<NEW PROJECT NAME>',
  description='<NEW PROJECT DESCRIPTION>',
)
```

## Datasets

These functions let you upload/download a full dataset, or fetch rows and LLM labels within a dataset.

### Get all current datasets

The get_datasets function will return a list of all datasets in the project:

```python
import refuel

options = {
    "api_key": "<YOUR API KEY>",
    "project": "<PROJECT NAME>",
}

refuel_client = refuel.init(**options)

datasets = refuel_client.get_datasets()
```

### Upload Dataset

This function lets you upload a local CSV file as a new dataset to Refuel.

```python
import refuel

options = {
    "api_key": "<YOUR API KEY>",
    "project": "<PROJECT NAME>",
}

refuel_client = refuel.init(**options)

dataset = refuel_client.upload_dataset(
  file_path='<PATH TO CSV FILE>',
  dataset_name='<NEW DATASET NAME>'
)
```

Some details about the function parameters:

| Option        | Is Required | Default Value | Comments |
| :-----------------   | :-----------| :----------| :------- |
| `file_path`   | Yes         | -          | Path to the data you wish to upload |
| `dataset_name`| Yes         | -          | Unique name of the dataset being uploaded |

### Download Dataset

This function create a snapshot of your dataset and generate presigned URL for secure download.

```python
response = refuel_client.download_dataset(
  dataset='<DATASET NAME>',
  email_address='<YOUR EMAIL ADDRESS>',
)
```

This is an asynchronous workflow, and when the dataset is available for download, Refuel will send an email to the email address. Make sure the email address belongs to a valid user from your team. Depending on the size of the dataset, this export step can take a few minutes. Once the download link is created and emailed, it will be valid for 24 hours.

### Adding items (rows) to an uploaded dataset

This function lets you append new rows to an existing dataset. Keep in mind that the dataset schema is decided during initial upload and is not updated here.

```python
import refuel

options = {
    "api_key": "<YOUR API KEY>",
    "project": "<PROJECT NAME>",
}

refuel_client = refuel.init(**options)

new_items_to_add = [
  {"column1": "value1", "column2": "value2", ...},
  {"column1": "value3", "column2": "value4", ...},
  ...
]

refuel_client.add_items(dataset='<DATASET NAME>', items=new_items_to_add)
```

### Querying items (rows) in a dataset

In addition to downloading the entire dataset, you can also fetch a list of items (rows) from the dataset as follows:

```python
import refuel

options = {
    "api_key": "<YOUR API KEY>",
    "project": "<PROJECT NAME>",
}

refuel_client = refuel.init(**options)

items = refuel_client.get_items(
  dataset='<DATASET NAME>',
  max_items=20,
  offset=0
)
```

This function will return a pandas dataframe. Some details about the function parameters: 

| Option       | Is Required | Default Value | Comments |
| :------------  | :-----------| :-------------| :------- |
| `dataset`    | Yes         | -            | Name of the dataset you want to query and retrieve items (rows) from |
| `max_items`  | No          | 20           | Max number of rows you want to fetch |
| `offset`     | No          | 0            | If this is set to a positive number, say N, then the first N rows will be skipped and the API will return “max_items” number of rows after skipping the first N rows. |

#### Querying items, along with labels from a labeling task

get_items() also allows you to provide an optional parameter - a labeling task. When provided, the function will also include the task results (LLM labels, confidence and manually confirmed labels, if any) for the returned items.

```python
items = refuel_client.get_items(
  dataset='<DATASET NAME>',
  task='<LABELING TASK NAME>',
  max_items=100
)
```

#### Applying sort ordering when querying items

By default, the API will use Refuel’s sort order (by decreasing order of diversity). You can use the `order_by` param to sort by any other columns in the dataset or by the label or confidence score from a labeling task.

1) Sort by dataset column

```python
items = refuel_client.get_items(
  dataset='<DATASET NAME>',
  max_items=100,
  order_by=[{'field': '<COLUMN NAME TO SORT BY>', 'direction': '<ASC or DESC>'}],
)
```

2) Sort by label or confidence score from a labeling task. Note that this requires a task name and a subtask name to be specified. `field` can be either 'label' or 'confidence'.

```python
items = refuel_client.get_items(
  dataset='<DATASET NAME>',
  task='<LABELING TASK NAME>',
  max_items=100,
  order_by=[{'field': 'confidence', 'direction': '<ASC or DESC>', 'subtask': '<SUBTASK NAME>'}],
)
```

You may have multiple dicts in the `order_by` list if you would like to sort by multiple columns (used in the case of ties). Some details about the keys for each dict in the `order_by` list:

| Key       | Is Required | Default Value           | Description | Comments |
| :---------------  | :-----------| :-----------------------|:-----------------------| :------- |
| `field`   | Yes          |  | The name of the column in the dataset to sort by | In addition to the columns in the dataset, the field can also be 'label' or 'confidence', if the task and subtask names are specified. |
| `direction`   | No          | `ASC` | The direction that you would like to sort the specified column by | Should be `ASC` or `DESC` |
| `subtask`   | No          | null | The name of the subtask for which you would like to sort by label or confidence | This should only be provided if the field is 'label' or 'confidence' and requires a task name to be specified in the function params. |

#### Applying filters when querying items

In addition to sorting options, you can also define filters to only fetch items in the dataset that match a specific criteria. The SDK supports three types of filters:

1) Metadata Filters:

Filter based on the value of a specific column, for e.g. a filter such as “column = value” is defined as a Python dictionary with three keys:

- field:  This is the column on which you want to apply the filter
- operator: This is the comparison operator
- value: This is the value to compare to

Here’s an example:

```python
items_filter = {
  'field': 'transaction_category',
  'operator': '=',
  'value': 'Food'
}

items = refuel_client.get_dataset_items(
  dataset='<DATASET NAME>',
  max_items=100,
  filters = [items_filter] 
)
```

2) LLM output value/confidence filter:

Filter items based on the LLM output value or confidence score from a specific task configured in Refuel.

Here's a concrete example: Let's say you configured a classification task called `Sentiment Analysis` in your Refuel account, which has two subtasks (output fields):

(i) `predicted_sentiment` - the predicted sentiment

(ii) `explanation` - a one sentence explanation of why the LLM output the predicted sentiment as Positive or Negative for the item.

Here are a few filters we can define for this task:

* "predicted sentiment is Positive":
  
```python
{
  'field': 'llm_label',
  'subtask': 'predicted_sentiment'
  'operator': '=',
  'value': 'Positive'
}
```

* "predicted sentiment confidence >= 80%":
  
```python
{
  'field': 'confidence',
  'subtask': 'predicted_sentiment'
  'operator': '>=',
  'value': '0.8'
}
```

* "predicted sentiment does not agree with the ground truth label (available in a column called `ground_truth_sentiment`)":
  
```python
{
  'field': 'llm_label',
  'subtask': 'predicted_sentiment'
  'operator': '<>',
  'value': 'ground_truth_sentiment'
}
```


3) Semantic search filter:

Filter based on semantic similarity to a “query item” in a dataset. Semantic search queries are mapped to approximate nearest neighbor searches in an embedding space. These filters are defined as a Python dictionary with two keys:

- operator: always set to "SIMILAR"
- value: This is query item, identified by the uuid in the `refuel_uuid` column.

Here’s an example of how to define a semantic search filter. The results are returned in decreasing order of similarity score:

```python
items_filter = {
  'operator': 'SIMILAR',
  'value': '020b434f-e1d7-4b0e-bfc1-beded36c806a'
}

items = refuel_client.get_dataset_items(
  dataset='<DATASET NAME>',
  task='<TASK NAME>'
  max_items=10,
  filters = [items_filter] 
)
```


Here’s the complete list of filter operators that are currently supported:

| Operator      | Description |
| :----------   | :-----------|
| `>`           | Greater than |
| `>=`          | Greater than or equal to |
| `=`           | Equals       |
| `<>`          | Not equal to |
| `<`           | Less than    |
| `<=`          | Less than or equal to  |
| `IS NULL`     | True if field is undefined  |
| `IS NOT NULL` | True if field is defined  |
| `LIKE`        | String matching: True if value is in field |
| `ILIKE`       | String matching (case insensitive): True if value is in field |
| `NOT LIKE`    | String does not match: True if value is not in field |
| `NOT ILIKE`   | String does not match (case insensitive): True if value is not in field |
| `SIMILAR`     | Semantic similarity filter operator |

## Labeling Tasks

These functions let you retrieve information about labeling tasks defined within a project, and start and cancel a task run.

### Define a new Task

You can create a new task programmatically within a given project using the `create_task` function:

```python
import refuel

options = {
    "api_key": "<YOUR API KEY>",
    "project": "<PROJECT NAME>",
}

refuel_client = refuel.init(**options)

refuel_client.create_task(
    task='<TASK NAME>',
    dataset='<DATASET NAME>',
    context = 'You are an expert at analyzing sentiment of an online review about a business...',
    fields = [
        {
          'name': '...',
          'type': '...',
          'guidelines': '...',
          'labels': [...],
          'input_columns': [...],
          'fallback_value': '...'
        }, 
        ...
    ],
    model = 'GPT-4 Turbo'
)
```

Some details about the various parameters you see in the function signature above:

| Parameter          | Is Required | Default Value | Comments |
| :-------------     | :-----------| :-------------| :------- |
| `task`             | Yes         | None          | Name of the new task you're creating |
| `dataset`          | Yes         | None          | Dataset (in Refuel) for which you are defining this task |
| `context`          | Yes         | None          | Context is a high level description of the problem domain and the dataset that the LLM will be working with. It typically starts with something like 'You are and expert at ...' |
| `fields`           | Yes         | None          | This is a list of dictionaries. Each entry in this list defines an output field generated in the task. See below for details about the schema of each field |
| `model`            | No          | team default  | LLM that will be used for this task. If not specified, we will use the default LLM set for your team, e.g. GPT-4 Turbo |


Next, let's take a look at the schema of each entry in the `fields` list above:

| Parameter          | Is Required | Default Value | Comments |
| :-------------     | :-----------| :-------------| :------- |
| `name`             | Yes         | None          | Name of the output field, e.g. `llm_predicted_sentiment` |
| `type`             | Yes         | None          | Type of output field. This is one of: [`classification`, `multilabel_classification`, `attribute_extraction`, `webpage_transform`, `web_search`] |
| `guidelines`       | Yes         | None          | Output guidelines for the LLM for this field. Note that if the field type is a `web_search` type, the guidelines will be simply the query template |
| `labels`           | Yes (for classification field types) | None          | list of valid labels, this field is only required for classification type tasks |
| `input_columns`    | Yes         | None          | Columns from the dataset to use as input when passing a "row" in the dataset to the LLM.|
| `ground_truth_column`| No       | None          | A column in the dataset that contains ground truth value for this field, if one exists. Note this is an optional parameter. |
| `fallback_value`   | No         | None          | A fallback/default value that the LLM should generate for this field if a row cannot be processed successfully |


Finally, here is the list of LLMs currently supported (use the model name as the parameter value):

| Provider      | Name |
| :----------   | :---|
| OpenAI        | GPT-4 Turbo |
| OpenAI        | GPT-4o      |
| OpenAI        | GPT-4o mini |
| OpenAI        | GPT-4       |
| OpenAI        | GPT-3.5 Turbo |
| Anthropic     | Claude 3.5 (Sonnet) |
| Anthropic     | Claude 3 (Opus) |
| Anthropic     | Claude 3 (Haiku) |
| Google        | Gemini 1.5 (Pro) |
| Mistral       | Mistral Small |
| Mistral       | Mistral Large |
| Refuel        | Refuel LLM-2 |
| Refuel        | Refuel LLM-2-small |


### Get Tasks

You can retrieve a list of all tasks within a given project as follows

```python

tasks = refuel_client.get_tasks()
```

### Start a Labeling Task Run

You can begin running a labeling task on a dataset with the following:

```python
response = refuel_client.start_task_run(
  task='<TASK NAME>',
  dataset='<DATASET NAME>',
  num_items=100
)
```

This will kick off a bulk labeling run for the specified task and dataset, and label 100 items in the dataset. If `num_items` parameter is not specified, it will label the entire dataset.

### Cancel an ongoing labeling task run

You can also cancel an ongoing labelling task with the same function as follows.

```python
response = refuel_client.cancel_task_run(
  task='<TASK NAME>',
  dataset='<DATASET NAME>'
)
```

### Get Task run status/progress

To check on the status of an ongoing labeling task run, you can use the following function

```python
task_run = refuel_client.get_task_run(
  task='<TASK NAME>',
  dataset='<DATASET NAME>'
)
```

The `task_run` object has the following schema:

```python
{
 'id': '...',
 'task_id': '...',
 'task_name': '...',
 'dataset_id': '...',
 'dataset_name': '...',
 'model_name': '...',   # This is the LLM used for the run
 'status': 'active',    # Status of the task run
 'metrics': [           # Metrics related to the task run
      {'name': 'num_labeled', 'value': ... },
      {'name': 'num_remaining', 'value': ...},
      {'name': 'time_elapsed_seconds', 'value': ...},
      {'name': 'time_remaining_seconds', 'value': ...},
 ]
}
```

`status` enum shows the current task run status. It can be one of the following values:

- `not_started`: this is the starting state before the batch run has been kicked off
- `active`: a batch task run is ongoing
- `paused`: the batch run was paused before the full dataset was labeled.
- `failed`: the batch run failed due to a platform error. This should ideally never happen
- `completed`: the batch run was completed successfully

`metrics` is a list containing all metrics for the current task run. Currently the platform supports the following metrics:

- `num_labeled`: number of rows from the dataset that have been labeled
- `num_remaining` number of rows from the dataset that are remaining
- `time_elapsed_seconds`: time (in seconds) since the task run started. This is only populated when the task run is active (since this metric is not valid when there is no active run).
- `time_remaining_seconds` estimated time (in seconds) remaining for the task run to complete. This is only populated when the task run is active (since this metric is not valid when there is no active run).

## Applications

Refuel allows you to deploy a labeling task as an application. Applications allow you to label data synchronously on demand, primarily for online workloads.

### Deploy labeling application

To deploy an existing task as a labeling application, you can use the following function

```python
import refuel

options = {
    "api_key": "<YOUR API KEY>",
    "project": "<PROJECT NAME>",
}

refuel_client = refuel.init(**options)

response = refuel_client.deploy_task(task='<TASK NAME>')
```

### Get all labeling application

To get all labeling applications that are currently deployed, use the following function

```python
applications = refuel_client.get_applications()
```

### Label using a deployed application

You can use the deployed application for online predictions as follows:

```python
inputs = [
  {'col_1': 'value_1', 'col_2': 'value_2' ...},
  {'col_1': 'value_1', 'col_2': 'value_2' ...},
]

response = refuel_client.label(application='<APPLICATION NAME>', inputs=inputs, explain=False)
```

Each element in `inputs` is a dictionary, with keys as names of the input columns defined in the task. For example, let's consider an application for sentiment classification called `my_sentiment_classifier`, with two input fields - `source` and `text`. You can use it as follows:

```python
inputs = [
  {'source': 'yelp.com', 'text': 'I really liked the pizza at this restaurant.'}
]

response = refuel_client.label(application='my_sentiment_classifier', inputs=inputs)
```

`response` has the following schema:

- `refuel_output[i]` contains the output for `inputs[i]`
- `refuel_fields` is a list whose length is equal to the number of fields defined in the application. For example, let's say `my_sentiment_classifier` has just one field, `sentiment`. In this case the output will be:

```
{
  'application_id': '...',
  'application_name': 'my_sentiment_classifier',
  'refuel_output': [
    {'refuel_uuid': '...',
     'refuel_api_timestamp': '...',
     'refuel_fields': 
        {
          'sentiment': {
            'label': ['positive'],
            'confidence': 0.9758
          }
        }
    }]
}
```

You can also set the optional `explain` parameter to `True` to get an explanation for why the provided label was returned. The explanation will be returned in the `explanation` field in the response, along with the `label` and `confidence`:

```
'sentiment': {
    'label': ['positive'],
    'confidence': 0.9758,
    'explanation': 'The model predicted a positive sentiment because the text contains positive words like "liked" and "good".'
  }
```

If you would only like to get an explanation for certain fields, you can optionally provide a list of field names in the `explain_fields` parameter for which you want an explanation returned. If `explain_fields` is provided, explanations will be returned regardless of whether `explain` is set to `True` or `False`. Here's an example of how to get explanations for the `sentiment` field in the `my_sentiment_classifier` application:

```python
response = refuel_client.label(application='my_sentiment_classifier', inputs=inputs, explain=True, explain_fields=['sentiment'])
```

You can also set the optional `telemetry` parameter to `True` to get additional info such as the model, provider, and number of tokens used (prompt, output, and total) in the request. The telemetry data will be returned in the `usage` field in the response.

```python
response = refuel_client.label(application='my_sentiment_classifier', inputs=inputs, telemetry=True)
```


### Share feedback for application outputs

The SDK allows users to log feedback for online predictions. When logging predictions, it is important to identify the input request for which you're logging feedback using `refuel_uuid` from the response above:

```python

label = {'sentiment': ['negative']}
refuel_client.feedback(application='my_sentiment_classifier', refuel_uuid='...', label=label)
```

Any row with logged feedback will appear with the verified check mark ("✓") in the cloud application.

## Finetuning

Refuel allows you to finetune a model based on all the human reviewed data and optionally data labeled by an LLM. Finetuned models allow you to reduce labeling cost and latency while achieving similar, and in some cases better, performance than LLMs like GPT-4.

### Starting a finetuning run

To start a model finetuning run, you can use the following function:

```python
import refuel

options = {
    "api_key": "<YOUR API KEY>",
    "project": "<PROJECT NAME>",
}

refuel_client = refuel.init(**options)

hyperparameters = {"num_epochs": 1}
datasets = ["dataset_0_id", "dataset_1_id"]
response = refuel_client.finetune_model(task_id='<TASK ID>', model='<BASE MODEL>', hyperparameters=hyperparameters, datasets=datasets)
```

Supported Base Models: ['refuel-llm-v2-large', 'refuel-llm-v2-small']
Supported hyperparameters: ['lora_r', 'lora_alpha', 'lora_dropout', 'weight_decay', 'learning_rate', 'cosine_min_lr_ratio']

### Get all finetuned models

You can retrieve a list of all models within a given project as follows

```python

models = refuel_client.get_finetuned_models(task_id='<TASK ID>')
```

### Cancel a finetuning run

You can also cancel an ongoing finetuning run as follows.

```python
response = refuel_client.cancel_finetuning(model_id='<FINETUNING MODEL ID>')
```