# DuckDB Data Pipeline

Build a simple data lake based on Medallion Archirecture, to incrementally ingest and process Github events.

## Clone The Repository

```bash
git clone https://github.com/ppkhai2612/duckdb-pipeline.git
```

## Setup Python Virtual Environment

```bash
cd duckdb-pipeline
python3 -m venv .venv
source .venv/bin/activate
pip3 install -r requirements.txt
```

## Setup LocalStack (AWS emulator)

- Install `lstk`: see [docs](https://docs.localstack.cloud/aws/getting-started/installation/#install-lstk)
- Start LocalStack: `lstk start`. LocalStack profile and AWS credentials in `~/.aws`
- Create S3 buckets: `lstk aws s3api create-bucket --bucket bucket_name` (e.g., `gharchive-bronze`)
- **Note**: After starting LocalStack, it run as a Docker container, so to allow DuckDB to connect, add `s3_endpoint_url_duckdb = localhost:4566` to the `aws` section in `config.ini` (on the same network on the host)

### Helpers

- Verify LocalStack is running: `lstk status`
- Verify buckets created: `lstk aws s3 ls`
- List objects for a given bucket: `lstk aws s3api list-objects --bucket bucket_name`

### Configuration

1. Rename `config.ini.template` to `config.ini`
2. `Edit config.ini` 

    - Fill in your actual AWS S3 credential values in the `[aws]` section. If you are using a S3 compatible storage, setup the `s3_endpoint_url` parameter as well. Otherwise remove the line.
    - Fill in the bucket names in `[datalake]` section for each zone in your data lake.

### Scheduling

There are Python scripts available in `scripts` directory for each phase (Ingestion, Serilisation, Aggregation) for calling with a scheduler like **Crontab**. Following shows sample cron statements to run each pipeline script at an appropriate time. Update the paths to match your setting and also ensure you allow enough time between each pipeline to complete.

```bash
# schedule the ingestion pipeline script to run 15 minutes past each hour
15 * * * * /path/to/your/venv/bin/python3 /path/to/your/duckdb-pipeline/scripts/run_ingest_source_data.py >> /tmp/ingest_source_data.out 2>&1
# schedule the serialisation pipeline script to run 30 minutes past each hour
30 * * * * /path/to/your/venv/bin/python3 /path/to/your/duckdb-pipeline/scripts/run_serialise_raw_data.py >> /tmp/serialise_raw_data.out 2>&1
# schedule the aggregation pipeline script to run 2 hours past midnight
0 2 * * * /path/to/your/venv/bin/python3 /path/to/your/duckdb-pipeline/scripts/run_agg_silver_data.py >> /tmp/aggregate_silver_data.out 2>&1
```