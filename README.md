
# AWS Serverless Order Processing with SQS and Lambda

## Project Overview

This project demonstrates a serverless, event-driven order processing system built using Amazon Web Services (AWS).

Amazon Simple Queue Service (SQS) receives and stores customer orders, while AWS Lambda automatically processes incoming messages using Python.

The project also implements automatic retries, error handling, a Dead Letter Queue (DLQ), and Amazon CloudWatch logging.

The goal was to understand how AWS services communicate and how asynchronous architectures improve reliability and fault tolerance.

## AWS Services and Technologies

- **Amazon SQS** — Asynchronous message queuing
- **AWS Lambda** — Serverless order processing
- **Amazon CloudWatch** — Monitoring and logging
- **AWS IAM** — Roles and permissions
- **Python 3.12** — Lambda function development
- **JSON** — Message formatting
- **AWS Management Console** — Infrastructure configuration

## System Architecture

```text
Customer Order (JSON)
        |
        v
  Amazon SQS
  (mcit-orders)
        |
        v
   AWS Lambda
(mcit-process-order)
        |
        v
  Order Processing
        |
        v
 Amazon CloudWatch
   (Logs & Errors)

On Processing Failure:
        |
        v
   Automatic Retry
        |
        v
  Dead Letter Queue
 (mcit-orders-dlq)
```

SQS acts as a buffer between incoming orders and the processing function.

When the Lambda trigger is temporarily disabled, messages remain stored in SQS until processing resumes.

Messages that repeatedly fail processing are moved to a Dead Letter Queue for investigation.

## 1. Amazon SQS Configuration

Created a standard SQS queue to receive incoming orders.

**Queue name:** `mcit-orders`

Configuration:

- Queue type: Standard
- Visibility timeout: 60 seconds
- Message retention: 4 days
- Delivery delay: 0 seconds
- Encryption: SSE-SQS
- Access: Private

A standard SQS queue supports asynchronous processing and provides at-least-once message delivery.

## 2. AWS Lambda Configuration

Created a serverless Python function to process incoming SQS messages.

**Function name:** `mcit-process-order`

Configuration:

- Runtime: Python 3.12
- Memory: 128 MB
- Timeout: 10 seconds
- AWS Region: us-east-1 (N. Virginia)

### IAM Permissions

The Lambda execution role was configured with:

- `AWSLambdaBasicExecutionRole`
- `AWSLambdaSQSQueueExecutionRole`

These policies allow the function to write logs to CloudWatch and consume messages from Amazon SQS.

## 3. Lambda Function — Python

The function reads incoming SQS messages, extracts order information, and logs successful processing.

It can also simulate an error to demonstrate retry behavior and Dead Letter Queue handling.

```python
import json
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    for record in event["Records"]:
        message_id = record["messageId"]
        attempts = record["attributes"].get(
            "ApproximateReceiveCount", "unknown"
        )

        logger.info(
            "Received %s attempt %s",
            message_id,
            attempts
        )

        order = json.loads(record["body"])

        if order.get("simulate_failure", False):
            raise ValueError(
                "Intentional MCIT lab failure"
            )

        logger.info(
            "Processed order %s | customer=%s | item=%s",
            order["order_id"],
            order["customer"],
            order["item"]
        )

    return {"status": "done"}
```

### How the Code Works

1. Lambda receives an event containing SQS messages.
2. The function loops through the received records.
3. It extracts the message ID and receive count.
4. It converts the JSON message body into a Python dictionary.
5. If `simulate_failure` is true, the function raises an exception.
6. Otherwise, it logs the successfully processed order.
7. CloudWatch stores the execution logs.

## 4. Connecting Amazon SQS to AWS Lambda

Configured an SQS event source mapping to automatically invoke Lambda when messages become available.

Trigger settings:

- Source: `mcit-orders`
- Trigger status: Enabled
- Batch size: 1
- Batch window: 0 seconds
- Partial batch response: Disabled

Using a batch size of 1 allows each order to be processed individually.

## 5. Successful Order Processing

Sent a JSON order to the main SQS queue.

### Example Message

```json
{
  "order_id": "ORD-002",
  "customer": "Sam",
  "item": "Laptop"
}
```

After the message was submitted:

1. Amazon SQS received the order.
2. The SQS trigger invoked AWS Lambda.
3. Lambda extracted the order information.
4. The function processed the message successfully.
5. Amazon CloudWatch recorded the result.

### CloudWatch Output

```text
Processed order ORD-002 | customer=Sam | item=Laptop
```

This confirmed that the SQS-to-Lambda integration was functioning correctly.

## 6. Testing Message Backlog

To understand asynchronous processing, the SQS trigger was temporarily disabled.

Two additional orders were submitted:

### Order 1

```json
{
  "order_id": "ORD-003",
  "customer": "Amina",
  "item": "Mouse"
}
```

### Order 2

```json
{
  "order_id": "ORD-004",
  "customer": "Karim",
  "item": "Keyboard"
}
```

### Observations

- The Lambda trigger was disabled.
- SQS retained both messages.
- The queue displayed 2 available messages.
- No messages were processed while the trigger was disabled.
- After re-enabling the trigger, Lambda processed both orders.

This test demonstrated how SQS decouples message producers from consumers.

Even when processing is temporarily unavailable, incoming work is not immediately lost.

## 7. Dead Letter Queue Configuration

Created a separate SQS queue to store messages that repeatedly fail processing.

