# Local Kubernetes Instance

This repo is for the documentation and config for a local Kubernetes instance running on my home network.

It's not really intended for anyone else to use (of course this doesn't prevent any parts of it being useful).

## Architecture

The hypervisor is KVM/Qemu running on Debian 11. 

- The hypervisor has a small ZFS pool which I will use for mounts within the VMs and for VM snapshots.

The Virtual Machines (VMs) for the Kubernetes cluster will be Rocky Linux Minimal install. 

- I have chosen Rocky for the VMs to introduce diversity into the environment (and my experience) as well as to learn about the built-in enterprise features of a distribution so closely associated with RHEL (the de-facto standard for enterprise Linux).

The cluster will have 3 nodes (1 `master` and 2 `worker` nodes).

These are each created with the following command:

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
--cdrom /var/lib/libvirt/isos/Rocky-8.5-x86_64-minimal.iso \
--disk /var/lib/libvirt/isos/debian_firmware-10.9.0-amd64-netinst.iso,device=cdrom \
--network bridge=br0,model=virtio \
--graphics vnc,listen=0.0.0.0

```

!!! note
	change the `name` of the VM to comething appropriate when creating the `worker` nodes.

