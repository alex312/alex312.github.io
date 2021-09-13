<h1 align="center">k8s单master集群二进制部署</h1>

##  1.集群规划

| hostname | role        | ip            | component                                                    |
| -------- | ----------- | ------------- | ------------------------------------------------------------ |
| k8s02    | master node | 109.52.36.182 | kubectl,kube-apiserver,kube-controller-manager,kube-scheduler,kubelet,kube-proxy,etcd,flannel |
| k8s01    | worker node | 109.52.36.181 | kubelet,kube-proxy,etcd,flannel                              |
| k8s03    | worker node | 109.52.36.183 | kubelet,kube-proxy,etcd,flannel                              |

## 2.环境初始化

1. 配置hosts文件(all)

```shell
cat >> /etc/hosts << EOF
109.52.36.181 k8s01
109.52.36.182 k8s02
109.52.36.183 k8s03
EOF
```

2. ssh免登录（master）

```shell
#生成密钥
mkdir /root/.ssh && cd /root/.ssh
ssh-keygen -t rsa
#分发
for i in k8s01 k8s02 k8s03 ;  do   ssh-copy-id -i ~/.ssh/id_rsa.pub root@$i ;   done
```

3. 关闭selinux(all)

```shell
# 永久关闭
sed -i 's#enforcing#disabled#g' /etc/sysconfig/selinux
# 临时关闭
setenforce 0
```

4. 关闭swap分区(all)

```shell
swapoff -a
sed -i.bak 's/^.*centos-swap/#&/g' /etc/fstab
echo 'KUBELET_EXTRA_ARGS="--fail-swap-on=false"' > /etc/sysconfig/kubelet
```

5. 关闭防火墙(all)

```shell
systemctl disable --now firewalld
```

6. 将桥接的IPV4流量传递到iptables的链(all)

```shell
#设置
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
#生效
sysctl --system
```

7. 时钟同步(all)

```shell
#设置
yum install ntp -y
ntpdate time2.aliyun.com
#写入定时任务
*/1 * * * * ntpdate time2.aliyun.com > /dev/null 2>&1
```

8. 安装docker(all)

```shell
#配置源
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
#安装
yum install docker-ce -y
#配置加速
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://8mh75mhz.mirror.aliyuncs.com"]
}
EOF
#开机启动
sudo systemctl daemon-reload ; systemctl restart docker;systemctl enable --now docker.service

```



## 3.安装证书生成工具

```shell
# 下载
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64

# 设置执行权限
chmod +x cfssljson_linux-amd64
chmod +x cfssl_linux-amd64

# 移动到/usr/local/bin
mv cfssljson_linux-amd64 cfssljson
mv cfssl_linux-amd64 cfssl
mv cfssljson cfssl /usr/local/bin
```



## 4.部署etcd集群

1. 生成证书(master)

```shell
mkdir -p /opt/cert/ca
cd /opt/cert/ca

#CA证书配置文件
cat > ca-config.json << EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "www": {
         "expiry": "87600h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      }
    }
  }
}
EOF
#1.default是默认策略，指定证书默认有效期是1年
#2.profiles是定义使用场景，这里只是kubernetes，其实可以定义多个场景，分别指定不同的过期时间,使用场景等参数,后续签名证书时使用某个profile;
#3.signing: 表示该证书可用于签名其它证书,生成的ca.pem证书
#4.server auth: 表示client 可以用该CA 对server 提供的证书进行校验;
#5.client auth: 表示server 可以用该CA 对client 提供的证书进行验证。

cat > ca-csr.json << EOF
{
    "CN": "etcd CA",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing"
        }
    ]
}
EOF
#生成CA证书
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -

mkdir -p /opt/cert/etcd
cd /opt/cert/etcd
#etcd证书配置文件
cat > etcd-csr.json << EOF
{
    "CN": "etcd",
    "hosts": [
    "127.0.0.1",
    "109.52.36.181",
    "109.52.36.182",
    "109.52.36.183"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
          "C": "CN",
          "ST": "BeiJing",
          "L": "BeiJing"
        }
    ]
}
EOF
#生成etcd证书
cfssl gencert -ca=../ca/ca.pem -ca-key=../ca/ca-key.pem -config=../ca/ca-config.json -profile=kubernetes etcd-csr.json | cfssljson -bare etcd

```

2. 证书详解

| 证书项 | 解释     |
| ------ | -------- |
| C      | 国家     |
| ST     | 省       |
| L      | 城市     |
| O      | 组织     |
| OU     | 组织别名 |



