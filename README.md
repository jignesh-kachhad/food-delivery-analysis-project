
---

# Food Delivery Data Pipeline

This repository contains code for a comprehensive data pipeline designed to handle food delivery data. The pipeline involves generating mock order data, streaming the data into an Amazon Kinesis stream, processing the data using a PySpark streaming job on an EMR cluster, and loading the processed data into Amazon Redshift for analysis and reporting.

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Components](#components)
  - [Airflow DAGs](#airflow-dags)
  - [PySpark Script](#pyspark-script)
  - [Data Generator](#data-generator)
- [Setup](#setup)
  - [Prerequisites](#prerequisites)
  - [Installation](#installation)
  - [Configuration](#configuration)
- [Usage](#usage)
- [Contributing](#contributing)
- [License](#license)

## Overview

This project demonstrates a real-time data pipeline using various AWS services such as Amazon Kinesis, EMR, and Redshift, orchestrated by Apache Airflow. The main components of the pipeline include data ingestion, processing, and storage, with a focus on scalability and real-time processing.

## Architecture

1. **Data Generation**: A Python script generates mock food delivery order data and sends it to an Amazon Kinesis stream.
2. **Streaming Data Processing**: A PySpark script running on an EMR cluster processes the data in real-time, performing tasks like parsing, deduplication, and transformation.
3. **Data Storage**: Processed data is loaded into Amazon Redshift for further analysis and reporting.
4. **Orchestration**: Apache Airflow manages the workflow, including data ingestion, processing, and triggering dependent tasks.

![Architecture Diagram](path/to/your/architecture_diagram.png)

## Components

### Airflow DAGs

#### 1. Create and Load Dimension Tables (create_and_load_dim.py)

This DAG sets up the schema and tables in Redshift, loads initial dimension data from S3, and triggers the PySpark streaming job.

```python
# Airflow DAG 1
from airflow import DAG
from airflow.providers.amazon.aws.transfers.s3_to_redshift import S3ToRedshiftOperator
from airflow.providers.postgres.operators.postgres import PostgresOperator
from airflow.utils.dates import days_ago
from datetime import timedelta
from airflow.operators.trigger_dagrun import TriggerDagRunOperator

# ... [Complete DAG code] ...
```

#### 2. Submit PySpark Streaming Job to EMR (submit_pyspark_streaming_job_to_emr.py)

This DAG submits a PySpark streaming job to an EMR cluster.

```python
# Airflow DAG 2
from airflow import DAG
from airflow.models import Variable
from airflow.providers.amazon.aws.operators.emr import EmrAddStepsOperator
from datetime import datetime

# ... [Complete DAG code] ...
```

### PySpark Script

The PySpark script reads data from the Kinesis stream, processes it, and writes it to Redshift.

```python
# pyspark_streaming.py
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, from_json
from pyspark.sql.types import (
    StructType,
    StructField,
    IntegerType,
    StringType,
    DecimalType,
    TimestampType,
)
import argparse

# ... [Complete PySpark script code] ...
```

### Data Generator

The data generator script creates mock order data and streams it into Kinesis.

```python
# data_generator.py
import pandas as pd
import json
import random
from datetime import datetime
from faker import Faker
import boto3
import os
import time

# ... [Complete data generator code] ...
```

## Setup

### Prerequisites

- Python 3.6+
- Apache Airflow
- AWS Account with permissions for Kinesis, EMR, Redshift, and S3
- Docker (optional, for running Airflow)

### Installation

1. **Clone the repository:**

```bash
git clone https://github.com/your-username/food-delivery-data-pipeline.git
cd food-delivery-data-pipeline
```

2. **Install dependencies:**

```bash
pip install -r requirements.txt
```

### Configuration

1. **Set up Airflow connections and variables:**

   - Add connections for AWS and Redshift in the Airflow UI.
   - Add Airflow variables for `redshift_user`, `redshift_password`, `aws_access_key`, and `aws_secret_key`.

2. **Prepare S3 Buckets:**

   - Create S3 buckets for storing dimension data and temporary files.

3. **Upload Initial Dimension Data to S3:**

   - Ensure the CSV files (`dimCustomers.csv`, `dimRestaurants.csv`, `dimDeliveryRiders.csv`) are uploaded to the designated S3 bucket.

## Usage

1. **Start Airflow Scheduler and Web Server:**

```bash
airflow scheduler &
airflow webserver &
```

2. **Trigger the DAGs:**

   - In the Airflow UI, trigger the `create_and_load_dim` DAG to set up the schema and load initial data.
   - The `submit_pyspark_streaming_job_to_emr` DAG will be triggered automatically after loading the data.

3. **Run the Data Generator:**

```bash
python data_generator.py
```

## Contributing

Contributions are welcome! Please fork the repository and submit pull requests for any enhancements or bug fixes.

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

---

Feel free to modify this README file according to your project's specific details and requirements.
