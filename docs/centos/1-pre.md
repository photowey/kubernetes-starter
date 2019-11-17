# 一、预先准备环境

## 1. 准备服务器

这里准备了三台centos虚拟机，每台一核cpu和2G内存，配置好root账户，后续的所有操作都是使用root账户。

| 系统类型                             | IP地址       | 节点角色 | CUP  | MEMORY | HOST    |
| ------------------------------------ | ------------ | -------- | ---- | ------ | ------- |
| CentOS Linux release 7.4.1708 (Core) | 192.168.0.12 | MASTER   | 1    | 2G     | master  |
| CentOS Linux release 7.4.1708 (Core) | 192.168.0.13 | WORKER1  | 1    | 2G     | worker1 |
| CentOS Linux release 7.4.1708 (Core) | 192.168.0.14 | WORKER2  | 1    | 2G     | worker2 |

## 2 安装Docker(所有的节点)

```shell
1.通过 uname -r 命令查看你当前的内核版本
$ uname -r

2.使用 root 权限登录 Centos。确保 yum 包更新到最新。
$ sudo yum update

3、卸载旧版本(如果安装过旧版本的话)
$ sudo yum remove docker docker-common docker-selinux docker-engine

4、安装需要的软件包, yum-util 提供yum-config-manager功能,
另外两个是devicemapper驱动依赖的
$ sudo yum install -y yum-utils device-mapper-persistent-data lvm2

5、设置yum源
$ sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

6、可以查看所有仓库中所有docker版本，并选择特定版本安装
$ yum list docker-ce --showduplicates | sort -r

7、安装docker
#由于repo中默认只开启stable仓库，故这里安装最新稳定版,比如:18.09.3-3.el7
$ sudo yum install -y docker-ce

8、启动并加入开机启动
$ sudo systemctl start docker
$ sudo systemctl enable docker

9、验证安装是否成功(有client和service两部分表示docker安装启动都成功了)
$ docker version
```

## 3 系统设置

### 3.1 关闭防火墙

```shell
## 开放对应端口也可以
## firewalld cmd
## --------------------------- 相关命令
$ systemctl start firewalld
$ systemctl stop firewalld
$ systemctl status firewalld 
$ systemctl disable firewalld
$ systemctl enable firewalld
## --------------------------- 查看列表
$ firewall-cmd --zone=public --list-ports
## --------------------------- 刷新
firewall-cmd --reload
```



### 3.2 设置系统参数

######  允许路由转发，不对bridge的数据进行处理

```shell
## 写入配置文件
$ cat <<EOF > /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
 
## 生效配置文件
$ sysctl -p /etc/sysctl.d/k8s.conf
```

### 3.3 修改host文件

```shell
## hostname
## --------------------------- hostname
$ vim /etc/hostname
# 192.168.0.12 
master.com
# 192.168.0.13 
worker1.com
# 192.168.0.14 
worker2.com
## --------------------------- hosts
$ vim /etc/hosts
192.168.0.12 master.com
192.168.0.13 worker1.com
192.168.0.14 worker2.com
## --------------------------- ping 节点
ping -w 4 master.com | mastworker1er.com | worker2.com
```

## 4 二进制文件准备

```shell
## 新增 kubernetes 目录
$ mkdir -p /usr/local/kubernetes
## 下载
https://pan.baidu.com/s/1bMnqWY -> kubernetes-bins
## 上传
rz -> kubernetes-bins.tar.gz

## 配置k8s环境变量
$ KUBERNETES_HOME=/usr/local/kubernetes
$ export $PATH:$KUBERNETES_HOME/bin
```



## 5 下载配置文件

### 5.1 克隆 kubernetes-starter

```shell
## 下载配置文件
$ mkdir -p /usr/local/src/kubernetes
$ git clone https://github.com/photowey/kubernetes-starter.git
$ mv kubernetes-starter ./kubernetes-sh
## 
/usr/local/src/kubernetes/kubernetes-sh
```
### 5.2 修改节点配置

```shell
## master | worker1 | worker2
cd ./kubernetes-starter
vim ./config.properties
# kubernetes 二进制文件目录,eg:/usr/local/kubernetes/bin
BIN_PATH=/usr/local/kubernetes/bin

# 当前节点ip, eg: 192.168.0.12
NODE_IP=192.168.0.12 | 192.168.0.13(worker1) | 192.168.0.14(worker2)

# etcd服务集群列表,eg: http://192.168.0.12:2379
# 如果已有etcd集群可以填写现有的。没有的话填写：http://${MASTER_IP}:2379 （MASTER_IP自行替换成自己的主节点ip）
# 如果用了证书,就要填写https://${MASTER_IP}:2379 （MASTER_IP自行替换成自己的主节点ip）
ETCD_ENDPOINTS=http://192.168.0.12:2379

# kubernetes主节点ip地址,eg:192.168.0.12
MASTER_IP=192.168.0.12
```

```shell
## 修改权限
chmod 755 ./gen-config.sh
## 修改文件格式
vim ./gen-config.sh
:set fileformat=unix
```

