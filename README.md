# 🛡️ DNS Threat Intelligence — Automated Pi-hole Security Pipeline

An automated, closed-loop threat intelligence pipeline that monitors your **Pi-hole** DNS traffic, enriches domains through **VirusTotal** and **AlienVault OTX**, autonomously blocks malicious actors, and visualizes everything in a real-time **Grafana** dashboard.

> Built with n8n · Pi-hole · InfluxDB · Grafana · VirusTotal · AlienVault OTX

![Grafana Dashboard](grafana_dashboard_image.PNG)

---

## 📖 How It Works

Every 5 minutes, the pipeline wakes up and runs through a fully automated sequence:

```
Pi-hole Logs → Deduplicate Domains → Cache Check (InfluxDB)
    → AlienVault OTX Scan → VirusTotal Scan
        → Block if Malicious (Pi-hole API)
            → Store Results (InfluxDB) → Visualize (Grafana)
```

1. **Extract** — n8n fetches the last 5 minutes of DNS queries from Pi-hole.
2. **Deduplicate** — Domains are cleaned, normalized, and deduplicated.
3. **Cache Check** — Already-scanned domains are skipped to preserve API quota.
4. **Enrich** — New domains are cross-referenced against AlienVault OTX (pulse activity) and VirusTotal (malicious engine detections).
5. **Act** — Domains with **≥ 5 malicious detections** are immediately added to Pi-hole's blocklist.
6. **Log** — All results (clean or malicious) are written to InfluxDB.
7. **Visualize** — Grafana renders live stats: threat rate, severity distribution, top offenders, and OTX pulse activity.

---

## 🛠️ Prerequisites

Before you begin, make sure you have the following running and accessible on your network:

| Service | Notes |
| :--- | :--- |
| **Pi-hole** | API access must be enabled (v5+ or FTL API) |
| **n8n** | Self-hosted — v1.x recommended |
| **InfluxDB 2.x** | Used as the time-series store |
| **Grafana** | Connected to your InfluxDB instance |