3. 分发etcd证书(master)

```shell
for ip in k8s01 k8s02 k8s03
do
  ssh root@${ip} "mkdir -pv /etc/etcd/ssl"
  scp ../ca/ca*.pem  root@${ip}:/etc/etcd/ssl
  scp ./etcd*.pem  root@${ip}:/etc/etcd/ssl
done

for ip in k8s01 k8s02 k8s03; do
    ssh root@${ip} "ls -l /etc/etcd/ssl";
done

```

4. 下载etcd(master)

```shell
wget https://mirrors.huaweicloud.com/etcd/v3.3.24/etcd-v3.3.24-linux-amd64.tar.gz

tar xf etcd-v3.3.24-linux-amd64.tar.gz

# 分发至其他节点
for i in k8s01 k8s02 k8s03
do
scp ./etcd-v3.3.24-linux-amd64/etcd* root@$i:/usr/local/bin/
done

```

5. 注册etcd服务(all)

```shell
mkdir -pv /etc/kubernetes/conf/etcd

ETCD_NAME=`hostname`
INTERNAL_IP=`hostname -i`
INITIAL_CLUSTER=k8s01=https://109.52.36.181:2380,k8s02=https://109.52.36.182:2380,k8s03=https://109.52.36.183:2380


cat << EOF | sudo tee /usr/lib/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/ssl/etcd.pem \\
  --key-file=/etc/etcd/ssl/etcd-key.pem \\
  --peer-cert-file=/etc/etcd/ssl/etcd.pem \\
  --peer-key-file=/etc/etcd/ssl/etcd-key.pem \\
  --trusted-ca-file=/etc/etcd/ssl/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster \\
  --initial-cluster ${INITIAL_CLUSTER} \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

#--cert-file	客户端与服务器之间TLS证书文件的路径
#--key-file	客户端与服务器之间TLS密钥文件的路径
#--peer-cert-file	对等服务器TLS证书文件的路径
#--peer-key-file	对等服务器TLS密钥文件的路径
#--trusted-ca-file	签名client证书的CA证书，用于验证client证书
#--peer-trusted-ca-file	签名对等服务器证书的CA证书。
#--trusted-ca-file	签名client证书的CA证书，用于验证client证书
#--peer-trusted-ca-file	签名对等服务器证书的CA证书。
#--listen-peer-urls	与集群其它成员之间的通信地址
#--listen-client-urls	监听本地端口，对外提供服务的地址
#--initial-advertise-peer-urls	通告给集群其它节点，本地的对等URL地址
#--advertise-client-urls	客户端URL，用于通告集群的其余部分信息
#--initial-cluster	集群中的所有信息节点
#--initial-cluster-token	集群的token，整个集群中保持一致
#--initial-cluster-state	初始化集群状态，默认为new
#--data-dir	指定节点的数据存储目录

```

6. 测试集群

```shell
ETCDCTL_API=3 etcdctl \
--cacert=/etc/etcd/ssl/etcd.pem \
--cert=/etc/etcd/ssl/etcd.pem \
--key=/etc/etcd/ssl/etcd-key.pem \
--endpoints="https://109.52.36.181:2379,https://109.52.36.182:2379,https://109.52.36.183:2379" \
endpoint status --write-out='table'

+----------------------------+------------------+---------+---------+-----------+-----------+------------+
|          ENDPOINT          |        ID        | VERSION | DB SIZE | IS LEADER | RAFT TERM | RAFT INDEX |
+----------------------------+------------------+---------+---------+-----------+-----------+------------+
| https://109.52.36.181:2379 | faee0f22e92692bf |  3.3.24 |  3.1 MB |     false |        11 |     401017 |
| https://109.52.36.182:2379 | 2bd83503a26f1287 |  3.3.24 |  3.1 MB |      true |        11 |     401017 |
| https://109.52.36.183:2379 | 111ce48bcbc3e791 |  3.3.24 |  3.1 MB |     false |        11 |     401017 |
+----------------------------+------------------+---------+---------+-----------+-----------+------------+

ETCDCTL_API=3 etcdctl \
--cacert=/etc/etcd/ssl/etcd.pem \
--cert=/etc/etcd/ssl/etcd.pem \
--key=/etc/etcd/ssl/etcd-key.pem \
--endpoints="https://109.52.36.181:2379,https://109.52.36.182:2379,https://109.52.36.183:2379" \
member list --write-out='table'

+------------------+---------+-------+----------------------------+----------------------------+
|        ID        | STATUS  | NAME  |         PEER ADDRS         |        CLIENT ADDRS        |
+------------------+---------+-------+----------------------------+----------------------------+
| 111ce48bcbc3e791 | started | k8s03 | https://109.52.36.183:2380 | https://109.52.36.183:2379 |
| 2bd83503a26f1287 | started | k8s02 | https://109.52.36.182:2380 | https://109.52.36.182:2379 |
| faee0f22e92692bf | started | k8s01 | https://109.52.36.181:2380 | https://109.52.36.181:2379 |
+------------------+---------+-------+----------------------------+----------------------------+
```

