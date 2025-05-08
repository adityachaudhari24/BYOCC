# Security and Compliance Requirements

## Security Requirements

### API Access Control

**Mechanism:** Amazon API Gateway API Keys and Usage Plans

**Flow:**
1. Frontend includes an API Key when making requests to API Gateway
2. API Gateway validates the API Key against configured Usage Plans
3. Requests with valid keys are allowed based on usage plan limits
4. Requests without valid keys are rejected by API Gateway

### Admin Endpoint Authorization

**Mechanism:** Additional security for secured API endpoints like PUT `/inventory/{productId}`

**Flow:**
1. Admin frontend sends request to API Gateway with an API Key and identity token/header
2. API Gateway passes the request to the Lambda function
3. The Lambda extracts identity information
4. The Lambda checks if the identity is authorized to perform updates
5. If authorized, the Lambda proceeds with the DynamoDB update. If not, it returns an authorization error

### Backend Resource Protection

**Mechanism:** AWS IAM roles and policies

**Flow:**
1. Lambda functions execute under specific IAM roles
2. These roles have IAM policies attached that grant only the necessary permissions
3. This prevents unauthorized access to DynamoDB, SNS, etc., directly from outside the trusted Lambda functions

### Frontend Hosting Security

**Mechanism:** S3 Bucket Policies and potentially CloudFront

**Flow:**
- The S3 bucket hosting the static frontend files has a bucket policy allowing `s3:GetObject` for public anonymous users
- Access is restricted to GET operations for public access

## Compliance Requirements
TODO