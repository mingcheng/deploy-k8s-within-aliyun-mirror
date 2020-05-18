# MetalLB 安装

详细参见：https://metallb.universe.tf/installation/ 具体步骤

1. 修改 address-pool.yaml 文件，配置网内对应的 IP 段，用于负载均衡器的 IP 池；
2. memberlist 的密钥

```
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```

3. apply namespace.yaml 以及 metallb.yaml 文件，完毕。

验证可以使用 `example/hello.yaml` 文件，然后查看 80 端口是否畅通。
