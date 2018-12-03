---
layout: post
title: "Install TensorFlow on NVIDIA Jetson TK1"
date: 2018-12-03
---

# Building TensorFlow for Jetson TK1:
We assume that latest JetPack was used to flash the Jetson Tk1.

## 0. Requirements:
```$ sudo apt-get install swig```

## 1. Java9:
the first step is to install Java9, but this is quite simple since Oracle provides a package:
```bash
$ sudo add-apt-repository ppa:webupd8team/java
$ sudo apt-get update
$ sudo apt-get install oracle-java8-installer
```

## 2. Protobuf:
Build and Install protobuf:

This need several other packages.:
```$ sudo apt-get install git zip unzip autoconf automake libtool curl zlib1g-dev```

Download the latest protobuf source from github:
```bash
$ sudo chown -R ubuntu:ubuntu /opt
$ cd /opt
$ git clone https://github.com/google/protobuf.git
```

First generate the configuration file and then run make:
```bash
$ cd /opt/protobuf
user@machine:/opt/protobuf$ ./autogen.sh
user@machine:/opt/protobuf$ ./configure --prefix=/usr
user@machine:/opt/protobuf$ make -j 4
user@machine:/opt/protobuf$ sudo make install
user@machine:/opt/protobuf$ protoc --version
libprotoc 3.4.0
```

## 2.1) Maven:
Install Maven java interface for building protobuf
```$ sudo apt-get install maven```

### Generate the protobuf-java-3.4.1.jar to Jetson Architecture:
```bash
user@machine:/opt/protobuf$ mvn -f java/pom.xml package
[INFO] --- maven-bundle-plugin:3.0.1:bundle (default-bundle) @ protobuf-java-util ---
[INFO] ...

user@machine:/opt/protobuf$ ls java/core/target/
protobuf-java-3.4.1.jar
```

