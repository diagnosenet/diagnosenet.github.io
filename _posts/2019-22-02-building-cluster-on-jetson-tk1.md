---
layout: post
title: "Building Cluster on NVIDIA Jetson TK1"
date: 2019-22-02
---

## Building Cluster on NVIDIA Jetson TK1

## 0. Requirements:

First, we need to add the Ubuntu's Universe and Multiverse repositories as they contain packages that many instructions assume. Universe contains community maintained software and Multiverse contains non-free software (licensing restrictions etc.)
```bash
sudo apt-add-repository universe
sudo apt-add-repository multiverse
```
Then, we need to upgrade the OS in the boards from Ubuntu 14.04.1 to 14.04.5 by running the following commands:
```bash
sudo apt-get update
sudo apt-get upgrade
```

Next, we need to install two packages: nano and OpenSSH. Nano is a terminal text editor needed to modify files directly on terminal. OpenSSH is a freely available version of the Secure Shell (SSH) protocol family of tools for remotely controlling or transferring files between computers. So on all the nodes, we run these two commands:
```bash
sudo apt-get install nano
sudo apt-get install openssh-server
```
On the master node, we install this package:
```bash
sudo apt-get install nfs-kernel-server
```
An on the workers node, we install this
```bash
sudo apt-get install nfs-common
```
MPICH2 is a freely available, portable implementation of MPI, a standard for message-passing for distributed-memory applications used in parallel computing. MPICH is supposed to be high-quality reference implementation of the latest MPI standard and the basis for derivative implementations to meet special purpose needs 
```bash
sudo apt-get install mpich2
```
Hydra is the default process manager. For the version Ubuntu 14.04.5, the default version is 1.3.x, so the configuration will follow the Hydra process manager.

## 1. Setting up static IP Addresses:
In order for us to establish a network between the cluster through Ethernet cables we will need to assign static IP Addresses to each of the nodes. To do this, we need to edit the interfaces file on each node. This can be done by doing the following:
```bash
sudo nano /etc/network/interfaces
```
Then modify the file as follow:
```bash
# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto eth0
iface lo inet loopback
address 192.168.137.10
netmask 255.255.255.0
gateway 192.168.137.1
dns-nameservers 192.168.137.1 8.8.8.8
```
To be able to distinguish between nodes without remembering there IP Addresses, we want to assign name on each node by changing the hostname file
```bash
sudo nano /etc/hostname
```
Here, I set the name for the master node is tegra1-c78, with c78 is the last three characters in its MAC address so that I can recognize them when working with the hardware. 
```bash
tegra1-c78
```
Similarly, I have the name for the three other nodes are:
```bash
tegra2-c3e
```
```bash
tegra3-c8f
```
```bash
tegra4-cdf
```
We need to do the same to change the name for other nodes. Then, in the hosts file of each node, we will add the static IP addresses of all the nodes so that to can connect to each other from any node.
```bash
sudo nano /etc/hosts
```
We need to change the content as below for all the boards. However, for the second line, we have to change the name in order to make sure it is the same hostname as the board we am modifying.
```bash
127.0.0.1 localhost
127.0.0.1 tegra1-c78
192.168.137.10 tegra1-c78
192.168.137.11 tegra2-c3e
192.168.137.12 tegra3-c8f
192.168.137.13 tegra4-cdf
```

To apply the changes, we have to reboot all the nodes using the command:
```bash
sudo reboot
```
After the boards turning on, on each node, we have to use the new static IP Address in order to remote them. We can notice that the hostnames have also changed. To check if our settings are applied by checking the resolv.conf to see if the nameservers are added. 
```bash
cat /etc/revolv.conf
```
We will see something like this
```bash
nameserver 192.168.137.1
nameserver 8.8.8.8
```
If the output is the same for all the nodes as above, we have changed the static IP addresses successfully.

## 2. Setting up SSH remote login

