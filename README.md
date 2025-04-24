# Grafana and ClickHouse Integration for Analytics Pipeline

## Architecture Overview

The complete analytics pipeline consists of the following components:

```
[Video Player/Client] → [Eyevinn SDK] → [ClickHouse DB] → [Grafana]
```

This guide focuses on setting up the visualization layer (Grafana) that connects to the ClickHouse database, allowing to build dashboards for analyzing playback sessions and events.


## Prerequisites

Before starting this guide, one should have:
- An existing ClickHouse instance with analytics data (this is typically already set up as part of the analytics worker)
  -  ClickHouse connection details:
  - Endpoint URL: https://<your-clickhouse-endpoint>/play
  - Database name (typically `epas_default`)
  - `<Clickhouse-Username>` and `<Clickhouse-password>`

## Step 1: Create a Grafana Instance

There are two options for running Grafana:

### Option A: Provision Grafana on Eyevinn OSC (Recommended)

1. **Create the admin password secret**
   - In OSC UI → Web Services → Service Secrets → New Secret
   - Name: `grafana` (or your preferred name)
   - `<Value>`: Desired password
   - Click "Create Secret"

  <img width="434" alt="Screenshot 2025-04-24 at 11 25 23" src="https://github.com/user-attachments/assets/fdeb0566-d53e-4d7d-8736-a5d20d767907" />
   


2. **Launch Grafana**
   - Go to OSC UI → Web Services → Grafana → Create Grafana
   - Set name: `grafana` 
   - Plugins to Preinstall: `vertamedia-clickhouse-datasource`
   - Attach the secret created
   - Click "Create" and wait for the status to change to "running."
  

  <img width="405" alt="Screenshot 2025-04-24 at 11 33 31" src="https://github.com/user-attachments/assets/937a50e6-2fda-44cb-8204-0041330b2ca6" />




3. **Bind the secret to the instance**
   - Go to "My grafanas" → find your instance → "⋮" → Instance parameters
   - Select the admin password secret → Save

4. **Log in to Grafana**
   - Open the provided URL
   - `<Username>`: `admin`
   - `<Password>`: (the value set in your secret)

### Option B: Run Grafana Locally with Docker

1. **Pull and run the Grafana container**
   ```bash
   docker run -d \
     -p 3000:3000 \
     --name=grafana \
     grafana/grafana:latest
   ```

2. **Access Grafana**
   - Open browser at http://localhost:3000
   - Default credentials:
     - Username: `admin`
     - Password: `admin`
   - Set a new password when prompted

3. **Install the ClickHouse Plugin**
   - Go to Settings (⚙️) → Plugins
   - Search for "ClickHouse"
   - Select and click "Install" → "Enable"
   - Restart Grafana if prompted:
     ```bash
     docker restart grafana
     ```

## Step 2: Connect Grafana to ClickHouse

1. **Add ClickHouse as a data source**
   - In the left menu, go to Configuration (⚙️) → Data Sources → Add data source
   - Choose "ClickHouse" (or "Altinity plugin for ClickHouse" if using OSC)
   - Enter your connection details:
     - URL: `https://<your-clickhouse-endpoint>/play`
     - Default database: database name (typically `epas_default`)
     - Basic Auth: Enable
     - User: `<ClickHouse username>`
     - Password: `<ClickHouse password>`
   - Click "Save & Test" - Check "Data source is working"

## Step 3: Create Analytics Dashboards

### Option A: Import the Sample Dashboard (Quickest)

1. In Grafana's sidebar, click "+ → Import"
2. Either:
   - Upload the JSON file from `/sample-grafana-dashboard/Clickhouse-1745399875287.json` in the repository
   - Or paste the contents of that JSON file
3. Select the ClickHouse data source when prompted
4. Click "Import"

### Option B: Build Custom Dashboard Panels

Create a new dashboard ("+ → Dashboard"), then add panels using these example queries:

#### Event Frequencies Panel

```sql
SELECT
  toStartOfHour(timestamp) AS time,
  event,
  count(*) AS count_of_events
FROM epas_default
WHERE timestamp BETWEEN $__from AND $__to
GROUP BY time, event
ORDER BY time ASC
```
*Visualization: Time series graph*

#### Most Popular Content

```sql
SELECT
  JSONExtractString(payload, 'contentTitle') AS content_title,
  COUNT(*) AS play_count
FROM epas_default
WHERE
  event = 'metadata'
  AND JSONExtractString(payload, 'contentTitle') != ''
  AND $__timeFilter(timestamp)
GROUP BY content_title
ORDER BY play_count DESC
LIMIT 10
```
*Visualization: Bar chart*

#### Playback Errors

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
*Visualization: Time series or table*

## Extending Your Dashboards

One can customize and extend the dashboards using ClickHouse SQL queries. Here are some useful patterns:

### 1. Time-based Grouping

Adjust time granularity by changing the time function:
```sql
-- Group by hour
toStartOfHour(timestamp) AS time

-- Group by day
toStartOfDay(timestamp) AS time

-- Group by week
toStartOfWeek(timestamp) AS time
```

### 2. JSON Field Extraction

Access nested data in the payload field:
```sql
-- Extract a single value
JSONExtractString(payload, 'fieldName') AS extracted_field

-- Extract nested values
JSONExtractString(JSONExtractString(payload, 'parent'), 'child') AS nested_field
```

### 3. Filtering By Event Type

```sql
-- Filter to specific events
WHERE event IN ('started', 'paused', 'resumed')

-- Combine with time range
WHERE event = 'error' AND $__timeFilter(timestamp) 
```

### 4. Using Variables

Create dashboard variables to make filters interactive:
```sql
-- Query for a dropdown of content titles
SELECT DISTINCT JSONExtractString(payload, 'contentTitle') FROM epas_default WHERE event = 'metadata'

-- Then use in your panel queries
WHERE JSONExtractString(payload, 'contentTitle') = '$content_title'
```
<img width="851" alt="Screenshot 2025-04-24 at 11 28 28" src="https://github.com/user-attachments/assets/9f70e8dd-6f45-409f-81c0-211f23aa613e" />






<img width="431" alt="Screenshot 2025-04-24 at 11 30 34" src="https://github.com/user-attachments/assets/4cfa8951-bd37-45a5-be03-8db63596abdf" />








<img width="669" alt="Screenshot 2025-04-24 at 11 30 53" src="https://github.com/user-attachments/assets/a63cfbb9-68b9-4c18-9986-28478b6d228f" />








Remember to save the dashboard regularly as you build it!
