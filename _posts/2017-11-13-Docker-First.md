---
layout: post
title: "Docker --- 初體驗摔斷腿"
author: "Richard Lin"
categories: documents
tags: documents
image:
  feature: docker.png
---

## 1. Docker for Windows 雷你一頓
* * *
之前嘗試開啟公司電腦裡的docker for windows時，就有發現開不起來了，但是那時天真的我想說反正也沒有用到就沒去理他。
但沒想到還是有要用到他的一天。<br>
開啟時都會出現，使用者不再'docker users'的group裡面，一開始我以為是要註冊公司帳號，用`docker login`試了好幾回。<br>
Google後才發現，!@#%&原來是我要去computer management裡面去修改我的使用者群組，把Domain Users給加進docker users group裡，才能具有開啟權限。

## 2. Docker for Windows 雷我兩頓
* * *
開啟後卻又跑出docker無法執行的錯誤，經過一翻徹查後才知道是要再安裝VirtualBox。<br>

## 3. 終於開始玩Docker
* * *
本次需要做的事非常簡單，就算是像我第一次玩docker的人都能快速了解。<br>
這次需要把之前做的專案包進image裡並push到公司的倉庫裡。<br>
這次會用到的指令有：<br>
1. `docker build -t <name:tag> .`
2. `docker push name`

<br>
首先準備好DockerFile，在此不多說關於dockerfile僅大致帶過

```docker
FROM golang:1.9 AS builder
COPY <src>  <dest>
WORKDIR <dest>
RUN GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build

FROM alpine:3.6

RUN apk update && \
    apk upgrade && \
    apk add --no-cache curl && \
    apk add --no-cache tzdata && \
    rm -rf /var/cache/apk/* && \
    mkdir -p <target>

COPY --from=builder <src> <target>
WORKDIR <target>
CMD <app>

HEALTHCHECK --interval=5s --timeout=10s CMD curl -f http://localhost/health || exit 1
```

採用Multi-Stage寫法，關於此種寫法的應用案[這裡](https://blog.wu-boy.com/2017/04/build-minimal-docker-container-using-multi-stage-for-go-app/)，主要是可以在build docker的同時也去做go build
。以免有本機golang version不同導致build有差異的狀況。<br><br>

dockerfile的語法講解請按[這裡](https://philipzheng.gitbooks.io/docker_practice/content/dockerfile/instructions.html)
<br><br>
### build
寫完dockerfile之後在那個目錄裡使用`docker build -t <name:tag> .`就會build出repository=name, tag=tag的image檔。<br>
再使用`docker run <name:tag>`，就能將image檔裝入container中執行。

### push
若要push image檔到倉庫裡，我們必須先更改想要push的image檔的名字，因為push的目標ip與檔案名稱會結合成[ip/檔案名稱]的形式。<br>
例如我今天想要推送abc映像檔到127.0.0.1:5000，我必須使用`docker tag <src name:tag> <target name:tag>`來複製出一個新的名字的檔案。

```shell
> docker tag abc 127.0.0.1:5000/abc
```

這樣就會出現一個新的跟abc一樣但名字叫做127.0.0.1:5000/abc的image。<br>
可以透過docker images來查看有哪些image在本機裡面。<br>
在來使用push就可以成功推送了。<br>

```shell
> docker push 127.0.0.1:5000/abc
```