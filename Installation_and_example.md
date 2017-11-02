# 安装Kubernetes

本文主要介绍Istio的安装，Istio是微服务通讯和治理的基础设施层，需要和微服务部署和集群管理的基础设施层共同工作。

Istio架构比较灵活，在设计上支持各种服务部署平台，包括kubernetes，cloud foundry，Mesos等，但作为Google亲儿子，对自家兄弟Kubernetes的支持肯定是首先考虑的。目前版本的0.2版本的手册中也只有Kubernetes集成的安装说明，其它部署平台和Istio的集成将在后续版本中支持。

![](/images/PilotAdapters.svg)![](images/PilotAdapters.svg)

不对Kubernetes安装过程进行描述。kubernetes集群的部署较为复杂，建议通过[Rancher](http://rancher.com)安装，可以大大简化kubernetes集群的安装部署过程。

本文的测试环境为两台虚机组成的集群，操作系统是Ubuntu 16.04.3 LTS。

通过Rancher安装Kubernetes集群的简要步骤如下：

## 在server和工作节点上安装docker

因为k8s并不支持最新版本的docker，因此需根据该页面安装指定版本的docker  
[http://rancher.com/docs/rancher/v1.6/en/hosts/](http://rancher.com/docs/rancher/v1.6/en/hosts/) ,目前是1.12版本。

```
curl https://releases.rancher.com/install-docker/1.12.sh | sh
```

## 启动Rancher server

```
sudo docker run -d --restart=always -p 8080:8080 rancher/server
```

## 登录Rancher管理界面，创建k8s集群

Rancher 管理界面的缺省端口为8080，在浏览器中打开该界面，通过菜单Default-&gt;Manage Environment-&gt;Add Environment添加一个kubernetes集群。这里需要输入名称Kubernetes，描述，然后选择kubernetes template，点击create，创建Kubernetes环境。

点击菜单Default-&gt;Kubernetes，然后点击右上方的Add a host，添加一台host到kubernetes集群中。注意添加到集群中的host上必须先安装好符合要求的docker版本。

然后根据Rancher页面上的提示在host上执行脚本，以将host加入ranch cluster。

host加入cluster后Rancher会在host上pull kubernetes的images并启动kubernetes相关服务，因此需要等待一段时间。根据网络情况不同等待时间可能不同，我等待了30分钟左右。

## 安装并配置kubectl

```
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.7.4/bin/linux/amd64/kubectl

chmod +x ./kubectl

sudo mv ./kubectl /usr/local/bin/kubectl
```

登录Rancher管理界面, 将 default-&gt;onap kubernetes-&gt;CLI create config 的内容拷贝到~/.kube/config 中，以配置Kubectl和kubernetes server的连接信息。

# 安装Istio

Istio提供了安装脚本，该脚本会根据操作系统下载相应的Istio安装包并解压到当前目录。

```
 curl -L https://git.io/getLatestIstio | sh -
```

根据脚本的提示将Istio命令行加入到系统PATH环境变量中

```
export PATH="$PATH:/home/ubuntu/istio-0.2.10/bin"
```

在kubernetes集群中部署Istio控制面服务

```
kubectl apply -f /home/ubuntu/istio-0.2.10/install/kubernetes/istio.yaml
```

确认Istio控制面服务已成功部署。Kubernetes会创建一个istio-system namespace，将Istio相关服务部署在该namespace中。

确认Istio相关Service的部署状态

```
kubectl get svc -n istio-system
```

```
NAME            CLUSTER-IP      EXTERNAL-IP        PORT(S)                                                  AGE
istio-egress    10.43.192.74    <none>             80/TCP                                                   25s
istio-ingress   10.43.16.24     10.12.25.116,...   80:30984/TCP,443:30254/TCP                               25s
istio-mixer     10.43.215.250   <none>             9091/TCP,9093/TCP,9094/TCP,9102/TCP,9125/UDP,42422/TCP   26s
istio-pilot     10.43.211.140   <none>             8080/TCP,443/TCP                                         25s
```

确认Istio相关Pod的部署状态

```
kubectl get pods -n istio-system
```

```
NAME                             READY     STATUS    RESTARTS   AGE
istio-ca-367485603-qvbfl         1/1       Running   0          2m
istio-egress-3571786535-gwbgk    1/1       Running   0          2m
istio-ingress-2270755287-phwvq   1/1       Running   0          2m
istio-mixer-1505455116-9hmcw     2/2       Running   0          2m
istio-pilot-2278433625-68l34     1/1       Running   0          2m
```

# 部署Bookinfo示例程序

在下载的Istio安装包的samples目录中包含了示例应用程序。

通过下面的命令部署Bookinfo应用

```
kubectl apply -f <(istioctl kube-inject -f /home/ubuntu/istio-0.2.10/samples/bookinfo/kube/bookinfo.yaml)
```

我们知道kubectl apply 命令用于部署服务，该命令行在部署Bookinfo应用时采用了istio的kube-inject工具将sidecar注入了Bookinfo的kubernetes yaml部署文件中。

通过下面的命令将注入了sidecar的yaml文件内容保存到文件中，可以查看kube-inject命令对yaml文件的修改内容

```
istioctl kube-inject -f /home/ubuntu/istio-0.2.10/samples/bookinfo/kube/bookinfo.yaml >> bookinfo_with_sidecar.yaml
```

打开bookinfo\_with\_sidecar.yaml文件，可以看到该命令为每一个服务都注入一个istio-proxy代理

```
      image: docker.io/istio/proxy_debug:0.2.10
        imagePullPolicy: IfNotPresent
        name: istio-proxy
        resources: {}
        securityContext:
          privileged: true
          readOnlyRootFilesystem: false
          runAsUser: 1337
        volumeMounts:
        - mountPath: /etc/istio/proxy
          name: istio-envoy
        - mountPath: /etc/certs/
          name: istio-certs
          readOnly: true
```

确认Bookinfo服务已经启动

```
kubectl get services
```

```
NAME          CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
details       10.43.175.204   <none>        9080/TCP   6m
kubernetes    10.43.0.1       <none>        443/TCP    5d
productpage   10.43.19.154    <none>        9080/TCP   6m
ratings       10.43.50.160    <none>        9080/TCP   6m
reviews       10.43.219.248   <none>        9080/TCP   6m
```

在浏览器中打开应用程序页面，地址为istio-ingress的External IP

`http://10.12.25.116/productpage`

# 创建路由规则

多次刷新productpage页面，你会发现该页面中显示的Book Reviews有时候有带红星的评价信息，有时有带黑星的评价信息，有时只有文字评价信息。这是因为Bookinfo应用程序部署了3个版本的Reviews服务，每个版本的返回结果不同，在没有设置路由规则时，缺省的路由会将请求随机路由到每个版本的服务上，如下图所示：

![](/images/withistio.svg)

通过创建一条路由规则route-rule.yaml，将请求流量都引导到Reviews-v1服务上

```
apiVersion: config.istio.io/v1alpha2
kind: RouteRule
metadata:
  name: reviews-default
spec:
  destination:
    name: reviews
  precedence: 1
  route:
  - labels:
      version: v1
```

启用该路由规则

```
 istioctl create -f route_rule.yaml -n default
```

再次打开productpage页面, 无论刷新多少次，显示的页面将始终是v1版本的输出，即不带星的评价内容。

