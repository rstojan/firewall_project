# firewall_project

Firewall project to test use cases.

## Palo Alto Firewall Log Generator - Dynatrace Workflow

A Dynatrace workflow that generates sample Palo Alto Networks PAN-OS traffic log files for testing whether firewall rules are correctly blocking or allowing traffic.

### Use Case

Validates that a Palo Alto firewall is partially blocking traffic as expected. The workflow generates a realistic mix of **allowed (~60%) and blocked (~40%)** traffic entries, covering:

- Multiple internal subnets (trust, DMZ) and external networks (untrust)
- 15 different application types (web-browsing, ssl, dns, ssh, rdp, ftp, sql, etc.)
- Named allow rules (`Allow-Outbound-Web`, `Allow-DNS`, etc.) and deny rules (`Deny-Inbound-RDP`, `Deny-SQL-External`, etc.)
- Three block actions: `deny`, `drop`, `reset-both`
- Realistic session metadata (bytes, packets, elapsed time, session end reasons)
- Higher-risk applications (RDP, unknown-tcp on port 4444) are always blocked

### Log Format

Generated logs follow the **PAN-OS 10.x CSV syslog traffic log format** with ~103 positional fields per line, matching the field order documented in:
- [PAN-OS 10.1 Traffic Log Fields](https://docs.paloaltonetworks.com/pan-os/10-1/pan-os-admin/monitoring/use-syslog-for-monitoring/syslog-field-descriptions/traffic-log-fields)

### Workflow Structure

| Task | Order | Description |
|------|-------|-------------|
| `generate_palo_alto_logs` | 1 | JavaScript action that creates 50 sample traffic log lines with a configurable mix of allow/deny actions |
| `ingest_logs_to_grail` | 2 | JavaScript action that sends each log line to the Dynatrace Log Ingest API v2 with structured `paloalto.*` attributes for easy DQL querying |
| `analyze_blocked_traffic` | 3 | JavaScript action that parses the generated logs and produces a validation report (blocked by rule, by app, by zone pair, top blocked destinations) |

### Files

| File | Format | Purpose |
|------|--------|---------|
| `workflows/palo_alto_log_generator.yaml` | YAML template | Import via Dynatrace Workflows App UI |
| `workflows/palo_alto_log_generator_api.json` | JSON API format | Import via Dynatrace Automation API |
| `queries/palo_alto_firewall_logs.dql` | DQL | 15 ready-to-use queries for the Logs app, Notebooks, or Dashboards |

### How to Import

**Via Dynatrace UI (YAML):**
1. Open your Dynatrace tenant
2. Navigate to **Automations > Workflows**
3. Click **Upload** and select `palo_alto_log_generator.yaml`
4. Run the workflow manually (trigger is set to Manual)

**Via API (JSON):**
```bash
curl -X POST "https://{your-tenant}.apps.dynatrace.com/platform/automation/v1/workflows" \
  -H "Authorization: Bearer {your-api-token}" \
  -H "Content-Type: application/json" \
  -d @workflows/palo_alto_log_generator_api.json
```

### Configuration

Edit these constants at the top of the `generate_palo_alto_logs` JavaScript action:

| Parameter | Default | Description |
|-----------|---------|-------------|
| `LOG_COUNT` | 50 | Number of log entries to generate |
| `BLOCK_RATIO` | 0.40 | Fraction of traffic that will be blocked (~40%) |
| `SERIAL_NUMBER` | 001801000001 | Simulated device serial |
| `DEVICE_NAME` | PA-5260-LAB | Simulated device hostname |

### DQL Queries

After the workflow runs, the logs are stored in Grail with structured `paloalto.*` attributes. Use the queries in `queries/palo_alto_firewall_logs.dql` to visualize the data. Key queries:

**Quick validation — is traffic being blocked?**
```
fetch logs
| filter log.source == "palo-alto-firewall"
| fieldsAdd is_blocked = paloalto.action != "allow"
| summarize total = count(),
            allowed = countIf(NOT is_blocked),
            blocked = countIf(is_blocked)
| fieldsAdd allowed_pct = round(toDouble(allowed) / toDouble(total) * 100, decimals: 1),
            blocked_pct = round(toDouble(blocked) / toDouble(total) * 100, decimals: 1)
```

**Blocked traffic by firewall rule:**
```
fetch logs
| filter log.source == "palo-alto-firewall"
| filter paloalto.action != "allow"
| summarize block_count = count(),
            unique_sources = countDistinct(paloalto.src),
            unique_apps = countDistinct(paloalto.app),
            by: { paloalto.rule }
| sort block_count desc
```

**Time series for dashboards:**
```
fetch logs
| filter log.source == "palo-alto-firewall"
| fieldsAdd is_blocked = paloalto.action != "allow"
| makeTimeseries allowed = countIf(NOT is_blocked),
                 blocked = countIf(is_blocked),
                 interval: 5m
```

See `queries/palo_alto_firewall_logs.dql` for all 15 queries including: blocked by application, zone pair analysis, high-risk port blocks, bandwidth analysis, top blocked sources/destinations, session end reasons, and raw CSV parsing fallback.

### Sample Output

The `analyze_blocked_traffic` task returns a validation report:

```json
{
  "totalEntries": 50,
  "allowedCount": 28,
  "blockedCount": 22,
  "blockPercentage": "44.0%",
  "validationResult": "PASS - Both allowed and blocked traffic detected",
  "blockedByRule": {
    "Deny-Inbound-RDP": { "count": 5, "apps": ["rdp", "unknown-tcp"] },
    "Deny-SQL-External": { "count": 4, "apps": ["ms-sql-server", "mysql"] },
    "Deny-All-Default": { "count": 8, "apps": ["ftp", "ssh", "smtp"] }
  }
}
```
