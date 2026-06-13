### Genereal
The home router is installed in the Kubernetes cluster as a Pod. The VyOS Pod is running in the `priviledged` namespace with privileged access to the host network stack. 
It servers as default gateway for the LAN network (10.0.97.1) and provides DHCP, DNS (caching), NAT and firewall services. The WAN interface is connected to the host network (enp7s0f1np1) and the LAN interface is connected to a private network (enp4s0).
The deployment is installed with the following command: 
```
kubectl apply -f vyos.yaml
```

### Notes
This are the steps to run VyOS in a Kubernetes Pod with privileged access and additional network interfaces using Multus.

1. Create namespace `priviledged` with Pod Security `privileged`:
k create namespace priviledged
kubectl label namespace priviledged pod-security.kubernetes.io/enforce=privileged

2. Run the vyos pod
````
k apply -n priviledged -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: vyos-old
spec:
  hostNetwork: true
  containers:
  - name: vyos
    image: ghcr.io/severindellsperger/vyos-container:latest
    securityContext:
     privileged: true 
EOF
````

```
kubectl apply -n priviledged -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: vyos
  namespace: priviledged
  labels:
    app: vyos
spec:
  hostNetwork: true
  # hostAliases injects the hostname into /etc/hosts automatically
  hostAliases:
  - ip: "127.0.0.1"
    hostnames:
    - "dels-cp-0"
  containers:
  - name: vyos
    image: ghcr.io/severindellsperger/vyos-container:latest
    stdin: true
    tty: true
    securityContext:
      privileged: true
      capabilities:
        add: 
          - NET_ADMIN
          - NET_RAW
          - SYS_ADMIN
    # This script runs immediately after the container starts
    lifecycle:
      postStart:
        exec:
          command:
            - /bin/bash
            - -c
            - |
              # Neutralize modprobe and rmmod for the VyOS config engine
              ln -sf /bin/true /usr/sbin/modprobe
              ln -sf /bin/true /sbin/modprobe
              ln -sf /bin/true /usr/sbin/rmmod
              ln -sf /bin/true /sbin/rmmod
              # Ensure the hostname is mapped (failsafe for hostAliases)
              echo "127.0.0.1 dels-cp-0" >> /etc/hosts
    volumeMounts:
      - name: modules
        mountPath: /lib/modules
        readOnly: true
      - name: config
        mountPath: /config
  volumes:
    - name: modules
      hostPath:
        path: /lib/modules
        type: Directory
    - name: config
      emptyDir: {} # Switch to a PVC or HostPath if you want config to survive Pod deletion
EOF
```

