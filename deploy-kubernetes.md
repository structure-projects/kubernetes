# 部署kubernetes v1.22.12
| 主机名    | IP地址                    | 角色       | 系统      |
| --------- | ------------------------- | ---------- | --------- |
| k8sMaster | 172.16.48.110             | k8s-master | Centos7.6 |
| k8sNode1  | 172.16.48.111             | k8s-node   | Centos7.6 |
| k8sNode2  | 172.16.48.112             | k8s-node   | Centos7.6 |

## 环境配置
### 关闭防火墙

```shell
systemctl stop firewalld && systemctl disable firewalld
```

### 关闭SELinux

```shell
setenforce 0
sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
```

### 关闭swap分区

```shell
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### 加载内核模块

```shell
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
modprobe -- br_netfilter
EOF

chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules
```

### 设置内核参数

```shell
cat << EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
net.ipv4.tcp_tw_recycle=0
vm.swappiness=0
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.inotify.max_user_watches=89100
fs.file-max=52706963
fs.nr_open=52706963
net.ipv6.conf.all.disable_ipv6=1
net.netfilter.nf_conntrack_max=2310720
EOF

sysctl -p /etc/sysctl.d/k8s.conf
```
## Docker 安装

[详情查看docker 部署](https://so.csdn.net/so/search?q=rpc&spm=1001.2101.3001.7020) 

### 设置镜像源

``` shell
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

yum makecache fast
```
### 安装k8s软件

```shell
yum install -y kubelet-1.22.12-0 kubeadm-1.22.12-0 kubectl-1.22.12-0

systemctl enable --now kubelet
```

### 拉取所需镜像

1. 查看镜像

```shell
 kubeadm config images list --kubernetes-version v1.22.12
```

2. 编写拉取脚本

```shell
vim get-k8s-images.sh
```

```sh
#!/bin/bash
# Script For Quick Pull K8S Docker Images

KUBE_VERSION=v1.22.12
PAUSE_VERSION=3.5
CORE_DNS_VERSION=v1.8.4
ETCD_VERSION=3.5.0-0

# pull kubernetes images from hub.docker.com
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy-amd64:$KUBE_VERSION
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager-amd64:$KUBE_VERSION
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver-amd64:$KUBE_VERSION
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler-amd64:$KUBE_VERSION
# pull aliyuncs mirror docker images
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:$PAUSE_VERSION
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:$CORE_DNS_VERSION
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:$ETCD_VERSION

# retag to k8s.gcr.io prefix
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy-amd64:$KUBE_VERSION  k8s.gcr.io/kube-proxy:$KUBE_VERSION
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager-amd64:$KUBE_VERSION k8s.gcr.io/kube-controller-manager:$KUBE_VERSION
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver-amd64:$KUBE_VERSION k8s.gcr.io/kube-apiserver:$KUBE_VERSION
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler-amd64:$KUBE_VERSION k8s.gcr.io/kube-scheduler:$KUBE_VERSION
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:$PAUSE_VERSION k8s.gcr.io/pause:$PAUSE_VERSION
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:$CORE_DNS_VERSION k8s.gcr.io/coredns:$CORE_DNS_VERSION
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:$ETCD_VERSION k8s.gcr.io/etcd:$ETCD_VERSION

# untag origin tag, the images won't be delete.
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy-amd64:$KUBE_VERSION
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager-amd64:$KUBE_VERSION
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver-amd64:$KUBE_VERSION
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler-amd64:$KUBE_VERSION
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/pause:$PAUSE_VERSION
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:$CORE_DNS_VERSION
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:$ETCD_VERSION
```

3. 导出镜像

   ```shell
   docker save $(docker images | grep -v REPOSITORY | awk 'BEGIN{OFS=":";ORS=" "}{print $1,$2}') -o k8s-1.22.12-images.tar
   ```

4. 导入镜像

```shell
docker image load -i k8s-1.22.12-images.tar
```

### 初始化集群

```shell
kubeadm init  --kubernetes-version=v1.22.12 --apiserver-advertise-address=172.16.48.110 --pod-network-cidr=10.244.0.0/16 --service-cidr=10.1.0.0/16 

```
### 加入集群
具体指令需要看上面初始化集群后给出的token信息

### 安装网络插件
以下两种网络插件二选一即可
#### 安装flannel网络

```shell
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

#### 安装Calico网络插件

```shell
curl  https://docs.projectcalico.org/v3.15/manifests/calico.yaml -O

kubectl apply -f calico.yaml

```
