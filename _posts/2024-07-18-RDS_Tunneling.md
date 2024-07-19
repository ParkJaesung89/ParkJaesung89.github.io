---
title: RDS Tunneling 하여 접근하기
date: 2024-07-18 09:30:00 +09:00
categories: [AWS, Tunnel]
tags: [putty, DBever]
---

# 개요

AWS 환경에서 bastion을 통해서 private subnet에 존재하는 RDS에 접근할 경우가 있다.
bastion에 직접 들어가서 cli로 db에 접근할 수도 있겠지만 사용이 불현하다.
이럴 경우, SSH Tunneling을 이용해서 로컬에서 bastion을 통해 RDS로 바로 붙을 수 있다.
말 그대로 SSH로 Bastion을 연결하는 매개체로 사용하며, 로컬서버에서 RDS의 서비스 포트를 직접 사용하는 것 처럼 연결 시키는 방법이다.

## [연결을 위한 정보]

1. SSH 접근 정보

   - bastion 서버의 IP
   - bastion 서버의 로그인 정보

2. tunneling 정보
   - 로컬서버에서 DB의 서비스포트에 매핑할 Port(현재 로컬서버에서 사용하지 않고 있어야됨.)
   - DB의 로그인 정보
   - 서비스 포트 : 예) 3306

---

## [연결 방법]

1. putty로 Bastion 서버에 SSh 접속
   ![putty_ssh](../assets/img/posts_img/RDS_Tunneling/putty%20ssh.png)
2. 접속 후 왼쪽 상단에 이미지 클릭하여 Change Settins 선택
   ![putty_tunneling](../assets/img/posts_img/RDS_Tunneling/putty%20tunneling.png)
3. Tunneling 설정
   1. Source port : Local PC의 tunneling할 Port
   2. Destination : {DB의 Endpoint & IP}:{DB Service Port}
   3. Add : 버튼 클릭하여 설정을 추가
   4. Apply : tunneling 적용  
      ![putty_tunneling2](../assets/img/posts_img/RDS_Tunneling/putty%20tunneling2.png)
4. 로컬서버에서 서비스 포트와 연결이 되있는지 확인  
   ![check_tunneling](../assets/img/posts_img/RDS_Tunneling/tunneling%20연결%20확인.png)

5. DB 연결  
   \*_DB 접속 툴은 각자 편한걸로 진행_
   - Tunneling이 되있기 때문에 rds의 실제 경로 및 port를 적는게 아니라 현재 local PC에서의 매핑된 정보를 입력해야됨.
   1. Server Host : 127.0.0.1(localhost)
   2. Port : Local pc에서 사용하는 Port
      ![rds 접근1](../assets/img/posts_img/RDS_Tunneling/rds%20접근1.png)
      ![rds 접근1](../assets/img/posts_img/RDS_Tunneling/rds%20접근2.png)
