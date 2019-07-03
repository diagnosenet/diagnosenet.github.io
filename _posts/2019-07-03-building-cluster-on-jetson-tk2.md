# Setting up the Jetson array

This is a guide to setting up the Jetson array for our experiments. We use TensorFlow 1.3 with CUDA 8.0 and CuDNN 6.0.

## Description of the hardware

We are working on a Nvidia Jetson TX2 array, containing 24 nodes, each with its own CPU, GPU and storage. The CPU architecture is ARM (aarch64), which mush be taken into account when building and installing software. Every node in the array is an independent machine and must be configured separately.

## Connect to the array

### Connect to the lab's network

The array can be controlled by SSH. It is only accessible through the I3S virtual private network. Connect to the VPN before continuing. If you don't have access to the VPN yet, contact I3S's IT staff.

### Access the nodes through the network

There are **two ways** of accessing the nodes: through the management interface, or directly with each node's IP address. The easiest way is through the OOBM interface, since it only requires one SSH connection and one IP address.

#### The OOBM interface

The out-of-band management interface is accessible _via_ SSH:

```bash
ssh utx@134.59.132.40
```

The password is `IADBproject2019`.

Once logged into this interface, you can open a shell on each node _via_ the serial port with the `minicom` program:

```bash
sudo minicom -D /dev/ttyUSB$X
```

Where `$X` must be replaced by the number of the node (0 through 23). You will again be asked for the account's password, `IADBproject2019`.

The minicom shell cannot be closed as usual with the `exit` command, it will just restart. To exit minicom and go back to the OOBM shell, use the shortcut `Ctrl+A`, then `X` and `Enter`.

#### The individual nodes

Each node in the array also has a specific IP address. The first node (numbered 0) can be accessed with this command:

```bash
ssh nvidia@134.59.132.206
```

The password for every node is `nvidia`.

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

During the installation, you may be asked again for the node's administrator password, which is `nvidia`. Since some parts (compilation of large codebases) take more than 15 minutes, this could happen several times.

After having ran all these scripts, CUDA and TensorFlow should be set up on the node, and ready to run our experiments.

### Test the installation

This is a small example that you can run on a node to test that the installation is complete.

```python
#TODO
```
