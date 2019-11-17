# 二、基础集群部署 - kubernetes-simple

## 1. 部署ETCD（主节点）

#### 1.1 简介

  kubernetes需要存储很多东西，像它本身的节点信息，组件信息，还有通过kubernetes运行的pod，deployment，service等等。都需要持久化。etcd就是它的数据中心。生产环境中为了保证数据中心的高可用和数据的一致性，一般会部署最少三个节点。

#### 1.2 部署

**etcd 的二进制文件和服务的配置我们都已经准备好，现在的目的就是把它做成系统服务并启动。**

```shell
## 目录
$ cd /usr/local/src/kubernetes/kubernetes-sh
## CentOS7 的服务 systemctl 脚本存放在: /usr/lib/systemd/
# 把服务配置文件 copy 到系统服务目录
$ cp ./target/master-node/etcd.service /usr/lib/systemd/system/

## NOTICE 在这个时候可能需要修改文件格式
## vim /usr/lib/systemd/system/etcd.service 
## :set fileformat=unix

# enable 服务
$ systemctl enable etcd.service
# daemon-reload
$ systemctl daemon-reload

# 创建工作目录(保存数据的地方)
# 配置文件里面已经配置了:勿改
$ mkdir -p /var/lib/etcd

# 启动服务
$ systemctl start etcd
# 查看服务日志,看是否有错误信息,确保服务正常
$ journalctl -f -u etcd.service
```



## 2. 部署APIServer（主节点）

#### 2.1 简介

kube-apiserver是Kubernetes最重要的核心组件之一，主要提供以下的功能

- 提供集群管理的REST API接口，包括认证授权（我们现在没有用到）数据校验以及集群状态变更等
- 提供其他模块之间的数据交互和通信的枢纽（其他模块通过API Server查询或修改数据，只有API Server才直接操作etcd）

> 生产环境为了保证apiserver的高可用一般会部署2+个节点，在上层做一个lb做负载均衡，比如haproxy。

#### 2.2 部署

APIServer 的部署方式也是通过系统服务。部署流程跟etcd完全一样。

```shell
## 目录
$ cd /usr/local/src/kubernetes/kubernetes-sh
$ cp ./target/master-node/kube-apiserver.service /usr/lib/systemd/system/

## NOTICE 在这个时候可能需要修改文件格式
## vim /usr/lib/systemd/system/kube-apiserver.service 
## :set fileformat=unix

$ systemctl enable kube-apiserver.service
$ systemctl daemon-reload

$ systemctl start kube-apiserver
$ journalctl -f -u kube-apiserver
```



## 3. 部署ControllerManager（主节点）

#### 3.1 简介

Controller Manager由kube-controller-manager和cloud-controller-manager组成，是Kubernetes的大脑，它通过apiserver监控整个集群的状态，并确保集群处于预期的工作状态。 kube-controller-manager由一系列的控制器组成，像Replication Controller控制副本，Node Controller节点控制，Deployment Controller管理deployment等等 cloud-controller-manager在Kubernetes启用Cloud Provider的时候才需要，用来配合云服务提供商的控制

> controller-manager、scheduler和apiserver 三者的功能紧密相关，一般运行在同一个机器上，我们可以把它们当做一个整体来看，所以保证了apiserver的高可用即是保证了三个模块的高可用。也可以同时启动多个controller-manager进程，但只有一个会被选举为leader提供服务。

#### 3.2 部署

**通过系统服务方式部署**

```shell
## 目录
$ cd /usr/local/src/kubernetes/kubernetes-sh
$ cp ./target/master-node/kube-controller-manager.service /usr/lib/systemd/system/

$ systemctl enable kube-controller-manager.service
$ systemctl daemon-reload

$ systemctl start kube-controller-manager

$ journalctl -f -u kube-controller-manager
```

```shell
[Unit]
Description=Kubernetes Controller Manager
...
[Service]
ExecStart=/usr/local/kubernetes/bin/kube-controller-manager \
# 对外服务的监听地址,这里表示只有本机的程序可以访问它
--address=127.0.0.1 \
# apiserver的url
--master=http://127.0.0.1:8080 \
# 服务虚拟ip范围，同apiserver的配置
--service-cluster-ip-range=10.68.0.0/16 \
# pod的ip地址范围
--cluster-cidr=172.20.0.0/16 \
# 下面两个表示不使用证书，用空值覆盖默认值
--cluster-signing-cert-file= \
--cluster-signing-key-file= \
...
```



