---
title: 使用kubeadm搭建k8s高可用集群
top: false
cover: false
mathjax: false
date: 2022-05-11 22:31:30
author: 红色蒲公英
img:
coverImg:
password:
summary:
tags: kubernetes
categories: kubernetes
---

## kubeadm搭建k8s环境

> 基于docker-ce-18.06.1.ce-3.el7，kubeadm/kubelet/kubectl 1.18.0

### 基础环境配置（master/node1/node2均需执行）

- **使用vmware新建三台虚拟机**

  - 去除无关硬件（USB、声卡、打印机）防止电脑蓝屏

  - 发行版本：CentOS7

  - 安装方式：Compute Node（请不要使用最小化安装，会导致集群部署失败，具体原因未知），内存1G，硬盘20G。注：最小化安装时默认不启用网卡，需修改`/etc/sysconfig/network-scripts/ifcfg-ens32`

  - 开启网卡
  - 指定hostname

- **修改`/etc/hosts`**

  ```shell
  cat >> /etc/hosts << EOF
  192.168.88.140 master
  192.168.88.141 node1
  192.168.88.139 node2
  EOF
  ```

- **禁用防火墙**

  ```shell
  systemctl stop firewalld && systemctl disable firewalld
  ```

- **禁用selinux**

  ```shell
  setenforce 0
  vi /etc/selinux/config
  # 将selinux = enforcing修改为disabled
  SELINUX=disabled
  ```

- **开启流量转发**

  ```shell
  touch /etc/sysctl.d/k8s.conf
  # 添加如下内容
  net.bridge.bridge-nf-call-ip6tables = 1 net.bridge.bridge-nf-call-iptables = 1 net.ipv4.ip_forward = 1
  # 或者使用cat创建
  cat > /etc/sysctl.d/k8s.conf << EOF
  net.bridge.bridge-nf-call-ip6tables = 1
  net.bridge.bridge-nf-call-iptables = 1
  net.ipv4.ip_forward = 1
  EOF
  ```

  执行如下命令使修改生效

  ```shell
  modprobe br_netfilter
  sysctl -p /etc/sysctl.d/k8s.conf
  ```

- **安装docker**

  ```shell
  # 版本：docker-ce-19.03.13
  # 安装yum工具
  yum install -y yum-utils
  # 移除目前环境的docker
  yum remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine
  # 配置docker仓库
  yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
  # 查看docker可用版本
  yum list docker-ce --showduplicates | sort -r
  # 安装docker
  yum install -y docker-ce-19.03.13
  # 设置开机启动docker并启动docker
  systemctl enable docker && systemctl start docker 
  # 设置镜像加速
  cat << EOF > /etc/docker/daemon.json
  {
  "registry-mirrors": ["https://bh50li9j.mirror.aliyuncs.com"],
  "exec-opts": ["native.cgroupdriver=systemd"]
  }
  EOF
  # 重启服务
  systemctl daemon-reload
  systemctl restart docker
  ```

- **关闭swap**

  ```shell
  # 临时关闭
  swapoff -a
  # 永久关闭，将/etc/fstab中包含swap的那一行注释掉
  vi /etc/fstab
  #
  # /etc/fstab
  # Created by anaconda on Tue Nov 30 06:27:02 2021
  #
  # Accessible filesystems, by reference, are maintained under '/dev/disk'
  # See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
  #
  /dev/mapper/centos-root /                       xfs     defaults        0 0
  UUID=c44d1f4b-5c65-44e2-870f-671aac9daa7c /boot                   xfs     defaults        0 0
  # /dev/mapper/centos-swap swap                    swap    defaults        0 0
  ```

- **开启时间同步**

  ```shell
  systemctl start chronyd && systemctl enable chronyd
  ```

- **安装kubelet/kubectl/kubeadm**

  ```shell
  # 配置阿里云yum源
  cat << EOF > /etc/yum.repos.d/kubernetes.repo
  [kubernetes]
  name=Kubernetes
  baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
  enabled=1
  gpgcheck=1
  repo_gpgcheck=1
  gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
  EOF
  # 安装kubeadm\kubelet\kubectl
  yum install -y kubelet-1.18.1 kubeadm-1.18.1 kubectl-1.18.1
  # 设置开机启动并启动
  systemctl enable kubelet && systemctl start kubelet
  ```

### 集群安装初始化（仅在master操作）

- **执行初始化命令**

```shell
kubeadm init --apiserver-advertise-address=192.168.88.140 --image-repository=registry.aliyuncs.com/google_containers  --kubernetes-version=v1.18.1 --service-cidr=10.96.0.0/12 --pod-network-cidr=10.244.0.0/16 --ignore-preflight-errors=all --v=5
# --apiserver-advertise-address 配置master地址
# --image-repository默认是k8s.io，国内无法连接，这里使用阿里云镜像仓库
# --kubernetes-version k8s版本号
# --service-cidr
# --pod-network-cidr
```

- **调整配置**

  ```
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
  ```

- **配置flannel网络**

  ```shell
  # 将flannel.yml上传到master节点~/k8s/flannel.yml
  kubectl apply -f flannel.yml
  ```

### node加入集群

