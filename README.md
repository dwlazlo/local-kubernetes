# Local Kubernetes Instance

This repo is for the documentation and config for a local Kubernetes instance running on my home network.

It's not really intended for anyone else to use (of course this doesn't prevent any parts of it being useful!).

## Architecture

The hypervisor is KVM/Qemu running on Debian 11. 

- The hypervisor has a small ZFS pool which I will use for mounts within the VMs and for snapshots of the VM image.
- There is a pre-configured KVM bridge network on the hypervisor with the ID of `192.168.3.0/24`.

The Virtual Machines (VMs) for the Kubernetes cluster will be Rocky Linux Minimal install. 

- I have chosen Rocky for the VMs to introduce diversity into the environment (and my experience) as well as to learn about the built-in enterprise features of a distribution so closely associated with RHEL (I used to use Redhat in the early 2000s, so it will refresh that experience).

The cluster will have 3 nodes (1 `master` and 2 `worker` nodes).

## Network

- Router (pfSense): 192.168.3.1
- Hypervisor (bridge: `br0`): 192.168.3.3
- rocky-master: 192.168.3.200
- rocky-node01: 192.168.3.201
- rocky-node02: 192.168.3.202


The VMs are each created with the following command:

```bash
$ sudo virt-install \
--virt-type=kvm \
--hvm \
--name=rocky_master \
--ram=4096 \
--cpu=host \
--vcpus=4 \
--os-type=linux \
--os-variant=rhel8.4 \
--disk path=/pool0/kvm/rocky_master.qcow2,format=qcow2,bus=virtio,size=50 \
--filesystem type=mount,mode=mapped,source=/pool0/vmdata/rocky_master,target=vmdata \
--cdrom /var/lib/libvirt/isos/Rocky-8.5-x86_64-minimal.iso \
--disk /var/lib/libvirt/isos/debian_firmware-10.9.0-amd64-netinst.iso,device=cdrom \
--network bridge=br0,model=virtio \
--graphics vnc,listen=0.0.0.0

```

Note: change the `name` of the VM to something appropriate when creating the `worker` nodes.At the moment these VMs will be installed manually via VNC over the network bridge. 

In future a `cloud-image` could be used, or another automation solution like Terraform and Ansible together.

## Post-install configuration

I will be following the following tutorial for the installation:

- https://www.linuxtechi.com/install-kubernetes-cluster-on-rocky-linux/

### Network

RockyLinux specific: 
- Edit interfaces in `/etc/sysconfig/network-scripts/ifcfg-<IFACE_NAME>`
	- This file allows you to set the IP, DNS Server, DHCP/STATIC, automatically enabling IF on boot, etc
- Command to restart network manager service: `systemctl restart|status NetworkManager`

- Command to initialise interface is `nmcli connection up|show <IFACE_NAME>`

- May need manual route configuration with `ip route add default via 192.168.3.1`

#### /etc/hosts

So that each node in the cluster can find each other by name, run the following command as root on each VM:

```bash
cat << EOF >> /etc/hosts
192.168.3.200 rocky-master.local
192.168.3.201 rocky-node01.local
192.168.3.202 rocky-node02.local
EOF
```

### Mount host filesystems in guest

`9p` is not supported in the Rocky/RHEL kernel as far as I can tell. Will save this issue for later as I might need to recompile or source a non-standard kernel.

- [ ] resolve plan9 filesystem in Rocky

### Initial updates

`sudo yum update`

`sudo yum install vim`

### Disable Swap on each VM

**Perform these steps in each node:**

An easy way to perform repetitive tasks across multiple nodes is to open multiple terminals, one for each node, and then group them together so that typing commands can be done on all nodes at once. 

1. Open three terminals
2. SSH into each node, one node per terminal
3. Using the Terminator terminal emulator, right click for the menu and navigate to `grouping > new group`,  create a new broadcast group called `kubernetes` (you can call it whatever you like).
4. Right click on each terminal, navigate to `grouping` and select group `kubernetes`
5. Right click on each terminal, navigate to `grouping` and select `broadcast to group`

Now every command you type in one of these terminals will automaticaly be repeated for each member of the group.

<video autoplay="yes" controls="yes" type="video/webm" width="100%" src="kubernetes-group-terminator.webm"></video>

#### Disable swap

```bash
sudo swapoff -a
```

Comment the start of the `swap` entry in `etc/fstab`:

```bash
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

Reload systemd daemons:

```bash
sudo systemctl daemon-reload
```

Change Selinux policy to `permissive`
```bash
 sudo setenforce 0
 sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

### Firewall settings: Master node

These have to be run as separate commands:

```bash
sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --add-port=2379-2380/tcp
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=10251/tcp
sudo firewall-cmd --permanent --add-port=10252/tcp
sudo firewall-cmd --reload
sudo modprobe br_netfilter
sudo sh -c "echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables"
sudo sh -c "echo '1' > /proc/sys/net/ipv4/ip_forward"
```

### Firewall settings: Worker nodes

Consider using a group broadcast for each worker node for these commands:

```bash
sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=30000-32767/tcp                                                  
sudo firewall-cmd --reload
sudo modprobe br_netfilter
sudo sh -c "echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables"
sudo sh -c "echo '1' > /proc/sys/net/ipv4/ip_forward"
```

### Install Docker on all nodes (Master and Workers)

```bash
# install
sudo dnf config-manager --add-repo=https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install docker-ce -y
# enable
sudo systemctl start docker
sudo systemctl enable docker
```

