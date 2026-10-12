# teamspeak

TeamSpeak 6 server of bulbashenko.com (`ts.bulbashenko.com`). This repo is the Kubernetes base; [bulbashenko-com/infra](https://github.com/bulbashenko-com/infra) deploys it, pinned to one commit of this repo (`clusters/hel1/teamspeak.yaml`).

## What runs

| | |
|---|---|
| Image | `teamspeaksystems/teamspeak6-server` (tag in `deploy/k8s/teamspeak.yaml`) |
| Node | `node.bulbashenko.com/role=edge` (hel1-edge): `ts.bulbashenko.com` is a DNS-only A record to that node's IP |
| Ports | voice `9987/udp`, file transfer `30033/tcp` (host ports); Web Query `10080` inside the pod network only, guests off |
| Data | PVC `teamspeak-data` (local-path on the edge node), never pruned by Flux |

## Provided by infra

- Namespace `teamspeak`.
- Secret `teamspeak` (SOPS): `TSSERVER_QUERY_ADMIN_API_KEY`, `RESTORE_S3_ACCESS_KEY_ID`, `RESTORE_S3_SECRET_ACCESS_KEY`.

## Runbook

**Upgrade.** Change the image tag here (PR). After it merges, CI opens the deploy PR `deploy/teamspeak` in infra; merging that rolls it out. TS6 is a beta: read the release notes, the server migrates its database on start.

**Admin.** The server password and the first ServerAdmin privilege key are kept by the owner (outside git). Web Query, from a machine with cluster access:
```
kubectl -n teamspeak port-forward deploy/teamspeak 10080:10080
curl -H "x-api-key: <TSSERVER_QUERY_ADMIN_API_KEY>" "http://127.0.0.1:10080/1/serverinfo"
```

**Restore.** On start, the `restore` init container fills an *empty* volume from `restore/teamspeak.tar.gz` in the cluster's Garage (a tar of the volume contents) and refuses to start on an empty volume without it. To restore: put the archive there, delete the PVC's data (or the PVC), restart the Deployment.

**Move to another node.** The data is on the node's disk: back up the volume (tar), restore on the new node as above, and point `ts.bulbashenko.com` at the new node's IP.