## 4. 部署Scheduler（主节点）

#### 4.1 简介

kube-scheduler负责分配调度Pod到集群内的节点上，它监听kube-apiserver，查询还未分配Node的Pod，然后根据调度策略为这些Pod分配节点。我们前面讲到的kubernetes的各种调度策略就是它实现的。

#### 4.2 部署

**通过系统服务方式部署**

```shell
## 目录
$ cd /usr/local/src/kubernetes/kubernetes-sh
$ cp ./target/master-node/kube-scheduler.service /usr/lib/systemd/system/

$ systemctl enable kube-scheduler.service
$ systemctl daemon-reload

$ systemctl start kube-scheduler

$ journalctl -f -u kube-scheduler
```

#### 4.3 重点配置说明

```shell
[Unit]
Description=Kubernetes Scheduler
...
[Service]
ExecStart=/usr/local/kubernetes/bin/kube-scheduler \
# 对外服务的监听地址,里表示只有本机的程序可以访问它
--address=127.0.0.1 \
# apiserver的url
--master=http://127.0.0.1:8080 \
...
```



## 5. 部署CalicoNode（所有节点）

#### 5.1 简介

Calico实现了CNI接口，是kubernetes网络方案的一种选择，它一个纯三层的数据中心网络方案（不需要Overlay），并且与OpenStack、Kubernetes、AWS、GCE等IaaS和容器平台都有良好的集成。 Calico在每一个计算节点利用Linux Kernel实现了一个高效的vRouter来负责数据转发，而每个vRouter通过BGP协议负责把自己上运行的workload的路由信息像整个Calico网络内传播——小规模部署可以直接互联，大规模下可通过指定的BGP route reflector来完成。 这样保证最终所有的workload之间的数据流量都是通过IP路由的方式完成互联的。

#### 5.2 部署

**calico是通过系统服务+docker方式完成的**

```shell
$ cd /usr/local/src/kubernetes/kubernetes-sh
$ cp ./target/all-node/kube-calico.service /usr/lib/systemd/system/
$ systemctl enable kube-calico.service
$ systemctl daemon-reload

$ systemctl start kube-calico
$ systemctl stop kube-calico

$ journalctl -f -u kube-calico
```

#### 5.3 calico可用性验证

**查看容器运行情况**

```shell
$ docker ps
```

**查看节点运行情况**

```shell
$ calicoctl node status
```

**查看端口BGP 协议是通过TCP 连接来建立邻居的，因此可以用netstat 命令验证 BGP Peer**

```shell
$ netstat -natp | grep ESTABLISHED | grep 179
## ----------------------------------------------------- 查看结果
$ netstat -natp | grep ESTABLISHED | grep 179
tcp        0      0 192.168.0.12:36608      192.168.0.14:179        ESTABLISHED 1507/bird           
tcp        0      0 192.168.0.12:52892      192.168.0.13:179        ESTABLISHED 1507/bird           
```

**查看集群 ippool 情况**

```shell
## etcd 节点执行 查看ippool
$ calicoctl get ipPool -o yaml
## ----------------------------------------------------- 查看结果
$ calicoctl get ipPool -o yaml
- apiVersion: v1
  kind: ipPool
  metadata:
    cidr: 172.20.0.0/16
  spec:
    nat-outgoing: true
```



## 6. 配置kubectl命令（任意节点）

#### 6.1 简介

kubectl是Kubernetes的命令行工具，是Kubernetes用户和管理员必备的管理工具。 kubectl提供了大量的子命令，方便管理Kubernetes集群中的各种功能。

#### 6.2 初始化

使用kubectl的第一步是配置Kubernetes集群以及认证方式，包括：

- cluster信息：api-server地址
- 用户信息：用户名、密码或密钥
- Context：cluster、用户信息以及Namespace的组合

我们这没有安全相关的东西，只需要设置好api-server和上下文就好啦：

```shell
# 示例选择在 主节点
# 指定apiserver地址
$ kubectl config set-cluster kubernetes --server=http://192.168.0.12:8080
# 指定设置上下文,指定cluster
$ kubectl config set-context kubernetes --cluster=kubernetes
# 选择默认的上下文
$ kubectl config use-context kubernetes
```

