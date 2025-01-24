# High-Level Design (HLD) for Bulk Operations System

## Overview

This document outlines the high-level design for the bulk operations system. The system supports bulk create, update, and delete operations using a server, RabbitMQ (or AWS SQS) for queuing, Redis for tracking operation statuses, PostgreSQL for storage, and optional serverless options using AWS API Gateway with Lambda.

---

## Architecture

### Components

1. **Node.js Server (or AWS API Gateway with Lambda)**:

   - Accepts bulk requests from clients.

   - Splits bulk data into manageable batches and publishes tasks to RabbitMQ or AWS SQS.

   - Provides API endpoints to track operation statuses and results.

   - Supports WebSocket connections for real-time tracking updates.

   - AWS API Gateway with Lambda (Optional):

     - Replaces the traditional server with a fully serverless alternative.

     - Handles client requests and triggers Lambda functions directly for processing tasks.

     - Integrates with AWS SQS and Redis for task distribution and tracking

   **RabbitMQ (or AWS SQS)**:

   - Acts as a message broker, decoupling the request processing from task execution.
   - RabbitMQ:
     - Ensures reliable task delivery using durable queues and acknowledgments.
   - AWS SQS (Optional):
     - Provides a fully managed, serverless queueing service.
     - Automatically scales with workload demands and integrates natively with AWS Lambda.

2. **Redis**:

   - Tracks real-time progress of operations, storing data such as the total number of batches, processed batches, and errors.
   - Provides fast read/write access for tracking information.

3. **Worker Nodes** (or **AWS Lambda**):

   - **Worker Nodes**:
     - Consume tasks from RabbitMQ or AWS SQS.
     - Perform batch processing (e.g., inserts, updates, deletes) on PostgreSQL.
     - Update Redis with progress and error details.
   - **AWS Lambda** (Optional Alternative):
     - Processes tasks directly from AWS SQS.
     - Offers serverless scalability and automatic resource provisioning.
     - Updates Redis and PostgreSQL in the same way as Worker Nodes.

4. **PostgreSQL**:

   - Stores the primary data (entities) and supports efficient batch operations.
   - Optimized for bulk inserts, updates, and deletes using indexing and query tuning.

---

### Workflow

#### 1. Request Handling

1. The **client** sends a bulk request (e.g., `/bulk-create`).
2. The **server** (or AWS API Gateway) validates the request and splits it into batches (e.g., 100 items per batch).
3. The server publishes each batch as a task to RabbitMQ or AWS SQS.
4. Redis is initialized to track the operation's status.

#### 2. Task Processing (Three Options)

1. **Worker Nodes**:

   - Worker nodes consume tasks from RabbitMQ or AWS SQS.
   - Workers process each batch (e.g., inserting rows into PostgreSQL).
   - Workers update Redis with progress (e.g., incrementing `processed_batches`).
   - Errors are logged in Redis, and the `failed_batches` count is updated.

2. **AWS Lambda**:

   - AWS SQS triggers Lambda functions for each batch.
   - Lambdas perform batch processing on PostgreSQL and update Redis with progress and errors.
   - Lambda scales automatically to handle increased workloads.

#### 3. Status Tracking

1. Clients query the `/status/:operationId` endpoint to get progress updates.
2. Clients query the `/result/:operationId` endpoint to fetch final results, including any errors.
3. Clients can also subscribe to real-time updates via WebSocket connections for seamless communication.
   - **Advantages of WebSocket Connection**:
     - Real-time updates: Clients receive instant notifications about progress or results.
     - Persistent connection: Avoids repetitive polling, reducing server load.
     - Bi-directional communication: Allows the client to send control messages, such as pausing or canceling operations.



---

## Architecture Diagram

```plaintext
           +-------------------+
           |    Client         |
           +-------------------+
                    |\
                    | \
                    |  +--------------------+
                    |  | WebSocket for      |
                    |  | Real-Time Updates  |
                    |  +--------------------+
                    |
                    v
          +--------------------+      +--------------------+
          | Server or AWS      | ---> | Job Tracker (Redis)|
          | API Gateway        |      +--------------------+
          +--------------------+               ^
                    |                          |
                    v                          |
          +--------------------+      +--------------------+
          | RabbitMQ or AWS    | ---> | Worker Nodes or    |
          | SQS (Queue)        |      | AWS Lambda         |
          +--------------------+      +--------------------+
                    |                          |
                    v                          |
          +--------------------+               |
          | PostgreSQL (DB)    |<--------------+
          +--------------------+
```

---

## API Endpoints

### Bulk Operation Endpoints

1. **Start Bulk Operation**:
   - **Endpoint**: `/bulk-create` (similar for `/bulk-update` and `/bulk-delete`)
   - **Method**: `POST`
   - **Request Body**:
     ```json
     {
       "entities": [
         { "key": "mykey1", "value": "myvalue1" },
         { "key": "mykey2", "value": "myvalue2" }
       ]
     }
     ```
   - **Response**:
     ```json
     {
       "operationId": "op12345",
       "message": "Bulk operation started"
     }
     ```

### Tracking Endpoints

2. **Get Operation Status**:

   - **Endpoint**: `/status/:operationId`
   - **Method**: `GET`
   - **Response**:
     ```json
     {
       "operationId": "op12345",
       "status": "in-progress",
       "total_batches": 10,
       "processed_batches": 5,
       "failed_batches": 1
     }
     ```

3. **Get Operation Result**:

   - **Endpoint**: `/result/:operationId`
   - **Method**: `GET`
   - **Response**:
     ```json
     {
       "operationId": "op12345",
       "status": "failed",
       "total_batches": 10,
       "processed_batches": 9,
       "failed_batches": 1,
       "errors": [
         "Batch 2 failed: Invalid data format."
       ]
     }
     ```

4. **WebSocket Connection**:

   - **Endpoint**: `/ws`
   - **Description**: Clients can connect to this WebSocket endpoint to receive real-time updates for operations.
   - **Workflow**:
     1. Client connects to `/ws`.
     2. Sends a message with the `operationId` to subscribe:
        ```json
        {
          "operationId": "op12345"
        }
        ```
     3. Server pushes updates whenever `processed_batches` or `failed_batches` changes in Redis:
        ```json
        {
          "operationId": "op12345",
          "status": "in-progress",
          "processed_batches": 6,
          "failed_batches": 1
        }
        ```

---

## Monitoring and Maintenance

### Metrics to Monitor

1. **Redis**:

   - Memory usage to avoid eviction issues.
   - Key expiration rates (if TTL is used).

2. **RabbitMQ or AWS SQS**:

   - Queue length and unacknowledged messages.
   - AWS SQS message visibility timeout and delivery metrics.

3. **Workers, Lambda, or API Gateway**:

   - Task processing rates and error rates.
   - Lambda invocation costs and durations.
   - API Gateway request metrics and error rates.

4. **Database**:

   - Query performance and connection pool utilization.

