---
title: "NGINX+ Ingress Controller - JWT 인증"
author: DDAONG
date:   2021-04-13
layout: post
category: "NGINX+ Ingress Controller"
---

## NGINX Plus Ingress Controller의 JWT 인증

NGINX Plus Ingress Controller에서 JWT 인증 관련 기능을 테스트한 내용입니다.

- NPIC의 JWT 인증은 아직 Preview Policies 상태입니다. Production 환경에서의 사용에 주의를 요합니다.

#### 테스트 환경

Helm을 사용해 NGINX Plus Ingress Controller가 설치된 상태에서 수행했습니다.

테스트용 백엔드는 NGINX Plus에서 제공하는 Cafe 서비스를 사용했습니다.
동일한 백엔드를 사용하려면 아래 YAML을 통해 서비스를 생성할 수 있습니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: coffee
spec:
  replicas: 1
  selector:
    matchLabels:
      app: coffee
  template:
    metadata:
      labels:
        app: coffee
    spec:
      containers:
      - name: coffee
        image: nginxdemos/hello:plain-text
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: coffee-svc
spec:
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
    name: http
  selector:
    app: coffee
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tea
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tea 
  template:
    metadata:
      labels:
        app: tea 
    spec:
      containers:
      - name: tea 
        image: nginxdemos/hello:plain-text
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: tea-svc
  labels:
spec:
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
    name: http
  selector:
    app: tea
```

#### Preview-policies Enable

VirtualServer에서 사용하기 위한 JWT 인증을 사용하려면 Policy를 사용해야 합니다.
단, Policy는 현재 Preview 기능으로서 기본값으로 Disabled 되어 있습니다.

아래 방법으로 enable-preview-policies Argument를 Enable로 수정해줍니다.

```yaml
#kubectl edit deployments.apps dddaong-nginx-ingress
 
 ...[ommitted]
 spec:
  template:
    spec:
      containers:
      - args:
        - -enable-preview-policies=true
      [ommitted] ...
