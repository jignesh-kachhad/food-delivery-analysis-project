
# Food Delivery AWS Streaming Pipeline

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

<details>
 <summary>Airflow DAG 1: Create and Load Dimension Tables</summary>

This DAG sets up the schema and tables in Redshift, loads initial dimension data from S3, and triggers the PySpark streaming job.

```python

from airflow import DAG
from airflow.providers.amazon.aws.transfers.s3_to_redshift import S3ToRedshiftOperator
from airflow.providers.postgres.operators.postgres import PostgresOperator
from airflow.utils.dates import days_ago
from datetime import timedelta
from airflow.operators.trigger_dagrun import TriggerDagRunOperator

# from airflow.operators.dagrun_operator import TriggerDagRunOperator

default_args = {
    "owner": "airflow",
    "depends_on_past": False,
    "email_on_failure": False,
    "email_on_retry": False,
    "retries": 1,
    "retry_delay": timedelta(minutes=5),
}

with DAG(
    "create_and_load_dim",
    default_args=default_args,
    description="ETL for food delivery data into Redshift",
    schedule_interval=None,
    start_date=days_ago(1),
    catchup=False,
    tags=["Create Schema"],
) as dag:

    # Create schema if it doesn't exist
    create_schema = PostgresOperator(
        task_id="create_schema",
        postgres_conn_id="redshift_connection_id",
        sql="CREATE SCHEMA IF NOT EXISTS food_delivery_datamart;",
    )

    # Drop tables if they exist
    drop_dimCustomers = PostgresOperator(
        task_id="drop_dimCustomers",
        postgres_conn_id="redshift_connection_id",
        sql="DROP TABLE IF EXISTS food_delivery_datamart.dimCustomers;",
    )

    drop_dimRestaurants = PostgresOperator(
        task_id="drop_dimRestaurants",
        postgres_conn_id="redshift_connection_id",
        sql="DROP TABLE IF EXISTS food_delivery_datamart.dimRestaurants;",
    )

    drop_dimDeliveryRiders = PostgresOperator(
        task_id="drop_dimDeliveryRiders",
        postgres_conn_id="redshift_connection_id",
        sql="DROP TABLE IF EXISTS food_delivery_datamart.dimDeliveryRiders;",
    )

    drop_factOrders = PostgresOperator(
        task_id="drop_factOrders",
        postgres_conn_id="redshift_connection_id",
        sql="DROP TABLE IF EXISTS food_delivery_datamart.factOrders;",
    )

    # Create dimension and fact tables
    create_dimCustomers = PostgresOperator(
        task_id="create_dimCustomers",
        postgres_conn_id="redshift_connection_id",
        sql="""
            CREATE TABLE food_delivery_datamart.dimCustomers (
                CustomerID INT PRIMARY KEY,
                CustomerName VARCHAR(255),
                CustomerEmail VARCHAR(255),
                CustomerPhone VARCHAR(50),
                CustomerAddress VARCHAR(500),
                RegistrationDate DATE
            );
        """,
    )

    create_dimRestaurants = PostgresOperator(
        task_id="create_dimRestaurants",
        postgres_conn_id="redshift_connection_id",
        sql="""
            CREATE TABLE food_delivery_datamart.dimRestaurants (
                RestaurantID INT PRIMARY KEY,
                RestaurantName VARCHAR(255),
                CuisineType VARCHAR(100),
                RestaurantAddress VARCHAR(500),
                RestaurantRating DECIMAL(3,1)
            );
        """,
    )

    create_dimDeliveryRiders = PostgresOperator(
        task_id="create_dimDeliveryRiders",
        postgres_conn_id="redshift_connection_id",
        sql="""
            CREATE TABLE food_delivery_datamart.dimDeliveryRiders (
                RiderID INT PRIMARY KEY,
                RiderName VARCHAR(255),
                RiderPhone VARCHAR(50),
                RiderVehicleType VARCHAR(50),
                VehicleID VARCHAR(50),
                RiderRating DECIMAL(3,1)
            );
        """,
    )

    create_factOrders = PostgresOperator(
        task_id="create_factOrders",
        postgres_conn_id="redshift_connection_id",
        sql="""
            CREATE TABLE food_delivery_datamart.factOrders (
                OrderID INT PRIMARY KEY,
                CustomerID INT REFERENCES food_delivery_datamart.dimCustomers(CustomerID),
                RestaurantID INT REFERENCES food_delivery_datamart.dimRestaurants(RestaurantID),
                RiderID INT REFERENCES food_delivery_datamart.dimDeliveryRiders(RiderID),
                OrderDate TIMESTAMP WITHOUT TIME ZONE,
                DeliveryTime INT,
                OrderValue DECIMAL(8,2),
                DeliveryFee DECIMAL(8,2),
                TipAmount DECIMAL(8,2),
                OrderStatus VARCHAR(50)
            );
        """,
    )

    # Load data into dimension tables from S3
    load_dimCustomers = S3ToRedshiftOperator(
        task_id="load_dimCustomers",
        schema="food_delivery_datamart",
        table="dimCustomers",
        s3_bucket="food-delivery-data-analysis1",
        s3_key="dims/dimCustomers.csv",
        copy_options=["CSV", "IGNOREHEADER 1", "QUOTE as '\"'"],
        aws_conn_id="aws_default",
        redshift_conn_id="redshift_connection_id",
    )

    load_dimRestaurants = S3ToRedshiftOperator(
        task_id="load_dimRestaurants",
        schema="food_delivery_datamart",
        table="dimRestaurants",
        s3_bucket="food-delivery-data-analysis1",
        s3_key="dims/dimRestaurants.csv",
        copy_options=["CSV", "IGNOREHEADER 1", "QUOTE as '\"'"],
        aws_conn_id="aws_default",
        redshift_conn_id="redshift_connection_id",
    )

    load_dimDeliveryRiders = S3ToRedshiftOperator(
        task_id="load_dimDeliveryRiders",
        schema="food_delivery_datamart",
        table="dimDeliveryRiders",
        s3_bucket="food-delivery-data-analysis1",
        s3_key="dims/dimDeliveryRiders.csv",
        copy_options=["CSV", "IGNOREHEADER 1", "QUOTE as '\"'"],
        aws_conn_id="aws_default",
        redshift_conn_id="redshift_connection_id",
    )

    trigger_spark_streaming_dag = TriggerDagRunOperator(
        task_id="trigger_spark_streaming_dag",
        trigger_dag_id="submit_pyspark_streaming_job_to_emr",
    )

# First, create the schema
create_schema >> [
    drop_dimCustomers,
    drop_dimRestaurants,
    drop_dimDeliveryRiders,
    drop_factOrders,
]

# Once the existing tables are dropped, proceed to create new tables
drop_dimCustomers >> create_dimCustomers
drop_dimRestaurants >> create_dimRestaurants
drop_dimDeliveryRiders >> create_dimDeliveryRiders
drop_factOrders >> create_factOrders

[
    create_dimCustomers,
    create_dimRestaurants,
    create_dimDeliveryRiders,
] >> create_factOrders

# After each table is created, load the corresponding data
create_dimCustomers >> load_dimCustomers
create_dimRestaurants >> load_dimRestaurants
create_dimDeliveryRiders >> load_dimDeliveryRiders

[
    load_dimCustomers,
    load_dimRestaurants,
    load_dimDeliveryRiders,
] >> trigger_spark_streaming_dag

```
</details>
<details>
 <summary>Airflow DAG 2: Submit PySpark Streaming Job to EMR </summary>

