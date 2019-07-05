---
layout: post
title: "Building Cluster on NVIDIA Jetson TK2"
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

### On the clients

We need to run the following steps on all clients.

Ensure that the NFS client software is installed:

```bash
sudo apt install nfs-common
```

Mount the NFS in a new directory (replace `XXX.XXX.XXX.XXX` with node n°15's IP address):

```bash
sudo mkdir -p /mnt/cluster_data/cluster0
sudo mount -t nfs -o proto=tcp,port=2049 XXX.XXX.XXX.XXX:/exports /mnt/cluster_data/cluster0
```

## Install TensorFlow

We need to install TensorFlow 1.3, CUDA 8.0 and CuDNN 6.0 on every node. TensorFlow must be compiled with support for:

- GPU computing (Nvidia CUDA)
- Message Passing Interface (MPI), we use the MVAPICH implementation
- Hadoop File System (HDFS)

Since these options are not standard, and MPI and HDFS are not supported by the `tensorflow-gpu` package in pip, we need to compile TensorFlow ourselves. This is done with the installation scripts on the project's git repository:

```bash
git clone git@github.com:IADBproject/buildEmbeddedClusters.git
```

Run the installation scripts in the right order:

```bash
cd buildEmbeddedClusters/UTXJetson2/buildTensorFlow
./0-installPrerequisites.sh
./1-installBazel.sh
./2-buildAndInstallMVAPICH.sh
./3-installCUDA.sh
./4-buildAndInstallTensorFlow.sh
```

During the installation, you may be asked again for the node's administrator password. Since some parts (compilation of large codebases) take more than 15 minutes, this could happen several times.

After having ran all these scripts, CUDA and TensorFlow should be set up on the node.

This installation process will be automated in the future, with one script launching the installation on every nodes through the OOBM interface.

## Test the installation

This is a small example that you can run on a node to test that the installation is complete.

```python
#TODO
```
