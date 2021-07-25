# Yelp Businesses Data Quality with Great Expectations

## Table of Contents

- [Yelp Businesses Data Quality with Great Expectations](#yelp-businesses-data-quality-with-great-expectations)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
  - [Setup](#setup)
    - [Python Setup](#python-setup)
    - [Great Expectations Setup](#great-expectations-setup)
  - [Data Preparation](#data-preparation)
    - [Splitting JSON Data for Profiling and Validating](#splitting-json-data-for-profiling-and-validating)
  - [Writing Expectations and Generating a Data Doc](#writing-expectations-and-generating-a-data-doc)
  - [Conclusion](#conclusion)

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

    To open up the Jupyter Notebook to edit the Expectations again, run the CLI command with the new Suite's name:

        great_expectations --v3-api suite edit yelp_suite_updated --interactive

6. Validate new batch of data with the following (I named the new checkpoint `yelp_checkpoint`):

        great_expectations --v3-api checkpoint new yelp_checkpoint

    Change the `data_asset_name` to `yelp_validating.json` to validate this new data batch, and change
    `expectation_suite_name` to `yelp_suite_updated` to pair up Datasource with Expectation Suite.
    Run all cells and inspect Data Doc to see if validation passes or fails.

    To rerun the checkpoint after editing Expectations, use the CLI command:

        great_expectations --v3-api checkpoint run yelp_checkpoint

    After rerunning checkpoint, the Data Docs need to be built again. To build and open up the Data Doc again after closing it, use the CLI command:

        great_expectations --v3-api docs build

    which will open up a Data Doc HTML page in the web browser.

## Data Preparation

### Splitting JSON Data for Profiling and Validating

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

The data preparation notebook converts the non-standard raw JSON file into two column-oriented JSON files. The `yelp_profiling.json`
was used for automatic profiling and dirty data was introduced into the `yelp_validating.json` file to validate it against
the Expectation Suite obtained from profiling.

**NOTE: The automatic profiler by GE only reads either record-oriented or column-oriented JSON files, and throws
an error if the file is in some other format.**

## Writing Expectations and Generating a Data Doc

In Step 5 of the [Great Expectations Setup](#great-expectations-setup), I introduced dirty data into some of the
columns below and wrote Expectations to test these cases:

- `state`: to test that the values conform to a specific regex pattern. I wrote a new Expectation to ensure that `state` values are always a two-letter uppercase string. Then I changed some the states into 3-letter strings and some into lowercase.
- `postal_code`: although these are strings, I converted some of them into integers for testing.
- `latitude`: latitude is bounded between `[-90,90]`. I wrote Expectations to ensure that lat-long coordinates lie within these bounds.
- `longitude`: longitude is bounded between `[-180,180]`. What if someone mistakenly swapped the lat and long columns? GE should be able to pick that up.
- `stars`: stars given to businesses should be half-integer values between 1 and 5. I introduced numbers such as 3.2 and 4.8.
- `is_open`: an integer value that is either 1 if open or 0 if closed. I introduced `true` and `false` booleans instead of 1 and 0.

These cases were validated in Step 6 and GE caught most of the offending columns, as shown below:

Data Doc overall validation assesment:

![Data Doc presentatipon](assets/images/7.%20data_doc_presentation.png)

Boolean values not caught (they are probably interpreted as 1 and 0):

![True-False dirty data not caught](assets/images/8.%20true_and_false_values_not_detected.png)

Swapping latitude and longitude columns:

![Swapped lat and long](assets/images/9.%20swapped_lat_long.png)

Since the columns were swapped, the latitude values went beyond the `[-90,90]` boundary set by the Expectation:

![Unexpected latitude values](assets/images/10.%20unexpected_latitude_values_caught.png)

The `state` codes that do not conform to the regex pattern were also caught. The `mostly` parameter allowed
some null values, as long as they are no less than 99% of the column:

![Unexpected state codes](assets/images/11.%20unexpected_state_codes_and_some_nulls.png)

It is easier to edit these Expectations inside a Jupyter Notebook rather than editing the JSON files (aka Expectation Suites)
themselves.

**NOTE: The Jupyter Notebooks where Expectations are edited are inside the `uncommited` folder, hidden by the `.gitignore` that was generated
when initializing the GE workspace.** This time, I unignored this folder to allow Git to track these files.

## Conclusion

Overall, GE is quite a promising tool to ensure data quality, but it has not completely matured yet. Even the latest official
version number is 0.13.23. GE also has the Version 3 command line tool to make things easier, but again that is also under development.
The automatic profiling feature is quite powerful, and the Data Doc makes it very convenient to see at a glance which columns don't meet the
Expectations but it has the following drawbacks:

- Requires you to be familiar with the GE jargon to understand how the different components fit together.
- Requires you to do some data preparation before automatic profiling can be done.
- If you don't want to prepare data, then you will have to write Expectations from scratch.
- Some links in the official documentation are broken, and you need to do some digging around before finding what you need.
