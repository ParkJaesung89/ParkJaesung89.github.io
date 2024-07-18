---
title: Github+Jenkins 활용 terraform CI/CD pipeline 구성하기
date: 2024-04-08 14:16:00 +09:00
categories: [CICD, jenkins]
tags: [jenkins, github] # TAG names should always be lowercase
---

# Jenkis활용한 Terraform pipeline 구성

## docker 설치

### docker의 apt repo 셋업

```bash
# 충돌 방지를 위한 기존 도커엔진 삭제
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done

# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

### docker 패키지 설치

```bash
# 최신버전 설치
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin


## 특정버전 설치
## List the available versions:
#apt-cache madison docker-ce | awk '{ print $3 }'
#
#5:26.1.0-1~ubuntu.24.04~noble
#5:26.0.2-1~ubuntu.24.04~noble
#...
#
#VERSION_STRING=5:26.1.0-1~ubuntu.24.04~noble
#sudo apt-get install docker-ce=$VERSION_STRING docker-ce-cli=$VERSION_STRING containerd.io docker-buildx-plugin docker-compose-plugin


# docker test
sudo docker run hello-world

# Install docker compose(yaml 파일로 관리위해)
sudo apt install docker-compose
```

## Jenkins 설치

docker-compose.yml 파일 생성

```bash
$ vi docker-compose.yml
$ mkdir /home/jenkins_home
# yml 파일에 jenkins 작성
version: "3"
services:
  jenkins:
    image: jenkins/jenkins:lts
    user: root
    volumes:
      - ./jenkins:/home/jenkins_home
    ports:
      - 8080:8080

$ sudo docker-compose up -d     ## yml 파일위치에서 데몬으로 실행명령
```

## jenkins 로그인

jenkins log에 생성 후 password 정보가 주어지며 해당 pawssword 값으로 로그인 가능.

```bash
$ sudo docker logs jenkins

*************************************************************
*************************************************************
*************************************************************

Jenkins initial setup is required. An admin user has been created and a password generated.

Please use the following password to proceed to installation:


XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX


This may also be found at: /var/jenkins_home/secrets/initialAdminPassword

*************************************************************
*************************************************************
*************************************************************
```

## Terraform pipeline 구성 작업

1. Jenkins 관리 > Plugins 에서 "terraform" 을 검색 후 install
   ![terraform plugins search]()
   ![terraform plugins install]()

2. github에서 인증을 위한 token 생성 후 Jenkins New credentails에서 github 접근 정보 및 token값 저장
   ![create new credentials]()

3. Pipeline 생성
   new item > pipeline 선택
   ![create new item]()

- General 설정에서 "Github project"를 체크하고 Project url을 작성
  ![Configure_1]()

- Pipeline 연동을 위한 설정 값 작성
  Github를 webhook 통하여 변경된 부분을 체크하여 Jenkinsfile에 미리 작성된 액션을 수행하도록 하기위하여 아래와 같이 작성한다.
  ![Configure_2]()

4. github 웹훅 설정

- github에서 jenkins와의 웹훅 연동을 한다. 이 설정이 없으면 jenkins에서 생성한 pipeline이 동작하지 않는다.
  ![github_jenkins webhook_1]()
  ![github_jenkins webhook_2]()
