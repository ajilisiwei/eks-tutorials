## 安装 Metrics Server

下载 Metrics Server 资源清单

```sh
$ wget https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.4.2/components.yaml
```

将资源清单里的`k8s.gcr.io/metrics-server/metrics-server:v0.4.2` 的镜像路径替换为阿里云的镜像路径 `registry.aliyuncs.com/google_containers/metrics-server:v0.4.2`。替换完成通过一下命令部署 Metrics Server。

```sh
$ kubectl apply -f components.yaml
```

验证 Metrics Server 是否已经部署成功

```sh
$ kubectl top nodes
NAME                                               CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
ip-172-31-16-158.cn-northwest-1.compute.internal   32m          1%     441Mi           13%
ip-172-31-40-33.cn-northwest-1.compute.internal    46m          2%     474Mi           14%
```

## 附录

- [安装 Kubernetes Metrics Server](https://docs.amazonaws.cn/eks/latest/userguide/metrics-server.html)