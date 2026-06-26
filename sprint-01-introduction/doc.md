# Sprint 1 — OpenShift Virtualization Lab on a Hetzner Dedicated Server

A runbook for standing up a single-host OpenShift 4 cluster (compact, 3 schedulable
masters) on a Hetzner dedicated server and installing the OpenShift Virtualization
(CNV) operator on top, using nested virtualization.

| Item | Value |
| --- | --- |
| Server | Hetzner dedicated, AMD Ryzen 5 3600 (6c/12t), 64 GB ECC, 4 × 512 GB NVMe |
| Public IP | `116.202.226.60` |
| OS | CentOS Stream 10, software RAID 10 across the 4 NVMe |
| Hostname | `myocp` |
| Cluster name / domain | `ocp` / `example.com` (resolved via local DNS only) |
| Internal network | libvirt network `ocp`, subnet `192.168.50.0/24`, gateway `192.168.50.1` |
| Install dir | `/root/gitrepos/hetzner-ocp4/ocp` |
| Repo | https://github.com/RedHat-EMEA-SSA-Team/hetzner-ocp4 |

---

## 1. Cluster install (Ansible — summarized)

The cluster itself is built by the `hetzner-ocp4` Ansible automation. We only had to
provide a `cluster.yml`; the playbook installs libvirt/KVM, creates the node VMs, the
HAProxy load balancer, and runs the OpenShift bare-metal install.

Prep on the host:

```bash
dnf install -y python3-pip podman git
python3 -m pip install ansible-navigator --user
echo 'export PATH=$HOME/.local/bin:$PATH' >> ~/.profile && source ~/.profile

ssh-keygen -t rsa -b 4096
cat ~/.ssh/*.pub >> ~/.ssh/authorized_keys

git clone https://github.com/RedHat-EMEA-SSA-Team/hetzner-ocp4.git
cd hetzner-ocp4
cp cluster-example.yml cluster.yml   # then edit (see below)
```

Our `cluster.yml` (compact cluster, no public DNS, NFS storage, nested-virt enabled):

```yaml
# --- Identity (resolved via /etc/hosts only, so names are arbitrary) ---
cluster_name: ocp
public_domain: example.com

# --- Compact: 3 schedulable masters, no workers ---
master_count: 3
compute_count: 0
masters_schedulable: true

# --- Node sizing for 64 GB / Ryzen 3600 running CNV guests ---
master_vcpu: 6
master_memory_size: 16384      # 16 GB each; keep low because the bootstrap VM
master_memory_unit: 'MiB'      # runs concurrently with the 3 masters at peak
master_root_disk_size: '150G'

# --- Nested virt: expose host CPU so guests get AMD-V + full instruction set ---
master_special_cpu: "<cpu mode='host-passthrough'></cpu>"

# --- DNS: none = no public records, no Let's Encrypt (we use /etc/hosts) ---
dns_provider: none

# --- Local NFS StorageClass on the RAID10 array (CNV needs a StorageClass) ---
storage_nfs: true

# --- Red Hat pull secret (download from console.redhat.com) ---
image_pull_secret: 'PASTE_PULL_SECRET_JSON_HERE'

# --- Required field; unused with dns_provider: none ---
letsencrypt_account_email: 'you@example.com'
```

Run it:

```bash
ansible-navigator run ansible/setup.yml
```

---

## 3. Finishing the install manually

The `wait-for bootstrap` step was interrupted, so the post-install flow was completed
by hand instead of re-running the playbook. All commands use the install dir's kubeconfig:

```bash
export KUBECONFIG=/root/gitrepos/hetzner-ocp4/ocp/auth/kubeconfig
```


**a. Destroy the bootstrap VM** to reclaim ~16 GB. It is a UEFI guest, so the disk is
not libvirt-managed and NVRAM must be dropped explicitly:

```bash
virsh destroy ocp-bootstrap
virsh undefine ocp-bootstrap --nvram
rm -f /var/lib/libvirt/images/ocp-bootstrap.qcow2
rm -f /var/lib/libvirt/images/ocp-bootstrap.ign
```

**b. NFS StorageClass + image registry** — run only the `storage`-tagged tasks (this
skips all VM-creation tasks, so it is safe to run against the live cluster):

```bash
cd /root/gitrepos/hetzner-ocp4
ansible-navigator run -m stdout ./ansible/02-create-cluster.yml --tags post-install,storage
```

**c. Wait for full install:**

```bash
openshift-install wait-for install-complete \
  --dir /root/gitrepos/hetzner-ocp4/ocp --log-level=info
```