**DLQ name:** `mcit-orders-dlq`

Configuration:

- Queue type: Standard
- Message retention: 14 days
- Encryption: SSE-SQS
- Maximum receives: 5

The DLQ was connected to the main SQS queue using a redrive policy.

Messages that repeatedly fail processing are moved to the DLQ instead of being retried indefinitely.

## 8. Testing Automatic Retries and Error Handling

Sent an intentionally failing order:

```json
{
  "order_id": "ORD-FAIL-001",
  "customer": "Sam",
  "item": "Laptop",
  "simulate_failure": true
}
```

The Lambda function detected `simulate_failure: true` and raised an exception.

### Expected Error

```text
ValueError: Intentional MCIT lab failure
```

### Observations

- Lambda failed to process the message.
- CloudWatch recorded the exception.
- SQS made the message available for additional processing attempts.
- The receive count increased across retries.
- The message was eventually moved to the Dead Letter Queue.
- The failed message was available in the DLQ for investigation.

This demonstrated automatic retry behavior and failure isolation using Amazon SQS and AWS Lambda.

## 9. Processing a Corrected Order

A corrected version of the failed order was sent to the main SQS queue.

```json
{
  "order_id": "ORD-FAIL-001",
  "customer": "Sam",
  "item": "Laptop",
  "simulate_failure": false
}
```

After removing the simulated failure condition, Lambda successfully processed the new message.

### CloudWatch Output

```text
Processed order ORD-FAIL-001 | customer=Sam | item=Laptop
```

The original failed message remained in the DLQ until explicitly removed or expired.

This test demonstrated how errors can be investigated while normal processing continues.

## 10. Key Concepts Learned

### Asynchronous Processing

SQS allows applications to submit work without waiting for immediate processing.

### Decoupling

The message producer and consumer operate independently.

A temporary failure in the processing component does not necessarily interrupt message submission.

### Visibility Timeout

When a consumer receives a message, SQS temporarily hides it from other consumers while processing is underway.

If processing fails, the message can become available again.

### Message Retention

SQS stores messages for a configured period before they expire.

### Automatic Retries

Failed messages can be delivered again for another processing attempt.

### At-Least-Once Delivery

Standard SQS queues may occasionally deliver a message more than once.

Production applications should implement idempotent processing to avoid unintended duplicate operations.

### Dead Letter Queues

A DLQ isolates repeatedly failing messages for troubleshooting without blocking other orders.

### Serverless Architecture

AWS Lambda executes code without requiring the management of dedicated application servers.

## 11. Project Results

Successfully implemented and tested:

- [x] Standard Amazon SQS queue
- [x] Python-based AWS Lambda function
- [x] IAM execution role permissions
- [x] SQS event source mapping
- [x] Successful order processing
- [x] CloudWatch logging
- [x] Message backlog during trigger downtime
- [x] Automatic processing after trigger reactivation
- [x] Dead Letter Queue configuration
- [x] Automatic retries for failed messages
- [x] Intentional error simulation
- [x] Successful processing of a corrected order

## 12. Screenshots

The following screenshots document the AWS configuration and testing process.

<img width="2559" height="1309" alt="Screenshot 2026-10-20 174442" src="https://github.com/user-attachments/assets/dd73f55b-5a1d-4f85-b6c6-211ed0101e10" />
<img width="2559" height="1304" alt="Screenshot 2026-10-20 175358" src="https://github.com/user-attachments/assets/71fd7075-d276-45ef-b766-d0a4b174edcc" />
<img width="2559" height="1349" alt="Screenshot 2026-10-20 175630" src="https://github.com/user-attachments/assets/b52d80d0-d0e8-47b2-9431-c6f92bc1c398" />
<img width="2559" height="1310" alt="Screenshot 2026-10-20 180025" src="https://github.com/user-attachments/assets/4ff43ba1-be35-41c9-8893-1f296e604c2d" />
<img width="2558" height="1309" alt="Screenshot 2026-10-20 180430" src="https://github.com/user-attachments/assets/d5159959-f54d-4111-9c4e-4546a54b3aa7" />
<img width="2559" height="1306" alt="Screenshot 2026-10-20 182217" src="https://github.com/user-attachments/assets/01f0666c-5ffe-48fd-b967-6a2004d22842" />
<img width="2559" height="1308" alt="Screenshot 2026-10-20 182930" src="https://github.com/user-attachments/assets/809895ec-20ae-4737-8baf-580c4066cc77" />
<img width="2558" height="1308" alt="Screenshot 2026-10-20 184301" src="https://github.com/user-attachments/assets/89ad08ff-ff52-48ee-bf3c-a93b4fab4cb4" />
<img width="2557" height="1309" alt="Screenshot 2026-10-20 184617" src="https://github.com/user-attachments/assets/24ad1edd-ea2f-40b5-9abd-ef6fb6bceebc" />
<img width="2557" height="1296" alt="Screenshot 2026-10-20 185244" src="https://github.com/user-attachments/assets/4f797305-ad8b-4b0d-83e7-41dbb8fbb7cf" />
<img width="1694" height="368" alt="Screenshot 2026-10-20 185709" src="https://github.com/user-attachments/assets/796ed0ab-d5f4-4a74-92bb-4abac92cdb88" />
<img width="2543" height="731" alt="Screenshot 2026-10-20 190017" src="https://github.com/user-attachments/assets/04bbbee3-0b68-4dae-b723-fe20d2236afa" />



