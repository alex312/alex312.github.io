<h1 align="center"> k8s kubeadm方式部署笔记 </h1>

1. 部署系统版本

| 软件       | 系统版本                             |
| ---------- | ------------------------------------ |
| CentOS     | CentOS Linux release 7.4.1708 (Core) |
| Docker     | 20.10.7                              |
| Kubernetes | V1.19.1                              |
| Flannel    | v0.12.0-amd64                        |
| etcd       | 3.4.13-0                             |

2. 节点规划

| Hostname         | IP            | Kernel version              |
| ---------------- | ------------- | --------------------------- |
| k8s-node2-master | 109.52.36.182 | 3.10.0-693.el7.x86_64       |
| k8s-node1        | 109.52.36.181 | 3.10.0-1160.31.1.el7.x86_64 |
| k8s-node3        | 109.52.36.183 | 3.10.0-693.el7.x86_64       |

3. 关闭selinux (all：所有集群内设备执行，下同)

```shell
#永久关闭
sed -i.bak 's#enforcing#disabled#g' /etc/selinux/config
#临时关闭
setenforce 0
```

4. 关闭swap分区(all)

```shell
#永久关闭
sed  -i.bak -r 's/.*swap.*/#&/' /etc/fstab
#临时关闭
swapoff -a
```

5. 关闭防火墙(all)

```shell
#永久关闭
systemctl disable firewalld
#临时关闭
systemctl stop firewalld
```

6. 设置主机名称(all)

```shell
#设置名称
hostnamectl set-hostname k8s-master1
hostnamectl set-hostname k8s-node1
hostnamectl set-hostname k8s-node2
#验证
hostname
```

7. 配置host文件(master)

```shell
#设置
cat >> /etc/hosts << EOF
109.52.36.182 k8s-master1
109.52.36.181 k8s-node1
109.52.36.183 k8s-node2
EOF
```

8. 将桥接的IPV4流量传递到iptables的链(all)

有一些ipv4的流量不能走iptables链，linux内核的一个过滤器，每个流量都会经过他，然后再匹配是否可进入当前应用进程去处理，导致流量丢失。

```shell
#设置
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
#生效
sysctl --system
```

9. 时钟同步(all)

```shell
#设置
yum install ntp -y
ntpdate time2.aliyun.com
#写入定时任务
*/1 * * * * ntpdate time2.aliyun.com > /dev/null 2>&1
```

10. 安装docker(all)

```shell
# remove docker 
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
# Set up repository
sudo yum install -y yum-utils
# Use Aliyun Docker
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
# install docker from yum
yum install  -y docker-ce docker-ce-cli containerd.io
# restart docker
systemctl restart docker
# enable docker
systemctl enable --now docker.service
# cat version 
docker --version
```

11. 配置docker加速(all)

```shell
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://etdea28s.mirror.aliyuncs.com"]
}
EOF

# reload
sudo systemctl daemon-reload
sudo systemctl restart docker
```

12. 配置kubernetes源(all)

```shell
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

13. 安装kubectl kubelet kubeadm(all)

```shell
# install kubectl kubelet kubeadm
yum install -y kubelet-1.18.8 kubeadm-1.18.8 kubectl-1.18.8
# set boot on opening computer
systemctl enable kubelet && systemctl start kubelet
```

14. 集群初始化(master)

```shell
kubeadm init \
--apiserver-advertise-address=109.52.36.182   \
--image-repository=registry.cn-hangzhou.aliyuncs.com/k8sos \
--kubernetes-version=v1.18.8 \
--service-cidr=10.96.0.0/12 \
--pod-network-cidr=10.244.0.0/16
```

15. 配置kubernetes用户信息(master)

```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

16. 节点加入集群

复制集群初始化信息

```shell
kubeadm join 109.52.36.182:xxx --token xxx
```

17. 安装flannel插件

```shell
docker pull registry.cn-hangzhou.aliyuncs.com/k8sos/flannel:v0.12.0-amd64 ;\
 docker tag  registry.cn-hangzhou.aliyuncs.com/k8sos/flannel:v0.12.0-amd64 \
 quay.io/coreos/flannel:v0.12.0-amd64

kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

18. 查看部署结果

```shell
kubectl get nodes -o wide
```

19. 部署dashboard

* 下载yaml文件，如失败多重试几次

```shell
wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.4/aio/deploy/recommended.yaml
sed -i 's#kubernetesui/dashboard#registry.cn-hangzhou.aliyuncs.com/k8sos/dashboard#g' recommended.yaml
sed -i 's#kubernetesui/metrics-scraper#registry.cn-hangzhou.aliyuncs.com/k8sos/metrics-scraper#g' recommended.yaml
```

* 修改service部分

```shell
# 将
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  ports:
    - port: 443
      targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard

# 修改为
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30001
  selector:
    k8s-app: kubernetes-dashboard
  type: NodePort

```

* 部署dashboard

```shell
kubectl apply -f recommended.yaml
```

* 部署token

生成token.yaml文件，内容如下：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system

```

```shell
kubectl apply -f token.yaml
```

查看token

```shell
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```

访问https://109.52.36.182:30001

选择token认证方式，输入token访问



目前访问dashboard使用如下token

eyJhbGciOiJSUzI1NiIsImtpZCI6Ii1lN0xQemdobjk0TV94RDl2a1FPN054TmV4RG1mdkZKbEZxQnYyQXJxR3cifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLXBkemtrIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI5ZDI3N2FlOC03MTdiLTQ2MDYtOWJhMy05NmEyMjc1ZjliMzEiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.YKCWu5LqijyBV1FKBinb8a2hSAZ1UWxDxuPuQ3kKP_c7jhHddY-2ooaaM1277E9OCVApA5DmKwTwHdJ6JH8d8sG-bE6gg7_ucEedFbhp6vUEwTxylFAzwWb5qsIS2Z9cOxIwk9dgD9Z6EQnp1dXQfSwsVK5t2Z8pES9AvdHzgtOKszJOjvC_bnEBozb_V1rNUnZr7tNMjz5BXbbFPBOZ9-43RCsknYrKbbAZ4TQmOtLVJPSxX8hPrEWcNrtb6VwGS9xRdnb-Gm8RvA0H7ueUktqNRJbjQvCBSFGSsO8uwZfR2DuF3EO5kAje2rNLKIRozgygOvujwezkGDTadS2uTQ