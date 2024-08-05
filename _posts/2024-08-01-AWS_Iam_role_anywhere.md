---
title: AWS IAM Roles Anywhere
date: 2024-08-02 18:23:00 +/-0000
categories: [aws, iam]
tags: [IAM_Roles_Anywhere] # TAG names should always be lowercase
---

외부 환경에서 AWS 리소스에 접근해야될 경우가 있다. 이럴경우

## IAM Roles Anywhere란 무엇인가?

AWS의 역할과 정책을 사용하여 AWS 외부에서 AWS 리소스(서비스)에 대한 엑세스를 관리할 수 있습니다.  
AWS가 아닌 워크로드에 대한 장기 자격증명을 발급하고 관리할 필요가 없으므로 엑세스 관리 부담이 줄어들며 보안을 강화할 수 있습니다.
쉽게 말해서 Access Key 취약점조치사용하여 AWSCLI로 원격에서 리소스를 생성, 삭제, 변경, 조회 등을 하던 방식에서 좀더 보안적, 관리적인 목적으로써 Key사용하지 않고 인증서를 통해 인증을 받아서 사용할 수 있다는 말이다.
"Key rotation 90일" 이라는 취약점 점검 항목(실제로 지키기 어려운 항목..)에 대해서 해결도 가능한 기능이라 볼 수 있다.
다만, 이 방법도 access key 인증 방식과 동일하게 key 값이 외부 노출이 될 경우 access key와 동일하게 보안에 취약해질 수 있습니다.
그러므로 이 인증서에 대해서도 관리할 방법에 대해서 강구해야 됩니다.

자 이제, IAM Roles Anywhere에 대해서 알아봅시다.

## IAM Roles Anywhere의 구조

![IAM_Roles_Anywhere_구조](../assets/img/posts_img/AWS_iam_roles_anywhere/AWS-Summit-Seoul-2023-IAM-Roles-Anywhere.png)

### 구성요소

1. 신뢰된 주체 (Trusted Entity):
   IAM Roles Anywhere는 신뢰된 인증 기관(Certificate Authority, CA)에서 발급한 X.509 인증서를 통해 신뢰된 주체를 식별합니다. AWS 계정 내에서 신뢰할 수 있는 주체를 정의합니다.
2. IAM 역할 (IAM Role):
   AWS IAM 역할은 특정 권한을 가지고 있으며, 인증된 주체는 이 역할을 가정(Assume)하여 해당 권한을 얻을 수 있습니다.
   IAM Roles Anywhere는 이 역할을 통해 인증된 애플리케이션이 AWS 리소스에 액세스할 수 있도록 합니다.
3. 프로파일 (Profile):
   프로파일은 역할과 신뢰할 수 있는 인증서를 연결하는 설정입니다. 프로파일은 하나 이상의 IAM 역할과 연관될 수 있으며, 각 프로파일은 특정 권한을 부여합니다.  
   _\*AWSCLI 사용을 위한 aws configure과는 상관이 없는 용어입니다.\*_
4. 크리덴셜 공급자 (Credential Provider):
   IAM Roles Anywhere에서 제공하는 크리덴셜 공급자를 사용하여 외부 애플리케이션이 IAM 역할을 가정하고, AWS API를 호출하는 데 필요한 일시적인 보안 자격 증명을 획득합니다.

### 인증 동작의 흐름

1. 인증서 발급: 신뢰할 수 있는 인증 기관(CA)에서 인증서를 발급받습니다.
2. 인증서 사용: 애플리케이션은 이 인증서를 사용하여 AWS API 요청을 서명합니다.
3. 검증 및 신원 확인: AWS IAM Roles Anywhere는 이 요청을 받아 인증서를 검증하고 신원을 확인합니다.
4. assume role 및 자격 증명 발급: 검증이 완료되면 애플리케이션에 일시적인 자격 증명이 발급되고, 이를 통해 AWS IAM assume role를 통해 리소스에 접근합니다.

## Iam Roles Anywhere 구성하기

### ROOT CA 구성

