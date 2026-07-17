# OpenShift Virtualization Lab Runbook

**Cluster:** Hetzner lab (hetzner-ocp4), OpenShift Virtualization 4.22.2 (HCO)
**Storage:** `managed-nfs-storage` (nfs-subdir provisioner `redhat-emea-ssa-team/hetzner-ocp4`, non-CSI)
**Date:** 2026-07-17

## Task Checklist

- [x] Create VM templates
- [x] Configure cloud-init
- [x] Clone VMs (host-assisted copy path — snapshot-based clone not supported on this storage)
- [x] Resize VM resources (CPU/memory incl. hot-plug; disk expansion not supported on this storage)

---

## 1. Create VM Templates

### Concept

A `Template` (`template.openshift.io/v1`) wraps a full `VirtualMachine` object under `objects:` plus `parameters:` that get substituted at processing time (`${NAME}`, `${CLOUD_USER_PASSWORD}`). Red Hat stock templates live in the `openshift` namespace and appear in the catalog cluster-wide; custom templates are visible in the catalog only for the namespace they live in — put shared templates in `openshift`.

Alternative (newer) path: instance types + preferences + bootable volumes. Less boilerplate, where the project is heading; templates remain right for parameterized processing and console catalog self-service.

### Procedure (console)

1. Virtualization → Templates → **Create Template** → edit the YAML example → Create
2. Or clone a stock template (kebab → Clone) and edit the clone — stock templates are read-only except labels/annotations

### Working template (final state)

```yaml
kind: Template
apiVersion: template.openshift.io/v1
metadata:
  name: example
  namespace: openshift          # cluster-wide catalog visibility
  labels:
    os.template.kubevirt.io/fedora: 'true'
    template.kubevirt.io/type: vm
    workload.template.kubevirt.io/server: 'true'
    template.kubevirt.io/architecture: amd64   # catalog is multi-arch aware
  annotations:
    description: VM template example
    iconClass: icon-fedora
    name.os.template.kubevirt.io/fedora: Fedora
    openshift.io/display-name: Fedora VM
objects:
  - apiVersion: kubevirt.io/v1
    kind: VirtualMachine
    metadata:
      name: '${NAME}'
      labels:
        app: '${NAME}'
    spec:
      runStrategy: Halted
      dataVolumeTemplates:
        - metadata:
            name: '${NAME}-rootdisk'
          spec:
            sourceRef:
              kind: DataSource
              name: fedora
              namespace: openshift-virtualization-os-images
            storage:
              resources:
                requests:
                  storage: 30Gi
      template:
        metadata:
          labels:
            kubevirt.io/domain: '${NAME}'
        spec:
          domain:
            cpu:
              cores: 1
              sockets: 1
              threads: 1
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
                  model: virtio
              rng: {}
          hostname: '${NAME}'
          networks:
            - name: default
              pod: {}
          terminationGracePeriodSeconds: 180
          volumes:
            - name: rootdisk
              dataVolume:
                name: '${NAME}-rootdisk'
            - name: cloudinitdisk
              cloudInitNoCloud:
                userData: |
                  #cloud-config
                  user: xiyuefedora
                  password: ${CLOUD_USER_PASSWORD}
                  chpasswd:
                    expire: false
                  ssh_authorized_keys:
                    - ssh-ed25519 AAAA... xiysun@xiysun-mac
                  packages:
                    - qemu-guest-agent
                  runcmd:
                    - systemctl enable --now qemu-guest-agent
                  timezone: Europe/Berlin
parameters:
  - name: NAME
    description: Name for the new VM
    generate: expression
    from: 'example-[a-z0-9]{16}'
  - name: CLOUD_USER_PASSWORD
    description: Randomized password for the cloud-init user
    generate: expression
    from: '[a-z0-9]{4}-[a-z0-9]{4}-[a-z0-9]{4}'
```

### Create a VM from the template (CLI)

```bash
# generated NAME + password
oc process -n openshift example | oc create -n <project> -f -

# explicit parameters
oc process -n openshift example -p NAME=fedora-dv-01 -p CLOUD_USER_PASSWORD=changeme123 \
  | oc create -n <project> -f -

# preview without creating
oc process -n openshift example -p NAME=test -o yaml

virtctl start fedora-dv-01 -n <project>     # runStrategy: Halted → no autostart
oc get vmi -n <project> -w
virtctl console fedora-dv-01 -n <project>
```

Recover a generated cloud-init password from an existing VM:

```bash
oc get vm <vm> -n <ns> -o jsonpath='{.spec.template.spec.volumes[?(@.name=="cloudinitdisk")].cloudInitNoCloud.userData}'
```

