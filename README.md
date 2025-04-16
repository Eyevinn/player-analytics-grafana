# Grafana + ClickHouse Integration & Dashboard Setup

This guide walks you through all the steps to:

- Install Grafana in Docker
- Install and configure the ClickHouse plugin
- Connect Grafana to your ClickHouse instance
- Build and save a series of panels for viewing session and event data

## 1. Prerequisites

- Docker installed and running on your machine
- ClickHouse server running and accessible
- Basic familiarity with SQL and Grafana's UI

## 2. Install Grafana via Docker

Pull the official Grafana image and run it:

```bash
docker run -d \
  -p 3000:3000 \
  --name=grafana \
  grafana/grafana:latest
```

Wait a few seconds for startup, then open your browser at http://localhost:3000.

Log in with the default credentials:
- Username: admin
- Password: admin

You'll be prompted to set a new admin password—do that now.

## 3. Install & Enable the ClickHouse Plugin

In the left-hand menu, click Settings (⚙️) → Plugins.

Search for ClickHouse and select ClickHouse.

Click Install, then Enable.

Restart Grafana if prompted:

```bash
docker restart grafana
```

## 4. Add ClickHouse as a Data Source

In Grafana's left menu, go to Configuration (⚙️) → Data Sources → Add data source.

Choose ClickHouse.

Fill in connection details:
- URL: http://<clickhouse-host>:8123
- Default database: default
- User / Password as appropriate

Click Save & Test—you should see "Data source is working."

## 5. Build Your Dashboards & Panels

Create a new dashboard (+ → Dashboard), then add panels one by one. For each:

1. Click Add new panel
2. Select ClickHouse as the data source
3. Choose Query type: Time Series (or Table as noted)
4. Paste the SQL and hit Run Query
5. Adjust the visualization type (Time series, Bar gauge, Table, etc.)
6. Title your panel and click Apply

### 5.1. Event Frequencies per Hour

```sql
SELECT
toStartOfHour(timestamp) AS time,
event,
count(*) AS count_of_events
FROM default.epas_default
WHERE timestamp BETWEEN $__from AND $__to
GROUP BY time, event
ORDER BY time ASC
```

Visualization: Time series (lines)

### 5.2. Concurrent Sessions per Hour

```sql
SELECT
toStartOfHour(timestamp) AS time,
countDistinct(sessionId) AS concurrent_sessions
FROM default.epas_default
WHERE $__timeFilter(timestamp)
GROUP BY time
ORDER BY time ASC
```

Visualization: Time series (line)

### 5.3. Distribution of Event Types

```sql
SELECT
event,
count(*) AS total_count
FROM default.epas_default
GROUP BY event
ORDER BY total_count DESC
```

Visualization: Bar chart or Pie chart

### 5.4. Top Content Titles (Metadata Events)

```sql
SELECT
JSONExtractString(payload,'contentTitle') AS contentTitle,
count(*) AS total_count
FROM default.epas_default
WHERE event = 'metadata'
GROUP BY contentTitle
ORDER BY total_count DESC
```

Visualization: Table (or Bar chart)

### 5.5. Top Videos (with Filtering)

```sql
SELECT
  JSONExtractString(payload, 'contentTitle') AS content_title,
  COUNT(*) AS play_count
FROM default.epas_default
WHERE
  event = 'metadata'
  AND JSONExtractString(payload, 'contentTitle') != ''
  AND NOT startsWith(JSONExtractString(payload, 'contentTitle'), 'Report: ["myTime"')
  AND $__timeFilter(timestamp)
GROUP BY content_title
ORDER BY play_count DESC
LIMIT 10
```

Visualization: Table or Bar chart

### 5.6. Daily Video Count (Normalized Titles)

```sql
WITH merged AS (
  SELECT
    CASE
      WHEN lower(JSONExtractString(payload, 'contentTitle')) LIKE '%hls%'
        THEN 'HLS Stream'
      WHEN lower(JSONExtractString(payload, 'contentTitle')) LIKE '%sample%'
        THEN 'Sample Video'
      ELSE JSONExtractString(payload, 'contentTitle')
    END AS normalized_contentTitle,
    COUNT(*) AS total_plays
  FROM default.epas_default
  WHERE
    event = 'metadata'
    AND JSONExtractString(payload, 'contentTitle') != ''
    AND $__timeFilter(timestamp)
  GROUP BY normalized_contentTitle
)
SELECT
  normalized_contentTitle AS contentTitle,
  total_plays
FROM merged
ORDER BY total_plays DESC
LIMIT 10
```

