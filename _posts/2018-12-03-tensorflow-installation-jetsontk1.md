---
layout: post
title: "Install TensorFlow on NVIDIA Jetson TK1"
date: 2018-12-03
---

## Building TensorFlow for Jetson TK1:
We assume that latest JetPack was used to flash the Jetson Tk1.

## 0. Requirements:
```bash
user@machine$ sudo apt-get install swig
user@machine$ sudo apt-get install build-essential gfortran libatlas-base-dev python-pip python-dev
```

## 1. Java8:
the first step is to install Java9, but this is quite simple since Oracle provides a package:
```bash
user@machine$ sudo add-apt-repository ppa:webupd8team/java
user@machine$ sudo apt-get update
user@machine$ sudo apt-get install oracle-java8-installer
```

## 2. Protobuf:
Build and Install protobuf:

This need several other packages:
```bash
user@machine$ sudo apt-get install git zip unzip autoconf automake libtool curl zlib1g-dev
```

Download the latest protobuf source from github:
```bash
user@machine$ sudo chown -R ubuntu:ubuntu /opt
user@machine$ cd /opt
user@machine:/opt/$ git clone https://github.com/google/protobuf.git
```

First generate the configuration file and then run make:
```bash
user@machine$ cd /opt/protobuf
user@machine:/opt/protobuf$ ./autogen.sh
user@machine:/opt/protobuf$ ./configure --prefix=/usr
user@machine:/opt/protobuf$ make -j 4
user@machine:/opt/protobuf$ sudo make install
user@machine:/opt/protobuf$ protoc --version
libprotoc 3.4.0
```

## 2.1. Maven:
Install Maven java interface for building protobuf
```user@machine$ sudo apt-get install maven```

### Generate the protobuf-java-3.4.1.jar to Jetson Architecture:
```bash
user@machine:/opt/protobuf$ mvn -f java/pom.xml package
[INFO] --- maven-bundle-plugin:3.0.1:bundle (default-bundle) @ protobuf-java-util ---
[INFO] ...

user@machine:/opt/protobuf$ ls java/core/target/
protobuf-java-3.4.1.jar
```

## 3. Bazel:
Install bazel to compile TensorFlow 0.8.0:
```bash
user@machine$ cd /opt
user@machine:/opt$  git clone https://github.com/bazelbuild/bazel.git
user@machine:/opt$ cd bazel
user@machine:/opt$ git checkout tags/0.1.4
```

Copy the files generated to Jetson architecture:
```bash
user@machine:/opt/bazel$ cp /usr/bin/protoc  third_party/protobuf/protoc-linux-arm32.exe
user@machine:/opt/bazel$ cp ../protobuf/java/core/target/protobuf-java-3.4.1.jar third_party/protobuf/protobuf-java-3.0.0-beta-1.jar
user@machine:/opt/bazel$ rm third_party/protobufprotobuf-java-3.0.0-beta-1.jar
```

Compile Bazel:
```bash
user@machine:/opt/bazel$ ./compile.sh
Build successful! Binary is here: /opt/bazel/output/bazel

user@machine:/opt/bazel$ sudo cp output/bazel /usr/local/bin/
user@machine:/opt/bazel$ ls /usr/local/bin/bazel
/usr/local/bin/bazel
```

## 4. Add Swap Memory:
Plug usb memory and use it as swap
```bash
user@machine$ sudo fdisk -l
    (check the device name of usb memory. This time, the name is /dev/sda1)  
user@machine$ sudo umount /dev/sda1     
    (This is not typo. "umount" is correct.)  
user@machine$ sudo mkswap /dev/sda1
    (There are some outputs.)
user@machine$ sudo swapon /dev/sda1
    (There are some outputs.)
```

## 5. Install CUDA 6.5, cuDNN v2 and CUDA-7.0:
### 5.1 Install CUDA 6.5:
```bash
user@machine$ wget http://developer.download.nvidia.com/compute/cuda/6_5/rel/installers/cuda-repo-l4t-r21.2-6-5-prod_6.5-34_armhf.deb
```

Jetson TK1 Version
```bash
user@machine$ sudo dpkg -i cuda-repo-l4t-r21.2-6-5-prod_6.5-34_armhf.deb   
user@machine$ sudo apt-get update 
user@machine$ sudo apt-get install cuda-toolkit-6-5
```

