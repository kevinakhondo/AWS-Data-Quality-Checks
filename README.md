# AWS-Data-Quality-Checks
This is a continuation of _Serverless-ETL-Pipeline-With-AWS-Glue-and-Athena_ Where we perform data quality Checks using terraform
Data quality ensures the created pipeline catches bad data before it contaminates downstream analytics.

## Step 1: Create Data Quality Rules File
What this does:
- Defines 8 data quality rules in JSON format.
- Critical severity: Pipeline should fail if violated.
- Warning severity: Log the issue, but continue.

  The Rules are:
- TransactionIDUniqueness: Ensures no duplicate transaction IDs
- QuantityPositive: Catches negative or zero quantities
- PricePositive: Catches negative or zero prices
- NoNullTransactionID: Every record must have an ID
- NoNullTimestamp: Every record must have a timestamp
- ValidRegion: Region must be one of 6 valid values
- PriceRange: Catches unrealistic prices (< $1 or > $10,000)
- TimestampFormat: Validates timestamp column type

Create data_quality_rules.json and add the following
```
{
  "Version": 1.0,
  "Rules": [
    {
      "Name": "TransactionIDUniqueness",
      "Description": "Transaction IDs must be unique",
      "Rule": "IsUnique \"transaction_id\"",
      "Severity": "Critical"
    },
    {
      "Name": "QuantityPositive",
      "Description": "Quantity must be greater than 0",
      "Rule": "ColumnValues \"quantity\" > 0",
      "Severity": "Critical"
    },
    {
      "Name": "PricePositive",
      "Description": "Price must be greater than 0",
      "Rule": "ColumnValues \"price\" > 0",
      "Severity": "Critical"
    },
    {
      "Name": "NoNullTransactionID",
      "Description": "Transaction ID cannot be null",
      "Rule": "IsComplete \"transaction_id\"",
      "Severity": "Critical"
    },
    {
      "Name": "NoNullTimestamp",
      "Description": "Timestamp cannot be null",
      "Rule": "IsComplete \"timestamp\"",
      "Severity": "Critical"
    },
    {
      "Name": "ValidRegion",
      "Description": "Region must be from predefined list",
      "Rule": "ColumnValues \"region\" in [\"us-east\", \"us-west\", \"eu-west\", \"eu-central\", \"ap-south\", \"ap-northeast\"]",
      "Severity": "Warning"
    },
    {
      "Name": "PriceRange",
      "Description": "Price should be between 1 and 10000",
      "Rule": "ColumnValues \"price\" between 1.0 and 10000.0",
      "Severity": "Warning"
    },
    {
      "Name": "TimestampFormat",
      "Description": "Timestamp must match expected format",
      "Rule": "ColumnDataType \"timestamp\" = \"String\"",
      "Severity": "Warning"
    }
  ]
}
```
## Step 2: Update the Terraform to include the data quality

Navigate to terraform folder

```
cd ~/glue-etl-pipeline/terraform
```

Create the _data_quality.tf_ file and add the following:

```
# Upload data quality rules to S3
resource "aws_s3_object" "data_quality_rules" {
  bucket = aws_s3_bucket.data_lake.id
  key    = "scripts/data_quality_rules.json"
  source = "${path.module}/../scripts/data_quality_rules.json"
  etag   = filemd5("${path.module}/../scripts/data_quality_rules.json")
}

# Data Quality Job
resource "aws_glue_job" "data_quality_job" {
  name     = "${var.project_name}-data-quality-job"
  role_arn = aws_iam_role.glue_service_role.arn
  
  glue_version      = var.glue_version
  worker_type       = var.worker_type
  number_of_workers = 2
  
  command {
    name            = "glueetl"
    script_location = "s3://${aws_s3_bucket.data_lake.id}/scripts/data_quality_job.py"
    python_version  = "3"
  }

  default_arguments = {
    "--job-language"                     = "python"
    "--enable-metrics"                   = "true"
    "--enable-glue-datacatalog"          = "true"
    "--enable-continuous-cloudwatch-log" = "true"
    "--SOURCE_BUCKET"                    = aws_s3_bucket.data_lake.id
    "--RULES_PATH"                       = "s3://${aws_s3_bucket.data_lake.id}/scripts/data_quality_rules.json"
    "--DATABASE_NAME"                    = aws_glue_catalog_database.data_lake_db.name
    "--TABLE_NAME"                       = "raw"
  }

  execution_property {
    max_concurrent_runs = 1
  }

  timeout = 30
}

# CloudWatch Log Group for Data Quality Results
resource "aws_cloudwatch_log_group" "data_quality_logs" {
  name              = "/aws/glue/data-quality/${var.project_name}"
  retention_in_days = 30
}

# Add data quality check to workflow (runs after raw crawler)
resource "aws_glue_trigger" "run_data_quality" {
  name          = "${var.project_name}-run-data-quality"
  type          = "CONDITIONAL"
  workflow_name = aws_glue_workflow.etl_workflow.name

  predicate {
    conditions {
      crawler_name = aws_glue_crawler.raw_crawler.name
      crawl_state  = "SUCCEEDED"
    }
  }

  actions {
    job_name = aws_glue_job.data_quality_job.name
  }

  start_on_creation = true
}

# Update ETL trigger to run after data quality
resource "aws_glue_trigger" "run_etl_job_v2" {
  name          = "${var.project_name}-run-etl-job-v2"
  type          = "CONDITIONAL"
  workflow_name = aws_glue_workflow.etl_workflow.name

  predicate {
    conditions {
      job_name = aws_glue_job.data_quality_job.name
      state    = "SUCCEEDED"
    }
  }

  actions {
    job_name = aws_glue_job.etl_job.name
  }

  start_on_creation = true
}

```
The above files: 

