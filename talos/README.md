# Talos Home Lab Config

This is a basic Talos home lab config for a single node k8s cluster, which will be used as a 25 Gbit/s router/firewall and home server.

First create customized Talos ISO images with AMD GPU and CPU microcode support and secure boot enabled.
The customized Talos ISO images can be found with secure boot enabled can be found under the following link:
- [AMD-Metal-ISO-Secure-Boot-v1.12.1](https://factory.talos.dev/?arch=amd64&bootloader=auto&cmdline=talos.config%3Dmetal-iso&cmdline-set=true&extensions=-&extensions=siderolabs%2Famdgpu&extensions=siderolabs%2Famd-ucode&platform=metal&secureboot=true&target=metal&version=1.12.1)

Hint: To update the version just change the version in the URL.

Following customization is added to the Talos image via the factory URL:
```yaml
customization:
    extraKernelArgs:
        - talos.config=metal-iso
    systemExtensions:
        officialExtensions:
            - siderolabs/amd-ucode
            - siderolabs/amdgpu
```

From the [documentation](https://docs.siderolabs.com/talos/v1.12/reference/kernel): 
`metal-iso`:  When the kernel parameter `talos.config=metal-iso` is set, Talos will attempt to load the machine configuration from any block device with a filesystem label of `metal-iso`. Talos will look for a file named `config.yaml` in the root of the filesystem.

Hint: This customization ISO is made for the Minisforum MS-A2 mini PC with AMD CPU and GPU.

Download to ISO from the [Talos Image Factory](https://factory.talos.dev/) and write to USB stick:
```bash
dd if=path/to/talos.iso of=/dev/sdX bs=4M status=progress conv=fsync
```


Generate secrets bundle:
```bash
talosctl gen secrets -o secrets.yaml
```
These secrets are stored in 1Password.

Set environment variables for the Talos CLI:
```bash
export CLUSTER_NAME=dels-cluster
export K8s_API_ENDPOINT=https://api.dellsperger.me:6443
```

In order to create the machine configurations, first create the machine patches under `machine-patches/` for each node.
The `disk-patch.yaml` contains the disk layout configuration and ensures that there is an ephemeral partition for the system.
Then the full configuration can be generated using the following commands.

Hint: _If you install the cluster for the first time, you can already boot from the USB stick to get the correct interface and disk names. This is needed in order to create the correct machine patches._

```bash
export INSTALLER_IMAGE=factory.talos.dev/metal-installer-secureboot/b4ad7bee3282d3c8b0a5a171eebee32be8f7d91ace5942bcb2126b3a7edc7846:v1.12.5
export NODE_NAME=dels-cp-0
talosctl gen config $CLUSTER_NAME $K8s_API_ENDPOINT \
  --with-secrets secrets.yaml \
  --config-patch @machine-patches/base-patch.yaml \
  --config-patch @machine-patches/$NODE_NAME-patch.yaml \
  --config-patch @machine-patches/disk-patch.yaml \
  --output-types controlplane \
  --install-image $INSTALLER_IMAGE \
  --install-disk "" \
  --force \
  -o machine-configs/$NODE_NAME.yaml
```

## Initial config on first boot provided via ISO
Mount a USB stick and call it `metal-iso`.
Generate a `metal-iso` ISO with the new config and burn to additional USB stick:
```bash
NODE_NAME=dels-cp-0
cp machine-configs/$NODE_NAME.yaml metal-config/config.yaml

mkisofs -joliet -rock -volid 'metal-iso' -output metal-config.iso metal-config/config.yaml
# Or if you don't have mkisofs
docker run --rm -v $(pwd)/:/data alpine:latest \
 sh -c "apk add cdrkit && mkisofs -joliet -rock -volid 'metal-iso' -output /data/metal-config.iso /data/metal-config/config.yaml"
 sudo dd if=metal-config.iso of=/dev/disk4 bs=4M status=progress conv=fsync
 rm -d metal-config.iso
```
Hint: Adjust the disk path `/dev/disk4` accordingly.

See [Metal ISO](https://www.talos.dev/v1.12/reference/kernel/#metal-iso) for more details.

Or use talosctl to apply the config remotely:
```bash
talosctl apply-config --file machine-configs/$NODE_NAME.yaml --insecure -n <IP_ADDRESS>
```
After the first boot you can always re-apply the config using:
```bash
talosctl apply-config --file machine-configs/$NODE_NAME.yaml -n <IP_ADDRESS>
```
## Generate Talosconfig to access the cluster
```bash
talosctl gen config $CLUSTER_NAME $K8s_API_ENDPOINT \
  --with-secrets secrets.yaml \
  --output-types talosconfig \
  --force \
  -o talosconfig.yaml
```
Set the Talos endpoint, as the `--endpoints` flag does not work at the moment.
You can also set the node so you don't have to provide `-n <IP_ADDRESS>` all the time.
```bash
talosctl config merge talosconfig.yaml
export IP_ADDRESS=172.16.254.1
talosctl config endpoint $IP_ADDRESS
talosctl config node $IP_ADDRESS
```

## Bootstrap
Make sure to run the bootstrap command only against one node in the cluster.
```bash
talosctl bootstrap
```

## Generate kubeconfig to access the k8s cluster
```bash
talosctl kubeconfig ~/.kube/talos-dellsperger-me.config
```

Hint: If secure boot option was not set to custom it's possible that Talos reports secure boot as false (since it's not working).
If so, change the secure boot settings to Custom (allow keys to be written) and boot again from the Talos USB stick.
Then select "install keys auto".
Afterward, secure boot is enabled and reported as true.


## Base cilium setup

```bash
helm repo add cilium https://helm.cilium.io
helm repo update
helm upgrade --install cilium cilium/cilium \
 --namespace kube-system \
 --values ../infra/cilium/app/values.yaml
```

## Base CSR Approver setup
```bash
helm repo add kubelet-csr-approver https://postfinance.github.io/kubelet-csr-approver
helm repo update
helm fetch kubelet-csr-approver/kubelet-csr-approver --version 1.2.12 
helm upgrade --install kubelet-csr-approver kubelet-csr-approver/kubelet-csr-approver \
 --namespace kube-system \
 --values ../infra/kubelet-csr-approver/app/values.yaml
```

Restart kubelet with after the CSR Approver is installed:
```bash
talosctl service kubelet restart
```


## Single Node Cluster
Make control plane node schedulable:
```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane:NoSchedule-
```

## Cilium Config

It's important that following machineconfig and clusterconfigs are set:
```yaml
cluster:
  network:
    cni:
      name: none
  proxy:
    disabled: true

machine:
  features:
    kubePrism:
      enabled: true
      port: 7445
```

## Home "Router" Setup
Connect local network ports to br0 interface on the Talos node.
Provide a DHCP/DNS server via Pi-hole container through Multus with bridge interface or without Multus and host network. (host DNS caching probably needs to be disabled in Talos config, if using host network, check open listening ports on Talos with `ss -tuln`)
```yaml
machine:
  features:
    hostDNS:
      enabled: true
```
Alternatively run a DHCP/DNS server on old router and just use Talos as a firewall/router.
Enable IP forwarding and masquerading(at least for IPv4) in Cilium?


### Generate the config for another update
```bash
export CLUSTER_NAME=dels-cluster
export K8s_API_ENDPOINT=https://api.dellsperger.me:6443
export INSTALLER_IMAGE=factory.talos.dev/metal-installer-secureboot/b4ad7bee3282d3c8b0a5a171eebee32be8f7d91ace5942bcb2126b3a7edc7846:v1.12.5
export NODE_NAME=dels-cp-0
talosctl gen config $CLUSTER_NAME $K8s_API_ENDPOINT \
  --with-secrets secrets.yaml \
  --config-patch @machine-patches/base-patch.yaml \
  --config-patch @machine-patches/$NODE_NAME-patch.yaml \
  --config-patch @machine-patches/disk-patch.yaml \
  --output-types controlplane \
  --install-image $INSTALLER_IMAGE \
  --install-disk "" \
  --force \
  -o machine-configs/$NODE_NAME.yaml
```
Re-apply the config using:

After the first boot you can always re-apply the config using:
```bash
talosctl apply-config --file machine-configs/$NODE_NAME.yaml
```