Visualization: Bar chart or Table

### 5.7. Monthly Top Videos

```sql
WITH monthly_video_counts AS (
  SELECT
    toStartOfMonth(timestamp) AS month,
    JSONExtractString(payload, 'contentTitle') AS content_title,
    COUNT(*) AS play_count
  FROM default.epas_default
  WHERE
    event = 'metadata'
    AND JSONExtractString(payload, 'contentTitle') != ''
    AND NOT startsWith(JSONExtractString(payload, 'contentTitle'), 'Report: ["myTime"')
  GROUP BY
    month, content_title
),
ranked_monthly_videos AS (
  SELECT
    *,
    ROW_NUMBER() OVER (PARTITION BY month ORDER BY play_count DESC) AS rank
  FROM monthly_video_counts
)
SELECT
  month AS time,
  content_title,
  play_count AS count_per_month
FROM ranked_monthly_videos
WHERE rank <= 10
ORDER BY
  time ASC,
  count_per_month DESC
```

Visualization: Time series or Table

### 5.8. Playback Errors Over Time

```sql
SELECT
    toStartOfMinute(timestamp) AS time,
    JSONExtractString(payload, 'reason') AS reason,
    count(*) AS count_events
FROM epas_default
WHERE
    event = 'stopped'
    AND JSONExtractString(payload, 'reason') = 'error'
    AND $__timeFilter(timestamp)
GROUP BY time, reason
ORDER BY time ASC
```

Visualization: Time series

### 5.9. Session‐level Detail for a Given Session ID

```sql
SELECT
timestamp AS time,
sessionId,
JSONExtractString(payload,'contentTitle') AS contentTitle,
payload
FROM default.epas_default
WHERE event = 'metadata'
  AND sessionId = '$sessionId'
ORDER BY timestamp ASC
```

Visualization: Table (optionally switch to Time series)

Dashboard Variable: Create a Query variable named sessionId that does

```sql
SELECT DISTINCT sessionId FROM default.epas_default WHERE sessionId NOT LIKE '%localhost%'
```

### 5.10. Top 5 Videos & Top 5 Users per Video (6‑hour buckets)

```sql
WITH unified_titles AS (
    SELECT
        multiIf(
            JSONExtractString(payload,'contentTitle') LIKE '%Sample%', 'Sample Video',
            JSONExtractString(payload,'contentTitle') LIKE '%HLS%',    'HLS content',
            'skip'
        ) AS unifiedTitle,
        sessionId,
        timestamp
    FROM default.epas_default
    WHERE event = 'metadata'
      AND sessionId NOT LIKE '%localhost%'
),

top_users AS (
    SELECT unifiedTitle, sessionId, count(*) AS play_count
    FROM unified_titles
    WHERE unifiedTitle IN ('Sample Video','HLS content')
    GROUP BY unifiedTitle, sessionId
    ORDER BY unifiedTitle, play_count DESC
    LIMIT 5 BY unifiedTitle
)

SELECT
toStartOfInterval(timestamp, INTERVAL 6 HOUR) AS time_bucket,
unifiedTitle,
sessionId,
count(*) AS event_count
FROM unified_titles
WHERE unifiedTitle IN ('Sample Video','HLS content')
  AND (unifiedTitle, sessionId) IN (SELECT unifiedTitle, sessionId FROM top_users)
GROUP BY time_bucket, unifiedTitle, sessionId
ORDER BY time_bucket ASC, event_count DESC
```

Visualization: Table

## 6. Save and Share Your Dashboard

Click Save dashboard in the top-right.

Give it a name (e.g. ClickHouse Playback Metrics).

If desired, share the dashboard link or embed code via the Share button.

Now anyone with Grafana access can explore real‐time session & event data from ClickHouse.