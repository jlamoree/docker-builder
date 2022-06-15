# Docker Builders on Ubuntu 22.04 LTS in Proxmox

Create a Docker environment to build amd64 containers in a home lab running Proxmox VE 7.x.

## Prerequisite

The following assumes that DHCP is configured with reservations for the builder nodes (if you want more than one). The following scheme is used to organize the hostnames and MAC addresses:

| Hostname          | MAC Address       |
|-------------------|-------------------|
| docker-builder-01 | 00:00:00:00:01:21 |
| docker-builder-02 | 00:00:00:00:01:22 |
| docker-builder-0x | 00:00:00:00:01:2x |

Before spinning up the VM(s), create the reservations in advance so everybody has a good time.

## Proxmox Setup
The following are common variables for subsequent scripts, to be executed on a Proxmox cluster node.

```shell
pve_image_storage=/locker/images
ubuntu_cloud_image_base=https://cloud-images.ubuntu.com/jammy/current
dist_image=jammy-server-cloudimg-amd64-disk-kvm.img
cloud_init_user_sshkey_local="/tmp/docker-builder-project-user-sshkey"

# Fetch the latest cloud image (unless it exists)
test -r "${pve_image_storage}/${dist_image}" || curl -o "${pve_image_storage}/${dist_image}" "${ubuntu_cloud_image_base}/${dist_image}"

# Put the desired SSH key in place for the clout-init user; used later
curl -s -o "$cloud_init_user_sshkey_local" https://lamoree.com/joseph@lamoree.com.pub

# Create the builder VMs as needed, incrementing the vmid, vmname, and vmmac for each
vmid=401
vmname=docker-builder-01
vmmac=00:00:00:00:01:21
storage_sys=pool1
qm create $vmid --name $vmname --memory 4096 --cores 2 --net0 virtio,bridge=vmbr0,macaddr=$vmmac
qm importdisk $vmid "${pve_image_storage}/${dist_image}" $storage_sys
qm set $vmid --scsihw virtio-scsi-pci --scsi0 $storage_sys:vm-$vmid-disk-0
qm set $vmid --boot c --bootdisk scsi0
qm set $vmid --ide2 local-lvm:cloudinit
qm set $vmid --serial0 socket --vga serial0
qm set $vmid --agent enabled=1
qm set $vmid --ipconfig0 ip=dhcp
qm resize $vmid scsi0 16G
qm set $vmid --sshkeys "${cloud_init_user_sshkey_local}"
qm start $vmid
```

## Ansible Playbook

Add the new instance(s) to the Ansible inventory and update `~/.ssh/config`.

Let the fresh VM update and reboot, then run the playbook:
```shell
ssh docker-builder-01 'sudo apt update; sudo apt upgrade -y; sudo shutdown -r now'
ansible-playbook docker-builder.yaml
```

## Docker Buildx Setup

Add the local builder for Apple M1 hardware:
```shell
docker buildx create \
  --name builders \
  --node apple_mac_studio \
  --platform linux/arm64,linux/arm/v7,linux/arm/v6 \
  --driver-opt env.BUILDKIT_STEP_LOG_MAX_SIZE=10000000 \
  --driver-opt env.BUILDKIT_STEP_LOG_MAX_SPEED=10000000

docker buildx create \
  --name builders \
  --append \
  --node docker_builder_01 \
  --platform linux/amd64,linux/386 \
  ssh://docker-builder-01 \
  --driver-opt env.BUILDKIT_STEP_LOG_MAX_SIZE=10000000 \
  --driver-opt env.BUILDKIT_STEP_LOG_MAX_SPEED=10000000
```

Then use the configuration and bootstrap it:
```shell
docker buildx use builders
docker buildx inspect --bootstrap
```

Finally, build something for both architectures:
```shell
cd ~/projects/project
docker login
docker buildx build --platform=linux/amd64,linux/arm64 --push \
  --tag username/project:1.33.7 \
  --tag username/project:latest .
```
