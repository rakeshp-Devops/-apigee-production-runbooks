# Apigee Production Runbooks

> Sanitized runbooks from real production incidents on Apigee Edge OPDK (Private Cloud).
> All sensitive IPs, hostnames, and credentials have been replaced with placeholders.

---

## 📋 Runbook Index

| # | Incident | Severity | Components |
|---|---|---|---|
| 01 | [504 Gateway Timeout — Cassandra PoolTimeoutException](#01-504-gateway-timeout) | P1 | Apigee MP, Cassandra |
| 02 | [Stuck Message Processor Thread](#02-stuck-mp-thread) | P1 | Apigee MP |
| 03 | [PostgreSQL Disk Space Exhaustion](#03-postgresql-disk-space) | P2 | PostgreSQL |
| 04 | [ZooKeeper Health Degradation](#04-zookeeper-health) | P2 | ZooKeeper, Apigee |
| 05 | [Proxy Undeploy Failure](#05-proxy-undeploy-failure) | P2 | Apigee MP |

---

## 01 — 504 Gateway Timeout

**Root Cause:** Cassandra PoolTimeoutException causing Apigee Message Processor to time out on policy execution.

### Symptoms
- API consumers receiving `504 Gateway Timeout`
- Kibana showing spike in `fault.name = messaging.adaptors.http.flow.RequestTimeout`
- `nodetool tpstats` showing dropped messages on `READ` and `MUTATION` stages

### Diagnosis Steps

```bash
# Check Apigee MP logs
tail -f /opt/apigee/var/log/edge-message-processor/logs/system.log | grep -i "cassandra\|timeout\|pool"

# Check Cassandra dropped messages
nodetool tpstats | grep -E "Dropped|READ|MUTATION|WRITE"

# Check Cassandra heap
nodetool info | grep -i heap

# Check pending compactions
nodetool compactionstats

# Check GC pause times
grep "GCInspector" /var/log/cassandra/system.log | tail -20
```

### Resolution Steps

```bash
# 1. Identify the overloaded Cassandra node
nodetool status

# 2. Flush memtables to reduce pending writes
nodetool flush

# 3. If heap is high (>75%), trigger GC
nodetool gcstats

# 4. Restart Cassandra on the affected node (last resort)
/opt/apigee/apigee-service/bin/apigee-service apigee-cassandra restart

# 5. Verify recovery
nodetool tpstats
nodetool status
```

### Prevention
- Set up cron-based tpstats monitoring (see: linux-monitoring-scripts repo)
- Alert when dropped messages > 0 for READ or MUTATION stages
- Monitor heap usage — alert at 70%, page at 80%

---

## 02 — Stuck MP Thread

**Root Cause:** A Message Processor thread stuck in WAITING state, preventing proxy undeployment.

### Symptoms
- Proxy undeploy returns success but proxy remains active
- MP logs show UUID thread in WAITING state
- `curl` to MP management API returns stale deployment info

### Diagnosis Steps

```bash
# Check MP deployment status
curl -u admin:PASSWORD http://MP-HOST:8082/v1/runtime/organizations/ORG/environments/ENV/apis/PROXY/revisions

# Get thread dump
curl http://MP-HOST:8082/v1/threads > /tmp/threaddump-$(date +%s).txt

# Search for stuck thread by proxy name
grep -A 20 "PROXY_NAME" /tmp/threaddump-*.txt
```

### Resolution Steps

```bash
# 1. Identify stuck thread UUID from thread dump
# Look for: state: WAITING, proxy: YOUR_PROXY_NAME

# 2. Restart the specific MP (if single-node issue)
/opt/apigee/apigee-service/bin/apigee-service edge-message-processor restart

# 3. Verify proxy is undeployed after restart
curl -u admin:PASSWORD http://MP-HOST:8082/v1/runtime/organizations/ORG/environments/ENV/apis/PROXY/revisions

# 4. Redeploy proxy from Edge UI or management API
```

---

## 03 — PostgreSQL Disk Space Exhaustion

**Root Cause:** `log_statement = 'all'` enabled in PostgreSQL config causing rapid log growth.

### Symptoms
- Disk usage on PostgreSQL server >85%
- Apigee analytics data not appearing in Edge UI
- PostgreSQL service slow or unresponsive

### Diagnosis Steps

```bash
# Check disk usage
df -h /

# Find largest files
du -sh /var/lib/postgresql/*/main/pg_log/* | sort -rh | head -10

# Check current log_statement setting
sudo -u postgres psql -c "SHOW log_statement;"
```

### Resolution Steps

```bash
# 1. Truncate the large log file (DO NOT delete — file may be open)
sudo truncate -s 0 /var/log/postgresql/postgresql-*.log

# 2. Disable log_statement in postgresql.conf
sudo sed -i "s/^log_statement = 'all'/log_statement = 'none'/" /etc/postgresql/*/main/postgresql.conf

# 3. Reload PostgreSQL config
sudo -u postgres psql -c "SELECT pg_reload_conf();"

# 4. Verify disk freed
df -h /

# 5. Verify setting applied
sudo -u postgres psql -c "SHOW log_statement;"
```

### Prevention
- Never enable `log_statement = 'all'` in production
- Set up disk usage alerts at 70% and 80%
- Configure logrotate for PostgreSQL logs

---

## 04 — ZooKeeper Health Degradation

**Root Cause:** ZooKeeper node losing quorum or high latency causing Apigee cluster instability.

### Diagnosis Steps

```bash
# Check ZooKeeper status on each node
echo "ruok" | nc ZK-HOST 2181
echo "stat" | nc ZK-HOST 2181 | grep -E "Mode|Connections|Outstanding"
echo "mntr" | nc ZK-HOST 2181 | grep -E "zk_avg_latency|zk_outstanding_requests|zk_pending_syncs"

# Check ZooKeeper process
/opt/apigee/apigee-service/bin/apigee-service apigee-zookeeper status
```

### Resolution Steps

```bash
# Restart ZooKeeper on the degraded node
/opt/apigee/apigee-service/bin/apigee-service apigee-zookeeper restart

# Verify quorum restored
echo "stat" | nc ZK-HOST 2181 | grep Mode
# Expected: Mode: follower (or leader on leader node)
```

---

## 05 — Proxy Undeploy Failure

**Root Cause:** Proxy stuck in deployed state due to MP thread or ZooKeeper sync issue.

### Resolution Steps

```bash
# Force undeploy via management API
curl -X DELETE \
  -u admin:PASSWORD \
  "http://MS-HOST:8080/v1/organizations/ORG/environments/ENV/apis/PROXY/revisions/REV/deployments"

# If above fails, restart MP on affected DC
/opt/apigee/apigee-service/bin/apigee-service edge-message-processor restart
```

---

## 🏗️ Environment Reference

| Component | Count | Notes |
|---|---|---|
| Gateway (Router + MP) | 4 nodes | Combined deployment |
| MicroGateway | 4 nodes | Separate cluster |
| Cassandra / ZooKeeper | 3 nodes | Co-located |
| PostgreSQL | 1 node | Analytics |

---

## 📁 Folder Structure

```
apigee-production-runbooks/
├── README.md                  # This file — runbook index
├── runbooks/
│   ├── 01-cassandra-pool-timeout.md
│   ├── 02-stuck-mp-thread.md
│   ├── 03-postgresql-disk-space.md
│   ├── 04-zookeeper-health.md
│   └── 05-proxy-undeploy-failure.md
├── scripts/
│   └── health-check-all.sh    # Runs all health checks
└── templates/
    └── rca-template.md        # RCA document template
```

---

*Maintained by Rakesh Gowda — Apigee Infrastructure Engineer*
