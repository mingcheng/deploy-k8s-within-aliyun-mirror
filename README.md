# 使用阿里云镜像快速部署 Kubernetes 集群

<!-- TOC -->

- [更新记录](#更新记录)
- [Linux 发行版的镜像源](#linux-发行版的镜像源)
  - [基于 APT 的发行版](#基于-apt-的发行版)
  - [openSUSE](#opensuse)
- [系统配置](#系统配置)
- [容器运行时配置](#容器运行时配置)
  - [Docker 配置](#docker-配置)
  - [CRI-O 配置](#cri-o-配置)
- [初始化 Kubernetes](#初始化-kubernetes)
- [安装网络模块](#安装网络模块)
  - [Flannel](#flannel)
  - [Calico（废弃）](#calico废弃)
- [验证 Kubernetes](#验证-kubernetes)
- [安装 Dashboard](#安装-dashboard)
- [安装 MetalLB](#安装-metallb)
- [注意事项](#注意事项)
  - [找回 join 命令](#找回-join-命令)
  - [安全删除控制面节点](#安全删除控制面节点)
- [参考链接](#参考链接)

<!-- /TOC -->

由于国内「众所周知」的原因，在默认部署 Kubernetes（以下简称 K8S） 以及相关容器服务可能会碰到网络无法联通的问题，因此使用国内的镜像源是非常有必要的，可以减少很多不必要的麻烦。

阿里云其实提供了很多 Linux 发行版以及 Docker、K8S 相关的镜像源，用于加快部署以及更新镜像以及软件包。下面，简单说明下使用的步骤，以便可以一通百通。

## 更新记录

`20210424`
rebase 了部分的提交记录，并同时更新配置文件到 K8S v1.21

`20210106`
由于 K8S 逐渐的将 Docker 与运行时拆分，因此部署的时候考虑到这个因素，所以使用了尝试性质的 CRI-O 运行时，详细见对应的章节

`20200212`
初始化版本

## Linux 发行版的镜像源

### 基于 APT 的发行版

使用 Debian 以及其他的发行版，例如 CentOS 等都可以找到对应的软件镜像源。

例如，在 Debian 下可以直接使用 `source.list` 文件覆盖（记得备份）`/etc/apt/sources.list` 路径。

然后添加 `kubernetes.list` 文件到路径 `/etc/apt/sources.list.d/kubernetes.list` 。

然后更新源 `apt update -y && apt upgrade -y`，安装使用详细的可以参考阿里云的介绍，由于上面已经加入了阿里云的 K8S 源，因此直接安装即可：

```bash
apt-get update -y && apt-get install -y apt-transport-https gnupg
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
apt-get install -y kubelet kubeadm kubectl
```

注意，阿里云镜像源提供的 K8S 命令都比较新，因此如果需要指定版本（例如 1.18）则使用 apt 对应的命令。

### openSUSE

2020 年后，统一使用 openSUSE 作为物理机以及虚拟机的运行镜像系统，其自带了 K8S 的软件源（Leap 可能会较老旧），直接使用 zypper 安装即可：

```
zypper install kubernetes1.18-kubeadm kubernetes1.18-kubelet kubernetes1.18-controller-manager
```

## 系统配置

K8S 部署需要主机的包转发支持，所以记得开启相应的内核参数，修改 `/etc/sysctl.conf` 文件，添加下面主要的配置：

```
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
```

下面的配置不是必须的，但是建议也一并开启，至于各项的内容和具体的参数值，详细建议的配置请参考 `sysctl.conf` 文件

## 容器运行时配置

### Docker 配置

Debian 下 Docker 的安装和配置相对来说不会太复杂，软件包方面直接 `sudo apt install docker docker-compose` 即可。

相应的配置可以参考 `daemon.json` 这个文件，主要需要注意的地方有

```
"registry-mirrors": ["https://<your-token>.mirror.aliyuncs.com"]
```

这个字段。针对 Docker 的镜像源阿里云有对应的服务，可以自行申请。

最后使用 `systemctl enable docker` 开机自启以及使用 `docker info` 查看安装是否正确。

### CRI-O 配置

配置文件路径在 `/etc/containers/registries.conf` ，对应的内容可以参考 `registries.conf` 文件。详细参考：

- https://docs.openshift.com/container-platform/3.11/crio/crio_runtime.html
- https://github.com/containers/image/blob/master/docs/containers-registries.conf.5.md

## 初始化 Kubernetes

先 `kubeadm version` 即可知道目前安装的版本等相应的信息。注意查看 `kubeadm-init.yaml` 这个文件，注意下面几个参数：

```yaml
# ...
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
networking:
  # ...
  podSubnet: 10.100.0.1/24
```

分别对应的是镜像库的地址，这里指定阿里云的；以及 Pod 的网域地址，需要和下面 Calico 的地址对应。如果使用 flannel ，则对应的配置改成 `podSubnet: "10.244.0.0/16"` 。

然后，使用 `kubeadm init --config kubeadm-init.yaml` 开始初始化。具体预置的 config 可以使用 `kubeadm config print init-defaults` 查看。

如果需要集群模式，则加上 `--upload-certs` 这个参数，具体参见： https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/

```
The --upload-certs flag is used to upload the certificates that should be shared across all the control-plane instances to the cluster.
```

所以需要加上参数：

```
kubeadm init --upload-certs --config kubeadm-init.yaml
```

如果无误，则会提示 nodes 上 `kubeadm join` 需要的相关信息。如果遗忘了 `kubeadm join` 命令，可以使用：

```
kubeadm token create --print-join-command
```

重新获得，via https://github.com/kubernetes/kubeadm/issues/659#issuecomment-357726502

然后，就可以使用 `kubectl get nodes -A` 以及 `kubectl get pod -A -o wide` 等命令查看 K8S 控制面集群的运行状态了。

## 安装网络模块

### Flannel

当集群初始化完成以后则需要安装网络平面，一般来说使用 Flannel 足矣，直接使用

`kubectl apply -f kube-flannel.yml`

即可安装，注意 quay.io 有可能存在国内无法拉取的情况，需要额外的注意。

### Calico（废弃）

_由于使用了一段时间也没有使用 Calio 的功能属性，因此切换回 Flannel 网络模块 by mingcheng 20200106_

K8S 的网络模块有很多可以选择，普遍使用 Flannel 比较多，这里我个人使用 Calico 因为它有比较详细的权限控制以及客户端。

默认情况下，Calico 使用 `192.168.0.0/24` 网段，但是上述的 `init-default.yaml` 指定的 Pods 网段为 `10.100.0.1/24` 所以需要稍微更改下配置：

```
- name: CALICO_IPV4POOL_CIDR
  value: "10.100.0.1/24"
```

更详细的信息查看官方网站： https://www.projectcalico.org/

直接使用 `kubectl apply -f calico.yaml` 即可安装网络模块，然后等待一段时间后查看各个 Pods 的运行情况。

先查看 CoreDNS 的运行情况：

```
for p in $(kubectl get pods --namespace=kube-system -l K8S-app=kube-dns -o name); do kubectl logs --namespace=kube-system $p; done
```

如果没有报错，则移除 taint 以便在 kube-system 这个 namespace 上部署相关的工具 Pod 。

```
kubectl taint nodes --all node-role.kubernetes.io/master-
```

然后测试 DNS、网络时候正常，先部署 dnsutils 这个 Pod 到 kube-system 这个 namespce：

```
kubectl apply -f example/hello.yaml
```

部署完成，Pod 的状态 Ready 以后，分别执行

```
kubectl exec -it dnsutils -- cat /etc/resolv.conf
kubectl exec -it dnsutils -- nslookup kubernetes.default
```

说明网络已经部署完成，同时可以正常使用了。

## 验证 Kubernetes

然后在各个 Node 上使用 kubeadm join 加入集群和部署 kubelet 相关的进程。这里有个简单的使用 nginx 测试集群的情况。

```
kubectl apply -f nginx.yaml
```

然后使用 `port-forward` 或者使用 NodePort 的方式查看端口是否正常返回数据，以便判断运行是否正常。

## 安装 Dashboard

首先使用 admin-role.yaml 文件生成 admin 权限的 token，`kubectl apply -f admin-role.yaml`。然后，获取 admin token，参考命令：

```
TOKEN_NAME=$(kubectl -n kube-system get secret | grep admin-token | awk '{print $1}')
kubectl -n kube-system get secret $TOKEN_NAME -o jsonpath={.data.token} | base64 -d
```

安装 Dashboard，具体参见。项目中有 `dashboard.yaml` 可以供参考：

https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/

接下来，使用先前生成的 `admin-role.yaml` 生成的 `token` 即可登录。

## 安装 MetalLB

具体的文件和配置在 metallb 目录中，没有使用 Ingress 是因为需求的缘故，更需要 TCP 端口的汇聚和输出，而七层应用这块交给业务配置。

## 注意事项

<del>使用 apt 阿里云源安装的 K8S 比较新，目前为 1.18 版本，这个版本和 Istio 1.5.2 有冲突，需要等待版本更新才能正常安装。详见：https://github.com/istio/istio/issues/22215#issuecomment-599665040</del> 已解决

### 找回 join 命令

如果忘记了 join 命令，可以使用 `kubeadm token create --print-join-command` 命令加入节点。如果忘记控制面的命令，则比较麻烦一点，先重置 certificate-key：

```
$certificate-key = kubeadm init phase upload-certs --upload-certs

$(kubeadm token create --print-join-command) \
--control-plane \
--certificate-key $certificate-key
```

然后组合命令，再到节点上执行即可。

### 安全删除控制面节点

注意，如果只是 `kubectl delete node` 只会删除节点，但比不会让其他的 etcd 退出节点，因此需要在其他的 etcd 节点中手工执行删除命令，在对应的 Pod 中执行：

```
etcdctl --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/peer.crt --key=/etc/kubernetes/pki/etcd/peer.key --endpoints <https://your-etcd-endpoint:2379> member list

etcdctl --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/peer.crt --key=/etc/kubernetes/pki/etcd/peer.key --endpoints <https://your-etcd-endpoint:2379> member remove <464c2ab521decd41>
```

## 参考链接

- https://blog.scottlowe.org/2019/08/15/reconstructing-the-join-command-for-kubeadm/
- http://ljchen.net/2018/10/23/%E5%9F%BA%E4%BA%8E%E9%98%BF%E9%87%8C%E4%BA%91%E9%95%9C%E5%83%8F%E7%AB%99%E5%AE%89%E8%A3%85kubernetes/
- https://github.com/kubernetes/kubernetes/issues/56038
- https://cloud.tencent.com/developer/article/1482739
- https://juejin.im/post/5dde7e4be51d4505f45f2495
- https://juejin.im/post/5dde7e4be51d4505f45f2495
- https://www.lijiaocn.com/%E9%A1%B9%E7%9B%AE/2017/04/11/calico-usage.html
