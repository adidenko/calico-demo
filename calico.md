Calico for Kubernetes
=====================

## Architecture:

### Introduction

http://docs.projectcalico.org/v2.0/reference/architecture/

- Felix, the primary Calico agent that runs on each machine that hosts endpoints.
- BGP client that distributes routing information.
- The Orchestrator plugin, orchestrator-specific code that tightly integrates Calico into that orchestrator.
- etcd, the data store.

### On kubernetes cluster:

- [Felix, BGP] Every k8s worker node runs `calico-node` container
- [The Orchestrator plugin] Every k8s worker node runs `kubelet + cni`:
    --network-plugin=cni --network-plugin-dir=/etc/cni/net.d
  K8s knows nothing about `calico` or network topology, it simply runs CNI plugin via exec.
  CNI config is /etc/cni/net.d/10-calico.conf
- [etcd] Etcd cluster is required for k8s and calico, so it's installed along with those.


### Anatomy of a calico-node container

http://docs.projectcalico.org/v2.0/reference/architecture/components

Runs the following processes in a privileged hostnet=true container:

- Felix - its primary job is to program routes and ACL’s on a workload host to provide desired connectivity to and from workloads on the host.
- BGP:
  - confd dynamically generates BIRD configuration files based on the data in etcd, triggered automatically from updates to the data. When the configuration file changes, confd triggers BIRD to load the new files
  - BIRD is an open source BGP client that is used to exchange routing information between hosts

## Creating K8s POD and Calico endpoint

### Step-by-step

1. kubelet creates a container for POD

2. kubelet runs CNI plugin (`/opt/cni/bin/calico`)
    
  There's no existing endpoint, so we need to do the following:

  1. Call the configured IPAM plugin to get IP address(es)
    
    * `/opt/cni/bin/calico-ipam`

  2. Configure the Calico endpoint

    * update etcd DB

  3. Create the veth, configuring it on both the host and container namespace.

    * Create veth pair in container namespace (via netlink Go library which uses syscalls):
      ```
      cali69829ecd4bd <---> eth0
      ```  
    * Create the routes inside the namespace, first for IPv4 then IPv6
      For IPv4 add a connected route to a dummy next hop (/32 mask) so that a default route can be set:

      ```
      169.254.1.1 dev eth0  scope link
      default via 169.254.1.1 dev eth0
      ```  
    * Move the "host" end of the veth (`cali*` interface) into the host namespace

3. [calico-node] calico-felix/calico-iptables-plugin sets up arp and routing
```
  /sbin/arp -s 10.233.97.132 d2:cc:f0:3e:b5:d9 -i cali69829ecd4bd
  /sbin/ip route replace 10.233.97.132 dev cali69829ecd4bd
```
4. [calico-node] calico-felix configures the various proc file system parameters for the interface for IPv4:

  * Allow packets from controlled interfaces to be directed to localhost
  * Enable proxy ARP (responding to workload ARP requests with the host MAC)
  * Enable the kernel's RPF check.
```
  1 > /proc/sys/net/ipv4/conf/cali69829ecd4bd/rp_filter
  1 > /proc/sys/net/ipv4/conf/cali69829ecd4bd/route_localnet
  1 > /proc/sys/net/ipv4/conf/cali69829ecd4bd/proxy_arp
  0 > /proc/sys/net/ipv4/neigh/cali69829ecd4bd/proxy_delay
```
5. kubelet gets info about POD IP address
```
  /usr/bin/nsenter -t 5833 -n -F -- /sbin/ethtool --statistics eth0
  /usr/bin/nsenter --net=/proc/5833/ns/net -F -- ip -o -4 addr show dev eth0 scope global
```

