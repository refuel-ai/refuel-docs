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
  dataset_name='<NEW DATASET NAME>',
  embedding_columns=['<COLUMN 1>', …]
)
```

Some details about the function parameters:

| Option        | Is Required | Default Value | Comments |
| :-----------------   | :-----------| :----------| :------- |
| `file_path`   | Yes         | -          | Path to the data you wish to upload |
| `dataset_name`| Yes         | -          | Unique name of the dataset being uploaded |
| `embedding_columns` | No    | Empty List | Column(s) that will be used as inputs when computing embedding representation for this row |

### Download Dataset

This function create a snapshot of your dataset and generate presigned URL for secure download.

```python
response = refuel_client.download_dataset(
  dataset='<DATASET NAME>',
  email_address='<YOUR EMAIL ADDRESS>',
)
```

This is an asynchronous workflow, and when the dataset is available for download, Refuel will send an email to the email address. Make sure the email address belongs to a valid user from your team. Depending on the size of the dataset, this export step can take a few minutes. Once the download link is created and emailed, it will be valid for 24 hours.

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
  max_items=100,
  offset=0
)
```

This function will return a pandas dataframe. Some details about the function parameters: 

| Option       | Is Required | Default Value | Comments |
| :------------  | :-----------| :-------------| :------- |
| `dataset`    | Yes         | -            | Name of the dataset you want to query and retrieve items (rows) from |
| `max_items`  | No          | 100          | Max number of rows you want to fetch |
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

By default, the API will use Refuel’s sort order (by decreasing order of diversity). But you can specify any other column in the dataset that you would lille to sort by, when querying for items in the dataset: 

```python
items = refuel_client.get_items(
  dataset='<DATASET NAME>',
  max_items=100,
  order_by='<COLUMN NAME TO SORT BY>',
  order_direction='ASC'
)
```

Some details about sorting related function parameters:

| Option       | Is Required | Default Value           | Comments |
| :---------------  | :-----------| :-----------------------| :------- |
| `order_by`   | No          | Refuel’s default sort (by diversity) | Name of the dataset you want to query and retrieve items (rows) from |
| `order_direction` | No     | 100          | Valid values: ASC or DESC |

#### Applying filters when querying items

In addition to sorting options, you can also define filters to only fetch items in the dataset that match a certain criteria. A filter such as “column = value” is defined as a Python dictionary with three keys:

- field:  This is the column on which you want to apply the filter
- operator: This is the comparison operator
- value: This is the value to compare to

Here’s an example of how to define and use filters in the SDK:

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

Here’s the complete list of filter operators that are currently supported

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

## Labeling Tasks

These functions let you retrieve information about labeling tasks defined within a project, and start and cancel a task run.

### Define a new Task

You can create a new task programmatically within a given project as follows:

```python
import refuel

options = {
    "api_key": "<YOUR API KEY>",
    "project": "<PROJECT NAME>",
}

refuel_client = refuel.init(**options)

refuel_client.create_task(
    task='<TASK NAME>',
    task_type='<TASK TYPE>',
    dataset='<DATASET NAME>',
    input_columns=['col 1', 'col 2' ...],
    context = '...',
    fields = [{'name': '...', 'guidelines': ...}]
)
```

- task_type is one of: `classification`, `multilabel_classification` or `attribute_extraction`
- input_columns is the subset of columns from the dataset that will be used as input for LLM
- fields is a list of dictionaries. Each dictionary contains a fixed set of keys: name (name of the LLM label field as it will be appear in the exported dataset), guidelines (labeling guidelines for the LLM) and labels (list of valid labels, this field is only required for classification type tasks)

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

response = refuel_client.label(application='<APPLICATION NAME>', inputs=inputs)
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
            'label': 'positive,
            'confidence': 0.9758
          }
        }
    }]
}
```


### Share feedback for application outputs

The SDK allows users to log feedback for online predictions. When logging predictions, it is important to identify the input request for which you're logging feedback using `refuel_uuid` from the response above:

```python

label = {'sentiment': 'negative'}
refuel_client.feedback(application='my_sentiment_classifier', refuel_uuid='...', label=label)
```

Any row with logged feedback will appear with the verified check mark ("✓") in the cloud application.
