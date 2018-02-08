# kubernetes 1.9.1
> 文中涉及到的所有文件均已上传至百度云盘。链接: https://pan.baidu.com/s/1jJMBjQA 密码: 4epq

## 环境说明

| 主机名称 | IP | 备注 | 系统 | 配置 |
| ----- |:----:|:-----:|:-----:|:----:|
| k8s-master-149 | 192.168.134.149 | master、etcd、flannel、node | CentOS7.4 x86_64 | 2C2G20G(虚拟机) |
| k8s-mastet-150 | 192.168.134.150 | master、etcd、flannel、node | CentOS7.4 x86_64 | 2C2G20G(虚拟机) |
| k8s-master-151 | 192.168.134.151 | master、etcd、flannel、node | CentOS7.4 x86_64 | 2C2G20G(虚拟机) |

## 初始化环境

```
hostnamectl set-hostname k8s-master-149
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
setenforce 0
systemctl disable firewalld
systemctl stop firewalld
```

```
#编辑 /etc/hosts 文件

vim /etc/hosts

192.168.134.149 k8s-master-149
192.168.134.150 k8s-master-150
192.168.134.151 k8s-master-151
```
# 创建验证

> 这里使用CloudFlare的PKI工具集cfssl来生成Certificate Authority(CA)证书和秘钥文件。

## 安装cfssl
```
mkdir -p /opt/cfssl
tar zxvf cfs.tar.gz
chmod +x *
```
# 创建CA证书
```
mkdir -p /opt/ssl
cd /opt/ssl

# config.json

cat << EOF >> config.json

{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "87600h"
      }
    }
  }
}

EOF
```
```
# csr.json

cat << EOF >> csr.json

{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Beijing",
      "L": "Beijing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}

EOF
```
## 生成CA证书和私钥
```
cd /opt/ssl/
/opt/cfssl/cfssl gencert -initca csr.json | /opt/cfssl/cfssljson -bare ca

[root@k8s-master-149 ssl]# ls -lt
-rw-r--r--. 1 root root 1001 2月   7 00:40 ca.csr
-rw-------. 1 root root 1679 2月   7 00:40 ca-key.pem
-rw-r--r--. 1 root root 1359 2月   7 00:40 ca.pem
-rw-r--r--. 1 root root  208 2月   7 00:39 csr.json
-rw-r--r--. 1 root root  292 2月   7 00:39 config.json
```
## 分发证书
```
# 创建证书目录
mkdir -p /etc/kubernetes/ssl
ssh k8s-master-150 "mkdir -p /etc/kubernetes/ssl"
ssh k8s-master-151 "mkdir -p /etc/kubernetes/ssl"

# 拷贝文件到证书目录下
cp ca* /etc/kubernetes/ssl

# 将文件拷贝到所有的k8s 机器上

scp ca* k8s-master-150:/etc/kubernetes/ssl/

scp ca* k8s-master-151:/etc/kubernetes/ssl/

```
# 安装Docker
> 所有服务器安装Docker，次文档中安装的docker版本为17.12.0-ce。
> 官方1.9中建议的docker版本：The validated docker versions are the same as for v1.8: 1.11.2 to 1.13.1 and 17.03.x
```
yum -y install yum-utils
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
yum list docker-ce.x86_64  --showduplicates |sort -r
yum -y install docker-17.12.0-ce

docker version

Client:
 Version:	17.12.0-ce
 API version:	1.35
 Go version:	go1.9.2
 Git commit:	c97c6d6
 Built:	Wed Dec 27 20:10:14 2017
 OS/Arch:	linux/amd64
```
# 更改docker配置
```
cat << EOF >> /etc/systemd/system/docker.service

[Unit]
Description=Docker Application Container Engine
Documentation=http://docs.docker.com
After=network.target docker-storage-setup.service
Wants=docker-storage-setup.service

[Service]
Type=notify
Environment=GOTRACEBACK=crash
ExecReload=/bin/kill -s HUP $MAINPID
Delegate=yes
KillMode=process
ExecStart=/usr/bin/dockerd \
          $DOCKER_OPTS \
          $DOCKER_STORAGE_OPTIONS \
          $DOCKER_NETWORK_OPTIONS \
          $DOCKER_DNS_OPTIONS \
          $INSECURE_REGISTRY
LimitNOFILE=1048576
LimitNPROC=1048576
LimitCORE=infinity
TimeoutStartSec=1min
Restart=on-abnormal

[Install]
WantedBy=multi-user.target

EOF
```

