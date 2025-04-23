# Grafana and ClickHouse Integration & Dashboard Setup

This guide walks you through all the steps to:
- Install Grafana in Docker
- Install and configure the ClickHouse plugin
- Connect Grafana to your ClickHouse instance
- Build and save a series of panels for viewing session and event data

## 0. Provision a ClickHouse Server on Eyevinn OSC

If you haven’t yet created a ClickHouse instance in Eyevinn’s Open Source Cloud, follow the quick start in the Eyevinn OSC docs:
[Service: ClickHouse – Getting Started](https://docs.osaas.io/osaas.wiki/Service%3A-ClickHouse.html)

1. **Sign up** for a free Eyevinn OSC account (if you don’t already have one).
2. **Setup secrets**
    - In the OSC web UI, create a **service secret** to hold the password for the default ClickHouse user.
3. **Create your server instance**
    - Click **Create clickhouse-server**, fill in the dialog (name, region, node size, attach your secret), then hit **Create**.
    - Wait until the status reads **Running**.
4. **Grab your connection details**
    - Copy the server’s endpoint URL, default database name, and the credentials you stored in your secret.

With ClickHouse endpoint, database, username and password on hand, you’re ready to move on.

## 0.2. Provision Grafana 

You can also spin up a managed Grafana in OSC rather than running it locally:

In the OSC web UI, go to Web Services → Grafana.

https://app.osaas.io/dashboard/service/grafana-grafana

Click Create grafana.

Fill in the dialog:

Name: e.g. my-grafana

Region/Size: choose as needed

Attach Secret: create/select a secret for the admin user password (e.g. grafana-admin-pw).

Click Create and wait until it shows Running.

Copy the Public Endpoint URL for Grafana (e.g. https://grafana-...auto.prod.osaas.io).

Open it in your browser, and log in with:

Username: admin

Password: the value from your grafana-admin-pw secret.

Tip: If you prefer, you can skip managed Grafana and instead install it locally via Docker (see Section 1).



---

## 1. Prerequisites

- Docker installed and running on your machine
- A running ClickHouse server (see Section 0)
- Basic familiarity with SQL and Grafana’s UI

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
- **Username:** admin
- **Password:** admin

You’ll be prompted to set a new admin password now.

## 3. Install & Enable the ClickHouse Plugin

1. In the left-hand menu, click **Settings (⚙️) → Plugins**.
2. Search for **ClickHouse**, select it, then click **Install** → **Enable**.
3. Restart Grafana if prompted:
   ```bash
   docker restart grafana
   ```

## 4. Add ClickHouse as a Data Source

1. In the left menu, go to **Configuration (⚙️) → Data Sources → Add data source**.
2. Choose **ClickHouse**.
3. Fill in your connection details (example placeholders below):

   | Field                | Value                                     |
      |----------------------|-------------------------------------------|
   | **URL**              | `https://<your-osc-endpoint>/play`        |
   | **Default database** | `epas_default`                            |
   | **User**             | `<your-secret-username>`                  |
   | **Password**         | `<your-secret-password>`                  |

4. Click **Save & Test** – you should see “Data source is working.”

## 5. Build Dashboards & Panels

Create a new dashboard (+ → **Dashboard**), then add panels one by one. For each:

1. Click **Add new panel**
2. Select **ClickHouse** as the data source
3. Choose **Query type**: Time series (or Table)
4. Paste your SQL and hit **Run query**
5. Adjust the visualization (Time series, Bar gauge, Table, etc.)
6. Title your panel and click **Apply**

### 5.1. Event Frequencies per Hour/Day/Month

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

**Visualization:** Bar chart or Pie chart

### 5.2. Distribution of Event Types

```sql
SELECT
  event,
  count(*) AS total_count
FROM default.epas_default
GROUP BY event
ORDER BY total_count DESC
```

**Visualization:** Bar chart or Pie chart

### 5.3. Top Videos (with Filtering)

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

![Analytics Dashboard Example 1](media/image1.png)
![Analytics Dashboard Example 2](media/image2.png)

### 5.4. Playback Errors Over Time

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

**Visualization:** Table or Bar chart, Time series, etc.

## 6. Save and Share Your Dashboard

Click **Save dashboard** in the top-right.
Give it a name (e.g. “ClickHouse Playback Metrics”).
Use the **Share** button to copy a link or embed code.

---

## 7. Import the Provided Sample Dashboard

To get started even faster, we’ve included a ready-made Grafana dashboard JSON in this repo:

```text
/sample-grafana-dashboard/Clickhouse-1745399875287.json
```

1. In Grafana’s sidebar click **+ → Import**.
2. Either paste the contents of that JSON file, or point Grafana at the raw URL (if you’re hosting it).
3. Select your ClickHouse data source when prompted.
4. Click **Import** — you’ll now have a pre-built dashboard you can customize!

---

Now you—and anyone on your team—can spin up Grafana, connect to ClickHouse in Eyevinn OSC, and immediately start exploring session & event metrics with a fully-fleshed template.
