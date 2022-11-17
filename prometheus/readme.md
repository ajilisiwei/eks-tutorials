## 安装 helm

注意！由于我在 mac 上安装 helm-3.9 版本时使用会报错，这里退回 3.8 版本。

从 github 直接下载二进制安装包安装 (https://github.com/helm/helm/releases?page=2)

## 安装 Prometheus

### 创建命名空间

```sh
$ kubectl create namespace prometheus
```

### 添加 helm 库地址

```sh
$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

### 安装

```sh
$ helm upgrade -i prometheus prometheus-community/prometheus \
    --namespace prometheus \
    --set alertmanager.persistentVolume.storageClass="gp2",server.persistentVolume.storageClass="gp2"
```

输出如下：

```sh
Release "prometheus" does not exist. Installing it now.
NAME: prometheus
LAST DEPLOYED: Sat Nov 12 17:43:29 2022
NAMESPACE: prometheus
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The Prometheus server can be accessed via port 80 on the following DNS name from within your cluster:
prometheus-server.prometheus.svc.cluster.local


Get the Prometheus server URL by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace prometheus -l "app=prometheus,component=server" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace prometheus port-forward $POD_NAME 9090


The Prometheus alertmanager can be accessed via port 80 on the following DNS name from within your cluster:
prometheus-alertmanager.prometheus.svc.cluster.local


Get the Alertmanager URL by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace prometheus -l "app=prometheus,component=alertmanager" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace prometheus port-forward $POD_NAME 9093
#################################################################################
######   WARNING: Pod Security Policy has been moved to a global property.  #####
######            use .Values.podSecurityPolicy.enabled with pod-based      #####
######            annotations                                               #####
######            (e.g. .Values.nodeExporter.podSecurityPolicy.annotations) #####
#################################################################################


The Prometheus PushGateway can be accessed via port 9091 on the following DNS name from within your cluster:
prometheus-pushgateway.prometheus.svc.cluster.local


Get the PushGateway URL by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace prometheus -l "app=prometheus,component=pushgateway" -o jsonpath="{.items[0].metadata.name}")
  kubectl --namespace prometheus port-forward $POD_NAME 9091

For more information on running Prometheus, visit:
https://prometheus.io/
```

### 查看看板

```sh
$ kubectl --namespace=prometheus port-forward deploy/prometheus-server 9090
```

如下所示：

![avatar](https://resources.laihua.com/2022-11-12/prometheus-1.png)


## 安装 grafana

### 安装 grafana

下载 grafana 资源清单文件(https://grafana.com/docs/grafana/v9.0/setup-grafana/installation/kubernetes/)，grafana service 以 loadbalancer 方式部署。将 yaml 文件里 svc 部分的的默认端口从 3000 改为 80。

```sh
$ kubectl create ns grafana
namespace/grafana created

$ kubectl apply -f grafana.yaml -n grafana
persistentvolumeclaim/grafana-pvc created
deployment.apps/grafana created
service/grafana created
```

查看 grafana 相关资源情况，如下：

```sh
$ kubectl get all -n grafana
NAME                           READY   STATUS    RESTARTS   AGE
pod/grafana-58d576f784-4f48h   0/1     Running   0          36s

NAME              TYPE           CLUSTER-IP      EXTERNAL-IP                                                                      PORT(S)        AGE
service/grafana   LoadBalancer   10.100.170.45   aa7099ea4f885424c8ace82d0131dfd5-483351023.cn-northwest-1.elb.amazonaws.com.cn   80:31944/TCP   36s

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/grafana   0/1     1            0           37s

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/grafana-58d576f784   1         1         0       37s
```

浏览器打开 elb 地址，初始账号和密码均为`admin`，登录并修改密码。看板初始页面如下所示：

![avatar](https://resources.laihua.com/2022-11-14/grafana-1.png)

### 配置 Data Sources

查看 Prometheus 的 svc 信息，记下 `prometheus-server ` 的 CLUSTER-IP ,一小步要填写。

```sh
$ kubectl get svc -n prometheus
NAME                            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
prometheus-alertmanager         ClusterIP   10.100.3.123     <none>        80/TCP     46h
prometheus-kube-state-metrics   ClusterIP   10.100.243.213   <none>        8080/TCP   46h
prometheus-node-exporter        ClusterIP   10.100.138.24    <none>        9100/TCP   46h
prometheus-pushgateway          ClusterIP   10.100.62.151    <none>        9091/TCP   46h
prometheus-server               ClusterIP   10.100.51.48     <none>        80/TCP     46h
```

打开 grafana 的 configuration > Data Sources > select Prometheus 填入 Prometheus 的相关信息，如下所示：

![avatar](https://resources.laihua.com/2022-11-14/grafana-2.png)

配置完毕，点击 `Save & Test`。接下来配置 Dashboard。

### 添加 Dashboard

点击左侧 `+`，Create > Dashboard > Add a new panel。添加相应的检测对象和检测指标，以下以检测 Prometheus 的 pod 内存使用情况为例。

![avatar](https://resources.laihua.com/2022-11-14/grafana-4.png)

配置完毕，点击 `Use query` 并点击右上角的 `Apply`，保存并填写 Dashboard 的名称。回到 grafana 的首页选择对应名称的 Dashboard 即可查看资源情况。

![avatar](https://resources.laihua.com/2022-11-14/grafana-6.png)

## 错误

1. `prometheus-kube-state-metrics` 服务镜像拉取失败。
 查看日志发现`prometheus-kube-state-metrics` 服务会使用到 `registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.6.0` 镜像，由于网络原因拉取失败，先本地翻墙拉取镜像，并推送到集群所在的 ecr 私有镜像服务。

 ```sh