**API Keys required (both have free tiers):**
- [VirusTotal](https://www.virustotal.com/) — domain reputation (4 req/min on free tier)
- [AlienVault OTX](https://otx.alienvault.com/) — threat pulse intelligence

---

## 📦 Installation

### 1. n8n Workflow

![n8n Workflow Canvas](n8n_workflow_image.PNG)

1. Open your n8n instance and create a **New Workflow**.
2. Copy the contents of `n8n_workflow.json` and paste it directly onto the n8n canvas (or use **Import from JSON**).
3. Replace all `[YOUR_*]` placeholders in the HTTP Request nodes — see the [Configuration Variables](#️-configuration-variables) table below.
4. **Activate** the workflow. It will run automatically every 5 minutes via the Schedule Trigger.

> **Note:** The VirusTotal node uses a 16-second batch interval to respect the free tier rate limit of ~4 requests/minute. Upgrade your API plan to increase throughput.

---

### 2. InfluxDB Setup

1. Log into your InfluxDB UI and navigate to **Buckets**.
2. Create a new bucket — e.g., `AI` or `DNS_Security`.
3. Go to **API Tokens** and generate a token with **Read + Write** access to your bucket.
4. Note your **Organization name** — you'll need it for the workflow and dashboard.

---

### 3. Grafana Dashboard

1. In Grafana, go to **Dashboards → New → Import**.
2. Upload `grafana_dashboard.json` or paste the JSON directly.
3. Before importing, **find and replace** `[YOUR_BUCKET_NAME]` in the JSON with your actual InfluxDB bucket name — or update the Flux queries in each panel manually after import.
4. Set your InfluxDB datasource as the data source for all panels.

---

## ⚙️ Configuration Variables

Replace every placeholder below in both JSON files before deploying. Remove the surrounding `[ ]` brackets.

| Placeholder | Description | Where to find it |
| :--- | :--- | :--- |
| `[YOUR_PIHOLE_IP]` | Local IP address of your Pi-hole | Run `hostname -I` on your Pi-hole host |
| `[YOUR_PIHOLE_AUTH_TOKEN]` | Pi-hole Web API password/token | Pi-hole Settings → API / Web Interface |
| `[YOUR_INFLUXDB_IP]` | Local IP where InfluxDB 2.x is hosted | Your server or container's IP |
| `[YOUR_INFLUX_TOKEN]` | InfluxDB API token (Read + Write) | InfluxDB UI → API Tokens |
| `[YOUR_ORG_NAME]` | Your InfluxDB organization name | InfluxDB UI → Settings → Organization |
| `[YOUR_BUCKET_NAME]` | The InfluxDB bucket for storing results | InfluxDB UI → Buckets |
| `[YOUR_VIRUSTOTAL_API_KEY]` | Your VirusTotal API key | [VT Account Settings](https://www.virustotal.com/gui/my-apikey) |
| `[YOUR_ALIENVAULT_API_KEY]` | Your AlienVault OTX API key | [OTX Settings → API Key](https://otx.alienvault.com/api) |

---

## 📊 Grafana Dashboard Panels

The dashboard includes 9 panels covering different facets of your DNS threat landscape:

| Panel | Type | Description |
| :--- | :--- | :--- |
| 🔍 Domains Scanned | Stat | Total domains analyzed in the selected time range |
| 🚫 Threats Blocked | Stat | Count of domains added to Pi-hole's blocklist |
| ⚠️ Threat Rate | Gauge | Percentage of scanned domains flagged as threats |
| 🦠 Max VT Score | Stat | Highest VirusTotal malicious detection count seen |
| 📈 Blocked vs. Scanned | Time series | Blocked threats overlaid against total scan volume |
| 🎯 Top 10 Malicious Domains | Bar chart | Highest-scoring domains by VirusTotal detections |
| 🔴 Severity Distribution | Donut chart | Breakdown by severity tier: Clean / Low / Medium / High / Critical |
| 📡 OTX Pulse Activity | Time series | AlienVault pulse spikes indicating coordinated campaigns |
| 📋 Threat Intelligence Log | Table | Full searchable log of all scanned domains with scores |

---

## 🧠 Threat Scoring Logic

A domain is **blocked** when its VirusTotal malicious detection count reaches **5 or more**. This threshold balances sensitivity against false positives — adjust it in the `If` node in n8n to suit your environment.

| VT Detections | Severity | Action |
| :--- | :--- | :--- |
| 0 | ✅ Clean | Logged only |
| 1–5 | 🟡 Low | Logged only |
| 6–15 | 🟠 Medium | Logged + Blocked |
| 16–30 | 🔴 High | Logged + Blocked |
| 30+ | 🔥 Critical | Logged + Blocked |

---

## 🔧 Troubleshooting

**Workflow runs but no data appears in Grafana**
- Confirm the InfluxDB bucket name matches exactly in both n8n and the Grafana dashboard queries.
- Check that the InfluxDB API token has Write access to the correct bucket.

**VirusTotal scan always errors**
- Free tier is limited to ~4 requests/minute. The 16-second batch interval handles this, but if you're processing many unique domains you may hit daily limits (500 req/day on free).

**Domains are not being blocked in Pi-hole**
- Verify your `[YOUR_PIHOLE_AUTH_TOKEN]` is correct. For Pi-hole v6+, this is a session token obtained from the `/api/auth` endpoint, not the password hash.
- Ensure the `X-Pihole-Sudo: true` header is present in the Block Domain node.

**`Is Cached?` node always routes to re-scan**
- The cache check looks for `_result` in the InfluxDB response body. If your bucket is empty or the query syntax is wrong, all domains will be re-scanned. This is safe but may exhaust API quota quickly on first run.

---

## 🤝 Contributing

Contributions are welcome! If you have ideas for new threat intelligence sources, alerting integrations (e.g., Telegram/Slack notifications), or dashboard improvements, feel free to open an issue or submit a pull request.

---

## 📄 License

[MIT](https://choosealicense.com/licenses/mit/) — free to use, modify, and distribute.
