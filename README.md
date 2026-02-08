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