> 通过上面的设置最终目的是生成了一个配置文件：~/.kube/config。

```shell
$ vim ~/.kube/config 
## ----------------------------------------------------- 查看结果
apiVersion: v1
clusters:
- cluster:
    server: http://192.168.0.12:8080
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: ""
  name: kubernetes
current-context: kubernetes
kind: Config
preferences: {}
users: []
```



## 7. 配置kubelet（工作节点）

#### 7.1 简介

每个工作节点上都运行一个kubelet服务进程，默认监听10250端口，接收并执行master发来的指令，管理Pod及Pod中的容器。每个kubelet进程会在API Server上注册节点自身信息，定期向master节点汇报节点的资源使用情况，并通过cAdvisor监控节点和容器的资源。

#### 7.2 部署

**通过系统服务方式部署，但步骤会多一些，具体如下：**

```shell
# 目录
$ cd /usr/local/src/kubernetes/kubernetes-sh
# 确保相关目录存在
$ mkdir -p /var/lib/kubelet
$ mkdir -p /etc/kubernetes
$ mkdir -p /etc/cni/net.d

# 复制kubelet服务配置文件
$ cp ./target/worker-node/kubelet.service /usr/lib/systemd/system/
# 复制kubelet依赖的配置文件
$ cp ./target/worker-node/kubelet.kubeconfig /etc/kubernetes/
# 复制kubelet用到的cni插件配置文件
$ cp ./target/worker-node/10-calico.conf /etc/cni/net.d/

$ systemctl enable kubelet.service
$ systemctl daemon-reload

$ systemctl start kubelet

$ journalctl -f -u kubelet
```

#### 7.3 重点配置说明

```shell
[Unit]
Description=Kubernetes Kubelet
[Service]
# kubelet工作目录,存储当前节点容器,pod等信息
WorkingDirectory=/var/lib/kubelet
ExecStart=/home/michael/bin/kubelet \
# 对外服务的监听地址
--address=192.168.0.13 \
# 指定基础容器的镜像，负责创建Pod 内部共享的网络、文件系统等，这个基础容器非常重要：K8S每一个运行的 POD里面必然包含这个基础容器，如果它没有运行起来那么你的POD 肯定创建不了
--pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/imooc/pause-amd64:3.0 \
# 访问集群方式的配置，如api-server地址等
--kubeconfig=/etc/kubernetes/kubelet.kubeconfig \
# 声明cni网络插件
--network-plugin=cni \
# cni网络配置目录，kubelet会读取该目录下得网络配置
--cni-conf-dir=/etc/cni/net.d \
# 指定 kubedns 的 Service IP(可以先分配，后续创建 kubedns 服务时指定该 IP)，--cluster-domain 指定域名后缀，这两个参数同时指定后才会生效
--cluster-dns=10.68.0.2 \
...
```

**kubelet.kubeconfig** 

> kubelet依赖的一个配置，格式看也是我们后面经常遇到的yaml格式，描述了kubelet访问apiserver的方式

```yaml
apiVersion: v1  
clusters:  
- cluster:  
 # 跳过tls，即是kubernetes的认证  
 insecure-skip-tls-verify: true  
 # api-server地址  
 server: http://192.168.0.12:8080  
...  
```

**10-calico.conf** 
calico作为kubernets的CNI插件的配置

```json
{  
  "name": "calico-k8s-network",  
  "cniVersion": "0.1.0",  
  "type": "calico",  
    "ed_endpoints": "http://192.168.0.12:2379",  
    "logevel": "info",  
    "ipam": {  
        "type": "calico-ipam"  
   },  
    "kubernetes": {
        "k8s_api_root": "http://192.168.0.12:8080"  
    }  
}  
```

#### 部署kubernetes-bootcamp

