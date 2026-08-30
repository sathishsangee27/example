# VSI Issue — Must-Gather Guide (IPOPS)

This document defines the baseline data that IPOPS must collect whenever a Virtual Server Instance (VSI) has any issue — regardless of the symptom. Collect everything in this guide first, then follow the symptom-specific runbook.

**Covered issue types:**
- VSI stuck in `starting` / `stopping` / `deleting`
- VSI crashed or stopped unexpectedly
- VSI fails to provision
- VSI fails to power on (including encrypted volumes)
- VSI console not accessible
- VSI scheduling failure (capacity / profile / SGX)

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Step 1 — Identify the VSI](#step-1--identify-the-vsi)
- [Step 2 — Control-Plane Health](#step-2--control-plane-health)
- [Step 3 — kube-vminfo Snapshot](#step-3--kube-vminfo-snapshot)
- [Step 4 — vmstuck & vmtree (Stuck VSIs)](#step-4--vmstuck--vmtree-stuck-vsis)
- [Step 5 — IBM Cloud Logs (ICL)](#step-5--ibm-cloud-logs-icl)
- [Step 6 — Hypervisor Host Evidence](#step-6--hypervisor-host-evidence)
- [Step 7 — Blast Radius & Change Correlation](#step-7--blast-radius--change-correlation)
- [Symptom-Specific Additions](#symptom-specific-additions)
- [Escalation Guide](#escalation-guide)

---

## Prerequisites

| Requirement | Notes |
|-------------|-------|
| IBM Cloud Logs (ICL) access | For the affected region |
| `kubectl` access to the RIAS IKS cluster | For the region |
| Deployer access | `/opt/compute/bin/kube-vminfo`, `vmstuck`, `vmtree` |
| Genctl Kubernetes read access | In the affected zone namespace |
| *(Optional)* Serial/console access | Via customer portal or ops tooling |
| *(Hypervisor steps only)* Compute SRE (COM) zone access | Required for Phase 6 |

---

## Step 1 — Identify the VSI

Collect these details before running any commands.

- [ ] **VSI ID** — format: `<zone-prefix>_<UUID>`, e.g., `02f7_cfc063e2-3bbb-4db4-85f6-ed95c4ecf55f`
- [ ] **Account ID / namespace** — 32-character hex string, e.g., `28b6b731d8494187865b8721696d3d8a`
- [ ] **Region and zone** — e.g., `us-south`, `us-south-2`
- [ ] **Observed symptom and current VSI state** — e.g., stuck in `starting` for 20 min
- [ ] **Timestamp (UTC) when the issue was first noticed**
- [ ] **VSI profile** — e.g., `bx2-4x16`, `cx2-2x4`
- [ ] **OS type** — RHEL / Ubuntu / Windows / custom image
- [ ] **ServiceNow check** — search for any CR or customer-initiated action (stop/reboot/delete) within ±30 min of the event
  - If a matching CR exists → likely platform-initiated; note the CR number and continue gathering

---

## Step 2 — Control-Plane Health

### 2.1 Deployment health

Always check deployment health first — a degraded deployment explains many VSI issues.

```bash
kubectl get deployment -n rias regional-compute
kubectl get deployment -n rias instance-spec-controller
```

Expected: all pods `READY` and `AVAILABLE` (e.g., `3/3`). If not, describe the deployment for details:

```bash
kubectl describe deployment -n rias regional-compute
kubectl describe deployment -n rias instance-spec-controller
```

Check for recent pod restarts:

```bash
kubectl get pods -n rias --sort-by='.status.containerStatuses[0].restartCount'
```

For regional-health or dedicated compute alerts:

```bash
kubectl get deployments -n rias -l app=regional-health
kubectl get deployments -n rias -l app=regional-dedicated-compute
```

### 2.2 InstanceSpec & InstanceStatus

```bash
kubectl get instancespec <vsi-id> -n <namespace> -o yaml
```

Note:
- `status.state` — expected `running`; note if `starting`, `stopping`, `stopped`, or `deleting`
- `status.shouldRun` — `true` = platform wants it running; `false` = platform wants it stopped
- `status.deletionTimestamp` — present means a delete has been requested
- `status.powerState`
- Finalizers on the spec: `compute.rias.ibm.com/volume-cleanup`, `reservedip-cleanup`, `vni-cleanup`, `deleted-event`

```bash
kubectl get instancestatus <vsi-id> -n <account-namespace> -o yaml
```

Note:
- `state`, `reason`, `message`
- `shouldRun`, `deletionTimestamp`, `DeletionStarted`
- Finalizer: `compute.rias.ibm.com/eventbillingstatus-cleanup`
- Timestamps of last state transitions

### 2.3 VirtualMachine object

```bash
kubectl get virtualmachine <vsi-id> -n <namespace> -o yaml
```

Note:
- `status.phase` / `status.powerState`
- `status.nodeName` — the hypervisor host (needed for Step 6)
- Any `conditions` with `status: False`
- `metadata.deletionTimestamp`

### 2.4 ComputeDomain & CapacityPoolVMAttachment

```bash
kubectl get computedomain -n <namespace> -o yaml
```

Note: `status.powerState`, any `deletionTimestamp`, and presence of finalizers.

Check for unexpected detachment or eviction events on `CapacityPoolVMAttachment`.

---

## Step 3 — kube-vminfo Snapshot

Run on the deployer for the affected zone. This is the single most important diagnostic tool.

```bash
/opt/compute/bin/kube-vminfo -n <namespace> -v <vsi-id>
```

Capture the full output. Key fields:

| Field | What to record |
|-------|---------------|
| `hvState` | Hypervisor state: `RUNNING`, `STOPPED`, `CRASHED`, `STOPPING` |
| `vmStatus` | Control-plane state: `RUNNING`, `STARTING`, `STOPPING`, `STOPPED`, `DELETING` |
| `shouldRun` | Platform intent |
| `schedulable` / `runnable` | Readiness flags — `False` means something is blocking |
| `COMPUTE NODE / IPADDR` | Hypervisor hostname and IP — needed for Step 6 |
| `Pod` | The `compute-agent` pod name on that host |
| `(Waiting for …)` lines | Shows which resource is blocking (cloud-init disk, vdisk, vnic, virtualBwg) |
| `cloud-init` `ready` | `True` / `False` |
| `vdisk` entries | `volumeProvisioned`, `readyToAttach`, `attached` for each disk |
| `vnic` entries | `attached` status for each NIC |
| `State change history` | Recent state transitions with timestamps |

---

## Step 4 — vmstuck & vmtree (Stuck VSIs)

Run these when a VSI is stuck in `starting`, `stopping`, or `deleting`.

### vmstuck — why is the VM not in the desired state?

```bash
/opt/compute/bin/vmstuck -v <vsi-id> -n <namespace>
```

Record the full output. The tool identifies the blocking cause, e.g.:
- `SNAPSHOT computeAction in progress`
- `bandwidthgroup is pending deletion`
- `NEI is pending deletion`
- `OS License Activation failed`
- `no vdisk found for spec`

> **Do NOT delete a SNAPSHOT computeAction** without following the stuck-stopping runbook.

### vmtree — full resource dependency tree

```bash
/opt/compute/bin/vmtree -n <namespace> -v <vsi-id> 
```

Review to determine if the root cause is in vdisk, vnic, or compute.

---

## Step 5 — IBM Cloud Logs (ICL)

> **Time window:** issue timestamp ± 30 minutes for all log searches.

### 5.1 Control-plane logs

| Component | Search for | Patterns to look for |
|-----------|-----------|---------------------|
| `instance-spec-controller` | VSI ID | `unexpected stop`, `shouldRun=false`, `powerStateChange`, `reconcileError`, `Publishing instance deleted event` |
| `regional-compute` | VSI ID or request ID | `vm state change`, `STOPPED`, `CRASHED`, `evicted`, `host failure`, `Unable to power on instance` |
| `vm-instance-controller` | VSI ID or VM name | `domain destroyed`, `libvirt error`, `qemu exit`, `watchdog`, `VirtualNic is still attached`, `ComputeDomain not shut down` |
| `regional-health` | `ERROR` entries | Provisioning failures, API degradation |

### 5.2 Infrastructure logs

| Component | Search for | Patterns to look for |
|-----------|-----------|---------------------|
| `compute-agent` | Hypervisor host name (from Step 3) AND VSI ID | `domain crash`, `guest panic`, `hw error`, `node eviction`, `OOM kill`, `kernel panic received`, `Timed out during operation: cannot acquire state change lock` |
| DNS / CoreDNS | Crash timestamp window | `failed to resolve hostname`, `no such host`, `dns timeout`, `lookup failed` |
| ETCD | Crash timestamp window | `grpc: addrConn.createTransport failed`, `etcd connection failed`, `etcd timeout` |
| HAProxy / Cilium | Crash timestamp window | `cilium pod restart`, `cni error`, `haproxy os update`, `node reboot` |

> **Why DNS/ETCD/HAProxy matter:** HAProxy node OS updates → Cilium restarts → DNS failures → compute-agent loses control-plane connectivity → VSIs appear to crash or get stuck. This is a known cascading failure chain.

### 5.3 Storage-layer logs

| Component | Search for | Patterns to look for |
|-----------|-----------|---------------------|
| `volume-controller` / `storage-backend-controller` | VSI's volume IDs (from `kube-vminfo`) | `volume detach`, `storage backend unavailable`, `block agent restart`, `CrashLoopBackOff` |
| `block-agent` / `storage-agent` | Volume IDs | Crash-loop or mount failure events |
| `regional-storage` | Volume ID / encryption key ID | `encryption_key_deleted`, `Volume encryption key state is not active` |
| `keylore` | Encryption key ID (from `volumespec`) | `Keylore` errors, key unwrap failures |

### 5.4 Network-layer logs

| Component | Search for | Patterns to look for |
|-----------|-----------|---------------------|
| `network-controller` / `vni-controller` | VSI's NIC or ReservedIP ID | `endpoint deleted`, `NIC detached unexpectedly`, `reservedIP released` |
| `fabcon-manager` | VirtualNetworkInterface ID | `found dangling macvtap`, `attempt to delete in-use interface` |
| `regional-network-mock` | Request ID | Any failures related to vnic attachment |

---

## Step 6 — Hypervisor Host Evidence

> **Access required:** Compute SRE (COM) or equivalent zone-level access.
> Only proceed if `kube-vminfo` returned a `COMPUTE NODE` value.

- [ ] **6.1** Confirm the hypervisor hostname from `kube-vminfo` COMPUTE NODE field (Step 3)

- [ ] **6.2** Check the host system journal around the event time:
  ```bash
  sudo journalctl -u libvirtd --since "<event-time - 10min>" --until "<event-time + 10min>" | grep -E "(error|fail|warn)" | tail -20
  ```
  Patterns: `domain crash`, `guest panicked`, `watchdog timeout`, `qemu: terminating on signal`, `End of file from qemu monitor`, `Read-only file system`

- [ ] **6.3** Check QEMU/KVM log for the specific VM:
  ```
  /var/log/libvirt/qemu/<virsh domain name>.log
  ```
  Look for: `qemu: hardware error`, `CPU reset`, `NMI`, `MCE (Machine Check Exception)`

- [ ] **6.4** Check QEMU process state (for stuck-stopping):
  ```bash
  virsh list                          # get domain ID
  
  ```
  A timeout error like `cannot acquire state change lock (held by monitor=remoteDispatchDomainSnapshotCreateXML)` means a QEMU lock is held.

- [ ] **6.5** Check host `dmesg` for hardware faults:
  ```bash
  dmesg | grep -E "mce|hardware error|oom|killed process|memory error"
  ```
  - MCE entries → hardware memory/CPU fault
  - OOM killer entries → host memory pressure evicted the VM process

- [ ] **6.6** Determine if the hypervisor host rebooted near the event time:
  ```bash
  last reboot
  who -b
  ```

---

## Symptom-Specific Additions

After completing Steps 1–6, collect these additional items based on the observed symptom.

### VSI stuck in `starting`

- [ ] From `kube-vminfo`: note which `(Waiting for …)` lines are present — cloud-init, vdisk, vnic, or virtualBwg
- [ ] Check the relevant controller logs:
  - vnic issue → RNOS logs: search `regional-network-mock` + request ID
  - vdisk issue → RCOS logs: search `regional-compute` + request ID
  - compute scheduling → search `vm-instance-controller` + VSI ID for scheduling errors

### VSI stuck in `stopping` or `deleting`

- [ ] Run `vmstuck` (Step 4) and record the exact blocking reason
- [ ] Check `InstanceSpec` `shouldRun` (should be `false`) and `deletionTimestamp`
- [ ] Identify which finalizers remain on `InstanceSpec` and `InstanceStatus`
- [ ] If a SNAPSHOT computeAction is in progress: do **not** delete it; follow the [Instance Stuck Stopping runbook](Instance_Stuck_Stopping.md)
- [ ] If `bandwidthgroup` or `NetworkInterface` (NEI) is pending deletion: follow the [Networking Issues with VSI Not Stopping or Deleting runbook](genctl-network-vsi-stopping-deletion-issues.md)
- [ ] Check `virsh list` on the hypervisor for a still-running libvirt domain

### VSI crashed / stopped unexpectedly

- [ ] Verify no CR or customer-initiated stop/reboot within ±30 min (Step 1)
- [ ] Check `shouldRun` on `InstanceSpec` — if `false` set by controller, it was a platform-triggered stop
- [ ] Collect serial/console log if available; look for kernel panic, OOM killer output, NMI watchdog
- [ ] Check `osActivation.phase` for RHEL instances — unexpected `deactivated` can indicate a platform stop misread as a crash
- [ ] Check if `kdump`/`kexec` was enabled and a vmcore was captured (`/var/crash/`)

### VSI fails to provision (stuck in `starting` from creation)

- [ ] Verify image is valid and the `MinimumProvisionedSizeGB` is ≥ 10 GB
- [ ] Check `kube-vminfo` `(Waiting for …)` lines — storage and network readiness
- [ ] Check boot volume `capacity` is between 10 GB and 250 GB
- [ ] If provisioning from a volume: `GET /v1/volumes/<volume-id>` — verify `operating_system` is present and `capacity` is in range

### VSI fails to power on (encrypted volume)

- [ ] Collect the encryption key ID from `kubectl get volumespec <instance-id> -n <namespace> -o yaml` (field: `spec.encryptionKeyCrn`)
- [ ] Search ICL logs for `app:regional-compute <instance-id> "Unable to power on instance. Volume encryption key state is not active"`
- [ ] Search ICL logs for `app:regional-storage <key-id>` — look for `status_reasons.code: encryption_key_deleted`
- [ ] Search ICL logs for `app:keylore <key-id>` — look for Keylore microservice errors (key unwrap failures)
- [ ] If the key was deleted: direct the customer to [IBM Key Protect key restoration steps](https://cloud.ibm.com/docs/key-protect?topic=key-protect-restore-keys)

### VSI console not accessible

- [ ] Check `kubectl get secrets -A | grep console-<vsi-id>` — verify the console secret exists and is not expired
- [ ] Decode console state: `kubectl get secret <console-id> -n <account-id> -o yaml | base64 -d | jq .`
- [ ] Check `kubectl get pods -A | grep console-proxy` — verify the console-proxy pod is running (not `Init:0/2`)
- [ ] Check `kubectl -n genctl get pods | grep compute-agent` — verify the compute-agent on the VM's host is `Running` and `READY` (2/2)
- [ ] Check for two compute domains: `kubectl get computedomains -n <account-id> -l InstanceID=<vsi-id>` — if two exist, the stale one must be patched
- [ ] Verify Activity Tracker shows both `create console-access-token` and `read instance` events within 3 minutes

### VSI scheduling failure (no suitable host / capacity)

- [ ] Check `vm-instance-controller` logs for scheduling failure messages
- [ ] Check ICL `capacity_alert` or `RCOS_insufficient_running_pods` alerts for the zone
- [ ] For SGX / Secure Boot VSIs: verify the node annotation `IntelSGX=true` and `SecureBootCapable` are present on available nodes
- [ ] Check zone capacity in ServiceNow or the capacity dashboard

---

## Escalation Guide

| Symptom / Root Cause | Team | Channel |
|----------------------|------|---------|
| VSI stuck in starting (vnic) | RNOS (Network SRE) | PagerDuty: Network SRE |
| VSI stuck in starting (vdisk) | RSOS (Storage SRE) | PagerDuty: Storage SRE |
| VSI stuck in starting (compute) | RCOS / Compute SRE |  PagerDuty: Compute SRE |
| VSI stuck in stopping / deleting | RCOS / Compute SRE | PagerDuty: Compute SRE |
| VSI crashed / unexpected stop | Compute SRE  | PagerDuty: Compute SRE |
| Hardware fault (multiple VMs on same host) | Compute SRE| PagerDuty: Compute SRE |
| Encrypted volume / Keylore failure | SSRE / BYOK team | PagerDuty: BYOK |
| Storage backend failure | RSOS (Storage SRE) | PagerDuty: Storage SRE |
| Network / macvtap / NEI issue | RNOS (Network SRE) | PagerDuty: Network SRE |
| Console not accessible | Compute SRE | PagerDuty: Compute SRE |
| Scheduling failure / capacity | RCOS / Compute SRE 
| DNS / ETCD / HAProxy cascade | Compute SRE + Platform SRE | PagerDuty: Compute SRE |
| Insufficient RCOS pods | RCOS team |PagerDuty: Compute SRE |
| Guest OS fault | Customer case
| Unknown | Compute SRE | PagerDuty: Compute SRE |

If not resolved within **30 minutes**, page out via PagerDuty using the [IPOPS VPC master on-call schedule](http://9.208.66.19:3002/core-master-on-call).

---


## Related Runbooks

| Symptom | Runbook |
|---------|---------|
| VSI stuck in `starting` | [Instance_Stuck_Starting.md](Instance_Stuck_Starting.md) |
| VSI stuck in `stopping` | [Instance_Stuck_Stopping.md](Instance_Stuck_Stopping.md) |
| VSI stuck in `deleting` | [Instance_Stuck_Deleting.md](Instance_Stuck_Deleting.md) |
| VSI crashed / unexpected stop | [VSI_Crash_Must_Gather_README.md](VSI_Crash_Must_Gather_README.md) |
| Encrypted VSI power on failure | [byokVSIPowerOnFailures.md](byokVSIPowerOnFailures.md) |
| VSI console not accessible | [VSI_Consoles.md](VSI_Consoles.md) |
| SGX / Secure Boot scheduling failure | [SGX_VSI_Scheduling_Failed.md](SGX_VSI_Scheduling_Failed.md) |
| Networking issues (stopping / deleting) | [genctl-network-vsi-stopping-deletion-issues.md](genctl-network-vsi-stopping-deletion-issues.md) |
| Insufficient RCOS pods | [RCOS_insufficient_running_pods_alert.md](RCOS_insufficient_running_pods_alert.md) |
| ETCD cluster health | [etcd-cluster-status-check.md](etcd-cluster-status-check.md) |
| Capacity issues | [Process-for-Handling-Capacity-issues.md](Process-for-Handling-Capacity-issues.md) |

---