```

#### 주의 : JWK 사용 필수 요건

- Secret Resource는 반드시 Policy와 같은 namespace에 위치해야 합니다.
- Secret Resource는 반드시  `nginx.org/jwk` type으로 지정되어야 하며,
  JWK 내용은 반드시 `jwk` key 아래 저장되어야 합니다. 그렇지 않으면 JWT Validation에 실패합니다.

------

### NPIC + JWT + RS256 테스트

Kubernetes 기본 리소스인 Secret 리소스를 JWK로 사용한 JWT 인증을 테스트합니다.
테스트의 JWT 알고리즘은 RS256을 사용했습니다.

#### 1. RSA Private Key / Public Key 생성

jwk 생성을 위해 [mkjwk: simple JSON Web Key generator](https://mkjwk.org/)를 사용합니다.

아래는 생성시 선택한 옵션입니다.

| Tab  | Key Size |  Key Use  | Algorithm |     Key ID     | Show X.509 |
| :--: | :------: | :-------: | :-------: | :------------: | :--------: |
| RSA  |   2048   | Signature |   RS256   | Specify [0001] |    Yes     |

이 사이트에서 생성된 내용을 바탕으로 JWKS Secret 리소스 및 테스트를 위한 JWT를 생성합니다.

#### 2. JWK Secret 생성

직전 과정에서 생성한 Public Key로 NGINX Ingress Controller가 요구하는 형태의 Secret 리소스를 생성합니다.

주의사항(필수요건)을 한 번 더 짚어보면...

> 반드시 Policy와 같은 namespace에 위치해야 합니다.
>
> Secret Resource는 반드시  `nginx.org/jwk` type으로 지정되어야 하며,
> JWK 내용은 반드시 `jwk` key 아래 저장되어야 합니다.

mkjwk에서 생성된 JSON 형식의 Public Key는 아래와 같습니다.

```json
{
    "kty": "RSA",
    "e": "AQAB",
    "use": "sig",
    "kid": "0001",
    "alg": "RS256",
    "n": "ttMDOhyzsRtk5GGBsz7bcIimto_vY2QXU5Vgg3hic7lXMsGVmfypF1aYlyC1a6nkuwmf1rSp5yZd0fJDg4vPWeYT8OtynGF-lzc7PQ-cuSxR-m1uc6qQsP7zvxicl-Jb81nDn2eZ8eVJor0ME0XrsnOsQfy4kPJtex8tkWMZ7DNycaxo_dxcBwqhVFJkGbFv9vzuVctEjIsozCE5XJbVsp8UYKtOIrDomDmar4XSw-JtylgUWti5XbSZmAFM_oF-Vp_wGFhbgmNATnddtoT8VRdu9ky7UyUSIuEG5q-QkfbFrlxeGqJc_2Y7OpYJ_P6jzmHOUo8exLp8EkT6B55qmw"
}
```

```json
{ "keys":
[{
    "kty": "RSA",
    "e": "AQAB",
    "use": "sig",
    "kid": "0001",
    "alg": "RS256",
    "n": "ttMDOhyzsRtk5GGBsz7bcIimto_vY2QXU5Vgg3hic7lXMsGVmfypF1aYlyC1a6nkuwmf1rSp5yZd0fJDg4vPWeYT8OtynGF-lzc7PQ-cuSxR-m1uc6qQsP7zvxicl-Jb81nDn2eZ8eVJor0ME0XrsnOsQfy4kPJtex8tkWMZ7DNycaxo_dxcBwqhVFJkGbFv9vzuVctEjIsozCE5XJbVsp8UYKtOIrDomDmar4XSw-JtylgUWti5XbSZmAFM_oF-Vp_wGFhbgmNATnddtoT8VRdu9ky7UyUSIuEG5q-QkfbFrlxeGqJc_2Y7OpYJ_P6jzmHOUo8exLp8EkT6B55qmw"
}]
}
```

이 내용을 Key Set 형태로 바꿔서 `jwk` 파일로 저장해줍니다.
필수 요건을 만족하기 위해 파일이름을 `jwk`로 지정하는 것이 좋습니다.

만일 이름을 다르게 사용하려고 한다면, 생성한 JWKS가 요건을 만족하도록 Secret 리소스의 Key 이름을 수정하는 과정이 필요합니다.

이 파일을 통해 아래 명령어를 통해 Secret 리소스를 생성합니다.

```bash
kubectl create secret generic dddaong-jwks --type=nginx.org/jwk --from-file=jwk
```

아래의 명령어로 Secret 리소스를 확인할 수 있습니다.

```bash
kubectl get secret dddaong-jwks -o yaml
```

```yaml
apiVersion: v1
data:
  jwk: eyAia2V5cyI6Clt7CiAgICAia3R5IjogIlJTQSIsCiAgICAiZSI6ICJBUUFCIiwKICAgICJ1c2UiOiAic2lnIiwKICAgICJraWQiOiAiMDAwMSIsCiAgICAiYWxnIjogIlJTMjU2IiwKICAgICJuIjogInR0TURPaHl6c1J0azVHR0JzejdiY0lpbXRvX3ZZMlFYVTVWZ2czaGljN2xYTXNHVm1meXBGMWFZbHlDMWE2bmt1d21mMXJTcDV5WmQwZkpEZzR2UFdlWVQ4T3R5bkdGLWx6YzdQUS1jdVN4Ui1tMXVjNnFRc1A3enZ4aWNsLUpiODFuRG4yZVo4ZVZKb3IwTUUwWHJzbk9zUWZ5NGtQSnRleDh0a1dNWjdETnljYXhvX2R4Y0J3cWhWRkprR2JGdjl2enVWY3RFaklzb3pDRTVYSmJWc3A4VVlLdE9JckRvbURtYXI0WFN3LUp0eWxnVVd0aTVYYlNabUFGTV9vRi1WcF93R0ZoYmdtTkFUbmRkdG9UOFZSZHU5a3k3VXlVU0l1RUc1cS1Ra2ZiRnJseGVHcUpjXzJZN09wWUpfUDZqem1IT1VvOGV4THA4RWtUNkI1NXFtdyIKfV0KfQo=
