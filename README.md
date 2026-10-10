# k3s Dolt

Dolt SQL Server for beads server mode.

## Deploy

Create the local password file used by Kustomize. Do not commit this file.

```bash
mkdir -p /var/Projects/k3s-Dolt/secrets
openssl rand -hex 32 > /var/Projects/k3s-Dolt/secrets/beads_dolt_password.txt
chmod 600 /var/Projects/k3s-Dolt/secrets/beads_dolt_password.txt
```

```bash
kubectl apply -k /var/Projects/k3s-Dolt
```

Create the Dolt SQL user from inside the Dolt pod. Dolt initializes the root user as `root@localhost`, so first-time user creation must run locally in the Dolt container rather than from an external bootstrap pod.

```bash
PW="$(tr -d '\n' < /var/Projects/k3s-Dolt/secrets/beads_dolt_password.txt)"
kubectl exec deployment/dolt -- env BEADS_DOLT_PASSWORD="$PW" /bin/sh -c 'dolt --data-dir /var/lib/dolt sql -q "CREATE USER IF NOT EXISTS '\''beads'\''@'\''%'\'' IDENTIFIED BY '\''${BEADS_DOLT_PASSWORD}'\''; GRANT ALL PRIVILEGES ON *.* TO '\''beads'\''@'\''%'\''; FLUSH PRIVILEGES;"'
unset PW
```

The preferred internal endpoint for clients outside the cluster is
`dolt.styxut.net:3307`, through the private pfSense HAProxy TCP listener.
In-cluster clients should use `dolt-service.default.svc:3307`. The direct TCP
NodePort `30307` remains available as a diagnostic/fallback on the current
Dolt pod node:

- `192.168.50.221:30307` (`k3s-worker-01`)
- `192.168.50.223:30307` (`k3s-worker-03`)

The Service uses `externalTrafficPolicy: Local`, so only a worker with a local
Dolt endpoint will accept the NodePort connection. HAProxy performs TCP
health checks against both eligible workers and follows the ready local
endpoint. Its SQL transport is not TLS-encrypted. The pod is restricted to
iSCSI-capable nodes and explicitly excludes `k3s-worker-02`, the internet-egress
worker. Its data claim is statically bound to the existing Synology LUN named
`k8s-csi-pvc-338b8bc1-abe2-48f7-a4e9-a9913bfcfe5b`; the PV reclaim policy is
`Retain`.

## Beads Client Configuration

For a local project using beads server mode:

```bash
export BEADS_DOLT_SERVER_HOST=dolt.styxut.net
export BEADS_DOLT_SERVER_PORT=3307
export BEADS_DOLT_SERVER_USER=beads
export BEADS_DOLT_PASSWORD="$(cat /var/Projects/k3s-Dolt/secrets/beads_dolt_password.txt)"
bd init --server
bd dolt test
```

Use the direct NodePort only for troubleshooting or when the HAProxy endpoint
is unavailable; target the worker currently hosting the Dolt pod.

## Notes

- The deployment uses one replica and `Recreate` so only one Dolt server process mounts the iSCSI volume.
- `dolt.styxut.net:3307` is the internal HAProxy endpoint; the raw SQL connection is not TLS-encrypted.
- For direct fallback, use `192.168.50.221:30307` (worker-01) or `192.168.50.223:30307` (worker-03), whichever currently hosts the pod; worker-02 is excluded from scheduling.
- Data is stored on the `synology-dolt` PVC using `synology-iscsi-storage`.
- The first-time user creation command connects locally inside the Dolt pod because the initial root account is `root@localhost`.
- Use the documented `openssl rand -hex 32` command so the bootstrap SQL can safely embed the generated password.
