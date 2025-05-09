# Grafana and ClickHouse Integration for Analytics Pipeline

## Architecture Overview

The complete analytics pipeline consists of the following components:

```
[Video Player/Client] → [Eyevinn SDK] → [ClickHouse DB] → [Grafana]
```

This guide focuses on setting up the visualization layer (Grafana) that connects to the ClickHouse database, allowing to build dashboards for analyzing playback sessions and events.


## Prerequisites

Before starting this guide, one should have:
- An existing ClickHouse instance with analytics data (this is already set up as part of the analytics worker)
  -  ClickHouse connection details:
  - Endpoint URL: URL: `https://<your-clickhouse-endpoint>/play`
  - Database name (typically `epas_default`)
  - `<Clickhouse-Username>` and `<Clickhouse-password>`

## Step 1: Create a Grafana Instance

There are two options for running Grafana:

### Option A: Provision Grafana on Eyevinn OSC (Recommended)

1. **Launch Grafana**
   - Go to OSC UI → Web Services → Grafana → Create Grafana
   - Set name: `grafana` 
   - Plugins to Preinstall: `vertamedia-clickhouse-datasource`
   - Click "Create" and wait for the status to change to "running."
  
 <img width="448" alt="Screenshot 2025-05-06 at 10 55 08" src="https://github.com/user-attachments/assets/9b7ca314-d547-4291-a9cc-6d3c02e23e33" />

  

 <img width="405" alt="Screenshot 2025-04-24 at 11 33 31" src="https://github.com/user-attachments/assets/937a50e6-2fda-44cb-8204-0041330b2ca6" />



2. **Log in to Grafana**
   - Open the provided URL
   - Default credentials:
   - Username: `admin`
   - Password: `admin`
   - Set a new password when prompted

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
  
     
 <img width="652" alt="Screenshot 2025-05-08 at 12 55 38" src="https://github.com/user-attachments/assets/d2c025cc-2ffd-446b-aebc-a4d792c54804" />





 <img width="701" alt="Screenshot 2025-05-08 at 12 55 51" src="https://github.com/user-attachments/assets/bc119c38-a153-4924-8bc6-4e01952d8b65" />




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

<img width="692" alt="Screenshot 2025-05-08 at 12 52 46" src="https://github.com/user-attachments/assets/60c48be3-faa6-4371-9a22-84a974b3670b" />


<img width="696" alt="Screenshot 2025-05-08 at 12 53 05" src="https://github.com/user-attachments/assets/f995d9a5-edb9-49df-abed-ba468ac703c7" />


<img width="694" alt="Screenshot 2025-05-08 at 12 52 56" src="https://github.com/user-attachments/assets/0b64cd03-2baa-4c3f-b40d-8bd57cdbbc4c" />





Remember to save the dashboard regularly as you build it!
