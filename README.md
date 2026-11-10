# AWS Data Pipeline for Clickstream Analysis

An end-to-end clickstream analytics pipeline on AWS. Apache access logs from a café website are shipped to Amazon CloudWatch Logs by the CloudWatch agent, analyzed with CloudWatch Logs Insights, visualized in a CloudWatch dashboard, and archived in Amazon S3.

**Scenario:** AnyCompany Café wants to understand how visitors use its website (menu views, purchases, visitor locations) to decide where to advertise and where to open new locations.

## Architecture

```
Visitor -> Apache httpd (Cloud9 EC2 instance, IAM role CafeRole)
        -> CloudWatch agent
        -> CloudWatch Logs (log group: apache/access)
        -> CloudWatch Logs Insights (queries)
        -> CloudWatch dashboard (cafe-dashboard)
        -> Amazon S3 (log archive)
```

![Architecture diagram](screenshots/01-architecture-diagram.png)

| Component | Purpose |
|---|---|
| EC2 (Cloud9 instance) | Hosts the café site (Apache, PHP, MariaDB) and the CloudWatch agent |
| IAM role `CafeRole` | Lets the instance send logs and metrics to CloudWatch |
| CloudWatch Logs | Stores `apache/access` and `apache/error` (180-day retention) |
| Logs Insights | Ad hoc queries on the JSON access log |
| CloudWatch dashboard | `cafe-dashboard` with 4 widgets for the café owner |
| Amazon S3 | Archive of the geolocation log, queryable later (for example with Athena) |

## Cost estimate

> TODO: add the AWS Pricing Calculator summary (us-east-1, 12 months) and the link or export.

![Cost estimate](screenshots/02-cost-estimate.png)

## Phase 2: Website and access logs

- Opened inbound TCP port 80 in the instance's security group.
- Loaded the site at `http://<public-ip>/cafe` and watched entries arrive with `tail -f`.
- Backed up the original log to `initial_access_log`.

![Security group](screenshots/03-security-group.png)
![Café website](screenshots/04-cafe-website.png)
![Tail access log](screenshots/05-tail-access-log.png)

## Phase 3: CloudWatch agent and JSON logging

- Installed `amazon-cloudwatch-agent` and downloaded `config.json`.
- Edited `httpd.conf` so access and error logs are written as JSON:
  - new `ErrorLog` and `ErrorLogFormat` lines
  - new JSON `LogFormat ... cloudwatch`
  - new `CustomLog "/var/log/www/access/access_log" cloudwatch`
  - commented out the `logio_module` block
- Created `/var/log/www/access` and `/var/log/www/error`, restarted `httpd`, and started the agent.

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json -s
```

![config.json](screenshots/07-agent-config-json.png)
![httpd.conf changes](screenshots/08-httpd-conf-changes.png)
![configtest and logio block](screenshots/09-configtest-and-logio.png)
![Agent status](screenshots/10-agent-status-running.png)

## Phase 4: Verifying ingestion

The agent log shows both files being piped to CloudWatch, and the `apache/access` and `apache/error` log groups were created.

![Agent log piping](screenshots/11-agent-log-piping.png)
![Log groups](screenshots/12-log-groups.png)
![JSON log entry](screenshots/13-json-log-entry.png)
![processOrder entry](screenshots/14-processorder-entry.png)

## Phase 5: Simulated traffic

Replaced `access_log` with the provided `samplelogs/access_log.log` (187,051 JSON lines) to simulate real traffic, then restarted the agent.

![Sample log](screenshots/15-sample-log-head-wc.png)
![Stream with same timestamps](screenshots/16-stream-same-timestamps.png)

## Phase 6: Funnel analysis with Logs Insights

Queries ran on the data from `access_log.log`. They are saved in the `non-geo-results` folder.

**Menu visitors** (`menu-visitors`):
```
fields @timestamp, remoteIP
| filter request = "/cafe/menu.php"
| count(remoteIP) as Visitors by @timestamp
| sort @timestamp asc
```

**Purchasers** (`purchasers`):
```
fields @timestamp, remoteIP
| filter request = "/cafe/processOrder.php"
| count(remoteIP) as Purchasers by @timestamp
| sort @timestamp asc
```

| Metric | Result |
|---|---|
| Menu page visits (records matched) | 49,270 |
| Purchases (records matched) | 36,898 |
| Menu visits without a purchase | 12,372 |
| Purchase rate | 74.89% |

Menu without purchase = 49,270 - 36,898. Purchase rate = 36,898 / 49,270 x 100.

![Menu visitors query](screenshots/17-menu-visitors-query.png)
![Purchasers query](screenshots/18-purchasers-query.png)
![Saved queries](screenshots/19-saved-queries.png)

## Phase 7: Geolocation, dashboard and S3

Replaced the log with `samplelogs/access_log_geo.log` (5,428 lines), which adds `country`, `region`, `city`, `lat` and `lon` to each entry. Deleted and recreated the `apache/access` log group so the new format was ingested cleanly.

Top cities by events (custom range Nov 9-10):

| City | Events |
|---|---|
| London | 238 |
| Whitehall | 197 |
| Beijing | 106 |
| New York | 102 |
| Chicago | 99 |

![City query](screenshots/23-city-query.png)

### Dashboard: `cafe-dashboard`

| Widget | Type |
|---|---|
| Cities visiting the menu the most | Pie |
| Cities placing the most orders | Logs table |
| Regions visiting the website the most | Pie |
| Regions placing the most orders | Bar |

```
# Cities visiting the menu the most
fields remoteIP, city
| filter request = "/cafe/menu.php"
| filter ispresent(city)
| stats count() as menupopular by city
| sort menupopular desc
| limit 10