```
# 修改其他配置,低版本内核， kernel 3.10.x  配置使用 overlay2

cat << EOF >> /etc/docker/daemon.json

{
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}

EOF

# 添加docker配置文件
mkdir -p /etc/systemd/system/docker.service.d/

cat << EOF >> /etc/systemd/system/docker.service.d/docker-options.conf

[Service]
Environment="DOCKER_OPTS=--insecure-registry=10.10.0.0/16 \
    --graph=/opt/docker --log-opt max-size=50m --log-opt max-file=5"
EOF

cat << EOF >> /etc/systemd/system/docker.service.d/docker-dns.conf

[Service]
Environment="DOCKER_DNS_OPTIONS=\
    --dns 10.10.0.2 --dns 114.114.114.114  \
    --dns-search default.svc.cluster.local --dns-search svc.cluster.local  \
    --dns-opt ndots:2 --dns-opt timeout:2 --dns-opt attempts:2"

EOF
```
```
# 重新读取配置，启动 docker 
systemctl daemon-reload
systemctl start docker
systemctl enable docker
```
# ETCD集群
## 安装etcd
> 官方地址 https://github.com/coreos/etcd/releases
```
tar zxvf etcd-v3.2.11-linux-amd64.tar.gz
cd etcd-v3.2.11-linux-amd64

cp etcd etcdctl /usr/bin/
scp etcd etcdctl k8s-master-150:/usr/bin
scp etcd etcdctl k8s-master-151:/usr/bin

source /etc/profile
ssh k8s-master-150 "source /etc/profile"
ssh k8s-master-151 "source /etc/profile"
```
## 创建ETCD证书
```
cat << EOF >> /opt/ssl/etcd-csr.json

{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "192.168.134.149",
    "192.168.134.150",
    "192.168.134.151"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Beijing",
      "L": "Beijing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}

EOF
```
```
# 生成 etcd 密钥
/opt/cfssl/cfssl gencert -ca=/opt/ssl/ca.pem \
  -ca-key=/opt/ssl/ca-key.pem \
  -config=/opt/ssl/config.json \
  -profile=kubernetes etcd-csr.json | /opt/cfssl/cfssljson -bare etcd
  
# 拷贝到etcd服务器

cp etcd*.pem /etc/kubernetes/ssl/

scp etcd*.pem k8s-master-150:/etc/kubernetes/ssl/

scp etcd*.pem k8s-master-151:/etc/kubernetes/ssl/

```