### ❌ Mistakes made & reasons

| Mistake | Symptom | Reason / Fix |
|---|---|---|
| Template created in `default` namespace | Template invisible in Create-VM catalog (only Red Hat templates shown) | Catalog shows `openshift`-namespace templates cluster-wide + templates of the selected project only. Fix: move template to `openshift` (or the VM project). Also general hygiene: don't use `default` for anything. |
| Tried to "move" template by editing `namespace:` in the YAML editor | `templates.template.openshift.io "example" not found` on Save | Namespace is part of object identity, never editable. Moving = export → strip `uid`/`resourceVersion`/`creationTimestamp`/`managedFields` → create in new ns → delete old. |
| Re-pasted YAML into Create Template flow with a `generate` name still active | Template landed as `example-9yhhdcpxv`; later `oc process -n openshift example` failed with "not found" | The console/name generation gave it a suffixed name. Fix: `oc get template -A | grep example` to find the real name; process that, or export-rename-recreate. |
| Added new `dataVolumeTemplates:` block but left old `dataVolumeTemplates: []` at the bottom | `duplicated mapping key (131:7)` on Save | Same key twice in one YAML mapping is invalid. Delete the leftover empty key. |
| First rootdisk used `containerDisk` (quay.io/containerdisks/fedora) | VM worked, but: no persistence, cloud-init reran every boot, and Clone/Migration/Snapshot all unavailable ("No migratable disks found") | `containerDisk` is ephemeral, lives in the virt-launcher pod, no PVC underneath → not snapshottable/migratable/clonable. Switch to `dataVolumeTemplates` + `sourceRef` to a DataSource for persistent golden-image-backed disks. |

### 💡 Things to know

- **Editing a template never changes existing VMs** — templates are consumed only at creation time.
- The `os.template.kubevirt.io/*`, `flavor.*`, `workload.*` labels drive catalog categorization and wizard defaults; `template.kubevirt.io/architecture` matters on arch-filtering catalogs.
- Strip server-side fields (`uid`, `resourceVersion`, `creationTimestamp`, `managedFields`) before re-applying any exported YAML.
- Catalog UI filters to check when a template "is missing": template-project dropdown, **User templates** tab, **Boot source available** checkbox.
- `oc process` works regardless of catalog visibility — use it to prove the template itself is healthy.
- Red Hat golden images are auto-updated DataSources in `openshift-virtualization-os-images` (`oc get datasource -n openshift-virtualization-os-images`).

---

## 2. Configure cloud-init

### Concept

Cloud-init personalizes a generic golden image on **first boot of the disk**: users, SSH keys, hostname, packages, scripts. KubeVirt packs `userData`/`networkData` into a small virtual disk (NoCloud datasource) attached to the VM; the guest's cloud-init reads and executes it. Golden image stays generic, per-VM identity comes from the template parameters.

### Where it lives in the VM spec

Two entries paired **by name**:

1. `spec.template.spec.domain.devices.disks` → `- name: cloudinitdisk` (device)
2. `spec.template.spec.volumes` → `- name: cloudinitdisk` with `cloudInitNoCloud:` (payload)

`cloudInitNoCloud` for almost everything; `cloudInitConfigDrive` only for guests expecting OpenStack-style config drive.

### Sensible baseline userData

```yaml
#cloud-config
user: xiyuefedora
password: ${CLOUD_USER_PASSWORD}
chpasswd:
  expire: false
ssh_authorized_keys:
  - ssh-ed25519 AAAA... you@machine
packages:
  - qemu-guest-agent
runcmd:
  - systemctl enable --now qemu-guest-agent
timezone: Europe/Berlin
```

- `#cloud-config` first line is mandatory (otherwise treated as shell script — which is also a valid mode, starting with `#!/bin/bash`)
- **Always install + enable `qemu-guest-agent`** — console IP display, graceful shutdown, dynamic SSH key injection depend on it
- `hostname` not needed in cloud-config if the VMI spec sets `hostname: '${NAME}'` (KubeVirt handles it)
- Static networking goes in a separate `networkData:` key (netplan v2 syntax); not needed for default masquerade/DHCP

### Production notes

- Don't inline secrets in Git-tracked templates: use `secretRef` / `networkDataSecretRef` pointing at a Secret with keys `userdata` / `networkdata` (lowercase); combine with Vault/SealedSecrets/ESO for GitOps
- SSH keys can't be `generate`-parameterized — hardcode team keys, make a required `${SSH_KEY}` parameter, or use console dynamic injection (guest-agent-based)
- Windows: no cloud-init; use `sysprep` volume + `autounattend.xml` in a ConfigMap

### Debugging in the guest