Check that nvcc is working:
```bash
user@machine:/usr/local/cuda-6.5$ sudo chown -R ubuntu:ubuntu samples/
user@machine:/usr/local/cuda-6.5$ cd samples/0_Simple/matrixMul
user@machine:/usr/local/cuda-6.5/samples/0_Simple/matrixMul$ make
user@machine:/usr/local/cuda-6.5/samples/0_Simple/matrixMul$ ./matrixMul
[Matrix Multiply Using CUDA] - Starting...
GPU Device 0: "GK20A" with compute capability 3.2

MatrixA(320,320), MatrixB(640,320)
Computing result using CUDA Kernel...
done
Performance= 21.69 GFlop/s, Time= 6.044 msec, Size= 131072000 Ops, WorkgroupSize= 1024 threads/block
Checking computed result for correctness: Result = PASS
```

### 5.2 Install cuDNN library:
Download the cuDNN: https://developer.nvidia.com/cudnn
```bash
user@machine:~/Downloads$ tar -xzvf cudnn-8.0-linux-x64.v5.1.tgz
```

Check or copy the cuDNN files into the cuda-6.5:
```bash
user@machine:~/Downloads$ cd cudnn/cudnn-6.5-linux-ARMv7-v2
user@machine:~/cudnn/cudnn-6.5-linux-ARMv7-v2$ sudo cp cudnn.h /usr/local/cuda-6.5/include
user@machine:~/cudnn/cudnn-6.5-linux-ARMv7-v2$ sudo cp libcudnn* /usr/local/cuda-6.5/lib
```

### 5.3 Install CUDA 7.0:
We need the cuda-7.0 SDK to generate the object files and compile with cuda-6.5.
```bash
user@machine:~/Downloads/$ wget http://developer.download.nvidia.com/embedded/L4T/r24_Release_v1.0/CUDA/cuda-repo-l4t-7-0-local_7.0-76_armhf.deb
```
Now install it as usual:
```bash
user@machine:~/Downloads/$ sudo dpkg -i cuda-repo-l4t-7-0-local_7.0-76_armhf.deb 
user@machine$ sudo apt-get update
user@machine$ sudo apt-get install cuda-toolkit-7-0
user@machine$ cd /usr/local   
user@machine:/usr/local$ sudo rm cuda   
user@machine:/usr/local$ sudo ln -s cuda-6.5/ cuda
```
Add path to .bashrc
```bash
user@machine$ echo "export CPAHT=/usr/local/cuda/include:$CPATH" >> ~/.bashrc   
user@machine$ echo "export PAHT=/usr/local/cuda-6.5/bin:$PATH" >> ~/.bashrc   
user@machine$ echo "export LD_LIBRARY_PATH=/usr/local/cuda-7.0/lib:$LD_LIBRARY_PATH" >> ~/.bashrc   
user@machine$ source ~/.bashrc 
```

Check the CUDA compiler versions
```bash
user@machine$ nvcc -V
nvcc: NVIDIA (R) Cuda compiler driver
Copyright (c) 2005-2014 NVIDIA Corporation
Built on Tue_Feb_17_22:53:16_CST_2015
Cuda compilation tools, release 6.5, V6.5.45

user@machine$ /usr/local/cuda-7.0/bin/nvcc -V
nvcc: NVIDIA (R) Cuda compiler driver
Copyright (c) 2005-2015 NVIDIA Corporation
Built on Mon_Feb_22_15:38:26_CST_2016
Cuda compilation tools, release 7.0, V7.0.74
```

## 6. Build and Install Tensorflow 0.8.0
This need several other packages:
```bash
user@machine$ sudo apt-get install libcurl3-dev swig  python-numpy python-dev 
```

Download and switchs to tensorflow 0.8
```bash
user@machine:/opt$ git clone --recurse-submodules https://github.com/tensorflow/tensorflow
user@machineu:/opt$ cd tensorflow
user@machine:/opt/tensorflow$ git checkout r0.8
Branch r0.8 set up to track remote branch r0.8 from origin.
Switched to a new branch 'r0.8'
```

TensorFlow is expecting a 64bit system, we will need to change all the reference from lib64 to lib.
We can find all the files with the strings and apply all the changes with these commands:
```bash
user@machine:/opt$ cd tensorflow
user@machine:/opt/tensorflow$ grep -Rl "lib64"| xargs sed -i 's/lib64/lib/g'
user@machine:/opt/tensorflow$ grep -Rl "so.7.0"| xargs sed -i 's/so\.7\.0/so\.6\.5/g' //Not used
```



### References:
[This tutorial is an update of Massimiliano blog for building TensorFlow on Jetson TK1.](https://cudamusing.blogspot.com/2015/11/building-tensorflow-for-jetson-tk1.html)
