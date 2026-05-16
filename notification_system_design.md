# Stage 1

## REST API Contract

The notification platform supports listing notifications, retrieving unread items, marking items as read, creating notifications, bulk delivery, and real-time delivery to active clients.

### Common Headers

```http
Authorization: Bearer <token>
Content-Type: application/json
Accept: application/json
X-Request-ID: <uuid>
```

### List Notifications

`GET /api/v1/notifications?limit=20&page=1&notification_type=Placement&status=unread`

```json
{
  "notifications": [
    {
      "id": "9b3c1a4e-19d7-4b9a-91a7-2bb7ff1df870",
      "studentId": 1042,
      "type": "Placement",
      "title": "Placement update",
      "message": "A new company has opened applications.",
      "isRead": false,
      "createdAt": "2026-04-22T17:51:18Z",
      "readAt": null
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "hasNextPage": true
  }
}
```

### Mark Notification As Read

`PATCH /api/v1/notifications/{notificationId}/read`

```json
{
  "isRead": true,
  "readAt": "2026-04-22T18:00:00Z"
}
```

### Mark All As Read

`PATCH /api/v1/notifications/read-all`

```json
{
  "studentId": 1042
}
```

Response:

```json
{
  "updatedCount": 15
}
```

### Create Notification

`POST /api/v1/notifications`

```json
{
  "recipientStudentIds": [1042, 1043],
  "type": "Placement",
  "title": "Placement update",
  "message": "Applications are open.",
  "channels": ["in_app", "email"],
  "metadata": {
    "company": "Example Corp"
  }
}
```

Response:

```json
{
  "batchId": "f8e0c196-703b-4e52-8f55-c76d86fb4794",
  "acceptedCount": 2,
  "status": "queued"
}
```

## Real-Time Mechanism

Use WebSockets for logged-in users. Each client connects to `/ws/notifications` with a short-lived token. The server authenticates the token, joins the socket to a student-specific channel, and pushes notification payloads after the notification is persisted. If a socket is offline, the client fetches missed notifications from the REST API on reconnect using `createdAfter` or cursor pagination.

# Stage 2

## Storage Choice

PostgreSQL is a strong fit because notifications require relational integrity, indexed lookups by student, transactional writes, and SQL reporting. It supports enum types, JSONB metadata, partitioning, and efficient composite indexes.

## Schema

```sql
CREATE TYPE notification_type AS ENUM ('Event', 'Result', 'Placement');
CREATE TYPE delivery_channel AS ENUM ('in_app', 'email');
CREATE TYPE delivery_status AS ENUM ('queued', 'sent', 'failed');

CREATE TABLE students (
  id BIGSERIAL PRIMARY KEY,
  roll_no VARCHAR(32) NOT NULL UNIQUE,
  email VARCHAR(255) NOT NULL UNIQUE,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE notification_batches (
  id UUID PRIMARY KEY,
  notification_type notification_type NOT NULL,
  title VARCHAR(160) NOT NULL,
  message TEXT NOT NULL,
  metadata JSONB NOT NULL DEFAULT '{}',
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE notifications (
  id UUID PRIMARY KEY,
  batch_id UUID REFERENCES notification_batches(id),
  student_id BIGINT NOT NULL REFERENCES students(id),
  notification_type notification_type NOT NULL,
  title VARCHAR(160) NOT NULL,
  message TEXT NOT NULL,
  is_read BOOLEAN NOT NULL DEFAULT false,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  read_at TIMESTAMPTZ
);

CREATE TABLE notification_deliveries (
  id UUID PRIMARY KEY,
  notification_id UUID NOT NULL REFERENCES notifications(id),
  channel delivery_channel NOT NULL,
  status delivery_status NOT NULL DEFAULT 'queued',
  attempt_count INT NOT NULL DEFAULT 0,
  last_error TEXT,
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Indexes

```sql
CREATE INDEX idx_notifications_student_unread_created
ON notifications (student_id, is_read, created_at DESC);

CREATE INDEX idx_notifications_type_created
ON notifications (notification_type, created_at DESC);

CREATE INDEX idx_deliveries_status_updated
ON notification_deliveries (status, updated_at);
```

## Growth Problems And Solutions

As volume increases, the main risks are slow unread queries, oversized indexes, heavy write bursts, and expensive full-table scans. Use cursor pagination, composite indexes that match access patterns, monthly partitioning by `created_at`, background delivery workers, read replicas for analytics, and retention or archival policies for old notifications.

## Example Queries

```sql
SELECT id, notification_type, title, message, is_read, created_at
FROM notifications
WHERE student_id = $1
ORDER BY created_at DESC
LIMIT $2;

