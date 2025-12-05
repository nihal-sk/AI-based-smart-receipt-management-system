# Project ReciSight - AI-based Smart Receipt Management System
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![AWS](https://img.shields.io/badge/Platform-AWS-orange.svg)](https://aws.amazon.com/)
[![Python](https://img.shields.io/badge/Python-3.12-blue.svg)](#)

A cloud-native pipeline that ingests receipts (images / PDFs), extracts structured data via **Amazon Textract**, stores normalized receipt data in **DynamoDB**, and exposes a lightweight API + static UI to view and search receipts — similar to a personal expense wallet.


Table of Contents
- Overview
- Key Features
- Tech stack
- Architecture
- Quick Demo
- Repo Layout
- Local Development
- Deployment Guide
- API Reference
- DynamoDB Schema
- Lambda function responsibilities
- Minimal IAM Policy
- Future Enhancements
- Troubleshooting
- Contributing
- License

Overview
--------
ReciSight accepts receipt images or PDFs, runs OCR/field-extraction (Textract AnalyzeExpense), normalizes extracted fields (vendor/date/total/line-items), and stores the data in DynamoDB table. A simple API (API Gateway + Lambda) allows fetching receipts, and a static frontend renders them. The system is intentionally fully serverless, zero-maintenance, and AWS Free-Tier friendly.

Key Features
------------
- Ingest receipts from S3 (images or PDFs)
- Automatic field + line-item extraction using Amazon Textract AnalyzeExpense
- Normalized storage in a DynamoDB `receipts` table
- Read API (API Gateway + Lambda) to list and retrieve receipts
- Lightweight static frontend (S3/CloudFront) to browse receipts
- Extensible: add categories, tagging, analytics, export

🛠 Tech Stack
-----------------
- **Compute**: AWS Lambda (Python 3.12)
- **Storage**: S3
- **OCR / NLP**: Amazon Textract (AnalyzeExpense)
- **Database**: DynamoDB (`receipts` table)
- **API**: API Gateway HTTP API (`GET /receipts`)
- **Frontend**: Vanilla HTML/CSS/JS, hosted on S3 static website
- **Language**: Python, JavaScript

Architecture
------------
High-level flow:
1. User uploads receipt image/PDF to S3.
2. S3 triggers `ingest-receipt-lambda`.
3. Lambda calls Amazon Textract AnalyzeExpense and extracts/normalizes the data.
4. Normalized receipt stored in DynamoDB table `receipts`.
5. `list-receipts-lambda` (behind API Gateway HTTP API) serves GET requests for listing/fetching receipts.
6. Static frontend hosted on S3 (or CloudFront) calls the API and renders results.

(See `docs/architecture-diagram.png` in the repo for a diagram.)

Quick Demo
----------
- Upload a receipt image to the configured S3 bucket (or use the provided UI).
- The ingest Lambda will process the file and write a receipt item into DynamoDB.
- Visit the frontend (S3 static site) and load receipts for the demo `user_id`.

Repo Layout
----------
```
ReciSight/
├── backend/
│   ├── ingest-receipt-lambda/
│   │   └── lambda_function.py      # S3 → Textract → DynamoDB pipeline
│   └── list-receipts-lambda/
│       └── lambda_function.py      # GET /receipts API
├── docs/
│   └── architecture-diagram.png
|   └── UI-Screenshot.jpeg 
├── frontend
│   └── index.html                  # Static UI (S3/CloudFront)
├── LICENSE
├── README.md
└── .gitignore     

```

Local Development
-----------------
1. Clone the repo:
   git clone https://github.com/nihal-sk/AI-based-smart-receipt-management-system.git
   ~ cd AI-based-smart-receipt-management-system

2. Backend (Lambda functions)
   - Create a Python venv:
     / python3.12 -m venv .venv
     / source .venv/bin/activate
     / pip install -r backend/requirements.txt  # create this file with exact deps if missing

   - Unit test your Lambda handlers locally by invoking the function file or using pytest.

3. Frontend
   - Open `frontend/index.html` in your browser (or run a local static server):
     python -m http.server 8000/frontend/index.html
   - Update `API_BASE_URL` in the JS to point at your API Gateway endpoint for local testing.

Deployment Guide
---------------------------
Minimal manual setup:

1. S3 bucket → receipt uploads
2. S3 bucket → static frontend hosting
3. DynamoDB table: receipts
   - PK: receipt_id
   - GSI: user_id-index
4. Lambda functions:
   - ingest-receipt (triggered by S3 PUT)
   - list-receipts (invoked by API Gateway)

5. API Gateway HTTP API
   - GET /receipts
   - GET /receipts/{id}

6. IAM roles with least-privilege policies (below).

API Reference
-------------
GET /receipts
Parameters:
   - user_id (required)
   - limit (optional)
   - last_evaluated_key (optional)

Returns list of receipts for a user.

GET /receipts/{receipt_id}
Returns one receipt object with all parsed fields and line items.

DynamoDB Schema
-----------------------------
| Field        | Type              | Description           |
| ------------ | ----------------- | --------------------- |
| receipt_id   | String            | UUID                  |
| user_id      | String            | Owner of receipt      |
| vendor       | String            | Extracted vendor      |
| date         | String (ISO 8601) | Purchase date         |
| total        | Number            | Total amount          |
| currency     | String            | Optional              |
| line_items   | List<Map>         | Extracted items       |
| s3_bucket    | String            | Where file stored     |
| s3_key       | String            | Object path           |
| raw_textract | Map               | Optional raw response |
| created_at   | String ISO8601    | Timestamp             |

Lambda Functions: responsibilities
----------------------------------
- ingest-receipt-lambda/lambda_function.py
  - Trigger: S3 ObjectCreated (PUT)
  - Steps:
    - Validates file type
    - Calls Textract AnalyzeExpense 
    - Parses vendor/date/total/line_items from Textract response
    - Normalizes and write items to DynamoDB
    - Optionally persists raw Textract JSON to S3 or raw_textract field

- list-receipts-lambda/lambda_function.py
  - Trigger: API Gateway HTTP endpoint
  - Steps:
    - Validates query params 
    - Queries DynamoDB GSI for user receipts with pagination
    - Returns JSON response

Minimal IAM Policy (principle of least privilege)
-------------------------------------------------
Attach a role with the following minimal permissions to the ingest Lambda:

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "textract:AnalyzeExpense",
        "textract:AnalyzeDocument",
        "textract:GetDocumentAnalysis"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:PutItem",
        "dynamodb:UpdateItem",
        "dynamodb:Query",
        "dynamodb:GetItem"
      ],
      "Resource": "arn:aws:dynamodb:REGION:ACCOUNT_ID:table/receipts"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::YOUR_RECEIPT_BUCKET/*"
    }
  ]
}

Future Enhancements
-------
Frontend (UI/UX):
  - Drag-and-drop upload
  - Receipt thumbnails + confidence badges
  - Filters (date range, vendor, category)
  - Editable parsed fields
  - CSV/PDF export

Backend:
  - Auto-categorization using ML
  - Multi-user auth (Cognito)
  - Daily/monthly expense analytics

Troubleshooting / Tips
----------------------
- If Textract responses are missing line items, enable AnalyzeExpense and check the service limits/permissions.
- Use CloudWatch Logs for Lambdas. Add structured logs (JSON) for easier search.
- Add DLQ (SQS) for failed Lambda invocations and a scheduled retry mechanism.

Contributing
------------
Contributions welcome.
- Fork the repo, create feature branches, open PRs to main.
- Please follow these guidelines:
  - Run tests locally
  - Keep changes small and well-documented
  - Include screenshots for UI/front-end changes

License
-------
MIT License — see LICENSE in the repo.
