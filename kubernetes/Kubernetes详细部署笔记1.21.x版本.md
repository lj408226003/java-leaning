## Kubernetes详细部署笔记1.21.x版本



[TOC]

## 一、安装前的准备工作



### 1.关闭防火墙

```bash
# 方便起见，关闭所有节点防火墙，生产环境可以考虑开放相关端口
systemctl stop firewalld					#关闭服务
systemctl disable firewalld				#关闭自启
```

### 2.关闭linux selinux

```bash
# 关闭linux selinux：关闭安全机制，永久关闭，注意永久需要重启，所以带上一条临时就不用重启
sed -i 's/enforcing/disabled/' /etc/selinux/config  	#这是永久关停，这个是临时关闭[setenforce 0]
setenforce 0																					#临时
```

### 3.关闭swap分区

```bash
# 关闭swap分区[避免k8s使用swap分区降低性能]，需要永久关闭
sed -ri 's/.*swap.*/#&/' /etc/fstab  	#这个是临时关闭[swapoff -a]
swapoff -a  													#临时
```

### 4.修改各个节点hostname

```bash
hostnamectl set-hostname <name> 
```

### 5.设置/etc/hosts

```bash
cat >> /etc/hosts << EOF
192.168.9.185  kubernetes-master
192.168.9.253  kubernetes-node1
192.168.9.241  kubernetes-node2
EOF
```

### 6.设置网桥参数以及配置优化

```bash
cat >> /etc/sysctl.d/kubernetes.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1							#kubernetes官方建议的配置[网桥的配置]
net.bridge.bridge-nf-call-iptables = 1							#kubernetes官方建议的配置[网桥的配置]

####################### 以下为优化配置：将桥接ipv4流量传递给iptables ####################### 
net.ipv4.ip_forward=1
net.ipv4.tcp_tw_recycle=0
vm.swappiness=0																			#禁止使用swap空间，只有当系统OOM时才允许使用它
vm.overcommit_memory=1															#不检查物理内存是否够用
vm.panic_on_oom=0																		#开启OOM
fs.inotify.max_user_instances=8192
fs.inotify.max_user_watches=1048576
fs.file-max=52706963
fs.nr_open=52706963
net.ipv6.conf.all.disable_ipv6=1										#关闭ipv6模式
net.netfilter.nf_conntrack_max=2310720
EOF

sysctl -p /etc/sysctl.d/kubernetes.conf							#使配置文件能开机被调用
sysctl  --system																		#立即生效
```

> **注意：** 放在/etc/sysctl.d/该目录的内容可以设置开机自动加载

### 7.关闭系统不需要的服务

```bash
systemctl stop postfix && systemctl disable postfix
```

### 8.准备ipvs模块的启用

```bash
modprobe br_netfilter			#加载netfilter模块，ipvs调用时会有依赖
sysctl -p									#立即生效

# 1、开启ipvs支持，由于ipvs已经加入到了内核的主干，所以为kube-proxy开启ipvs模式的前提就是需要加载以下的内核模块
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh 
modprobe -- nf_conntrack
EOF

# 2、授权并且使ipvs.modules运行生效，并通过lsmod显示加载是否生效
chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack_ipv4

# 3、为了便于查看ipvs的代理转发规则，最好安装一下管理工具ipvsadm
yum install -y ipset ipvsadm
```

> 普及：linux默认支持iptables对网络包转发也支持ipvs转发，但iptables如果转发列表过多的话性能下降会比较快，远远不如ipvs的性能，而ipvs也是lvs的实现原理，
>
> ipvs可以通过ipvsadm命令进行管理调用，ipvsadm是一个命令行工具，可以通过【ipvsadm -L -n】来查看转发列表

### 9.同步时钟

```bash
yum install -y ntpdate
ntpdate time.windows.com
```

### 10.配置日志服务管理进程

> 普及 -> centos7内核中增加了新的进程管理器：systemd，用来取代老版本的initd
>
> 同时增加了新的日志管理服务journald(systemd-journald.service)，用于代替老版本的rsyslog日志服务
>
> 此处我们使用journald接管日志，他的存储目录默认/run/log，但重启后会清空，可以通过指定永久存储来避过这个缺点，首先使用他的永久保存需要增加一个保存目录，如下

