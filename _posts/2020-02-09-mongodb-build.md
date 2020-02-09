---
title: centos 7 源码编译MongoDB
---

# 源码编译mongodb
> 编译方法适用于Mongodb和percona-mongodb-server

```bash
#添加scl安装源
sudo yum -y install centos-release-scl
#安装gcc8
sudo yum -y install devtoolset-8-gcc devtoolset-8-gcc-c++
#安装所依赖的开发库
sudo yum -y install libcurl-devel openssl-devel
#安装scons和依赖的python library
sudo pip3 install scons -i http://mirrors.aliyun.com/pypi/simple --trusted-host mirrors.aliyun.com
sudo pip3 install psutil -i http://mirrors.aliyun.com/pypi/simple --trusted-host mirrors.aliyun.com
sudo pip3 install -r etc/pip/compile-requirements.txt -i http://mirrors.aliyun.com/pypi/simple --trusted-host mirrors.aliyun.com
#编译mongod, 可以选择其它的编译target, 需要手工指定gcc和g++路径
scons -j2  CC=/opt/rh/devtoolset-8/root/usr/bin/gcc CXX=/opt/rh/devtoolset-8/root/usr/bin/g++ MONGO_VERSION=4.2.2 mongod
```

验证编译后的程序可以正常运行

```bash
./mongod --dbpath /tmp/mongodb/
```
