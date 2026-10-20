
## Review Questions & Answers

### 1. What problem does Amazon SQS solve when the processing application is unavailable?

Amazon SQS decouples the message producer from the consumer.

If AWS Lambda is temporarily unavailable or its trigger is disabled, incoming orders remain stored in the SQS queue.

Once processing resumes, Lambda can consume the waiting messages.

This improves reliability and prevents the immediate loss of incoming orders.

### 2. What is the difference between Visibility Timeout and Message Retention?

**Visibility Timeout** is the period during which a received message is temporarily hidden from other consumers.

If processing fails and the message is not deleted, it can become visible again.

**Message Retention** determines how long SQS stores a message before it expires.

In this project:
- Visibility Timeout: 60 seconds
- Message Retention: 4 days

### 3. Why is the SQS Visibility Timeout set to 60 seconds when the Lambda timeout is 10 seconds?

The visibility timeout should allow enough time for the Lambda function to execute and for message processing to complete.

AWS recommends configuring the visibility timeout to at least six times the Lambda function timeout.

In this project:

10 seconds × 6 = 60 seconds

This provides additional time for invocation delays, retries, and throttling.

### 4. What happens if Lambda returns an error response instead of raising an exception?

If the Lambda function returns a normal response, such as:

```python
return {"error": "Order processing failed"}
```

AWS Lambda considers the invocation successful because no exception was raised.

For a standard SQS event source mapping without partial batch failure reporting, the message is then removed from the queue.

To trigger retries, the function must raise an exception or use the supported partial batch failure response mechanism.

### 5. Why can duplicate messages occur in an Amazon SQS Standard Queue?

Amazon SQS Standard Queues provide at-least-once message delivery.

A message may occasionally be delivered more than once because copies can exist across the distributed SQS infrastructure or because processing is retried.

Applications should use idempotent processing to ensure that duplicate deliveries do not create duplicate business operations.

For example, an application could check the order ID before charging a customer or creating a new order.

### 6. What is the purpose of a Dead Letter Queue, and what happens if a failed message is retried without fixing the error?

A Dead Letter Queue stores messages that repeatedly fail processing.

It allows developers to inspect problematic messages without blocking the processing of other orders.

In this project, messages were moved to `mcit-orders-dlq` after repeated failed processing attempts.

If a message containing `"simulate_failure": true` is sent back for processing without correcting the underlying problem, the Lambda function will fail again.

The error must be fixed before the message can be processed successfully.