Since we want the cluster nodes to communicate with each other without having to ask for permission each time we will set up SSH (Secure Shell) remote login.
```bash
ssh-keygen -t rsa -b 2048
```
Now, we will copy the SSH ID to all the nodes including the head node.
```bash
ssh-copy-id tegra1-c78
ssh-copy-id tegra2-c3e
ssh-copy-id tegra3-c8f
ssh-copy-id tegra4-cdf
```
Then, we will need to build the known_hosts file in the .ssh directory. The known_hosts holds id of all the nodes in the cluster and allows password-less access to and from all the nodes in the cluster. To do this we need to create file with the name of all nodes in the .ssh folder and then use ssh keyscan to generate the known_hosts file.
```bash
cd .ssh
sudo nano name_of_hosts
```
We will save this file and then change its permissions in order for ssh-keyscan to be able to read the file. Ssh-keyscan is a utility for gathering the public ssh host keys of a number of hosts. It was designed to aid building and verifying ssh_known_hosts file. 
```bash
cd
sudo chmod 666 ~/.ssh/name_of_hosts
```
In this command, chmod 666 means that we will set the permission so that everyone can read and write this file.
The following command will then generate the known_hosts file:
```bash
ssh-keyscan -t rsa -f ~/.ssh/name_of_hosts >~/.ssh/known_hosts
```
In this case, -t specifies the type of the key to fetch from the scanned hosts, as we are using RSA key-pair so we put rsa after –t. Next, -f means the ssh-keyscan will read the hosts from a file, one per line .
Our last step for this setup will be to copy known_hosts, id_rsa public and private keys from the .ssh folder in the head node to the .ssh folder of all the other nodes. Public host keys are stored on and distributed to SSH clients, private keys are stored on SSH servers. We can do this using secure copy. Secure copy (scp) allows files to be copied to, from or between different hosts. It uses SSH for data transfer and provides the same authentication and same level of security as SSH. We use the following steps on the head node to do this:
```bash
cd .ssh
scp known_hosts id_rsa id_rsa.pub ubuntu@tegra1-c78:.ssh
scp known_hosts id_rsa id_rsa.pub ubuntu@tegra2-c3e:.ssh
scp known_hosts id_rsa id_rsa.pub ubuntu@tegra3-c8f:.ssh
scp known_hosts id_rsa id_rsa.pub ubuntu@tegra4-cdf:.ssh 
```
Now, we can remote other nodes using SSH protocol from any node in the cluster without password. 

## 3. Mount Network File System

The Network File System (NFS) mounting is crucial part of the cluster set up in order for all the nodes to have one common working directory. We are going to use the nfs-kernel-server and nfs-common which we had installed earlier. To do that, we will need to create the mount point for the HDD on the master node. Make sure that the mount point is not in the home directory of the ubuntu user. We chose to put it in / as follows:
```bash
sudo mkdir /media/cluster_files
```
Then we can check the drive by using the command
```bash
sudo fdisk –l
```
Information about the new connected device will display with name /dev/sda. We will need this to make the location.
/dev/ is the part of the unix directory tree that contains all device files, sda signifies the order of the device in which it was first found, so if there is a second device that is added into the system, it will be named as sdb. we have to set the new added device as type ext4 and mount it to the directory where we wish to share with other nodes.
```bash
sudo mkfs.ext4 /dev/sda -L cluster_files
```
Next, we will edit the exports file on the master node. This file will contain the information as to where we will be exporting the cluster_files directory on the slaves. 
```bash
sudo nano /etc/exports
```
Add the following line at the end:
```bash
/media/cluster_files *(rw,sync,no_subtree_check)
```
The /etc/exports file controls which file systems are exported to remote hosts and specifies options. It contains a table of local physical file systems on an NFS server that are accessible to NFS clients. The contents of the file are maintained by the server’s system administrator. In my setting, rw stands for the file system that is writable, sync means reply clients after data have been stored to stable storage and no_subtree_check will disable subtree checking, which has mild security implications 
We will need to permanently mount the external drive on the head node so that it automatically mounts itself in /media/cluster_files when the system is rebooted. To do this we will edit the fstab file on the master node by adding the following line at the end:
```bash
sudo nano /etc/fstab
```
```bash
/dev/sda1 /media/cluster_files ext4 defaults 0 0
```
The fstab file is used to define how disk partitions should be mounted into the file system, it contains the necessary information to automate the process of mounting partitions. Now, we can mount the external drive on the master node permanently.
```bash
sudo mount /dev/sda /media/cluster_files
```
The slave nodes will now need to mount this external drive and we will need to make the cluster_files directory and edit the fstab file on each of the slave nodes:
```bash
sudo mkdir /media/cluster_files
sudo nano /etc/fstab
```
Add in the end the following:
```bash
tegra1-c78:/media/cluster_files /media/cluster_files nfs rsize=8192,wsize=8192,timeo=14,intr
```
With rsize is the number of bytes NFS uses when reading files from an NFS server, wsize is the number of bytes NFE uses when writing files from an NFS server, timeo is the value in tenths of a second before sending the first retransmission and intr means the operations is not allowed to interrupt. Now our NFS mounting should be complete. In order to check this, we will first restart our cluster. Once the cluster is restarted, we will simply copy some files to this directory on master node and check on other that whether they have those files on this directory.