- 在node1/node2上执行

  ```shell
  kubeadm join 192.168.88.140:6443 --token zm9fyg.e74e11uq6jt9q82z \
      --discovery-token-ca-cert-hash sha256:b74bc3b2fbc54ef413a0a95fec3a39b0bd68ee7b0fe1af73e5cda537f230fcde
  ```

### 验证集群可用性

- 新建一个Nginx节点，测试能否访问

  ```shell
  # 新增deployment
  kubectl create deployment nginx --image=nginx
  # 暴露端口
  kubectl expose deployment nginx --port=80 --type=NodePort
  # 查看端口
  [root@master ssh]# kubectl get svc
  NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
  kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP        80m
  nginx        NodePort    10.110.68.195   <none>        80:32187/TCP   5s
  # 本地测试
  curl localhost:32187
  # 使用浏览器访问http://192.168.88.140:32187
  ```

### 安装dashboard

```shell
# 将kubernetes-dashborad.yml上传到/root/k8s/下

# 执行部署命令
kubectl create -f kubernertes-dashboard.yaml

# 创建访问账户，获取token
# 创建账号
kubectl create serviceaccount dashboard-admin -n kubernetes-dashboard

# 授权
kubectl create clusterrolebinding dashboard-admin-rb --clusterrole=cluster-admin --serviceaccount=kubernetes-dashboard:dashboard-admin

# 获取账号token
kubectl get secrets -n kubernetes-dashboard | grep dashboard-admin

kubectl describe secrets dashboard-admin-token-rqxsr -n kubernetes-dashboard

[root@master k8s]# kubectl describe secrets dashboard-admin-token-rqxsr -n kubernetes-dashboard
Name:         dashboard-admin-token-rqxsr
Namespace:    kubernetes-dashboard
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: dashboard-admin
              kubernetes.io/service-account.uid: f4fe684a-da98-498d-aee3-c1e9bb310831

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  20 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IjJqaHBvTGE1ZDdhSUxwaV9ZbFVlcEM0aGVuM2tRMUN2UXJYcmVnejlzVzQifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtYWRtaW4tdG9rZW4tcnF4c3IiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiZjRmZTY4NGEtZGE5OC00OThkLWFlZTMtYzFlOWJiMzEwODMxIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmVybmV0ZXMtZGFzaGJvYXJkOmRhc2hib2FyZC1hZG1pbiJ9.Nz9is-x3TekN-mJcKRPPbwxexONLJuKPnvqMUtG3WbFJXgMTVt5r1BcvGvtHxmrYzDjhlU3dKm0rHxg-3bI16TImr0KA1CHxjbgfTdA-3tamW95BsZaUal0cVo1eWZuA3_ZzR09J4KIQsoqhGtgBNqXqhGtBfMEEYz7-K9GOZwCui0E9N6VOEF5dzxrUR607EAbnX-0oIR9okzLScbdZBkCtxxXVR0qALneYsHEgGvTPGDXb8pK2HnhHVA1_2W7MCLwe5SjSqf2BIyL2ucMPdXRCIJH_XyKhOQDh65_py-e4W3imnt7IS-43Hzq10reznM0nafGl0N1r8N-D5NvgKg

# 浏览器打开https://192.168.88.140:30001（注意以https方式访问，端口为dashboard.yml文件中service资源中配置的端口）
```

### 服务器关机后重新搭建

- 重置k8s并重新初始化即可

  ```shell
  # 如果前期环境准备环节中某步骤不是永久生效，则需要再次执行该步骤
  # 1、删除配置
  # 如果不删除，可能会有如下报错信息
  {"level":"warn","ts":"2021-12-02T07:21:10.347-0500","caller":"clientv3/retry_interceptor.go:61","msg":"retrying of unary invoker failed","target":"endpoint://client-3bc363f7-6a42-4c81-bd71-705fae197b2b/192.168.88.140:2379","attempt":0,"error":"rpc error: code = Unknown desc = etcdserver: re-configuration failed due to not enough started members"}
  rm -rf /etc/kubernetes/*
  rm -rf /root/.kube/
  
  # 2、重置
  kubeadm reset
  
  # 3、初始化
  kubeadm init --apiserver-advertise-address=192.168.88.140 --image-repository=registry.aliyuncs.com/google_containers  --kubernetes-version=v1.18.1 --service-cidr=10.96.0.0/12 --pod-network-cidr=10.244.0.0/16 --ignore-preflight-errors=all --v=5
  ```

- 再次初始化时遇到的问题

```shell
# 再次初始化后发现etcd-master等容器时间停留在几天前，不是刚创建的
# 删除干净后，再安装k8s
yum remove -y kubelet kubeadm kubectl
kubeadm reset -f
modprobe -r ipip
lsmod
rm -rf ~/.kube/
rm -rf /etc/kubernetes/
rm -rf /etc/systemd/system/kubelet.service.d
rm -rf /etc/systemd/system/kubelet.service
rm -rf /usr/bin/kube*
rm -rf /etc/cni
rm -rf /opt/cni
rm -rf /var/lib/etcd
rm -rf /var/etcd

# 安装kubeadm\kubelet\kubectl
yum install -y kubelet-1.18.1 kubeadm-1.18.1 kubectl-1.18.1

systemctl daemon-reload

# 设置开机启动并启动
systemctl enable kubelet && systemctl start kubelet
```

