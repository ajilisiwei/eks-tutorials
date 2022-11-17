## Autoscaling

### 先决条件

- 现有 Amazon EKS 集群
- 集群的现有 IAM OIDC 提供商
- 带有 Auto Scaling 组标签的节点组
  - k8s.io/cluster-autoscaler/${your-cluster-name}:owned
  - k8s.io/cluster-autoscaler/enabled:true

### 创建 IAM policy 和角色

1. 创建 policy

```sh
$ aws iam create-policy \
    --policy-name AmazonEKSClusterAutoscalerPolicy \
    --policy-document file://cluster-autoscaler-policy.json
```
输出：
```sh
{
    "Policy": {
        "PolicyName": "AmazonEKSClusterAutoscalerPolicy",
        "PolicyId": "ANPA24MUOIAB4WKLUDEZQ",
        "Arn": "arn:aws-cn:iam::<YOUR AWS ACCOUNT>:policy/AmazonEKSClusterAutoscalerPolicy",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2022-11-08T08:43:40+00:00",
        "UpdateDate": "2022-11-08T08:43:40+00:00"
    }
}
```
2.创建角色

```sh
$ eksctl create iamserviceaccount \
  --cluster=my-cluster-1 \
  --namespace=kube-system \
  --name=cluster-autoscaler \
  --attach-policy-arn=arn:aws-cn:iam::<YOUR AWS ACCOUNT>:policy/AmazonEKSClusterAutoscalerPolicy \
  --override-existing-serviceaccounts \
  --approve
```

### 部署 Cluster Autoscaler

1. 下载资源清单

```sh
$ curl -o cluster-autoscaler-autodiscover.yaml https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml
```
将 `<YOUR CLUSTER NAME> ` 替换为你集群的名称。

