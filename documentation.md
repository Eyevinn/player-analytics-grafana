Grafana and ClickHouse Integration & Dashboard Setup

This guide walks you through all the steps to:

Install Grafana in Docker

Install and configure the ClickHouse plugin

Connect Grafana to your ClickHouse instance

Build and save a series of panels for viewing session and event data

1. **Prerequisites**

Docker installed and running on your machine

ClickHouse server running and accessible

Basic familiarity with SQL and Grafana’s UI

2. **Install Grafana via Docker**

Pull the official Grafana image and run it:

docker run -d \
-p 3000:3000 \
--name=grafana \
grafana/grafana:latest

Wait a few seconds for startup, then open your browser at http://localhost:3000.

Log in with the default credentials:

Username: admin

Password: admin

You’ll be prompted to set a new admin password—do that now.

3. **Install & Enable the ClickHouse Plugin**

In the left-hand menu, click Settings (⚙️) → Plugins.

Search for ClickHouse and select ClickHouse.

Click Install, then Enable.

Restart Grafana if prompted:

docker restart grafana

4. **Add ClickHouse as a Data Source**

In Grafana’s left menu, go to Configuration (⚙️) → Data Sources → Add data source.

Choose ClickHouse.

Fill in connection details:

URL: http://<clickhouse-host>:8123

Default database: default

User / Password as appropriate

Click Save & Test one should see “Data source is working.”

5. **Build Your Dashboards & Panels**

Create a new dashboard (+ → Dashboard), then add panels one by one. For each:

Click Add new panel

Select ClickHouse as the data source

Choose Query type: Time Series (or Table as noted)

Paste the SQL and hit Run Query

Adjust the visualization type (Time series, Bar gauge, Table, etc.)

Title your panel and click Apply

5. **Event Frequencies per Hour/Day/Months/Year**

SELECT
toStartOfHour(timestamp) AS time,
event,
count(*) AS count_of_events
FROM default.epas_default
WHERE timestamp BETWEEN $__from AND $__to
GROUP BY time, event
ORDER BY time ASC

Visualization: Table (or Bar chart)


6. **Save and Share Your Dashboard**

Click Save dashboard in the top-right.

Give it a name (e.g. ClickHouse Playback Metrics).

If desired, share the dashboard link or embed code via the Share button.

Now anyone with Grafana access can explore real‐time session & event data from ClickHouse.