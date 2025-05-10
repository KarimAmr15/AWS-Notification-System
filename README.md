# AWS Event-Driven Order Notification System

## Overview

This project implements a simplified event-driven backend for an e-commerce platform using AWS services. The system accepts orders via SNS, queues them in SQS, processes them with Lambda, stores them in DynamoDB, and ensures reliability with a Dead-Letter Queue (DLQ).

---

## ✅ Architecture Diagram

![Architecture Diagram](./architecture.png)

---

## 📦 AWS Services Used

- **Amazon SNS** – to broadcast new order notifications.
- **Amazon SQS** – to queue messages for Lambda.
- **AWS Lambda** – to process messages and insert orders into DynamoDB.
- **Amazon DynamoDB** – to persist order data.
- **CloudWatch** – to log and monitor Lambda executions.

---

## ⚙️ Setup Instructions

### 1. DynamoDB Table – `Orders`

- Go to **DynamoDB > Create table**
- Table Name: `Orders`
- Partition Key: `orderId` (String)  
  *(Important: This must be exactly `orderId`, case-sensitive.)*

---

### 2. SNS Topic – `OrderTopic`

- Go to **SNS > Topics > Create topic**
- Name: `OrderTopic`
- Type: Standard
- After creation, note the ARN for use in SQS subscription.

---

### 3. SQS Queues

#### A. Create Main Queue – `OrderQueue`

- Go to **SQS > Create queue**
- Name: `OrderQueue`
- Type: Standard
- Under **Dead-letter queue**:
  - Enable DLQ
  - Choose: `OrderDLQ` (create first if needed)
  - Set **MaxReceiveCount**: `3`

#### B. Create Dead-Letter Queue – `OrderDLQ`

- Name: `OrderDLQ`
- Type: Standard
- Leave DLQ and redrive policies disabled (this is already a DLQ).

---

### 4. SNS → SQS Subscription

- Go to **SNS > OrderTopic > Create subscription**
- Protocol: **Amazon SQS**
- Endpoint: Select `OrderQueue`
- Make sure the subscription is **confirmed**

---

### 5. Lambda Function – `OrderProcessor`

#### A. Create the IAM Role

- Go to **IAM > Roles > Create role**
- Use case: **Lambda**
- Attach these policies:
  - `AmazonDynamoDBFullAccess`
  - `AmazonSQSFullAccess`
  - `AWSLambdaBasicExecutionRole`
- Name: `LambdaOrderProcessorRole`

#### B. Create the Lambda Function

- Go to **Lambda > Create Function**
- Runtime: **Python 3.10**
- Function name: `OrderProcessor`
- Use the IAM role above

#### C. Add SQS Trigger

- Go to **Configuration > Triggers > Add trigger**
- Choose SQS
- Select `OrderQueue`

#### D. Lambda Code

```python
import json
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Orders')

def lambda_handler(event, context):
    for record in event['Records']:
        try:
            print("Raw SQS record body:", record['body'])

            message = json.loads(record['body'])
            if isinstance(message, str):
                message = json.loads(message)

            print("Parsed message:", message)

            # Save to DynamoDB
            table.put_item(Item=message)
            print("Order saved:", message['orderId'])

        except Exception as e:
            print("Error processing message:", e)
            raise e
