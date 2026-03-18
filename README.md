# 🔧 Apigee Production Runbooks

> Real-world incident reports, RCAs, and operational runbooks from Apigee Edge OPDK production environments.
> Maintained by **Rakesh P** — Apigee Administrator | API Platform Engineer

---

## 📁 Incident Index

| # | Date | Severity | Component | Title | Status |
|---|------|----------|-----------|-------|--------|
| 001 | 2026-03-18 | 🟡 Medium | Message Processor | [Apigee Proxy Undeploy Failure — UnknownEventReceived](#-incident-001--apigee-proxy-undeploy-failure) | ✅ Resolved |

---

## 🚨 Incident 001 — Apigee Proxy Undeploy Failure

### `messaging.runtime.UnknownEventReceived` on DC2 Message Processor

---

### 📋 Incident Summary

| Field | Details |
|-------|---------|
| **Incident ID** | INC-001 |
| **Date & Time** | March 18, 2026 — 13:52:18 IST |
| **Environment** | Production — `org/prod` |
| **Affected Proxy** | `getCase-Passthrough-v1` — Revision 1 |
| **Severity** | 🟡 Medium |
| **Duration** | ~10 minutes |
| **Impact** | Deployment pipeline blocked — No live traffic impact |
| **Resolved By** | Apigee Administrator |

---

### ⏱️ Timeline

```
13:52:07  Analytics stats query executed successfully (PostgreSQL, 75ms)
13:52:18  Undeploy (DELETE) triggered for getCase-Passthrough-v1 rev 1
13:52:18  ZooKeeper: spec.xml and _deploymentBean paths deleted successfully
13:52:19  ✅ Undeploy successful on all 19 Routers
13:52:19  ❌ ERROR: DC2 Message Processor — UnknownEventReceived
13:52:19  ✅ Undeploy successful on 7 of 8 Message Processors
13:52:19  ⚠️  ZooKeeper success-node statuses rolled back (partial failure)
13:52:19  ❌ Overall undeploy marked FAILED by Management Server
~14:00    Admin identified failing MP UUID → mapped to DC2 node
~14:02    MP restarted → undeploy retried → completed successfully ✅
```

---

### 🔍 Error from Logs

```log
ERROR DISTRIBUTION - HTTPWaiter.await() : Exception java.util.concurrent.ExecutionException
  com.apigee.events.EventHandlingFailureException{
    code    = events.EventHandlingFailure,
    message = Remote server failed to handle event.
    error code: messaging.runtime.UnknownEventReceived
    error message: Received an unknown event with description
      DELETE Application /organizations/<org>/apiproxies/getCase-Passthrough-v1/revisions/1/
  }

ERROR DISTRIBUTION - RemoteServicesUnDeploymentHandler.unDeployFromServers()
  Undeploy failed from message-processor <mp-uuid>
  cause = Remote server failed to handle event.
  error code: messaging.runtime.UnknownEventReceived
  communication error = false
```

---

### 🧩 Root Cause

The Message Processor on DC2 (Disaster Recovery datacenter) had its **internal event-handling thread stuck in a degraded state**.

- The JVM process was alive and responding to health checks on port `8082`
- Internal Netty/event-loop threads responsible for processing deployment lifecycle events were deadlocked
- The MP appeared healthy externally (`isUp: true`, `reachable: true`) but silently rejected all deployment events
- All other 7 MPs in the pod processed the undeploy successfully — only the DC2 MP failed

> **Key Insight:** In Apigee OPDK, even 1 MP failure out of 8 causes the entire undeploy to be marked FAILED and ZooKeeper status nodes are rolled back.

---

### 💥 Impact Assessment

| Area | Impact |
|------|--------|
| Live API Traffic | ✅ No impact |
| Other Proxies | ✅ No impact |
| Deployment Pipeline | ❌ Blocked ~10 minutes |
| ZooKeeper | ⚠️ Temporary stale status nodes (auto-cleaned post-restart) |
| Data / Transactions | ✅ None |

---

### 🛠️ Resolution Steps

**Step 1 — Identify the failing MP UUID from Management Server logs**
```
Look for: messaging.runtime.UnknownEventReceived
Extract the MP UUID from the error line in ms-log
```

**Step 2 — Map UUID to hostname using Management API**
```bash
curl -s -u <admin-user>:<password> \
  "http://<management-server>:8080/v1/servers?pod=<pod-name>&region=<region>&type=message-processor"
```

**Step 3 — Confirm UUID on the suspect node**
```bash
curl -s http://<mp-host>:8082/v1/servers/self | grep -i uuid
```

**Step 4 — Restart the Message Processor**
```bash
ssh root@<mp-host>
/opt/apigee/apigee-service/bin/apigee-service edge-message-processor restart
```

**Step 5 — Retry the undeploy**
```bash
curl -X DELETE \
  -u <admin-user>:<password> \
  "http://<management-server>:8080/v1/organizations/<org>/environments/<env>/apiproxies/<proxy-name>/revisions/<rev>/deployments"
```

✅ Undeploy completed successfully after restart.

---

### 🔒 Preventive Actions

| # | Action | Target Date |
|---|--------|-------------|
| 1 | Add MP health check validating event-handling via `/v1/servers/self` — not just process status | Week 1 |
| 2 | Configure monitoring alert for MP uptime > 7 days — schedule rolling restarts in maintenance window | Week 2 |
| 3 | Ensure DC2/DR MPs are included in monitoring dashboards equally with DC1 | Week 1 |
| 4 | Document runbook: Recovery steps for `UnknownEventReceived` MP failures | Week 3 |

---

### 💡 Lessons Learned

- **Process running ≠ Service healthy** — always validate via the MP management API (`/v1/servers/self`), not just `apigee-service status`
- **Multi-DC OPDK** — DC2/DR MPs must be included in all monitoring and health-check routines equally with DC1
- **Apigee all-or-nothing behavior** — a single MP failure blocks the entire deploy/undeploy operation; MP health monitoring is critical
- **UUID-to-IP mapping** — the fastest way to identify a failing physical node from log UUIDs is via `/v1/servers` Management API

---

### 🏷️ Tags

`apigee` `apigee-opdk` `message-processor` `undeploy-failure` `UnknownEventReceived` `production-incident` `rca` `apigee-edge` `rhel` `zookeeper`

---

## 🧰 Environment Reference

| Component | Details |
|-----------|---------|
| Platform | Apigee Edge OPDK (Private Cloud) |
| OS | RHEL 8 |
| Deployment | Multi-datacenter (DC1 Primary + DC2 DR) |
| Components | Management Server, Router, Message Processor, ZooKeeper, Cassandra, PostgreSQL |

---

## 👤 Author

**Rakesh P**
Apigee Administrator | API Platform Engineer
[GitHub](https://github.com/rakeshp-Devops)

> ⚠️ *All internal IPs, hostnames, org names, credentials, and environment-specific details have been masked or replaced with placeholders before publishing. This repository is for knowledge sharing and professional reference only.*

---

*Last updated: March 18, 2026*