This DAG submits a PySpark streaming job to an EMR cluster.

```python

from airflow import DAG
from airflow.models import Variable
from airflow.providers.amazon.aws.operators.emr import EmrAddStepsOperator
from datetime import datetime

dag = DAG(
    "submit_pyspark_streaming_job_to_emr",
    start_date=datetime(2021, 1, 1),
    catchup=False,
    tags=["streaming"],
)

spark_packages = [
    "com.qubole.spark:spark-sql-kinesis_2.12:1.2.0_spark-3.0",
    "io.github.spark-redshift-community:spark-redshift_2.12:6.2.0-spark_3.5",
]
packages_list = ",".join(spark_packages)

jdbc_jar_s3_path = "s3://food-delivery-data-analysis1/redshift-connector-jar/redshift-jdbc42-2.1.0.12.jar"

# Fetch Redshift credentials from Airflow Variables
redshift_user = Variable.get("redshift_user")
redshift_password = Variable.get("redshift_password")
aws_access_key = Variable.get("aws_access_key")
aws_secret_key = Variable.get("aws_secret_key")

step_adder = EmrAddStepsOperator(
    task_id="add_step",
    job_flow_id="j-1C8HYM1LEFEIX",
    aws_conn_id="aws_default",
    steps=[
        {
            "Name": "Run PySpark Streaming Script",
            "ActionOnFailure": "CONTINUE",
            "HadoopJarStep": {
                "Jar": "command-runner.jar",
                "Args": [
                    "spark-submit",
                    "--deploy-mode",
                    "cluster",
                    "--num-executors",
                    "3",
                    "--executor-memory",
                    "6G",
                    "--executor-cores",
                    "3",
                    "--packages",
                    packages_list,
                    "--jars",
                    jdbc_jar_s3_path,
                    "s3://food-delivery-data-analysis1/pyspark_script/pyspark_streaming.py",
                    "--redshift_user",
                    redshift_user,
                    "--redshift_password",
                    redshift_password,
                    "--aws_access_key",
                    aws_access_key,
                    "--aws_secret_key",
                    aws_secret_key,
                ],
            },
        }
    ],
    dag=dag,
)

```
</details>

