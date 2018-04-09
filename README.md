# CDK network spaces testing

Documented test process and results for using Juju network spaces with the Canonical Distribution of Kubernetes.

## The plan

The plan is to create a MAAS with KVM instances, all running on a single host,
with 7 network spaces to play with.

The network spaces:

| SPACE        | IFACE | CIDR         |
| ------------ | ----- | ------------ |
| juju         | br0   | 10.88.0.0/24 |
| etcd-peer    | br1   | 10.88.1.0/24 |
| etcd-client  | br2   | 10.88.2.0/24 |
| flannel      | br3   | 10.88.3.0/24 |
| apiserver    | br4   | 10.88.4.0/24 |
| loadbalancer | br5   | 10.88.5.0/24 |
| workload     | br6   | 10.88.6.0/24 |

The instances, and the spaces they'll be on:

| NODE          | SPACES |
| ------------- | ------ |
| controller0   | juju |
| easyrsa0      | juju |
| etcd0         | juju, etcd-peer, etcd-client |
| etcd1         | juju, etcd-peer, etcd-client |
| etcd2         | juju, etcd-peer, etcd-client |
| master0       | juju, etcd-client, flannel, apiserver |
| loadbalancer0 | juju, apiserver, loadbalancer |
| worker0       | juju, etcd-client, flannel, apiserver, loadbalancer, workload |
| worker1       | juju, etcd-client, flannel, apiserver, loadbalancer, workload |
| worker2       | juju, etcd-client, flannel, apiserver, loadbalancer, workload |

## Installing MAAS and KVM

1. Install apt packages
```
sudo apt update
sudo apt install maas libvirt-bin qemu-kvm virtinst
```

2. Add maas user to libvirtd group
```
sudo usermod -aG libvirtd maas
```

3. Restart MAAS to pick up the group change (I rebooted the host, but there's probably a better way)

4. Optional: clean up the default kvm network
```
sudo virsh net-undefine default
```

5. Create admin credentials to MAAS
```
sudo maas createadmin
```

6. Log into GUI with admin credentials at `http://<maas-host>/MAAS`

7. Configure upstream DNS (click Settings, scroll down to DNS)
Use the DNS from the host's `/etc/resolv.conf`. You might also need to disable
DNSSEC validation.

## Creating the virtual networks

Carefully review and run the create-network-bridges script in this repo.
Pay attention to which subnets belong to the virtual networks.

In the MAAS GUI, navigate to the Subnets section and create a space
(Add -> Add Space) for each virtual subnet.

Next, for each subnet, click "untagged" on the same row to enter configuration
for the corresponding VLAN. Enable DHCP, using the MAAS host's corresponding
bridge address as the gateway (e.g. for subnet 10.88.0.0/24, use 10.88.0.1).
Also add the network to its corresponding space here.

## Creating the VMs

Carefully review and run the create-vms script in this repo.

## Commissioning and configuring MAAS nodes

After running create-vms, you should hopefully see several nodes in the Nodes
section of MAAS.

You will need to match up the MAAS nodes with the KVM instances, by looking at
MAC addresses. Use `virsh list --all` to see the names of instances. For each
instance, do the following:
1. `virsh domiflist <instance-name>` to view MAC addresses.
2. Find the MAAS node with corresponding MAC addresses.
3. Rename the node in MAAS to match the instance name.
4. Set power configuration for the node:
  1. Power type: Virsh (virtual systems)
  2. Virsh address: qemu:///system
  3. Virsh VM ID: <instance-name>

Once you have configured all the nodes, you can select them in MAAS and run the
"Commission" action.

After commissioning completes, go into the node's configuration and set all of
the network interfaces to "Auto assign".

## Bootstrap the Juju controller

Use `juju add-cloud` and `juju add-credential` to add your MAAS info. Then,
bootstrap:
```
juju bootstrap my-maas my-maas --config test-mode=true
```

## Deploy CDK
See the bundle files in this repo.