```bash
cloud-init status --long
sudo cat /var/log/cloud-init.log
sudo cat /var/log/cloud-init-output.log     # stdout of runcmd
sudo cloud-init schema --system             # validate received config
sudo cloud-init clean --logs && sudo reboot # force full re-run
```

### 💡 Things to know

- Most modules run **once per disk lifetime** (state recorded on disk). Changing userData + restart does nothing without `cloud-init clean`.
- On `containerDisk` VMs, every boot is a first boot → cloud-init reruns each start (incl. package installs: slower boots, needs mirror access).
- Keep cloudinitdisk after the root disk in `devices.disks`, don't give it a bootOrder.
- Base image must contain cloud-init (cloud images do; manually installed disks usually don't).
- One cloud-init volume per VM; modularity via cloud-init's own `#include`/vendor-data.

---

## 3. Clone VMs

### Capability check first (this decides everything)

```bash
oc get volumesnapshotclass                       # empty on this cluster
oc get storageprofile managed-nfs-storage -o yaml
```

Findings on this cluster:

```yaml
status:
  cloneStrategy: copy                 # host-assisted copy is the ONLY method
  conditions:
    - reason: UnrecognizedProvisioner # CDI doesn't know the nfs-subdir provisioner
```

**Consequence:** UI Clone action and `VirtualMachineClone` CRD are snapshot-based → **permanently unavailable here** (button greyed: "No migratable disks found" / snapshot statuses `enabled: false, No VolumeSnapshotClass`). This is a storage capability limit, not a misconfiguration.

> Lab storage (nfs-subdir, non-CSI) supports host-assisted DataVolume cloning only; VM snapshots, VirtualMachineClone, and console Clone require a CSI driver with a VolumeSnapshotClass.

### Method used: host-assisted DataVolume copy (works everywhere)

```bash
# 1. Stop source for a consistent disk
virtctl stop fedora-dv-01 -n default

# 2. Copy the disk
cat <<EOF | oc create -f -
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: fedora-clone-01-rootdisk
  namespace: default
spec:
  source:
    pvc:
      namespace: default            # ← may differ: cross-namespace source IS allowed
      name: fedora-dv-01-rootdisk
  storage:
    resources:
      requests:
        storage: 30Gi
EOF
oc get dv fedora-clone-01-rootdisk -n default -w   # wait for Succeeded

# 3. New VM on the copied disk (no cloudinitdisk — the disk is already initialized)
cat <<EOF | oc create -f -
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: fedora-clone-01
  namespace: default
spec:
  runStrategy: Halted
  template:
    metadata:
      labels:
        kubevirt.io/domain: fedora-clone-01
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
          interfaces:
            - name: default
              masquerade: {}
      networks:
        - name: default
          pod: {}
      hostname: fedora-clone-01
      volumes:
        - name: rootdisk
          dataVolume:
            name: fedora-clone-01-rootdisk
EOF
virtctl start fedora-clone-01 -n default
```

Debug a stuck copy: `oc get pods -n <ns> | grep -E 'clone|import'` and `oc describe dv <name>` (events at the bottom).

### The snapshot-based method (for clusters with proper CSI storage)

```yaml
apiVersion: clone.kubevirt.io/v1alpha1
kind: VirtualMachineClone
metadata:
  name: clone-example
  namespace: my-project
spec:
  source:
    apiGroup: kubevirt.io
    kind: VirtualMachine        # or VirtualMachineSnapshot (clean pattern: snapshot once, clone many)
    name: source-vm
  target:
    apiGroup: kubevirt.io
    kind: VirtualMachine
    name: cloned-vm
```

`oc get vmclone <name> -w` until `Succeeded`. **Same-namespace only** (source/target refs are name-only). Cross-namespace = the manual path above: DataVolume with cross-ns `source.pvc` (RBAC-gated on the source ns) + exported/repointed VM spec.

### ❌ Mistakes made & reasons

| Mistake | Symptom | Reason / Fix |
|---|---|---|
| Expected Clone to work on the containerDisk VM | Clone greyed out: "No migratable disks found" | No PVC → nothing to clone/migrate/snapshot. Fixed by switching template to DataVolume. |
| Expected Clone to work after the DataVolume switch | Still greyed out on `fedora-dv-01` | Second, independent blocker: no VolumeSnapshotClass on `managed-nfs-storage`. Console Clone + VMClone CRD are snapshot-based. Confirmed by StorageProfile `cloneStrategy: copy`. |
| `oc process -n openshift example` | "template not found" | Template was actually named `example-9yhhdcpxv` (see Task 1 mistakes). |

### 💡 Things to know

- Clone = identical twin. Post-clone hygiene before shared networks (or pre-clone on the source): `truncate -s 0 /etc/machine-id`, `rm /etc/ssh/ssh_host_*`, `cloud-init clean` → identity regenerates on first boot (re-triggers cloud-init too).
- A containerDisk VM's "clone" is simply re-processing the template — the spec is all there is.
- CDI clone priority: CSI smart clone → snapshot clone → host-assisted copy (fallback that always works).
- StorageProfile is editable (relevant when a real CSI driver isn't auto-recognized) — but can't conjure capabilities the provisioner lacks.
- To make the button light up in the lab: LVMS/TopoLVM (light, snapshots, RWO → no live migration) or ODF (RWX + snapshots, resource-hungry).

---

## 4. Resize VM Resources

### CPU / Memory — three ways

1. **Console:** VM → Configuration → Details → edit CPU | Memory
2. **Patch:**
   ```bash
   oc patch vm rhel9-1 -n openshift-cnv --type merge -p \
     '{"spec":{"template":{"spec":{"domain":{"memory":{"guest":"4Gi"}}}}}}'
   oc patch vm rhel9-1 -n openshift-cnv --type merge -p \
     '{"spec":{"template":{"spec":{"domain":{"cpu":{"sockets":2}}}}}}'
   ```
3. **`oc edit vm <name>`**

**Hot-plug (no reboot), conditions all met by `rhel9-1`** (`LiveMigratable: True`, RWX disk, `AgentConnected: True`):

- Grow-only; applied via automatic live migration of the VMI
- CPU: increase **`sockets`** (changing `cores`/`threads` ⇒ restart) — structure VMs as fixed cores × scalable sockets
- Memory: increase `memory.guest`; RHEL 9 onlines hot-plugged memory automatically
- Shrinking ⇒ `RestartRequired` condition; applies at next `virtctl restart`

Observe:

```bash
oc get vmi rhel9-1 -n openshift-cnv -w     # live migration happens
oc get vmim -n openshift-cnv               # migration object
# guest: free -h ; nproc
oc get vm rhel9-1 -n openshift-cnv -o jsonpath='{.status.conditions}' | jq   # RestartRequired?
```

Demo sequence used: 2→4Gi hot (watch migration, `free -h`) → sockets 1→2 hot (`nproc`) → 4→3Gi shrink (observe `RestartRequired`, clear with restart).

### Disk — not possible on this cluster

Two independent blockers:

1. `managed-nfs-storage` has `allowVolumeExpansion` unset → API rejects PVC growth outright
2. Even if flipped to `true`, the nfs-subdir provisioner doesn't implement expansion → resize would hang forever

On capable storage the procedure would be:

```bash
oc get sc -o custom-columns=NAME:.metadata.name,EXPAND:.allowVolumeExpansion
oc patch pvc <vm>-rootdisk -p '{"spec":{"resources":{"requests":{"storage":"50Gi"}}}}'
# then inside the guest (unless cloud-init growpart handles it on first boot):
sudo growpart /dev/vda 4
sudo xfs_growfs /          # or resize2fs for ext4
```

**Workaround used here — bigger-disk-via-clone:** DataVolume with `source.pvc` → larger `storage` request (host-assisted copy into a bigger volume) → repoint/new VM → `growpart` + `xfs_growfs` in the guest.

**Shrinking a PVC: never possible.** Plan sizes accordingly.

### 💡 Things to know

- Block device growth ≠ filesystem growth — the guest step is separate (cloud images automate it on first boot via cloud-init's growpart module).
- Failed hot-plug isn't an error: the change parks as `RestartRequired` and applies on restart (the boring always-works path).
- Hot-unplug doesn't exist.
- `rhel9-1` lives in `openshift-cnv` (operator namespace) — works, but workloads belong in regular projects; same rule as `default`.

---

## Cluster Capability Summary

| Feature | Status here | Gate |
|---|---|---|
| Templates + parameters | ✅ | — |
| Cloud-init (NoCloud) | ✅ | — |
| DataVolume from golden image | ✅ | DataSource present |
| Live migration | ✅ | RWX (NFS provides it) |
| CPU/memory hot-plug | ✅ | LiveMigratable + guest agent |
| Host-assisted DV clone | ✅ | always available |
| VM snapshot / VMClone CRD / UI Clone | ❌ | needs VolumeSnapshotClass (CSI) |
| PVC expansion (disk grow) | ❌ | needs allowVolumeExpansion + capable provisioner |
| PVC shrink | ❌ | never supported in Kubernetes |

**Future lab upgrade for full feature coverage:** LVMS (snapshots, loses live migration) or ODF (everything, resource cost).