### PySpark Script
<details>
<summary>PySpark Script</summary>

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

parser = argparse.ArgumentParser(description="PySpark Streaming Job Arguments")
parser.add_argument("--redshift_user", required=True, help="Redshift Username")
parser.add_argument("--redshift_password", required=True, help="Redshift Password")
parser.add_argument("--aws_access_key", required=True, help="aws_access_key")
parser.add_argument("--aws_secret_key", required=True, help="aws_secret_key")
args = parser.parse_args()

appName = "KinesisToRedshift"
kinesisStreamName = "incoming-food-order-data-streaming"
kinesisRegion = "ap-south-1"
checkpointLocation = "s3://stream-checkpointing1/kinesis-to-redshift/"
redshiftJdbcUrl = f"jdbc:redshift://redshift-cluster-1.coyx8twvpafs.ap-south-1.redshift.amazonaws.com:5439/dev"
redshiftTable = "food_delivery_datamart.factOrders"
tempDir = "s3://redshift-temp-data-kcd/temp-data/streaming_temp/"

# Define the schema of the incoming JSON data from Kinesis
schema = StructType(
    [
        StructField("OrderID", IntegerType(), True),
        StructField("CustomerID", IntegerType(), True),
        StructField("RestaurantID", IntegerType(), True),
        StructField("RiderID", IntegerType(), True),
        StructField("OrderDate", TimestampType(), True),
        StructField("DeliveryTime", IntegerType(), True),
        StructField("OrderValue", DecimalType(8, 2), True),
        StructField("DeliveryFee", DecimalType(8, 2), True),
        StructField("TipAmount", DecimalType(8, 2), True),
        StructField("OrderStatus", StringType(), True),
    ]
)

spark = SparkSession.builder.appName(appName).getOrCreate()

df = (
    spark.readStream.format("kinesis")
    .option("streamName", kinesisStreamName)
    .option("startingPosition", "latest")
    .option("region", kinesisRegion)
    .option("awsUseInstanceProfile", "false")
    .option("endpointUrl", "https://kinesis.ap-south-1.amazonaws.com")
    .option("awsAccessKeyId", args.aws_access_key)
    .option("awsSecretKey", args.aws_secret_key)
    .load()
)

print("Consuming From Read Stream...")
parsed_df = (
    df.selectExpr("CAST(data AS STRING)")
    .select(from_json(col("data"), schema).alias("parsed_data"))
    .select("parsed_data.*")
)

# Perform stateful deduplication
deduped_df = parsed_df.withWatermark("OrderDate", "10 minutes").dropDuplicates(
    ["OrderID"]
)


