# 高可用下部署NFS服务共享存储
## 环境基础信息
| 主机名       | IP地址                    | 角色                    | 系统      |
| ------------ | ------------------------- | ----------------------- | --------- |
| dockerMaster | 172.16.48.110			   |server 	| Centos7.6 |
| dockerNode1  | 172.16.48.111             | client | Centos7.6 |
| dockerNode2  | 172.16.48.112             | client | Centos7.6 |

## 服务端搭建

### 服务端安装 nfs,rpc 服务

```shell
yum install -y nfs-utils rpcbind && yum install -y nfs-utils
```

### 配置 nfs 服务

#### 创建共享目录

``` shell
mkdir -p /opt/share
```

#### 修改nfs配置文件

```shell
vim /etc/exports
```

```
/opt/share  *(rw,no_root_squash,no_all_squash,sync)
```

### 查看服务端是否正常加载/etc/exports配置文件

```shell
exportfs -r
showmount -e localhost
```

### 启动RPC,nfs服务

```shell
systemctl start rpcbind && systemctl start nfs
```

## 客户端搭建

### 安装nfs客户端nfs-utils

```shell
yum install nfs-utils vim -y
```

### 查看服务端可共享的目录

```shell
showmount -e 172.16.48.110
```

### 挂载服务端共享目录

```shell
mkdir -p /opt/share && mount -t nfs 172.16.48.110:/opt/share /opt/share/ -o nolock,nfsvers=3,vers=3
```

### 设置开机自动挂载

将挂载命令添加到启动文件中

```shelli
vim /etc/fstab
```

```shell
172.16.48.110:/opt/share /opt/share/ nfs defaults        0 0
```