## 3.部署Master node

1. 生成kube-apiserver证书

``` shell
mkdir -p ~/TLS/{etcd,k8s}
cd ~/TLS/k8s
#CA证书
cat > ca-config.json << EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
         "expiry": "87600h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      }
    }
  }
}
EOF

cat > ca-csr.json << EOF
{
    "CN": "kubernetes",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF

#生成证书
cfssl gencert -initca ca-csr.json | cfssljson -bare ca -

#使用CA签发kube-apiserver https证书
cat > server-csr.json << EOF
{
    "CN": "kubernetes",
    "hosts": [
      "10.0.0.1",
      "127.0.0.1",
      "109.52.36.181",
      "109.52.36.182",
      "109.52.36.183",
      "109.52.36.184",
      "109.52.36.185",
      "109.52.36.186",
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
            "L": "BeiJing",
            "ST": "BeiJing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF

#生成证书
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes server-csr.json | cfssljson -bare server

```

2. 部署kube-apiserver

```shell
#下载master二进制文件
https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.18.md#v1183


#解压
mkdir -p /opt/kubernetes/{bin,cfg,ssl,logs} 
tar zxvf kubernetes-server-linux-amd64.tar.gz
cd kubernetes/server/bin
cp kube-apiserver kube-scheduler kube-controller-manager /opt/kubernetes/bin
cp kubectl /usr/bin/

#创建配置文件
cat > /opt/kubernetes/cfg/kube-apiserver.conf << EOF
KUBE_APISERVER_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--etcd-servers=https://109.52.36.181:2379,https://109.52.36.182:2379,https://109.52.36.183:2379 \\
--bind-address=109.52.36.182 \\
--secure-port=6443 \\
--advertise-address=109.52.36.182 \\
--allow-privileged=true \\
--service-cluster-ip-range=10.0.0.0/24 \\
--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota,NodeRestriction \\
--authorization-mode=RBAC,Node \\
--enable-bootstrap-token-auth=true \\
--token-auth-file=/opt/kubernetes/cfg/token.csv \\
--service-node-port-range=30000-32767 \\
--kubelet-client-certificate=/opt/kubernetes/ssl/server.pem \\
--kubelet-client-key=/opt/kubernetes/ssl/server-key.pem \\
--tls-cert-file=/opt/kubernetes/ssl/server.pem  \\
--tls-private-key-file=/opt/kubernetes/ssl/server-key.pem \\
--client-ca-file=/opt/kubernetes/ssl/ca.pem \\
--service-account-key-file=/opt/kubernetes/ssl/ca-key.pem \\
--etcd-cafile=/etc/etcd/ssl/ca.pem \\
--etcd-certfile=/etc/etcd/ssl/etcd.pem \\
--etcd-keyfile=/etc/etcd/ssl/etcd-key.pem \\
--audit-log-maxage=30 \\
--audit-log-maxbackup=3 \\
--audit-log-maxsize=100 \\
--audit-log-path=/opt/kubernetes/logs/k8s-audit.log"
EOF


#–logtostderr：启用日志
#—v：日志等级
#–log-dir：日志目录
#–etcd-servers：etcd集群地址
#–bind-address：监听地址
#–secure-port：https安全端口
#–advertise-address：集群通告地址
#–allow-privileged：启用授权
#–service-cluster-ip-range：Service虚拟IP地址段
#–enable-admission-plugins：准入控制模块
#–authorization-mode：认证授权，启用RBAC授权和节点自管理
#–enable-bootstrap-token-auth：启用TLS bootstrap机制
#–token-auth-file：bootstrap token文件
#–service-node-port-range：Service nodeport类型默认分配端口范围
#–kubelet-client-xxx：apiserver访问kubelet客户端证书
#–tls-xxx-file：apiserver https证书
#–etcd-xxxfile：连接Etcd集群证书
#–audit-log-xxx：审计日志


#拷贝证书
cp ~/TLS/k8s/ca*pem ~/TLS/k8s/server*pem /opt/kubernetes/ssl/

#启动TLS Bootstrapping机制,用于给客户端颁发证书
#创建token文件
# 必须要用自己机器创建的Token
TLS_BOOTSTRAPPING_TOKEN=`head -c 16 /dev/urandom | od -An -t x | tr -d ' '`

cat > /opt/kubernetes/cfg/token.csv << EOF
${TLS_BOOTSTRAPPING_TOKEN},kubelet-bootstrap,10001,"system:kubelet-bootstrap"
EOF

#systemd管理apiserver
cat > /usr/lib/systemd/system/kube-apiserver.service << EOF
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-apiserver.conf
ExecStart=/opt/kubernetes/bin/kube-apiserver \$KUBE_APISERVER_OPTS
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl start kube-apiserver
systemctl enable kube-apiserver

#授权kubelet-bootstrap用户允许请求证书
kubectl create clusterrolebinding kubelet-bootstrap \
--clusterrole=system:node-bootstrapper \
--user=kubelet-bootstrap

```

