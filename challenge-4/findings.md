# Challenge 4 — Findings

## Root cause

The AWS DevOps Agent investigated the Lambda function `challenge4-app-fn` and determined that the issue was caused by missing DynamoDB permissions in the Lambda execution role.

Although the application code, runtime configuration, environment variables, and DynamoDB table were correctly configured, the IAM role lacked the required permissions to read data from the `challenge4-data` table. This resulted in an `AccessDeniedException` whenever the Lambda attempted to perform a `dynamodb:GetItem` operation.

## Fix applied

I updated the Lambda execution role by adding the required DynamoDB permissions:

* `dynamodb:GetItem`
* `dynamodb:Query`
* `dynamodb:Scan`

for the `challenge4-data` table.

After applying the IAM policy changes, the AWS DevOps Agent confirmed that the Lambda function, DynamoDB table, and environment configuration were all correctly configured and healthy.

## Evidence

* [x] Screenshot 1: DevOps Agent diagnosis showing the missing DynamoDB permissions (`screenshots/ss1.png`)
* [x] Screenshot 2: DevOps Agent confirmation after the IAM fix was applied (`screenshots/ss2.png`)
