---
title: "Docker 로그 파일 local에 남기기"
description: 
date: 2022-12-07T21:26:26+09:00
image: /img/docker.png
categories:
    - Docker
draft: true
---

Docker 컨테이너로 어플리케이션을 올리게 되면 (로그 설정이 기본 값인 경우) 점점 늘어나는 로그의 양으로 인해 컨테이너 용량이 커지게 됩니다.
또한 컨테이너를 삭제하게 되면 컨테이너 내 로그 파일도 함께 삭제됩니다.
이 글에서는 Docker 컨테이너 로그에 대해서, 이 로그 파일에 대한 정책은 어떻게 설정하는지, 그리고 로컬에 이 파일을 어떻게 저장할지 함께 살펴보겠습니다.

### Docker logs
```
$ docker logs <container-id>
```