```bash
mkdir /var/log/journal 								#持久化保存日志的目录

mkdir /etc/systemd/journald.conf.d  	#创建配置文件并增加配置内容
cat > /etc/systemd/journald.conf.d/99-prophet.conf <<EOF
[Journal]
Storage=persistent  		#持久化保存到磁盘
Compress=yes						#压缩历史日志
SyncIntervalSec=5m
RateLimitInterval=30s
RateLimitBurst=1000
SystemMaxUse=10G				#最大占用空间 10G
SystemMaxFileSize=200M	#单日志文件最大 200M
MaxRetentionSec=2week   #日志保存时间2周
ForwardToSyslog=no			#不将日志转发到 syslog
EOF

# 重启journald服务，使之生效
systemctl restart systemd-journald
```

### 11.升级linux内核

```bash
# centos7默认是3.10版本，对kubernetes支持不太好、不稳定，所以升级到4.4以上即可
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
yum --enablerepo=elrepo-kernel install -y kernel-lt
grub2-set-default 'CentOS Linux (5.4.191-1.el7.elrepo.x86_64) 7 (Core)'      #设置使用5.4.191内核启动系统，重启可以发现生效了
```

## 二、安装docker服务

### 12.增加yum源用于安装docker

```bash
# 安装必要依赖
yum install -y yum-utils device-mapper-persistent-data lvm2
# 添加阿里云的 docker-ce yum源，避免连接到国外无法现在成功
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo 
# 重建 yum 缓存 
yum makecache fast
# 查看可用 docker 版本
yum list docker-ce.x86_64 --showduplicates | sort -r
```

> 替代：以上命令更换yum源，可以用一行代码替代：wget http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo

### 13.安装docker-ce

```bash
# 根据以上列出的支持版本，此处选择docker-ce-20.10.8-3.el7版本【注意版本号不包含“:”与之前的数字】
yum install -y docker-ce-20.10.8-3.el7

# 确保网络模块开机自动加载：如果执行报错或提示不存在，则执行如下的的命令
lsmod | grep overlay
lsmod | grep br_netfilter
```

> #以上两行命令出错，执行一下命令
>
> cat > /etc/modules-load.d/docker.conf <<EOF 
> overlay 
> br_netfilter 
> EOF
>
> modprobe overlay 
> modprobe br_netfilter

### 14.优化docker配置文件

```bash
# 更改docker拉取镜像的源：使用阿里云的加速需要指定自己账号创建的加速，我用的自己账号的lj408226003
# native.cgroupdriver=systemd是因为docker默认的cgroup驱动配置的不是systemd进程管理器，但k8s推荐使用systemd，所以指定使用systemd
# 此处确保docker使用systemd，后续安装k8s时也会进行相应的配置
mkdir -p /etc/docker	#首次创建目录
# 除以上配置还有些优化配置并指定了数据存储目录
cat > /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://mtu7rhzd.mirror.aliyuncs.com"],
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ],
  "data-root": "/data/docker"
}
EOF
```

### 15.限制docker生成core日志

```bash
vi /lib/systemd/system/docker.service
# 替换第13行，增加了 --default-ulimit core=0:0 ，用于限制不让docker产生core日志，因为core日志没什么用基本都是资源不足等警告，但占用空间特别大
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --default-ulimit core=0:0
# 重新加载 & 重启docker
systemctl daemon-reload
systemctl enable --now docker
```

## 三、安装kubeadm/kubelet/kubectl

### 16.增加yum源用于安装k8s

```bash
# 使用阿里云源
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

# 重建yum缓存
yum makecache fast
```

### 17.kubeadm/kubelet/kubectl

```bash
# 查看支持安装的版本
yum list kubeadm --showduplicates | sort -r  
# 选择1.21.3版本，注意这里的【-0】是查找支持版本时列出来的，所以也需要指定
yum -y install kubeadm-1.21.3-0 kubelet-1.21.3-0 kubectl-1.21.3-0
# 设置kubelet自启动，kubelet运行在各个节点，负责创建pod、运行容器等，所以需要自启动
sudo systemctl enable --now kubelet
```

