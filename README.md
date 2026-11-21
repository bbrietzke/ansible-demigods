# demigods: a Raspberry Pi 5 Kubernetes cluster

One Ansible playbook (`demigods.yml`) that takes five freshly flashed Raspberry Pi 5s and builds the **demigods** cluster: it checks the network, applies any pending OS updates, prepares the OS, installs Kubernetes 1.37.2, sets up kube-vip and Calico, joins the workers, wipes the NVMe drives for Rook/Ceph, and checks the result.

The machines are named after demigods because they are single-board computers rather than full-sized computing resources.

---

## 1. What you get

| Host | IP | Role | Storage node | RAM | MAC |
|---|---|---|---|---|---|
| hercules | 192.168.5.9 | control plane | no | 8 GB | 88:A2:9E:29:ED:25 |
| achilles | 192.168.5.11 | worker | yes (`/dev/nvme0n1`) | 16 GB | 2c:cf:67:c7:86:d2 |
| perseus | 192.168.5.13 | worker | yes (`/dev/nvme0n1`) | 16 GB | 88:a2:9e:00:07:d2 |
| helen | 192.168.5.17 | worker | yes (`/dev/nvme0n1`) | 16 GB | 2C:CF:67:E4:89:EB |
| aenes | 192.168.5.19 | worker | yes (`/dev/nvme0n1`) | 16 GB | 88:A2:9E:7A:3B:62 |

| Item | Value |
|---|---|
| Network | `k8s0`, 192.168.5.0/26, behind OpenWRT |
| Gateway / DNS | 192.168.5.1 |
| Transient DHCP pool | 192.168.5.50 – 192.168.5.59 |
| API endpoint (kube-vip VIP) | **192.168.5.5:6443** |
| Cluster name / kubeconfig context | `demigods` / `demigods` |
| Kubernetes | v1.37.2 (kubeadm, containerd 2.x, cgroup v2) |
| kube-vip | v1.3.1, ARP mode, static pod on hercules |
| CNI | Calico v3.33.1 via the Tigera operator, **without** Goldmane or Whisker |
| Pod network | 10.253.0.0/23, one /26 block per node (8 blocks; 5 used, 3 spare) |
| Service network | 172.16.30.0/23 (API service 172.16.30.1, CoreDNS 172.16.30.10) |
| Node labels | `topology.kubernetes.io/region=HOME`, `topology.kubernetes.io/zone=DEMIGODS` |
| iSCSI initiator | `iqn.2026-11.home.demigods:<hostname>` on every node |

---

## 2. What the playbook does

`demigods.yml` contains five plays, run in order.

**Play 0: rolling OS updates for nodes already in the cluster (`hosts: demigods`, `serial: 1`)**

