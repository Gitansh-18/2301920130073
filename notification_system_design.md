# Stage 1: Notification System Architecture & API Design

## 1. REST API Contract & Endpoints
All application communication is stateless, utilizing predictable naming conventions and standardized JSON request/response bodies.

### A. Fetch Notifications
* **Endpoint:** `GET /api/v1/notifications`
* **Description:** Retrieves a paginated list of notification events for the authenticated student.
* **Headers:**
  ```http
  Authorization: Bearer <AFFORDMED_BEARER_TOKEN>
  Accept: application/json
### B. Query Parameters
page (integer, default: 1) - The current page window.

limit (integer, default: 10) - Total objects per batch.

notification_type (string, allowed: Event, Result, Placement) - Category filter.

### C. Response (Status Code: 200 OK)
JSON
{
  "success": true,
  "page": 1,
  "limit": 10,
  "total": 142,
  "notifications": [
    {
      "id": "d146095a-0d86-4a34-9e69-3900a14576bc",
      "type": "Result",
      "title": "Mid-Sem Evaluation Result",
      "message": "Results for the mid-sem assessment have been declared.",
      "isRead": false,
      "createdAt": "2026-06-26T11:30:00Z"
    }
  ]
}
## 2. JSON Validation Schemas
Below are the strict structure schemas enforced by our application processing layer.

Notification Object Schema
JSON
{
  "$schema": "[http://json-schema.org/draft-07/schema#](http://json-schema.org/draft-07/schema#)",
  "title": "Notification",
  "type": "object",
  "properties": {
    "id": { "type": "string", "format": "uuid" },
    "type": { "type": "string", "enum": ["Event", "Result", "Placement"] },
    "title": { "type": "string", "maxLength": 100 },
    "message": { "type": "string", "maxLength": 500 },
    "isRead": { "type": "boolean" },
    "createdAt": { "type": "string", "format": "date-time" }
  },
  "required": ["id", "type", "title", "message", "isRead", "createdAt"]
}
## 3. Real-Time Notification Broadcast Mechanism
To meet the core requirement for broadcasting real-time notification updates to students without exhausting database connections via HTTP polling, we utilize WebSockets (via Socket.io).

Architectural Flow Diagram
Trigger: An admin or background worker publishes a notification (e.g., a new Placement Drive).

Event Dispatch: The backend engine intercepts the notification object, validates it against the schema, and writes it to persistent storage.

Broadcasting Layer: The server pushing engine maps connected active student connection descriptors and shoots a direct network event over the established duplex persistent channel.

Client Reception: The frontend UI instantly intercepts the packet event and shifts application state to add the notification item to the tray badge in real time.

WebSocket Event Signature
Connection Event: connection (Requires passing authentication token via handshake query string).

Incoming Push Payload Event Name: notification:received

JSON
{
  "id": "e5c4ff20-31bf-4d40-8f02-72fda59e8918",
  "type": "Placement",
  "title": "New Drive: CSX Corporation",
  "message": "CSX Corporation hiring application window is now open.",
  "createdAt": "2026-06-26T11:32:00Z"
}
# Stage 2: Database Storage Engine & Scalability Modeling
## 1. Persistent Storage Selection: PostgreSQL (Relational DBMS)
For an institutional notification system handling student records, exam results, and career placement timelines, a relational model like PostgreSQL is chosen over NoSQL alternatives for the following reasons:

ACID Compliance & Strong Consistency: Financial-level data integrity constraints guarantee that when a high-priority placement or result notification is broadcasted, no phantom or missing data reads occur across user profiles.

Complex Multi-Table Joins: Seamless operational handling for pairing student academic status filters, tracking individual viewed/unviewed metrics, and optimizing data retrieval paths.

Advanced Indexing and Partitioning Support: Excellent built-in capability for handling structural partitions when records scale into tens of millions.

## 2. Comprehensive Database Schema (SQL DDL)
SQL
-- Enums for strict type control
CREATE TYPE notification_category AS ENUM ('Event', 'Result', 'Placement');

-- Core Notification Template Library Table
CREATE TABLE notification_templates (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    type notification_category NOT NULL,
    title VARCHAR(100) NOT NULL,
    message TEXT NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Student Notification Mapping and Status Table
CREATE TABLE student_notifications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_id BIGINT NOT NULL, -- Corresponds to the unique Student Roll Number
    template_id UUID REFERENCES notification_templates(id) ON DELETE CASCADE,
    is_read BOOLEAN DEFAULT FALSE NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
## 3. Mitigating Scalability Challenges as Volume Increases
When data climbs to 50,000+ students and millions of notification vectors, standard database tables face serious read/write latency bottlenecks due to sequential full table scans. We implement architectural mitigation pillars:

Database Horizontal Table Partitioning: Partition the student_notifications mapping ledger table by hashing or ranging the created_at timestamp field. Keeping hot active operational quarters physically distinct from archived cycles dramatically optimizes storage.

Read/Write Replica Segregation: Configure a Primary database machine to exclusively process ingestion traffic (incoming broadcast mutations) while replicating to dual Read Replica mirror nodes to process student tray query requests.

Compound Index B-Trees: Establish strategic multi-column indexing paths across lookups matching standard API route parameters.

## 4. Implementation Queries
### A. Fetching Unread Notifications for a Student (Matches Stage 1 API endpoint)
SQL
SELECT 
    sn.id, 
    nt.type, 
    nt.title, 
    nt.message, 
    sn.is_read, 
    sn.created_at
FROM student_notifications sn
JOIN notification_templates nt ON sn.template_id = nt.id
WHERE sn.student_id = 2301920130073  -- Target Student Roll ID
  AND sn.is_read = FALSE
ORDER BY sn.created_at DESC
LIMIT 10 OFFSET 0;
### B. Mass Ingestion / Broadcasting Query Example
SQL
-- Inserting a centralized notification blueprint template
INSERT INTO notification_templates (type, title, message)
VALUES ('Placement', 'CSX Corporation Hiring', 'CSX Corporation application window has opened.');
# Stage 3: Query Optimization and Indexing
## 1. Analysis of the Existing Slow Query
The current query being executed on the system is:

SQL
SELECT * FROM notifications
WHERE studentID = 1042 AND isRead = false
ORDER BY createdAt ASC;
Why is this query running slowly?
Full Table Scan: With 5,000,000 notifications in the table, if there is no index covering the studentID column, the database engine is forced to scan all 5 million rows line-by-line just to find student 1042.

Ordering Overhead: The ORDER BY createdAt ASC clause requires the database to sort the filtered results in memory. Without an index on createdAt, this file-sort operational cost grows exponentially with data volume.

Unoptimized Asterisk (*): Fetching all columns via SELECT * unnecessarily increases disk I/O and network payload bandwidth.

Evaluation of Blanket Index Advice
Another developer suggested adding indexes to every single column to be safe.

Verdict: This advice is highly ineffective. While it speeds up reads slightly, creating a separate index for every column dramatically degrades database write/ingestion speeds (since every INSERT or UPDATE must rebuild all indexes). It also wastes massive amounts of memory storage.

## 2. The Optimized Index Solution
Instead of individual blanket indexes, we implement a targeted Composite (Multi-column) B-Tree Index.

SQL
CREATE INDEX idx_student_unread_notifications 
ON student_notifications (student_id, is_read, created_at ASC);
Why this works perfectly:
This single compound index satisfies the WHERE filters and the ORDER BY sorting path simultaneously. The database can jump directly to the student via a quick binary search tree, filter out the read items instantly, and pull them out pre-sorted.

## 3. Verification Query (7-Day Last Placement Notification)
SQL
SELECT 
    sn.student_id,
    nt.title,
    sn.created_at
FROM student_notifications sn
JOIN notification_templates nt ON sn.template_id = nt.id
WHERE nt.type = 'Placement'
  AND sn.created_at >= CURRENT_TIMESTAMP - INTERVAL '7 days';
# Stage 4: Performance Improvements for Notification Fetching
## 1. The Core Problem: Database Bottleneck on Page Load
When 50,000+ students log into the platform simultaneously (e.g., during placement season), fetching notifications directly from the disk-bound PostgreSQL database on every single page load will saturate connection pools and result in high latencies or database crashes.

## 2. Proposed Architectural Solution: In-Memory Caching (Redis)
To alleviate this bottleneck, an In-Memory Caching Layer (using Redis) is introduced between the application server and PostgreSQL database to store unread notification metadata.

## 3. Caching Strategies & Trade-offs
Strategy A: Cache-Aside (Lazy Loading)
How it works: The application first checks Redis for the student's notification packet. If a cache miss occurs, it queries PostgreSQL, populates the cache, and returns the response.

Pros: Highly resource-efficient; memory is only consumed for students who actively log in.

Cons: The very first page load for a user experiences slight latency (cache miss cost).

Strategy B: Write-Through / Pre-Caching
How it works: When a notification is generated, it is simultaneously written to both PostgreSQL and the active Redis cache segments for all targeted students.

Pros: Blazing fast 100% cache hits for students upon logging in.

Cons: High write-amplification cost and massive memory consumption if caching data for inactive students.

## 4. Implementation Choice & Justification
We choose a Hybrid Cache-Aside Strategy with an aggressive Time-To-Live (TTL) expiration of 15 minutes. Unread notifications change frequently as students click them; a short TTL prevents stale cache issues while completely insulating the core database from redundant, rapid page-refresh traffic.

# Stage 5: Reliable Notification Broadcasting Redesign
## 1. Shortcomings of the Initial Implementation
The initial synchronous loop design executes network I/O blockades point-by-point. If an external vendor API breaks down midway, the loop halts completely, creating system-wide blockages.

## 2. Architectural Redesign: Asynchronous Message Queue
To make broadcasting robust, reliable, and fast, we decouple ingestion from delivery using an Asynchronous Message Queue (RabbitMQ or Apache Kafka).

## 3. Revised Asynchronous Pseudocode
JavaScript
function broadcastNotification(notificationData, studentIdList) {
    const templateId = db.saveTemplate(notificationData);
    for (let studentId of studentIdList) {
        const jobPayload = { studentId, templateId, message: notificationData.message };
        messageQueue.publish("notification_broadcast_pool", jobPayload);
    }
    return { status: "Broadcast processing initiated sequentially." };
}

messageQueue.consume("notification_broadcast_pool", async (job) => {
    try {
        await db.saveToStudentLedger(job.studentId, job.templateId);
        await websocketServer.sendToUser(job.studentId, job.message);
        await emailClient.sendEmail(job.studentId, job.message);
    } catch (error) {
        await Log("backend", "error", "cron_job", `Delivery failure for user ${job.studentId}: ${error.message}`);
        await messageQueue.moveToDeadLetter(job);
    }
});

