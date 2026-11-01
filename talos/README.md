# Talos Home Lab Config

Single node Kubernetes cluster on a Minisforum MS-A2 (AMD CPU/GPU), used as a 25 Gbit/s router/firewall and home server.

The machine config uses the native Talos v1.14 multi-document format (`KubeClusterConfig`, `KubeletConfig`, `UnattendedInstallConfig`, ...).
Only fields without a replacement document (`machine.certSANs`, `cluster.etcd`) remain in the `v1alpha1` document.

## Hardware

| Disk | Purpose |
|---|---|
| Kingston KC3000 512GB (NVMe) | Talos system disk (install target, `EPHEMERAL` + `local-system-nvme` volume) |
| Kingston KC3000 2TB (NVMe) | Data disk (planned ZFS mirror) |
| Crucial P310 2TB (NVMe) | Data disk (planned ZFS mirror) |

| Network | Interface(s) | Address |
|---|---|---|
| Management | `br-mgmt`: right SFP `enp5s0f0np0` + `dummy0` | `172.16.254.1/24`, `fd00:254::1/64` |
| LAN (Access Points) | `br-lan`: 2.5 Gbit NICs `enp3s0`, `enp4s0` | `10.0.97.1/24` (DHCP/DNS/NAT by VyOS) |
| WAN | `enp7s0f1np1` | DHCP from the ISP |

Hint: _Interface names depend on the PCIe bus position. Adding or moving PCIe devices (e.g. NVMe disks) can rename them. Check with `talosctl get links` and adjust `machine-patches/dels-cp-0-patch.yaml` and `../infra/vyos/cm.yaml`._

## Full (re)installation