> 注意：kubectl只有master安装即可，作为worker的node节点，kubectl无法使用，可以不装

### 18.安装脚本自动补全工具

```bash
# 安装bash自动补全插件
yum install bash-completion -y
# 设置 kubectl 与 kubeadm 命令补全，下次 login 生效
kubectl completion bash >/etc/bash_completion.d/kubectl
kubeadm completion bash > /etc/bash_completion.d/kubeadm
```

> 注意：只需master安装即可，因为子节点不会执行kubeadm和kubectl命令

### 19.准备安装k8s master节点

> 普及：使用kubeadm安装k8s只需两步，一是通过kubeadm init安装master，二是通过kubeadm join将子节点node加入到k8s集群，即加入由master管理
>
> 因为初始化时会从google仓库拉取镜像文件，所以可以提前拉取或从国内阿里云拉取，下边配置的是阿里云
>
> 【但可以提前拉取下载下来，在通过docker导入，速度会快很多】
>
> 

```bash
# 查看安装--kubernetes-version v1.21.3需要拉取哪些docker镜像
kubeadm config images list --kubernetes-version v1.21.3

	k8s.gcr.io/kube-apiserver:v1.21.3
	k8s.gcr.io/kube-controller-manager:v1.21.3
	k8s.gcr.io/kube-scheduler:v1.21.3
	k8s.gcr.io/kube-proxy:v1.21.3
	k8s.gcr.io/pause:3.4.1
	k8s.gcr.io/etcd:3.4.13-0
	k8s.gcr.io/coredns/coredns:v1.8.0
```

> 总共7个镜像，所以我们写一段脚本，来快速拉取镜像、拷贝到其他节点、导入到docker

```bash
# 编写一个拉取镜像的sh脚本
vi pullimages.sh

#!/bin/bash
# pull images
 
ver=v1.21.3
registry=registry.cn-hangzhou.aliyuncs.com/google_containers
images=`kubeadm config images list --kubernetes-version=$ver |awk -F '/' '{print $2}'`
 
for image in $images
do
if [ $image != coredns ];then
    docker pull ${registry}/$image
    if [ $? -eq 0 ];then
        docker tag ${registry}/$image k8s.gcr.io/$image
        docker rmi ${registry}/$image
    else
        echo "ERROR: 下载镜像报错，$image"
    fi
else
    docker pull coredns/coredns:1.8.0
    docker tag coredns/coredns:1.8.0  k8s.gcr.io/coredns/coredns:v1.8.0
    docker rmi coredns/coredns:1.8.0
fi
done   

# 授权并执行
chmod +x pullimages.sh && ./pullimages.sh
# 查看镜像
docker images
# 导出镜像到当前目录的k8s-images.tar包
docker save $(docker images | grep -v REPOSITORY | awk 'BEGIN{OFS=":";ORS=" "}{print $1,$2}') -o k8s-images.tar
# 拷贝到node节点
scp k8s-images.tar root@kubernetes-node1:~
scp k8s-images.tar root@kubernetes-node2:~
# 在其他node节点导入到
docker load -i k8s-images.tar
```

### 20.修改k8s进程管理为systemd

```bash
# 修改 kubelet 配置默认 cgroup driver
#【上边已经把docker的cgroup驱动修改成了systemd，这里也要确保kubernetes的也是systemd】
mkdir /var/lib/kubelet

cat > /var/lib/kubelet/config.yaml <<EOF
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
EOF
```

### 21.kubeadm init 初始化

> 注意：修改init默认配置kubeadm-config.yaml和修改k8s使用ipvs模式进行网络数据转发

