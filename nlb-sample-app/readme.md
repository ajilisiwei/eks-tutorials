## 先决条件

### 准备好 eks 集群

- 名称：my-cluster-1
- 对应 k8s 版本：1.23
- 已为 eks 集群添加了一个节点组和两个节点，并已正常运行

集群节点信息如下：

```sh
$ kubectl get nodes
NAME                                              STATUS   ROLES    AGE     VERSION
ip-172-31-26-34.cn-northwest-1.compute.internal   Ready    <none>   4h11m   v1.23.9-eks-ba74326
ip-172-31-8-230.cn-northwest-1.compute.internal   Ready    <none>   4h11m   v1.23.9-eks-ba74326
```

### 为集群创建 IAM OIDC 提供商

```sh
$ oidc_id=$(aws eks describe-cluster --name my-cluster-1 --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)

$ aws iam list-open-id-connect-providers | grep $oidc_id

$ eksctl utils associate-iam-oidc-provider --cluster my-cluster-1 --approve
```

### 为集群的子网需要打上以下标签：

- kubernetes.io/role/elb: 1
- kubernetes.io/role/internal-elb: 1
- kubernetes.io/cluster/${your-cluster-name}: owned

如下所示：

![avatar](https://resources.laihua.com/2022-11-8/subnet1.png)，如未配置子网标签，则后续创建的 service 将报以下错误:
```sh
  Normal   EnsuringLoadBalancer  5m24s  service-controller  Ensuring load balancer
  Warning  FailedBuildModel      5m24s  service             Failed build model due to unable to discover at least one subnet
```

## 安装 Amazon Load Balancer Controller

### 创建 IAM policy

从 https://github.com/kubernetes-sigs/aws-load-balancer-controller/blob/v2.4.4/docs/install/iam_policy_cn.json 下载角色策略模板，并执行以下策略创建命令。

```sh
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy_cn.json
```
output:
```sh
{
    "Policy": {
        "PolicyName": "AWSLoadBalancerControllerIAMPolicy",
        "PolicyId": "ANPA24MUOIAB3AGYSXZ27",
        "Arn": "arn:aws-cn:iam::<YOUR AWS ACCOUNT>:policy/AWSLoadBalancerControllerIAMPolicy",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2022-11-08T03:49:01+00:00",
        "UpdateDate": "2022-11-08T03:49:01+00:00"
    }
}
```

### 创建 IAM role

创建一个 IAM 角色。在 Amazon Load Balancer Controller 的 kube-system 命名空间中创建名为 aws-load-balancer-controller 的 Kubernetes 服务账户，并使用 IAM 角色的名称注释 Kubernetes 服务账户。

```sh
$ eksctl create iamserviceaccount \
  --cluster=my-cluster-1 \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name "AmazonEKSLoadBalancerControllerRole" \
  --attach-policy-arn=arn:aws-cn:iam::<YOUR AWS ACCOUNT>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

### 安装 cert-manager

```sh
$ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.5.4/cert-manager.yaml
```

### 安装 aws-load-balancer-controller

下载 aws-load-balancer-controller 服务所需的资源清单。

```sh
$ wget https://github.com/kubernetes-sigs/aws-load-balancer-controller/releases/download/v2.4.4/v2_4_4_full.yaml
```

将 yaml 里的集群名称改为自己的集群名称
```sh
apiVersion: apps/v1
kind: Deployment
. . .
name: aws-load-balancer-controller
namespace: kube-system
spec:
    . . .
    template:
        spec:
            containers:
                - args:
                    - --cluster-name=<INSERT_CLUSTER_NAME>
```

执行以下命令创建资源。

```sh
$ kubectl apply -f v2_4_4_full.yaml
```

### 安装 IngressClass 和 IngressClassParams

```sh
$ wget https://github.com/kubernetes-sigs/aws-load-balancer-controller/releases/download/v2.4.4/v2_4_4_ingclass.yaml

$ kubectl apply -f v2_4_4_ingclass.yaml
```

## 创建网络负载应用

### 创建 namespace

```sh
$ kubectl create namespace nlb-sample-app
```

### 创建 deployment

```sh
$ kubectl apply -f sample-deployment.yaml
```

### 创建 serivice

```sh
$ kubectl apply -f sample-service.yaml
```

### 查看服务信息

```sh
$ kubectl get po -n nlb-sample-app -owide
NAME                              READY   STATUS    RESTARTS   AGE   IP              NODE                                              NOMINATED NODE   READINESS GATES
nlb-sample-app-5c87bb6c86-h4g9d   1/1     Running   0          26m   172.31.28.119   ip-172-31-26-34.cn-northwest-1.compute.internal   <none>           <none>
nlb-sample-app-5c87bb6c86-j7pjx   1/1     Running   0          26m   172.31.11.250   ip-172-31-8-230.cn-northwest-1.compute.internal   <none>           <none>
nlb-sample-app-5c87bb6c86-q6n2d   1/1     Running   0          26m   172.31.16.153   ip-172-31-26-34.cn-northwest-1.compute.internal   <none>           <none>
```

```sh
$ kubectl get svc nlb-sample-service -n nlb-sample-app
NAME                 TYPE           CLUSTER-IP     EXTERNAL-IP                                                                             PORT(S)        AGE
nlb-sample-service   LoadBalancer   10.100.96.84   k8s-nlbsampl-nlbsampl-2e871da6aa-2e891476b66c1462.elb.cn-northwest-1.amazonaws.com.cn   80:31713/TCP   5m39s
```

server 的类型为 `LoadBalancer`, 通过对应的 `EXTERNAL-IP` 验证服务信息如下。

```sh
$ curl http://k8s-nlbsampl-nlbsampl-2e871da6aa-2e891476b66c1462.elb.cn-northwest-1.amazonaws.com.cn
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

说明 service 已经成功通过 aws elb 创建了网络负载。从 aws 控制台上也能看见相应的负载均衡器和目标组均已创建，从目标组的状态和 ip 信息上看出，已经成功负载到我们创建的三个 pod 上。

![avatar](https://resources.laihua.com/2022-11-8/elb1.png)

![avatar](https://resources.laihua.com/2022-11-8/elb2.png)

### 清除实验数据

```sh
$ kubectl delete namespace nlb-sample-app
```

## 附录

- [创建托管节点组](https://docs.amazonaws.cn/eks/latest/userguide/create-managed-node-group.html)
- [Amazon EKS 上的网络负载均衡](https://docs.amazonaws.cn/eks/latest/userguide/network-load-balancing.html)
- [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/deploy/installation/)
- [如何在 Amazon EKS 中的 Amazon EC2 节点组上使用 AWS 负载均衡器控制器设置](https://aws.amazon.com/cn/premiumsupport/knowledge-center/eks-alb-ingress-controller-setup/)