# Cities placing the most orders
fields remoteIP, city
| filter request = "/cafe/processOrder.php"
| filter ispresent(city)
| stats count() as ordercount by city
| sort ordercount desc
| limit 10

# Regions visiting the website the most
fields remoteIP, region
| filter request = "/cafe"
| filter ispresent(region)
| stats count() as regionpopular by region
| sort regionpopular desc
| limit 10

# Regions placing the most orders
fields remoteIP, region
| filter request = "/cafe/processOrder.php"
| filter ispresent(region)
| stats count() as orderregion by region
| sort orderregion desc
| limit 10
```

![Dashboard](screenshots/24-dashboard-full.png)

### S3 archive

```bash
aws s3 cp ~/environment/samplelogs/access_log_geo.log s3://accap4-logsbucket--d5bf94a0/access_log_geo.log
aws s3 ls s3://accap4-logsbucket--d5bf94a0/
```

The archived object is 2,544,435 bytes.

![S3 upload](screenshots/26-s3-upload.png)

## Challenges and fixes

| Problem | Cause | Fix |
|---|---|---|
| No log groups appeared in CloudWatch | The agent ran as `cwagent` and got `Permission denied` on the log file | Set `"run_as_user": "root"` in `config.json` and reloaded the agent |
| JSON and plain-text (CLF) lines were mixed in one file | `/var/log/www/access/access_log` was a symlink to `/var/log/httpd/access_log`, so both `CustomLog` lines wrote to the same file | Replaced the symlink with a regular file and restarted `httpd` |
| `httpd.conf` edits had no effect | The edits were never saved to the file Apache reads | Re-applied them and checked with `grep` and `apachectl configtest` |
| Agent resumed from a stale position after the log file was replaced | The agent kept its saved offset from the old file | Stopped the agent, cleared its state, deleted the log group and restarted |
| Dashboard showed only one city | The time range did not cover the ingestion time | Set a custom range covering the load |
| Extra non-geo lines in the live log | Internet scanners hit the open port 80 | Filtered with `ispresent(city)` and archived the clean sample file to S3 |
| `log-gen_geo.py` produced `lat`/`lon` of `None` | The lab blocks outbound geolocation lookups (HTTP 403) | Used the provided `access_log_geo.log` |

## Repository structure

```
.
├── cw-agent/config.json        # CloudWatch agent configuration
├── httpd-conf-snippet.txt      # JSON log format changes for httpd.conf
├── samplelogs/                 # Sample logs provided by the lab
├── queries/                    # Saved Logs Insights and dashboard queries
├── screenshots/                # Evidence for each phase
├── phase6-results.txt          # Funnel analysis results
└── README.md
```

## Notes

- Built in the AWS Academy lab environment (us-east-1).
- Account IDs and user names are blurred in the screenshots.
