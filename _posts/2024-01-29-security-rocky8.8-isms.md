---
title: rocky8 취약점 조치 script
date: 2024-03-21 16:50:00 +09:00
categories: [ISMS, rocky8 취약점 조치 스크립트]
tags: [rocky8, ISMS]     # TAG names should always be lowercase
---

# Rocky8 의 취약점 조치 스크립트 작성
프로젝트 건으로 약 400대의 서버를 마이그레이션 해야되는 상황이 발생하여 모든 서버에 적용할 프로비저닝 작업이 필요했습니다.  
모든 서버에 적용할 기본적인 취약점 조치 및 파라미터 값 수정에 대하여 스크립트 작성하였습니다.  
*아래 스크립트는 상황에 따라 다를 수 있으니 참고만 하시면 됩니다.*  

```bash
#!/bin/bash
# Writted by HOSTWAY
# Version ISMS-P_Apply-2024_01_16
# update
yum -y update && yum -y upgrade
yum -y install vim-enhanced
yum -y install rsyslog
# disable services
systemctl disable cockpit
# timezone
# 인터뷰 후, timezone 적용
timedatectl set-timezone Asia/Seoul
# 인터뷰 후, disable selinux
sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
# define PS
echo "PS1='\[\e]0;\w\a\]\n\[\e[32m\]\u@\h:\[\e[33m\]\w\[\e[0m\]# '" >> /root/.bash_profile
# define vim editor
echo "alias vi='vim'" >> /etc/profile
# define banner
echo "
//////////////////////////////////////////////////////////////////////////////
Warning: This system is restricted to authorized users for business purposes.
Unauthorized access is a violation of the law.
This service may be monitored for administrative and security reasons.
By proceeding, you consent to this monitoring.
Thank you.
//////////////////////////////////////////////////////////////////////////////" > /etc/motd
# define default tune
echo "
#vm.swappiness=1
#net.ipv4.tcp_timestamps=0
#net.ipv4.tcp_sack=1
#net.core.netdev_max_backlog=250000
#net.core.rmem_max=4194304
#net.core.wmem_max=4194304
#net.core.rmem_default=4194304
#net.core.wmem_default=4194304
#net.core.optmem_max=4194304
#net.ipv4.tcp_rmem=4096 87380 4194304
#net.ipv4.tcp_wmem=4096 65536 4194304
#net.ipv4.tcp_low_latency=1
#net.ipv6.conf.all.disable_ipv6 = 1
#net.ipv6.conf.default.disable_ipv6 = 1
#net.ipv6.conf.lo.disable_ipv6 = 0
#net.ipv4.tcp_adv_win_scale=1
#net.ipv4.tcp_max_syn_backlog = 8192
fs.aio-max-nr = 1048576
fs.file-max = 6815744" >> /etc/sysctl.conf
# define default limits
echo "
*       soft    nofile  65535
*       hard    nofile  65535
*       soft    nproc   65535
*       hard    nproc   65535" >> /etc/security/limits.conf
# define HISTSIZE, FORMAT
echo "export HISTSIZE=9999" >> /etc/profile
echo "export HISTFILESIZE=9999" >> /etc/profile
echo "export HISTTIMEFORMAT=\"%Y-%m-%d [%H:%M:%S] : \"" >> /etc/profile
# BackUP dir
mkdir -p /root/HOSTWAY
# password auth
sed -i 's/ssh_pwauth:   false/ssh_pwauth:   true/g' /etc/cloud/cloud.cfg
# master username && password
export MASTER_USER=hae
export MASTER_USER_PASSWORD={PASSWD작성}
echo ""
echo ""
echo "==============================  START  =============================="
echo ""
###################  ISMS-P 2023.01.ver 진단항목 기준 ##################
# [U-1]root 계정 원격 접속 제한
echo ###########################################
echo [U-1]root 계정 원격 접속 제한
echo ###########################################
rsync -a /etc/ssh/sshd_config /root/HOSTWAY/
sed -i 's/PermitRootLogin yes/PermitRootLogin no/g' /etc/ssh/sshd_config
sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config

# [U-2]패스워드 복잡성 설정
echo ###########################################
echo [U-2]패스워드 복잡성 설정
echo ###########################################
rsync -a /etc/security/pwquality.conf /root/HOSTWAY/
cat << EOF > /etc/security/pwquality.conf
minlen = 8
dcredit = -1
ucredit = -1
lcredit = -1
ocredit = -1
EOF


# [U-3]계정 잠금 임계값 설정
echo ###########################################
echo [U-3]계정 잠금 임계값 설정
echo ###########################################
rsync -a /etc/pam.d/system-auth /root/HOSTWAY/
rsync -a /etc/pam.d/password-auth /root/HOSTWAY/
cat << EOF > /etc/pam.d/system-auth
#%PAM-1.0
# This file is auto-generated.
# User changes will be destroyed the next time authselect is run.
auth        required                                     pam_env.so
auth        required                                     pam_faildelay.so delay=2000000
auth        required                                     pam_faillock.so preauth silent
auth        [default=1 ignore=ignore success=ok]         pam_usertype.so isregular
auth        [default=1 ignore=ignore success=ok]         pam_localuser.so
auth        sufficient                                   pam_unix.so nullok try_first_pass
auth        [default=1 ignore=ignore success=ok]         pam_usertype.so isregular
auth        sufficient                                   pam_sss.so forward_pass
auth        required                                     pam_faillock.so authfail
auth        required                                     pam_deny.so
account     required                                     pam_faillock.so
account     required                                     pam_unix.so
account     sufficient                                   pam_localuser.so
account     sufficient                                   pam_usertype.so issystem
account     [default=bad success=ok user_unknown=ignore] pam_sss.so
account     required                                     pam_permit.so
password    requisite                                    pam_pwquality.so try_first_pass local_users_only
password    sufficient                                   pam_unix.so sha512 shadow nullok try_first_pass use_authtok
password    sufficient                                   pam_sss.so use_authtok
password    required                                     pam_deny.so
session     optional                                     pam_keyinit.so revoke
session     required                                     pam_limits.so
-session    optional                                     pam_systemd.so
#session     optional                                     pam_oddjob_mkhomedir.so
session     [success=1 default=ignore]                   pam_succeed_if.so service in crond quiet use_uid
session     required                                     pam_unix.so
session     optional                                     pam_sss.so
EOF
cat <<EOF > /etc/pam.d/password-auth
#%PAM-1.0
# This file is auto-generated.
# User changes will be destroyed the next time authselect is run.
auth        required      pam_env.so
auth        required      pam_faillock.so preauth silent audit deny=5 unlock_time=600
auth        required      pam_faildelay.so delay=2000000
auth        [default=1 ignore=ignore success=ok]        pam_usertype.so isregular
auth        [default=1 ignore=ignore success=ok]        pam_localuser.so
auth        sufficient    pam_unix.so try_first_pass nullok
auth        [default=die]       pam_faillock.so authfail audit deny=5 unlock_time=600
auth        [default=1 ignore=ignore success=ok]        pam_usertype.so isregular
auth        sufficient    pam_sss.so forward_pass
auth        required      pam_deny.so
account     required      pam_unix.so
account     sufficient    pam_localuser.so
account     sufficient    pam_usertype.so issystem
account     [default=bad success=ok user_unknown=ignore]        pam_sss.so
account     required      pam_permit.so
account     required      pam_faillock.so

password    requisite     pam_pwquality.so try_first_pass local_users_only retry=3 authtok_type=
password    sufficient    pam_unix.so try_first_pass use_authtok nullok sha512 shadow
password    sufficient    pam_sss.so use_authtok
password    required      pam_deny.so
session     optional      pam_keyinit.so revoke
session     required      pam_limits.so
-session     optional      pam_systemd.so
session     [success=1 default=ignore] pam_succeed_if.so service in crond quiet use_uid
session     required      pam_unix.so
seesion     optional      pam_sss.so
EOF


# [U-4]패스워드 파일 보호
echo ###########################################
echo [U-4]패스워드 파일 보호
echo ###########################################
if [ `head -1 /etc/passwd | awk -F: '{print $2}' | egrep "^x" | wc -c` -eq 2 ]
    then
        echo ""
    else
        echo "패스워드를 /etc/passwd 파일에 저장함"
        echo "Manual Check"
fi


# [U-5]root 이외의 UID가 '0' 금지
echo ###########################################
echo [U-5]root 이외의 UID가 '0' 금지
echo ###########################################
if [ `awk -F: '$3==0 {print $0}' /etc/passwd | grep -v 'root' | wc -l` -eq 0 ]
    then
        echo ""
    else
        echo "root 이외의 UID가 '0'인 계정이 존재함"
        echo "Manual Check"
fi


# [U-6]root 계정 su 제한
echo ###########################################
echo [U-6]root 계정 su 제한
echo ###########################################
rsync -a /etc/pam.d/su /root/HOSTWAY/
cat <<EOF > /etc/pam.d/su
#%PAM-1.0
auth            sufficient      pam_rootok.so
# Uncomment the following line to implicitly trust users in the "wheel" group.
#auth           sufficient      pam_wheel.so trust use_uid
# Uncomment the following line to require a user to be in the "wheel" group.
auth            required        pam_wheel.so use_uid
auth            substack        system-auth
auth            include         postlogin
account         sufficient      pam_succeed_if.so uid = 0 use_uid quiet
account         include         system-auth
password        include         system-auth
session         include         system-auth
session         include         postlogin
session         optional        pam_xauth.so
EOF
chgrp wheel /usr/bin/su
chmod 4750 /usr/bin/su


# [U-7]패스워드 최소 길이 설정
echo ###########################################
echo [U-7]패스워드 최소 길이 설정
echo ###########################################
rsync -a /etc/login.defs /root/HOSTWAY/
sed -i 's/PASS_MAX_DAYS\t99999/PASS_MAX_DAYS\t90/g' /etc/login.defs
sed -i 's/PASS_MIN_DAYS\t0/PASS_MIN_DAYS\t1/g' /etc/login.defs
sed -i 's/PASS_MIN_LEN\t5/PASS_MIN_LEN\t8/g' /etc/login.defs


# [U-8]패스워드 최대 사용 기간 설정
echo ###########################################
echo [U-8]패스워드 최대 사용 기간 설정
echo ###########################################
echo [U-7] 에 기 적용됨.


# [U-9]패스워드 최소 사용기간 설정
echo ###########################################
echo [U-9]패스워드 최소 사용기간 설정
echo ###########################################
echo [U-7]에 기 적용됨.


# [U-10]불필요한 계정 제거
echo ###########################################
echo [U-10]불필요한 계정 제거
echo ###########################################
rsync -a  /etc/passwd* /root/HOSTWAY/
rsync -a  /etc/group* /root/HOSTWAY/
rsync -a  /etc/shadow* /root/HOSTWAY/
rsync -a  /etc/gshadow* /root/HOSTWAY/
## 쉘이 부여된 계정 중 90일 이상 로그인하지 않은 계정 삭제
tshells="/bin/sh|/bin/bash|/bin/dash|/bin/tcsh|/bin/csh|/bin/mksh|/bin/ksh|/bin/zsh"
cat /etc/passwd | egrep "$tshells" | awk -F: '{print $1}' > U-10_1.txt
for i in `cat U-10_1.txt`; do lastlog -b 90 | grep "^$i "; done > U-10_2.txt
# Manual Check
# userdel -r `cat U-10_2.txt | awk '{print $1}'`


# [U-11]관리자 그룹에 최소한의 계정 포함
echo ###########################################
echo [U-11]관리자 그룹에 최소한의 계정 포함
echo ###########################################
# 계정이 존재하지 않는 500 이상 GID가 존재 유무
grep "^root" /etc/group | awk -F ":" '{print $4}' | sed s/,/\\n/g | grep -v "^root$" | wc -w > U-11_1.txt
cat /etc/passwd | grep -v "^root:" | awk -F: '$4==0 {print $0}' > U-11_2.txt
# Manual Check
# userdel -r `cat U-11_2.txt | awk '{print $1}'`


# [U-12]계정이 존재하지 않는 GID 금지
echo ###########################################
echo [U-12]계정이 존재하지 않는 GID 금지
echo ###########################################
awk -F : '$4 == null {print $0}' /etc/group | awk -F : '$3 >= 500 {print $0}' > U-12_group.txt
awk -F : '{print $4}' /etc/passwd > U-12_passwd.txt
for TGID in `cat U-12_passwd.txt`
    do
        grep -v ":$TGID:" U-12_group.txt > U-12_Results.txt
        cat U-12_Results.txt > U-12_group.txt
done
cat U-12_group.txt
echo "Manual Check"


# [U-13]동일한 UID 금지
echo ###########################################
echo [U-13]동일한 UID 금지
echo ###########################################
awk -F : '{print $3}' /etc/passwd > U-13_passwd.txt
if [ `cat U-13_passwd.txt | sort | uniq -d | wc -l` -eq 0 ]
    then
        echo ""
    else
        echo "중복된 UID가 존재함"
        echo "Manual Check"
fi


# [U-14]사용자 shell 점검
echo ###########################################
echo [U-14]사용자 shell 점검
echo ###########################################
if [ `cat /etc/passwd | egrep "^daemon|^bin|^sys|^adm|^listen|^nobody|^nobody4|^noaccess|^diag|^listen|^operator|^games|^gopher" | grep -v "admin" |  awk -F: '{print $7}'| egrep -v 'false|nologin|null|halt|sync|shutdown' | wc -l` -eq 0 ]
    then
        echo ""
    else
        echo "시스템 계정에 쉘이 부여됨"
        echo "Manual Check"
fi


# [U-15]세션 Timeout 설정
echo ###########################################
echo [U-15]세션 Timeout 설정
echo ###########################################
rsync -a /etc/profile /root/HOSTWAY/
echo "TMOUT=300" >> /etc/profile
echo [U-16]root 홈, 패스 디렉터리 권한 및 패스 설정
if [ `echo $PATH | grep "\.:" | wc -l` -eq 0 ]
    then
        echo ""
    else
        echo "PATH 환경변수에 '.'이 맨 앞 또는 중간에 위치함"
        echo "Manual Check"
fi


# [U-16]
echo ###########################################
echo [U-16]root 홈, 패스 디렉터리 권한 및 패스 설정
echo ###########################################
if [ `echo $PATH | grep "\.:" | wc -l` -eq 0 ]
    then
        echo ""
    else
        echo "PATH 환경변수에 '.'이 맨 앞 또는 중간에 위치함"
        echo "Manual Check"
fi

# [U-17]파일 및 디렉터리 소유자 설정
echo ###########################################
echo [U-17]파일 및 디렉터리 소유자 설정
echo ###########################################
ls -lL /home | awk '{print $3}' | grep "^[0-9]" > U-17_1.txt
for i in `cat U-17_1.txt`; do ls -lLd /home/* | grep -w $i >> U-17_2.txt; done
if [ -f U-17_2.txt ]
    then
        echo "/home 디렉토리에 소유자가 존재하지 않는 파일이 존재함"
        echo "Manual Check"
    else
        echo "/home 디렉토리에 소유자가 존재하지 않는 파일이 존재하지 않음"
fi


# [U-18]/etc/passwd 파일 소유자 및 권한 설정
echo ###########################################
echo [U-18]/etc/passwd 파일 소유자 및 권한 설정
echo ###########################################
chown root:root /etc/passwd && chmod 644 /etc/passwd


# [U-19]/etc/shadow 파일 소유자 및 권한 설정
echo ###########################################
echo [U-19]/etc/shadow 파일 소유자 및 권한 설정
echo ###########################################
chown root:root /etc/shadow && chmod 000 /etc/shadow


# [U-20]/etc/hosts 파일 소유자 및 권한 설정
echo ###########################################
echo [U-20]/etc/hosts 파일 소유자 및 권한 설정
echo ###########################################
chown root:root /etc/hosts && chmod 644 /etc/hosts


# [U-21]/etc/xinetd.conf 파일 소유자 및 권한 설정
echo ###########################################
echo [U-21]/etc/xinetd.conf 파일 소유자 및 권한 설정
echo ###########################################
# 설치되어 있지 않음
yum -y remove xinetd


# [U-22]/etc/syslog.conf 파일 소유자 및 권한 설정
echo ###########################################
echo [U-22]/etc/syslog.conf 파일 소유자 및 권한 설정
echo ###########################################
chown root:root /etc/rsyslog.conf && chmod 644 /etc/rsyslog.conf


# [U-23]/etc/services 파일 소유자 및 권한 설정
echo ###########################################
echo [U-23]/etc/services 파일 소유자 및 권한 설정
echo ###########################################
chown root:root /etc/services && chmod 644 /etc/services


# [U-24]SUID, SGID, Sticky bit 설정 파일 점검
echo ###########################################
echo [U-24]SUID, SGID, Sticky bit 설정 파일 점검
echo ###########################################
find / -user root -type f \( -perm -4000 -o -perm -2000 \) -exec ls -al {} \;
chmod -s /usr/bin/newgrp
chmod -s /sbin/unix_chkpwd
chmod -s /usr/bin/at


# [U-25]사용자, 시스템 시작파일 및 환경파일 소유자 및 권한 설정
echo ###########################################
echo [U-25]사용자, 시스템 시작파일 및 환경파일 소유자 및 권한 설정
echo ###########################################
chmod 644 /etc/profile


# [U-26]world writable 파일 점검
echo ###########################################
echo [U-26]world writable 파일 점검
echo ###########################################
find /etc -perm -2 -a -not -type l -ls > U-26.txt
if [ `cat U-26.txt | wc -l` -eq 0 ]
    then
        echo ""
    else
        echo "/etc 디렉토리 하위에 Others에 쓰기 권한이 부여된 파일이 존재함"
        echo "Manual Check"
fi


# [U-27]/dev에 존재하지 않는 device 파일 점검
echo ###########################################
echo [U-27]/dev에 존재하지 않는 device 파일 점검
echo ###########################################
## /dev아래의  MAKEDEV, .mount, .udev, /dev/shm 제외한 파일들을 찾은 후 갯수가 0개 이상인지 체크
## 리스트가 존재할 경우 수동으로 해당 파일 확인 후 필요여부에 따라 삭제.
find /dev -type f -exec ls -lL {} \; | grep -Ev "MAKEDEV|\.mount|\.udev|/dev/shm" > U-27.txt
if [ `cat U-27.txt | wc -l` -eq 0 ]
    then
        echo ""
    else
        echo "/dev 디렉토리에 major, minor nubmer를 가지지 않는 파일이 존재함"
        echo "Manual Check"
fi


# [U-28]$HOME/.rhosts, hosts.equiv 사용 금지
echo ###########################################
echo [U-28]$HOME/.rhosts, hosts.equiv 사용 금지
echo ###########################################
find /home -name "*.rhosts" | xargs rm -f
rm -f /etc/hosts.equiv


# [U-29]접속 IP 및 포트 제한
echo ###########################################
echo [U-29]접속 IP 및 포트 제한
echo ###########################################
if [ -f /etc/hosts.deny ]
    then
        if [ `cat /etc/hosts.deny | grep -v "#" | grep -iE "(sshd|all) *: *all$" | wc -l` -eq 0 ]
            then
                echo "/etc/hosts.deny 파일에 Deny 설정이 존재하지 않음"
                echo "Manual Check"
            else
                echo "/etc/hosts.deny 파일에 Deny 설정이 적용됨"
                cat /etc/hosts.allow | grep -v "#" | grep -v "^$"
        fi
    else
        echo "/etc/hosts.deny 파일이 존재하지 않음"
        echo "Manual Check"
fi


# [U-30]hosts.lpd 파일 소유자 및 권한 설정
echo ###########################################
echo [U-30]hosts.lpd 파일 소유자 및 권한 설정
echo ###########################################
# 설치되어 있지 않음
#rm -f /etc/hosts.lpd


# [U-31]NIS 서비스 비활성화
echo ###########################################
echo [U-31]NIS 서비스 비활성화
echo ###########################################
# 설치되어 있지 않음
# systemctl stop ypbind && systemctl disable ypbind
# SKIP


# [U-32]UMASK 설정 관리
echo ###########################################
echo [U-32]UMASK 설정 관리
echo ###########################################
echo "umask 022" >> /etc/profile


# [U-33]홈디렉토리 소유자 및 권한 설정
echo ###########################################
echo [U-33]홈디렉토리 소유자 및 권한 설정
echo ###########################################
user_list=$(cat /etc/passwd | grep -v nologin | grep -v root | grep -v :/sbin: | awk -F: '{print $1}')
for user in $user_list; do
    home_dir=$(getent passwd "$user" | cut -d: -f6)
    if [ ! -w "$home_dir" ]; then
        chmod o-w "$home_dir"
        echo "$home_dir 디렉토리로부터 쓰기권한 제거했습니다."
    else
        echo "$home_dir 디렉토리는 쓰기권한이 없습니다."
    fi
done


# [U-34]홈디렉토리로 지정한 디렉토리의 존재 관리
## User 홈디렉토리 확인 및 홈디렉토리가 이상 있을 경우 경로 지정하거나 삭제
# cat /etc/passwd
# uesrdel {USER}
echo ###########################################
echo [U-34]홈디렉토리로 지정한 디렉토리의 존재 관리
echo ###########################################
echo "Manual Check"


# [U-35]숨겨진 파일 및 디렉토리 검색 및 제거
#find / -type f -name ".*"
#find / -type d -name ".*"
echo ###########################################
echo [U-35]숨겨진 파일 및 디렉토리 검색 및 제거
echo ###########################################
echo "Manual Check"


# [U-36]Finger 서비스 비활성화
echo ###########################################
echo [U-36]Finger 서비스 비활성화
echo ###########################################
# 설치되어 있지 않음

# [U-37]Anonymous FTP 비활성화
echo ###########################################
echo [U-37]Anonymous FTP 비활성화
echo ###########################################
## FTP(anonymous) 계정 확인 후 존재 시 삭제
if id "ftp" &>/dev/null; then
    userdel -r ftp
    echo "FTP 계정 삭제 완료."
else
    echo "FTP 계정 존재하지 않음."
fi


# [U-38]r 계열 서비스 비활성화
echo ###########################################
echo [U-38]r 계열 서비스 비활성화
echo ###########################################
# 설치되어 있지 않음
## 'r'command 관련 데몬 파일 존재확인 및 disable 설정
FILES=("/etc/xinetd.d/rlogin" "/etc/xinetd.d/rsh" "/etc/xinetd.d/rexec")
for FILE in "${FILES[@]}"; do
    if [ -e "$FILE" ]; then
        sed -i 's/disable[ \t]*=[ \t]*no/disable = yes/g' "$FILE"
        echo "File $FILE: disable 옵션을 yes로 설정함."
    else
        echo "File $FILE: 파일이 존재하지 않음."
    fi
done


# [U-39]cron 파일 소유자 및 권한 설정
echo ###########################################
echo [U-39]cron 파일 소유자 및 권한 설정
echo ###########################################
chmod o-w /etc/crontab
chmod o-w /etc/cron.daily/*
chmod o-w /etc/cron.hourly/*
chmod o-w /etc/cron.weekly/*
chmod o-w /etc/cron.monthly/*
chmod o-w /var/spool/cron/*
chmod o-r /etc/cron.deny


# [U-40]DoS 공격에 취약한 서비스 비활성화
echo ###########################################
echo [U-40]DoS 공격에 취약한 서비스 비활성화
echo ###########################################
# 설치되어 있지 않음
## Dos 취약 공격에 취약한 서비스 활성화 확인 및 disable 설정(echo, discard, daytime, charget, NTP, DNS, SNMP)
## 위 서비스 중 사용하지 않는 서비스 제외 모두 비활성화
FILES=("/etc/xinetd.d/echo" "/etc/xinetd.d/discard" "/etc/xinetd.d/daytime" "/etc/xinetd.d/chargen")
for FILE in "${FILES[@]}"; do
    if [ -e "$FILE" ]; then
        sed -i 's/disable[ \t]*=[ \t]*no/disable = yes/g' "$FILE"
        echo "File $FILE: disable 옵션을 yes로 설정함."
    else
        echo "File $FILE: 파일이 존재하지 않음."
    fi
done


# [U-41]NFS 서비스 비활성화
echo ###########################################
echo [U-41]NFS 서비스 비활성화
echo ###########################################
## nfs 프로세스 확인 후 동작시 사용여부 확인하여 필요 없을 시 서비스 다운
if [ `ps -ef | grep -i "nfsd" | grep -v "grep" | wc -l` -eq 0 ]
    then
        echo "NFS 서비스가 실행중이지 않음"
    else
        systemctl stop nfs
        systemctl disable nfs
        echo "NFS 서비스 중지하였음"
fi


# [U-42]NFS 접근통제
echo ###########################################
echo [U-42]NFS 접근통제
echo ###########################################
## SKIP & manual check
## NFS 서비스 사용시 접근 통제 설정이 필요.
## 아래 스크립트로 NFS 서비스 및 everyone 설정 체크 후 everyone 공유시 특정 ip나 host등으로 접근 권한 설정필요
if [ `ps -ef | grep -i "nfsd" | grep -v "grep" | wc -l` -eq 0 ]
    then
        echo "NFS 서비스가 실행중이지 않음"
    else
        if [ -f /etc/exports ]
            then
                if [ `cat /etc/exports | grep -i "everyone" | grep -v "^ *#" | wc -l` -eq 0 ]
                    then
                        echo "NFS 서비스가 실행중이나 everyone 공유가 존재하지 않음"
                    else
                        echo "NFS 서비스가 실행중이고 everyone 공유가 존재함"
                fi
            else
                echo "NFS 서비스가 실행중이나 /etc/exports 파일을 찾을 수 없음"
        fi
fi
#/etc/exports 파일에 권한 설정 예
#sudo vi /etc/exports
#/test/nfs  10.10.10.0/24(rw,root_squash)   10.10.20.10(ro)


# [U-43]automountd 제거
echo ###########################################
echo [U-43]automountd 제거
echo ###########################################
# 설치되어 있지 않음
## automount 동작중인지 체크 후 동작중이면 삭제 & 중지
if [ `ps -ef | grep -i "automountd" | grep -v "grep" | wc -l` -eq 0 ]
    then
        echo "automountd 서비스가 실행중이지 않음"
    else
        ls -al /etc/rc*.d/* | grep automount
        ls -al /etc/rc*.d/* | grep autofs
        systemctl disable --now autofs.service
        echo "automountd 서비스가 실행중임"
fi


# [U-44]RPC 서비스 확인
echo ###########################################
echo [U-44]RPC 서비스 확인
echo ###########################################
# 설치되어 있지 않음
systemctl disable rpcbind


# [U-45]NIS, NIS+ 점검
echo ###########################################
echo [U-45]NIS, NIS+ 점검
echo ###########################################
# 설치되어 있지 않음
## Manual check - NIS 서비스 사용이 필요할 경우 NIS+ 서비스로 사용 그 외에는 취약
## 아래 스크립트로 NIS 서비스 사용여부 확인 후 데몬 중지
SERVICE_NIS="ypserv|ypbind|ypxfrd|rpc.yppasswdd|rpc.ypupdated|rpc.nisd"
if [ `ps -ef | egrep $SERVICE_NIS | grep -v "grep" | wc -l` -eq 0 ]
    then
        echo "NIS 서비스가 실행중이지 않음"
    else
        echo "NIS 서비스가 실행중임"
fi


# [U-46]tftp, talk 서비스 비활성화
echo ###########################################
echo [U-46]tftp, talk 서비스 비활성화
echo ###########################################
# 설치되어 있지 않음
systemctl disable tftp
systemctl disable talk


# [U-47]Sendmail 버전 점검
echo ###########################################
echo [U-47]Sendmail 버전 점검
echo ###########################################
# 설치되어 있지 않음


# [U-48]스팸 메일 릴레이 제한
echo ###########################################
echo [U-48]스팸 메일 릴레이 제한
echo ###########################################
# 설치되어 있지 않음


# [U-49]일반사용자의 Sendmail 실행 방지
echo ###########################################
echo [U-49]일반사용자의 Sendmail 실행 방지
echo ###########################################
# 설치되어 있지 않음


# [U-50]DNS 보안 버전 패치
echo ###########################################
echo [U-50]DNS 보안 버전 패치
echo ###########################################
# 설치되어 있지 않음


# [U-51]DNS ZoneTransfer 설정
echo ###########################################
echo [U-51]DNS ZoneTransfer 설정
echo ###########################################
# 설치되어 있지 않음


# [U-52]Apache 디렉토리 리스팅 제거
echo ###########################################
echo [U-52]Apache 디렉토리 리스팅 제거
echo ###########################################
# 설치되어 있지 않음


# [U-53]Apache 웹 프로세스 권한 제한
echo ###########################################
echo [U-53]Apache 웹 프로세스 권한 제한
echo ###########################################
# 설치되어 있지 않음


# [U-54]Apache 상위 디렉토리 접근 금지
echo ###########################################
echo [U-54]Apache 상위 디렉토리 접근 금지
echo ###########################################
# 설치되어 있지 않음


# [U-55]Apache 불필요한 파일 제거
echo ###########################################
echo [U-55]Apache 불필요한 파일 제거
echo ###########################################
# 설치되어 있지 않음


# [U-56]Apache 링크 사용금지
echo ###########################################
echo [U-56]Apache 링크 사용금지
echo ###########################################
# 설치되어 있지 않음


# [U-57]Apache 파일 업로드 및 다운로드 제한
echo ###########################################
echo [U-57]Apache 파일 업로드 및 다운로드 제한
echo ###########################################
# 설치되어 있지 않음


# [U-58]Apache 웹 서비스 영역의 분리
echo ###########################################
echo [U-58]Apache 웹 서비스 영역의 분리
echo ###########################################
# 설치되어 있지 않음


# [U-59]ssh 원격접속 허용
echo ###########################################
echo [U-59]ssh 원격접속 허용
echo ###########################################
echo "Manual Check"

# [U-60]ftp 서비스 확인
echo ###########################################
echo [U-60]ftp 서비스 확인
echo ###########################################
# 설치되어 있지 않음
echo "Manual Check"


# [U-61]ftp 계정 shell 제한
echo ###########################################
echo [U-61]ftp 계정 shell 제한
echo ###########################################
# 설치되어 있지 않음
echo "Manual Check"


# [U-62]ftpusers 파일 소유자 및 권한 설정
echo ###########################################
echo [U-62]Ftpusers 파일 소유자 및 권한 설정
echo ###########################################
# 설치되어 있지 않음
echo "Manual Check"


# [U-63]Ftpusers 파일 설정
echo ###########################################
echo [U-63]Ftpusers 파일 설정
echo ###########################################
# 설치되어 있지 않음
echo "Manual Check"


# [U-64]at 파일 소유자 및 권한 설정
echo ###########################################
echo [U-64]at 파일 소유자 및 권한 설정
echo ###########################################
chown root:root /etc/at.deny && chmod 640 /etc/at.deny
chown root:root /etc/at.allow && chmod 640 /etc/at.allow


# [U-65]SNMP 서비스 구동 점검
echo ###########################################
echo [U-65]SNMP 서비스 구동 점검
echo ###########################################
# 설치되어 있지 않음
echo "Manual Check"


# [U-66]SNMP 서비스 커뮤니티스트링의 복잡성 설정
echo ###########################################
echo [U-66]SNMP 서비스 커뮤니티스트링의 복잡성 설정
echo ###########################################
# 설치되어 있지 않음
echo "Manual Check"


# [U-67]로그온 시 경고 메시지 제공
echo ###########################################
echo [U-67]로그온 시 경고 메시지 제공
echo ###########################################
# define banner 인터뷰 후, 수정 적용

# [U-68]NFS 설정 파일 접근 권한
echo ###########################################
echo [U-68]NFS 설정 파일 접근 권한
echo ###########################################
chown root:root /etc/exports && chmod 644 /etc/exports

# [U-69]expn, vrfy 명령어 제한
echo ###########################################
echo [U-69]expn, vrfy 명령어 제한
echo ###########################################
# 설치되어 있지 않음
echo "Manual Check"


# [U-70]Apache 웹서비스 정보 숨김
echo ###########################################
echo [U-70]Apache 웹서비스 정보 숨김
echo ###########################################
# 설치되어 있지 않음
echo "Manual Check"


# [U-71]최신 보안패치 및 벤더 권고사항 적용
echo ###########################################
echo [U-71]최신 보안패치 및 벤더 권고사항 적용
echo ###########################################
# 기 적용함


# [U-72]로그의 정기적 검토 및 보고
echo ###########################################
echo [U-72]로그의 정기적 검토 및 보고
echo ###########################################
# SKIP
echo "Manual Check"


# [U-73]정책에 따른 시스템 로깅 설정
echo ###########################################
echo [U-73]정책에 따른 시스템 로깅 설정
echo ###########################################
echo 'function logging'  >> /etc/profile.d/cmd_logging.sh
echo '{'  >> /etc/profile.d/cmd_logging.sh
echo '        stat="$?"'  >> /etc/profile.d/cmd_logging.sh
echo '        cmd=$(history|tail -1)'  >> /etc/profile.d/cmd_logging.sh
echo '        if [ "$cmd" != "$cmd_old" ]; then'  >> /etc/profile.d/cmd_logging.sh
echo '                logger -p local1.notice "[2] STAT=$stat"'  >> /etc/profile.d/cmd_logging.sh
echo '                logger -p local1.notice "[1] PID=$$, PWD=$PWD, CMD=$cmd"'  >> /etc/profile.d/cmd_logging.sh
echo '        fi'  >> /etc/profile.d/cmd_logging.sh
echo '        cmd_old=$cmd'  >> /etc/profile.d/cmd_logging.sh
echo '}'  >> /etc/profile.d/cmd_logging.sh
echo 'trap logging DEBUG'  >> /etc/profile.d/cmd_logging.sh
echo 'local1.*                                                /var/log/bash.history'  >> /etc/rsyslog.conf
sed -i "s/cron.none/cron.none;local1.none/g" /etc/rsyslog.conf
echo "authpriv.info                                           /var/log/sulog" >> /etc/rsyslog.conf
echo "*.crit;*.warn;*.alert;*.emerg;*.err;authpriv.*          /var/log/rsyslog" >> /etc/rsyslog.conf
sed -i 's/^rotate 4/rotate 24/' /etc/logrotate.conf
systemctl restart rsyslog
chmod 600 /var/log/bash.history
chmod 600 /var/log/sulog
# create admin accout
adduser $MASTER_USER -g wheel
echo $MASTER_USER:$MASTER_USER_PASSWORD | chpasswd
# chage account ages
chage -M 90 -m 1 -W 7 root
# Delete temporary files
rm -f U-*.txt
```