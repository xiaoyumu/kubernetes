Rancher  HA 安装手册 (v2.3.3)
===

这篇手册详细介绍如何在内网环境(Air gapped / Offline)中安装 HA (高可用) 模式的 Rancher Cluster. 在开始前请确保已经部署了内网镜像仓库 (推荐使用[harbor](https://github.com/goharbor/harbor)).

本手册使用的相关软件包版本如下:
* rke (1.0.0)
* kubernetes (1.16.3)
* kubectl (1.16.3)
* rancher (2.3.3)
* cert-manager (0.9.1)
* helm (3.0.2)

基本步骤包括:

* 准备节点(物理机/虚拟机)
* 配置节点, 安装软件
* 准备相关工具
* 准备相关镜像并push到内网镜像仓库中
* 创建 RKE k8s Cluster 配置, 部署 k8s Cluster
* 下载, 配置和部署 cert manager
* 下载, 配置和部署 rancher

## 准备节点(物理机/虚拟机)

本手册使用下面的节点来部署, 请根据实际情况自行调整.

```plaintext
192.168.1.71 k8s-master-01
192.168.1.72 k8s-master-02
192.168.1.73 k8s-master-03
```

为确保各节点可以互相访问, 将节点和对应的 IP 添加到 ```/etc/hosts ```中

## 配置节点, 安装软件

请参考 Rancher 官方文档中的最佳实践进行配置

* [基本配置](https://docs.rancher.cn/rancher2x/install-prepare/basic-environment-configuration.html#_4-2-%E9%94%81%E5%AE%9Adocker%E7%89%88%E6%9C%AC)
* [最佳实践](https://docs.rancher.cn/rancher2x/install-prepare/basic-environment-configuration.html#_4-5-%E6%9B%B4%E5%A4%9A%E9%85%8D%E7%BD%AE%E8%AF%B7%E8%AE%BF%E9%97%AE%E6%9C%80%E4%BD%B3%E5%AE%9E%E8%B7%B5)

包括配置 OS (内核, 网络,文件系统等), 以及安装 docker.

准备一个账号 (这里使用 ```user_name``` 作为示例,请根据实际情况创建)

* 确保这个账号有 sudo 权限
* 将这个账号加入 docker 组

然后在执行部署的节点上生成并复制 ssh 公钥到所有节点, 以确保执行部署的节点可以无需密码访问所有节点. 例如这里选则  ```k8s-master-01``` 作为部署节点.

```shell
# 生成 ssh key
ssh-keygen -t rsa

# 复制到所有节点
ssh-copy-id -i ~/.ssh/id_rsa.pub user_name@k8s-master-01
ssh-copy-id -i ~/.ssh/id_rsa.pub user_name@k8s-master-02
ssh-copy-id -i ~/.ssh/id_rsa.pub user_name@k8s-master-03

# 完成后可以通过ssh命令测试,正常情况下能直接登录不再提示输入密码
ssh user_name@k8s-master-01
ssh user_name@k8s-master-02
ssh user_name@k8s-master-03
```

## 准备相关工具

下载下列工具到执行部署的节点上 (```k8s-master-01```)
* rke (1.0.0)
* kubectl (1.16.3)
* helm (3.0.2)

```sh
# Download and install RKE tool https://github.com/rancher/rke/releases/tag/v1.0.0
curl -L -o  rke_linux-amd64 \
    https://github.com/rancher/rke/releases/download/v1.0.0/rke_linux-amd64    
chmod +x rke_linux-amd64
sudo mv rke_linux-amd64 /usr/local/bin/rke

# Download and install kubectl
curl -L -o  kubectl-v1.16.3 \
    https://storage.googleapis.com/kubernetes-release/release/v1.16.3/bin/linux/amd64/kubectl
chmod +x kubectl-v1.16.3
sudo mv kubectl-v1.16.3 /usr/local/bin/kubectl

# Download and install helm3
curl -L -o helm-v3.0.2-linux-amd64.tar.gz  https://get.helm.sh/helm-v3.0.2-linux-amd64.tar.gz
tar -zxvf helm-v3.0.2-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm
```

如需使用socks5代理,可以添加如下参数:

```sh
curl -L -o FILENAME URL --socks5 IP:PORT
```

## 准备相关镜像并push到内网镜像仓库中

参考官方文档中关于准备[离线镜像](https://docs.rancher.cn/rancher2x/installation/helm-ha-install/offline/prepare-private-registry.html#_1-%E5%87%86%E5%A4%87%E6%96%87%E4%BB%B6)的部分.

也可使用 ```transfer-images.sh``` 脚本将镜像搬到指定的私有仓库中 ([参考](https://github.com/xiaoyumu/scripting/tree/master/rancher)).

由于我们使用了 harbor 作为私有仓库,它暂时不支持自动创建项目 (镜像目录), 所以这里将镜像导入到 私有仓库的 library 项目 (目录) 中, 后面部分也都使用这一镜像前缀.

```sh
./transfer-images.sh rancher-images.txt hub.fshome.net/library
```

## 创建 RKE k8s Cluster 配置, 部署 k8s Cluster

RKE 工具使用一个 cluster yaml 文件了来指导部署 k8s cluster, 下面是本例中的 cluster 配置文件, 更多配置可以参考 Rancher [官网示例](https://rancher.com/docs/rke/latest/en/example-yamls/).

```yaml
cluster_name: rancher-cluster

private_registries:
  - url: hub.fshome.net/library
    is_default: true

# Use following command to list rke tool supported k8s versions and related docker images 
# rke config --system-images --all
kubernetes_version: v1.16.3-rancher1-1

nodes:
  - address: 192.168.1.71
    hostname_override: k8s-master-01
    user: user_name
    role: [controlplane,etcd,worker]

  - address: 192.168.1.72
    hostname_override: k8s-master-02
    user: user_name
    role: [controlplane,etcd,worker]

  - address: 192.168.1.73
    hostname_override: k8s-master-03
    user: user_name
    role: [controlplane,etcd,worker]

services:
  etcd:
    extra_args:
      auto-compaction-retention: 240
      quota-backend-bytes: '6442450944'
    backup_config:
      enabled: true     # enables recurring etcd snapshots
      interval_hours: 6 # time increment between snapshots
      retention: 60     # time in days before snapshot purge
  kubelet:
    # Fail if swap is on, disable it
    fail_swap_on: false
    # Set max pods to 150 instead of default 110
    extra_args:
      max-pods: 100

# Kubernetes Authorization mode
# Use `mode: rbac` to enable RBAC
# Use `mode: none` to disable authorization
authorization:
  mode: rbac
    
# There are several network plug-ins that work, but we default to canal
network:
  plugin: flannel
  
# Specify DNS provider (coredns or kube-dns)
dns:
  provider: kube-dns
    
# Currently only nginx ingress provider is supported.
# # To disable ingress controller, set `provider: none`
ingress:
  provider: nginx

```

将上面的文件保存到 ```rke-cluster-2.3.3-v1.16.3_071.yml```, 然后执行下面的命令开始部署 k8s.

```sh
rke up --config ./rke-cluster-2.3.3-v1.16.3_071.yml
```

这时控制台开始输出部署日志.

```log
INFO[0186] [addons] Successfully saved ConfigMap for addon rke-ingress-controller to Kubernetes
INFO[0186] [addons] Executing deploy job rke-ingress-controller
INFO[0191] [ingress] ingress controller nginx deployed successfully
INFO[0191] [addons] Setting up user addons
INFO[0191] [addons] no user addons defined
INFO[0191] Finished building Kubernetes cluster successfully
```

等待数分钟后如果看到下面的内容说明部署成功完成:

```log
Finished building Kubernetes cluster successfully
```

rke 会在执行目录中生成两个文件:
```sh
rke-cluster-2.3.3-v1.16.3_071.rkestate
kube_config_rke-cluster-2.3.3-v1.16.3_071.yml
```
其中 ```rkestate``` 文件用于通过 rke 工具管理 cluster, 在以后升级 cluster时回用到 ([参考](https://rancher.com/docs/rke/latest/en/installation/#save-your-files)).
而 ```kube_config_rke-cluster-2.3.3-v1.16.3_071.yml``` 文件则用于通过 kubectl 操作 cluster.
例如下面的命令可以用于在部署完成后检查 cluster 的运行状况. 另外建议使用命令 ```kubectl get all -A``` 检查  cluster 中所有资源均已部署完毕并运行正常, 没有 pending 或 error 的资源. Pod 数量均达到预期.

```bash
export KUBECONFIG=$(pwd)/kube_config_rke-cluster-2.3.3-v1.16.3_071.yml
kubectl get nodes

NAME            STATUS   ROLES                      AGE   VERSION
k8s-master-01   Ready    controlplane,etcd,worker   9h    v1.16.3
k8s-master-02   Ready    controlplane,etcd,worker   9h    v1.16.3
k8s-master-03   Ready    controlplane,etcd,worker   9h    v1.16.3

kubectl get all -A
```

确认 cluster 已正常运行后可以继续后面的步骤. 注意上面的 ```export KUBECONFIG=``` 命令只是临时使用，重新打开控制台会话将需要重新设置．

## 下载, 配置和部署 cert manager
通过 helm 可以在线安装 cert-manager, 但由于其镜像由于网络限制可能无法直接拉取，所以需要先下载 chart 包，然后基于 chart 包生成含有私仓镜像路径的 kubectl 部署文件．
执行下面的命令获取 cert manager chart 包，这里我们使用 v0.9.1 版本，经测试与 rancher 2.3.3 工作正常．更低或者更高的版本可能出现部署错误．

```shell
# 下载并部署 cert manager 的 CRD (Customerized Resource Definition) 这一步必须先做.
curl -L -o cert-manager-crd_0.9.yaml \
    https://raw.githubusercontent.com/jetstack/cert-manager/release-0.9/deploy/manifests/00-crds.yaml

# 部署 CRD
kubectl apply -f cert-manager-crd_0.9.yaml

# 在 k8s 中创建 cert-manager namespace
kubectl create namespace cert-manager
# 标记 namespace cert-manager 禁用验证
kubectl label namespace cert-manager certmanager.k8s.io/disable-validation=true

# 添加 repo 并获取 cert manager 的 chart 包, 也可以从其他可以联网的机器上下载后复制到当前节点.
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm fetch --version v0.9.1 jetstack/cert-manager

# 根据
helm template cert-manager ./cert-manager-v0.9.1.tgz --output-dir . \
 --namespace cert-manager \
 --set image.repository=$YOUR_IMAGE_REGISTRY/quay.io/jetstack/cert-manager-controller \
 --set cainjector.image.repository=$YOUR_IMAGE_REGISTRY/quay.io/jetstack/cert-manager-cainjector \
 --set webhook.image.repository=$YOUR_IMAGE_REGISTRY/quay.io/jetstack/cert-manager-webhook

```