## 修改ETCD配置
```
mkdir -p /opt/etcd
cat << EOF >> /etc/systemd/system/etcd.service

[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
WorkingDirectory=/opt/etcd/
User=root
# set GOMAXPROCS to number of processors
ExecStart=/usr/bin/etcd \
  --name=etcd1 \
  --cert-file=/etc/kubernetes/ssl/etcd.pem \
  --key-file=/etc/kubernetes/ssl/etcd-key.pem \
  --peer-cert-file=/etc/kubernetes/ssl/etcd.pem \
  --peer-key-file=/etc/kubernetes/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
  --peer-trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
  --initial-advertise-peer-urls=https://192.168.134.149:2380 \
  --listen-peer-urls=https://192.168.134.149:2380 \
  --listen-client-urls=https://192.168.134.149:2379,http://127.0.0.1:2379 \
  --advertise-client-urls=https://192.168.134.149:2379 \
  --initial-cluster-token=k8s-etcd-cluster \
  --initial-cluster=etcd1=https://192.168.134.149:2380,etcd2=https://192.168.134.150:2380,etcd3=https://192.168.134.151:2380 \
  --initial-cluster-state=new \
  --data-dir=/opt/etcd/
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

EOF
```
> 注意每台etcd服务器的配置，修改相应的配置项：**--name=etcd1、IP地址**
## 启动ETCD
```
systemctl daemon-reload
systemctl enable etcd
systemctl start etcd
systemctl status etcd
```
## 验证etcd集群
```
etcdctl --endpoints=https://192.168.134.149:2379 \ 
        --cert-file=/etc/kubernetes/ssl/etcd.pem \
        --ca-file=/etc/kubernetes/ssl/ca.pem \
        --key-file=/etc/kubernetes/ssl/etcd-key.pem \
        cluster-health
        
member 4e6f9c30e7817ad5 is healthy: got healthy result from https://192.168.134.150:2379
member 6c4f6ad990d9782b is healthy: got healthy result from https://192.168.134.149:2379
member 91f9a75d83569a04 is healthy: got healthy result from https://192.168.134.151:2379
cluster is healthy

etcdctl --endpoints=https://192.168.134.149:2379 \ 
        --cert-file=/etc/kubernetes/ssl/etcd.pem \
        --ca-file=/etc/kubernetes/ssl/ca.pem \
        --key-file=/etc/kubernetes/ssl/etcd-key.pem \
        member list

4e6f9c30e7817ad5: name=etcd2 peerURLs=https://192.168.134.150:2380 clientURLs=https://192.168.134.150:2379 isLeader=true
6c4f6ad990d9782b: name=etcd1 peerURLs=https://192.168.134.149:2380 clientURLs=https://192.168.134.149:2379 isLeader=false
91f9a75d83569a04: name=etcd3 peerURLs=https://192.168.134.151:2380 clientURLs=https://192.168.134.151:2379 isLeader=false

```
# 配置kubernetes集群
> Master 需要部署 kube-apiserver , kube-scheduler , kube-controller-manager 这三个组件。 kube-scheduler 作用是调度pods分配到那个node里，简单来说就是资源调度。 kube-controller-manager 作用是 对 deployment controller , replication controller, endpoints controller, namespace controller, and serviceaccounts controller等等的循环控制，与kube-apiserver交互。 Node 需要部署 kubelet ， kube-proxy 。本次文档中服务既是master，也是node，所有组件都需要安装。
## 安装组件
```
tar -xzvf kubernetes-server-linux-amd64.tar.gz

cd kubernetes

cp -r server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler,kubectl,kube-proxy,kubelet} /usr/local/bin/

scp server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler,kubectl,kube-proxy,kubelet} k8s-master-150:/usr/local/bin/

scp server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler,kubectl,kube-proxy,kubelet} k8s-master-151:/usr/local/bin/
```
## 创建证书
> kubectl 与 kube-apiserver 的安全端口通信，需要为安全通信提供 TLS 证书和秘钥。
```
cat << EOF >> /opt/ssl/admin-csr.json

{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Beijing",
      "L": "Beijing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}

EOF
```
```
# 生成 admin 证书和私钥
cd /opt/ssl/

/opt/cfssl/cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
  -config=/opt/ssl/config.json \
  -profile=kubernetes admin-csr.json | /opt/cfssl/cfssljson -bare admin

# 分发证书
scp admin*.pem k8s-master-149:/etc/kubernetes/ssl/

scp admin*.pem k8s-master-150:/etc/kubernetes/ssl/

scp admin*.pem k8s-master-151:/etc/kubernetes/ssl/

```
## 配置 kubectl kubeconfig 文件
```
# 配置 kubernetes 集群

kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443

# 配置 客户端认证

kubectl config set-credentials admin \
  --client-certificate=/etc/kubernetes/ssl/admin.pem \
  --embed-certs=true \
  --client-key=/etc/kubernetes/ssl/admin-key.pem
  
kubectl config set-context kubernetes \
  --cluster=kubernetes \
  --user=admin

kubectl config use-context kubernetes

ls /root/.kube/config
-rw-------. 1 root root 6283 2月   7 01:19 /root/.kube/config
```
## 创建 kubernetes 证书
```
cat << EOF >> /opt/ssl/kubernetes-csr.json
{
  "CN": "kubernetes",
  "hosts": [
    "127.0.0.1",
    "192.168.134.149",
    "192.168.134.150",
    "192.168.134.151",
    "10.10.0.1",
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Beijing",
      "L": "Beijing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}

EOF
# 10.10.0.1 为 kubernetes SVC 的 IP，一般是部署网络的第一个IP , 如: 10.10.0.1, 在启动完成后，我们使用 kubectl get svc ，就可以查看到
```
## 生成 kubernetes 证书和私钥
```
/opt/cfssl/cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
  -config=/opt/ssl/config.json \
  -profile=kubernetes kubernetes-csr.json | /opt/cfssl/cfssljson -bare kubernetes

# 分发证书
scp kubernetes*.pem k8s-master-149:/etc/kubernetes/ssl/

scp kubernetes*.pem k8s-master-150:/etc/kubernetes/ssl/

scp kubernetes*.pem k8s-master-151:/etc/kubernetes/ssl/

```
## 配置 kube-apiserver
> kubelet 首次启动时向 kube-apiserver 发送 TLS Bootstrapping 请求，kube-apiserver 验证 kubelet 请求中的 token 是否与它配置的 token 一致，如果一致则自动为 kubelet生成证书和秘钥。
```
# 生成 token

TOKEN=`head -c 16 /dev/urandom | od -An -t x | tr -d ' '`

# 创建 token.csv 文件

cat << EOF >> /opt/ssl/token.csv

$TOKEN,kubelet-bootstrap,10001,"system:kubelet-bootstrap"

# 拷贝

scp token.csv k8s-master-149:/etc/kubernetes/

scp token.csv k8s-master-150:/etc/kubernetes/

scp token.csv k8s-master-151:/etc/kubernetes/

# 生成高级审核配置文件

cat << EOF >> /etc/kubernetes/audit-policy.yaml

# Log all requests at the Metadata level.
apiVersion: audit.k8s.io/v1beta1
kind: Policy
rules:
- level: Metadata
EOF

# 拷贝

scp audit-policy.yaml k8s-master-149:/etc/kubernetes/
scp audit-policy.yaml k8s-master-150:/etc/kubernetes/
scp audit-policy.yaml k8s-master-151:/etc/kubernetes/

```
## 创建 kube-apiserver.service 文件
```
# 配置为 各自的本地 IP

cat << EOF >> /etc/systemd/system/kube-apiserver.service

[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
User=root
ExecStart=/usr/local/bin/kube-apiserver \
  --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,NodeRestriction \
  --advertise-address=192.168.134.149 \
  --allow-privileged=true \
  --apiserver-count=3 \
  --audit-policy-file=/etc/kubernetes/audit-policy.yaml \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=3 \
  --audit-log-maxsize=100 \
  --audit-log-path=/var/log/kubernetes/audit.log \
  --authorization-mode=Node,RBAC \
  --bind-address=0.0.0.0 \
  --secure-port=6443 \
  --client-ca-file=/etc/kubernetes/ssl/ca.pem \
  --enable-swagger-ui=true \
  --etcd-cafile=/etc/kubernetes/ssl/ca.pem \
  --etcd-certfile=/etc/kubernetes/ssl/etcd.pem \
  --etcd-keyfile=/etc/kubernetes/ssl/etcd-key.pem \
  --etcd-servers=https://192.168.134.149:2379,https://192.168.134.150:2379,https://192.168.134.151:2379 \
  --event-ttl=1h \
  --kubelet-https=true \
  --insecure-bind-address=0.0.0.0 \
  --insecure-port=8080 \
  --service-account-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --service-cluster-ip-range=10.20.0.0/16 \
  --service-node-port-range=10000-32000 \
  --tls-cert-file=/etc/kubernetes/ssl/kubernetes.pem \
  --tls-private-key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
  --enable-bootstrap-token-auth \
  --token-auth-file=/etc/kubernetes/token.csv \
  --v=2
Restart=on-failure
RestartSec=5
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

EOF
```
## 启动 kube-apiserver
```
systemctl daemon-reload
systemctl enable kube-apiserver
systemctl start kube-apiserver
systemctl status kube-apiserver
```
## 配置 kube-controller-manager
```
cat << EOF >> /etc/systemd/system/kube-controller-manager.service

[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \
  --address=0.0.0.0 \
  --master=http://127.0.0.1:8080 \
  --allocate-node-cidrs=true \
  --service-cluster-ip-range=10.20.0.0/16 \
  --cluster-cidr=10.30.0.0/16 \
  --cluster-name=kubernetes \
  --cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem \
  --cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --root-ca-file=/etc/kubernetes/ssl/ca.pem \
  --leader-elect=true \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target

EOF
```