```bash
# 生成kubeadm初始化配置文件 【其实下边的命令也指定了这些所以可以不用生成配置文件，此处生成配置文件只是方便后续查看】
# 下边就是k8s的kube-proxy使用ipvs模式的配置
# 将kubeadm init的默认配置写入配置文件
kubeadm config print init-defaults > kubeadm-config.yaml

# 修改内容为如下
localAPIEndpoint:
  advertiseAddress: 192.168.9.185		#master节点的ip地址
nodeRegistration:
  name: kubernetes-master						#master的hostname，默认叫node
kubernetesVersion: 1.21.3						#当前要安装的k8s版本
networking:
  podSubnet: 10.244.0.0/16					#10.244.0.0/16 是 flannel 固定使用的 IP 段，所以此处设置成什么取决于CNI网络组件的要求
  serviceSubnet: 10.96.0.0/12				#这个是k8s的service使用网段

# 以下为修改kube-proxy使用ipvs进行网络规则转发，默认是iptables
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration  
mode: ipvs

# 测试初始配置是否正常 
kubeadm init phase preflight --config kubeadm-config.yaml

# 执行初始化
# --ignore-preflight-errors=2 表示安装过程出现错误可以忽略，2应该是表示可以忽略2个错误的出现，即出现错误不中断初始化
kubeadm init --config=kubeadm-config.yaml --ignore-preflight-errors=2 --upload-certs | tee kubeadm-init.log

# 注意
# kubeadm init 失败后，修正好配置后可以通过kubeadm reset重新初始化
# 重试之前需手动删除遗留历史文件
# rm -rf $HOME/.kube
```

> ***此命令也可以做初始化操作，但不方便查看日志，所以知道即可***
>
> kubeadm init \  
>   --apiserver-advertise-address 192.168.9.185 \ #apiserver节点地址，即master的ip
>   --image-repository registry.aliyuncs.com/google_containers \  #k8s拉取镜像使用阿里云
>   --kubernetes-version=v1.21.3 \  #k8s版本
>
>   #k8s的svc网络的通讯网段（k8s子网之一），这里当然值得一般是service组件被分配的ip地址了，
>
>   #用于给pod暴露外网端口，当然这个会通过ipvs或iptable配置转发以实现对外路由，而内部也是通过kubeproxy来协调
>
>   --service-cidr=10.96.0.0/12 \ 
>
>   #k8s的pod之间的网络的网段（k8s子网之一），pod内部是docker容器，他们的ip会用这个网段组网，
>
>   #而正好flannel插件的网段就是10.244.0.1正好作为所有pod内容器的网关，即容器通过flannel与外界通信，而flannel则和kubeproxy进行组网达到实现互通的效果。
>
>   --pod-network-cidr=10.244.0.0/16

### 22.kubeadm join 加入集群

```bash
# 初始化完成后按提示执行系列命令

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
  
# 接着根据提示拷贝admin.config的内容到当前用户的$HOME/.kube中并授权（为了使当前用户使用kubelet操作服务器）
# 而如果有其他linux子账号(如账号lij),需要使用kubernetes的访问权限，需要将如下文件拷贝到对应的子账户的home目录
su - lij
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/admin.conf
sudo chown $(id -u):$(id -g) $HOME/.kube/admin.conf
echo "export KUBECONFIG=$HOME/.kube/admin.conf" >> ~/.bashrc
exit

# 配置 master 认证：
# 如果不配置会提示如下输出：The connection to the server localhost:8080 was refused - did you specify the right host or port?
# 此时 master 节点已经初始化成功，但是还未安装网络组件，还无法与其他节点通讯
echo 'export KUBECONFIG=/etc/kubernetes/admin.conf' >> /etc/profile 
. /etc/profile

# 在子节点node上执行加入集群的操作
kubeadm join 192.168.9.185:6443 --token abcdef.0123456789abcdef \
        --discovery-token-ca-cert-hash sha256:44b965b36594ccd9bacc66fc8478ccf833a481185752996f416ada8921d32112

# 如果没有token了可以使用命令创建token
# kubeadm token create --print-join-command
```

### 23.k8s安装CNI网络插件

> 主流插件包括flannel和calico
>
> 安装完后发现通过kubectl get nodes 查看各个节点状态都是NotReady，是因为网络还没有连通，
>
> 本例使用flannel进行连通，也可以使用calico，注意都是k8s的插件，负责pod之间通信转发的，具体可以查百度。
>
> flannel是一组daemonset管理的pod，即每个node节点都会运行一个flannel的pod，用来转发各个节点的通信
>
> 所以安装了flannel各个节点才能互相识别，可以粗略的认为flannel就是pod网络与外界交互的桥梁，类似虚拟网卡的效果

