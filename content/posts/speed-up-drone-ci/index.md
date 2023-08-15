---
title: "加速 Drone CI"
date: 2023-08-13T16:55:43+08:00
tags: [drone, ci, docker]
# draft: true
---

使用 [Drone](https://github.com/harness/drone) 可以简化项目发布流程,
经过一段时间的使用和摸索, 这里分享几个加速 `Drone` 发布流程的方法.

<!--more-->

### 镜像拉取设置

[pull](https://docs.drone.io/pipeline/kubernetes/syntax/images/#pulling-images)
设置为 `if-not-exists`, 可以防止每次拉取最新的 `images`

```yml
kind: pipeline
type: docker
name: default

steps:
  - name: build
    image: golang:1.20
    pull: if-not-exists # Configure the runner to only pull the image if not found in the local cache
    environment:
      CGO_ENABLED: 0
      GOOS: linux
      GOPROXY: https://proxy.golang.com.cn,direct
    commands:
      - go build -o chatbot
```

### 编译阶段缓存依赖

[Host Volumnes](https://docs.drone.io/pipeline/kubernetes/syntax/volumes/host/)(该设置只能用于受信任的项目),
提速效果明显

```yml
kind: pipeline
type: docker
name: default

steps:
  - name: build
    image: golang:1.20
    pull: if-not-exists
    environment:
      CGO_ENABLED: 0
      GOOS: linux
      GOPROXY: https://proxy.golang.com.cn,direct
    volumes: # 挂载依赖目录
      - name: deps
        path: /go
    commands:
      - go build -o chatbot

volumes:
  - name: deps
    host:
      path: /var/lib/cache/go
```

### 打包阶段使用 host docker.sock

这种方式可以复用镜像打包过程中的基础镜像

```yml
kind: pipeline
type: docker
name: default

steps:
  ...
  - name: docker
    image: docker:cli
    pull: if-not-exists
    volumes: # 使用 host docker.sock
      - name: dockersock
        path: /var/run/docker.sock
    environment:
      USERNAME:
        from_secret: docker_username
      PASSWORD:
        from_secret: docker_password
      IMAGE: chatbot/chatbot-api
      REGISTRY: registry.xxx.com
    commands:
      - docker info
      - IMAGE=$REGISTRY/$IMAGE
      - docker build -t $IMAGE .
      - docker login -u $USERNAME -p $PASSWORD $REGISTRY
      - docker push $IMAGE


volumes:
  ...
  - name: dockersock
    host:
      path: /var/run/docker.sock
```

### 打包阶段 network_mode 设置为 host

由于私有库 `registry.xxx.com` 也部署在 `host` 上, 在 `host` 的 `/etc/hosts`
文件中加入:

```
127.0.0.1 registry.xxx.com
```

这样拉取和推送私有库镜像, 可以直接使用本地网络.

```yml
kind: pipeline
type: docker
name: default

steps:
  ...
  - name: docker
    image: docker:cli
    pull: if-not-exists
    volumes:
      - name: dockersock
        path: /var/run/docker.sock
    network_mode: host
    environment:
      USERNAME:
        from_secret: docker_username
      PASSWORD:
        from_secret: docker_password
      IMAGE: chatbot/chatbot-api
      REGISTRY: registry.xxx.com
  ...
```

经过这些加速措施, 项目的发布时间从 `3分钟` 缩短到 `30秒`, 喝口水的功夫,
就收到发布成功的消息了.
