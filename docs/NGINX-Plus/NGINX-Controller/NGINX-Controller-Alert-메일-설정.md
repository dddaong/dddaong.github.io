---
title: "NGINX Controller - Alert 메일 설정"
author: DDAONG
date:   2021-05-04
category: "NGINX Controller"
layout: post
---


## NGINX Controller - Alert 메일 설정

NGINX Controller Alert mail을 위한 Sendmail 설정입니다.  

NGINX Controller의 Alert 메뉴에서는 NGINX Controller에서 제공하는 Alert, Notification 기능을 설정할 수 있습니다. 이를 통해 System, App Performance의 상태를 파악할 수 있습니다.  

단, Alert Mail은 Verified Address로만 발송이 되며, 이를 위한 간단한 설정을 이 포스트에서 소개합니다.
덧붙여, NGINX Controller 설치 중 설정했던 SMTP 정보 수정에 대한 내용입니다.

### Sendmail 설치 및 설정

Sendmail은 Postfix와 함께, SMTP를 포함하여 수많은 종류의 메일 전송 및 전달 방식을 지원하는 리눅스 범용 메일 전송 툴입니다.

#### 1. `sendmail` 설치

```bash
yum -y install sendmail sendmail-cf
```

#### 2. `/etc/mail/sendmail.mc`  편집

sendmail은 기본값으로 Localhost[127.0.0.1]의 메일 발송만을 허용합니다.

NGINX Controller는 Single Node Kubernetes의 구조로서,
Alert Mail 등은 nats-streaming-worker 파드로부터 발신되므로,
Pod network 대역의 메일 발송 허용이 필요합니다.

1. sendmail.mc를 수정해 sendmail 외부 접근을 허용 설정을 수행합니다.

```bash
vi /etc/mail/sendmail.mc
```

인증 메커니즘을 선언하는 부분입니다. `dnl #`은 주석 역할을 하는 것으로 보이며, 삭제하면 설정이 적용됩니다.

```bash
dnl # TRUST_AUTH_MECH(`EXTERNAL DIGEST-MD5 CRAM-MD5 LOGIN PLAIN')dnl
dnl # define(`confAUTH_MECHANISMS', `EXTERNAL GSSAPI DIGEST-MD5 CRAM-MD5 LOGIN PLAIN')dnl

> TRUST_AUTH_MECH(`EXTERNAL DIGEST-MD5 CRAM-MD5 LOGIN PLAIN')dnl
> define(`confAUTH_MECHANISMS', `EXTERNAL GSSAPI DIGEST-MD5 CRAM-MD5 LOGIN PLAIN')dnl
```

마찬가지로 sendmail.mc 파일에서 Listen IP를 변경합니다.

```bash
DAEMON_OPTIONS(`Port=smtp,Addr=127.0.0.1, Name=MTA')dnl

> DAEMON_OPTIONS(`Port=smtp,Addr=0.0.0.0, Name=MTA')dnl
```

아래 명령어로 cf 파일을 덮어쓰기 합니다.

```bash
m4 /etc/mail/sendmail.mc > /etc/mail/sendmail.cf
```

#### 3. `/etc/mail/access` 편집

아래 명령어로 Pod Network 대역을 Access 허용 설정해줍니다.

```bash
echo 'Connect:10.244.0                        RELAY' >> /etc/mail/access
```

#### 4. 메일 발송 테스트

1. 아래 Curl 명령으로 NGINX SMTP 설정을 확인할 수 있습니다.

```bash
   NGINX_CTLR='xxx.xxx.xxx.xxx'
   NUSERNAME=''
   NPASSWORD=''
   
   # Authentication
   curl -sk -c cookie.txt -X POST --url "https://$NGINX_CTLR/api/v1/platform/login" --header 'Content-Type: application/json' --data '{"credentials": {"type": "BASIC","username": "'"$NUSERNAME"'","password": "'"$NPASSWORD"'"}}'
   
   # API Query - Global setting
   curl -sk -b cookie.txt -c cookie.txt -X GET --url "https://$NGINX_CTLR/api/v1/platform/global" --header 'Content-Type: application/json'
```

2. `/opt/nginx-controller` 디렉토리의 `helper.sh` 스크립트를 사용해 SMTP 정보를 수정할 수 있습니다.
   수정에는 최대 2분까지 시간이 소요될 수 있습니다.

```bash
   #Usage
   ./helper.sh configsmtp <address> <port> <tls> <from> [auth] [username] [password]
   
   # Plaintext, 25 Port, No Auth
   ./helper.sh configstmp $NGINX_CTLR 25 false noreply@itian.co.kr false
```

3. NGINX Controller GUI에서, Analytics - Alert Rules 메뉴로 진입합니다.

![Analytics - Alert Rules](../imgs/analytics-alertrules.png)

Alert Rules 생성 화면에서 Manage E-mail Address를 선택해 E-mail Verification을 수행할 수 있습니다.

![](../imgs/analytics-alertrules-create2.png)

Manage Email Addresses에서 Resend Verification Mail 버튼으로 Verification Mail을 발송할 수 있습니다.

- [Notice] Alert Mail은 Verified Email Address로만 발송합니다.
  Verification이 완료되지 않으면 발송되지 않습니다.

![Manage Email Addresses](../imgs/analytics-alertrules-managemail.png)