3. 部署kube-controller-manager

```shell
#创建配置文件
cat > /opt/kubernetes/cfg/kube-controller-manager.conf << EOF
KUBE_CONTROLLER_MANAGER_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--leader-elect=true \\
--master=127.0.0.1:8080 \\
--bind-address=127.0.0.1 \\
--allocate-node-cidrs=true \\
--cluster-cidr=10.244.0.0/16 \\
--service-cluster-ip-range=10.0.0.0/24 \\
--cluster-signing-cert-file=/opt/kubernetes/ssl/ca.pem \\
--cluster-signing-key-file=/opt/kubernetes/ssl/ca-key.pem  \\
--root-ca-file=/opt/kubernetes/ssl/ca.pem \\
--service-account-private-key-file=/opt/kubernetes/ssl/ca-key.pem \\
--experimental-cluster-signing-duration=87600h0m0s"
EOF

#–master：通过本地非安全本地端口8080连接apiserver。
#–leader-elect：当该组件启动多个时，自动选举（HA）
#–cluster-signing-cert-file/–cluster-signing-key-file：自动为kubelet颁发证书的CA，与apiserver保持一致

#systemd管理controller-manager
cat > /usr/lib/systemd/system/kube-controller-manager.service << EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes
[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-controller-manager.conf
ExecStart=/opt/kubernetes/bin/kube-controller-manager \$KUBE_CONTROLLER_MANAGER_OPTS
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF

#设置开机启动
systemctl daemon-reload
systemctl start kube-controller-manager
systemctl enable kube-controller-manager
```

4. 部署kube-scheduler

```shell
#创建配置文件
cat > /opt/kubernetes/cfg/kube-scheduler.conf << EOF
KUBE_SCHEDULER_OPTS="--logtostderr=false \
--v=2 \
--log-dir=/opt/kubernetes/logs \
--leader-elect \
--master=127.0.0.1:8080 \
--bind-address=127.0.0.1"
EOF

#-–master：通过本地非安全本地端口8080连接apiserver。
#-–leader-elect：当该组件启动多个时，自动选举（HA）

#systemd管理scheduler
cat > /usr/lib/systemd/system/kube-scheduler.service << EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes
[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-scheduler.conf
ExecStart=/opt/kubernetes/bin/kube-scheduler \$KUBE_SCHEDULER_OPTS
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF

#设置开机启动
systemctl daemon-reload
systemctl start scheduler
systemctl enable scheduler

#生成kubectl连接集群得证书
cat > admin-csr.json <<EOF
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
      "L": "BeiJing",
      "ST": "BeiJing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin

#生成kubeconfig文件
mkdir /root/.kube

KUBE_CONFIG="/root/.kube/config"
KUBE_APISERVER="https://109.52.36.182:6443"

kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=${KUBE_CONFIG}
kubectl config set-credentials cluster-admin \
  --client-certificate=./admin.pem \
  --client-key=./admin-key.pem \
  --embed-certs=true \
  --kubeconfig=${KUBE_CONFIG}
kubectl config set-context default \
  --cluster=kubernetes \
  --user=cluster-admin \
  --kubeconfig=${KUBE_CONFIG}
kubectl config use-context default --kubeconfig=${KUBE_CONFIG}
```

5. 设置master角色