The kubeadmin password is at `/root/gitrepos/hetzner-ocp4/ocp/auth/kubeadmin-password`.

---

## 4. DNS fix for the `*.apps` wildcard (the important one)

Because we used `dns_provider: none`, nothing resolved the cluster's `*.apps.ocp.example.com`
wildcard. This left the **ingress** operator `Degraded` (failed canary route check) and
blocked **authentication** and **console** from going Available. The errors looked like:

```
... CanaryChecksSucceeding=False (Canary route checks for the default ingress
controller are failing ... lookup canary-openshift-ingress-canary.apps.ocp.example.com
on 172.30.0.10:53: no such host)
```

CoreDNS only resolves `cluster.local` internally and forwards everything else upstream;
the apps domain had no answer anywhere. The OpenShift DNS operator can only *forward* a
zone, not hold static records — so the fix is a small local resolver that answers the
wildcard, plus a CoreDNS forward to it.

**a. Run dnsmasq on the host** answering the wildcard, on port 5353 to avoid clashing
with libvirt's own dnsmasq on `:53`:

```bash
dnf install -y dnsmasq bind-utils
cat > /etc/dnsmasq.d/ocp-apps.conf <<'EOF'
port=5353
listen-address=192.168.50.1
bind-interfaces
no-resolv
no-hosts
address=/apps.ocp.example.com/192.168.50.1
address=/api.ocp.example.com/192.168.50.1
EOF
systemctl enable --now dnsmasq

# verify it answers and does NOT recurse
dig @192.168.50.1 -p 5353 canary-openshift-ingress-canary.apps.ocp.example.com +short  # -> 192.168.50.1
dig @192.168.50.1 -p 5353 google.com +short                                            # -> (nothing)
```

> `no-resolv` / `no-hosts` are essential — without them dnsmasq also acts as a recursive
> resolver and CoreDNS rejects its replies as "server misbehaving".

**b. Open the firewall.** Pod→host traffic crosses the libvirt bridge `virbr1`, which
firewalld puts in the **`libvirt`** zone (not `public`). libvirt re-asserts that zone, so
the port must be added to the `libvirt` zone directly:

```bash
firewall-cmd --get-zone-of-interface=virbr1          # -> libvirt
firewall-cmd --permanent --zone=libvirt --add-port=5353/udp
firewall-cmd --reload
```

**c. Point CoreDNS at dnsmasq** for the apps zone:

```bash
oc patch dns.operator/default --type=merge -p \
'{"spec":{"servers":[{"name":"ocp-apps","zones":["apps.ocp.example.com"],"forwardPlugin":{"upstreams":["192.168.50.1:5353"]}}]}}'

oc -n openshift-dns rollout restart ds/dns-default
```

**d. Verify resolution from inside the cluster** (this must return an IP, not time out):

```bash
oc -n openshift-dns rsh ds/dns-default \
  dig @192.168.50.1 -p 5353 canary-openshift-ingress-canary.apps.ocp.example.com +short
```

After a few minutes ingress, authentication, and console all recover:

```bash
oc get co ingress authentication console
```

