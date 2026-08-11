# Sprint 3 — Storage for Virtual Machines

**Cluster:** Hetzner lab, OpenShift 4.22.8, compact 3-master (schedulable), OpenShift Virtualization 4.22.2
**Platform:** `none` (bare metal / libvirt)
**Date:** 2026-08-11

---

## Starting point

The cluster shipped with a single StorageClass, `managed-nfs-storage`, provided by the `hetzner-ocp4` repo. It uses the `nfs-subdir` external provisioner — **not a CSI driver**. Consequence chain:

```
platform: none
  → OpenShift installs no CSI driver
    → only the repo's nfs-subdir provisioner exists (non-CSI)
      → no VolumeSnapshotClass
        → snapshot / clone / restore all unavailable
          → CDI falls back to StorageProfile cloneStrategy: copy
```

Verified with:

```bash
oc get csidrivers                                        # No resources found
oc get pods -n openshift-cluster-csi-drivers             # empty namespace
oc get infrastructure cluster -o jsonpath='{.status.platform}'   # None
oc get co storage -o jsonpath='{.status.conditions}' | jq
# → "DefaultStorageClassControllerAvailable: No default StorageClass for this platform"
```

The `storage` cluster operator reporting `Available: True / reason: AsExpected` is **correct behaviour**, not a failure — it correctly determined there was nothing to deploy.

---

## Task 1 — Configure StorageClasses for Virtual Machines (#18)

### Why a new StorageClass was needed

`managed-nfs-storage` cannot support snapshots at any layer. Snapshots are a CSI feature: the `external-snapshotter` sidecar calls `CreateSnapshot` on a CSI driver. With no CSI driver, there is nothing to call. This was previously investigated (July session) and confirmed dead — the console Clone button and the `VirtualMachineClone` CRD were both permanently unavailable.

### Options considered

| Option | Verdict |
|---|---|
| **LVMS / TopoLVM** | Rejected — requires a spare block device. `lsblk` confirmed all four NVMe drives fully partitioned into `md0` (swap), `md1` (/boot), `md2` (/). `unused devices: <none>` |
| **ODF (Ceph)** | Rejected — needs significant CPU/RAM per node. Only ~5 GiB free per node |
| **csi-driver-nfs** | **Chosen** — reuses existing NFS exports, no new hardware, keeps RWX (live migration preserved) |

### Steps

Confirm the snapshot CRDs and controller already exist (OpenShift ships these — do **not** install the community snapshot-controller, it will conflict):

```bash
oc get crd | grep snapshot
oc get co csi-snapshot-controller
```

Expected: `volumesnapshots`, `volumesnapshotclasses`, `volumesnapshotcontents` CRDs present, operator `Available: True`.

Install the driver:

```bash
curl -skSL https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/deploy/install-driver.sh | bash -s v4.11.0 --
oc get csidrivers
oc get pods -n kube-system | grep csi-nfs
```

Expected: `nfs.csi.k8s.io` in `csidrivers`; controller `5/5 Running`, three node plugins `3/3 Running`. No SCC intervention was required.

Create the StorageClass and VolumeSnapshotClass:

```bash
cat <<'EOF' | oc apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi
provisioner: nfs.csi.k8s.io
parameters:
  server: 192.168.50.1
  share: /var/lib/libvirt/images/ocp-pv-user-pvs
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: true
mountOptions:
  - nfsvers=4.1
---
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-nfs-snapclass
driver: nfs.csi.k8s.io
deletionPolicy: Delete
EOF
```

`192.168.50.1` is the libvirt bridge address — how the guest nodes reach the host's NFS server. The existing exports already permit `192.168.50.0/24`, so no `exportfs` changes were needed.

`allowVolumeExpansion: true` also resolves a separate finding from the July session (disk expansion was blocked because the old SC left this unset).

Make it the default:

```bash
oc patch sc managed-nfs-storage -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
oc patch sc nfs-csi -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
oc get sc
```

### Verification — minimal PVC + snapshot test

Done before touching any VM, to isolate the storage layer:

```bash
# 1Gi PVC on nfs-csi → Bound
# VolumeSnapshot against it → READYTOUSE: true
oc get volumesnapshot -n default
oc describe volumesnapshot snaptest-snap -n default | tail -20
```

Events showed `CreatingSnapshot` → `SnapshotCreated` → `SnapshotReady`. Test resources deleted afterwards.

### Key finding — CDI auto-recognition

The two StorageProfiles side by side are the whole point of this task:

| | `managed-nfs-storage` | `nfs-csi` |
|---|---|---|
| `Recognized` | `False` / `UnrecognizedProvisioner` | `True` / `RecognizedProvisioner` |
| `claimPropertySets` | empty — **manual patch required** | auto-populated RWX / Filesystem |
| `cloneStrategy` | `copy` | `snapshot` |
| `snapshotClass` | none | `csi-nfs-snapclass` |

For `managed-nfs-storage`, CDI could not infer anything and the profile had to be patched by hand:

```bash
oc patch storageprofile managed-nfs-storage --type merge -p \
  '{"spec":{"claimPropertySets":[{"accessModes":["ReadWriteMany"],"volumeMode":"Filesystem"}]}}'
```

Without this, all six common boot-image DataVolumes sat in `ImportScheduled` for 15 hours with no error — CDI simply had no way to construct a valid PVC spec.

For `nfs-csi`, **no patch was needed**. CDI recognised the provisioner, queried its capabilities, found the VolumeSnapshotClass, and set `cloneStrategy: snapshot` on its own.

---

## Task 2 — Import QCOW2 VM Image (#19)

`enableCommonBootImageImport: true` in the HyperConverged CR triggers automatic import of the golden images from `registry.redhat.io`. These are qcow2 disk images.

```bash
oc get dv -A
oc get datasource -n openshift-virtualization-os-images
```

Result: six images `Succeeded 100%` — centos-stream9, centos-stream10, fedora, rhel8, rhel9, rhel10.

These imports were blocked until the `managed-nfs-storage` StorageProfile patch above. They unstuck within seconds of it being applied.

### Creating a VM from the imported image

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: tiny-vm
  namespace: default
spec:
  runStrategy: Always
  dataVolumeTemplates:
  - metadata:
      name: tiny-vm-rootdisk
    spec:
      storage:
        accessModes: [ReadWriteMany]
        storageClassName: nfs-csi
        resources:
          requests:
            storage: 40Gi
      sourceRef:
        kind: DataSource
        name: centos-stream10
        namespace: openshift-virtualization-os-images
  template:
    spec:
      domain:
        cpu:
          cores: 1
        memory:
          guest: 2Gi
        devices:
          disks:
          - name: rootdisk
            disk:
              bus: virtio
          - name: cloudinitdisk
            disk:
              bus: virtio
          interfaces:
          - name: default
            masquerade: {}
        resources: {}
      networks:
      - name: default
        pod: {}
      volumes:
      - name: rootdisk
        dataVolume:
          name: tiny-vm-rootdisk
      - name: cloudinitdisk
        cloudInitNoCloud:
          userData: |
            #cloud-config
            user: centos
            password: centos
            chpasswd: { expire: False }
```

Result: `tiny-vm` Running on `master-2`, `READY: True`, `LIVE-MIGRATABLE: True`.

**Must use `dataVolumeTemplates` + `sourceRef`, not `containerDisk`.** A containerDisk VM has no PVC, which blocks cloning, migration and snapshots entirely — a mistake made in the July session.

---

## Task 3 — Create VM Snapshots (#20)

```bash
cat <<'EOF' | oc apply -f -
apiVersion: snapshot.kubevirt.io/v1beta1
kind: VirtualMachineSnapshot
metadata:
  name: tiny-vm-snap1
  namespace: default
spec:
  source:
    apiGroup: kubevirt.io
    kind: VirtualMachine
    name: tiny-vm
EOF

oc get vmsnapshot -n default
```

Result: `Succeeded`, `readyToUse: true`.

Notable fields in the resulting object:

- `indications: [GuestAgent, Online]` — the QEMU guest agent was present in the CentOS Stream 10 golden image, so this is a **filesystem-consistent online snapshot**, not merely crash-consistent. Without the agent, stop the VM first for a clean snapshot.
- `includedVolumes: [rootdisk]` / `excludedVolumes: [cloudinitdisk]` — cloud-init disks are ephemeral and correctly excluded. This matches the Diagnostics tab warning "Snapshot is not supported for this volumeSource type [cloudinitdisk]".

The underlying VolumeSnapshot and VolumeSnapshotContent are created automatically:

```bash
oc get volumesnapshot -A
oc get volumesnapshotcontent
```

---

## Task 4 — Restore VM from Snapshot (#21)

**The VM must be stopped.** A restore replaces the VM's disk — it does not "boot from a snapshot". While the VM runs, qemu holds an open handle on the disk file and the VMI's volume binding is fixed at Pod creation time.

- **Restore** = roll this VM back in time → requires stop
- **Clone** = create a new VM from that point in time → source VM keeps running

```bash
virtctl stop tiny-vm -n default
oc get vm,vmi -n default        # vmi must be empty