```shell
kubectl label nodes k8s02 node-role.kubernetes.io/master=k8s02
```

6. 为master节点打污点

```shell
kubectl taint nodes k8s02 node-role.kubernetes.io/master=k8s02:NoSchedule --overwrite
```

## 4.部署Worker node

1. 部署kubelet（master）

```shell
#master分发组件 kubelet kube-proxy
#master执行
for ip in k8s01 k8s03
do
  ssh root@${ip} "mkdir -p /opt/kubernetes/{bin,cfg,ssl,logs}"
  scp kubernetes/server/bin/kubelet  root@${ip}:/opt/kubernetes/bin/
  scp kubernetes/server/bin/kube-proxy  root@${ip}:/opt/kubernetes/bin/
done

for ip in k8s01 k8s03; do
    ssh root@${ip} "ls -l /opt/kubernetes/bin/";
done


#master执行
cat > /opt/kubernetes/cfg/kubelet.conf << EOF
KUBELET_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--hostname-override=k8s02 \\
--network-plugin=cni \\
--kubeconfig=/opt/kubernetes/cfg/kubelet.kubeconfig \\
--bootstrap-kubeconfig=/opt/kubernetes/cfg/bootstrap.kubeconfig \\
--config=/opt/kubernetes/cfg/kubelet-config.yml \\
--cert-dir=/opt/kubernetes/ssl \\
--pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/k8sos/pause:3.2"
EOF

#–hostname-override：显示名称，集群中唯一
#–network-plugin：启用CNI
#–kubeconfig：空路径，会自动生成，后面用于连接apiserver
#–bootstrap-kubeconfig：首次启动向apiserver申请证书
#–config：配置参数文件
#–cert-dir：kubelet证书生成目录
#–pod-infra-container-image：管理Pod网络容器的镜像


##配置参数文件
#master执行
cat > /opt/kubernetes/cfg/kubelet-config.yml << EOF
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
address: 0.0.0.0
port: 10250
readOnlyPort: 10255
cgroupDriver: cgroupfs
clusterDNS:
- 10.0.0.2
clusterDomain: cluster.local 
failSwapOn: false
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 2m0s
    enabled: true
  x509:
    clientCAFile: /opt/kubernetes/ssl/ca.pem 
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
maxOpenFiles: 1000000
maxPods: 110
EOF

#生成bootstrap.kubeconfig文件
#master执行
KUBE_APISERVER="https://109.52.36.182:6443" # apiserver IP:PORT
TOKEN="8f803b629676680dce7ac3f5da30be44" # 与token.csv里保持一致

# 生成 kubelet bootstrap kubeconfig 配置文件
#master执行
kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=bootstrap.kubeconfig
kubectl config set-credentials "kubelet-bootstrap" \
  --token=${TOKEN} \
  --kubeconfig=bootstrap.kubeconfig
kubectl config set-context default \
  --cluster=kubernetes \
  --user="kubelet-bootstrap" \
  --kubeconfig=bootstrap.kubeconfig
kubectl config use-context default --kubeconfig=bootstrap.kubeconfig


cp bootstrap.kubeconfig /opt/kubernetes/cfg

#systemd管理kubelet
cat > /usr/lib/systemd/system/kubelet.service << EOF
[Unit]
Description=Kubernetes Kubelet
After=docker.service
[Service]
EnvironmentFile=/opt/kubernetes/cfg/kubelet.conf
ExecStart=/opt/kubernetes/bin/kubelet \$KUBELET_OPTS
Restart=on-failure
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl start kubelet
systemctl enable kubelet

# 查看kubelet证书请求
kubectl get csr
NAME                                                   AGE    SIGNERNAME                                    REQUESTOR           CONDITION
node-csr-uCEGPOIiDdlLODKts8J658HrFq9CZ--K6M4G7bjhk8A   6m3s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending

# 批准申请
kubectl certificate approve node-csr-uCEGPOIiDdlLODKts8J658HrFq9CZ--K6M4G7bjhk8A

# 查看节点
kubectl get node
NAME         STATUS     ROLES    AGE   VERSION
k8s02        NotReady   <none>   7s    v1.18.3

```



2. 部署kube-proxy(master)