This play does nothing on a fresh build, because no node has joined yet. On later runs it handles each existing cluster member **one at a time**:
1. Refreshes the apt package lists and simulates `apt-get dist-upgrade` to see whether anything needs upgrading. The list of pending packages is printed.
2. If there are updates: drains the node (`kubectl drain --ignore-daemonsets --delete-emptydir-data`, which respects PodDisruptionBudgets such as Rook's), runs a full upgrade and autoremove, and reboots if `/var/run/reboot-required` exists.
3. Waits until the API answers and the node is Ready, then uncordons it before moving to the next node.
4. Nodes with nothing to upgrade are left alone: no drain, no reboot.


**Play 1: prepare every node (`hosts: demigods`)**
1. **Preflight checks.** These run on every invocation, whatever tags you pass:
   - The inventory has 1 control plane and 4 workers. Only workers are storage nodes. Addresses and MACs are unique. No static address, the gateway or the VIP falls inside the .50–.59 DHCP pool.
   - Each node is ARM64 Ubuntu.
   - **Network checks.** Each node must have the expected IP on its default interface and the expected MAC (this catches a swapped Pi or a wrong lease). The netmask must be /26, the default gateway 192.168.5.1 and the DHCP DNS server 192.168.5.1. The gateway must answer ping, `pkgs.k8s.io`, `ghcr.io` and `raw.githubusercontent.com` must resolve, and every other demigod must answer ping.
   - `/dev/nvme0n1` exists and is a block device on each storage node.
2. **OS updates for new nodes.** Nodes that haven't joined the cluster yet are checked and updated **in parallel** in the same way: refresh package lists, list pending upgrades, full upgrade if any, and a reboot if one is required. Facts are re-gathered after a reboot so the rest of the run sees the new kernel. Cluster members were already handled by Play 0.
3. **OS setup.** Stops cloud-init from managing the hostname and `/etc/hosts`. Sets the hostname and writes `/etc/hosts` entries for all nodes plus `demigods-api` → 192.168.5.5. Installs the base packages. Disables ufw, turns swap off permanently and masks zram swap. Adds `cgroup_enable=memory cgroup_memory=1` to `/boot/firmware/cmdline.txt` and **reboots automatically** if it had to change that file. Confirms cgroup v2 has the memory controller. Loads `overlay`, `br_netfilter`, `rbd`, `ceph`, `iscsi_tcp` (and `nbd`, `dm_crypt` where available) now and at boot. Sets the Kubernetes sysctls.
4. **Kubernetes packages.** Adds the `pkgs.k8s.io` v1.37 apt repo, installs `kubelet`, `kubeadm` and `kubectl` at `1.37.2-1.1`, and holds them at that version.
5. **containerd.** Checks the version is 2.x (required by Kubernetes 1.37). Generates a default `config.toml`, sets `SystemdCgroup = true` and pins the sandbox (pause) image to the one kubeadm expects. Writes `/etc/crictl.yaml`.
6. **Storage.**
   - Every node gets a unique iSCSI initiator name derived from its hostname, for example `iqn.2026-11.home.demigods:achilles`, and `iscsid` is enabled.
   - Nodes that have not joined a cluster yet get a stale `/var/lib/rook` removed.
   - **Storage nodes only:** the playbook refuses to continue if `/dev/nvme0n1` holds `/` or is mounted. Otherwise it removes Ceph LVM/device-mapper leftovers and runs `sgdisk --zap-all`, `wipefs --all`, a zero-fill of the first 100 MiB and `blkdiscard`. It then confirms that no partitions or signatures remain. A marker file (`/etc/demigods/storage-wiped`) stops later runs from wiping again, so re-running the playbook never destroys a live Ceph OSD.

**Play 2: control plane (`hosts: control_plane` → hercules)**
1. On first init, checks that nothing already answers on the VIP 192.168.5.5.
2. Uses the kube-vip v1.3.1 image itself (`kube-vip manifest pod --controlplane --arp --leaderElection`) to generate the kube-vip static pod, bound to hercules's primary interface. During `kubeadm init`, the manifest mounts `super-admin.conf` (needed on Kubernetes 1.29+). Afterwards it is switched back to `admin.conf`.
3. Renders `/etc/demigods/kubeadm-init.yaml` with:
   - cluster name `demigods`
   - `controlPlaneEndpoint` 192.168.5.5:6443
   - pod subnet 10.253.0.0/23, with controller-manager `node-cidr-mask-size=26`
   - service subnet 172.16.30.0/23
   - certificate SANs for the VIP and hostnames
   - kubelet `node-ip` and the topology labels
   
   It validates the file, pre-pulls images and runs `kubeadm init`.
4. Waits for the API on the VIP, then copies the admin kubeconfig to `~bbrietzke/.kube/config`.
5. **Calico.**
   - Applies Calico v3.33.1's CRDs and the Tigera operator, using `v3_projectcalico_org.yaml`, the CRD manifest for Kubernetes 1.36 and later.
   - Creates one `Installation` with the IP pool `demigods-pool`:
     - CIDR 10.253.0.0/23, `blockSize: 26`
     - VXLANCrossSubnet (all nodes share one L2 segment, so traffic is natively routed)
     - `natOutgoing`
     - node address autodetection pinned to 192.168.5.0/26, so the Pi's Wi-Fi can never be chosen
   - The `Goldmane` and `Whisker` resources are never created, so neither component is installed.
   - Waits for `tigerastatus/calico` to be Available.
6. Creates a 2-hour bootstrap token and computes the CA cert hash for the workers.

**Play 3: workers (`hosts: workers`)**
1. Checks that each worker can reach the API on the VIP.
2. Renders a `JoinConfiguration` with the token, CA hash, `node-ip` and topology labels. Runs `kubeadm join` against 192.168.5.5:6443, then deletes the join file, because it contains the token.

**Play 4: finalize and verify (`hosts: control_plane`)**
1. Waits until all 5 nodes are registered and Ready.
2. Applies the labels and checks them on every node:
   - `topology.kubernetes.io/region=HOME`
   - `topology.kubernetes.io/zone=DEMIGODS`
   - `demigods.home/storage-node=true|false`
   - `node-role.kubernetes.io/worker` on workers
   
   It also checks that each node's InternalIP is its static address and that its `podCIDR` is a /26 in 10.253.0.0/23.
3. Checks Calico: exactly one IP pool (`demigods-pool`, 10.253.0.0/23, /26 blocks), no Goldmane or Whisker, and the kube-vip pod Running.
4. **Pod network smoke test.** Deploys a busybox DaemonSet (one pod per node, control plane included) and checks:
   - every pod has a 10.253.x address
   - every node has a /26 block affinity
   - the `kubernetes` service is 172.16.30.1 and `kube-dns` is 172.16.30.10
   - every pod can ping every other pod across nodes
   - cluster DNS resolves `kubernetes.default.svc.cluster.local`
   - pods can reach the OpenWRT gateway through NAT outgoing
   
   The test namespace is then deleted.
5. Exports the kubeconfig to `./artifacts/demigods.kubeconfig` with the context renamed to `demigods`, and prints a node summary.

The playbook is idempotent: an initialized control plane, nodes that have already joined and disks that have already been wiped are detected and skipped.

---

## 3. Prerequisites

**Control machine (where you run Ansible)**
- `ansible-core` 2.15 or newer (uses only `ansible.builtin` modules) and `kubectl` 1.37.x for using the cluster afterwards.
- A route to 192.168.5.0/26. The OpenWRT router routes between the parent network (192.168.1.0/24) and `k8s0` without NAT, so you need either a static route on your parent router/workstation (`192.168.5.0/26 via <OpenWRT WAN IP>`) or a workstation on `k8s0` itself.
- An SSH key.

**Each Raspberry Pi 5**
- **Ubuntu Server 26.04 LTS (arm64)**, booting from **microSD or USB, not the NVMe**. The NVMe drive is wiped, and the playbook refuses to continue if the OS is on it.
- User `bbrietzke` with passwordless sudo and your SSH public key. Raspberry Pi Imager's OS customization does this. If you use a different user, change `ansible_user` in `inventory/hosts.yml`.
- Internet access through OpenWRT, for apt, `pkgs.k8s.io`, `registry.k8s.io`, `ghcr.io`, `quay.io`, `docker.io` and `raw.githubusercontent.com`.
- Storage nodes: an NVMe HAT that shows up as `/dev/nvme0n1`. If yours has no HAT+ EEPROM, add `dtparam=pciex1` to `/boot/firmware/config.txt`.

---

## 4. Exact commands, in order

### Step 1: Configure OpenWRT (static leases and the transient pool)
SSH to the OpenWRT router. These commands assume the OpenWRT interface serving 192.168.5.1/26 is named `k8s0`. If it has another name (for example `lan`), substitute it in the first block.

```sh
# DHCP on k8s0: transient pool .50-.59 (start 50, 10 leases). Gateway/DNS default to the router (192.168.5.1).
uci set dhcp.k8s0=dhcp
uci set dhcp.k8s0.interface='k8s0'
uci set dhcp.k8s0.start='50'
uci set dhcp.k8s0.limit='10'
uci set dhcp.k8s0.leasetime='12h'

# Static leases for the demigods
for entry in \
  "hercules 88:A2:9E:29:ED:25 192.168.5.9" \
  "achilles 2C:CF:67:C7:86:D2 192.168.5.11" \
  "perseus  88:A2:9E:00:07:D2 192.168.5.13" \
  "helen    2C:CF:67:E4:89:EB 192.168.5.17" \
  "aenes    88:A2:9E:7A:3B:62 192.168.5.19"; do
  set -- $entry
  uci add dhcp host
  uci set dhcp.@host[-1].name="$1"
  uci set dhcp.@host[-1].mac="$2"
  uci set dhcp.@host[-1].ip="$3"
done

uci commit dhcp
/etc/init.d/dnsmasq restart
```
192.168.5.5 (the VIP) is deliberately left out of DHCP. kube-vip claims it with ARP.

### Step 2: Flash and boot the Pis
Flash Ubuntu Server 26.04 LTS arm64 onto each Pi's boot media with Raspberry Pi Imager: set the hostname, user `bbrietzke` and your SSH key. Then boot all five.

### Step 3: Get the project onto the control machine
```sh
unzip demigods-ansible.zip && cd demigods-ansible
python3 -m pip install --user 'ansible-core>=2.15'
ansible-galaxy collection install -r requirements.yml   # no-op today; kept for consistency
```

### Step 4: Trust the hosts and check Ansible can reach them
```sh
for ip in 9 11 13 17 19; do ssh-keyscan -H 192.168.5.$ip >> ~/.ssh/known_hosts; done
ansible demigods -m ping
```
All five should return `pong`.

### Step 5: Run preflight only (no changes made)
```sh
ansible-playbook demigods.yml --tags preflight
```
Fix anything this reports before continuing. Common causes are a lease on the wrong MAC, the wrong DNS server, or a missing NVMe drive.

### Step 6: Build the cluster
```sh
ansible-playbook demigods.yml
```
Expect 20–40 minutes on Pi 5s, depending on image download speed and how many OS updates are pending. Nodes may reboot up to twice: once after OS updates if the kernel changed, and once for the cgroup kernel argument. **This step wipes `/dev/nvme0n1` on achilles, perseus, helen and aenes.**

### Step 7: Use the cluster
```sh
export KUBECONFIG=$PWD/artifacts/demigods.kubeconfig
kubectl config current-context            # -> demigods
kubectl get nodes -o wide -L topology.kubernetes.io/region,topology.kubernetes.io/zone,demigods.home/storage-node
```

### Step 8 (optional): Check by hand
```sh
kubectl get tigerastatus                                  # calico / ippools Available
kubectl get ippools.projectcalico.org -o wide             # demigods-pool 10.253.0.0/23, blockSize 26
kubectl get blockaffinities.projectcalico.org             # one /26 per node
kubectl get svc -A                                        # ClusterIPs in 172.16.30.0/23
kubectl -n kube-system get pods -o wide | grep kube-vip   # kube-vip-hercules Running
kubectl get nodes -o custom-columns=NODE:.metadata.name,PODCIDR:.spec.podCIDR
ansible demigods -b -m command -a 'cat /etc/iscsi/initiatorname.iscsi'
ansible workers  -b -m command -a 'lsblk -f /dev/nvme0n1'  # a bare disk, no partitions or FSTYPE
```

---

## 5. Additional commands

| Purpose | Command |
|---|---|
| Apply OS updates only (rolling on cluster members) | `ansible-playbook demigods.yml --tags updates` |
| Skip OS updates for this run | `ansible-playbook demigods.yml -e demigods_os_updates=false` |
| Re-run only the verification and smoke tests | `ansible-playbook demigods.yml --tags verify` |
| Re-run storage prep only (skips disks already wiped) | `ansible-playbook demigods.yml --tags storage` |
| **Force** the NVMe drives to be wiped again (destroys Ceph data) | `ansible-playbook demigods.yml --tags storage -e demigods_force_wipe=true` |
| Leave the netcheck DaemonSet running for debugging | `ansible-playbook demigods.yml --tags verify -e netcheck_keep=true` |
| Re-join or rebuild a single worker | `ansible-playbook demigods.yml --limit hercules,perseus` (hercules must be included: it issues the join token) |

**Full reset (start from scratch).** This destroys the cluster:
```sh
ansible demigods -b -m shell -a 'kubeadm reset -f; rm -rf /etc/cni/net.d /var/lib/rook /etc/demigods /etc/kubernetes/manifests/kube-vip.yaml $HOME/.kube; iptables -F; iptables -t nat -F; iptables -t mangle -F; iptables -X; ip link delete vxlan.calico 2>/dev/null; true'
ansible demigods -b -m reboot
ansible-playbook demigods.yml
```
Removing `/etc/demigods` also removes the wipe marker, so the NVMe drives are wiped again on the next run.

---

## 6. Design notes and choices to be aware of

- **containerd 2.x is mandatory.** The Kubernetes 1.37 kubelet will not start on containerd 1.7. The playbook installs Ubuntu 26.04's `containerd` package and checks its version.
- **Calico CRD manifest.** From Kubernetes 1.36, Calico uses native `projectcalico.org/v3` CRDs (`v3_projectcalico_org.yaml`) instead of the older `operator-crds.yaml`. The playbook pins Calico v3.33.1. To change it, edit `calico_version` in `inventory/group_vars/all.yml`.
- **No Goldmane or Whisker.** Upstream's `custom-resources.yaml` creates them as separate operator resources. This playbook writes its own `Installation` only, and Play 4 checks they are absent.
- **Service CIDR.** 172.16.30.0/23 gives 512 ClusterIPs. kubeadm gives the API service the first address (172.16.30.1) and CoreDNS the tenth (172.16.30.10). It doesn't overlap `k8s0` (192.168.5.0/26), the pod network (10.253.0.0/23) or the parent network (192.168.1.0/24). The service subnet can't be changed after `kubeadm init`, so rebuild the cluster (section 5, full reset) if you change it later.
- **OS updates.** Every run checks for updates (`demigods_os_updates: true` in `group_vars/all.yml`). `kubelet`, `kubeadm` and `kubectl` are held, so OS updates never change the Kubernetes version; upgrade those deliberately by changing the version variables. containerd *is* updated with the OS, which is why cluster members are drained first. If a drain or upgrade fails, the play stops and that node stays cordoned for you to inspect; run `kubectl uncordon <node>` once it's fixed. Once Rook is running, check `ceph status` is `HEALTH_OK` before an update run; draining one node at a time keeps 3 of 4 OSD hosts up. Apt waits up to 10 minutes (`demigods_apt_lock_timeout`) if unattended-upgrades holds the lock.
- **Pod CIDR arithmetic.** 10.253.0.0/23 holds 512 addresses, which is 8 × /26. Each node starts with one /26 (62 usable pod IPs). Calico gives a node a second spare block only if it fills its first. The default kubelet `maxPods` is 110, so a very busy node could take a second block. With 3 spare blocks this is fine for a lab.
- **kube-vip mode.** ARP and control-plane only (`--controlplane --arp --leaderElection`). Service `LoadBalancer` support (`--services` plus the kube-vip cloud provider) is not enabled because the spec didn't ask for it.
- **Labels.** The topology labels are set by kubelet at registration (`--node-labels`) and then re-applied with `kubectl label --overwrite`. The extra `demigods.home/storage-node` and `node-role.kubernetes.io/worker` labels are there for Rook node placement later.
- **Rook/Ceph prerequisites in place.** `lvm2`, the `rbd`/`ceph`/`nbd` kernel modules, chrony, raw unpartitioned `/dev/nvme0n1` disks and no stale `/var/lib/rook`. Rook itself is not installed. Point a `CephCluster` at `nvme0n1` with `nodeSelector: demigods.home/storage-node=true`.
- **Wipe safety.** The NVMe drives are wiped only once (the marker file), and never if the OS is on them.
- **hercules stays tainted.** The 8 GB control plane keeps its `NoSchedule` taint. Workloads and Ceph run on the 16 GB workers.
- The hostname `aenes` is kept exactly as given in the spec.

---

## 7. Files

```
demigods-ansible/
├── README.md                           this file
├── ansible.cfg                         points at the inventory, enables SSH pipelining
├── demigods.yml                        the playbook (5 plays)
├── requirements.yml                    no collections needed
├── .ansible-lint
├── inventory/
│   ├── hosts.yml                       the five demigods, IPs, MACs, roles, storage flag
│   └── group_vars/all.yml              versions, networks, labels, paths
├── templates/
│   ├── kubeadm-init.yaml.j2            InitConfiguration + ClusterConfiguration + Kubelet/KubeProxy
│   ├── kubeadm-join.yaml.j2            worker JoinConfiguration
│   ├── calico-installation.yaml.j2     Tigera Installation (IP pool, /26 blocks)
│   └── netcheck.yaml.j2                temporary smoke-test DaemonSet
└── artifacts/                          demigods.kubeconfig lands here (treat it as a secret)
```

## 8. Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| Preflight: "has 192.168.5.5x … expected …" | The Pi got a transient lease. Check its MAC in the OpenWRT static leases, then renew (`sudo networkctl renew eth0` or reboot). |
| Preflight: MAC mismatch | Two Pis' cables or SD cards are swapped. |
| Drain times out | A pod has no spare replica, or a PodDisruptionBudget blocks eviction. Check `kubectl get pdb -A`, fix it, then `kubectl uncordon <node>` and re-run. Raise `demigods_drain_timeout` if pods are just slow to stop. |
| `modprobe rbd` fails | The kernel modules package is missing. Run `sudo apt install linux-modules-extra-$(uname -r)`, then reboot if the kernel was updated. |
| `VIP must be free` fails | Something already owns 192.168.5.5 (a leftover kube-vip after a partial reset, or a DHCP client). |
| Calico never Available | `kubectl -n calico-system get pods`, `kubectl get tigerastatus -o yaml`. Usually image pulls are still in progress on slow links. Re-run with `--tags calico,verify`. |
| Get URL 404 for the Calico CRDs | The manifest name changed for that Calico version. Check the Calico on-premises install page and set `calico_crd_manifest`. |
