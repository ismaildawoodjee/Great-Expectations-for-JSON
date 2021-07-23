# Yelp Businesses Data Quality with Great Expectations

## Table of Contents

## Introduction

This is the continuation of an exploration I did using
[Great Expectations for an e-commerce CSV dataset](https://github.com/ismaildawoodjee/Great-Expectations-for-CSV).

Great Expectations as a library is similar to Pandas, and has methods for `read_excel`, `read_csv`,
`read_json`, `read_parquet` and more (not sure if its directly inherited from Pandas).
Here, I will be applying Great Expectations (GE) to explore JSON data. The reason is that JSON is used in a lot
of places to transfer data, since many web APIs serve data in JSON format. For example, I may want
to extract data from an API, obtain it in JSON format and stored it in a data lake in its raw
form before moving it to a staging area.

JSON data may not always come in clean key-value pairs. Values can be nested, and some nested
dictionaries could turn out to be strings, making the un-nesting (or flattening) process difficult
to carry out. Great Expectations currently does not have any methods for flattening JSON data, so
other JSON-flattening libraries are required. However, with the `read_json` method
provided by the GE library, it is easy to read JSON data and convert it into a data frame.

## Setup

### Python Setup

Clone this repo, set up a Python virtual environment and install GE:

    git clone https://github.com/ismaildawoodjee/Great-Expectations-for-JSON

    cd Great-Expectations-for-JSON

    python -m venv .venv

    source .venv/bin/activate

### Great Expectations Setup

0. Install Great Expectations:

        pip install great_expectations

    or `pip install -r requirements.txt` if you want to install a JSON flattening library as well.

1. Create a Data Context:

        great_expectations --v3-api init

2. Configure a Datasource:

        great_expectations --v3-api datasource new

3. Create an Expectations Suite:

        great_expectations --v3-api suite new

4. Choose `3. Automatically, using a profiler`. Then, choose `3. yelp_profiling.json`.
Follow the instructions in the Jupyter Notebook to choose which columns to set Expectations.
Generate Expectations and inspect the Data Doc that is opened.

5. Afterwards, edit these Expectations with the following (I named the suite `yelp_suite.json`):

        great_expectations --v3-api suite edit yelp_suite --interactive

    Rename the suite with a new name (for instance, `yelp_suite_updated`) and update the Expectations
    as you wish (the details of how I edited them are in the [Great Expectations](#great-expectations) section).

6. Validate new batch of data with the following (I named the new checkpoint `yelp_checkpoint`):

        great_expectations --v3-api checkpoint new yelp_checkpoint

    Change the `data_asset_name` to `yelp_validating.json` to validate this new data batch, and change
    `expectation_suite_name` to `yelp_suite_updated` to pair up Datasource with Expectation Suite.
    Run all cells and inspect Data Doc to see if validation passes or fails.

## Data Preparation

The data for this exploration can be obtained from the publicly available
[Yelp businesses](https://www.kaggle.com/yelp-dataset/yelp-dataset?select=yelp_academic_dataset_business.json)
dataset from Kaggle. Each line in the original data `yelp_academic_dataset_business` contains data about
a specific business. However, the data is just provided like that, without adhering to a standard JSON format
(even without any commas to separate between businesses), which may be in the form of a list of dictionaries
(record-oriented JSON), or a "dictionary of dictionaries" (column-oriented JSON).

GE failed to read this raw data, unless the argument `lines=True` were passed into the `read_json` method.
But then, if you want to pass in arguments, this would mean automatic profiling is not possible, and
Expectations will need to be written from scratch, which may take a lot of time if a dataset has many columns.

![Read JSON line in the read_json method](./assets/images/1.%20specify_read_json_lines.png)

Also, even if the argument is passed via `batch_requests`, the resulting dataframe contains nested dictionaries.

![Nested data frame](./assets/images/2.%20unwanted_nested_dataframe.png)

I tried using `json_normalize` from the `pandas.io.json` library, but that didn't work for some reason.

![json_normalize error](./assets/images/4.%20but_results_in_error.png)

Later I discovered that some of the nested data that looked like dictionaries were actually strings, which was
why flattening was not possible. Also, JSON uses double quotes, but the internal elements of the string-dictionaries
use single quotes, making it more complicated to process.

![Look like dictionaries, but are actually strings](assets/images/6.%20strings_not_dictionaries.png)

The data preparation notebook converts the non-standard raw JSON file into two column-oriented JSON files.

**NOTE: The automatic profiler by GE only reads either record-oriented or column-oriented JSON files, and throws
an error if the file is in some other format.**

- introduce dirty data

## Great Expectations

- editing expectations, adding new expectations

## Conclusion