```shell
$ kubectl run kubernetes-bootcamp --image=jocatalin/kubernetes-bootcamp:v1 --port=8080
## 通过kubectl proxy
$ kubectl proxy

## 获取pods
$ kubectl get pods
NAME                                   READY     STATUS    RESTARTS   AGE
kubernetes-bootcamp-6b7849c495-hlnxx   1/1       Running   0          39m
## --------------------------------------
## 访问
$ curl -X GET http://localhost:8001/api/v1/proxy/namespaces/default/pods/kubernetes-bootcamp-6b7849c495-hlnxx/
## response
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-6b7849c495-hlnxx | v=1
## --------------------------------------
# 扩容
$ kubectl scale deploy kubernetes-bootcamp --replicas=4
# 缩容
$ kubectl scale deploy kubernetes-bootcamp --replicas=2
## --------------------------------------
## 查看描述
$ kubectl describe deploy kubernetes-bootcamp
## --------------------------------------
## 更新镜像
$ kubectl set image deploy kubernetes-bootcamp kubernetes-bootcamp=jocatalin/kubernetes-bootcamp:v2
## --------------------------------------
## rollout
$ kubectl rollout status deploy kubernetes-bootcamp
## --------------------------------------
## 回滚
$ kubectl rollout undo deploy kubernetes-bootcamp
$ kubectl rollout status deploy kubernetes-bootcamp

## -------------------------------------- 部署 nginx 
$ mkdir -p /usr/local/src/kubernetes/services
$ cd /usr/local/src/kubernetes/services
$ vim ./nginx-pod.yaml
## --------------------------------------
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.13
      ports:
      - containerPort: 80
## --------------------------------------

## 部署
$ kubectl create -f ./nginx-pod.yaml
## 启动代理
$ kubectl proxy
## 查看
$ curl -X GET http://localhost:8001/api/v1/proxy/namespaces/default/pods/nginx/
```

```xml
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

```yaml
## 创建 nginx deployment
# cd /usr/local/src/kubernetes/services
# vim ./nginx-deployment.yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  template:
    metadata:
      labels:
       app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.13
        ports:
         - containerPort: 80
```

```shell
$ kubectl create -f ./nginx-deployment.yaml
```





## 9. 为集群增加service功能 - kube-proxy（工作节点）

#### 9.1 简介

每台工作节点上都应该运行一个kube-proxy服务，它监听API server中service和endpoint的变化情况，并通过iptables等来为服务配置负载均衡，是让我们的服务在集群外可以被访问到的重要方式。

#### 9.2 部署

**通过系统服务方式部署：**

```shell
# 目录
$ cd /usr/local/src/kubernetes/kubernetes-sh
# 确保工作目录存在
$ mkdir -p /var/lib/kube-proxy
# 复制kube-proxy服务配置文件
$ cp ./target/worker-node/kube-proxy.service /usr/lib/systemd/system/
# 复制kube-proxy依赖的配置文件
$ cp ./target/worker-node/kube-proxy.kubeconfig /etc/kubernetes/

$ systemctl enable kube-proxy.service
$ systemctl daemon-reload

$ systemctl start kube-proxy

$ journalctl -f -u kube-proxy
```

#### 9.3 重点配置说明

**kube-proxy.service**

```shell
[Unit]
Description=Kubernetes Kube-Proxy Server ...
[Service]
# 工作目录
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/usr/local/kubernetes/bin/kube-proxy \
# 监听地址
--bind-address=192.168.0.13 \
# 依赖的配置文件,描述了kube-proxy如何访问api-server
--kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig \
...
```



**查看服务**

```shell
## 查看服务
$ kubectl get services
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.68.0.1    <none>        443/TCP   5h

## 查看服务描述
$ kubectl describe service kubernetes
Name:              kubernetes
Namespace:         default
Labels:            component=apiserver
                   provider=kubernetes
Annotations:       <none>
Selector:          <none>
Type:              ClusterIP
IP:                10.68.0.1
Port:              https  443/TCP
TargetPort:        6443/TCP
Endpoints:         192.168.0.12:6443
Session Affinity:  ClientIP
Events:            <none>
```



**获取 deployment**

```shell
## 获取 deployment
$ kubectl get deploy
NAME                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
kubernetes-bootcamp   2         2         2            2           2h
nginx-deployment      2         2         2            2           57m
```



**暴露服务**

```shell
## expose
$ kubectl expose deploy kubernetes-bootcamp --type="NodePort" --target-port=8080 --port=80

## 查看服务
$ kubectl get services
NAME                  TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
kubernetes            ClusterIP   10.68.0.1      <none>        443/TCP        5h
kubernetes-bootcamp   NodePort    10.68.94.245   <none>        80:20503/TCP   1m

