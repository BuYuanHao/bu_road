## idea中连接docker

 下载插件:


![](/img/idea-docker-plugin.png)



## 新建Dockerfile文件


![](/img/Dockerfile.png)

## idea配置

![](/img/idea-docker-config.png)

## mvn 打包

## 构建镜像和容器

![](/img/idea-build-docker.png)

注意如果build提示网络问题，可能需要配置docker镜像



```json
{
  "builder": {
    "gc": {
      "defaultKeepStorage": "20GB",
      "enabled": true
    }
  },
  "experimental": false,
  "registry-mirrors": [
    "https://ccr.ccs.tencentyun.com",
    "https://docker.rainbond.cc",
    "https://elastic.m.daocloud.io",
    "https://elastic.m.daocloud.io",
    "https://docker.m.daocloud.io",
    "https://gcr.m.daocloud.io",
    "https://ghcr.m.daocloud.io",
    "https://k8s-gcr.m.daocloud.io",
    "https://k8s.m.daocloud.io",
    "https://mcr.m.daocloud.io",
    "https://nvcr.m.daocloud.io",
    "https://quay.m.daocloud.io"
  ]
}
```

![](/img/docker-mirrors.png)

## 成功