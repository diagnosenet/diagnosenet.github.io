---
layout: post
title: "Compiling TensorFlow on an NVIDIA Jetson TX2"
date: 2019-07-11
---

This is a guide to creating a Python wheel for TensorFlow 1.13.1 on a Jetson TX2, with support for

- Hadoop File System (HDFS)
- Message Passing Interface (MPI)
- GPU computation (Nvidia CUDA)

The resulting wheel file wan be used in the [guide for building a Jetson TX2 cluster](2019-07-02-building-cluster-on-jetson-tx2.md).

**This guide must be followed on a Jetson TX2 machine.**
Addresses and passwords for the machines at I3S can be found in the private documentation.

## Install the prerequisites

### Python 3.6

We need Python 3.6 to install this version, but the TX2s run Ubuntu 16.04 which contains only Python 3.5.
We will use [a PPA](https://launchpad.net/~deadsnakes/+archive/ubuntu/ppa):

```bash
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt update
sudo apt install python3.6 python3.6-dev
```

### The TensorFlow build prerequisites

#### pip packages

Don't mix up the Python versions here, the command `python3` still leads to Python 3.5, which we don't want to use.

```bash
python3.6 -m pip install -U --user pip six numpy wheel setuptools mock future>=0.17.1
python3.6 -m pip install -U --user keras_applications==1.0.6 --no-deps
python3.6 -m pip install -U --user keras_preprocessing==1.0.5 --no-deps
```

#### Bazel

Bazel is TensorFlow's build system.
We need Bazel 0.19.2, which is the [tested version](https://www.tensorflow.org/install/source#linux) for TensorFlow 1.13.1.
Since there is no official build for aarch64 (Jetson TX2's CPU architecture), we have to build Bazel from source.

##### Prerequisites

```bash
sudo apt install build-essential openjdk-8-jdk python zip unzip
```

##### Compilation

```bash
wget https://github.com/bazelbuild/bazel/releases/download/0.19.2/bazel-0.19.2-dist.zip
mkdir bazel
cd bazel
unzip ../bazel-0.19.2-dist.zip
env EXTRA_BAZEL_ARGS="--host_javabase=@local_jdk//:jdk" bash ./compile.sh
```

Note: bazel could fail with this error:

```none
# A fatal error has been detected by the Java Runtime Environment:
#
#  SIGILL (0x4) at pc=0x0000007f882df040, pid=26577, tid=0x0000007f94640200
```

This [seems to be a bug in the JDK](https://bugs.openjdk.java.net/browse/JDK-8219698), which is not fixed in our version.
It happens randomly.
Launch the compilation again until it works (usually only two tries are necessary).

*To be continued.*