kind: Secret
metadata:
  creationTimestamp: "2021-04-13T06:55:14Z"
  managedFields:
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      f:data:
        .: {}
        f:jwk: {}
      f:type: {}
    manager: kubectl-create
    operation: Update
    time: "2021-04-13T06:55:14Z"
  name: dddaong-jwks
  namespace: default
  resourceVersion: "3335260"
  uid: bfd83b04-0ce2-4302-b53d-a9b115320b91
type: nginx.org/jwk
```

#### 3. 테스트를 위한 JWT 생성

인증하려는 JWT도 작성합니다.
참고 링크: [RSA Signed JWT 생성 방법](https://learn.akamai.com/en-us/webhelp/iot/jwt-access-control/GUID-CB17F8FF-3367-4D4B-B3FE-FDBA53A5EA02.html)

NGINX Plus에서 JWT 작성에 사용한 방법과 동일하게  jwt.io의 Debugger를 사용합니다.

1. Header와 Payload를 각각 작성해 Base64url 인코딩합니다.

```bash
HEAD=$(echo -n '{"alg":"RS256","typ":"JWT","kid":"0001"}' | base64 | sed s/\+/-/ | sed -E s/=+$//)
PAYLOAD=$(echo -n '{"sub":"RS256NPICTEST","aud":"dddaong","iat": 1618295120}' | base64 | sed s/\+/-/ | sed -E s/=+$//)
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
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IjAwMDEifQ.eyJzdWIiOiJSUzI1Nk5QSUNURVNUIiwiYXVkIjoiZGRkYW9uZyIsImlhdCI6MTYxODI5NTEyMH0.kaJmcz1YLWOjBUrqMnPR8Kzh8nqG92GFMXYmpOVkjlx7NzxNZ_6e6gv6ziR9UhiiYOICqwwzpLUN9DerumGSRCNe6m_640oA0HnS1MkgSc9Q4V9g7DT66UpM4gfDC9WJlRoigolD8cXG5V479Dun3VOvKvwmDDr4NYMPSh8w-L056Lk6lqZEZktXjIYbTaB9S4gLWhbxsrA61C97Va71dbqKqDc0TJ0oTTCLtyRx3EzsfETGil44uMowfGGVb5Y_S5vojv_dwsFkmxjBH7EbvLsvRwYcahrqtxfF6YCXTwvnppxhI9iHWBQaNF4Zkvsk1XzzhCE3NPjpWPxqjQaE8g
```

4. 테스트 과정에서는 편의를 위해 전체 토큰 스트링을 dddaong.token 이름의 별도 파일로 저장했습니다.

```bash
echo 'eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IjAwMDEifQ.eyJzdWIiOiJSUzI1Nk5QSUNURVNUIiwiYXVkIjoiZGRkYW9uZyIsImlhdCI6MTYxODI5NTEyMH0.kaJmcz1YLWOjBUrqMnPR8Kzh8nqG92GFMXYmpOVkjlx7NzxNZ_6e6gv6ziR9UhiiYOICqwwzpLUN9DerumGSRCNe6m_640oA0HnS1MkgSc9Q4V9g7DT66UpM4gfDC9WJlRoigolD8cXG5V479Dun3VOvKvwmDDr4NYMPSh8w-L056Lk6lqZEZktXjIYbTaB9S4gLWhbxsrA61C97Va71dbqKqDc0TJ0oTTCLtyRx3EzsfETGil44uMowfGGVb5Y_S5vojv_dwsFkmxjBH7EbvLsvRwYcahrqtxfF6YCXTwvnppxhI9iHWBQaNF4Zkvsk1XzzhCE3NPjpWPxqjQaE8g' > dddaong.token
```

------

#### 4-1. Ingress Resource에 연동

##### Ingress의 JWT 인증 설정 - Annotation의 사용

JWT 인증을 Ingress Resource에 연동할 때는 Annotation 형태로 정의 됩니다.

NPIC가 지원하는 Annotation은 네 가지 입니다.

- [필수] `nginx.com/jwt-key: "secret"` : JWT를 인증할 Secret 리소스의 이름을 지정합니다.

- [옵션] `nginx.com/jwt-realm: "realm"` : Realm 명을 지정합니다.

- [옵션] `nginx.com/jwt-token: "token"` : JWT의 위치를 지정합니다. 변수를 사용할 수 있습니다.

  - token : 기본값이며, Authorization 헤더에서 Bearer 토큰을 사용하는 방식입니다.
  - `$http_*name*` : JWT의 위치를 'name' 헤더로 변경합니다.
  - `$cookie_auth_token` : JWT의 위치를 auth_token이라는 이름의 cookie 값으로 사용합니다.
  - `$arg_apijwt` : JWT의 위치를 apijwt라는 이름의 Argument의 값으로 사용합니다.
- [옵션]`nginx.com/jwt-login-url: "url"` : JWT가 없거나 유효하지 않은 경우 Redirect할 URL을 지정합니다.

cookie_name 변수와 arg_name의 변수 이름은 lowercase로,  `-`는 `_`로 대체됩니다.
더 자세한 내용은 [ngx_http_core_module 모듈의 Embedded Variables](http://nginx.org/en/docs/http/ngx_http_core_module.html#variables)를 참조하면 좋습니다.

##### Ingress Resource의 Annotation 작성 샘플 (YAML)

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: cafe-ingress
  annotations:
    nginx.com/jwt-key: "cafe-jwk" 
    nginx.com/jwt-realm: "Cafe App"  
    nginx.com/jwt-token: "$cookie_auth_token"
    nginx.com/jwt-login-url: "https://login.example.com"
spec:
```

Ingress Resource 관련한 이상의 내용은 [NGINX INC Github 참고 페이지](https://github.com/nginxinc/kubernetes-ingress/blob/master/examples/jwt/README.md)에 예시와 함께 상세히 설명되어 있습니다.

##### Ingress 리소스 생성

테스트에 사용한 Ingress 리소스는 아래와 같습니다.

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: dddaong-jwt-test
  annotations:
    nginx.com/jwt-key: "dddaong-jwks"
    nginx.com/jwt-realm: "dddaong"
spec:
  rules:
  - host: auth-ing.test
    http:
      paths:
      - path: /
        backend:
          serviceName: coffee-svc
          servicePort: 80
```

#### Ingress JWT 인증 TEST 결과

NPIC의 Service 리소스의 ClusterIP로 Curl 명령어를 통해 인증 여부를 확인합니다.

- Case : Authentication Failure

```bash
curl -IvH 'host:auth-ing.test' http://10.97.46.125                   * About to connect() to 10.97.46.125 port 80 (#0)
*   Trying 10.97.46.125...
* Connected to 10.97.46.125 (10.97.46.125) port 80 (#0)
> HEAD / HTTP/1.1
> User-Agent: curl/7.29.0
> Accept: */*
> host:auth-ing.test
>
< HTTP/1.1 401 Unauthorized
HTTP/1.1 401 Unauthorized
< Server: nginx/1.19.5
Server: nginx/1.19.5
< Date: Tue, 13 Apr 2021 07:14:10 GMT
Date: Tue, 13 Apr 2021 07:14:10 GMT
< Content-Type: text/html
Content-Type: text/html
< Content-Length: 179
Content-Length: 179
< Connection: keep-alive
Connection: keep-alive
< WWW-Authenticate: Bearer realm="dddaong"
WWW-Authenticate: Bearer realm="dddaong"

<
* Connection #0 to host 10.97.46.125 left intact
```

- Case : Authentication Success

```bash
# curl -IvH 'host:auth-ing.test' -H "Authorization: Bearer `cat dddaong.token`" http://10.97.46.125
* About to connect() to 10.97.46.125 port 80 (#0)
*   Trying 10.97.46.125...
* Connected to 10.97.46.125 (10.97.46.125) port 80 (#0)
> HEAD / HTTP/1.1
> User-Agent: curl/7.29.0
> Accept: */*
> host:auth-ing.test
> Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IjAwMDEifQ.eyJzdWIiOiJSUzI1Nk5QSUNURVNUIiwiYXVkIjoiZGRkYW9uZyIsImlhdCI6MTYxODI5NTEyMH0.kaJmcz1YLWOjBUrqMnPR8Kzh8nqG92GFMXYmpOVkjlx7NzxNZ_6e6gv6ziR9UhiiYOICqwwzpLUN9DerumGSRCNe6m_640oA0HnS1MkgSc9Q4V9g7DT66UpM4gfDC9WJlRoigolD8cXG5V479Dun3VOvKvwmDDr4NYMPSh8w-L056Lk6lqZEZktXjIYbTaB9S4gLWhbxsrA61C97Va71dbqKqDc0TJ0oTTCLtyRx3EzsfETGil44uMowfGGVb5Y_S5vojv_dwsFkmxjBH7EbvLsvRwYcahrqtxfF6YCXTwvnppxhI9iHWBQaNF4Zkvsk1XzzhCE3NPjpWPxqjQaE8g
>
< HTTP/1.1 200 OK
HTTP/1.1 200 OK
< Server: nginx/1.19.5
Server: nginx/1.19.5
< Date: Tue, 13 Apr 2021 07:15:00 GMT
Date: Tue, 13 Apr 2021 07:15:00 GMT
< Content-Type: text/plain
Content-Type: text/plain
< Content-Length: 157
Content-Length: 157
< Connection: keep-alive
Connection: keep-alive
< Expires: Tue, 13 Apr 2021 07:14:59 GMT
Expires: Tue, 13 Apr 2021 07:14:59 GMT
< Cache-Control: no-cache
Cache-Control: no-cache

<
* Connection #0 to host 10.97.46.125 left intact
```

------

#### 4-2. VirtualServer Crd에 연동

##### VirtualServer의 JWT 인증 설정 - Policy Crd 사용

JWT 인증을 VirtualServer Resource에 연동할 때는 별도의 Policy 리소스로 정의 됩니다.
JWT의 위치 변경 (Header, Cookie, Argument) 역시 Policy에서 설정합니다.

아래는 YAML 형식의 Policy Crd 샘플입니다.

```yaml
apiVersion: k8s.nginx.org/v1
kind: Policy
metadata:
  name: jwk-auth
spec:
  jwt:
    secret: dddaong-jwk-secret  # Secret 리소스 이름을 지정합니다.
    realm: "myrealm"    # Realm 이름을 지정합니다.
    token: $http_token  # Token 위치를 지정합니다.
```

Ingress 리소스와 마찬가지로 Token 위치를 별도로 지정할 수 있습니다.
지정하지 않으면 기본값을 사용하며, 기본값은 표준 헤더 `Authorization: Bearer <token>` 입니다.

```yaml
spec:
  jwt:
    secret: jwk-secret
    realm: "myrealm"
    token: $cookie_cookiename #$arg_argname 또는 $http_headername
```

다른 경우와 마찬가지로 변수 이름은 lowercase로,  `-`는 `_`로 대체됩니다.
더 자세한 내용은 [ngx_http_core_module 모듈의 Embedded Variables](http://nginx.org/en/docs/http/ngx_http_core_module.html#variables)를 참조하면 좋습니다.

##### Policy 생성

인증 테스트에 사용한 Policy Crd는 아래와 같습니다.

```yaml
apiVersion: k8s.nginx.org/v1
kind: Policy
metadata:
  name: dddaong-jwk-auth
spec:
  jwt:
    secret: dddaong-jwks
    realm: "dddaong"
```

Policy를 아래와 같이 VirtualServer에 연동해줍니다.

```yaml
apiVersion: k8s.nginx.org/v1
kind: VirtualServer
metadata:
  name: dddaong-jwt-test
spec:
  host: auth.test
  policies: 
  - name: dddaong-jwk-auth
  upstreams: 
  - name: coffee
    service: coffee-svc
    port: 80
  routes: 
  - path: /
    action:
      pass: coffee
```

VirtualServer의 upstream과 routes 부분은 테스트할 Application에 맞게 수정해서 사용하시면 됩니다.

#### VirtualServer JWT 인증 TEST 결과

NPIC의 Service 리소스의 ClusterIP로 Curl 명령어를 통해 인증 여부를 확인합니다.

- Case : Authentication Failure

```bash
[root@kube-master jwt]# curl -IvH 'host:auth.test' http://10.97.46.125
* About to connect() to 10.97.46.125 port 80 (#0)
*   Trying 10.97.46.125...
* Connected to 10.97.46.125 (10.97.46.125) port 80 (#0)
> HEAD / HTTP/1.1
> User-Agent: curl/7.29.0
> Accept: */*
> host:auth.test
>
< HTTP/1.1 401 Unauthorized
HTTP/1.1 401 Unauthorized
< Server: nginx/1.19.5
Server: nginx/1.19.5
< Date: Tue, 13 Apr 2021 07:04:59 GMT
Date: Tue, 13 Apr 2021 07:04:59 GMT
< Content-Type: text/html
Content-Type: text/html
< Content-Length: 179
Content-Length: 179
< Connection: keep-alive
Connection: keep-alive
< WWW-Authenticate: Bearer realm="dddaong"
WWW-Authenticate: Bearer realm="dddaong"

<
* Connection #0 to host 10.97.46.125 left intact
```

- Case : Authentication Success

```bash
# curl -IvH 'host:auth.test' -H "Authorization: Bearer `cat dddaong.token`" http://10.97.46.125
* About to connect() to 10.97.46.125 port 80 (#0)
*   Trying 10.97.46.125...
* Connected to 10.97.46.125 (10.97.46.125) port 80 (#0)
> HEAD / HTTP/1.1
> User-Agent: curl/7.29.0
> Accept: */*
> host:auth.test
> Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IjAwMDEifQ.eyJzdWIiOiJSUzI1Nk5QSUNURVNUIiwiYXVkIjoiZGRkYW9uZyIsImlhdCI6MTYxODI5NTEyMH0.kaJmcz1YLWOjBUrqMnPR8Kzh8nqG92GFMXYmpOVkjlx7NzxNZ_6e6gv6ziR9UhiiYOICqwwzpLUN9DerumGSRCNe6m_640oA0HnS1MkgSc9Q4V9g7DT66UpM4gfDC9WJlRoigolD8cXG5V479Dun3VOvKvwmDDr4NYMPSh8w-L056Lk6lqZEZktXjIYbTaB9S4gLWhbxsrA61C97Va71dbqKqDc0TJ0oTTCLtyRx3EzsfETGil44uMowfGGVb5Y_S5vojv_dwsFkmxjBH7EbvLsvRwYcahrqtxfF6YCXTwvnppxhI9iHWBQaNF4Zkvsk1XzzhCE3NPjpWPxqjQaE8g
>
< HTTP/1.1 200 OK
HTTP/1.1 200 OK
< Server: nginx/1.19.5
Server: nginx/1.19.5
< Date: Tue, 13 Apr 2021 07:06:11 GMT
Date: Tue, 13 Apr 2021 07:06:11 GMT
< Content-Type: text/plain
Content-Type: text/plain
< Content-Length: 157
Content-Length: 157
< Connection: keep-alive
Connection: keep-alive
< Expires: Tue, 13 Apr 2021 07:06:10 GMT
Expires: Tue, 13 Apr 2021 07:06:10 GMT
< Cache-Control: no-cache
Cache-Control: no-cache

<
* Connection #0 to host 10.97.46.125 left intact
```

별도 참조... - [HS256 JWT 생성기](http://jwtbuilder.jamiekurtz.com/)
