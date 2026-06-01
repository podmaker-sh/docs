# Vault Bridge CA — backup + restore runbook

The Bridge CA lives in the platform meta-vault. Lose its root key
and **every agent cert in the field becomes invalid** — operators
have to re-bootstrap each bridge, which means downtime for every
self-hosted vault that routes through one.

Treat the meta-vault root the same way you treat a TLS root CA in
prod: offline backup, geographically separated copies, restore
drill once a quarter.

## What lives where

| Object                                                  | Path inside the meta-vault           |
|---------------------------------------------------------|--------------------------------------|
| Active root cert + private key                          | `meta/bridge-ca/root`                |
| Archived roots (post-rotation)                          | `meta/bridge-ca/archive`             |
| Per-workspace active intermediate (cert + private key)  | `meta/bridge-ca/workspaces/<ws>/intermediate` |
| Per-workspace archived intermediates                    | `meta/bridge-ca/workspaces/<ws>/archive`      |

The actual KV values are JSON-encoded `{cert, key, created_at, …}`
maps. The private keys are NOT separately encrypted — meta-vault
seal is the encryption boundary.

## Daily backup

Run from a host with network reach to the meta-vault endpoint and
write access to the offline backup bucket. The backup file is the
raw KV snapshot — meta-vault seals it on import.

```bash
# /opt/podmaker/bin/bridge-ca-backup.sh
set -euo pipefail
BACKUP_DIR=${BACKUP_DIR:-/var/backups/podmaker/bridge-ca}
STAMP=$(date -u +%Y%m%dT%H%M%SZ)
mkdir -p "$BACKUP_DIR"

# Dump every meta/bridge-ca/* path through the OpenBao API
bao kv get -mount=secret -format=json meta/bridge-ca/root \
    > "$BACKUP_DIR/root-${STAMP}.json"
bao kv list -mount=secret -format=json meta/bridge-ca/workspaces \
    | jq -r '.[]' | while read -r ws; do
    bao kv get -mount=secret -format=json "meta/bridge-ca/workspaces/${ws}intermediate" \
        > "$BACKUP_DIR/ws-${ws%/}-${STAMP}.json"
done

gpg --encrypt --recipient backups@podmaker.sh \
    --output "$BACKUP_DIR/bridge-ca-${STAMP}.tar.gz.gpg" \
    "$BACKUP_DIR"/*-"$STAMP".json
aws s3 cp "$BACKUP_DIR/bridge-ca-${STAMP}.tar.gz.gpg" \
    "s3://podmaker-coldstorage/bridge-ca/"
```

Schedule via cron at 03:00 UTC. Keep 30 days of snapshots on S3 +
weekly snapshots on a separate filer in another region.

## Restore drill

Twice a year, full end-to-end:

1. Spin up a throwaway meta-vault cluster on staging.
2. Decrypt the most recent backup with the team's GPG key (4-eyes
   rule — at least two custodians present).
3. Re-import each JSON blob with `bao kv put -mount=secret <path> @file.json`.
4. Verify `php artisan tinker --execute='app(\App\Services\Vault\Bridge\BridgeCA::class)->caFingerprint();'`
   matches the fingerprint stamped on the latest live agent cert.
5. Wipe the staging cluster.

## Disaster — meta-vault is gone

If both the live meta-vault and the latest backup are unrecoverable:

1. Mint a brand-new root via `rotateRoot()` in the Filament admin.
2. Force every workspace to rotate via `rotateIntermediate(workspaceId)`.
3. Trigger a fleet-wide cert rotation: instruct each customer to
   restart the agent (`systemctl restart podmaker-vault-bridge`)
   so it fetches a fresh leaf against the new chain.
4. Issue a status-page incident — the downtime equals the longest
   bridge restart cycle in the fleet.

## Restore from a single backup file (single workspace)

```bash
bao kv put -mount=secret meta/bridge-ca/workspaces/<ws>/intermediate \
    @backup-ws-<ws>-<stamp>.json
```

`pruneArchive()` will later remove the entry once the validity
period of the restored intermediate lapses.