AWS에서 제공해주고 있는 'Private Certificate Authority' 를 통해서 CA 구성하고 인증서를 생성할 수 있으나, 해당 기능은 월에 '400 USD'라는 비용이 발생하기 때문에 비용 절감을 위해서 openssl을 이용하여 인증서를 생성할 것 입니다.

```bash
$openssl genrsa -aes256 -out jsp-rootca.key 2048
#jsp-rootca.key 생성
```

### CSR(Certificate Signing Request) 요청 파일 생성

```bash
$touch jsp_rootca_openssl.conf
$cat << EOF > jsp_rootca_openssl.conf
[ req ]
default_bits            = 2048
default_md              = sha256
default_keyfile         = jsp-rootca.key
distinguished_name      = req_distinguished_name
extensions             = v3_ca
req_extensions = v3_ca
[ v3_ca ]
basicConstraints       = critical, CA:TRUE, pathlen:0
subjectKeyIdentifier   = hash
##authorityKeyIdentifier = keyid:always, issuer:always
keyUsage               = keyCertSign, cRLSign
nsCertType             = sslCA, emailCA, objCA
[req_distinguished_name ]
countryName                     = Country Name (2 letter code)
countryName_default             = KR
countryName_min                 = 2
countryName_max                 = 2
organizationName              = Organization Name (eg, company)
organizationName_default      = jsp Inc.
organizationalUnitName          = Organizational Unit Name (eg, section)
organizationalUnitName_default  = jsp Project
commonName                      = Common Name (eg, your name or your server's hostname)
commonName_default             = jsp's Self Signed CA
commonName_max                  = 64
EOF

# 아래 명령어를 통해서
$openssl req -new -key jsp-rootca.key -out jsp-rootca.csr -config jsp_rootca_openssl.conf
Enter pass phrase for jsp-rootca.key:
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [KR]:
Organization Name (eg, company) [jsp Inc.]:
Organizational Unit Name (eg, section) [jsp Project]:
Common Name (eg, your name or your servers hostname) [jsps Self Signed CA]:

# csr 인증요청파일 확인
$cat jsp-rootca.csr
```

### Certificate file 생성

```bash
$openssl x509 -req -days 3650 \
  -extensions v3_ca \
  -set_serial 1 \
  -in jsp-rootca.csr \
  -signkey jsp-rootca.key \
  -out jsp-rootca.crt \
  -extfile jsp_rootca_openssl.conf \
  -sha256

Enter pass phrase for jsp-rootca.key:
Certificate request self-signature ok
subject=C = KR, O = jsp Inc., OU = jsp Project, CN = jsps Self Signed CA

# 명령을 통해서 인증서의 내용확인
$openssl x509 -text -in jsp-rootca.crt

Certificate:
    Data:
        Version: 3 (0x2)      # Certifacates는 X.509v3 이어야 됨.
        Serial Number: 1 (0x1)
        Signature Algorithm: sha256WithRSAEncryption     # Signature Algorithm => Trust Anchor에 등록을 위해서 필요한 값
        Issuer: C = KR, O = jsp Inc., OU = jsp Project, CN = jsps Self Signed CA
        Validity
            Not Before: Aug  2 05:21:48 2024 GMT
            Not After : Jul 31 05:21:48 2034 GMT
        Subject: C = KR, O = jsp Inc., OU = jsp Project, CN = jsps Self Signed CA
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:d0:03:d3:51:3c:64:41:25:ed:42:5b:98:70:8f:
                    bd:17:9f:75:3c:55:87:f9:f7:3d:82:45:c3:cb:86:
                    48:d5:87:c0:1b:69:09:d0:2f:2e:63:5b:07:bc:56:
                    49:85:f3:4c:8f:0c:82:69:58:38:7d:48:d7:9f:31:
                    27:1d:2c:c9:06:93:bd:c9:9b:80:1a:4a:b4:c6:9a:
                    e1:ba:d3:21:d2:2e:d9:78:aa:71:c3:9e:d6:e0:55:
                    54:db:03:6b:a5:31:74:57:55:a9:6f:22:9d:0c:c7:
                    1d:9d:4b:21:71:4c:b7:5b:ae:87:a1:60:21:5d:27:
                    76:35:28:51:aa:07:ef:18:09:1b:09:ba:06:f0:7b:
                    54:3c:ae:35:60:48:3d:15:68:8a:26:61:dd:58:1b:
                    d4:15:79:3f:17:6f:d0:f4:64:97:34:19:cd:66:d3:
                    d7:82:70:9f:c9:32:33:53:ee:10:39:8a:41:4f:86:
                    3b:e6:8f:53:b8:a7:4d:38:a2:ab:44:9f:13:60:5b:
                    ef:2a:77:ee:ba:05:fb:94:39:64:aa:f5:da:62:5a:
                    e9:0b:97:72:4b:f1:46:e6:7d:f3:ad:94:73:2e:6d:
                    05:81:17:5a:91:f1:11:65:1f:0f:46:fb:ca:84:63:
                    83:5f:44:f3:b5:fc:ec:04:89:1f:ba:9b:74:61:bb:
                    4f:bb
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Basic Constraints: critical
                CA:TRUE, pathlen:0              # CA:TRUE => Trust Anchor에 등록을 위해서 필요한 값
            X509v3 Subject Key Identifier:
                36:3B:4F:EA:92:7F:5C:C7:13:EC:78:07:0A:D5:FD:A8:2A:40:2A:61
            X509v3 Key Usage:
                Certificate Sign, CRL Sign      # Certificate Sign, CRL Sign => Trust Anchor에 등록을 위해서 필요한 값
            Netscape Cert Type:
                SSL CA, S/MIME CA, Object Signing CA
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
...
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
```

