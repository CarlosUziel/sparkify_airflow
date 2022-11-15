<div id="top"></div>

<!-- TABLE OF CONTENTS -->
<details>
  <summary>Table of Contents</summary>
  <ol>
    <li>
      <a href="#about-the-project">About The Project</a>
      <ul>
        <li><a href="#premise">Premise</a></li>
        <li><a href="#goal">Goal</a></li>
        <li><a href="#data">Data</a></li>
      </ul>
    </li>
    <li>
      <a href="#getting-started">Getting Started</a>
      <ul>
        <li><a href="#setting-up-a-conda-environment">Setting up a conda environment</a></li>
        <li><a href="#setting-up-an-amazon-emr-cluster">Setting up an Amazon EMR cluster</a></li>
      </ul>
    </li>
    <li><a href="#usage">Usage</a></li>
    <li><a href="#additional-notes">Additional Notes</a></li>
    <li><a href="#license">License</a></li>
    <li><a href="#contact">Contact</a></li>
    <li><a href="#acknowledgments">Acknowledgments</a></li>
  </ol>
</details>

# Sparkify (using Apache Airflow)

A project from the [Data Engineer Nanodegree Program at Udacity](https://www.udacity.com/course/data-engineer-nanodegree--nd027) to practice data lakes on the cloud using Spark and AWS services.

## About The Project

### Premise

> A music streaming startup, Sparkify, has grown their user base and song database even more and want to move their data warehouse to a data lake. Their data resides in S3, in a directory of JSON logs on user activity on the app, as well as a directory with JSON metadata on the songs in their app.
>
> As their data engineer, you are tasked with building an ETL pipeline that extracts their data from S3, processes them using Spark, and loads the data back into S3 as a set of dimensional tables. This will allow their analytics team to continue finding insights in what songs their users are listening to.
>
> You'll be able to test your database and ETL pipeline by running queries given to you by the analytics team from Sparkify and compare your results with their expected results.

<p align="right">(<a href="#top">back to top</a>)</p>

### Goal

The goal of this project is to apply what I have learned on Spark and data lakes to build an ETL pipeline for a data lake hosted on S3. Data will be loaded from an S3 bucket, processed into dimensional and fact tables and stored into another S3 bucket.

<p align="right">(<a href="#top">back to top</a>)</p>

### Data

Data is stored in an S3 bucket:

- **Song data**: `s3://udacity-dend/song_data`
- **Log data**: `s3://udacity-dend/log_data`

Log data json path: `s3://udacity-dend/log_json_path.json`

#### The song dataset

A subset of real data from the [Million Song Dataset](http://millionsongdataset.com/). Each file is in JSON format and contains metadata about a song and the artist of that song. Files are located under in `s3://udacity-dend/song_data`.

For example, this is how the first file (`TRAAAAW128F429D538.json`) looks like:

```json
{
  "num_songs": 1,
  "artist_id": "ARD7TVE1187B99BFB1",
  "artist_latitude": null,
  "artist_longitude": null,
  "artist_location": "California - LA",
  "artist_name": "Casual",
  "song_id": "SOMZWCG12A8C13C480",
  "title": "I Didn't Mean To",
  "duration": 218.93179,
  "year": 0
}

```

#### Log dataset

It is composed of log files in NDJSON format generated by this [event simulator](https://github.com/Interana/eventsim) based on the songs in the dataset above. These simulate activity logs from a music streaming app based on specified configurations. Files are located under `s3://udacity-dend/log_data`.

The log files are named following a date pattern (`{year}_{month}_{day}_events.json`), and below is the first line of the first file (`2018-11-01-events`) as an example:

```json
{
  "artist": null,
  "auth": "Logged In",
  "firstName": "Walter",
  "gender": "M",
  "itemInSession": 0,
  "lastName": "Frye",
  "length": null,
  "level": "free",
  "location": "San Francisco-Oakland-Hayward, CA",
  "method": "GET",
  "page": "Home",
  "registration": 1540919166796.0,
  "sessionId": 38,
  "song": null,
  "status": 200,
  "ts": 1541105830796,
  "userAgent": "\"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/36.0.1985.143 Safari/537.36\"",
  "userId": "39"
}

```

<p align="right">(<a href="#top">back to top</a>)</p>

### Data Schema

The new tables follow a star schema that is better visualized through an Entity Relationship Diagram (ERD). The following image shows the ERD of an example PostreSQL database with the final tables defined.

![Sparkify ERD](images/ERD.png)

Here, *songplays* is a **fact table**, whereas *artists*, *songs*, *time* and *users* are **dimension tables**. These tables make it easy to query relevant information with few joins.

<p align="right">(<a href="#top">back to top</a>)</p>

## Getting Started

To make use of this project, I recommend managing the required dependencies with Anaconda.

### Setting up a conda environment

Install miniconda:

```bash
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh
```

Install mamba:

```bash
conda install -n base -c conda-forge mamba
```

Install environment using provided file:

```bash
mamba env create -f environment.yml # alternatively use environment_core.yml if base system is not debian
mamba activate sparkify_spark
```

### Setting up an Amazon EMR cluster

**Create an IAM user:**

  1. IAM service is a global service, meaning newly created IAM users are not restricted to a specific region by default.
  2. Go to [AWS IAM service](https://console.aws.amazon.com/iam/home#/users) and click on the "**Add user**" button to create a new IAM user in your AWS account.
  3. Choose a name of your choice.
  4. Select "**Programmatic access**" as the access type. Click Next.
  5. Choose the **Attach existing policies directly tab**, and select the "**AdministratorAccess**" (<span style="color:red;font-weight:bold">Only for testing purposes!</span>). Click Next.
  6. Skip adding any tags. Click Next.
  7. Review and create the user. It will show you a pair of access key ID and secret.
  8. Take note of the pair of access key ID and secret. This pair is collectively known as Access key.

**Save access key and secret locally:**

  1. Create a new file, `_user.cfg`, and add the following:

   ```bash
   AWS_ACCESS_KEY_ID = <YOUR_AWS_KEY>
   AWS_SECRET_ACCESS_KEY = <YOUR_AWS_SECRET>
   ```

  2. This file will be loaded internally to connect to AWS and perform various operations.
  3. **DO NOT SHARE THIS FILE WITH ANYONE!** I recommend adding this file to .gitignore to avoid accidentally pushing it to a git repository: `printf "\n_user.cfg\n" >> .gitignore`.

**Create cluster:**

Since the python scripts that are part of this project are meant to be run within an EMR cluster, the easiest way is to create the EMR cluster using the AWS UI. Check this [great guide](https://hands-on.cloud/working-with-emr-in-python-using-boto3/) on how to do it using boto3.

Create cluster with the following settings (using advanced options):

- **Release**: emr-6.8.0 or later.
- **Applications**: Hadoop 3.2.1, JupyterEnterpriseGateway 2.1.0, and Spark 3.3.0 (or later versions).
- **Instance type**: m3.xlarge.
- **Number of instances**: 3.
- **EC2 key pair**: Proceed without an EC2 key pair or feel free to use one if you'd like to.

It is a good idea to set cluster auto-termination on (e.g. to one hour after being idle).

<span style="color:red;font-weight:bold">DO NOT FORGET TO TERMINATE YOUR CLUSTER WHEN FINISHED WORKING ON THE PROJECT TO AVOID UNWANTED COSTS!</span>

## Usage

Project structure:

- `notebooks`: contains the main Jupyter notebook to run the project (`notebooks/main.ipynb`).
- `src`: contains the source files and scripts to build and populate the Data Lake.

Ensure you have set the `PYTHONPATH` environment variable as needed (e.g., `PYTHONPATH=~/sparkify_spark/src`)

The whole project can be run as follows:

```bash
python src/etl.py
```

Alternatively, follow along `notebooks/main.ipynb`.

<p align="right">(<a href="#top">back to top</a>)</p>

## Additional Notes

Source files formatted using the following commands:

```bash
isort .
autoflake -r --in-place --remove-unused-variable --remove-all-unused-imports --ignore-init-module-imports .
black .
```

## License

Distributed under the MIT License. See `LICENSE` for more information.

<p align="right">(<a href="#top">back to top</a>)</p>

## Contact

[Carlos Uziel Pérez Malla](https://www.carlosuziel-pm.dev/)

[GitHub](https://github.com/CarlosUziel) - [Google Scholar](https://scholar.google.es/citations?user=tEz_OeIAAAAJ&hl=es&oi=ao) - [LinkedIn](https://at.linkedin.com/in/carlos-uziel-p%C3%A9rez-malla-323aa5124) - [Twitter](https://twitter.com/perez_malla)

<p align="right">(<a href="#top">back to top</a>)</p>

## Acknowledgments

This README includes a summary of the official project description provided to the students of the [Data Engineer Nanodegree Program at Udacity](https://www.udacity.com/course/data-engineer-nanodegree--nd027).

<p align="right">(<a href="#top">back to top</a>)</p>