$ kubectl get deploy -n prometheus
NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
prometheus-alertmanager         0/1     1            0           2m22s
prometheus-kube-state-metrics   0/1     1            0           2m22s
prometheus-pushgateway          1/1     1            1           2m21s
prometheus-server               0/1     1            0           2m21s
 ```

编辑 `prometheus-kube-state-metrics`，将镜像地址替换为私有仓库地址：

```sh
$ kubectl edit deploy prometheus-kube-state-metrics -n prometheus
```

2. `prometheus-alertmanager` 和 `prometheus-server` 服务一直处于 `Pending` 状态。

```sh
$ kubectl get po -n prometheus
NAME                                             READY   STATUS    RESTARTS   AGE
prometheus-alertmanager-9f7fddcbd-4r4j9          0/2     Pending   0          16m
prometheus-kube-state-metrics-659d9f64b4-x4r2c   1/1     Running   0          3m5s
prometheus-node-exporter-25tvc                   1/1     Running   0          16m
prometheus-node-exporter-4lxqw                   1/1     Running   0          16m
prometheus-node-exporter-4wm2b                   1/1     Running   0          16m
prometheus-pushgateway-678445f4dd-8285l          1/1     Running   0          16m
prometheus-server-788d8bbf46-kxt2l               0/2     Pending   0          16m
```

查看日志，如下：

```sh
...
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason            Age    From               Message
  ----     ------            ----   ----               -------
  Warning  FailedScheduling  5m31s  default-scheduler  running PreBind plugin "VolumeBinding": binding volumes: timed out waiting for the condition
```

怀疑是自己没安装 ebs 存储插件，回到 aws 控制台 eks 管理页面，发现果然没添加 ebs 存储插件，将插件添加上，如下所示：

![avatar](https://resources.laihua.com/2022-11-12/ebs-1.png)

删除对应的 pod ，但是新启动的 pod 依然报这个错误，继续进一步排查，既然是网络存储创建失败，那就先查看 Prometheus 创建的 pvc。

```sh
$ kubectl get pvc -n prometheus
NAME                      STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
prometheus-alertmanager   Pending                                      gp2            19m
prometheus-server         Pending                                      gp2            19m
```

果然一直处于 Pending 状态。进一步查看日志如下：

```sh
$ kubectl describe pvc prometheus-alertmanager -n prometheus
...
Warning  ProvisioningFailed    19m   ebs.csi.aws.com_ebs-csi-controller-b6c8b644d-k8q56_ad3b8113-a2e4-48c9-b0ab-e09873b17402  failed to provision volume with StorageClass "gp2": rpc error: code = Internal desc = Could not create volume "pvc-dcf35be1-a1e7-4363-abea-44a76aecf0ae": could not create volume in EC2: UnauthorizedOperation: You are not authorized to perform this operation. Encoded authorization failure message: buOuqVPzWkbluUkXwLe1_deVgFpDKexlGOF6-nhdrMQIf--G1ZNHPYnBolBw4Zh-lP96wSoX4cuNW5DG6OtZnZ7osK2nJlsda6FOR52hNCNjtS9SrkYBuiKe41u1RNW2E6_Vfzow6xDVZfXC30dy7fsd0SxlXVMZimyuzlz6Srn0ecSBem7XWeBPXVwZ2WIxWEO08l1pOimzfv-O9GP0khfPuq_QTTgLWvGEj3tvYmv5nXlzOh0a8jVroMIobJ_taXwGug1uQ1ENjDXgv5pMAlhe4YA8Fruel0bkyEayOvaA4u_ZoD5imnwBrWEF3n6PNk3vC6bZh4TSfKA48FEe7zD_blleYt2W84plA3yG2ZAUm9wF25skQrRgug_ho8bEn5y0coHLhA9m9WOexTEkWPLSKrzsZAF37PerLrtkTLR0VofJ1GpAC4vGaaAQsySCtFNbW7KMCR4psEIohqq4h9nibaEaXo5iXVXphF7rssl0AWTmZA_gY3NXWcG6tBnDPJo3DGlSRQy02Q5WE6eVx6fo0fCENg5zxXtl-AY1DfgMuAaAx7Lp0W29x0H6tCSLxbqDPJh-X26YNQZTXOWWggExQn3XAeZdzt_8ljup3oigS5QiyWiXLLM
           status code: 403, request id: 5317cff9-1924-44b3-8e9b-a79d039ab9ca
```

很明显，对应 EC2 节点没有权限创建 EBS。回到 AWS 控制台 IAM 管理界面，找到 eks 的 Cluster 角色和 Node 角色，分别给角色添加 EBS 相关的权限，即 `AmazonEBSCSIDriverPolicy`。等待几分钟，再次查看 PVC 的状态，发现已经是 `Bound` 状态了。

## 附录

- [Prometheus 的控制层面指标](https://docs.amazonaws.cn/eks/latest/userguide/prometheus.html)
- [helm](https://helm.sh/docs/intro/install/)
- [Kubernetes with Prometheus and Grafana on Amazon EKS — better together](https://www.nclouds.com/blog/amazon-eks-grafana-prometheus/)
- [Grafana documentation](https://grafana.com/docs/grafana/v9.0/)