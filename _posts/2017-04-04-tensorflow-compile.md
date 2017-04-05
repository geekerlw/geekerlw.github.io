---
layout: post
title: "如何在Power8架构服务器上编译GPU加速的tensorflow"
author: "Steven Lee"
meta: "Springfield"
---

## 简介
　　Supervessel超能云是IBM中国研究院和中国系统与技术中心基于POWER架构和OpenStack技术共同构建的， 
支撑开发者进行远程研究开发的云平台 。它是IBM支撑OpenPOWER产业开放性发展的又一重要投入和贡献。   
　　Supervessel免费提供了配置了gpu加速器的docker镜像，源于目前市场镜像提供的gpu加速的tensorflow
环境中提供的tensorflow为0.5.0版本，且服务器的ppc64le架构不能通过pip install 安装tensorflow，所以
本教程从源码开始构建ppc64le架构v1.0.1版本的tensorflow。

## 配置java环境
    安装bazel依赖openjdk8
        sudo add-apt-repository ppa:openjdk-r/ppa  
        sudo add-apt-repository -y ppa:openjdk-r/ppa  
        sudo apt-get update  
        sudo apt-get install openjdk-8-jdk  
        sudo update-alternatives --set java /usr/lib/jvm/java-8-openjdk-ppc64el/jre/bin/java  
        export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-ppc64el 
    添加java环境变量到~/.bashrc
        export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-ppc64el
        export JRE_HOME=${JAVA_HOME}/jre
        export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
        export PATH=${JAVA_HOME}/bin:$PATH
  
## 编译安装protobuf
    tensorflow的编译使用bazel工具，而bazel的编译依赖于protobuf和grpc-java插件，  
    原镜像带的bazel版本为0.2.0，已经不满足tensorflow1.0.1版本的编译版本最低要求，  
    所以我们准备编译的bazel版本号为0.4.3，查看bazel/third_party/protobuf目录得知  
    protobuf版本依赖为3.0.0  
        git clone https://github.com/google/protobuf  
        cd protobuf  
        git checkout -b v3.0.0  
        ./autogen.sh  
        ./configure --prefix=/usr && make  
        sudo make install  
    
## 编译安装grpc-java plugin
    编译bazel还需要grpc-java plugin，查看bazel/third_party/grpc目录得知所需grpc  
    版本为0.15.0
        git clone https://github.com/grpc/grpc-java  
        cd grpc-java  
        git checkout -b v0.15.0  
    此处需要添加ppc64le架构支持  
    根据https://github.com/ibmsoe/grpc-java/commit/0b5ba9dace95b3cb75a60e471d5c5b1e329e2e77  
    在对应位置添加ppc64le架构支持
        cd compiler  
        ./gradlew java_pluginExecutable -Pprotoc=/usr/bin/protoc  
    生成的protoc-gen-grpc-java在compiler/build 目录  
    执行./gradlew install安装到/usr/bin  
    或者手动拷贝protoc-gen-grpc-java到/usr/bin/  
    
## 编译安装bazel
        git clone https://github.com/ibmsoe/bazel  
        cd bazel  
        git checkout v0.4.3-ppc
        export PROTOC=/usr/bin/protoc
        export GRPC-JAVA-PLUGIN=/usr/bin/protoc-gen-grpc-java
        ./compile.sh
        sudo cp output/bazel /usr/local/bin/bazel  

## 编译安装tensorflow
    安装tensorflow之前，请自行安装nvidia驱动，配置好cuda以及cudnn环境
        git clone https://github.com/tensorflow/tensorflow
        cd tensorflow
        git checkout -b v1.0.1  
        ./configure  
        bazel build -c opt --config=cuda --linkopt='-lrt' --local_resources 8096,.5,1.0 //tensorflow/tools/pip_package:build_pip_package  
        bazel-bin/tensorflow/tools/pip_package/build_pip_package /tmp/tensorflow_pkg  
        sudo pip install /tmp/tensorflow_pkg/tensorflow*.whl  