**Laptop side** — to reach the console from a browser, add to the laptop's `/etc/hosts`
(hosts files can't wildcard, so list the routes you use):

```
116.202.226.60 api.ocp.example.com
116.202.226.60 console-openshift-console.apps.ocp.example.com
116.202.226.60 oauth-openshift.apps.ocp.example.com
116.202.226.60 downloads-openshift-console.apps.ocp.example.com
```

Console: `https://console-openshift-console.apps.ocp.example.com` (self-signed cert
warning is expected), user `kubeadmin`.

---

## 5. Install the OpenShift Virtualization operator

**Web console:** *Ecosystem → Software Catalog* → filter **Virtualization** → the
**OpenShift Virtualization** tile (Red Hat source) → **Install** → channel **stable**,
namespace **openshift-cnv** (mandatory), approval **Automatic**.

**Or via CLI** — subscribe:

```bash
oc apply -f - <<'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-cnv
  labels:
    openshift.io/cluster-monitoring: "true"
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: kubevirt-hyperconverged-group
  namespace: openshift-cnv
spec:
  targetNamespaces:
  - openshift-cnv
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: hco-operatorhub
  namespace: openshift-cnv
spec:
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  name: kubevirt-hyperconverged
  channel: "stable"
EOF

watch oc get csv -n openshift-cnv   # wait for PHASE = Succeeded
```

Then create the **HyperConverged** CR (defaults are fine — keep the name
`kubevirt-hyperconverged`). This is what actually deploys virtualization:

```bash
oc apply -f - <<'EOF'
apiVersion: hco.kubevirt.io/v1beta1
kind: HyperConverged
metadata:
  name: kubevirt-hyperconverged
  namespace: openshift-cnv
spec: {}
EOF
```

Verify health and that nested virt reached the nodes:

```bash
oc get hyperconverged kubevirt-hyperconverged -n openshift-cnv \
  -o jsonpath='{range .status.conditions[*]}{.type}={.status}{"\n"}{end}'
# Available=True, Progressing=False, Degraded=False

oc get nodes -o json | jq '.items[].status.allocatable | keys[]' | grep kvm
# devices.kubevirt.io/kvm  -> proves /dev/kvm passthrough works
```

---

## 6. Storage profile fix (required before VM disks provision)

CNV provisions VM disks through a CDI **StorageProfile**, not the StorageClass directly.
CDI doesn't recognize the repo's custom NFS provisioner, so it left the access/volume
modes blank — which is why the StorageClass was greyed out / unselectable in the VM
wizard. Tell CDI what NFS supports:

```bash
oc patch storageprofile managed-nfs-storage --type=merge -p \
'{"spec":{"claimPropertySets":[{"accessModes":["ReadWriteMany"],"volumeMode":"Filesystem"}]}}'

oc get storageprofile managed-nfs-storage -o jsonpath='{.status.claimPropertySets}'; echo
```

> RWX is chosen deliberately — it's what live migration would need later. (NFS here is
> Filesystem mode, so migration works but isn't high-performance.)

---

## 7. First VM and SSH access

Created a RHEL 9 VM (`rhel9-1`) from a template via the console. Sized small (2 GiB) to
fit alongside platform pods on a 16 GB master.

Check status:

```bash
oc get vm,vmi -A -o wide
```

**SSH via `virtctl`** — install it from the cluster (it is not a dnf package). The
download URL is under `*.apps`, so resolve it explicitly from the host:

```bash
curl -Lk --resolve hyperconverged-cluster-cli-download-openshift-cnv.apps.ocp.example.com:443:192.168.50.1 \
  -o /tmp/virtctl.tar.gz \
  https://hyperconverged-cluster-cli-download-openshift-cnv.apps.ocp.example.com/amd64/linux/virtctl.tar.gz
tar xzf /tmp/virtctl.tar.gz -C /usr/local/bin/
chmod +x /usr/local/bin/virtctl
```

`virtctl ssh` handles the **transport** to the guest (no routing/DNS to the VM needed),
but the guest's sshd still authenticates normally. RHEL cloud images disable password
auth, so a key is required. Put a public key in the guest (`~cloud-user/.ssh/authorized_keys`
via the web console), then:

```bash
virtctl ssh cloud-user@vmi/rhel9-1/openshift-cnv -i ~/.ssh/id_rsa
```

> Cleaner for future VMs: add the public key on the **SSH** tab of the create wizard so
> it's injected via cloud-init at boot.

**Optional — reach the VM from the laptop** by exposing SSH as a NodePort and jumping
through the host:

```bash
virtctl expose vmi rhel9-1 --name=rhel9-1-ssh --port=22 --type=NodePort -n openshift-cnv
oc get svc rhel9-1-ssh -n openshift-cnv          # note the NodePort
# from laptop:
ssh -J root@116.202.226.60 cloud-user@192.168.50.11 -p <nodeport>
```

---

## Sprint backlog status

- [x] Install OpenShift lab (CentOS Stream 10 + hetzner-ocp4, compact cluster)
- [x] Deploy virtualization operator (OpenShift Virtualization, `openshift-cnv`)
- [x] Validate HyperConverged CR (`Available=True`, `kvm` device present on nodes)
- [x] Deploy Linux VM (`rhel9-1`, RHEL 9)
- [x] Verify VM console access (web console + `virtctl ssh`)

## Known limitations / notes

- **Live migration** is not really usable: NFS is Filesystem-mode RWX, and the docs
  recommend Block-mode RWX. Set `evictionStrategy: None` on VMs so node drains power
  them down instead of attempting migration.
- **Wildcard DNS** only exists where we configured it (cluster CoreDNS + laptop hosts).
  New app routes need a matching laptop `/etc/hosts` entry, or set up dnsmasq locally.
- The DNS, dnsmasq, and firewalld pieces are load-bearing — if the host is rebuilt,
  section 4 must be redone.
- Ingress was degraded purely from the canary DNS issue; it is not a routing fault.