## 4. Message Passing Interface configuration

In order to setup Hydra, we need to create one file on the master node. This file contains all the host names of the compute nodes. We can create this file anywhere we want, but for simplicity we create it in the NFS shared folder.
```bash
cd /media/cluster_files
touch hosts
```
To be able to send out jobs to the other nodes in the network, add the host names of all compute nodes to the hosts file
```bash
192.168.137.10:4
192.168.137.11:4
192.168.137.12:4
192.168.137.13:4
```
We need to choose to include master in this file, which would mean that the master node would also act as a compute node. The hosts file only needs to be present on the node that will be used to start jobs on the cluster, usually the master node. But because the home directory is shared among all nodes, all nodes will have the hosts file.

## 5. Running MPICH2 example applications on the cluster
The MPICH2 package comes with a few example applications that users can run on their cluster. To obtain these examples, download the MPICH2 source package from the MPICH website and extract the archive to a directory. We will check by running an example code which is CPI. CPI is a c program that using MPICH2 library to display the number of processors that will be used to run it. We need to execute the following command to use all 16 processors of four Jetson Tegra K1 boards by using this command:4
```bash
sudo apt-get build-dep mpich2
wget http://www.mpich.org/static/downloads/1.4.1/mpich2-1.4.1.tar.gz
tar -xvzf mpich2-1.4.1.tar.gz
cd mpich2-1.4.1/
./configure
make
cd examples/
```
You can try with any source code you want, as in this case, I will try with CPI:
```bash
mpiexec –f hosts –n 16 bin/cpi
```
Up to now, we have already finished building a cluster with four Jetson Tegra K1 boards and be able to run some examples using MPI. 

## References:
Allan, A. (2015, August 26). Build a Compact 4 Node Raspberry Pi Cluster. Retrieved from https://makezine.com/projects/build-a-compact-4-node-raspberry-pi-cluster/

Arrow. (2017, September 17). Drive name? What is the correct term for the “sda” part of “/dev/sda”? Retrieved from Stack Exchange: https://unix.stackexchange.com/questions/392701/drive-name-what-is-the-correct-term-for-the-sda-part-of-dev-sda

Barney, B. (2018, July 6). Message Passing Interface (MPI). Retrieved from https://computing.llnl.gov/tutorials/mpi/

Building a Nvidia Jetson TK1 Cluster. (2017, August 17). Retrieved from http://selkie-macalester.org/csinparallel/modules/RosieCluster/build/html/#physical-cluster-set-up

ckimes. (2017, August 21). Introduction to fstab. Retrieved from Ubuntu: https://help.ubuntu.com/community/Fstab

Dalcin, L. (2017). MPI for Python. Retrieved from mpi4py: https://mpi4py.readthedocs.io/en/stable/

Embedded system. (2018, September 25). Retrieved from Wikipedia: https://en.wikipedia.org/wiki/Embedded_system