Overview:
1. [Prerequisites](#0-prerequisites)
2. [Talos USB stick](#1-talos-usb-stick)
3. [Secrets](#2-secrets)
4. [Machine config](#3-machine-config)
5. [Config USB stick (`metal-iso`)](#4-config-usb-stick-metal-iso)
6. [Install](#5-install)
7. [Talosconfig and checks](#6-talosconfig-and-checks)
8. [Bootstrap](#7-bootstrap)
9. [Cilium](#8-cilium)
10. [Kubelet CSR approver](#9-kubelet-csr-approver)
11. [Storage](#10-storage)
12. [VyOS (LAN router)](#11-vyos-lan-router)
13. [Flux](#12-flux)

### 0. Prerequisites

- Tools on the Mac: `talosctl`, `kubectl`, `helm`, `docker` (to build the config ISO).
- `secrets.yaml` from 1Password in this directory (see [Secrets](#2-secrets)).
- Two USB sticks: one for the Talos ISO, one for the config (`metal-iso`).
- Screen and keyboard at the node for the install (boot menu, install progress on the Talos console).
- WAN cable plugged in: the installer image is pulled from `factory.talos.dev` during the install, and all container images come over the WAN too.
- Mac connected to the management network (right SFP port) with a static IP, e.g. `172.16.254.2/24`.

All commands below are run from this `talos/` directory and use these variables:

```bash
export TALOS_VERSION=v1.14.5
export KUBERNETES_VERSION=1.36.2
export SCHEMATIC_ID=b4ad7bee3282d3c8b0a5a171eebee32be8f7d91ace5942bcb2126b3a7edc7846
export INSTALLER_IMAGE=factory.talos.dev/metal-installer-secureboot/$SCHEMATIC_ID:$TALOS_VERSION
export CLUSTER_NAME=dels-cluster
export K8s_API_ENDPOINT=https://api.ip6.run:6443
export NODE_NAME=dels-cp-0
export IP_ADDRESS=172.16.254.1
export KUBECONFIG=~/.kube/talos-dellsperger-me.config
```

### 1. Talos USB stick

The ISO is customized via the [Talos Image Factory](https://factory.talos.dev/?arch=amd64&bootloader=auto&cmdline=talos.config%3Dmetal-iso&cmdline-set=true&extensions=-&extensions=siderolabs%2Famdgpu&extensions=siderolabs%2Famd-ucode&platform=metal&secureboot=true&target=metal&version=1.14.5) with AMD GPU and CPU microcode support and secure boot.
The schematic below results in `SCHEMATIC_ID`; to update Talos, only `TALOS_VERSION` changes.
```yaml
customization:
    extraKernelArgs:
        - talos.config=metal-iso
    systemExtensions:
        officialExtensions:
            - siderolabs/amd-ucode
            - siderolabs/amdgpu
```

With `talos.config=metal-iso`, Talos loads the machine config from `config.yaml` in the root of any block device with the filesystem label `metal-iso` (see [kernel reference](https://docs.siderolabs.com/talos/v1.14/reference/kernel)).

Download the ISO, verify the version and write it to the first USB stick (find the stick with `diskutil list external`):
```bash
curl -fL -o ../tmp/image/metal-amd64-secureboot-$TALOS_VERSION.iso \
  https://factory.talos.dev/image/$SCHEMATIC_ID/$TALOS_VERSION/metal-amd64-secureboot.iso
LC_ALL=C grep -aom1 "VERSION_ID=v[0-9.]*" ../tmp/image/metal-amd64-secureboot-$TALOS_VERSION.iso   # must print $TALOS_VERSION

diskutil unmountDisk /dev/diskX
sudo dd if=../tmp/image/metal-amd64-secureboot-$TALOS_VERSION.iso of=/dev/rdiskX bs=4m
diskutil eject /dev/diskX
```

### 2. Secrets

The secrets bundle contains the cluster PKI and tokens. It is generated once and stored in 1Password.
Always reuse it; regenerating it creates a new cluster identity (new CAs, existing talosconfig/kubeconfig stop working).
```bash
# Only for a brand new cluster:
talosctl gen secrets -o secrets.yaml
```

### 3. Machine config

Patches under `machine-patches/`:
- `base-patch.yaml`: cluster wide settings (cluster endpoint, pod/service subnets, Cilium prerequisites: no CNI, no kube-proxy, KubePrism, IP forwarding, no control-plane taint).
- `$NODE_NAME-patch.yaml`: node specific settings (certSANs, etcd/kubelet subnets, kernel modules, hostname, network links).
- `disk-patch.yaml`: install disk selection and volume layout (`EPHEMERAL` limited to 50GiB, rest of the system disk as user volume `local-system-nvme`).

```bash
talosctl gen config $CLUSTER_NAME $K8s_API_ENDPOINT \
  --with-secrets secrets.yaml \
  --kubernetes-version $KUBERNETES_VERSION \
  --config-patch @machine-patches/base-patch.yaml \
  --config-patch @machine-patches/$NODE_NAME-patch.yaml \
  --config-patch @machine-patches/disk-patch.yaml \
  --output-types controlplane \
  --install-image $INSTALLER_IMAGE \
  --force \
  -o machine-configs/$NODE_NAME.yaml
talosctl validate --config machine-configs/$NODE_NAME.yaml --mode metal --strict
```

Notes:
- The install disk is selected by the CEL expression in `UnattendedInstallConfig` (`disk-patch.yaml`): the only NVMe disk smaller than 1TB (KC3000 512GB). The 2TB data disks are never touched by the installer.
- Patches can't delete map keys containing `/` (e.g. the control-plane taint). Therefore, `base-patch.yaml` deletes the generated `KubeNodeConfig` and the node patch redefines it without taints.
- `KubeFlannelCNIConfig` is deleted in `base-patch.yaml`. Without it Talos deploys Flannel, which conflicts with Cilium.

### 4. Config USB stick (`metal-iso`)

Build an ISO with the label `metal-iso` containing the config and write it to the second USB stick:
```bash
mkdir -p metal-config
cp machine-configs/$NODE_NAME.yaml metal-config/config.yaml
docker run --rm -v "$(pwd)":/data alpine:3.20 \
  sh -c "apk add -q cdrkit && mkisofs -joliet -rock -volid 'metal-iso' -output /data/metal-config.iso /data/metal-config/config.yaml"

diskutil unmountDisk /dev/diskX
sudo dd if=metal-config.iso of=/dev/rdiskX bs=4m
diskutil eject /dev/diskX
rm metal-config.iso
```
Hint: _Don't use `hdiutil makehybrid`: it uppercases the label to `METAL_ISO`, which Talos doesn't find._

Alternative without a config stick: boot the Talos stick (maintenance mode) and apply the config over the network:
```bash
talosctl apply-config --insecure --file machine-configs/$NODE_NAME.yaml -n <MAINTENANCE_IP>
```

### 5. Install

1. Plug in both USB sticks and the WAN cable.
2. Boot from the Talos stick (firmware boot menu).
3. Watch the console: it must show `$TALOS_VERSION` and install to the KC3000 512GB. Talos pulls the installer image, installs and reboots.
4. Remove both sticks during the reboot (otherwise the Talos stick boots again).

If Talos reports secure boot as `false`: set the firmware secure boot mode to *Custom* (allow keys to be written), boot from the Talos stick, select *Enroll Secure Boot keys: auto* and boot again.
The keys are stored in the firmware, so a disk replacement doesn't require enrolling them again.

### 6. Talosconfig and checks

The talosconfig is derived from `secrets.yaml`, so an existing one keeps working after a reinstall.
```bash
talosctl gen config $CLUSTER_NAME $K8s_API_ENDPOINT \
  --with-secrets secrets.yaml \
  --output-types talosconfig \
  --force \
  -o talosconfig.yaml
export TALOSCONFIG=$PWD/talosconfig.yaml
talosctl config endpoint $IP_ADDRESS
talosctl config node $IP_ADDRESS
```

Check the node before bootstrapping:
```bash
talosctl version          # Server Tag: $TALOS_VERSION
talosctl get systemdisk   # must be the KC3000 512GB
talosctl get disks        # map the dev paths to the models
talosctl service etcd     # waiting for bootstrap, NOT Running
```

If `172.16.254.1` is not reachable, check the cable is in the right SFP port and the interface names (see the hint under [Hardware](#hardware)).

### 7. Bootstrap

Run only once, against one node:
```bash
talosctl bootstrap
talosctl kubeconfig ~/.kube/talos-dellsperger-me.config
kubectl get pods -A
```
The control-plane pods start, but the node stays `NotReady` and CoreDNS stays `Pending` until Cilium is installed.
For the same reason `talosctl health` doesn't complete yet; don't wait for it.

### 8. Cilium

Cilium uses KubePrism (`localhost:7445`) to reach the API server and replaces kube-proxy.
Gateway API is enabled in the values, so the Gateway API CRDs must be installed first (same version as in `../infra/cilium/crd/gateway-api.yaml`, Flux takes them over later):
```bash
kubectl apply --server-side -k "https://github.com/kubernetes-sigs/gateway-api/config/crd?ref=v1.6.3"

helm repo add cilium https://helm.cilium.io
helm repo update
helm upgrade --install cilium cilium/cilium \
  --version 1.20.4 \
  --namespace kube-system \
  --values ../infra/cilium/app/values.yaml

kubectl get nodes -w                  # dels-cp-0 → Ready
kubectl -n kube-system get pods       # cilium, cilium-operator, coredns → Running
```

### 9. Kubelet CSR approver

The kubelet requests its serving certificate via CSR (`serverTLSBootstrap: true`), which kubelet-csr-approver approves.
Until then `kubectl logs`/`exec` fail with TLS errors.
```bash
helm repo add kubelet-csr-approver https://postfinance.github.io/kubelet-csr-approver
helm repo update
helm upgrade --install kubelet-csr-approver kubelet-csr-approver/kubelet-csr-approver \
  --version 1.2.12 \
  --namespace kube-system \
  --values ../infra/kubelet-csr-approver/app/values.yaml
talosctl service kubelet restart

kubectl get csr                                   # kubelet-serving → Approved,Issued
kubectl -n kube-system logs ds/cilium --tail=5    # works = serving cert OK
```

### 10. Storage

local-path-provisioner plus a default StorageClass (required by the VyOS PVC):
```bash
kubectl apply -f ../infra/local-path-provisioner/app/app/local-path-provisioner.yaml
kubectl label namespace local-path-storage pod-security.kubernetes.io/enforce=privileged
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: default-storage-class
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: rancher.io/local-path
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
EOF
kubectl -n local-path-storage rollout status deploy/local-path-provisioner
```

### 11. VyOS (LAN router)

VyOS runs as a privileged host network pod and provides DHCP, DNS, NAT and firewall for the LAN (see `../infra/vyos/README.md`).
It is applied manually, as the LAN has no internet before it runs.
```bash
kubectl create namespace priviledged
kubectl label namespace priviledged pod-security.kubernetes.io/enforce=privileged
kubectl apply -f ../infra/vyos/cm.yaml
kubectl apply -f ../infra/vyos/vyos.yaml
kubectl -n priviledged get pvc,pods -w
```

Check: a LAN client gets an address in `10.0.97.50-150` and has internet; the APs get `10.0.97.10`/`.11` (static mappings).
```bash
kubectl -n priviledged exec -it deploy/vyos -- su - vyos -c "show interfaces; show dhcp server leases"
```

### 12. Flux

Everything else (cert-manager, External Secrets, KubeVirt, ...) is deployed by Flux, see [`../infra/README.md`](../infra/README.md).

## Updating

### Machine config
Regenerate the config ([step 3](#3-machine-config)) and apply it:
```bash
talosctl apply-config --file machine-configs/$NODE_NAME.yaml
```

### Talos / Kubernetes version
Update `TALOS_VERSION` / `KUBERNETES_VERSION`, regenerate the config and upgrade:
```bash
talosctl upgrade --image $INSTALLER_IMAGE
talosctl upgrade-k8s --to $KUBERNETES_VERSION
```