UPDATE notifications
SET is_read = true, read_at = now()
WHERE id = $1 AND student_id = $2;

UPDATE notifications
SET is_read = true, read_at = now()
WHERE student_id = $1 AND is_read = false;
```

# Stage 3

The query is logically accurate if the API needs every unread notification for one student in oldest-first order. It can still be slow because `SELECT *` reads unnecessary columns, the result can be large, and the database may scan many rows if there is no composite index matching `studentID`, `isRead`, and `createdAt`.

Recommended query:

```sql
SELECT id, notification_type, title, message, created_at
FROM notifications
WHERE student_id = 1042
  AND is_read = false
ORDER BY created_at ASC
LIMIT 50;
```

Recommended index:

```sql
CREATE INDEX idx_notifications_unread_student_created
ON notifications (student_id, is_read, created_at ASC);
```

With the index, the database can seek directly to the unread rows for the student and return ordered rows without sorting the entire table. The likely cost becomes `O(log n + k)`, where `k` is the number of unread rows returned, instead of scanning a large portion of the table.

Adding indexes on every column is not effective. Indexes consume storage, slow inserts and updates, increase maintenance cost, and are only useful when they match real query predicates and ordering.

Placement notification query:

```sql
SELECT DISTINCT student_id
FROM notifications
WHERE notification_type = 'Placement'
  AND created_at >= now() - interval '7 days';
```

# Stage 4

Fetching notifications from the database on every page load creates repeated reads for data that often changes slowly. The better approach is a layered strategy.

Use browser caching and conditional requests with `ETag` or `If-Modified-Since` so unchanged inboxes return `304 Not Modified`. This reduces payload size but requires careful invalidation.

Use Redis or another in-memory cache for unread counts and the first page of recent notifications. This removes pressure from PostgreSQL for repeated reads, but introduces cache invalidation complexity.

Use WebSockets to push new notifications to active users. Clients can update their local state immediately and avoid refetching on every page. This improves freshness, but requires connection management and fallback polling.

Use cursor pagination rather than offset pagination. Cursor pagination is stable and efficient for large inboxes, but clients must track cursors instead of page numbers.

Use database indexing and partitioning for long-term scale. This keeps queries fast as data grows, but adds operational complexity.

# Stage 5

The proposed `notify_all` loop is slow and fragile. It performs network calls and database writes serially, has no retry strategy, has no idempotency, and can leave the system in a partial state when one channel fails midway.

If `send_email` fails for 200 students, those failures should be recorded in a delivery table and retried by a worker with exponential backoff. Successful in-app notifications should remain available, and failed email delivery should not require recreating the notification.

Saving to the database and sending email should not happen in one tightly coupled synchronous step. Persist the notification and delivery jobs first, then let workers deliver email and real-time pushes asynchronously. This gives durability, retryability, and better throughput.

```python
function notify_all(student_ids, message):
    batch_id = create_notification_batch(type="Placement", message=message)

    transaction:
        for student_id in student_ids:
            notification_id = insert_notification(
                batch_id=batch_id,
                student_id=student_id,
                message=message,
                is_read=false
            )
            enqueue_delivery(notification_id, channel="email")
            enqueue_delivery(notification_id, channel="in_app")

    return { "batch_id": batch_id, "status": "queued" }

function delivery_worker():
    while true:
        job = claim_next_delivery_job()
        try:
            if job.channel == "email":
                send_email(job.student_id, job.message)
            if job.channel == "in_app":
                push_to_app(job.student_id, job.message)
            mark_delivery_sent(job.id)
        except error:
            mark_delivery_failed(job.id, error)
            schedule_retry(job.id, backoff=next_backoff(job.attempt_count))
```

# Stage 6

Priority is calculated using type weight and recency. Placement notifications receive the highest base weight, followed by Result and Event. Recency is added so newer notifications within the same category appear first.

The actual implementation is in `notification_app_be/src/priority.ts`, with a runnable CLI in `notification_app_be/src/priorityInbox.ts`.

To maintain the top 10 efficiently as new notifications arrive, keep a min-heap of size 10 per user. For each new unread notification, compute its score. If the heap has fewer than 10 items, insert it. If the heap is full and the new score is higher than the smallest score, replace the smallest item. This keeps each update at `O(log 10)`, which is effectively constant time.

# Stage 7

The frontend application is implemented in `notification_app_fe`. It uses React with Material UI, runs on `http://localhost:3000`, supports all notifications, priority notifications, top-n selection, notification type filtering, viewed state tracking in local storage, loading states, and error handling.

