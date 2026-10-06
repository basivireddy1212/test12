That error usually happens because Bash is seeing placeholders like `<MY_PROJECT_ID>` or `<...>` as shell redirection operators.

Use the commands **without `< >`**.

### 1. Set your GCP project

Replace `my-gcp-project-123` with your actual project ID:

```bash
gcloud config set project my-gcp-project-123
```

### 2. Create the dataset

```bash
bq mk --location=US tfc_logs
```

### 3. Create the BigQuery table

Use this exact command:

```bash
bq query --use_legacy_sql=false '
CREATE TABLE `tfc_logs.audit_logs` (
  id STRING,
  timestamp TIMESTAMP,
  action STRING,
  organization STRING,
  request STRUCT<
    id STRING
  >,
  resource STRUCT<
    id STRING,
    type STRING,
    action STRING
  >,
  auth STRUCT<
    id STRING,
    username STRING,
    email STRING
  >
)'
```

### 4. Convert your JSON to NDJSON

Your original file is an array:

```json
[
  {
  "id": "audit-003",
  "timestamp": "2026-10-06T02:30:00Z",
  "action": "run.create",
  "organization": "my-org",
  "request": {
    "id": "ws-123456"
  },
  "resource": {
    "id": "run-abc123",
    "type": "run",
    "action": "errored"
  },
  "auth": {
    "id": "user-123",
    "username": "admin",
    "email": "admin@example.com"
  }
}
]
```

Run:

```bash
jq -c '.[]' tfc-audit.json > tfc-audit.ndjson
```

Check it:

```bash
cat tfc-audit.ndjson
```

You should see **one JSON object on one line**.

### 5. Load it into BigQuery

```bash
bq load \
  --source_format=NEWLINE_DELIMITED_JSON \
  tfc_logs.audit_logs \
  tfc-audit.ndjson
```

### 6. Verify

```bash
bq query --use_legacy_sql=false '
SELECT
  id,
  timestamp,
  action,
  organization,
  request.id AS workspace_id,
  resource.id AS run_id,
  resource.type AS resource_type,
  resource.action AS resource_action,
  auth.id AS user_id,
  auth.username,
  auth.email
FROM `tfc_logs.audit_logs`
'
```

### Expected result

```text
audit-003
2026-10-06 02:30:00 UTC
run.create
my-org
ws-123456
run-abc123
run
errored
user-123
admin
admin@example.com
```

**If you're running this on Windows**, tell me whether you're using **PowerShell, CMD, or Git Bash**. The commands are slightly different, especially the multiline `bq query` command.





If you mean **analytics on the 100 TFC audit records in BigQuery**, a good setup is:

```text
TFC Audit Logs
      ↓
BigQuery
      ↓
SQL Views / Tables
      ↓
Looker Studio / Looker
      ↓
TFC Analytics Dashboard
```

## 1. First create useful analytics views

### Run status summary

```sql
CREATE OR REPLACE VIEW `tfc_logs.v_run_status_summary` AS
SELECT
  resource.action AS run_state,
  COUNT(*) AS run_count
FROM `tfc_logs.audit_logs`
WHERE resource.type = 'run'
GROUP BY run_state
ORDER BY run_count DESC;
```

Query it:

```sql
SELECT *
FROM `tfc_logs.v_run_status_summary`;
```

You'll get something like:

```text
run_state          run_count
----------------------------
created            10
pending            10
planning           10
planned            10
applying           10
applied            10
errored            10
canceled           10
discarded          10
policy_override    10
```

---

## 2. Workspace analytics

This tells you how many events each workspace has.

```sql
CREATE OR REPLACE VIEW `tfc_logs.v_workspace_analytics` AS
SELECT
  request.id AS workspace_id,
  COUNT(*) AS total_events,
  COUNTIF(resource.action = 'applied') AS successful_runs,
  COUNTIF(resource.action = 'errored') AS failed_runs,
  COUNTIF(resource.action = 'canceled') AS canceled_runs,
  COUNTIF(resource.action = 'planning') AS planning_runs,
  COUNTIF(resource.action = 'applying') AS applying_runs
FROM `tfc_logs.audit_logs`
WHERE resource.type = 'run'
GROUP BY workspace_id;
```

---

## 3. Workspace failure rate

This is more useful for a dashboard.

```sql
CREATE OR REPLACE VIEW `tfc_logs.v_workspace_failure_rate` AS
SELECT
  request.id AS workspace_id,

  COUNT(*) AS total_runs,

  COUNTIF(resource.action = 'applied')
    AS successful_runs,

  COUNTIF(resource.action = 'errored')
    AS failed_runs,

  ROUND(
    SAFE_DIVIDE(
      COUNTIF(resource.action = 'errored'),
      COUNT(*)
    ) * 100,
    2
  ) AS failure_rate_percent

FROM `tfc_logs.audit_logs`

WHERE resource.type = 'run'

GROUP BY workspace_id;
```

---

