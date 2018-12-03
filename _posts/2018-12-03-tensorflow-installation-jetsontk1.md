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


### References:
[This tutorial is an update of Massimiliano blog for building TensorFlow on Jetson TK1.](https://cudamusing.blogspot.com/2015/11/building-tensorflow-for-jetson-tk1.html)
