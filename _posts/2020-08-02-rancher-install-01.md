---
title: "Rancher安装入坑指南"
date: 2020-08-02 16:04:00 +0800
author: pengxu
categories: rancher
---

# Rancher安装入坑指南

>  本文介绍如何利用virtualbox来搭建由三台k8s节点组成的rancher集群。

经过几番折腾，在virtualbox环境下，搭建成功rancher集群，把步骤详细记录一下，后续大概率还会用上。

##  环境准备

以下所列机器统一使用centos 7.8, 可以到osboxes.org直接下载os镜像，减轻安装成本

osboxes.org上提供的镜像默认用户名/密码 **osboxes/osboxes.org**

.机器列表

| ip   | hostname | 用途 |
| ---- | -------- | ---- |
|192.168.56.101|k8s-node-00|k8s节点1|
|192.168.56.102|k8s-node-01|k8s节点2|
|192.168.56.103|k8s-node-02|k8s节点3|
|192.168.56.110|k8s-nginx|外部访问rancher webui的入口|

## nginx节点安装

安装nginx

```bash
sudo yum -y install epel-release
sudo yum -y install nginx
```

修改nginx配置文件, */etc/nginx/nginx.conf*

```bash
load_module /usr/lib64/nginx/modules/ngx_stream_module.so;
user nginx;
worker_processes 4;
worker_rlimit_nofile 40000;
events {
    worker_connections 8192;
}
http {
    # Gzip Settings
    gzip on;
    gzip_disable "msie6";
    gzip_disable "MSIE [1-6]\.(?!.*SV1)";
    gzip_vary on;
    gzip_static on;
    gzip_proxied any;
    gzip_min_length 0;
    gzip_comp_level 8;
    gzip_buffers 16 8k;
    gzip_http_version 1.1;
    gzip_types text/xml application/xml application/atom+xml application/rss+xml application/xhtml+xml image/svg+xml application/font-woff text/javascript application/javascript application/x-javascript text/x-json application/json application/x-web-app-manifest+json text/css text/plain text/x-component font/opentype application/x-font-ttf application/vnd.ms-fontobjectfont/woff2 image/x-icon image/png image/jpeg;
    server {
        listen         80;
        return 301 https://$host$request_uri;
    }
}
stream {
    upstream rancher_servers {
        least_conn;
        server 192.168.56.101:443 max_fails=3 fail_timeout=5s;
        server 192.168.56.102:443 max_fails=3 fail_timeout=5s;
        server 192.168.56.103:443 max_fails=3 fail_timeout=5s;
    }
    server {
        listen     443;
        proxy_pass rancher_servers;
    }
}
```

.启动nginx

```bash
sudo systemctl daemon-reload
sudo systemctl enable nginx
sudo systemctl start nginx
```

设置主机名

```bash
hostnamectl set-hostname k8s-nginx
```

.修改*/etc/hosts*，设置rancher webui的域名, 添加以下内容

```bash
192.168.56.111 rancher.bi.io <1>
```
<1> 域名可以替换成你的实际值，注意要和后续使用到该域名的地方保持一致


## k8s节点环境准备

在每台k8s节点机器上都要执行下面所述的步骤

主机名设置

```bash
sudo hostnamectl set-hostname k8s-node-00 <1>
```
<1> 依次对其它节点进行设置

关闭防火墙

```bash
sudo systemctl stop firewalld
sudo systemctl disable firewalld
sudo setenforce 0
sudo swapoff -a
```

关闭SELinux, 修改 */etc/sysconfig/selinux*

```bash
SELINUX=disabled
```

网络配置相关

```bash
cat <<EOF >  /etc/sysctl.d/k8s.conf
vm.swappiness = 0
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
```

```bash
sudo modprobe br_netfilter
sudo sysctl -p /etc/sysctl.d/k8s.conf
```

加载ipv4模块

```bash
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4

chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack_ipv4

yum install ipset ipvsadm -y
```

安装docker-ce, 使用阿里云的centos repository

```bash
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum makecache fast
yum install docker-ce
systemctl enable docker && systemctl start docker
```

.添加普通用户osboxes到docker组

```bash
usermod -aG osboxes docker <1>
```
<1> 根据实际情况，替换用户osboxes为你实际的用户名

创建 */etc/docker/daemon.json*, 使用国内镜像，加速docker image下载

```json
{
  "registry-mirrors": [
   "https://reg-mirror.qiniu.com",
   "https://1nj0zren.mirror.aliyuncs.com",
    "https://docker.mirrors.ustc.edu.cn",
    "https://dockerhub.azk8s.cn",
    "https://hub-mirror.c.163.com"
  ]
}
```

修改完上述配置，重启docker以生效

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

.安装kubectl

```bash
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

sudo yum -y install kubectl
```

.修改 */etc/hosts*

```bash
192.168.56.101 k8s-node-00
192.168.56.102 k8s-node-01
192.168.56.103 k8s-node-02
192.168.56.111 rancher.bi.io
```

## rke安装k8s集群

本节操作全部在k8s-node-00上执行，当然也可以在其它机器上执行，记住，只需在一台机器上执行即可

.免密码登录

```bash
cd $HOME
ssh-keygen
ssh-copy-id -i ~/.ssh/id_rsa.pub osboxes@192.168.56.101
ssh-copy-id -i ~/.ssh/id_rsa.pub osboxes@192.168.56.102
ssh-copy-id -i ~/.ssh/id_rsa.pub osboxes@192.168.56.103
```

.下载rke