## 4. User analytics

Find who is triggering the runs.

```sql
CREATE OR REPLACE VIEW `tfc_logs.v_user_activity` AS
SELECT
  auth.username,
  auth.email,

  COUNT(*) AS total_events,

  COUNTIF(resource.action = 'applied')
    AS successful_runs,

  COUNTIF(resource.action = 'errored')
    AS failed_runs,

  COUNTIF(resource.action = 'canceled')
    AS canceled_runs

FROM `tfc_logs.audit_logs`

WHERE resource.type = 'run'

GROUP BY
  auth.username,
  auth.email

ORDER BY total_events DESC;
```

---

# 5. Organization analytics

```sql
CREATE OR REPLACE VIEW `tfc_logs.v_organization_analytics` AS
SELECT
  organization,

  COUNT(*) AS total_events,

  COUNTIF(resource.action = 'applied')
    AS successful_runs,

  COUNTIF(resource.action = 'errored')
    AS failed_runs,

  COUNTIF(resource.action = 'canceled')
    AS canceled_runs

FROM `tfc_logs.audit_logs`

WHERE resource.type = 'run'

GROUP BY organization

ORDER BY total_events DESC;
```

---

# 6. Daily analytics

For production data, this becomes very useful.

```sql
CREATE OR REPLACE VIEW `tfc_logs.v_daily_runs` AS
SELECT
  DATE(timestamp) AS run_date,

  COUNT(*) AS total_events,

  COUNTIF(resource.action = 'applied')
    AS successful_runs,

  COUNTIF(resource.action = 'errored')
    AS failed_runs,

  COUNTIF(resource.action = 'canceled')
    AS canceled_runs,

  ROUND(
    SAFE_DIVIDE(
      COUNTIF(resource.action = 'errored'),
      COUNT(*)
    ) * 100,
    2
  ) AS failure_rate_percent

FROM `tfc_logs.audit_logs`

WHERE resource.type = 'run'

GROUP BY run_date

ORDER BY run_date;
```

---

# 7. Create a dashboard

For a simple GCP setup, use **Looker Studio** with BigQuery as the data source.

Your dashboard could look like:

```text
┌──────────────────────────────────────────────────────┐
│              TFC WORKSPACE ANALYTICS                 │
├──────────────┬──────────────┬──────────────┬─────────┤
│ Total Runs   │ Successful   │ Failed       │ Failure │
│    100       │     10       │     10       │   10%   │
└──────────────┴──────────────┴──────────────┴─────────┘

┌─────────────────────────┐  ┌─────────────────────────┐
│ Run Status              │  │ Runs by Workspace       │
│                         │  │                         │
│ Applied       █████     │  │ prod-infra     ███████  │
│ Errored       █████     │  │ dev-infra      █████    │
│ Planning      █████     │  │ staging        ████     │
│ Canceled      █████     │  │ networking     ███      │
└─────────────────────────┘  └─────────────────────────┘

┌──────────────────────────────────────────────────────┐
│              Workspace Failure Rate                  │
│                                                      │
│ prod-infrastructure       ████████ 20%              │
│ dev-infrastructure        ████      10%              │
│ staging-infrastructure    ██         5%              │
└──────────────────────────────────────────────────────┘
```

### Recommended dashboard components

| Metric                 | Visualization   |
| ---------------------- | --------------- |
| Total runs             | Scorecard       |
| Successful runs        | Scorecard       |
| Failed runs            | Scorecard       |
| Failure rate           | Scorecard       |
| Run status             | Pie/Donut chart |
| Runs by workspace      | Bar chart       |
| Runs by organization   | Bar chart       |
| Runs over time         | Time-series     |
| Failed runs            | Table           |
| User activity          | Table           |
| Workspace failure rate | Bar chart       |

---

## 8. Most important analytics for TFC

For a real TFC environment, I'd focus on these:

**Deployment health**

* Total runs
* Successful runs
* Failed runs
* Canceled runs
* Failure %
* Success %

**Workspace health**

* Most active workspaces
* Failed workspaces
* Workspace failure rate
* Last run status

**User activity**

* Runs by user
* Failed runs by user
* Most active users

**Operational trends**

* Runs per day
* Failures per day
* Failure trend
* Workspace state distribution

**Security/audit**

* Who triggered a run
* Which workspace was affected
* When it happened
* IP/request information if available
* Unusual activity

### One important improvement

Your current sample has **one audit record per run state**, so `run-abc123` isn't represented through its complete lifecycle.

For realistic analytics, I'd generate data like:

```text
run-001
  ├── run.create
  ├── run.plan
  ├── run.planned
  ├── run.apply
  └── run.complete

run-002
  ├── run.create
  ├── run.plan
  └── run.errored

run-003
  ├── run.create
  ├── run.plan
  ├── run.planned
  └── run.canceled
```

That allows you to calculate **run success rate, lifecycle duration, failed deployment trends, workspace health, and user activity** much more accurately.
 