## 启动 kube-controller-manager
```
systemctl daemon-reload
systemctl enable kube-controller-manager
systemctl start kube-controller-manager
systemctl status kube-controller-manager
```
## 配置 kube-scheduler
```
cat << EOF >> /etc/systemd/system/kube-scheduler.service

[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \
  --address=0.0.0.0 \
  --master=http://127.0.0.1:8080 \
  --leader-elect=true \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
## 启动 kube-scheduler
```
systemctl daemon-reload
systemctl enable kube-scheduler
systemctl start kube-scheduler
systemctl status kube-scheduler
```
## 验证 Master 节点
```
[root@k8s-master-149 kubernetes]# kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok
etcd-1               Healthy   {"health":"true"}
etcd-2               Healthy   {"health":"true"}
etcd-0               Healthy   {"health":"true"}
scheduler            Healthy   ok
```
## 配置 kubelet
> kubelet 启动时向 kube-apiserver 发送 TLS bootstrapping 请求，需要先将 bootstrap token 文件中的 kubelet-bootstrap 用户赋予 system:node-bootstrapper 角色，然后 kubelet 才有权限创建认证请求(certificatesigningrequests)。

```
# 先创建认证请求
# user 为 master 中 token.csv 文件里配置的用户
# 只需创建一次就可以

kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap
```
## 创建 kubelet kubeconfig 文件
```
# 配置集群

kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=bootstrap.kubeconfig

# 配置客户端认证

kubectl config set-credentials kubelet-bootstrap \
  --token=d2d7f3a19490ff667fbe94b0f31f9967 \
  --kubeconfig=bootstrap.kubeconfig

# 配置关联

kubectl config set-context default \
  --cluster=kubernetes \
  --user=kubelet-bootstrap \
  --kubeconfig=bootstrap.kubeconfig
 
kubectl config use-context default --kubeconfig=bootstrap.kubeconfig

# 拷贝生成的 bootstrap.kubeconfig 文件

scp bootstrap.kubeconfig k8s-master-149:/etc/kubernetes/

scp bootstrap.kubeconfig k8s-master-150:/etc/kubernetes/

scp bootstrap.kubeconfig k8s-master-151:/etc/kubernetes/
```
## 创建 kubelet.service 文件
> 配置为 node 本机 IP
```
# 创建 kubelet 目录
mkdir /var/lib/kubelet

cat << EOF >>  /etc/systemd/system/kubelet.service

[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
ExecStart=/usr/local/bin/kubelet \
  --cgroup-driver=cgroupfs \
  --hostname-override=k8s-master-149 \
  --pod-infra-container-image=gcr.io/google_containers/pause-amd64:3.1 \
  --experimental-bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig \
  --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \
  --cert-dir=/etc/kubernetes/ssl \
  --cluster_dns=10.10.0.2 \
  --cluster_domain=cluster.local. \
  --hairpin-mode promiscuous-bridge \
  --allow-privileged=true \
  --fail-swap-on=false \
  --serialize-image-pulls=false \
  --logtostderr=true \
  --max-pods=512 \
  --v=2

[Install]
WantedBy=multi-user.target

EOF

# gcr.io/google_containers/pause-amd64:3.1 这个是pod的基础镜像,默认的镜像仓库在google，由于国内网络被和谐，需要自行下载镜像导入。
docker load -i pause-amd64

[root@k8s-master-149 kubernetes]# docker images
REPOSITORY                                            TAG                 IMAGE ID            CREATED             SIZE
gcr.io/google_containers/pause-amd64                  3.1                 da86e6ba6ca1        7 weeks ago         742kB

```
## 启动 kubelet
```
systemctl daemon-reload
systemctl enable kubelet
systemctl start kubelet
systemctl status kubelet
```
## 配置 TLS 认证
```
# 查看 csr 的名称

[root@k8s-master-149 kubernetes]# kubectl get csr
NAME                                                   AGE       REQUESTOR           CONDITION
node-csr-UstWvAjJXHrfgNxDgtwl4_xHMBNONvEfAO1f_l-x9cM   15m       kubelet-bootstrap   Approved,Issued

# 增加 认证

kubectl get csr | grep Pending | awk '{print $1}' | xargs kubectl certificate approve

# 验证 nodes
[root@k8s-master-149 kubernetes]# kubectl get nodes
NAME             STATUS    ROLES     AGE       VERSION
k8s-master-149   Ready     <none>    1d        v1.9.1
k8s-master-150   Ready     <none>    1d        v1.9.1
k8s-master-151   Ready     <none>    1d        v1.9.1

# 成功以后会自动生成配置文件与密钥
[root@k8s-master-149 kubernetes]# ls -lt /etc/kubernetes/kubelet.kubeconfig
-rw-------. 1 root root 2276 2月   7 01:39 /etc/kubernetes/kubelet.kubeconfig


# 密钥文件  这里注意如果 csr 被删除了，请删除如下文件，并重启 kubelet 服务
[root@k8s-master-149 kubernetes]# ls -lt /etc/kubernetes/ssl/kubelet*
-rw-r--r--. 1 root root 1050 2月   7 01:39 /etc/kubernetes/ssl/kubelet-client.crt
-rw-------. 1 root root  227 2月   7 01:39 /etc/kubernetes/ssl/kubelet-client.key
-rw-r--r--. 1 root root 1131 2月   7 01:39 /etc/kubernetes/ssl/kubelet.crt
-rw-------. 1 root root 1675 2月   7 01:39 /etc/kubernetes/ssl/kubelet.key
```
## 配置 kube-proxy
### 创建 kube-proxy 证书
```
cat << EOF >> /opt/ssl/kube-proxy-csr.json

{
  "CN": "system:kube-proxy",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Beijing",
      "L": "Beijing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}

EOF
```
```
# 生成 kube-proxy 证书和私钥
/opt/cfssl/cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
  -config=/opt/ssl/config.json \
  -profile=kubernetes  kube-proxy-csr.json | /opt/cfssl/cfssljson -bare kube-proxy
  
# 分发证书
scp kube-proxy* k8s-master-149:/etc/kubernetes/ssl
scp kube-proxy* k8s-master-150:/etc/kubernetes/ssl
scp kube-proxy* k8s-master-151:/etc/kubernetes/ssl

```
## 创建 kube-proxy kubeconfig 文件
```
# 配置集群

kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=kube-proxy.kubeconfig

# 配置客户端认证

kubectl config set-credentials kube-proxy \
  --client-certificate=/etc/kubernetes/ssl/kube-proxy.pem \
  --client-key=/etc/kubernetes/ssl/kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig

# 配置关联

kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig

kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig

# 拷贝到node

scp kube-proxy.kubeconfig k8s-master-149:/etc/kubernetes/

scp kube-proxy.kubeconfig k8s-master-150:/etc/kubernetes/

scp kube-proxy.kubeconfig k8s-master-151:/etc/kubernetes/
```

## 创建 kube-proxy.service 文件
> 1.9 官方 ipvs 已经 beta , 尝试开启 ipvs 测试一下.
** 打开 ipvs 需要安装 ipvsadm 软件， 在 node 中安装 ** 
```
yum install ipvsadm -y

# 创建 kube-proxy 目录
mkdir -p /var/lib/kube-proxy

cat << EOF >> /etc/systemd/system/kube-proxy.service

[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/usr/local/bin/kube-proxy \
  --bind-address=192.168.134.149 \
  --hostname-override=k8s-master-149 \
  --cluster-cidr=10.30.0.0/16 \
  --masquerade-all \
  --feature-gates=SupportIPVSProxyMode=true \
  --proxy-mode=ipvs \
  --ipvs-min-sync-period=5s \
  --ipvs-sync-period=5s \
  --ipvs-scheduler=rr \
  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig \
  --logtostderr=true \
  --v=2
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

# 节点修改为自己的ip和主机名称
```
## 启动 kube-proxy
```
systemctl daemon-reload
systemctl enable kube-proxy
systemctl start kube-proxy
systemctl status kube-proxy
```
```
# 检查  ipvs
[root@k8s-master-149 ssl]# ipvsadm -L -n
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.20.0.1:443 rr persistent 10800
  -> 192.168.134.149:6443         Masq    1      0          0
  -> 192.168.134.150:6443         Masq    1      0          0
  -> 192.168.134.151:6443         Masq    1      0          0
```

## 配置 Flannel 网络
```
# 安装flannel
tar zxvf flannel-v0.9.1-linux-amd64.tar.gz
scp flanneld mk-docker-opts.sh k8s-master-149:/usr/bin/
scp flanneld mk-docker-opts.sh k8s-master-149:/usr/bin/
scp flanneld mk-docker-opts.sh k8s-master-149:/usr/bin/

# 配置证书
cd /opt/ssl

/opt/cfssl/cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem   -ca-key=/etc/kubernetes/ssl/ca-key.pem   -config=/opt/ssl/config.json   -profile=kubernetes flanneld-csr.json | /opt/cfssl/cfssljson -bare flanneld

# 分发证书
scp flanneld*.pem k8s-master-149:/etc/kubernetes/ssl/
scp flanneld*.pem k8s-master-150:/etc/kubernetes/ssl/
scp flanneld*.pem k8s-master-151:/etc/kubernetes/ssl/

```
## 配置 Flannel 启动文件
```
cat << EOF >> /etc/systemd/system/flanneld.service
[Unit]
Description=Flanneld overlay address etcd agent
After=network.target
After=network-online.target
Wants=network-online.target
After=etcd.service
Before=docker.service

[Service]
Type=notify
ExecStart=/usr/bin/flanneld \
  -etcd-cafile=/etc/kubernetes/ssl/ca.pem \
  -etcd-certfile=/etc/kubernetes/ssl/flanneld.pem \
  -etcd-keyfile=/etc/kubernetes/ssl/flanneld-key.pem \
  -etcd-endpoints=https://192.168.134.149:2379,https://192.168.134.150:2379,https://192.168.134.151:2379 \
  -etcd-prefix=/flannel/network
  -iface=ens33
ExecStartPost=/usr/bin/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/docker
Restart=on-failure

[Install]
WantedBy=multi-user.target
RequiredBy=docker.service

EOF

# 配置 flannel 网段

etcdctl --endpoints=https://192.168.134.149:2379,https://192.168.134.149:2379,https://192.168.134.149:2379\
        --cert-file=/etc/kubernetes/ssl/etcd.pem \
        --ca-file=/etc/kubernetes/ssl/ca.pem \
        --key-file=/etc/kubernetes/ssl/etcd-key.pem \
        set /flannel/network/config \ '{"Network":"10.30.0.0/16","SubnetLen":24,"Backend":{"Type":"xvlan"}}'
```

## 启动 flannel 
```
systemctl daemon-reload
systemctl enable flanneld
systemctl start flanneld
systemctl status flanneld
```

## 验证 网络
```
ifconfig

flannel.1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        inet 10.30.60.0  netmask 255.255.255.255  broadcast 0.0.0.0
        inet6 fe80::802:65ff:fead:b293  prefixlen 64  scopeid 0x20<link>
        ether 0a:02:65:ad:b2:93  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 8 overruns 0  carrier 0  collisions 0
```
