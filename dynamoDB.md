# Bug Report Flow Table

The bug-report flow collects the required fields first, then invokes the backend tool/Lambda, which is designed to persist the structured record to DynamoDB.

I configured a DynamoDB table for bug reports and updated the Lambda logic, environment variables, and IAM permissions to support `PutItem` operations. During final validation, the Lambda still returned a DynamoDB key-schema validation error related to the `ticket_id` field, so end-to-end persistence could not be fully confirmed before submission.

Even with that limitation, the implemented flow, tool wiring, and backend logic reflect the intended architecture: agent/flow → tool invocation → Lambda → DynamoDB.
