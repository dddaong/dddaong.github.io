---
title: "NGINX+ - JWT 인증 테스트"
author: DDAONG
date:   2021-04-07 18:50:00 +0900
category: NGINX+
layout: post
---

NGINX Plus로 간략한 JWT 인증 테스트를 수행합니다.

## NGINX Plus + JWT 인증 테스트

### NGINX의 JWT Validation Supports

NGINX Plus는 R10 버전부터 Native JWT Validation을 지원하며,
NGINX OSS에서는 [lua-resty-openidc](https://github.com/zmartzone/lua-resty-openidc) 등 외부 모듈 사용이 필요합니다.

NGINX Plus 는 표준에 정의된  `HSxxx`, `RSxxx`, and `ESxxx` [signature algorithms](https://nginx.org/en/docs/http/ngx_http_auth_jwt_module.html)을 지원합니다.

#### Sample Config : NGINX Plus JWT Authentication Config

아래는 [Nginx Plus r17 Release](https://www.nginx.com/blog/nginx-plus-r17-released/?_ga=2.18265401.1678908472.1617607349-1458027870.1616747210#r17-openid)에서 발췌한 NGINX Plus Config 예시입니다.

```nginx
# Create a directory to cache the keys received from IdP
proxy_cache_path /var/cache/nginx/jwk levels=1 keys_zone=jwk:1m max_size=10m;

server {
    listen 80; # Use SSL/TLS in production

    location / {
        auth_jwt "closed site";
        auth_jwt_key_request /_jwks_uri; # Keys will be fetched by subrequest

        proxy_pass http://my_backend;
    }
    location = /_jwks_uri {
        internal;
        proxy_method GET;
        proxy_cache  jwk; # Cache responses
        proxy_pass   https://idp.example.com/oauth2/keys; # Obtain keys from here
    }
}
```

1. 이 Config는 `auth_jwt_key_file`를 사용하지 않고 `auth_jwt_key_request` Directive를 사용해 JWK를 Remote Location에서 가져오도록 작성되어 있습니다.

   이 Config에서 Remote Location은 `proxy_pass`에 정의된 IdP URI입니다.
   (대부분의 idP는 Key Set을 가져올 수 있는 URL을 제공한다고 합니다.)

2. Client Request를 수신하면 `auth_jet_key_request` Directive에 의해 `/_jwk_uri` 블록으로 Subrequest를 보냅니다.

3. `/_jwk_uri` 블록의 proxy_pass에 의해 idP의 고정 URL로 Subrequest가 전달되고, 응답으로 Key Set을 수신합니다.
   `/_jwk_uri` Location 블록은 `internal` Directive를 사용해 외부에서는 접근할 수 없도록 합니다.

4. 수신한 Key Set을 `proxy_cache` Directive로  `jwk keys_zone`에 캐시하여, Validation Overhead를 줄여줄 수 있도록 되어 있습니다.

5. 이 Config는 `auth_jwt "closed site";` 에서 기본값인 Bearer Token을 사용하고, realm만 작성되어 있습니다.
   JWT는 일반적으로 `Authorization` 헤더에 Bearer Token으로 인증 요청되지만,
   Cookie, Query String으로 사용될 수 있도록, `auth_jwt` Directive에서 세부 설정이 가능하도록 되어 있습니다.

   ```nginx
   auth_jwt string [token=$variable] | off;
   ```

   예를 들어 아래와 같이 Query String을 통해 Token 인증 요청을 사용하려면

   ```
   curl http://192.168.1.1/products/widget1?apijwt=xxxxxxxxxxxxxxxxxxxxx...
   ```

   string을 realm 명, 토큰을 apijwt Argument의 Value를 참조하도록 합니다.

   ```nginx
   auth_jwt "API" token=$arg_apijwt;
   ```

   참고로, $arg_name 과 $cookie_name 변수는 [ngx_http_core_module 모듈의 Embedded Variables](http://nginx.org/en/docs/http/ngx_http_core_module.html#variables)를 참조하면 됩니다.

------

### NGINX Plus JWT Auth Test with RS256

NGINX Plus에서 JWT 인증 테스트를 기록한 내용입니다.

- RSA Pub/Priv Key 생성 - mkjwk.org 사용
- NGINX Plus에서 사용할 JWK 파일 작성 - mkjwk.org 사용
- JWT 생성 - jwt.io Debugger 사용

#### JWK와 JWT를 위한 RSA256 키 생성

테스트에서는 편의를 위해 [mkjwk JSON Web Key Generator](https://mkjwk.org/)를 사용했습니다.
아래는 생성시 선택한 옵션입니다.

| Tab  | Key Size |  Key Use  | Algorithm |     Key ID     | Show X.509 |
| :--: | :------: | :-------: | :-------: | :------------: | :--------: |
| RSA  |   2048   | Signature |   RS256   | Specify [0001] |    Yes     |

#### JWK 파일 생성

1. Public Key 박스의 내용을 복사해

2. NGINX Plus가 동작하고 있는 호스트에서 `vi /etc/nginx/conf.d/dddaong.jwk` 파일로 저장합니다.

3. 아래는 테스트에 사용한 샘플입니다.

   ```json
   { "keys":
   [{
       "kty":"RSA",
       "e":"AQAB",
       "use":"sig",
       "kid":"0001",
       "alg":"RS256",    "n":"oS4lA_ePN4bmcaAM5AFnGVtzMmBBXoYxVGgNEtbhotDMkiIfhilRSH_IVktaDwdorEpQZxg66gwV0y5hZ4Nqo3wWxRVWKoVmdY0xQyPCtIHOs40FYVoc85vBU4giQnl_XqNHVnGOIZz7hiKgzzBBWiLFqgajzgvYN6QZBT0rb8OMj2u-iKQra9sHhlXBgg6W3YUE8BoPiDX2ehI35zXbsf4rI1IYeJvzEFLVcv6kHNJ7vSO0qDmVY4_m-0Kdi0KmJ_XZSNs7fRFwp1MYui9SXs-rZxrmHHES3xuTutJ5sJD6J2LgqkA8NgqhiTva-Dfyj08KVEj5RH1Af4F-bo6aBQ"
   }]
   }
   ```

#### JWT 생성

JWT는 jwt.io의 Debugger를 사용하면 간단하게 생성할 수 있습니다.

먼저, Header를 Base64url 인코딩한 스트링 + . + Payload를 Base64url 인코딩한 스트링을 준비합니다.

1. Header와 Payload를 각각 작성해 Base64url 인코딩합니다.

```bash
HEAD=$(echo -n '{"alg":"RS256","typ":"JWT","kid":"0001"}' | base64 | sed s/\+/-/ | sed -E s/=+$//)
PAYLOAD=$(echo -n '{"sub":"RS256TEST","aud":"dddaong","iat": 1617778800}' | base64 | sed s/\+/-/ | sed -E s/=+$//)
echo "$HEAD.$PAYLOAD"
```

결과물은 아래와 같이 나옵니다.

```bash
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJSUzI1NlRFU1QiLCJhdWQiOiJkZGRhb25nIn0
```

2. 생성한 내용을 [jwt.io Debugger](https://jwt.io/#debugger-io)에 Encoded 부분에 붙여넣습니다.
3. [jwt.io Debugger](https://jwt.io/#debugger-io) 우측 Decoded의 Verify Signature 항목의 아래쪽 박스에
   [mkjwk JSON Web Key Generator](https://mkjwk.org/)에서 생성한 Private Key(X.509 PEM 포맷)을 붙여넣습니다.
   Private Key를 붙여넣는 즉시 Encoded에 JWT가 완성되어 나타납니다.

결과물은 아래와 같이 나옵니다.

```bash
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJSUzI1NlRFU1QiLCJhdWQiOiJkZGRhb25nIiwiaWF0IjoxNjE3Nzc4ODAwfQ.RgG2pJVjmNs0HRN13PRgI8lhVrZgCClbkA5JWgliewp0FJDqFE_B9HNUOQN_Iroj8s_bWRTSAVpCznVdHiT_4dCpRZxpYdZEYAx9PT0H5J-mj2rBUAJYy6OC7mIXFD11nF2uSId9oVMhaa_itvNOPiV5_rIYVjVOOQWEK08DwAkQJ-dMGf8BNAnWWcsCnHE8jyrKxCpINwGSVNVL65v2Dr66bhbY2uGYs658BXnq233TQMsnj_UdxQkAgFVkm1tnkqq-vPAGV662q22mICIjGDKWyj_abOsvDGaX4PwCxyv_zj0JhiTzhi5iL27WIxXf9Je03KOvsnDc67VFV-yOCA
```

4. 확인을 위해 Public Key로 Validation을 해보겠습니다.

   - 생성한 JWT 문자열을 메모장 등에 따로 보관해둡니다.
     테스트 과정에서는 편의를 위해 토큰의 내용을 /etc/nginx/conf.d/dddaong.token 파일로 저장했습니다.

     ```bash
     echo "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJSUzI1NlRFU1QiLCJhdWQiOiJkZGRhb25nIiwiaWF0IjoxNjE3Nzc4ODAwfQ.RgG2pJVjmNs0HRN13PRgI8lhVrZgCClbkA5JWgliewp0FJDqFE_B9HNUOQN_Iroj8s_bWRTSAVpCznVdHiT_4dCpRZxpYdZEYAx9PT0H5J-mj2rBUAJYy6OC7mIXFD11nF2uSId9oVMhaa_itvNOPiV5_rIYVjVOOQWEK08DwAkQJ-dMGf8BNAnWWcsCnHE8jyrKxCpINwGSVNVL65v2Dr66bhbY2uGYs658BXnq233TQMsnj_UdxQkAgFVkm1tnkqq-vPAGV662q22mICIjGDKWyj_abOsvDGaX4PwCxyv_zj0JhiTzhi5iL27WIxXf9Je03KOvsnDc67VFV-yOCA" > /etc/nginx/conf.d/dddaong.token
     ```

   - [jwt.io Debugger](https://jwt.io/#debugger-io)의 Decoded에 붙여넣은 Private Key를 지워줍니다.
     Private Key를 지우면 JWT String도 모두 사라집니다.

   - 보관한 JWT String을 다시 붙여넣어 줍니다. 이 부분만 있어도 Header와 Payload의 내용을 확인할 수 있습니다.

- [mkjwk JSON Web Key Generator](https://mkjwk.org/)의 Public Key(X.509 PEM Format)를 Decoded의 Verify Signature 항목의 위쪽 박스에 붙여넣습니다.
  
- Debugger 왼쪽 하단에 "Signature Verified"가 나타나면 정상입니다.

#### JWK를 NGINX Plus에 적용

NGINX Plus를 설치한 후, /etc/nginx/conf.d/ 디렉토리 아래 내용의 conf 파일을 생성합니다.
이 Config는 테스트를 위해 몇 가지 확인 기능을 넣었습니다.

- Authorization 성공시 index.html 페이지를 반환, 실패시 50x.html 페이지를 반환
- Access Log에서 토큰 Payload의 sub, aud Claim의 내용 확인 가능
- Error 발생시 Error Log에서 확인 가능

```nginx
map $jwt_claim_sub $jwt_sub {
    "RS256TEST" "Authed";
    default "CantReadSub";
}
map $jwt_claim_aud $jwt_aud {
    "dddaong" "Authed";
    default "CantReadAud";
}

log_format test_log '"Request: $request\n Status: $status\n Request_URI: $request_uri\n JWT Claim sub: $jwt_sub'\n JWT Claim aud: $jwt_aud'\n;

server {
    listen 8080;
    access_log  /etc/nginx/conf.d/jwt.access.log  test_log;
    error_log  /etc/nginx/conf.d/jwt.error.log debug;

    auth_jwt "dddaong";
    auth_jwt_key_file conf.d/dddaong.jwk;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }
    error_page 400 401 403 500 502 503 504  /50x.html;
    location = /50x.html {
        auth_jwt off;
        root   /usr/share/nginx/html;
    }
}
```

##### 테스트 - Authentication Failure

이 Config가 적용된 상태에서 `http://localhost:8080`으로 접근하면,
아래와 같이 401 Unauthorized를 반환합니다.

```bash
curl -Iv http://localhost:8080

* About to connect() to localhost port 8080 (#0)
*   Trying ::1...
* Connection refused
*   Trying 127.0.0.1...
* Connected to localhost (127.0.0.1) port 8080 (#0)
> HEAD / HTTP/1.1
> User-Agent: curl/7.29.0
> Host: localhost:8080
> Accept: */*
>
< HTTP/1.1 401 Unauthorized
HTTP/1.1 401 Unauthorized
< Server: nginx/1.19.5
Server: nginx/1.19.5
< Date: Wed, 07 Apr 2021 05:41:47 GMT
Date: Wed, 07 Apr 2021 05:41:47 GMT
< Content-Type: text/html
Content-Type: text/html
< Content-Length: 179
Content-Length: 179
< Connection: keep-alive
Connection: keep-alive
< WWW-Authenticate: Bearer realm="dddaong"
WWW-Authenticate: Bearer realm="dddaong"

<
* Connection #0 to host localhost left intact
```

```bash
# tail -5 access.jwt.log && tail -1 error.jwt.log

#Access Log
Request: HEAD / HTTP/1.1
 Status: 401
 Request_URI: /
 JWT Claim sub: CantReadSub
 JWT Claim aud: CantReadAud

#Error Log
2021/04/07 16:19:01 [info] 3405#3405: *93 client 127.0.0.1 closed keepalive connection
```

##### 테스트 - Authentication Success

이제 생성한 Token을 헤더에 삽입해 테스트해보겠습니다.

테스트에 사용한 명령어는 아래와 같습니다.

```bash
curl -IvH "Authorization: Bearer `cat /etc/nginx/conf.d/dddaong.token`" \
     http://localhost:8080
```

```bash
* About to connect() to localhost port 8080 (#0)
*   Trying ::1...
* Connection refused
*   Trying 127.0.0.1...
* Connected to localhost (127.0.0.1) port 8080 (#0)
> HEAD / HTTP/1.1
> User-Agent: curl/7.29.0
> Host: localhost:8080
> Accept: */*
> Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IjAwMDEifQ.eyJzdWIiOiJSUzI1NlRFU1QiLCJhdWQiOiJkZGRhb25nIiwiaWF0IjoxNjE3Nzc4ODAwfQ.GwapbNpOrl8nLlJEo9JenBRMrGXTm03PwWVe6Jtje-lsTakG4M384-TGnmP79NOobE5KHaQ_RvyYmF65jFjoLAnMsC00jb9HBMWmSKMM7lF5BWQhDWQUEjGdZsCuB7jPI-CbdkAno0wRUKECM-wP0chH7PN0dPsQgUetwpM4uCRTRzq2_v5CT_1gcWp2w1VNjBEhXoEzabNhgMDKsVSnpdl30Odg0arYLrJ1USCYPVXnfn54CPwZwts6kooWr28m5i9DGuRT2XuzsGdrrFV_xLtFFhXcTsoNtAHeVCstzB0BG7DwJiNR4Sx1CH1pTR3tkEVZII3ik_Vx0Y7duFitJg
>
< HTTP/1.1 200 OK
HTTP/1.1 200 OK
< Server: nginx/1.19.5
Server: nginx/1.19.5
< Date: Wed, 07 Apr 2021 09:17:33 GMT
Date: Wed, 07 Apr 2021 09:17:33 GMT
< Content-Type: text/html
Content-Type: text/html
< Content-Length: 612
Content-Length: 612
< Last-Modified: Mon, 07 Dec 2020 21:10:20 GMT
Last-Modified: Mon, 07 Dec 2020 21:10:20 GMT
< Connection: keep-alive
Connection: keep-alive
< ETag: "5fce9a3c-264"
ETag: "5fce9a3c-264"
< Accept-Ranges: bytes
Accept-Ranges: bytes

<
```

```bash
#tail -5 access.jwt.log && tail -2 error.jwt.log

#Access Log
Status: 200
 Request_URI: /
 JWT Claim sub: Authed
 JWT Claim aud: Authed

#Error Log
2021/04/07 18:17:33 [info] 3403#3403: *109 client 127.0.0.1 closed keepalive connection
2021/04/07 18:18:16 [info] 3405#3405: *110 client 127.0.0.1 closed keepalive connection
```

##### Trouble shooting

- 500 Internal Server Error를 반환하는 경우
  일반적으로 JWK의 형식이 맞지 않는 문제인 경우가 많습니다.
  Access Log에서 Header와 Payload를 읽을 수는 있으나,
  아래와 같이 Error Log에서 Invalid JWK set 이라는 로그를 남깁니다.

```bash
  Request: HEAD / HTTP/1.1
   Status: 500
   Request_URI: /
   JWT Claim sub: Authed
   JWT Claim aud: Authed
  
  2021/04/07 18:25:26 [info] 3405#3405: *114 invalid JWK set, client: 127.0.0.1, server: , request: "HEAD / HTTP/1.1", host: "localhost:8080"
```

- JWT에 오류가 있거나 변조되면
  401 Unauthorized를 반환하며, Access Log에서 Claim을 대조한 결과를 읽어오지 못합니다.
  Error 로그에서 상세 내용을 확인할 수 있습니다.

```bash
   Status: 401
   Request_URI: /
   JWT Claim sub: CantReadSub
   JWT Claim aud: CantReadAud
  
  2021/04/07 18:22:03 [info] 3402#3402: *111 invalid JWT header while logging request, client: 127.0.0.1, server: , request: "HEAD / HTTP/1.1", host: "localhost:8080"
```

이 테스트 내용을 바탕으로 NGINX Plus Ingress Controller에서 JWT 테스트를 진행할 예정입니다.
