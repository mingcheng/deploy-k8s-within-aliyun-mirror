# 使用阿里云镜像快速部署 Kubernetes 集群

<!-- TOC -->

- [Linux 发行版的镜像源](#linux-发行版的镜像源)
- [系统配置](#系统配置)
- [Docker 配置](#docker-配置)
- [初始化 Kubernetes](#初始化-kubernetes)
- [安装网络模块](#安装网络模块)
- [验证 Kubernetes](#验证-kubernetes)
- [注意事项](#注意事项)
- [参考链接](#参考链接)

<!-- /TOC -->

由于国内 「众所周知」的原因，在默认部署 Kubernetes（以下简称 K8S） 以及相关容器服务可能会碰到网络无法联通的问题，因此使用国内的镜像源是非常有必要的，可以减少很多不必要的麻烦。

阿里云其实提供了很多 Linux 发行版以及 Docker、K8S 相关的镜像源，用于加快部署以及更新镜像以及软件包。下面，简单说明下使用的步骤，以便可以一通百通。

## Linux 发行版的镜像源

使用 Debian 以及其他的发行版，例如 CentOS 等都可以找到对应的软件镜像源。

例如，在 Debian 下可以直接使用 `source.list` 文件覆盖（记得备份）`/etc/apt/sources.list` 路径。

然后添加 `kubernetes.list` 文件到路径 `/etc/apt/sources.list.d/kubernetes.list` 。

然后更新源 `apt update -y && apt upgrade -y` 即可。

## 系统配置

K8S 部署需要主机的包转发支持，所以记得开启相应的内核参数，修改 `/etc/sysctl.conf` 文件，添加下面主要的配置：

```
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
```

下面的配置不是必须的，但是建议也一并开启，至于各项的内容和具体的参数值，请自行了解：

```
net.ipv4.tcp_tw_recycle=0
net.ipv4.neigh.default.gc_thresh1=1024
net.ipv4.neigh.default.gc_thresh1=2048
net.ipv4.neigh.default.gc_thresh1=4096
vm.swappiness=0
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.inotify.max_user_instances=8192
fs.inotify.max_user_watches=1048576
fs.file-max=52706963
fs.nr_open=52706963
net.ipv6.conf.all.disable_ipv6=1
net.netfilter.nf_conntrack_max=2310720
```

## Docker 配置

Debian 下 Docker 的安装和配置相对来说不会太复杂，软件包方面直接 `sudo apt install docker docker-compose` 即可。

相应的配置可以参考 `daemon.json` 这个文件，主要需要注意的地方有

```
"registry-mirrors": ["https://<your-token>.mirror.aliyuncs.com"]
```

这个字段。针对 Docker 的镜像源阿里云有对应的服务，可以自行申请。

最后使用 `systemctl enable docker` 开机自启以及使用 `docker info` 查看安装是否正确。

## 初始化 Kubernetes

详细的可以参考阿里云的介绍，由于上面已经加入了阿里云的 K8S 源，因此直接安装即可：

```bash
apt-get update -y && apt-get install -y apt-transport-https gnupg
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
apt-get install -y kubelet kubeadm kubectl
```

然后简单的试试 `kubeadm version` 即可知道目前安装的版本等相应的信息。

注意查看 `init-default.yaml` 这个文件，注意下面几个参数：

```yaml
# ...
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
networking:
  # ...
  podSubnet: 10.100.0.1/24
```

分别对应的是镜像库的地址，这里指定阿里云的；以及 Pod 的网域地址，需要和下面 Calico 的地址对应。

然后，使用 `kubeadm init --config init-default.yaml` 开始初始化。具体预置的 config 可以使用 `kubeadm config print init-defaults` 查看。

如果无误，则会提示 nodes 上 `kubeadm join` 需要的相关信息。

然后，就可以使用 `kubectl get nodes -A` 以及 `kubectl get pod -A -o wide` 等命令查看 K8S 控制面集群的运行状态了。

## 安装网络模块

K8S 的网络模块有很多可以选择，普遍使用 Flannel 比较多，这里我个人使用 Calico 因为它有比较详细的权限控制以及客户端。

直接使用 `kubectl apply -f calico.yaml` 即可安装网络模块，然后等待一段时间后查看各个 Pods 的运行情况。

先查看 CoreDNS 的运行情况：

```
for p in $(kubectl get pods --namespace=kube-system -l k8s-app=kube-dns -o name); do kubectl logs --namespace=kube-system $p; done
```

如果没有报错，则移除 taint 以便在 kube-system 这个 namespace 上部署相关的工具 Pod 。

```
kubectl taint nodes --all node-role.kubernetes.io/master-
```

然后测试 DNS、网络时候正常，先部署 dnsutils 这个 Pod 到 kube-system 这个 namespce：

```
kubectl apply -f dnsutils.yaml
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

## 注意事项

已知问题：使用 apt 阿里云源安装的 K8S 比较新，目前为 1.18 版本，这个版本和 Istio 1.5.2 有冲突，需要等待版本更新才能正常安装。详见：https://github.com/istio/istio/issues/22215#issuecomment-599665040

## 参考链接

* http://ljchen.net/2018/10/23/%E5%9F%BA%E4%BA%8E%E9%98%BF%E9%87%8C%E4%BA%91%E9%95%9C%E5%83%8F%E7%AB%99%E5%AE%89%E8%A3%85kubernetes/
* https://github.com/kubernetes/kubernetes/issues/56038
* https://pkg.go.dev/k8s.io/kubernetes/cmd/kubeadm/app/apis/kubeadm/v1beta2?tab=doc
* https://cloud.tencent.com/developer/article/1482739
* https://juejin.im/post/5dde7e4be51d4505f45f2495
* https://juejin.im/post/5dde7e4be51d4505f45f2495
* https://www.lijiaocn.com/%E9%A1%B9%E7%9B%AE/2017/04/11/calico-usage.html
