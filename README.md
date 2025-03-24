# 概述
参考：https://github.com/wyy-holding/dify-k8s

主要通过`Kompose`将`docker-compose.yaml`转换为Kubernetes资源文件。
作者将这些文件进行了优化，分类。

我在此基础上，进行了大量修改，做了进一步优化，分类。

删除了一些无用的代码，所有组件统一使用一个pvc进行存储，方便管理。

支持dify以下版本
```bash
1.1.2
1.1.3
``` 

# 目录结构
```bash
env --> 全局环境变量
pvc --> 所有组件，统一使用一个pvc来进行持久化存储
databases --> 数据库相关：postgresql，redis，weaviate
middleware --> 中间件相关：plugin-daemon，sandox，ssf-proxy，nginx
services --> 服务相关：api，web，worker
```

# 部署
## 1. 创建命名空间dify
```bash
kubectl create namespace dify
```

## 2. 创建全局环境变量
```bash
kubectl apply -f env/env.yaml
```

## 3. 创建pv和pvc
这里的pv是自建的NFS，请根据实际情况修改
```bash
kubectl apply -f pvc/pv.yaml
kubectl apply -f pvc/pvc.yaml
```
进入到NFS根目录，创建dify相关持久化文件，并设置权限
```bash
mkdir -p dify/volumes/db/data
mkdir -p dify/volumes/redis/data
mkdir -p dify/volumes/weaviate
mkdir -p dify/volumes/plugin_daemon
mkdir -p dify/volumes/app/storage
chmod 777 -R dify
```

## 4. 数据库相关
postgresql
```bash
kubectl apply -f databases/postgres/postgres-StatefulSet.yaml
kubectl apply -f databases/postgres/postgres-Service.yaml
```
redis
```bash
kubectl apply -f databases/redis/redis-StatefulSet.yaml
kubectl apply -f databases/redis/redis-Service.yaml
```
weaviate
```bash
kubectl apply -f databases/weaviate/weaviate-StatefulSet.yaml
kubectl apply -f databases/weaviate/weaviate-Service.yaml
```

## 5. 中间件相关

`注意：这里的Nginx，最后一个运行。因为它是用来转发各个组件api接口的。`

plugin-daemon
```bash
kubectl apply -f middleware/plugin-daemon/plugin-daemon-Deployment.yaml
kubectl apply -f middleware/plugin-daemon/plugin-daemon-Service.yaml
```
sandbox
```bash
kubectl apply -f middleware/sandbox/sandbox-cm0.yaml
kubectl apply -f middleware/sandbox/sandbox-cm1.yaml
kubectl apply -f middleware/sandbox/sandbox-Deployment.yaml
kubectl apply -f middleware/sandbox/sandbox-Service.yaml
```
ssrf-proxy
```bash
kubectl apply -f middleware/ssrf-proxy/ssrf-proxy-cm0.yaml
kubectl apply -f middleware/ssrf-proxy/ssrf-proxy-cm1.yaml
kubectl apply -f middleware/ssrf-proxy/ssrf-proxy-Deployment.yaml
kubectl apply -f middleware/ssrf-proxy/ssrf-proxy-Service.yaml
```

## 6. 服务相关
api
```bash
kubectl apply -f services/api/api-StatefulSet.yaml
kubectl apply -f services/api/api-Service.yaml
```
web
```bash
kubectl apply -f services/web/web-Deployment.yaml
kubectl apply -f services/web/web-Service.yaml
```
worker
```bash
kubectl apply -f services/worker/worker-Deployment.yaml
kubectl apply -f services/worker/worker-Service.yaml
```

等待上面所有组件运行正常之后，部署Nginx组件
nginx
```bash
kubectl apply -f middleware/nginx/nginx-cm0.yaml
kubectl apply -f middleware/nginx/nginx-cm1.yaml
kubectl apply -f middleware/nginx/nginx-cm2.yaml
kubectl apply -f middleware/nginx/nginx-cm3.yaml
kubectl apply -f middleware/nginx/nginx-cm4.yaml
kubectl apply -f middleware/nginx/nginx-cm5.yaml
kubectl apply -f middleware/nginx/nginx-Deployment.yaml
kubectl apply -f middleware/nginx/nginx-Service.yaml
```

# 访问dify
直接使用nginx的nodeport端口访问即可

如果需要域名访问，则添加一条ingress规则，指向到nginx的svc即可。

# 常见问题
## 1. 镜像拉取失败
由于国内无法访问docker hub，正常拉取会失败

首先准备一台docker服务器，修改文件`/etc/docker/daemon.json`
```bash
{
	"registry-mirrors": [
		"https://docker.1ms.run",
		"https://docker.xuanyuan.me"
	]
}
```
重启docker进程，就可以拉取docker hub镜像了

然后将本地的docker镜像推送到私有仓库，比如：harbor

最后将yaml文件中的镜像改为私有仓库地址即可，注意添加私有仓库密钥。

## 2. 某些组件响应慢
这些组件限制的cpu和内存，用的是最小资源。如果使用场景访问量很大，请根据实际情况调整。
