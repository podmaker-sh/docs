# Region Failover Runbook

**Audience:** On-call SRE  
**Severity:** P1 — region-down triggers automatic alerting  
**RTO:** 15 min (NATS + Vault) / 30 min (full control-plane recovery)  
**RPO:** 0 for Vault secrets (Raft log), ≤5 min for regional event stream (NATS FileStorage replay)

---

## 1. Symptoms

| Signal | Meaning |
|--------|---------|
| NATS leaf connection drop alert | Regional edge node offline |
| Vault `health` endpoint returns 503 on primary | Primary unsealed but unreachable |
| `regional_databases.status = 'degraded'` | Regional Postgres unreachable |
| Orchestrator activities timing out (`StartToCloseTimeout`) | Agent gateway or build service down in region |

---

## 2. Triage

```bash
# 1. Confirm which region is down
curl -s https://control-plane.podmaker.io/api/internal/regions/health \
  -H "Authorization: Bearer $INTERNAL_TOKEN" | jq .

# 2. Check NATS hub connectivity
nats --server nats://hub.podmaker.io:4222 server report jetstream

# 3. Check Vault cluster
bao status --address https://vault.podmaker.io:8200

# 4. Check regional DB
psql "$REGIONAL_DSN" -c "SELECT 1" 2>&1
```

---

## 3. Vault Failover (Primary → Performance Standby)

> **Warning:** This promotes the standby to active primary. Requires Vault operator root token or recovery key quorum.

```bash
# Step 1 — verify standby is healthy and caught up
bao status --address https://vault-standby.podmaker.io:8200
# Expect: Initialized=true, Sealed=false, HA Enabled=true

# Step 2 — step down primary (if reachable; otherwise skip to Step 3)
bao operator step-down --address https://vault.podmaker.io:8200

# Step 3 — confirm new leader
bao status --address https://vault-standby.podmaker.io:8200 | grep "HA Mode"
# Expect: HA Mode: active

# Step 4 — update vault-broker env (VAULT_ADDR) in control-plane .env
# and restart vault-broker service
systemctl restart podmaker-vault-broker

# Step 5 — verify broker health
curl -sf http://localhost:8300/healthz && echo OK
```

---

## 4. NATS Leaf Node Failover

NATS leaf nodes reconnect automatically to the hub. No manual action needed unless the hub itself is down.

```bash
# Verify hub is healthy
nats --server nats://hub.podmaker.io:4222 server info

# If hub is down — promote secondary hub (if configured):
# Update leaf.conf `url` in affected edge nodes and reload NATS
nats-server --signal reload

# Drain inflight messages from down leaf (replay from FileStorage)
nats --server nats://hub.podmaker.io:4222 stream get HM-AGENTS-us-east --last
```

---

## 5. Control Plane Recovery

```bash
# Mark region degraded (prevents new provision dispatches)
php artisan regions:set-status us-east degraded

# Reroute pending orchestrator workflows to available region
# (Temporal search attribute: Region=us-east, Status=Running)
tctl workflow list \
  --query 'WorkflowType="ProvisionServerWorkflow" AND ExecutionStatus="Running"' \
  | while read -r wf; do
      tctl workflow terminate --workflow-id "$wf" --reason "region-failover"
  done

# Re-enqueue in eu-central (control plane API)
php artisan regions:requeue-workflows us-east eu-central

# Restore region when edge is back
php artisan regions:set-status us-east active
```

---

## 6. Regional Database Failover

```bash
# 1. Check replica lag before promoting
psql "$READ_REPLICA_DSN" -c "SELECT now() - pg_last_xact_replay_timestamp() AS lag;"

# 2. Promote read replica
psql "$READ_REPLICA_DSN" -c "SELECT pg_promote();"

# 3. Update regional_databases record
UPDATE regional_databases
SET db_dsn = '$NEW_PRIMARY_DSN',
    read_replica_dsn = NULL,
    status = 'active',
    metadata = jsonb_set(metadata, '{failover_at}', to_jsonb(now()::text))
WHERE region_id = 'us-east';

# 4. Clear Laravel DB connection cache
php artisan cache:clear
php artisan config:clear
```

---

## 7. Verification Checklist

Run after recovery:

- [ ] `nats server report jetstream` — all streams healthy, no lag
- [ ] `bao status` — active node reachable, unsealed
- [ ] `GET /api/internal/regions/health` — region `status: active`
- [ ] Trigger a test provision workflow in the recovered region
- [ ] Confirm workflow completes within SLA (15 min)
- [ ] Check `audit_logs` for auth failures during failover window
- [ ] Update incident ticket with timeline and RCA

---

## 8. Post-Incident

1. Run chaos drill again within 2 weeks to validate fix.
2. Update `region_assignments` in affected workspace records if routing changed.
3. Rotate any tokens that may have been exposed during outage.
4. Add monitoring alert if not already present for the failure mode.

---

## 9. Contacts

| Role | Contact |
|------|---------|
| On-call SRE | PagerDuty escalation policy `podmaker-sre` |
| Vault operator | `#vault-ops` Slack |
| Cloud provider incidents | Hetzner status page |