由于集群无法拉取 `k8s.gcr.io/autoscaling/cluster-autoscaler:v1.22.2` 镜像，所以先本地拉取镜像，并推送到集群所在 AWS ECR 服务，最后得到镜像的地址为`<YOUR AWS ACCOUNT>.dkr.ecr.cn-northwest-1.amazonaws.com.cn/google_containers/autoscaling/cluster-autoscaler:v1.22.2`，编辑 yaml 文件，将镜像地址修改为这个地址。注意，本地机器的 CPU 架构要与集群里节点的架构兼容，不然会导致部署服务后因为 CPU 架构不同执行失败。（具体操作查看[这篇文档](https://docs.amazonaws.cn/eks/latest/userguide/copy-image-to-repository.html)）

2. 部署资源

```sh
$ kubectl apply -f cluster-autoscaler-autodiscover.yaml
```

3. 修改服务账号注释

```sh
$ kubectl annotate serviceaccount cluster-autoscaler \
  -n kube-system \
  --overwrite \
  eks.amazonaws.com/role-arn=arn:aws-cn:iam::<YOUR AWS ACCOUNT>:role/eksctl-my-cluster-1-addon-iamserviceaccount-Role1-10CJUFCHK4O7D
```
注意！账号 arn 为准备工作时创建的服务账号，可在 AWS 管理控制台 IAM 模块查询。

1. 向 Cluster Autoscaler pods 添加 cluster-autoscaler.kubernetes.io/safe-to-evict 注释

```sh
$ kubectl patch deployment cluster-autoscaler \
  -n kube-system \
  -p '{"spec":{"template":{"metadata":{"annotations":{"cluster-autoscaler.kubernetes.io/safe-to-evict": "false"}}}}}'
```

5. 编辑 Cluster Autoscaler 部署,增加以下启动命令参数。

```sh
$ kubectl -n kube-system edit deployment.apps/cluster-autoscaler
```

- --balance-similar-node-groups
- --skip-nodes-with-system-pods=false

```yaml
    spec:
      containers:
      - command
        - ./cluster-autoscaler
        - --v=4
        - --stderrthreshold=info
        - --cloud-provider=aws
        - --skip-nodes-with-local-storage=false
        - --expander=least-waste
        - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/my-cluster
        - --balance-similar-node-groups
        - --skip-nodes-with-system-pods=false
```

6. 查看是否部署成功

```sh
$ kubectl -n kube-system logs -f deployment.apps/cluster-autoscaler
```
可看到不断打印如下内容：

```sh
...
I1108 10:34:00.831148       1 static_autoscaler.go:231] Starting main loop
I1108 10:34:00.831911       1 filter_out_schedulable.go:65] Filtering out schedulables
I1108 10:34:00.831924       1 filter_out_schedulable.go:132] Filtered out 0 pods using hints
I1108 10:34:00.831931       1 filter_out_schedulable.go:170] 0 pods were kept as unschedulable based on caching
I1108 10:34:00.831937       1 filter_out_schedulable.go:171] 0 pods marked as unschedulable can be scheduled.
I1108 10:34:00.831945       1 filter_out_schedulable.go:82] No schedulable pods
I1108 10:34:00.831980       1 static_autoscaler.go:420] No unschedulable pods
I1108 10:34:00.831996       1 static_autoscaler.go:467] Calculating unneeded nodes
I1108 10:34:00.832010       1 pre_filtering_processor.go:66] Skipping ip-172-31-32-202.cn-northwest-1.compute.internal - node group min size reached
I1108 10:34:00.832018       1 pre_filtering_processor.go:66] Skipping ip-172-31-8-230.cn-northwest-1.compute.internal - node group min size reached
I1108 10:34:00.832028       1 pre_filtering_processor.go:66] Skipping ip-172-31-26-34.cn-northwest-1.compute.internal - node group min size reached
I1108 10:34:00.832035       1 pre_filtering_processor.go:66] Skipping ip-172-31-26-159.cn-northwest-1.compute.internal - node group min size reached
I1108 10:34:00.832074       1 static_autoscaler.go:521] Scale down status: unneededOnly=false lastScaleUpTime=2022-11-08 09:32:40.452894743 +0000 UTC m=-3577.291125158 lastScaleDownDeleteTime=2022-11-08 09:32:40.452894743 +0000 UTC m=-3577.291125158 lastScaleDownFailTime=2022-11-08 09:32:40.452894743 +0000 UTC m=-3577.291125158 scaleDownForbidden=false isDeleteInProgress=false scaleDownInCooldown=false
I1108 10:34:00.832155       1 static_autoscaler.go:534] Starting scale down
I1108 10:34:00.832196       1 scale_down.go:918] No candidates for scale down
```

### 验证

1. 查看节点信息

```sh
$ kubectl top nodes
NAME                                               CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
ip-172-31-26-159.cn-northwest-1.compute.internal   36m          1%     561Mi           16%
ip-172-31-26-34.cn-northwest-1.compute.internal    45m          2%     601Mi           18%
ip-172-31-32-202.cn-northwest-1.compute.internal   31m          1%     527Mi           15%
ip-172-31-8-230.cn-northwest-1.compute.internal    48m          2%     568Mi           17%
```

2. 创建测试 Deployment 

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ac-sample-app
  namespace: ac-sample-app
spec:
  replicas: 5
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: public.ecr.aws/nginx/nginx:1.21
          ports:
            - name: tcp
              containerPort: 80
          resources:
            limits:
              memory: 2Gi
              cpu: "1"
            requests:
              memory: 2Gi
              cpu: "1"
```

如上所示，将 pod 副本数配置为 5，并指定比较大的 CPU 和 内存值，按以上的配置，每个节点最多只能启动一个 pod 实例，所以我们期望 Cluster Autoscaler Conterller 会为我们自动拓展一个节点，并将 pod 调度到新节点上。

执行命令创建该 deployment

```sh
$ kubectl create namespace ac-sample-app

$ kubectl apply -f ac-deployment.yaml
```

```sh
$ kubectl get po -n ac-sample-app
NAME                             READY   STATUS              RESTARTS   AGE
ac-sample-app-588f5d8cd6-4chhp   1/1     Running             0          34s
ac-sample-app-588f5d8cd6-9bq4l   0/1     ContainerCreating   0          34s
ac-sample-app-588f5d8cd6-krp22   0/1     ContainerCreating   0          34s
ac-sample-app-588f5d8cd6-xfxb2   1/1     Running             0          34s
ac-sample-app-588f5d8cd6-xhvv9   0/1     Pending             0          34s
```

如上所示，`ac-sample-app-588f5d8cd6-xhvv9` pod 一直处于 Pending 状态，等待一段时间之后，查看节点信息如下：

```sh
$ kubectl top nodes
NAME                                               CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
ip-172-31-26-159.cn-northwest-1.compute.internal   301m         15%    571Mi           17%
ip-172-31-26-34.cn-northwest-1.compute.internal    47m          2%     611Mi           18%
ip-172-31-32-202.cn-northwest-1.compute.internal   44m          2%     530Mi           15%
ip-172-31-45-50.cn-northwest-1.compute.internal    660m         34%    420Mi           12%
ip-172-31-8-230.cn-northwest-1.compute.internal    45m          2%     575Mi           17%
```

相比于开始，集群又增加了一个节点。再等待一会儿，再次查看 pod 调度情况。

```sh
kubectl get po -n ac-sample-app
NAME                             READY   STATUS              RESTARTS   AGE
ac-sample-app-588f5d8cd6-4chhp   1/1     Running             0          5m54s
ac-sample-app-588f5d8cd6-9bq4l   0/1     ContainerCreating   0          5m54s
ac-sample-app-588f5d8cd6-krp22   1/1     Running             0          5m54s
ac-sample-app-588f5d8cd6-xfxb2   1/1     Running             0          5m54s
ac-sample-app-588f5d8cd6-xhvv9   1/1     Running             0          5m54s
```

之前 Pending 状态的 pod 已经调度成功了。

```sh
$ kubectl delete namespace ac-sample-app
```

删除命名空间之后，过段时间，集群的节点数量又会回到之前的数量。

## 附录
- [AWS Autoscaling 文档](https://docs.amazonaws.cn/eks/latest/userguide/autoscaling.html)
- [Kubernetes Cluster Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler)
- [立足 AWS 对 Kubernetes 进行成本优化](https://aws.amazon.com/cn/blogs/china/cost-optimization-for-kubernetes-on-aws/)