# Writing Data to Redshift
def write_to_redshift(batch_df, batch_id):
    batch_df.write.format("jdbc").option("url", redshiftJdbcUrl).option(
        "user", args.redshift_user
    ).option("password", args.redshift_password).option(
        "dbtable", redshiftTable
    ).option(
        "tempdir", tempDir
    ).option(
        "driver", "com.amazon.redshift.jdbc.Driver"
    ).mode(
        "append"
    ).save()


# Write the deduplicated data to Redshift
query = (
    deduped_df.writeStream.foreachBatch(write_to_redshift)
    .outputMode("append")
    .trigger(processingTime="5 seconds")
    .option("checkpointLocation", checkpointLocation)
    .start()
)

print("Current batch written in Redshift")

query.awaitTermination()

```
</details>

### Data Generator
<details>

 <summary>Data Generator</summary>

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

# Initialize Faker and Boto3 Kinesis client
fake = Faker()
kinesis_client = boto3.client("kinesis")

# Directory setup
base_directory = os.path.dirname(__file__)
data_directory = os.path.join(base_directory, "data_for_dims")


def load_ids_from_csv(file_name, key_attribute):
    """Load IDs from the specified CSV file within the data directory."""
    file_path = os.path.join(data_directory, file_name)
    df = pd.read_csv(file_path)
    return df[key_attribute].tolist()


def generate_order(customer_ids, restaurant_ids, rider_ids, order_id):
    """Generate a mock order record."""
    order = {
        "OrderID": order_id,
        "CustomerID": random.choice(customer_ids),
        "RestaurantID": random.choice(restaurant_ids), 
        "RiderID": random.choice(rider_ids),
        "OrderDate": fake.date_time_between(
            start_date="-30d", end_date="now"
        ).isoformat(),
        "DeliveryTime": random.randint(15, 60),  # Delivery time in minutes
        "OrderValue": round(random.uniform(10, 100), 2),
        "DeliveryFee": round(random.uniform(2, 10), 2),
        "TipAmount": round(random.uniform(0, 20), 2),
        "OrderStatus": random.choice(
            ["Delivered", "Cancelled", "Processing", "On the way"]
        ),
    }
    return order


def send_order_to_kinesis(stream_name, order):
    """Send the generated order to a Kinesis stream."""
    response = kinesis_client.put_record(
        StreamName=stream_name,
        Data=json.dumps(order),
        PartitionKey=str(order["OrderID"]),  # Using OrderID as the partition key
    )
    print(f"Sent order to Kinesis with Sequence Number: {response['SequenceNumber']}")


# Load mock data IDs
customer_ids = load_ids_from_csv("dimCustomers.csv", "CustomerID")
restaurant_ids = load_ids_from_csv("dimRestaurants.csv", "RestaurantID")
rider_ids = load_ids_from_csv("dimDeliveryRiders.csv", "RiderID")

# Kinesis Stream Name
stream_name = "incoming-food-order-data-stream"

# Order ID initialization
order_id = 5000

for _ in range(10000):
    order = generate_order(customer_ids, restaurant_ids, rider_ids, order_id)
    print(order)
    send_order_to_kinesis(stream_name, order)
    order_id += 1  # Increment OrderID for the next order

```
</details>

## Setup

### Prerequisites

- Python 3.12
- Apache Airflow
- AWS Account with permissions for Kinesis, EMR, Redshift, and S3
- Airflow

### Installation

- Clone the repository:

```bash
git clone https://github.com/your-username/food-delivery-aws-streaming-pipeline.git
```

### Configuration

- Set up Airflow connections and variables:

   - Add connections for AWS and Redshift in the Airflow UI.
   - Add Airflow variables for `redshift_user`, `redshift_password`, `aws_access_key`, and `aws_secret_key`.

- Prepare S3 Buckets:

   - Create S3 buckets for storing dimension data and temporary files.

- Upload Initial Dimension Data to S3:

   - Ensure the CSV files (`dimCustomers.csv`, `dimRestaurants.csv`, `dimDeliveryRiders.csv`) are uploaded to the designated S3 bucket.



