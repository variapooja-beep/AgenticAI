# 🔍 AWS Bedrock Word Search Puzzle Agent

> **User-System Interaction Flow Document**
> A serverless word search solver powered by Amazon Bedrock, Claude Vision, AWS Lambda, and S3.

---

<div align="center">

![AWS](https://img.shields.io/badge/Amazon_AWS-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)
![Bedrock](https://img.shields.io/badge/Amazon_Bedrock-232F3E?style=for-the-badge&logo=amazonaws&logoColor=FF9900)
![Lambda](https://img.shields.io/badge/AWS_Lambda-FF9900?style=for-the-badge&logo=awslambda&logoColor=white)
![Python](https://img.shields.io/badge/Python_3.12-3776AB?style=for-the-badge&logo=python&logoColor=white)
![S3](https://img.shields.io/badge/Amazon_S3-569A31?style=for-the-badge&logo=amazons3&logoColor=white)

| Field | Value |
|---|---|
| **Document Type** | Interaction Flow + Build Guide |
| **Version** | 1.0 — 2025 |
| **Cloud Platform** | Amazon Web Services (AWS) |
| **Primary Services** | Bedrock · S3 · Lambda · IAM · API Gateway |

</div>

---

## 📑 Table of Contents

1. [Purpose & Scope](#1-purpose--scope)
2. [High-Level Architecture Flow](#2-high-level-architecture-flow)
3. [AWS Services Overview](#3-aws-services-overview)
4. [Step-by-Step Interaction Flow](#4-step-by-step-interaction-flow)
   - [Step 1 — User Uploads Image](#step-1--user-uploads-image)
   - [Step 2 — Amazon S3 Image Storage](#step-2--amazon-s3-image-storage)
   - [Step 3 — Bedrock Agent Orchestration](#step-3--bedrock-agent-orchestration)
   - [Step 4 — Vision Model OCR](#step-4--vision-model-ocr-bedrock-claude)
   - [Step 5 — Lambda Grid Parser](#step-5--lambda-grid-parser-grid-parser)
   - [Step 6 — Lambda Word Solver](#step-6--lambda-word-solver-word-solver)
   - [Step 7 — Formatted Results](#step-7--formatted-results)
5. [IAM Roles & Permissions](#5-iam-roles--permissions)
6. [Bedrock Action Group OpenAPI Schemas](#6-bedrock-action-group-openapi-schemas)
7. [Step-by-Step Build Checklist](#8-step-by-step-build-checklist)

---

## 1. Purpose & Scope

This document provides a complete, step-by-step specification for building the **Word Search Puzzle Agent** on AWS. It covers every AWS service required, exact configuration parameters, IAM permissions, Lambda function logic, and the data contracts between components — giving a developer everything needed to implement the agent from scratch.

The agent:
- Accepts a puzzle image uploaded by the user
- Extracts the letter grid using a Claude vision model via Amazon Bedrock
- Solves the word search in all 8 directions via a Lambda function
- Returns an interactive HTML result with highlighted words

---

## 2. High-Level Architecture Flow

```
  ┌─────────────────────────────────────────────────────────────────┐
  │                    USER (Browser / API Client)                  │
  └──────────────────────────┬──────────────────────────────────────┘
                             │  PUT /upload  (multipart image)
                             ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │              Amazon S3  ·  Bucket: word-search-puzzles           │
  │              Prefix: uploads/{session_id}/{filename}             │
  └──────────────────────────┬───────────────────────────────────────┘
                             │  S3 Pre-signed URL / Event trigger
                             ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │              Amazon Bedrock Agent                                │
  │              Model: anthropic.claude-3-5-sonnet-20241022-v2:0   │
  │              Action Groups: OCR · GridParser · WordSolver        │
  └──────┬───────────────────┬───────────────────────────────────────┘
         │                   │
         │ Invoke            │ Invoke
         ▼                   ▼
  ┌──────────────┐   ┌───────────────────┐
  │ Vision Model │   │ Lambda GridParser  │
  │  (OCR Tool)  │   │ fn: grid-parser    │
  │ Claude 3.5   │   │ Parses JSON grid   │
  └──────┬───────┘   └────────┬──────────┘
         │ JSON grid           │ Structured grid
         └─────────┬──────────┘
                   ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │              Lambda WordSolver                                   │
  │              fn: word-solver                                     │
  │              8-direction search · returns coordinates + HTML     │
  └──────────────────────────┬───────────────────────────────────────┘
                             │  Solution JSON + HTML
                             ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │              Bedrock Agent  (formats final response)            │
  └──────────────────────────┬───────────────────────────────────────┘
                             │  API Gateway response
                             ▼
  ┌──────────────────────────────────────────────────────────────────┐
  │              USER receives interactive HTML solution             │
  └──────────────────────────────────────────────────────────────────┘
```

---

## 3. AWS Services Overview

| # | AWS Service | Resource Name | Purpose | Region |
|---|---|---|---|---|
| 1 | **Amazon S3** | `word-search-puzzles` | Stores uploaded puzzle images | us-east-1 |
| 2 | **Amazon Bedrock Agent** | `word-search-agent` | Orchestrates the full pipeline | us-east-1 |
| 3 | **Bedrock Model (Vision)** | `claude-3-5-sonnet-v2` | OCR — reads letter grid from image | us-east-1 |
| 4 | **AWS Lambda** | `grid-parser` | Parses raw OCR text to 2D JSON grid | us-east-1 |
| 5 | **AWS Lambda** | `word-solver` | Solves 8-direction search, returns HTML | us-east-1 |
| 6 | **Amazon API Gateway** | `word-search-api` | HTTP entry point for image upload + query | us-east-1 |
| 7 | **AWS IAM** | `word-search-agent-role` | Execution role with all permissions | Global |
| 8 | **Amazon CloudWatch** | `word-search-logs` | Logging & monitoring for all Lambdas | us-east-1 |

---

## 4. Step-by-Step Interaction Flow

### Step 1 — User Uploads Image

The user uploads a word search puzzle image via a browser form, mobile app, or direct API call to API Gateway.

| Parameter | Detail |
|---|---|
| **Method** | `HTTP PUT` or `POST multipart/form-data` |
| **Endpoint** | `https://{api-id}.execute-api.us-east-1.amazonaws.com/prod/upload` |
| **Accepted Formats** | `image/jpeg`, `image/png`, `image/webp` (max 5 MB) |
| **Auth** | Amazon Cognito token or API Key header (`x-api-key`) |
| **Response** | `{ "s3_key": "uploads/sess123/puzzle.jpg", "session_id": "sess123" }` |

> 📌 **Note:** The API Gateway Lambda Proxy integration triggers an upload handler that generates a pre-signed S3 PUT URL. The client uploads directly to S3 — never through Lambda — to keep payload sizes within API Gateway limits.

---

### Step 2 — Amazon S3 Image Storage

The image lands in the S3 bucket and triggers downstream processing.

| Configuration | Value |
|---|---|
| **Bucket Name** | `word-search-puzzles` |
| **Bucket Region** | `us-east-1` |
| **Upload Prefix** | `uploads/{session_id}/{timestamp}_{filename}` |
| **Results Prefix** | `results/{session_id}/solution.json` |
| **Versioning** | Enabled |
| **Lifecycle Rule** | Delete `uploads/` objects after 24 hours |
| **Server-Side Encryption** | SSE-S3 (AES-256) |
| **Event Notification** | `s3:ObjectCreated:*` on prefix `uploads/` → SNS or direct Bedrock trigger |
| **CORS** | Allow PUT/GET from your application domain |

**S3 Bucket Policy — allow Bedrock Agent and Lambda to read:**

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowBedrockAndLambda",
    "Effect": "Allow",
    "Principal": {
      "AWS": [
        "arn:aws:iam::ACCOUNT_ID:role/word-search-agent-role",
        "arn:aws:iam::ACCOUNT_ID:role/word-search-lambda-role"
      ]
    },
    "Action": ["s3:GetObject", "s3:PutObject"],
    "Resource": "arn:aws:s3:::word-search-puzzles/*"
  }]
}
```

---

### Step 3 — Bedrock Agent Orchestration

The Bedrock Agent receives the `session_id` and `s3_key`, then orchestrates all downstream steps using its configured Action Groups.

| Agent Configuration | Value |
|---|---|
| **Agent Name** | `word-search-agent` |
| **Foundation Model** | `anthropic.claude-3-5-sonnet-20241022-v2:0` |
| **Agent Alias** | `PROD` |
| **Idle Session TTL** | `600` seconds (10 minutes) |
| **Instruction (System Prompt)** | *You are a word search puzzle solver. When given an S3 image key, retrieve the image, extract the letter grid using the OCR action, parse it using the GridParser action, then solve it using the WordSolver action. Always return the final HTML solution.* |
| **Knowledge Base** | None (stateless pipeline) |
| **Action Groups** | `ocr-action-group`, `grid-parser-action-group`, `word-solver-action-group` |
| **Memory** | DISABLED (session handled via `session_id`) |

**Action Group Definitions:**

| Action Group | Lambda ARN | API Schema | Triggered When |
|---|---|---|---|
| `ocr-action-group` | `arn:...grid-parser` | Inline OpenAPI | Agent needs raw grid text from image |
| `grid-parser-action-group` | `arn:...grid-parser` | Inline OpenAPI | Agent has raw OCR text, needs structured JSON grid |
| `word-solver-action-group` | `arn:...word-solver` | Inline OpenAPI | Agent has grid + words, needs solution + HTML |

---

### Step 4 — Vision Model OCR (Bedrock Claude)

The Bedrock Agent invokes Claude 3.5 Sonnet with the S3 image as a base64 payload. The model reads every letter in the grid and the word list from the puzzle image.

| Vision Call Parameter | Value |
|---|---|
| **Model ID** | `anthropic.claude-3-5-sonnet-20241022-v2:0` |
| **API** | `bedrock-runtime.invoke_model` |
| **Image Source** | S3 → GetObject → base64 encode in Lambda |
| **Max Tokens** | `2048` |
| **Temperature** | `0` (deterministic for OCR accuracy) |
| **System Prompt** | `Return ONLY JSON: {"grid":[[...]],"words":[...]}. No explanation.` |
| **User Message** | `[image attachment] + "Extract the word search grid and word list."` |
| **Output Format** | `{ "grid": [["M","A",...],[...]], "words": ["PORTUGAL",...] }` |

> 📌 **Enable model access** for `anthropic.claude-3-5-sonnet-20241022-v2:0` in the AWS Bedrock console under **Model Access** before invoking. This must be done manually per region.

**Python — invoke vision model from Lambda (`ocr-action-group` handler):**

```python
import boto3, base64, json

bedrock = boto3.client('bedrock-runtime', region_name='us-east-1')
s3      = boto3.client('s3')

def get_grid_from_image(bucket: str, key: str) -> dict:
    # 1. Fetch image from S3
    obj    = s3.get_object(Bucket=bucket, Key=key)
    b64img = base64.b64encode(obj['Body'].read()).decode()

    # 2. Call Claude vision
    body = json.dumps({
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 2048,
        "temperature": 0,
        "messages": [{
            "role": "user",
            "content": [
                {"type": "image", "source": {
                    "type": "base64",
                    "media_type": "image/jpeg",
                    "data": b64img
                }},
                {"type": "text", "text": "Extract the word search grid and word list."}
            ]
        }],
        "system": "Return ONLY JSON: {\"grid\":[...],\"words\":[...]}"
    })

    resp = bedrock.invoke_model(
        modelId='anthropic.claude-3-5-sonnet-20241022-v2:0',
        body=body,
        contentType='application/json',
        accept='application/json'
    )
    text = json.loads(resp['body'].read())['content'][0]['text']
    return json.loads(text)   # { "grid": [[...]], "words": [...] }
```

---

### Step 5 — Lambda Grid Parser (`grid-parser`)

This Lambda validates and normalises the raw OCR JSON output into a clean, uppercase, rectangular 2D grid.

| Lambda Configuration | Value |
|---|---|
| **Function Name** | `grid-parser` |
| **Runtime** | Python 3.12 |
| **Memory** | 256 MB |
| **Timeout** | 30 seconds |
| **Handler** | `handler.lambda_handler` |
| **Trigger** | Bedrock Agent Action Group (`ocr-action-group`) |
| **Environment Variables** | `LOG_LEVEL=INFO` |
| **IAM Role** | `word-search-lambda-role` (`s3:GetObject`, `logs:*`) |
| **Layers** | None (stdlib only) |

**Input / Output Contract:**

| | Input (from Bedrock Agent) | Output (to Bedrock Agent) |
|---|---|---|
| **Format** | `{ "raw_ocr": "{...}" }` | `{ "grid": [["M","A",...],...],"words":["PORTUGAL",...], "rows": 20, "cols": 20 }` |

```python
# Lambda: grid-parser/handler.py
import json, re

def lambda_handler(event, context):
    raw = event.get('raw_ocr', '')

    # Strip markdown fences if model wrapped output
    raw = re.sub(r'```[a-z]*\n?', '', raw).replace('```', '').strip()
    data = json.loads(raw)

    grid  = [[str(c).upper() for c in row] for row in data['grid']]
    words = [str(w).upper().strip() for w in data['words'] if str(w).strip()]

    # Normalise: all rows must be same length
    lengths = [len(r) for r in grid]
    target  = max(set(lengths), key=lengths.count)
    grid    = [r for r in grid if len(r) == target]

    return {
        'statusCode': 200,
        'body': json.dumps({
            'grid':  grid,
            'words': words,
            'rows':  len(grid),
            'cols':  target,
        })
    }
```

---

### Step 6 — Lambda Word Solver (`word-solver`)

This Lambda performs the 8-direction word search and returns both structured coordinates and a self-contained interactive HTML page.

| Lambda Configuration | Value |
|---|---|
| **Function Name** | `word-solver` |
| **Runtime** | Python 3.12 |
| **Memory** | 512 MB |
| **Timeout** | 60 seconds |
| **Handler** | `handler.lambda_handler` |
| **Trigger** | Bedrock Agent Action Group (`word-solver-action-group`) |
| **Environment Variables** | `LOG_LEVEL=INFO`, `OUTPUT_BUCKET=word-search-puzzles` |
| **IAM Role** | `word-search-lambda-role` (`s3:PutObject`, `logs:*`) |
| **Layers** | None (stdlib only) |

```python
# Lambda: word-solver/handler.py
import json, boto3

s3 = boto3.client('s3')

DIRECTIONS = [
    (0,  1),   # right
    (0, -1),   # left
    (1,  0),   # down
    (-1, 0),   # up
    (1,  1),   # diagonal down-right
    (1, -1),   # diagonal down-left
    (-1, 1),   # diagonal up-right
    (-1,-1),   # diagonal up-left
]

def find_word(grid, word):
    rows, cols = len(grid), len(grid[0])
    for r in range(rows):
        for c in range(cols):
            for dr, dc in DIRECTIONS:
                path, ok = [], True
                for i, ltr in enumerate(word):
                    nr, nc = r + dr*i, c + dc*i
                    if 0 <= nr < rows and 0 <= nc < cols and grid[nr][nc] == ltr:
                        path.append([nr, nc])
                    else:
                        ok = False; break
                if ok:
                    return path
    return None

def lambda_handler(event, context):
    body       = json.loads(event.get('body', event))
    grid       = body['grid']
    words      = body['words']
    session_id = body.get('session_id', 'default')

    solution, not_found = {}, []
    for word in words:
        path = find_word(grid, word)
        if path: solution[word] = path
        else:    not_found.append(word)

    result = {
        'found':     list(solution.keys()),
        'not_found': not_found,
        'solution':  solution,
        'stats': {
            'total':     len(words),
            'found':     len(solution),
            'rows':      len(grid),
            'cols':      len(grid[0]),
            'pct_complete': round(len(solution) / len(words) * 100, 1) if words else 0,
        }
    }

    # Persist result to S3
    s3.put_object(
        Bucket='word-search-puzzles',
        Key=f'results/{session_id}/solution.json',
        Body=json.dumps(result),
        ContentType='application/json'
    )

    return {'statusCode': 200, 'body': json.dumps(result)}
```

---

### Step 7 — Formatted Results

The Bedrock Agent assembles the final response from the word-solver output and returns it to the caller via API Gateway.

| Response Field | Description |
|---|---|
| `session_id` | Unique session identifier for this puzzle run |
| `found[]` | Array of words successfully located in the grid |
| `not_found[]` | Array of words not found (may indicate OCR error) |
| `solution{}` | Map of `word → [[row,col],...]` path coordinates |
| `stats{}` | `{ total, found, rows, cols, pct_complete }` |
| `html` | Self-contained interactive HTML page string with highlights |
| `s3_result_key` | S3 key where full solution JSON is stored for retrieval |

**Example Response:**

```json
{
  "session_id": "sess123",
  "found": ["PORTUGAL", "TURKEY", "AUSTRALIA"],
  "not_found": [],
  "solution": {
    "PORTUGAL": [[9,5],[9,6],[9,7],[9,8],[9,9],[9,10],[9,11],[9,12]],
    "TURKEY":   [[12,13],[12,12],[12,11],[12,10],[12,9],[12,8]]
  },
  "stats": {
    "total": 30,
    "found": 30,
    "rows": 20,
    "cols": 20,
    "pct_complete": 100.0
  },
  "s3_result_key": "results/sess123/solution.json"
}
```

---

## 5. IAM Roles & Permissions

### 5.1 Bedrock Agent Execution Role

- **Role name:** `word-search-agent-role`
- **Trusted entity:** `bedrock.amazonaws.com`

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "BedrockModelAccess",
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel",
        "bedrock:InvokeModelWithResponseStream"
      ],
      "Resource": "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-5-sonnet-*"
    },
    {
      "Sid": "S3ReadPuzzles",
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::word-search-puzzles/*"
    },
    {
      "Sid": "InvokeLambdaActions",
      "Effect": "Allow",
      "Action": ["lambda:InvokeFunction"],
      "Resource": [
        "arn:aws:lambda:us-east-1:ACCOUNT_ID:function:grid-parser",
        "arn:aws:lambda:us-east-1:ACCOUNT_ID:function:word-solver"
      ]
    }
  ]
}
```

### 5.2 Lambda Execution Role

- **Role name:** `word-search-lambda-role`
- **Trusted entity:** `lambda.amazonaws.com`

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3Access",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::word-search-puzzles/*"
    },
    {
      "Sid": "BedrockVision",
      "Effect": "Allow",
      "Action": ["bedrock:InvokeModel"],
      "Resource": "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-3-5-sonnet-*"
    },
    {
      "Sid": "CloudWatchLogs",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:us-east-1:ACCOUNT_ID:log-group:/aws/lambda/*"
    }
  ]
}
```

---

## 6. Bedrock Action Group OpenAPI Schemas

Each Bedrock Action Group requires an inline OpenAPI 3.0 schema. Paste these into the Action Group configuration in the AWS Console.

### 6.1 `ocr-action-group`

```yaml
openapi: 3.0.0
info:
  title: OCR Action
  version: 1.0.0
paths:
  /extract-grid:
    post:
      summary: Extract letter grid and word list from a puzzle image in S3
      operationId: extractGrid
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [s3_bucket, s3_key]
              properties:
                s3_bucket:
                  type: string
                  description: S3 bucket name containing the puzzle image
                s3_key:
                  type: string
                  description: S3 object key of the puzzle image
      responses:
        '200':
          description: Extracted grid and word list
          content:
            application/json:
              schema:
                type: object
                properties:
                  grid:
                    type: array
                    items:
                      type: array
                      items:
                        type: string
                  words:
                    type: array
                    items:
                      type: string
```

### 6.2 `word-solver-action-group`

```yaml
openapi: 3.0.0
info:
  title: Word Solver Action
  version: 1.0.0
paths:
  /solve:
    post:
      summary: Find all words in the grid using 8-direction search
      operationId: solveGrid
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [grid, words, session_id]
              properties:
                grid:
                  type: array
                  items:
                    type: array
                    items:
                      type: string
                  description: 2D letter grid (uppercase)
                words:
                  type: array
                  items:
                    type: string
                  description: List of words to find
                session_id:
                  type: string
                  description: Session identifier for result storage
      responses:
        '200':
          description: Solution with coordinates
          content:
            application/json:
              schema:
                type: object
                properties:
                  found:
                    type: array
                    items:
                      type: string
                  not_found:
                    type: array
                    items:
                      type: string
                  solution:
                    type: object
                  stats:
                    type: object
```

## 7. Step-by-Step Build Checklist

Follow these steps **in order** to deploy the agent from scratch:

| ✓ | Task | AWS Console Path | Notes |
|---|---|---|---|
| ☐ | Create S3 bucket `word-search-puzzles` | **S3 → Create Bucket** | Enable versioning + SSE-S3 |
| ☐ | Enable Bedrock model access (Claude 3.5 Sonnet) | **Bedrock → Model Access → Manage** | Takes up to 5 minutes to activate |
| ☐ | Create IAM role `word-search-lambda-role` | **IAM → Roles → Create Role** | Attach `AWSLambdaBasicExecutionRole` + inline policy (§5.2) |
| ☐ | Create IAM role `word-search-agent-role` | **IAM → Roles → Create Role** | Trusted entity: `bedrock.amazonaws.com`, attach inline policy (§5.1) |
| ☐ | Deploy `grid-parser` Lambda | **Lambda → Create Function → Python 3.12** | Paste `handler.py`, set timeout 30 s, memory 256 MB |
| ☐ | Deploy `word-solver` Lambda | **Lambda → Create Function → Python 3.12** | Paste `handler.py`, set timeout 60 s, memory 512 MB, set `OUTPUT_BUCKET` env var |
| ☐ | Create Bedrock Agent | **Bedrock → Agents → Create Agent** | Name: `word-search-agent`, model: `claude-3-5-sonnet-v2`, paste system prompt |
| ☐ | Add `ocr-action-group` | **Agent → Action Groups → Add** | Lambda: `grid-parser`, paste OpenAPI schema from §6.1 |
| ☐ | Add `word-solver-action-group` | **Agent → Action Groups → Add** | Lambda: `word-solver`, paste OpenAPI schema from §6.2 |
| ☐ | Create Agent Alias `PROD` | **Agent → Aliases → Create** | Alias for stable invocation endpoint |
| ☐ | Set up API Gateway *(optional)* | **API Gateway → Create REST API** | `POST /solve` route → Lambda invoke of Bedrock Agent |
| ☐ | Test with sample puzzle image | **Bedrock → Agent → Test** | Upload test image, verify OCR → grid → solution pipeline |
| ☐ | Enable CloudWatch logging | **Lambda → Configuration → Monitoring** | Verify log groups appear for both Lambdas |

---

## 📁 Structure

```
word-search-bedrock-agent/
├── README.md                          ← This document
├── lambdas/
│   ├── grid-parser/
│   │   └── handler.py                 ← Grid normalisation Lambda
│   └── word-solver/
│       └── handler.py                 ← 8-direction solver Lambda
├── iam/
│   ├── agent-role-policy.json         ← Bedrock Agent IAM policy
│   └── lambda-role-policy.json        ← Lambda IAM policy
├── schemas/
│   ├── ocr-action-group.yaml          ← OpenAPI schema for OCR action
│   └── word-solver-action-group.yaml  ← OpenAPI schema for solver action
└── s3/
    └── bucket-policy.json             ← S3 bucket policy
```

*AWS Bedrock Word Search Agent · Interaction Flow Document · Version 1.0 · 2025*
