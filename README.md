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
--filesystem /pool0/vmdata/rocky_master,/opt/vmdata \
--cdrom /var/lib/libvirt/isos/Rocky-8.5-x86_64-minimal.iso \
--disk /var/lib/libvirt/isos/debian_firmware-10.9.0-amd64-netinst.iso,device=cdrom \
--network bridge=br0,model=virtio \
--graphics vnc,listen=0.0.0.0

```

Note: change the `name` of the VM to something appropriate when creating the `worker` nodes.At the moment these VMs will be installed manually via VNC over the network bridge. 

In future a `cloud-image` could be used, or another automation solution like Terraform and Ansible together.

## Post-install configuration

### Network

RockyLinux specific: 
- Edit interfaces in `/etc/sysconfig/network-scripts/ifcfg-<IFACE_NAME>`

- Command to restart network manager service: `systemctl restart|status NetworkManager`

- Command to initialise interface is `nmcli connection up|show <IFACE_NAME>`

- May need manual route configuration with `ip route add default via 192.168.3.1`

#### /etc/hosts

```bash
sudo cat << EOF >> /etc/hosts
192.168.3.200 rocky-master
192.168.3.201 rocky-node01
192.168.3.202 rocky-node02
EOF
```

### Initial updates

`sudo yum update`

`sudo yum install vim`


