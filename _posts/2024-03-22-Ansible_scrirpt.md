---
title: Ansible을 활용한 취약점 점검 실행 및 결과파일 취합 script(수정중)
date: 2024-03-22 01:45:00 +09:00
categories: [Ansible, ISMS]
tags: [Ansible, ISMS] # TAG names should always be lowercase
---

회사 ISMS 담당자-리눅스 파트로 일을 수행하기 시작하고 처음 취약점 점검을 위하여 취약점 점검 스크립트를 구동하여야 했으나, 현재까지 200~300대 정도의 서버들에 대해서 전부 일일이 들어가서 작업을 하고 있었습니다. 파일을 수집하는데 모든 서버에서 수동으로 돌려서 빼야되는 번거로움과 시간을 절약하기 위한 목적으로 ansible을 통한 자동화를 구성하기로 하였습니다.

지금부터 ansible을 활용한 리눅스의 취약점 점검 스크립트 및 결과파일 취합하는 방법에 대해서 알아보겠습니다.

## 사전에 필요 한 구성

1. Master서버 hostway 계정에서 Node 서버의 ansible과 ssh-key 교환 및 hosts 설정

   ```bash
   #Hosts 등록
   $cat /etc/hosts
   x.x.x.x node1
   x.x.x.x node2
   .
   .

   #키 교환 작업
   $ssh-keygen
   $ssh-copy-id ansible@{node명}       # 예) ssh-copy-id ansible@node1
   ```

2. 서버들 안에 필요한 폴더 생성

[master server]

1. 특정 경로로 폴더 생성
   ```bash
   $mkdir /home/hostway/isms/
   $mkdir /home/hostway/isms/check
   ```
2. /home/hostway/isms/ 경로에 스크립트 파일 넣기
   ```bash
   $ll -alh /home/hostway/isms/ | awk '{print $9}'
   Debian_v3.0.sh
   Linux_CentOS_v5.0_221129.sh
   ```

[node server]

1. "ansible" user 계정 생성
   ```bash
   $adduser ansible
   ```
2. /home/ansible 디렉토리 필요
   ```bash
   $ls /home | grep ansible
   ansible
   ```

## Master Server의 Inventory 작성

```bash
[nodes]
node1           ansible_host=node1 ansible_user=ansible
#노드 명         hostname             접속할 user명
```

## playbook 존재하는 파일은 총 3개, script 파일

경로 : /home/hostway/playbook/jsp/

```bash
1) ISMS_Script_Check.yaml  # Server의 OS 종류 체크 및 특정 OS의 실행스크립트로 전달
2) copy_fetch_CentOS.yaml  # CentOS server에 대한 취약점 점검 스크립트 수행 및 Master server로 결과파일 전달
3) copy_fetch_Debian.yaml  # Debian server에 대한 취약점 점검 스크립트 수행 및 Master server로 결과파일 전달
```

## 동작 구조

1. ISMS_Script_Check.yaml 파일 동작 -> OS의 종류 체크
   [ CentOS or Rocky냐 ] => copy_fetch_CentOS.yaml
   [ Ubuntu 냐 ] => copy_fetch_Debian.yaml

## 파일 내용

1. ISMS_Script_Check.yaml

```bash
---
- name: OS_Check_Include_Tasks                        # tasks 윗 부분들은 글로벌 설정
  #hosts: all                                         # 실제 전체 적용할 때 사용
  hosts: nodos                                        # 각 서버별로 확인해보기 위해서 적용
  gather_facts: yes                                   # hosts의 OS단의 정보 수집 할지 설정
  become: yes                                         # 슈퍼유저권한을 갖는다.

  tasks:
    #- name: OS_check                                      # Check 단계에서 실제 OS 정보에 대해서 출력하는 작업
      #debug:
        #var: ansible_facts

    - name: CentOS_tasks                              # CentOS & Rocky 리눅스 일시 copy_fetch_CentOS.yaml 파일 동작
      include_tasks: /home/hostway/playbook/jsp/copy_fetch_CentOS.yaml
      when: ansible_distribution ==  'CentOS' or ansible_distribution == 'Rocky'

    - name: Debian_tasks                              # Ubuntu 리눅스 일시 copy_fetch_Debian.yaml 파일 동작
      include_tasks: /home/hostway/playbook/jsp/copy_fetch_Debian.yaml
      when: ansible_distribution ==  'Ubuntu'

```

2. copy_fetch_CentOS.yaml

```bash

---
- name: Gather Facts                                  # OS 정보 읽어들이는 작업
    setup:

- name: Create Directory                              # 디렉토리 생성 작업(권한 설정도 들어감)
    file:
    path: "/home/ansible/isms"
    state: directory
    mode: 0755
    become: no                                          # root 권한 없이 hosts에 접속 한 user 로 동작

- name: Copy File                                # ansible 서버에서 원격지 서버로 파일 복사
    copy:
    src: "/home/hostway/isms/Linux_CentOS_v5.0.sh"
    dest: "/home/ansible/isms/Linux_CentOS_v5.0.sh"

- name: Change File-Auth                         # 복사시킨 파일 권한 설정
    file:
    path: "/home/ansible/isms/Linux_CentOS_v5.0.sh"
    mode: 0755
    owner: "ansible"
    group: "ansible"

- name: Start shell_Script                       # 스크립트 파일 구동
    shell: /home/ansible/isms/Linux_CentOS_v5.0.sh

- name: Get File                                 # 스크립트 결과 파일 원격지 서버에서 ansible 서버로 복사
    fetch:
    src: "/home/ansible/CentOS@@{{ ansible_facts['fqdn'] }}@@{{ ansible_facts['default_ipv4']['address'] }}.txt"
    # ansible_facfs[] 변수는 위에서 OS 조회한 Gather_facts 정보에서 필요한 데이터만 추출한 것
    dest: "/home/hostway/isms/check/"
    flat: yes                                         # 복사해서 들어오는 파일이 dest에 별도의 하위 디렉토리 없이 직접 저장됩니다.

```

3. copy_fetch_Debian.yaml

```bash

---
- name: Gather Facts
    setup:

- name: Create Directory
    file:
    path: "/home/ansible/isms"
    state: directory
    mode: 0755
    become: no

- name: Copy File
    copy:
    src: "/home/hostway/isms/Debian_v3.0.sh"
    dest: "/home/ansible/isms/Debian_v3.0.sh"

- name: Change File-Auth
    file:
    path: "/home/ansible/isms/Debian_v3.0.sh"
    mode: 0755
    owner: "ansible"
    group: "ansible"

- name: Start shell_Script
    shell: /home/ansible/isms/Debian_v3.0.sh

- name: Get File
    fetch:
    src: "/home/ansible/result.txt"
    dest: "/home/hostway/isms/check/Debian@@{{ ansible_facts['fqdn'] }}@@{{ ansible_facts['default_ipv4']['address'] }}.txt"
    flat: yes

```
