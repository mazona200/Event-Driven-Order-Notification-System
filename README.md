# Event-Driven-Order-Notification-System
Building an Event-Driven Order Notification System Using AWS

## Overview
This backend project simulates an e-commerce platform that processes orders using AWS services in an event-driven architecture.

## Architecture Diagram
![Editor | Mermaid Chart-2025-05-10-164731](https://github.com/user-attachments/assets/a2441a07-b5c9-40ed-83b6-2acd63696862)

## AWS Services Used
- **DynamoDB**: Stores order data
- **SNS**: Broadcasts order events
- **SQS**: Queues messages between services
- **Lambda**: Processes queued messages and writes to DynamoDB
- **CloudWatch**: Logs function executions
- **DLQ (Dead Letter Queue)**: Captures failed messages

## Setup Instructions

### 1. DynamoDB Table
- Table name: `Orders`
- Partition key: `orderId` (String)

### 2. SNS Topic
- Name: `OrderTopic`
- Create a subscription with protocol: **Amazon SQS**
- Endpoint: Your SQS queue ARN

### 3. SQS Queue
- Name: `OrderQueue`
- Attach a DLQ named `OrderDLQ` with maxReceiveCount = 3

### 4. Lambda Function
- Runtime: Node.js 18.x
- Trigger: SQS (`OrderQueue`)
- Permissions: Attach `AmazonDynamoDBFullAccess` and `AmazonSQSFullAccess` to Lambda role

### 5. Test
- Publish a test message to SNS in this format:
`{
  "orderId": "O1234",
  "userId": "U123",
  "itemName": "Laptop",
  "quantity": 1,
  "status": "new",
  "timestamp": "2025-05-03T12:00:00Z"
}`


---

## 2. Lambda Function 
`const { DynamoDBClient } = require("@aws-sdk/client-dynamodb");
const { DynamoDBDocumentClient, PutCommand } = require("@aws-sdk/lib-dynamodb");
const client = new DynamoDBClient();
const docClient = DynamoDBDocumentClient.from(client);
exports.handler = async (event) => {
  for (const record of event.Records) {
    try {
      const snsWrapped = JSON.parse(record.body);
      const order = JSON.parse(snsWrapped.Message);
      const command = new PutCommand({
        TableName: "Orders",
        Item: order,
      });
      await docClient.send(command);
      console.log("Order saved:", order.orderId);
    } catch (err) {
      console.error("Error saving order:", err);
    }
  }
  return { statusCode: 200, body: "Done" };
};
`
3. Screenshots 
SNS Subscription:
<img width="546" alt="Screenshot 2025-05-10 at 5 54 35 PM" src="https://github.com/user-attachments/assets/e80dfb0c-e326-41d3-b031-4db81187c49d" />

SQS Message Metrics
<img width="1512" alt="Screenshot 2025-05-10 at 7 31 08 PM" src="https://github.com/user-attachments/assets/91bb94ac-cf5b-49e2-9821-b71b69958502" />
<img width="1511" alt="Screenshot 2025-05-10 at 7 31 56 PM" src="https://github.com/user-attachments/assets/c4038dd6-63ba-463d-9157-d6405a96c8c0" />

Dynamo DB Order table:
<img width="1512" alt="Screenshot 2025-05-10 at 7 33 11 PM" src="https://github.com/user-attachments/assets/08556dd5-6f67-4268-a84c-54cc745b3069" />

## How Visibility Timeout and DLQ Help Reliability?

**Visibility Timeout** is a period during which a message received by a consumer (Lambda) is hidden from other consumers. This prevents the same message from being processed multiple times. If processing fails or takes too long, the message reappears after the timeout, allowing retries.

**Dead Letter Queue (DLQ)** acts as a backup system. If a message fails to be processed after the allowed number of retries (`maxReceiveCount = 3`), it is sent to the DLQ instead of getting lost. This allows developers to inspect failed messages, troubleshoot errors, and manually retry if needed.

Together, they ensure **fault tolerance**, **reliable retries**, and **error traceability** in serverless systems.
