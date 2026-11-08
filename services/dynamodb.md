# AWS DynamoDB

Secondary datastore for the AI Mock Interview BE: report analysis + user metadata.

## Usage (BE only)

- Client: `database/dynamodb_config.py` + boto3.
- Tables (names from env):
  - `DYNAMODB_INTERVIEW_REPORT_ANALYSIS_TABLE` — report analysis records.
  - `DYNAMODB_USER_METADATA_TABLE` — user metadata.

## Env vars (names only)

Shares AWS creds with S3:
- `AWS_ACCESS_KEY_ID_1` 🔑 / `AWS_SECRET_ACCESS_KEY_1` 🔑 / `AWS_DEFAULT_REGION`
- `DYNAMODB_INTERVIEW_REPORT_ANALYSIS_TABLE`
- `DYNAMODB_USER_METADATA_TABLE`

## Open questions

- Table names per environment.
- Capacity / billing mode.
- GSIs and query patterns (fill in as learned).