## 主节点查看 端口 20503
$ netstat -tunlp | grep 20503
## 结果什么都没有 -> 因为主节点没有 kube-proxy
## ----------------------------------------------
## worker 节点查看
$ netstat -tunlp | grep 20503
tcp6      0      0 :::20503      :::*      LISTEN      8520/kube-proxy 

## ----------------------------------------------
## 访问服务
$ curl 192.168.0.13:20503
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-7689dc585d-q2cmq | v=2
$ curl 192.168.0.14:20503
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-7689dc585d-hh2gg | v=2
$ curl 192.168.0.12:20503
## 主节点没有kube-proxy
curl: (7) Failed connect to 192.168.0.12:20503; Connection refused

## ----------------------------------------------

# 查看pod
$ kubectl get pods -o wide
NAME                                   READY     STATUS    RESTARTS   AGE       IP              NODE
kubernetes-bootcamp-7689dc585d-hh2gg   1/1       Running   0          1h        172.20.134.68   192.168.0.14
kubernetes-bootcamp-7689dc585d-q2cmq   1/1       Running   0          1h        172.20.33.66    192.168.0.13
nginx                                  1/1       Running   0          1h        172.20.33.67    192.168.0.13
nginx-deployment-d8d99448f-fkvhm       1/1       Running   0          1h        172.20.33.68    192.168.0.13
nginx-deployment-d8d99448f-nhfc9       1/1       Running   0          1h        172.20.134.69   192.168.0.14
## -> 13 | 14 均运行着 kubernetes-bootcamp -> 任何一台均可访问 -> CLUSTER-IP(10.68.94.245)

## ----------------------------------------------

## 13 运行
$ docker ps | grep bootcamp
ec3da31a7afa        jocatalin/kubernetes-bootcamp                                "/bin/sh -c 'node se…"   2 hours ago         Up 2 hours                              k8s_kubernetes-bootcamp_kubernetes-bootcamp-7689dc585d-q2cmq_default_3b6584f2-0860-11ea-b4dc-000c29f678cc_0
7d8d0ba433be        registry.cn-hangzhou.aliyuncs.com/imooc/pause-amd64:3.0      "/pause"                 2 hours ago         Up 2 hours                              k8s_POD_kubernetes-bootcamp-7689dc585d-q2cmq_default_3b6584f2-0860-11ea-b4dc-000c29f678cc_0

## ->

$ docker exec -it ec(ec3da31a7afa) /bin/bash
## ---------------------------------------------- 进入容器
$ docker exec -it ec /bin/bash
root@kubernetes-bootcamp-7689dc585d-q2cmq:/
$ curl 10.68.94.245 ###### cluster ip = 10.68.94.245 && service-port=80
Hello Kubernetes bootcamp! | Running on: kubernetes-bootcamp-7689dc585d-hh2gg | v=2
## -> 通过 cluster-ip 80 端口也可以访问

## ---------------------------------------------- 访问 其他的 pod -> nginx
$ docker exec -it ec(ec3da31a7afa) /bin/bash
root@kubernetes-bootcamp-7689dc585d-q2cmq:/
$ curl 172.20.33.67  ###### pod ip = 172.20.33.67 && container-port=80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```



**创建服务-通过 yaml配置文件**

```yaml
## master 节点
## cd /usr/local/src/kubernetes/services
## vim ./nginx-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  ports:
  - port: 8080 ## service port,cluster-ip :: port
    targetPort: 80 ## 容器端口
    nodePort: 20000 ## 节点监听的端口,能对外提供服务
  selector:
    app: nginx
  type: NodePort
```

**创建服务**

```shell
## 创建服务
$ kubectl create -f ./nginx-service.yaml 
service "nginx-service" created

## 获取服务
$ kubectl get svc(services)
NAME                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
kubernetes            ClusterIP   10.68.0.1       <none>        443/TCP          6h
kubernetes-bootcamp   NodePort    10.68.94.245    <none>        80:20503/TCP     41m
nginx-service         NodePort    10.68.117.153   <none>        8080:20000/TCP   1m

## 多了一个 nginx-service 8080:20000
```