```shell
#创建配置文件
cat > /opt/kubernetes/cfg/kube-proxy.conf << EOF
KUBE_PROXY_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--config=/opt/kubernetes/cfg/kube-proxy-config.yml"
EOF

#配置参数文件
cat > /opt/kubernetes/cfg/kube-proxy-config.yml << EOF
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
metricsBindAddress: 0.0.0.0:10249
clientConnection:
  kubeconfig: /opt/kubernetes/cfg/kube-proxy.kubeconfig
hostnameOverride: k8s02
clusterCIDR: 10.0.0.0/24
EOF

#生成kube-proxy.kubeconfig文件
#生成kube-proxy证书
# 切换工作目录
cd TLS/k8s

# 创建证书请求文件
cat > kube-proxy-csr.json << EOF
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
      "L": "BeiJing",
      "ST": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF

# 生成证书
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy

ls kube-proxy*pem


#生成kubeconfig文件
KUBE_APISERVER="https://109.52.36.182:6443"

kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=kube-proxy.kubeconfig
kubectl config set-credentials kube-proxy \
  --client-certificate=./kube-proxy.pem \
  --client-key=./kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig
kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig


cp kube-proxy.kubeconfig /opt/kubernetes/cfg/


#systemd管理kube-proxy
cat > /usr/lib/systemd/system/kube-proxy.service << EOF
[Unit]
Description=Kubernetes Proxy
After=network.target
[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-proxy.conf
ExecStart=/opt/kubernetes/bin/kube-proxy \$KUBE_PROXY_OPTS
Restart=on-failure
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl start kube-proxy
systemctl enable kube-proxy

```

3. 部署cni插件

```shell
mkdir -p /opt/cni/bin
tar zxvf cni-plugins-linux-amd64-v0.8.6.tgz -C /opt/cni/bin


wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
sed -i -r "s#quay.io/coreos/flannel:.*-amd64#lizhenliang/flannel:v0.12.0-amd64#g" kube-flannel.yml


kubectl apply -f kube-flannel.yml

kubectl get pods -n kube-system
NAME                          READY   STATUS    RESTARTS   AGE
kube-flannel-ds-amd64-2pc95   1/1     Running   0          72s

kubectl get node
NAME         STATUS   ROLES    AGE   VERSION
k8s02        Ready    <none>   41m   v1.18.3

```

4. 授权apiserver访问kubelet

```shell
cat > apiserver-to-kubelet-rbac.yaml << EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
      - pods/log
    verbs:
      - "*"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF

kubectl apply -f apiserver-to-kubelet-rbac.yaml

```

5. 增加worker节点

```shell
#node1
scp -r /opt/kubernetes root@k8s01:/opt/

scp -r /usr/lib/systemd/system/{kubelet,kube-proxy}.service root@k8s01:/usr/lib/systemd/system

scp -r /opt/cni/ root@k8s01:/opt/

scp /opt/kubernetes/ssl/ca.pem root@k8s01:/opt/kubernetes/ssl

#删除
rm /opt/kubernetes/cfg/kubelet.kubeconfig 
rm -f /opt/kubernetes/ssl/kubelet*


vi /opt/kubernetes/cfg/kubelet.conf
--hostname-override=k8s01

vi /opt/kubernetes/cfg/kube-proxy-config.yml
hostnameOverride: k8s01

systemctl daemon-reload
systemctl start kubelet
systemctl enable kubelet
systemctl start kube-proxy
systemctl enable kube-proxy

#node2
scp -r /opt/kubernetes root@k8s03:/opt/

scp -r /usr/lib/systemd/system/{kubelet,kube-proxy}.service root@k8s03:/usr/lib/systemd/system

scp -r /opt/cni/ root@k8s03:/opt/

scp /opt/kubernetes/ssl/ca.pem root@k8s03:/opt/kubernetes/ssl

#删除
rm /opt/kubernetes/cfg/kubelet.kubeconfig 
rm -f /opt/kubernetes/ssl/kubelet*

vi /opt/kubernetes/cfg/kubelet.conf
--hostname-override=k8s03

vi /opt/kubernetes/cfg/kube-proxy-config.yml
hostnameOverride: k8s03

systemctl daemon-reload
systemctl start kubelet
systemctl enable kubelet
systemctl start kube-proxy
systemctl enable kube-proxy



kubectl get csr
NAME                                                   AGE   SIGNERNAME                                    REQUESTOR           CONDITION
node-csr-4zTjsaVSrhuyhIGqsefxzVoZDCNKei-aE2jyTP81Uro   89s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending

kubectl certificate approve node-csr-4zTjsaVSrhuyhIGqsefxzVoZDCNKei-aE2jyTP81Uro

```



## 5.部署dashboard

参考kubeadm dashboard部署

