---
title: BIG-IP - Syslog Custom Config Sample
author: DDAONG
category: "BIG-IP"
layout: post
---

# BIG-IP Syslog Custom Config

본 문서에서는 주요 Monitoring 대상 로그에 대해 Syslog Config include에서 사용할 수 있는 샘플 Config를 제시합니다. 

복사/붙여넣기 용도의 같은 이름의 TXT파일을 사용하면 좀더 안전하게 Custom Config를 사용할 수 있습니다.



## Syslog Custom Config 대상
- Syslog Custom Config 대상 주요 이벤트를 아래와 같이 정리합니다.

- 아래 표들은 말그대로 로그 샘플이며 Syslog는 해당 로그의 Text를 Pattern matching을 수행하여 매칭되는 Syslog Line을 Remote Server로 전달합니다.


### Hardware 로그
|     구분               |  위치 |     로그 샘플                                                                                       |
|------------------------|-------|-----------------------------------------------------------|
| CPU temp. high         |  ltm    |  `Cpu   %d: temperature (%d) is too high.`                          |
| FAN speed low      |  ltm    |  `Cpu %d: fan speed (%d) is too low.`                              |
| FAN down       |  ltm    |  `Chassis   fan %d: status (%d) is bad.`                               |
| MEM usage high     |  ltm    |  `%s: Aggressive mode %s %s (%llx) (%s %s).   (%u/%u %s)`                      |
| DISK usage high    |  ltm    |  `%s   disk usage exceeds  %d%%. Reduce log   disk space now`                  |
| Chassis Temperature|  ltm    |  `Chassis %d: temperature (%d) is too high.`                           |
| PSU Down       |  ltm    |  `Chassis   power supply %d is not supplying power (status: %d): make sure it is plugged   in.`|
| Interface Up/Down  |  ltm    |  `Link: %s is %s`                                          |

### 지정된 인증서 만료 로그
|     구분                   |     위치    |     로그 샘플                                                                                       |
|----------------------------|-------------|-------------------------------------------------------------------------|
|     Certificate 만료 예정 (30일부터)    |     ltm    |     `warning tmsh[16526]:   01420008:4: Certificate   'CN=test-domain.com,OU=ADC,O=IT,L=Seoul,ST=Seoul,C=KR' in file   /Common/be_expire_soon will expire on Apr 30 01:36:52 2022 GMT`    |
|     Certificate 만료                    |     ltm    |     `warning   tmsh[16900]: 01420007:4: Certificate 'CN=I000105479,OU=Client   certificates,O=NGINX Inc' in file /Common/expired expired on Jul  3 14:17:32 2021 GMT`                     |

### User 접속 log
|     구분                   |     위치    |     로그 샘플                                                                                       |
|----------------------------|-------------|-------------------------------------------------------------------------|
| SSH | secure | `info sshd(pam_audit)[15360]: 01070417:6: AUDIT - user kjs - RAW: sshd(pam_audit): user=kjs(kjs) partition=[All] level=Administrator tty=ssh host=175.196.233.10 attempts=1 start="Fri Apr 15 10:24:49 2022".`|
| HTTPD | secure | `info httpd(pam_audit)[28883]: 01070417:6: AUDIT - user kjs - RAW: httpd(pam_audit): user=kjs(kjs) partition=[All] level=Administrator tty=(unknown) host=175.196.233.10 attempts=1 start="Fri Apr 15 10:33:27 2022" end="Fri Apr 15 10:33:27 2022".`|


### User 접속 실패  log  (미인가 접속 시도)
|     구분                   |     위치    |     로그 샘플                                                                                       |
|----------------------------|-------------|-------------------------------------------------------------------------|
| SSH Fail | secure | `notice sshd[15293]: pam_unix(sshd:auth): authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=175.196.233.10  user=kjs` |
| HTTPD Fail | secure | `notice httpd[10459]: pam_unix(httpd:auth): authentication failure; logname= uid=48 euid=48 tty= ruser= rhost=175.196.233.10  user=admin` |


## Syslog Include 설정
tmsh# edit sys syslog all-properties 명령어로 vi 편집기로 진입해, include 부분 아래에 아래와 같이 작성합니다.
```tmsh
sys syslog {
    include "

    filter f_hw_1heat { facility(local0) and match(\"Cpu.*too\"); };
    filter f_hw_2fan { facility(local0) and match(\"Cpu.*fan"); };
    filter f_hw_3fan { facility(local0) and match(\"Chassis fan\"); };
    filter f_hw_4mem { facility(local0) and match(\"Aggressive mode\"); };
    filter f_hw_5disk { facility(local0) and match(\"disk usage exceeds"\); };
    filter f_hw_6psu { facility(local0) and match(\"power supply"\); };

    filter f_warntoemerg { facility(local0);  level(warning..emerg); };
    filter f_authinfo { facility(local0) and match(\"pam_audit|pam_unix\"); };
    filter f_checkcert { facility(local0) and match(\"Ceritificate.*expire\"); };

    destination d_kjs_test { file(/var/log/kjs_syslog_test create_dirs(yes)); };
    destination d_logsvr { udp(\"10.100.50.202\" port(514)); };

    log { source(s_syslog_pipe); filter(f_hw_1heat); destination(d_kjs_test); };
    log { source(s_syslog_pipe); filter(f_hw_2fan); destination(d_kjs_test); };
    log { source(s_syslog_pipe); filter(f_hw_3fan); destination(d_kjs_test); };
    log { source(s_syslog_pipe); filter(f_hw_4mem); destination(d_kjs_test); };
    log { source(s_syslog_pipe); filter(f_hw_5disk); destination(d_kjs_test); };
    log { source(s_syslog_pipe); filter(f_hw_6psu); destination(d_kjs_test); };
		
    log { source(s_syslog_pipe); filter(f_authinfo); destination(d_kjs_test); };
    log { source(s_syslog_pipe); filter(f_checkcert); destination(d_kjs_test); };
    log { source(s_syslog_pipe); filter(f_warntoemerg); destination(d_kjs_test); };
    "
}

```