### Trust Anchor 등록

1. Trust Anchor 생성
   IAM > Roles > Roles Anywhere > Manage > Create a trust anchor  
   ![Create Trust Anchor](../assets/img/posts_img/AWS_iam_roles_anywhere/trust_anchor-1.png)

2. IAM_Roles_Anywhere를 위한 role 생성
   IAM > Roles > Create role

   - Trusted entity type = AWS service
   - Use case의 Service or use cate = Roles Anywhere
     ![Create IAM_Roles_Anywhere_role](../assets/img/posts_img/AWS_iam_roles_anywhere/Create_Roles_Anywhere_role-1.png)

   - name 작성
     ![Create IAM_Roles_Anywhere_role](../assets/img/posts_img/AWS_iam_roles_anywhere/Create_Roles_Anywhere_role-2.png)

   - Add permissions = 필요한 접근에 대해서 policy 추가
     ![Create IAM_Roles_Anywhere_role](../assets/img/posts_img/AWS_iam_roles_anywhere/Create_Roles_Anywhere_role-3.png)

### Create Profile

IAM > Roles > Roles Anywhere > Manage > Create a profile

- Profile name = profile 이름 작성
- Roles = 위에서 만들었던 IAM Roles Anywhere Role 선택
- Session policy = 세션에 대한 policy 를 추가
- Session duration = Maximum session duration은 기본 1시간 ~ 12시간 선택이 가능하며, Custom duration 적용시 15분에서 12시간까지 가능  
  ![Create Profile-1](../assets/img/posts_img/AWS_iam_roles_anywhere/Create_Profile-1.png)  
  ![Create Profile-1](../assets/img/posts_img/AWS_iam_roles_anywhere/Create_Profile-2.png)

### Roles Anywhere 사용을 위한 Server 인증서 발급

서버상에서 사용할 인증서를 생성합니다. 개인키 생성 후 암호화 해제를 해줍니다.

```bash
$openssl genrsa -aes256 -out jsp.com.key 2048
Enter PEM pass phrase:
Verifying - Enter PEM pass phrase:

$cp jsp.com.key jsp.com.key.enc

# 개인키 Key를 복호화
$openssl rsa -in jsp.com.key.enc -out jsp.com.key
Enter pass phrase for jsp.com.key.enc:
writing RSA key

# key의 유효성 검증
$openssl rsa -in jsp.com.key -check
RSA key ok
writing RSA key
-----BEGIN PRIVATE KEY-----
...
-----END PRIVATE KEY-----
```