```bash
# 注意：如果安装kube-flannel时无法下载镜像，可以打开kube-flannel.yaml
# 修改sed -i 's#quay.io/coreos/flannel#quay.mirrors.ustc.edu.cn/coreos/flannel#' /opt/yaml/kube-flannel.yaml
# 将quay.io换成quay.mirrors.ustc.edu.cn（中科大）的镜像，然后再安装
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
# 安装flannel，这个过程会拉取镜像docker pull quay.io/coreos/flannel:v0.14.0
kubectl apply -f kube-flannel.yml  
```

> 安装完成后，等一会各个pod起来后就会发现集群kubectl get nodes已经正常了。

## 四、部署kubernetes-dashboard

```bash
curl -o recommended.yaml https://raw.githubusercontent.com/kubernetes/dashboard/v2.4.0/aio/deploy/recommended.yaml

# 默认 Dashboard 只能集群内部访问，修改 Service 为 NodePort 类型，暴露到外部
type: NodePort

# 安装dashboard，这个过程会拉取镜像docker pull kubernetesui/dashboard:v2.4.1和docker pull kubernetesui/metrics-scraper:v1.0.6
kubectl apply -f recommended.yaml

# 查看安装dashboard后自动安装的k8s的service的端口,然后直接https://master-ip:nodePord
kubectl get svc -A
```

> 注意：安装完dashboard并不会安装账号，所以需要手动创建一下

```bash
# 创建 service account 并绑定默认 cluster-admin 管理员集群角色
kubectl create serviceaccount dashboard-admin -n kube-system
kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin
# 查看token
kubectl describe secrets -n kube-system $(kubectl -n kube-system get secret | awk '/dashboard-admin/{print $1}')
```

## 五、NFS文件系统服务安装

```bash
# nfs文件系统我布置在了master节点

# 在每个机器。
yum install -y nfs-utils

# master 执行以下命令 
echo "/nfs/data/ *(insecure,rw,sync,no_root_squash)" > /etc/exports

# master 执行以下命令，启动 nfs 服务;创建共享目录
mkdir -p /nfs/data

# 在master执行
systemctl enable rpcbind
systemctl enable nfs-server
systemctl start rpcbind
systemctl start nfs-server

# 在master执行使配置生效
exportfs -r

# 在master执行，检查配置是否生效
[root@k8s-master ~]# exportfs
/nfs/data       <world>

# 在worker节点执行
showmount -e 192.168.9.185  # 这里需要是自己集群中配置nfs的ip（当然就是master的ip），我的放在了master节点上
mkdir -p /nfs/data
# 将 /nfs/data 目录和 当前节点的nfs文件系统进行挂载
mount -t nfs 192.168.9.185:/nfs/data /nfs/data
```

> - 创建存储yaml文件：vi storage.yaml

```bash
#准备创建一个存储类：vi storage.yaml
## 创建一个存储类   
######### ip地址和上面不一样时，记得仔细看下配置文件，有需要修改的地方 #########
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner
parameters:
  archiveOnDelete: "true"  ## 删除pv的时候，pv的内容是否要备份

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  labels:
    app: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: registry.cn-hangzhou.aliyuncs.com/lfy_k8s_images/nfs-subdir-external-provisioner:v4.0.2
          # resources:
          #    limits:
          #      cpu: 10m
          #    requests:
          #      cpu: 10m
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: k8s-sigs.io/nfs-subdir-external-provisioner
            - name: NFS_SERVER
              value: 192.168.9.185 ## 指定自己nfs服务器地址
            - name: NFS_PATH  
              value: /nfs/data  ## nfs服务器共享的目录
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.9.185
            path: /nfs/data
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: default
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: default
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: default
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
```

```bash
# 到k8s中执行
kubectl apply -f storage.yaml
# 查看结果
kubectl get sc
```

