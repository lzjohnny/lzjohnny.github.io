---
title: 重建 K8S 集群
date: 2019-09-30 08:12:16
tags:
---
## Master 节点操作
### 导出现有配置，以便重新 init 时使用

sudo kubectl -n kube-system get cm kubeadm-config -o yaml
注意输出的内容不是标准 yaml 格式，仅供参考，不能作为 kubeadm init 的配置文件内容。

以下是准备好的 kubeadm-config.yaml 文件内容：
```yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 10.10.10.140
  bindPort: 6443
---
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
networking:
  dnsDomain: cluster.local
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.1.0.0/16
kubernetesVersion: v1.15.0
apiServer:
  extraArgs:
    authorization-mode: Node,RBAC
  timeoutForControlPlane: 4m0s
controllerManager:
  extraArgs:
    "node-cidr-mask-size": "20"
certificatesDir: /etc/kubernetes/pki
imageRepository: registry.aliyuncs.com/google_containers
clusterName: kubernetes
scheduler: {}
```

每个配置中必须包含 apiVersion、kind
注意 podSubnet、serviceSubnet 分别确定了整个集群中（并非某个 Node 节点）的 Service、Pods 的 IP 地址范围。

可适当参考以下文档，以便查看示例、语法及合法配置内容：
https://godoc.org/k8s.io/kubernetes/cmd/kubeadm/app/apis/kubeadm/v1beta2
https://github.com/feiskyer/kubernetes-handbook/blob/master/deploy/kubeadm.md

### 清除原有配置

执行以下命令进行清理，不必担心误删除配置，此处删除的越干净之后遇到的问题越少：

```
kubeadm reset
systemctl stop kubelet
systemctl stop docker
rm -rf /var/lib/cni/
rm -rf /var/lib/kubelet/*
rm -rf /var/lib/etcd
rm -rf /run/flannel
rm -rf /etc/cni/
ifconfig cni0 down
ifconfig flannel.1 down
ifconfig docker0 down
ip link delete cni0
ip link delete flannel.1
systemctl start docker
```

如果 etcd 部署在 master 节点上，可使用 rm -rf /var/lib/etcd 删除集群配置。
如果 etcd 使用独立服务器，可以使用 etdc 命令行工具 etcdctl rm --recursive registry 命令删除相关配置。
在其他参考资料中可能需要使用 brctl 命令，该命令在 bridge-utils 包中，可通过 yum 安装。但是只要执行过上述清理命令后就已经清理干净，不再需要重复使用 brctl 相关命令清理。

### 初始化主节点

如果是初次部署需要提前拉取相应镜像，由于本次为重新部署不需要拉取镜像，直接执行初始化命令：
kubeadm init --config kubeadm-config.yaml

按照提示将配置文件放到家目录下，原来 K8S 的集群配置在重建后已经失效，如跳过这步则会无权操作集群：
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

记录下 join 命令参数，以便后续 node 节点加入集群使用：
kubeadm join 10.10.10.140:6443 --token <your-token> \
    --discovery-token-ca-cert-hash sha256:<your-ca-sha256>
    
提示：
如果忘记该命令参数，可以通过以下方法查看：
kubeadm token list 可查看 token
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
可查看证书文件的 SHA256 值

### 检查主节点状态、部署网络插件
kubectl get componentstatus
此时尚未创建 coredns，所以 kubectl get nodes 状态为 NotReady 是正常的。

创建网络插件：
curl -O https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
kubectl apply -f kube-flannel.yml

执行 kubectl get pods --all-namespaces，查看所有 Pods 处于 Running 状态，已经正常启动。
注意：如果 master 未完全清除原有配置，且修改了 PodCIDR，可能导致 IP 不一致导致 coredns 处于 ContainerCreating 状态，并报错：
> NetworkPlugin cni failed to set up pod "" network: failed to set bridge addr: "cni0" already has an IP address different from 10.244.0.1/24

执行 cat /run/flannel/subnet.env 查看 FLANNEL_SUBNET=**10.244.0.1/20**
执行 ifconfig cni0 查看 inet **10.244.0.1**  netmask **255.255.240.0**
执行 kubectl describe node al-bj2c-dev.k8smaster-1 | grep -i podcidr 查看 PodCIDR: **10.244.0.0/20**
这三处掩码长度必须保持一致，且前两处 IP 必须相同，且为第三处网段的第一个可用 IP。
所有节点都必须符合这个规律，如果哪个节点有误就重新 kubeadm reset 哪个节点，按前面步骤完整清除配置重建即可。

如果 flannel 和 coredns 启动没有问题，继续检查 Pods 地址范围是否修改成功。
本次重建集群的目的是缩短 PodCIDR 子网掩码长度，以便在同一个 node 上部署超过 256 个服务，执行：
kubectl describe node al-bj2c-dev.k8smaster-1 | grep -i podcidr
可见 PodCIDR: 10.244.0.0/20，由默认的 24 位掩码已缩短为 20 位。

## Node 节点操作
### Node 节点加入集群
使用 1.2 的清理命令同样清理好 Node 节点。使用 kubeadm join 重新加入集群即可。

---
其他参考文档：
- Flannel 原理
https://cloud.tencent.com/developer/article/1096997
- 环境安装：
https://www.cnblogs.com/hongdada/p/9761336.html
https://kubernetes.feisky.xyz/pai-cuo-zhi-nan/pod
- 清理环境：
https://stackoverflow.com/questions/41359224/kubernetes-failed-to-setup-network-for-pod-after-executed-kubeadm-reset
https://www.cnblogs.com/jinanxiaolaohu/p/9019212.html
