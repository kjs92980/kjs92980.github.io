---
title: "Windows에서 Docker 사용 시 MySQL 한글 깨짐 해결: Custom Configuration 적용하기"
description: Windows에서 Docker를 사용해 MySQL 이미지를 실행할 때 한글 깨짐 문제를 해결하는 방법을 소개합니다. 특히 볼륨 마운트된 *.cnf 파일 권한 문제에 초점을 맞춥니다.
slug: custom-configuration-to-mysql-in-windows
date: 2023-11-18T21:36:30+09:00
image: /img/docker.png
categories:
    - Q&A
tags:
    - Windows
---

## Why
Windows에서 Docker를 이용해 MySQL 이미지를 구동하던 중 한글이 깨지는 문제가 발생했습니다.  
볼륨으로 마운트한 *.cnf 파일에 characterset을 설정했지만, Windows에서는 이 파일이 777 권한으로 인식되면서 설정이 제대로 반영되지 않는 문제가 발생했습니다.
```
World-writable config file '/etc/my.cnf' is ignored
```  
여기서는 파일명을 my.cnf로 정의하겠습니다.
```my.cnf
[client]
default-character-set = utf8mb4

[mysqld]
authentication-policy = mysql_native_password
```

이 문제는 Windows와 Linux의 파일 시스템 차이에서 기인합니다.  
Windows에서 Docker를 사용하여 Linux 컨테이너 내의 볼륨으로 파일을 마운트할 때, 해당 파일의 권한 설정이 항상 일관적이지 않을 수 있습니다.

## ToDo
이러한 문제를 해결하기 위해, 다음과 같은 접근 방법을 사용했습니다.

1. Dockerfile 작성
```Dockerfile
FROM mysql:8.0.30
COPY ./db/conf.d/my.cnf /etc/mysql/conf.d/my.cnf
RUN chmod 644 /etc/mysql/conf.d/my.cnf
```
* MySQL의 기본 설정을 변경하기 위해 커스텀 Dockerfile을 만들었습니다. 이 파일을 통해 my.cnf 파일의 권한을 설정합니다.

2. Docker Compose 파일 수정
```yaml
version: "3.8"
services:
  mysql:
    build: .
    ports:
      - "3306:3306"
```
* Custom image를 빌드하고 실행하기 위해 Docker Compose 파일을 수정했습니다.

3. 이미지 빌드 및 컨테이너 실행
```bash
docker-compose up -d --build
```
* 새로운 Dockerfile을 이용해 이미지를 빌드하고 컨테이너를 실행합니다.