Example syntax for Secure Copy. (n.d.). Retrieved from Hypexr: http://www.hypexr.org/linux_scp_help.php

HILDENBRAND, J. (2014, August 3). A look at NVIDIA's Jetson TK1. Retrieved from androidcentral: https://www.androidcentral.com/look-nvidias-jetson-tk1

HOST KEY. (2017, August 3). Retrieved from SSH: https://www.ssh.com/ssh/host-key
japinator. (2015, December 29). 

Configure a static IP on Ubuntu Server 14.04. Retrieved from VSYSAD: http://www.vsysad.com/2015/12/configure-a-static-ip-on-ubuntu-server-14-04/

JetPack. (n.d.). Retrieved from NVIDIA Embedded Computing: https://developer.nvidia.com/embedded/jetpack

Jetson TK1. (2018, July 5). Retrieved from eLinux: https://elinux.org/Jetson_TK1

Kendall, W. (2018). MPI Broadcast and Collective Communication. Retrieved from MPI Tutorial: http://mpitutorial.com/tutorials/mpi-broadcast-and-collective-communication/

Kendall, W. (2018). MPI Scatter, Gather, and Allgather. Retrieved from MPI Tutorial: http://mpitutorial.com/tutorials/mpi-scatter-gather-and-allgather/

Kuan-Yu Yeh, H.-J. C.-D.-Y. (2016). Constructing a GPU Cluster Platform based on Multiple NVIDIA Jetson TKI. 2016 IEEE International Conference on Bioinformatics and Biomedicine (BIBM).

kulve. (2014, October 29). My Jetson focused Linux tips and tricks. Retrieved from NVIDIA Developer: https://devtalk.nvidia.com/default/topic/785551/embedded-systems/my-jetson-focused-linux-tips-and-tricks/

Li, A. &. (2014, November). An Overview of NVIDIA Tegra K1 Architecture. Retrieved from ResearchGate: https://www.researchgate.net/figure/NVIDIA-Tegra-K1-mobile-processor-32-bit-version_fig1_326920932

MASSIMILIANO. (2015, November 27). Building TensorFlow for Jetson TK1. Retrieved from CUDA MUSING: https://cudamusing.blogspot.com/2015/11/building-tensorflow-for-jetson-tk1.html

Mazieres, D., & Davison, W. (2018, March 5). ssh-keyscan. Retrieved from OpenBSD: https://man.openbsd.org/ssh-keyscan

MPICH . (n.d.). Retrieved from MPICH : https://www.mpich.org/

NVIDIA. (2018). Develop, Optimize and Deploy GPU-accelerated Apps. Retrieved from NVIDIA Developer: https://developer.nvidia.com/cuda-toolkit

NVIDIA. (2018). NVIDIA cuDNN. Retrieved from NVIDIA Developer: https://developer.nvidia.com/cudnn

Open MPI: Open Source High Performance Computing. (n.d.). Retrieved from Open MPI: https://www.open-mpi.org/

Pant, S. R. (2018, September 1). What is RSA? Retrieved from Quora: https://www.quora.com/What-is-RSA

Pereira, S. (2013, September 11). Building a simple Beowulf cluster with Ubuntu. Retrieved from https://www-users.cs.york.ac.uk/~mjf/pi_cluster/src/Building_a_simple_Beowulf_cluster.html

pl_rock. (2017, February 17). Failed to fetch http://ppa.launchpad.net [duplicate]. Retrieved from askubuntu: https://askubuntu.com/questions/879064/failed-to-fetch-http-ppa-launchpad-net

Slurm Workload Manager. (2013, March 6). Retrieved from SchedMD: https://slurm.schedmd.com/overview.html

Smith, C. (2006, May 2). Linux NFS-HOWTO. Retrieved from http://nfs.sourceforge.net/nfs-howto/ar01s02.html#whatis_nfs

THE /ETC/EXPORTS CONFIGURATION FILE. (n.d.). Retrieved from redhat: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/5/html/deployment_guide/s1-nfs-server-config-exports