### Server CSR 요청 파일 생성

jsp_server_openssl.conf 파일 생성하여 아래 내용 추가

```bash
$touch jsp_server_openssl.conf
$cat << EOF > jsp_server_openssl.conf
[req]
default_bits = 2048
default_md = sha256
default_keyfile = jsp-rootca.key
distinguished_name = req_distinguished_name
extensions = v3_user
[v3_user]
basicConstraints = CA:FALSE
authorityKeyIdentifier = keyid,issuer
subjectKeyIdentifier = hash
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth,clientAuth
subjectAltName = @alt_names
[alt_names]
DNS.1 = www.jsp.com
DNS.2 = jsp.com
DNS.3 = \*.jsp.com
[req_distinguished_name]
countryName = Country Name (2 letter code)
countryName_default = KR
countryName_min = 2
countryName_max = 2
organizationName = Organization Name (eg, company)
organizationName_default = jsp Inc.
organizationalUnitName = Organizational Unit Name (eg, section)
organizationalUnitName_default = jsp Project
commonName = Common Name (eg, your name or your server's hostname)
commonName_default = jsp.com
commonName_max = 64
EOF

# Server의 csr 파일 생성
$openssl req -new -key jsp.com.key -out jsp.com.csr -config jsp_server_openssl.conf
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [KR]:
Organization Name (eg, company) [jsp Inc.]:
Organizational Unit Name (eg, section) [jsp Project]:
Common Name (eg, your name or your servers hostname) [jsp.com]:
```

### Server Certification 생성

```bash
# 생성
$openssl x509 -req -days 1825 -extensions v3_user -in jsp.com.csr \
  -CA jsp-rootca.crt -CAcreateserial \
  -CAkey jsp-rootca.key \
  -out jsp.com.crt \
  -extfile jsp_server_openssl.conf \
  -sha256
Certificate request self-signature ok
subject=C = KR, O = jsp Inc., OU = jsp Project, CN = jsp.com
Enter pass phrase for jsp-rootca.key:           # password 입력

# 인증서 확인
$openssl x509 -text -in jsp.com.crt
Certificate:
    Data:
        Version: 3 (0x2)                           # 발급된 인증서 사용하기 위한 조건 = X.509v3
        Serial Number:
            3c:a1:6b:81:a5:25:9c:a1:de:7c:74:6b:e4:13:65:6e:95:d3:52:aa
        Signature Algorithm: sha256WithRSAEncryption        # 발급된 인증서 사용하기 위한 조건 = 인증서에 Signature Algorithm 가 SHA256이나 그 이상 이어야됨.
        Issuer: C = KR, O = jsp Inc., OU = jsp Project, CN = jsps Self Signed CA
        Validity
            Not Before: Aug  2 07:48:48 2024 GMT
            Not After : Aug  1 07:48:48 2029 GMT
        Subject: C = KR, O = jsp Inc., OU = jsp Project, CN = jsp.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    ...
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Basic Constraints:
                CA:FALSE                     # 발급된 인증서 사용하기 위한 조건 = CA:FALSE
            X509v3 Authority Key Identifier:
                36:3B:4F:EA:92:7F:5C:C7:13:EC:78:07:0A:D5:FD:A8:2A:40:2A:61
            X509v3 Subject Key Identifier:
                29:F7:87:E9:B7:B3:2A:C4:A0:84:27:EB:7D:92:A8:FC:B9:4F:E3:45
            X509v3 Key Usage:
                Digital Signature, Non Repudiation, Key Encipherment          # 발급된 인증서 사용하기 위한 조건 = 인증서에 Digital Signature 가 있어야됨.
            X509v3 Extended Key Usage:
                TLS Web Server Authentication, TLS Web Client Authentication
            X509v3 Subject Alternative Name:
                DNS:www.jsp.com, DNS:jsp.com, DNS:*.jsp.com
    Signature Algorithm: sha256WithRSAEncryption
    Signature Value:
        ...
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
```

