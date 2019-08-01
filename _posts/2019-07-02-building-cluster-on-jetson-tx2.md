---
layout: post
title: "Building Cluster on NVIDIA Jetson TX2"
date: 2019-07-02
---

This is a guide to setting up the Jetson array for our experiments. We use TensorFlow 1.3 with CUDA 8.0 and CuDNN 6.0, storing and sharing experiment data with the Network File System (NFS).

## Description of the hardware

### The nodes

We are working on a Nvidia Jetson TX2 array, containing 24 nodes, each with its own CPU, GPU and storage. The CPU architecture is ARM (aarch64), which mush be taken into account when building and installing software.

Every node runs an Ubuntu 16.04 operating system and has Internet access. They are independent machines and must be configured separately.

### The storage

Each node owns 30GB of local storage, represented by the block device `/dev/mmcblk0`.

In addition, nodes numbered 7, 15 and 23 each have access to one 1TB SATA SSD, through their block device `/dev/sda`. We will share these three SSD with the Network File System, to allow every node to use this storage.

### The management interface

All nodes are connected by serial port to an out-of-band management (OOBM) interface. Through it, it is possible to control all the nodes with the program `minicom`, as described in the next sections.

The OOBM interface is controlled by an Ubuntu 14.04 operating system. Like the nodes, it is accessible by SSH and has Internet access.

## Connect to the array

### Connect to the lab's network

The array can be controlled by SSH. It is **only accessible through the I3S virtual private network**. Connect to the VPN before continuing. If you don't have access to the VPN yet, contact I3S's IT staff.

### Getting the addresses and passwords

IP addresses referenced in this document are local to the I3S network and **not visible on this wiki** (replaced with `XXX.XXX.XXX.XXX`). You can find them in the private documentation, along with the passwords to the various user accounts.

### Access the nodes through the network

There are **two ways** of accessing the nodes: through the management interface, or directly by SSH with each node's IP address.

#### The OOBM interface

The out-of-band management interface is accessible _via_ SSH:

```bash
ssh utx@XXX.XXX.XXX.XXX
```

Once logged into this interface, you can open a shell on each node, using the serial port with the `minicom` program:

```bash
sudo minicom -D /dev/ttyUSB$X
```

Where `$X` must be replaced with the number of the node (0 through 23). You will again be asked for the OOBM account's password.

The minicom shell cannot be closed as usual with the `exit` command, it will just restart. To exit minicom and go back to the OOBM shell, use the shortcut `Ctrl+A`, then `X` and `Enter`.

#### The individual nodes

Every node in the array has a specific IP address on the I3S network. You can access it with SSH:

```bash
ssh nvidia@XXX.XXX.XXX.XXX
```

The `nvidia` user has sudo rights.

## Set up the first Network File System

The installation of CUDA and TensorFlow in the next section requires a large installation package (1.3 GB). To make the process efficient and avoid overloading the Internet link, we will set up NFS on the SSD owned by node 15, and copy the package on that filesystem, so that the other nodes can efficiently download it through the array's internal links.

### On the server

#### Setting up the NFS

Connect to node 15, which will be the master for this NFS:

```bash
ssh nvidia@XXX.XXX.XXX.XXX
```

Install NFS:

```bash
sudo apt install nfs-kernel-server
```

Check that the NFS kernel module is working by running the `lsmod` command. The output should be similar to:

```bash
Module                  Size  Used by
nfsd                  273557  11
auth_rpcgss            51022  1 nfsd
oid_registry            3359  1 auth_rpcgss
nfs_acl                 3418  1 nfsd
pci_tegra              72709  0
```

You can also check that the SSD is present with `lsblk`. The output should contain this line:

```bash
NAME         MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda            8:0    0 931.5G  0 disk
```

Format the SSD:

```bash
sudo mkfs.ext4 /dev/sda -L cluster_files
```

Create the mountpoint and modify `fstab` to mount the SSD on it automatically, then mount it:

```bash
sudo mkdir /exports
echo '/dev/sda /exports ext4 defaults 0 2' | sudo tee -a /etc/fstab
sudo mount /exports
```

Add the mountpoint to `/etc/exports` to make it accessible by the clients. In the next command, replace `XXX.XXX.XXX` with the 3 first segments of the nodes' IP addresses.

```bash
echo '/exports XXX.XXX.XXX.0/24(rw,fsid=0,insecure,no_subtree_check,async)' | sudo tee -a /etc/exports
```

Finally, restart the NFS server to apply the changes:

```bash
sudo systemctl restart nfs-kernel-server.service
```

#### Copying the TensorFlow and CUDA installation package

Get the installation package from the link under "Installation package location" in the private documentation. Copy it to node 15 through SSH (the following command must be ran on the machine that downloaded the package; replace `XXX.XXX.XXX.XXX` with the address of node 15):

```bash
scp path_to_download/UTXJetson2_install_packages.tar.xz nvidia@XXX.XXX.XXX.XXX:
```

On the Jetson node n°15, extract the package and remove the archive:

```bash
tar xf UTXJetson2_install_packages.tar.xz
rm UTXJetson2_install_packages.tar.xz
sudo mv UTXJetson2_install_packages /exports/
```

The installation data is now available to all the nodes.

## Launch the installation

The rest of the installation process is automated. Install the prerequisites on your local Linux machine:

```bash
sudo apt install sshpass git
```

Clone and launch the installation scripts:

```bash
git clone https://github.com/IADBproject/buildEmbeddedClusters.git
cd buildEmbeddedClusters/UTXJetson2/installWithoutCustomTF/
bash global_scripts/installAllNodes.sh
```

The installation process will do the following steps on every node:

- create a new user, `mpiuser`
- create an SSH key (if needed), then copy it on all the other nodes with `ssh-copy-id` (marking them as known hosts in the process)
- if the node is an NFS master, set up an NFS filesystem and make it accessible to all the nodes
- mount the three NFS filesystems in the `/home/mpiuser/cloud/{0,1,2}` directories
- install Python 3.6, CUDA 8.0, cuDNN 6.0 and a CUDA-enabled TensorFlow 1.3 wheel, along with the other Diagnosenet prerequisites
- change the hostname to `astroX`, where X is the node's number
- add all the nodes in `/etc/hosts` as `astro0` to `astro23`
- give special `sudo` permissions to `mpiuser`, to launch some commands related to Diagnosenet as root without entering a password

At this point, CUDA and TensorFlow should be set up on the node and ready to run experiments.

## Test the installation

This is a small example that you can run on a node to test that the installation is complete.

```python
#TODO
```