```xml
<!-- 查看 13 14 上 20000 端口是否可用 -->
<!-- $ curl 192.168.0.13:20000 -->
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```



## 10. 为集群增加dns功能 - kube-dns（app）

#### 10.1 简介

kube-dns为Kubernetes集群提供命名服务，主要用来解析集群服务名和Pod的hostname。目的是让pod可以通过名字访问到集群内服务。它通过添加A记录的方式实现名字和service的解析。普通的service会解析到service-ip。headless service会解析到pod列表。

#### 10.2 部署

**通过kubernetes应用的方式部署** kube-dns.yaml文件基本与官方一致（除了镜像名不同外）。 里面配置了多个组件，之间使用”---“分隔

```shell

# 到 kubernetes-starter 目录执行命令
## 已经创建好了 位置: ./target/services/kube-dns.yaml
## ----------------------------------------------

$ cp ./target/services/kube-dns.yaml /usr/local/src/kubernetes/services

## ----------------------------------------------
# cd /usr/local/src/kubernetes/services
$ kubectl create -f ./kube-dns.yaml
configmap "kube-dns" created
serviceaccount "kube-dns" created
service "kube-dns" created
deployment "kube-dns" created

## ---------------------------------------------- 查看 注意: namespace == kube-system
## 查看 svc
$ kubectl -n kube-system get svc
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
kube-dns   ClusterIP   10.68.0.2    <none>        53/UDP,53/TCP   11m
## 查看 deploy
[root@docker-master services]# kubectl -n kube-system get deploy
NAME       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
kube-dns   1         1         1            1           3m
## 查看 pods
$ kubectl -n kube-system get pods -o wide
NAME                       READY     STATUS    RESTARTS   AGE       IP              NODE
kube-dns-7496d9d9b-cf595   3/3       Running   0          5m        172.20.134.70   192.168.0.14

## -> 运行在 14 上面
## ---------------------------------------------- 14(worker2) 执行
$ docker ps | grep bootcamp
7edff54e487d        jocatalin/kubernetes-bootcamp                                         "/bin/sh -c 'node se…"   3 hours ago         Up 3 hours                              k8s_kubernetes-bootcamp_kubernetes-bootcamp-7689dc585d-hh2gg_default_3b604f06-0860-11ea-b4dc-000c29f678cc_0
a6691685c85e        registry.cn-hangzhou.aliyuncs.com/imooc/pause-amd64:3.0               "/pause"                 3 hours ago         Up 3 hours                              k8s_POD_kubernetes-bootcamp-7689dc585d-hh2gg_default_3b604f06-0860-11ea-b4dc-000c29f678cc_0

## -> 7edff54e487d
[root@docker-worker2 system]# docker exec -it 7e /bin/bash
root@kubernetes-bootcamp-7689dc585d-hh2gg:/
## ----------------------------------------------
## 通过 service-name 访问
$ curl nginx-service:8080 ## 8080 是配置的端口 nginx.service.yaml.spec.ports[0]
## 能访问到服务 -> dns 服务 帮忙解析了地址
## ----------------------------------------------
## 通过 cluster-ip 访问
$ curl 10.68.117.153:8080
## 能访问到服务 -> 其实 dns 服务最终也是解析到 cluster-ip==10.68.117.153

## 查看 dns 配置
$ cat /etc/resolv.conf
## ----------------------------
nameserver 10.68.0.2 ## dns 服务
search default.svc.cluster.local. svc.cluster.local. cluster.local. com
options ndots:5
```

```xml
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```



**测试删除 dns 是否还可以继续访问**

```shell
## master 执行
$ cd /usr/local/src/kubernetes/services
$ kubectl delete -f ./kube-dns.yaml
## ----------------------------------------------
$ kubectl delete -f ./kube-dns.yaml
configmap "kube-dns" deleted
serviceaccount "kube-dns" deleted
service "kube-dns" deleted
deployment "kube-dns" deleted
## ---------------------------------------------- 进入容器
$ curl nginx-service:8080 ## 阻塞
$ ping nginx-service ## 依然阻塞
## -> 确实是 dns 服务 提供解析服务
## ----------------------------------------------
## 继续创建
$ kubectl create -f ./kube-dns.yaml
## ---------------------------------------------- 容器内继续查看 -> 能访问
$ curl nginx-service:8080
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```