### 임시 보안 자격 증명 얻기

'credential helper tool' 설치를 위해서 아래 링크로 가서 다운로드 url 확인  
URL = [download_tool](https://docs.aws.amazon.com/rolesanywhere/latest/userguide/credential-helper.html)

OS 버전에 맞게 다운받아서 사용

```bash
$wget https://rolesanywhere.amazonaws.com/releases/1.1.1/X86_64/Linux/aws_signing_helper
$chmod +x aws_signing_helper
$./aws_signing_helper credential-process \
      --certificate jsp.com.crt \
      --private-key jsp.com.key \
      --trust-anchor-arn {Trust-anchor-ARN} \
      --profile-arn {Profile-ARN} \
      --role-arn {IAM_Roles_Anywhere_role-ARN}
```

결과는 아래와 같이 출력된다.
![signing_helper](../assets/img/posts_img/AWS_iam_roles_anywhere/start_signing_helper.png)

### 인증서를 aws config파일에 적용하기

```bash
$cp jsp.com.crt jsp.com.key ~/.aws/
$cat << EOF >> ~/.aws/config
[profile roles-anywhere]
credential_process = ./aws_signing_helper credential-process --certificate jsp.com.crt --private-key jsp.com.key --trust-anchor-arn {Trust-anchor-ARN} --profile-arn {Profile-ARN} --role-arn {IAM_Roles_Anywhere_role-ARN}
region=ap-northeast-2
EOF

# 정상적으로 roles-anywhere profile통해서 sts 값을 가져오는지 확인
$aws sts get-caller-identity --profile roles-anywhere
{
    "UserId": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    "Account": "xxxxxxxxxxxx",
    "Arn": "arn:aws:sts::xxxxxxxxxxxx:assumed-role/jsp-iam-roles-anywhere-role/xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
}
```

제대로 프로파일로 awscli가 동작되는지 확인해봅시다.

```bash
$aws s3 ls --profile roles-anywhere

2022-12-22 12:18:49 aws-cloudtrail-logs-xxxxxxxxxxxx-xxxxxxxx
2024-01-10 10:49:04 aws-cloudtrail-logs-xxxxxxxxxxxx-xxxxxxxx
2023-12-15 16:08:14 jsp-test-s3-bucket
```

##### _위의 방법을 사용하지 않고 Root CA를 AWS의 Private CA를 이용하여 구성시 알아야 될 지식_

Certificate Revocation(인증서 폐기)

단기 인증서의 경우에는 몇 시간 또는 며칠이면 자동 만료되기 때문에 폐기 할 필요가 없습니다.
그러나 그 외에는 앤드포인트에서 발급된 인증서를 더 이상 신뢰하지 않기 위하여 인증서를 폐기하는 방법이 필요합니다.
인증서를 폐기하는 방법은 2가지가 존재합니다.

1. 인증서 폐기 목록 - Certificate Revocation Lists (CRLs)
   인증서 발급자(CA)로부터 폐기된 인증서 목록을 의미하며, 폐기된 인증서의 Serial, 폐기 날짜, 폐기 사유 등의 정보를 담고있습니다.
   CRL은 배포 지점으로부터 주기적으로 다운을 받아서 리스트를 갱신해주어야 됩니다.
   갱신 주가기 길면 폐기된 인증서를 사용할 수 있으며, 짧으면 인증서를 사용할 때마다 오버헤드가 증가하게 됩니다.

2. 온라인 인증서 상태 프로토콜 - Online Certificate Status Protocol (OCSP)
   CRLs의 주기적인 갱신문제를 보완한 프로토콜입니다.
   사용자가 서버로 접근을 시도시 인증서 상태 정보를 실시간으로 요청하여 인증서의 유효성 여부를 즉시 응답해 줍니다.
   HTTP Header에 Content-Type으로 application/ocsp-request를 포함하고 있으며, 응답을 보낼 시 Content-Type으로 application/ocsp-response를 포함하여 보냅니다.