```bash
wget https://github.com/rancher/rke/releases/download/v1.0.6/rke_linux-amd64
mv rke_linux-amd64 rke
```

.创建k8s集群安装配置文件, *cluster.yml*

```yml
nodes:
  - address: 192.168.56.101
    user: osboxes
    role: [controlplane, worker, etcd]
  - address: 192.168.56.102
    user: osboxes
    role: [controlplane, worker, etcd]
  - address: 192.168.56.103
    user: osboxes
    role: [controlplane, worker, etcd]

services:
  etcd:
    snapshot: true
    creation: 6h
    retention: 24h
  kubelet:
    extra_binds:
     - /mnt:/mnt:rshared

network:
  plugin: canal
  options:
    canal_iface: enp0s8 <1>
    canal_flannel_backend_type: vxlan

ingress:
  provider: nginx
  options:
    use-forwarded-headers: "true"
```
<1> 如果在虚拟机上有多块网卡，要指定一块可以实现虚机内部通信的网卡，virtualbox中通常是指host-only类型的网卡


.执行安装

```bash
./rke up --config ./cluster.yml
```

该步骤需要等一段时间才能完成，期间可以使用如下指令来验证k8s集群ready与否

.检测安装状态

```bash
kubectl get pods
```

## 安装rancher

本节涉及的操作只需要在一台机器上执行即可， 假设继续使用k8s-node-00

.下载helm

```bash
wget https://get.helm.sh/helm-v3.1.2-linux-amd64.tar.gz
tar zvxf helm-v3.1.2-linux-amd64.tar.gz
mv linux-amd64/helm .
export PATH=$PATH:$PWD
#添加chart repository
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest
```

.安装cert-manager

```bash
kubectl apply -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.12/deploy/manifests/00-crds.yaml
kubectl create namespace cert-manager
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v0.12.0
```

.验证cert-manager安装是否完成

```bash
kubectl get pods --namespace cert-manager

NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-5c6866597-zw7kh               1/1     Running   0          2m
cert-manager-cainjector-577f6d9fd7-tr77l   1/1     Running   0          2m
cert-manager-webhook-787858fcdb-nlzsq      1/1     Running   0          2m
```

.安装rancher

```bash
kubectl create namespace cattle-system
helm install rancher rancher-latest/rancher \
  --namespace cattle-system \
  --set hostname=rancher.bi.io <1>
```
<1> 注意此处的rancher.bi.io必须要和/etc/hosts中的设置保持一致


.验证安装完成

```bash
kubectl -n cattle-system rollout status deploy/rancher
Waiting for deployment "rancher" rollout to finish: 0 of 3 updated replicas are available...
deployment "rancher" successfully rolled out
```

== 访问rancher webui

在host主机，假设是windows， 修改 **C:\Windows\System32\drivers\etc\hosts**
添加如下内容

.修改host主机上的hosts文件，如果没有就创建一个

```bash
192.168.56.111 rancher.bi.io
```

开始激动人心的验证， 用 *chrome* 或 *firefox* 打开 https://rancher.bi.io

## 跑路指南

假设上述步骤都没有让你掉到坑里，那么非常恭喜，最后送上跑路指南

.删除k8s集群，只需要在你的安装机k8s-node-00执行该操作

```bash
rke remove --config=./config.yaml
```

.删除其它，需要在k8s所有节点执行

```bash
sudo docker rm -f $(docker ps -qa)
docker volume rm $(docker volume ls -q)
cleanupdirs="/var/lib/etcd /etc/kubernetes /etc/cni /opt/cni /var/lib/cni /var/run/calico"
for dir in $cleanupdirs; do
  echo "Removing $dir"
  rm -rf $dir
done
```

.删除网络相关

```bash
sudo ip link delete virbr0
```

.清理iptables

```bash
sudo iptables --flush
sudo iptables --flush --table nat
sudo iptables --flush --table filter
sudo iptables --table nat --delete-chain
sudo iptables --table filter --delete-chain
```

把上述指令汇总成ansible playbook

.manager-k8s.yaml

```yml
- name: remove existed rancher
  gather_facts: false
  hosts: k8s
  tasks:
    - name: delete images
      shell: |
       sudo systemctl restart docker
       docker stop $(docker ps -q)
       sudo rm -rf /var/lib/rancher/state
       docker container prune -f
       docker rm -fv rancher-agent
       docker rm -fv rancher-agent-state
       sudo ip link delete virbr0-nic
       sudo ip link delete virbr0
       sudo iptables --flush
       sudo iptables --flush --table nat
       sudo iptables --flush --table filter
       sudo iptables --table nat --delete-chain
       rm -rf /etc/ceph \
       /etc/cni \
       /etc/kubernetes \
       /opt/cni \
       /opt/rke \
       /run/secrets/kubernetes.io \
       /run/calico \
       /run/flannel \
       /var/lib/calico \
       /var/lib/etcd \
       /var/lib/cni \
       /var/lib/kubelet \
       /var/lib/rancher/rke/log \
       /var/log/containers \
       /var/log/pods \
       /var/run/calico
       exit 0
```

## Trouble Shooting

这么长的安装步骤，怎么可能一帆风顺， 出现问题是正常，不出问题是不正常的，别紧张，一般情况都是因为Images没有下载导致的。

.检查pods的运行情况,获取详细的出错信息

```bash
docker get pods --all-namespaces
docker describe pod your_podname
```

### 网络排错

安装centos

```bash
kubectl run -it foo --image=centos --restart=Never -- /bin/bash
```
