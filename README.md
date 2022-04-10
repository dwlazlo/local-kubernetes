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

<video autoplay="yes" controls="yes" type="video/webm" width="100%" src="./kubernetes-group-terminator.webm"></video>

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

### Install the container runtime (`CRI-O`)

- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

I will be installing the `cri-o` container runtime instead of `docker-engine`.

Ensure that the required system settings and modules are persistent:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/crio.conf
overlay
br_netfilter
EOF
```

enable the modules:

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

Enable ipv4 forwarding and bridge:

```bash
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```

and enable the system settings:

```bash
sudo sysctl --system
```

The following guides mention CentOS, which is functionally equivalent with Rocky Linux, so I will follow the CentOS instructions.

https://kubernetes.io/docs/setup/production-environment/container-runtimes/#cri-o

https://computingforgeeks.com/install-cri-o-container-runtime-on-rocky-linux-almalinux/

Set `$VERSION` of `cri-o` to match installed kubernetes version.

```
https://github.com/cri-o/cri-o/releases
```

```bash
VERSION=1.23
```
and set `$OS` to `CentOS_8`

```bash
OS=CentOS_8
```

Then run the following commands:

```bash
sudo curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable.repo https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/devel:kubic:libcontainers:stable.repo
sudo curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.repo https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$VERSION/$OS/devel:kubic:libcontainers:stable:cri-o:$VERSION.repo
sudo dnf install cri-o cri-tools
```

Enable and confirm crio is running:

```bash
sudo systemctl daemon-reload
sudo systemctl enable crio --now
sudo systemctl status crio
```


### All Nodes: Enable Kubernetes Repo

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF
```

### All Nodes:  Install base Kubernetes programs

```bash
sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
```

### All Nodes:  Enable `Kubelet` on all nodes

```bash
sudo systemctl enable --now kubelet
```

Note:
-  as per the official documentation, `The kubelet is now restarting every few seconds, as it waits in a crashloop for kubeadm to tell it what to do.`


### Master node: kubeadm init

On the master node, initialise `kubeadm`, performing a `dry-run` first to check outputs and any errors.

The `--pod-network-cidr` argument is based on the `flannel` CNI plugin requirements below. Before performing the `kubeadm init` command, ensure that you have chosen a CNI plugin and can incorporate any requirements in the `init` stage. 

```bash
sudo kubeadm init --dry-run --cri-socket=/var/run/crio/crio.sock --pod-network-cidr=10.244.0.0/16
```

If no errors then perform the `init` command

```bash
sudo kubeadm init --cri-socket=/var/run/crio/crio.sock --pod-network-cidr=10.244.0.0/16
```

Make note of any commands suggested in the output, especially the `kubeadm join` command, to be run on each node. This ensures that the worker nodes use the appropriate PKI settings to authenticate with the master node.


The `kubeadm init` output will look something like this:

```bash
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a Pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  /docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

Pay attention to any recommendations, for example the commands to start using the cluster as a regular user but copying the config to your home directory.

If you prefer to run as root you can also run

`export KUBECONFIG=/etc/kubernetes/admin.conf`

### Container Network Interface (CNI)

The CNI provides the networking between containers on the various nodes.

You will need to choose and install a CNI plugin: I will be installing `flannel`.

You can install the CNI plugin using

`kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml`

this will create the required objects for `flannel` to run:

```
podsecuritypolicy
clusterrole
clusterrolebinding
serviceaccount
configmap
daemonset
```

Check that everything is running by 

```
kubectl get all -n kube-system
```

