---
title: RDS Tunneling 하여 접근하기
date: 2024-07-18 09:30:00 +09:00
categories: [AWS, Tunnel]
tags: [putty, DBever]
---

ㅅㄱ
AWS 환경에서 bastion을 통해서 private subnet에 존재하는 RDS에 접근할 경우가 있다.
bastion에 직접 들어가서 cli로 db에 접근할 수도 있겠지만 사용이 불현하다.
이럴 경우, SSH Tunneling을 이용해서 로컬에서 bastion을 통해 RDS로 바로 붙을 수 있다.
말 그대로 SSH로 Bastion을 연결하는 매개체로 사용하며, 로컬서버에서 RDS의 서비스 포트를 직접 사용하는 것 처럼 연결 시키는 방법이다.ㄴㅁ

연결을 하기 위해서 필요한 정보는 아래와 같다.

1. SSH 접근 정보

   - bastion 서버의 IP
   - bastion 서버의 로그인 정보

2. tunneling 정보
   - 로컬서버에서 DB의 서비스포트에 매핑할 Port(현재 로컬서버에서 사용하지 않고 있어야됨.)
     ㄹ - DB의 로그인 정보
   - 서비스 포트 : 예) 3306
