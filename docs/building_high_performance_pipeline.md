# Building a High-Performance Data Pipeline Using DuckDB

Link: https://www.pracdata.io/p/building-data-pipeline-using-duckdb

## Project Overview

The project is implemented as a pure Python application, with minimal dependencies. The only external dependency is a cloud object store for our data lake implementation, which doesn’t have to be AWS S3 specifically—it can be any S3-compatible object store. (e.g., [LocalStack](https://github.com/localstack/localstack))

The pipeline collects the Github Archive datasets, which provide a full record of GitHub activities for public repositories, and analyzes on top of that data.

## Data Lake Architecture

Data lake is a multi-tier architecture, aka Medallion architecture, in which organises data into three zones:
- **Bronze Zone**: Containing raw, unprocessed data ingested from various sources.
- **Silver Zone**: Containing cleaned, conformed and potentially modeled data.
- **Gold Zone**: Containing aggregated and curated data ready for reporting, dashboards, and advanced analytics.

By maintaining this multi-zone architecture, we ensure access to data at various stages of processing—ranging from raw data for detailed analysis to aggregated data for quick insights. This flexibility allows us to meet a wide range of analytical needs while optimising for both storage and query performance.

## Partitioning Scheme

Each zone will use an appropriate partitioning scheme to optimise data ingestion, query performance and improve overall efficiency.

There are no overlaps between individual job runs, and if a pipeline fails or an anomaly is detected in the data, we can safely rerun the job for that specific period. The pipeline will replace the affected partition and its contents, avoiding any negative side effects, data duplication or inconsistencies.

This approach is often referred to as functional data processing using immutable partitions. It was popularised by Maxim Beauchemen in [this blog](https://maximebeauchemin.medium.com/functional-data-engineering-a-modern-paradigm-for-batch-data-processing-2327ec32c42a).

Following the functional data processing pattern, data ingested hourly from the source and partition it by both day and hour in the Bronze Zone. Each hourly job writes atomically to its designated partition, ensuring data integrity at this level of granularity. The same partitioning strategy applied in the Silver Zone, since the transformation from Bronze to Silver will also run on an hourly basis.

In the Gold Zone, partitioning will be done only by day, since the aggregation job runs on daily internal. The daily job will produce results for the previous day's data from the Silver Zone.

This approach strikes the right balance between performance and storage—using fine-grained partitions in the early stages and higher-level aggregation in the final layer.

## Data Pipeline Architecture

The end-to-end data pipeline is broken down into three key steps:
- **Step 1** - Ingest the hourly GitHub Archive batch dataset from the [gharchive.org](https://www.gharchive.org/) website over HTTP and load it into the Bronze Zone of data lake.
- **Step 2** - Run an hourly transformation pipeline to clean and serialise the JSON files which loads the results into the Silver Zone.
- **Step 3** - Run a daily transformation pipeline to aggregate the data from the previous day and store it in the Gold Zone.

## Data Source Exploration

The source data are available from [gharchive.org](https://www.gharchive.org/) and the data dumps are made available on different intervals such as hourly and daily.

## Data Ingestion - Bronze Zone

To implement data ingestion pipeline, we need to identify things about the source system:
- Data Source: gharchive.com
- Interface: REST API
- Protocol: HTTP
- Data Type: `json.gz` files 
- Frequency: Hourly

To download, use the `requests` library to stream and buffer the file from the source, followed by the `boto3` library to upload the file to S3, publishing it to the Bronze zone of the data lake.

Class `DataLakeIngestor`, which fetches data from the GitHub Archive for a specific hour. It uses Python's `requests` library to download the compressed JSON file in memory. The data is then uploaded directly to the specified S3 bucket, with the S3 key based on the date and hour.

To execute the ingestion pipeline all we need to do is passing a timestamp to the ingestion method `ingest_hourly_gharchive()` , which will be used to determine the interval for the JSON file to collect and load from source.

**Output**:

```bash
2026-10-22 12:31:40,346 - INFO - The URL to download is: http://data.gharchive.org/2026-10-22-4.json.gz
2026-10-22 12:31:49,414 - INFO - Transferred: 262144 bytes to S3 bucket
2026-10-22 12:31:49,414 - INFO - Transferred: 262144 bytes to S3 bucket
2026-10-22 12:31:49,414 - INFO - Transferred: 262144 bytes to S3 bucket
...
2026-10-22 12:31:49,591 - INFO - Successfully uploaded gharchive/events/2026-10-22/04/2026-10-22-4.json.gz to gharchive-bronze
2026-10-22 12:31:49,592 - INFO - Successfully ingested data for 2026-10-22 04:00:00
```

### Managing Configurations and Secrets

Configuration settings, such as AWS credentials and bucket names, are stored in a configuration file (`config.ini`), to keep the code free from any sensitive and static data.

We utilise Python's `configparser` library to load these configurations into the code.

## Raw Data Serialisation - Silver Zone

After ingesting the raw GitHub Archive data into our data lake’s Bronze layer, the next critical step in the pipeline is to clean and serialise this data, preparing it for the Silver layer. This is where DuckDB plays a key role, performing the necessary transformations within the data lake.

The transformation logic is encapsulated in the `DataLakeTransformer` class. This class provides two primary methods: `serialise_raw_data()` for data cleaning and serialisation, and `aggregate_silver_data()` for aggregating the data. Key steps:
- **Source and Sink Configuration**: It determines the source (Bronze) and sink (Silver) bucket names based on the parameters specified in the configuration file (`config.ini`).
- **Data Loading**: The logic for importing the raw JSON data into an in-memory table using DuckDB’s [Relational API](https://duckdb.org/docs/2.0/clients/python/relational_api) is defined in `register_raw_gharchive()`.

### Data Modeling

Before we serialise the raw data into Parquet format we need to perform a data modeling exercise and only select the attributes we are interested in. The result of the SQL query encapsulated in the method `clean_raw_gharchive()`

**Data Export**: After the data has been cleaned, the result is written to the Silver layer in Parquet format using DuckDB’s engine:

```python
gharchive_clean_result.write_parquet(sink_path)
```

### Serialisation Performance

When running the serialisation process close to the data, and using DuckDB to handle the transformation, the entire process completes in under a minute.

This efficiency makes DuckDB an excellent choice for lightweight, in-place data transformations, especially when working with local or cloud-based object storage systems like S3.

**Output**:

```bash
2026-10-22 13:04:13,082 - INFO - DuckDB connection initiated
2026-10-22 13:04:13,082 - INFO - DuckDB - collect source data files: s3://gharchive-bronze/gharchive/events/2026-10-22/04/*
2026-10-22 13:04:16,282 - INFO - DuckDB - clean data
2026-10-22 13:04:16,397 - INFO - DuckDB - serialise and export cleaned data to s3://gharchive-silver/gharchive/events/2026-10-22/04/clean_20261022_04.parquet
2026-10-22 13:04:16,677 - INFO - Successfully serialised raw data for 2026-10-22 04:00:00
```

### Key Takeaways

The use of DuckDB for this transformation process is a key design choice:
- **In-Memory Processing**: DuckDB allows for efficient in-memory processing of the data, which is particularly useful for the typically large GitHub Archive datasets.
- **SQL Interface**: The use of SQL for data modeling provides a familiar and powerful interface for data transformations.
- **Parquet Writing**: DuckDB has very efficient Parquet reader and writer for fast and efficient serialisation of data from primitive data types such as JSON and CSV, while eliminating the need for intermediate steps or additional libraries.

## Data Aggregation - Gold Zone

After modeling and serialising our GitHub Archive raw data into the Silver zone, the next step in our data pipeline is to aggregate this data and publish it to the Gold zone on daily basis.

`aggregate_silver_data()` method for performing the daily aggregation
- **Source and Sink Configuration**: It identifies the source (Silver) and sink (Gold) bucket names based on the configuration.
- **Data Loading and Aggregation**: The aggregation logic is defined by using SQL applied to a DuckDB virtual table that has been defined over Parquet files present in the Silver zone. This aggregation focuses on counting GitHub events by type (e.g., stars, pull requests), repository, and date, providing an aggregated view of GitHub activity within a daily time window.

**Data Export**: The aggregated data is subsequently written to the Gold zone in Parquet format:

```python
gharchive_agg_result.write_parquet(sink_path)
```

### Aggregation Performance

The transformation pipeline on a cloud VM takes less than a minute to aggregate 24 Parquet files containing nearly 6 million records, and serialise the result into a Parquet file published in the Gold zone.

This efficiency underscores the capability of DuckDB to handle small to medium-scale data transformations quickly and effectively.

**Output**:

```bash
2026-10-22 20:08:04,628 - INFO - DuckDB connection initiated
2026-10-22 20:08:04,629 - INFO - DuckDB - aggregate silver data in s3://gharchive-silver/gharchive/events/2026-10-22/*/*.parquet
2026-10-22 20:08:04,752 - INFO - DuckDB - export aggregated data to s3://gharchive-gold/gharchive/events/2026-10-22/agg_20261022.parquet
2026-10-22 20:08:04,916 - INFO - Successfully aggregated bronze data for 2026-10-22 00:00:00
```

### Key Takeaways

The use of DuckDB for this data aggregation process offers several advantages:
- **Efficient Processing**: DuckDB's column-oriented storage and processing is well-suited for analytical queries and aggregations.
- **SQL Interface**: The use of SQL for aggregation in Python provides a familiar and powerful interface for complex data transformations.
- **Efficient Parquet Integration**: DuckDB's native support for Parquet files allows for efficient reading and writing of data in this format.

## Orchestration & Scheduling

In a production environment, you would typically use a workflow orchestrator like Apache Airflow, Dagster or Prefect to manage the pipeline execution.

However, since the goal here was to demonstrate how DuckDB can be used for data transformation, cron is used for simplicity.

## Interactive Data Analytics

Once the data is prepared for analytics, DuckDB's efficient querying capabilities allow us to easily extract meaningful insights from the aggregated data, making it an excellent tool for interactive data analysis.