- Uploads the data quality rules JSON to S3
- Creates a new Glue job specifically for data quality checks
- Creates CloudWatch log group to store quality check results
- Adds data quality job to the workflow
- NEW WORKFLOW ORDER: Raw Crawler → Data Quality → ETL Job → Processed Crawler

We are creating a separate job because Data quality runs before ETL, so bad data is caught early.

## Step 3: Create Data Quality Job Script

In your scripts folder, start by creating _data_quality_job.py_  and add the following:

```
cd ~/glue-etl-pipeline/scripts
```

```
import sys
import json
import boto3
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job
from awsglue.dataquality.DQRule import DQRule
from awsglue.dataquality.DQRuleset import DQRuleset

# Get job parameters
args = getResolvedOptions(sys.argv, ['JOB_NAME', 'SOURCE_BUCKET', 'RULES_PATH', 'DATABASE_NAME', 'TABLE_NAME'])

# Initialize
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

print(f"Starting Data Quality Check for table: {args['DATABASE_NAME']}.{args['TABLE_NAME']}")

# Read data from Glue Catalog
datasource = glueContext.create_dynamic_frame.from_catalog(
    database=args['DATABASE_NAME'],
    table_name=args['TABLE_NAME']
)

# Convert to DataFrame
df = datasource.toDF()
record_count = df.count()
print(f"Total records to validate: {record_count}")

# Download rules from S3
s3 = boto3.client('s3')
bucket = args['SOURCE_BUCKET']
rules_key = args['RULES_PATH'].replace(f"s3://{bucket}/", "")

rules_response = s3.get_object(Bucket=bucket, Key=rules_key)
rules_content = rules_response['Body'].read().decode('utf-8')
rules_json = json.loads(rules_content)

# Build DQDL (Data Quality Definition Language) ruleset
dqdl_rules = []
for rule in rules_json['Rules']:
    dqdl_rules.append(rule['Rule'])

ruleset_string = " AND ".join(dqdl_rules)

print(f"Loaded {len(rules_json['Rules'])} data quality rules")
print(f"Ruleset: {ruleset_string}")

# Create DQ Ruleset
dq_ruleset = DQRuleset(glueContext, ruleset_string)

# Run data quality evaluation
print("Running data quality evaluation...")
dq_result = dq_ruleset.evaluate(datasource)

# Get results
result_string = dq_result.resultString()
print(f"Data Quality Results:\n{result_string}")

# Check if critical rules failed
passed_rules = 0
failed_rules = 0
critical_failures = 0

for rule in rules_json['Rules']:
    rule_name = rule['Name']
    if rule_name in result_string and "Failed" in result_string:
        failed_rules += 1
        if rule['Severity'] == 'Critical':
            critical_failures += 1
            print(f"❌ CRITICAL FAILURE: {rule_name} - {rule['Description']}")
    else:
        passed_rules += 1
        print(f"✅ PASSED: {rule_name}")

# Write results to CloudWatch
log_group = f"/aws/glue/data-quality/{args['DATABASE_NAME']}"
summary = {
    "timestamp": str(spark.sql("SELECT current_timestamp()").collect()[0][0]),
    "table": f"{args['DATABASE_NAME']}.{args['TABLE_NAME']}",
    "total_records": record_count,
    "total_rules": len(rules_json['Rules']),
    "passed_rules": passed_rules,
    "failed_rules": failed_rules,
    "critical_failures": critical_failures,
    "result": result_string
}

print(f"\nSummary: {json.dumps(summary, indent=2)}")

# Fail job if critical rules failed
if critical_failures > 0:
    error_msg = f"Data Quality Check FAILED: {critical_failures} critical rule(s) violated"
    print(f"\n❌ {error_msg}")
    raise Exception(error_msg)
else:
    print(f"\n✅ Data Quality Check PASSED: All {passed_rules} rules validated successfully")

job.commit()

```














