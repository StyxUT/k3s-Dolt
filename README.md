# k3s Dolt

Dolt SQL Server for beads server mode.

## Deploy

Create the local password file used by Kustomize. Do not commit this file.

```bash
mkdir -p /home/styxut/Projects/k3s-Dolt/secrets
openssl rand -hex 32 > /home/styxut/Projects/k3s-Dolt/secrets/beads_dolt_password.txt
chmod 600 /home/styxut/Projects/k3s-Dolt/secrets/beads_dolt_password.txt
```

```bash
kubectl apply -k /home/styxut/Projects/k3s-Dolt
```

Create the Dolt SQL user from inside the Dolt pod. Dolt initializes the root user as `root@localhost`, so first-time user creation must run locally in the Dolt container rather than from an external bootstrap pod.

```bash
PW="$(tr -d '\n' < /home/styxut/Projects/k3s-Dolt/secrets/beads_dolt_password.txt)"
kubectl exec deployment/dolt -- env BEADS_DOLT_PASSWORD="$PW" /bin/sh -c 'dolt --data-dir /var/lib/dolt sql -q "CREATE USER IF NOT EXISTS '\''beads'\''@'\''%'\'' IDENTIFIED BY '\''${BEADS_DOLT_PASSWORD}'\''; GRANT ALL PRIVILEGES ON *.* TO '\''beads'\''@'\''%'\''; FLUSH PRIVILEGES;"'
unset PW
```

The service is reachable on the LAN at `k3s-nodes.home:3307` and through the configured k3s external IPs:

- `192.168.0.211:3307`
- `192.168.0.221:3307`
- `192.168.0.222:3307`
- `192.168.0.223:3307`
- `192.168.0.224:3307`

## Beads Client Configuration

For a local project using beads server mode:

```bash
export BEADS_DOLT_SERVER_HOST=k3s-nodes.home
export BEADS_DOLT_SERVER_PORT=3307
export BEADS_DOLT_SERVER_USER=beads
export BEADS_DOLT_PASSWORD="$(cat /home/styxut/Projects/k3s-Dolt/secrets/beads_dolt_password.txt)"
bd init --server
bd dolt test
```

Use `k3s-nodes.home` or any reachable external IP from the service list.

## Notes

- The deployment uses one replica and `Recreate` so only one Dolt server process mounts the iSCSI volume.
- Data is stored on the `synology-dolt` PVC using `synology-iscsi-storage`.
- The first-time user creation command connects locally inside the Dolt pod because the initial root account is `root@localhost`.
- Use the documented `openssl rand -hex 32` command so the bootstrap SQL can safely embed the generated password.
