# K8s Deployment Notes (Portable)

This folder contains manifests in `manifest/` copied from `ubitech/k8s/manifest/`.
To make adaptation easier, each manifest includes inline `CHECK` comments before lines that should be reviewed per infrastructure.

## 1. Choose deployment profile first

### 1.1 Profile A: Single-node or no dedicated VLAN network
- Do not apply `manifest/monitoring-vlan.yml`.
- In `manifest/5greplay.yml` and `manifest/mmt-probe.yml`, comment/remove the `k8s.v1.cni.cncf.io/networks` annotation.
- Keep only default Kubernetes pod networking.

### 1.2 Profile B: Multi-node or dedicated monitoring network
- Apply `manifest/monitoring-vlan.yml`.
- Ensure Multus and macvlan are installed.
- Update interface and static IP/subnet in `manifest/monitoring-vlan.yml` for your environment.

### 1.3 Important note for multi-node deployments
- Multi-node does not automatically require VLAN/macvlan.
- If default Kubernetes CNI networking is sufficient for your connectivity and test scope, use Profile A even on multi-node clusters.
- In that case, do not apply `manifest/monitoring-vlan.yml` and comment/remove Multus annotations in `manifest/5greplay.yml` and `manifest/mmt-probe.yml`.

## 2. Files that must be reviewed before install

### 2.1 `manifest/5greplay.yml`
- Optional Multus network annotation.
- Node selector hostname.

### 2.2 `manifest/kafka.yml`
- Node selector hostname (zookeeper and kafka deployments).
- Kafka `advertised.listeners` if service exposure differs.

### 2.3 `manifest/mmt-operator.yml`
- Node selector hostname.
- NodePort value (`30010`) and cluster policy.

### 2.4 `manifest/mmt-probe.yml`
- Optional Multus network annotation.
- Node selector hostname.

### 2.5 `manifest/mongodb.yml`
- hostPath location (`/tmp/.mmt-database`).
- Node selector hostname.

## 3. Minimal post-install checks

1. All pods are Running.
2. Workloads are scheduled on expected nodes (if nodeSelector remains enabled).
3. `mmt-operator` reaches `kafka` and `mmt-database`.
4. `mmt-probe` publishes to Kafka topic `mmt-reports`.
5. If Profile B is used, confirm Multus/macvlan attachment is present on `mmt-probe` and `5greplay`.

## 4. Usage (traffic replay)

After deployment is healthy, generate replay traffic from inside the 5greplay pod.

### 4.1 Enter the 5greplay pod
1. Get pods and identify the 5greplay pod:
	 - `kubectl get pods`
2. Open a shell in the pod:
	 - `kubectl exec -it <5greplay-pod-name> -- /bin/bash`

### 4.2 Run replay command
Example command (adapt values to your environment):

```bash
./5greplay replay \
	-t pcap/oai.pcap \
	-Xforward.target-protocols=SCTP \
	-Xforward.target-ports=38412 \
	-Xforward.target-hosts=<amf_ip_or_hostname> \
	-Xforward.nb-copies=1 \
	-Xforward.default=DROP \
	-Xforward.bind-ip=<source_ip_reachable_to_target>
```

### 4.3 Parameters to pay attention to
- `-Xforward.target-hosts`: destination host(s) receiving replayed packets (for example AMF IP/DNS). Use a reachable endpoint from the 5greplay pod network.
- `-Xforward.bind-ip`: source IP that 5greplay binds to when opening sockets. This IP must exist on an interface available in the pod network context.
	- Profile A (no VLAN/Multus): usually do not force this unless required; default routing is often enough.
	- Profile B (with VLAN/Multus): typically set this to the secondary network IP assigned for replay traffic.

### 4.4 Results

These screenshorts of the test performed on June 19, 2025 in UBITECH's Kubernetes environment targeting the AMF component:

- execution log of 5Greplay

<img src=img/5greplay.png>

- AMF log

<img src=img/amf-log.png>

- Security alerts

<img src=img/security-alerts.png>

