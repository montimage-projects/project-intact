# K8s Deployment Notes

This folder contains Kubernetes manifests in `manifest/`, which are copied from the folder `ubitech/k8s/manifest/`, so there may be some environment-specific configurations needed.
Before installing and testing, review the environment-specific values below.

## 1. Check list before installation

### 1.1 Cluster networking prerequisites
- Multus CNI must be installed (NetworkAttachmentDefinition support).
- `macvlan` CNI plugin must be available on worker nodes.
- The host interface in `manifest/monitoring-vlan.yml` is currently `eth0`.
  - Update it if your nodes use a different NIC name.
- Static IP in `manifest/monitoring-vlan.yml` is currently `10.100.50.249/29`.
  - Ensure this address/subnet is valid and free in your infrastructure.

### 1.2 Node pinning (hostname)
Several workloads are pinned to one node using:
- `kubernetes.io/hostname: nsit-intact-w0`

This appears in:
- `manifest/5greplay.yml`
- `manifest/kafka.yml`
- `manifest/mongodb.yml`
- `manifest/mmt-operator.yml`
- `manifest/mmt-probe.yml`

Update this value if your target node hostname is different.

### 1.3 Storage path and permissions
- MongoDB uses hostPath storage at `/tmp/.mmt-database` in `manifest/mongodb.yml`.
- Verify:
  - The path exists or can be created.
  - Node filesystem permissions are sufficient.
  - You accept hostPath behavior (node-local and non-portable).

### 1.4 Service exposure and port conflicts
- `mmt-operator` uses NodePort `30010` in `manifest/mmt-operator.yml`.
- Ensure this NodePort is allowed and not already in use.

### 1.5 Images and pull access
Verify cluster nodes can pull:
- `ghcr.io/montimage/5greplay:latest`
- `ghcr.io/montimage/mmt-operator:v1.7.7`
- `ghcr.io/montimage/mmt-probe:v1.6.1`
- `ubuntu/zookeeper:edge`
- `ubuntu/kafka:edge`
- `mongo:8`

## 2. Suggested install order
Apply manifests in this order:
1. `manifest/monitoring-vlan.yml`
2. `manifest/mongodb.yml`
3. `manifest/kafka.yml`
4. `manifest/mmt-operator.yml`
5. `manifest/mmt-probe.yml`
6. `manifest/5greplay.yml`

Reasons:
- Network attachment first.
- Data and messaging dependencies before consumers/producers.

## 3. Post-install validation

### 3.1 Pods and scheduling
- Check all pods are Running.
- Confirm pods are scheduled on expected node if nodeSelector is used.

### 3.2 Secondary network attachment
- Verify `mmt-probe` and `5greplay` got the `monitor-macvlan-net` attachment.
- Confirm assigned IP matches your expected subnet plan.

### 3.3 Kafka/Mongo connectivity
- `mmt-operator` should resolve and connect to `kafka` and `mmt-database` services.
- `mmt-probe` should publish to Kafka topic `mmt-reports`.

### 3.4 Functional smoke test
- Send traffic to `mmt` service on port `5000`.
- Confirm logs/events are produced and consumed end-to-end.
