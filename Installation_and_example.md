# 安装Kubernetes

虽然Istio在架构上支持各种服务部署平台，包括kubernetes，cloud foundry，Mesos等，但对Google自家Kubernetes的支持肯定是首先考虑的。目前版本的0.2版本也只支持了Kubernetes的集成。

本文主要介绍Istio的安装，因此不对Kubernetes安装过程进行详细描述。建议通过[Rancher](http://rancher.com)安装kubernetes集群，Rancher可以大大简化kubernetes集群的安装部署过程。

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

Istio提供了安装脚本，该脚本会根据操作系统下载相应的Istio二进制包并解压到当前目录。

```
 curl -L https://git.io/getLatestIstio | sh -
```

根据脚本的提示将Istio命令行加入到系统PATH环境变量中

```
export PATH="$PATH:/home/ubuntu/istio-0.2.10/bin"
```

部署Bookinfo示例程序



