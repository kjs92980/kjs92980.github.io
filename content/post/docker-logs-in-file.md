---
title: "Docker 로그 파일 local에 남기기"
description: 
slug: docker-logs-in-file
date: 2022-12-07T21:26:26+09:00
image: /img/docker.png
categories:
    - Docker
draft: true
---

Docker 컨테이너로 어플리케이션을 올리게 되면 (로그 설정이 기본 값인 경우) 점점 늘어나는 로그의 양으로 인해 컨테이너 용량이 커지게 됩니다.
또한 컨테이너를 삭제하게 되면 컨테이너 내 로그 파일도 함께 삭제됩니다.
이 글에서는 Docker 컨테이너 로그에 대해서, 이 로그 파일에 대한 정책은 어떻게 설정하는지, 그리고 로컬에 이 파일을 어떻게 저장할지 함께 살펴보겠습니다.

## Docker logs
```
docker logs <container-id>
```
컨테이너의 로그를 조회할 수 있습니다. 옵션을 사용하지 않으면 모든 로그를 조회합니다.

``` plain
Flag shorthand -h has been deprecated, please use --help

Usage:  docker logs [OPTIONS] CONTAINER

Fetch the logs of a container

Options:
      --details        Show extra details provided to logs
  -f, --follow         Follow log output
      --since string   Show logs since timestamp (e.g. 2013-01-02T13:23:37Z) or relative (e.g. 42m for 42 minutes)
  -n, --tail string    Number of lines to show from the end of the logs (default "all")
  -t, --timestamps     Show timestamps
      --until string   Show logs before a timestamp (e.g. 2013-01-02T13:23:37Z) or relative (e.g. 42m for 42 minutes)
```