cat <<'EOF' | oc apply -f -
apiVersion: snapshot.kubevirt.io/v1beta1
kind: VirtualMachineRestore
metadata:
  name: tiny-vm-restore1
  namespace: default
spec:
  target:
    apiGroup: kubevirt.io
    kind: VirtualMachine
    name: tiny-vm
  virtualMachineSnapshotName: tiny-vm-snap1
EOF

oc get vmrestore -n default
```

Progress passes through `Reason: Creating new PVCs` → `Waiting for new PVCs`. A new PVC named `restore-<uid>-rootdisk` appears and stays Pending while data is written. `COMPLETE: true` when finished.

No new VM appears — the restore rewrites `tiny-vm`'s rootdisk reference to point at the restored PVC. The original `tiny-vm-rootdisk` PVC becomes orphaned and can be deleted.

---

## Mistakes made & fixes

| Mistake | Symptom | Cause / fix |
|---|---|---|
| Target PVC sized 10Gi | `DataVolumeError`, DV stuck at `CloneScheduled` | `target resources requests storage size is smaller than the source 11381663335 < 34144990004`. CDI forbids a target smaller than the source. Golden image PVCs are 31.8 GiB, so target must be ≥ that. PVC size is immutable — delete the VM and recreate with 40Gi |
| Planned 1 GiB RAM for CentOS Stream 10 | (caught before applying) | CentOS Stream 10 / RHEL 10 minimum is 2 GiB. Also requires x86-64-v3 — works here only because `master_special_cpu: host-passthrough` exposes the real Zen 2 CPU to the guests |
| Patched the `nfs-csi` StorageProfile | `patched` but redundant | CDI had already auto-populated it. Harmless (values identical), but demonstrates why the `managed-nfs-storage` patch *was* necessary — there, CDI could infer nothing |

---

## Storage behaviour findings

Space accounting on `/dev/md2` (873 G total):

| Item | Declared | Actual on disk |
|---|---|---|
| Golden images (old `nfs-subdir` SC) | 31.8 GiB each | 1.7–2.6 G each — **sparse** |
| `tiny-vm-rootdisk` (`nfs-csi`) | 40 Gi | **40 G — fully pre-allocated** |
| VM snapshot tar | — | 1.7 G |

Two behaviours worth internalising:

1. **csi-driver-nfs pre-allocates the full declared capacity.** Every VM costs its declared disk size in real space, regardless of how empty it is. Budget accordingly — this is the opposite of the old provisioner's sparse behaviour.
2. **Snapshots are cheap.** The tar archive contains only the real data (~1.7 G), not the declared 40 G. Snapshots compress badly but only package what exists, so taking several is inexpensive.

Restore is a genuine data copy, not a pointer swap — disk usage grows by the full declared size during the operation.

---

## Cluster capability summary after Sprint 3

| Feature | `managed-nfs-storage` | `nfs-csi` |
|---|---|---|
| RWX | yes | yes |
| Live migration | yes | yes |
| Volume expansion | no | yes |
| VolumeSnapshot | no | yes |
| VM snapshot / restore | no | yes |
| CDI clone strategy | `copy` (host-assisted) | `snapshot` |

`nfs-csi` is a community driver, not Red Hat supported — acceptable for a lab, not for a customer environment. The supported bare-metal answers remain LVMS (needs a spare block device) and ODF (needs real resources).

---

## Operational notes

Carried over from the July incident post-mortem — the previous cluster died of **memory exhaustion**, not disk. CNV guest VMs all landed on `master-0`, pushing the node past what it could sustain alongside the control plane; etcd ended up in D-state and CRI-O wedged.

- `vm.swappiness=10` set persistently in `/etc/sysctl.d/99-ocp.conf`. Swapping guest kernel pages is far worse than the guest simply having less RAM.
- Only ~5 GiB free per node. A VM must fit entirely on one node — check placement after each creation: `oc get vmi -A -o wide`
- Spread VMs across nodes deliberately rather than letting the scheduler stack them.
- Kubelet certificates expire if the cluster idles ~30 days. Either shut down cleanly when not in use, or boot it monthly so rotation happens. Restart playbook: `ansible/04-start-cluster.yml`
- `auth/` directory backed up outside the repo — `99-destroy-cluster.yml` wipes the install directory, and the cert-based kubeconfig is what works when OAuth is broken.
