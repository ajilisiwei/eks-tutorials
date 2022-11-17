## eks 示例应用程序部署

### 创建 namespace

```sh
$ kubectl create namespace eks-sample-app
```

### 创建 deployment

```sh
$ kubectl apply -f eks-sample-deployment.yaml
```

### 创建 service

```sh
$ kubectl apply -f eks-sample-service.yaml
```

### 查看以上创建的资源

```sh
$ kubectl get all -n eks-sample-app
NAME                                               READY   STATUS    RESTARTS   AGE
pod/eks-sample-linux-deployment-764959fd66-b7528   1/1     Running   0          71s
pod/eks-sample-linux-deployment-764959fd66-bglxn   1/1     Running   0          71s
pod/eks-sample-linux-deployment-764959fd66-vdbts   1/1     Running   0          71s

NAME                               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/eks-sample-linux-service   ClusterIP   10.100.189.85   <none>        80/TCP    9s

NAME                                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/eks-sample-linux-deployment   3/3     3            3           72s

NAME                                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/eks-sample-linux-deployment-764959fd66   3         3         3       73s
```

### 测试

```sh
$ kubectl exec -it eks-sample-linux-deployment-764959fd66-b7528 -n eks-sample-app -- bash
```

```sh
root@eks-sample-linux-deployment-764959fd66-b7528:/# curl eks-sample-linux-service
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

```sh
root@eks-sample-linux-deployment-764959fd66-b7528:/# cat /etc/resolv.conf
nameserver 10.100.0.10
search eks-sample-app.svc.cluster.local svc.cluster.local cluster.local cn-northwest-1.compute.internal
options ndots:5
```

### 清理实验数据 

```sh
$ kubectl delete namespace eks-sample-app
```

## 附录

- [部署示例应用程序](https://docs.amazonaws.cn/eks/latest/userguide/sample-deployment.html)