3. Access the vyos pod
```bash
kubectl exec -n priviledged -it vyos -- su - vyos 
```
4. Configure VyOS inside the Pod
```
configure
# Interface Config
set interfaces ethernet enp7s0f1np1 address dhcp
set interfaces ethernet enp7s0f1np1 description 'WAN'
set interfaces ethernet enp4s0 address '10.0.97.1/24'
set interfaces ethernet enp4s0 description 'LAN'
# SSH
set service ssh port '22'
set service ssh listen-address '10.0.97.1'

# NTP
set service ntp server 0.ch.pool.ntp.org pool
set service ntp server 1.ch.pool.ntp.org pool
# DHCP 
set service dhcp-server shared-network-name LAN subnet 10.0.97.0/24 option default-router '10.0.97.1'
set service dhcp-server shared-network-name LAN subnet 10.0.97.0/24 option name-server '10.0.97.1'
set service dhcp-server shared-network-name LAN subnet 10.0.97.0/24 lease '86400'
set service dhcp-server shared-network-name LAN subnet 10.0.97.0/24 range 0 start '10.0.97.50'
set service dhcp-server shared-network-name LAN subnet 10.0.97.0/24 range 0 stop '10.0.97.150'
set service dhcp-server shared-network-name LAN subnet 10.0.97.0/24 subnet-id '1'
# DNS
set service dns forwarding listen-address '10.0.97.1'
set service dns forwarding allow-from '10.0.97.0/24'
# NAT
set nat source rule 100 outbound-interface name 'enp7s0f1np1'
set nat source rule 100 source address '10.0.97.0/24'
set nat source rule 100 translation address masquerade
# Firewall 
## Firewall Groups
set firewall group interface-group 'WAN' interface 'enp7s0f1np1'
set firewall group interface-group 'LAN' interface 'enp4s0'
set firewall group network-group NET-LAN-v4 network '10.0.97.0/24'
## Global Options
set firewall global-options state-policy established action accept
set firewall global-options state-policy related action accept
set firewall global-options state-policy invalid action drop
## Block Incoming Traffic from WAN to LAN
set firewall ipv4 name WAN-IN default-action 'drop'
set firewall ipv4 forward filter rule 100 action jump
set firewall ipv4 forward filter rule 100 jump-target WAN-IN
set firewall ipv4 forward filter rule 100 inbound-interface group WAN
set firewall ipv4 forward filter rule 100 destination group network-group NET-LAN-v4
set firewall ipv4 input filter default-action 'drop'
## Allow Management Access from LAN
set firewall ipv4 name VyOS_MANAGEMENT default-action 'return'
set firewall ipv4 input filter rule 20 action jump
set firewall ipv4 input filter rule 20 jump-target VyOS_MANAGEMENT
set firewall ipv4 input filter rule 20 destination port 22
set firewall ipv4 input filter rule 20 protocol tcp
set firewall ipv4 name VyOS_MANAGEMENT rule 15 action 'accept'
set firewall ipv4 name VyOS_MANAGEMENT rule 15 inbound-interface group 'LAN'
## Allow ICMP and DNS from LAN
set firewall ipv4 input filter rule 30 action 'accept'
set firewall ipv4 input filter rule 30 icmp type-name 'echo-request'
set firewall ipv4 input filter rule 30 protocol 'icmp'
set firewall ipv4 input filter rule 30 state new
set firewall ipv4 input filter rule 40 action 'accept'
set firewall ipv4 input filter rule 40 destination port '53'
set firewall ipv4 input filter rule 40 protocol 'tcp_udp'
set firewall ipv4 input filter rule 40 source group network-group NET-LAN-v4
##Allow Localhost Traffic
set firewall ipv4 input filter rule 50 action 'accept'
set firewall ipv4 input filter rule 50 source address 127.0.0.0/8
# Hardening
set system login user admin authentication public-keys admin@vyos key AAAAC3NzaC1lZDI1NTE5AAAAIP+e3TnaNHpXfDbGgkyaLvwmVOAPxjkrfQEAJ1SZ9vIO 
set system login user admin authentication public-keys admin@vyos type ssh-ed25519
set service ssh disable-password-authentication
# Commit with confirmation
commit-confirm 5




```

# 1) Install Multus CNI:

Follow official Talos Mutlus Guideline: https://docs.siderolabs.com/kubernetes-guides/cni/multus

`kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset-thick.yml`

# 2) Create a NetworkAttachmentDefinition for the additional interface:
```
kubectl apply -f - <<EOF
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: external-wan
  namespace: priviledged
spec:
  config: '{
      "cniVersion": "0.3.1",
      "type": "macvlan",
      "master": "enp4s0", 
      "mode": "bridge",
     "ipam": {
        "type": "host-local",
        "subnet": "192.168.1.0/24",
        "rangeStart": "192.168.1.200",
        "rangeEnd": "192.168.1.216",
        "routes": [
          { "dst": "0.0.0.0/0" }
        ],
        "gateway": "192.168.1.1"
      }
    }'
EOF
```

kubectl apply -f - <<EOF
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: external-wan
  namespace: priviledged
spec:
  config: '{
      "cniVersion": "0.3.1",
      "type": "host-device",
      "device": "enp4s0", 
     "ipam": {
        "type": "static"
      }
    }'
EOF


# 3) Create the VyOS Pod with Multus annotation:
```
kubectl apply -n priviledged -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: vyos
  namespace: priviledged
  annotations:
    k8s.v1.cni.cncf.io/networks: '[ { "name": "external-wan", "interface": "eth1", "ips": ["192.168.1.12/24"] } ]'
spec:
  containers:
  - name: vyos
    image: ghcr.io/severindellsperger/vyos-container:latest
    imagePullPolicy: Never
    securityContext:
      privileged: true # Required for VyOS to manage the network stack
EOF
```