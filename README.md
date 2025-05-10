# AWS Event-Driven Order Notification System

## Overview

This project implements a simplified event-driven backend for an e-commerce platform using AWS services. The system accepts orders via SNS, queues them in SQS, processes them with Lambda, stores them in DynamoDB, and ensures reliability with a Dead-Letter Queue (DLQ).

---

## ✅ Architecture Diagram

![Architecture Diagram](images/architecture.png)

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

![DynamoDB image](images/create_dynamo.png)

---

### 2. SNS Topic – `OrderTopic`

- Go to **SNS > Topics > Create topic**
- Name: `OrderTopic`
- Type: Standard
- After creation, note the ARN for use in SQS subscription.

![SNS image](images/create_sns.png)

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


![SQS image](images/create_sqs1.png)
![SQS image](images/create_sqs2.png)

---

### 4. SNS → SQS Subscription

- Go to **SNS > OrderTopic > Create subscription**
- Protocol: **Amazon SQS**
- Endpoint: Select `OrderQueue`

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
- Runtime: **Node js**
- Function name: `OrderProcessor`
- Use the IAM role above

#### C. Add SQS Trigger

- Go to **Configuration > Triggers > Add trigger**
- Choose SQS
- Select `OrderQueue`


![lambda image](images/create_lambda.png)

#### D. Lambda Code

```import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import { DynamoDBDocumentClient, PutCommand } from "@aws-sdk/lib-dynamodb";

// Initialize DynamoDB clients
const client = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(client);

// DynamoDB table name
const tableName = "Orders";

export const handler = async (event) => {
  for (const record of event.Records) {
    try {
      // Parse the SNS-wrapped message
      const snsWrapped = JSON.parse(record.body);
      const message = JSON.parse(snsWrapped.Message);

      console.log("Received order message:", message);

      // Save to DynamoDB
      const command = new PutCommand({
        TableName: tableName,
        Item: message,
      });

      const response = await docClient.send(command);
      console.log("DynamoDB response:", response);

    } catch (error) {
      console.error("Error processing message:", error);
      throw error;
    }
  }
};
---




### 4. Test case






