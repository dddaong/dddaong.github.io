---
layout: post
title:  "JWT의 이해, NGINX의 JWT 인증"
date:   2021-04-06 14:10:00 +0900
categories: NGINX JWT JWK
---

NGINX Plus Ingress Controller에서 JWT 인증 관련 기능을 테스트한 내용입니다.

## JWT의 이해, NGINX의 JWT 인증

### JWT

JWT는 JSON Web Token의 약자로, HTTP Request 헤더에 Token 삽입되어, 클라이언트와 서버 간 인증을 수행합니다. 

[JWT Introduction](https://jwt.io/introduction)



#### JWT의 구조

토큰은 헤더, 페이로드, 서명으로 세 부분으로 구성되며, Base64url 인코딩을 사용합니다.

- 헤더 : 
  일반적으로 토큰의 타입과 해시 암호화 알고리즘이 명시됩니다.

  ``` json
  {
    "alg": "HS256",
    "typ": "JWT"
  }
  ```

  

- 페이로드 : 
  클레임(Claim) 정보를 포함합니다. 
  클레임은 Name-Value 쌍으로 이뤄지며, 여러 개가 존재할 수 있습니다. 
  클레임이 많을 수록 토큰 사이즈가 커지며, 트래픽 크기에 영향을 줄 수 있습니다.

  ```json
  {
    "sub": "1234567890",
    "name": "John Doe",
    "admin": true
  }
  ```

  클레임에는 Registered, Public, Private의 세 종류가 있습니다.

  - Registered Claim : 사전 설정된 클레임으로서, iss, exp, sub 등이 있습니다. 필수는 아닙니다.
  - Public Claim : Claim 이름은 사용자가 마음대로 설정할 수 있습니다.
    하지만 Claim명의 충돌을 피하기 위해 지정한 이름을 준수하거나, Collision-Resistant Namespace를 사용할 것을 요구하는, IANA에서 지정한 Claim 이름들 입니다.
    [IANA JWT Claim Registry](https://www.iana.org/assignments/jwt/jwt.xhtml)
  - Private Claim : 인증/피인증자 간에 협의를 통해 사용하는 클레임을 의미합니다.

  

- 서명 (Signature) : 
  서명 부분은 [Base64url 인코딩된 Header]+"."+[Base64url 인코딩된 Payload]을 
  헤더에 명시된 암호화 알고리즘 및  [Secret]으로 암호화한 내용입니다.

  따라서 메시지가 변조되지는 않았는지 확인이 가능하며, 
  Private Key로 서명된 토큰의 경우, 송신자가 누구인지까지 확인할 수 있습니다.



#### JWT 특징

- JWT는 사용자 인증에 필요한 모든 정보를 토큰이 갖고 있기 때문에 별도의 인증 저장이 필요없다는 점이 장점입니다.
- Header와 Payload는 누구나 볼 수 있는 정보가 됩니다.
  따라서 반드시 HTTPS를 사용할 것이 권고되며, 비밀 정보는 Payload Claim에 작성하지 않는 것이 좋습니다.
- 일반적으로 `Authorization: Bearer <token> ` 형태로 삽입됩니다.
- 해싱 알고리즘으로 HS256을 대체로 많이 사용합니다. 
  HS256은 HMAC SHA256을 말하며, 지원하는 알고리즘을 프레임워크? 또는 언어에 따라 확인할 수 있습니다. [jwt.io Libraries](https://jwt.io/#libraries-io)







### NGINX의 JWT Validation

NGINX Plus는 R10 버전부터 Native JWT Validation을 지원하며, 
NGINX OSS에서는 [lua-resty-openidc](https://github.com/zmartzone/lua-resty-openidc) 등 외부 모듈 사용이 필요합니다.

NGINX Plus 는 표준에 정의된  `HS*xxx*`, `RS*xxx*`, and `ES*xxx*` [signature algorithms](https://nginx.org/en/docs/http/ngx_http_auth_jwt_module.html)을 지원합니다.



#### Sample: NGINX Plus JWT Authentication Config
아래는 [Nginx Plus r17 Release](https://www.nginx.com/blog/nginx-plus-r17-released/?_ga=2.18265401.1678908472.1617607349-1458027870.1616747210#r17-openid)에서 발췌한 NGINX Plus Config 예시입니다.

이 Config는 `auth_jwt_key_file`를 사용하지 않고 `auth_jwt_key_request`를 사용하는 방식으로 
JWK를 Remote Location에서 가져오도록 합니다.

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

1. 대부분의 idP는 (최신의)Key Set을 가져올 수 있는 URL을 제공합니다.

2. N+는 Client Request를 수신하면 `auth_jet_key_request` Directive에 의해 idP의 고정 URL로 Subrequest를 보냅니다.
3. `/_jwk_uri` 블록의 Subrequest에 대한 응답으로 Key Set을 수신합니다.
   `/_jwk_uri` Location 블록은 `internal` Directive를 사용해 외부에서는 접근할 수 없도록 합니다.
4. 수신한 Key Set을 `proxy_cache` Directive로  `jwk keys_zone`에 캐시하여, Validation Overhead를 줄여줄 수 있습니다.



5. JWT는 일반적으로 `Authorization` 헤더에 Bearer Token으로 인증 요청되지만, Cookie, Query String으로 사용될 수도 있기 때문에, `auth_jwt` Directive에서 세부 설정이 가능하도록 되어 있습니다.

   ```nginx
   auth_jwt string [token=$variable] | off;
   ```

   

   예를 들어 아래와 같이 Query String을 통해 Token 인증 요청을 사용하려면

   ```
   $ curl http://192.168.1.1/products/widget1?apijwt=xxxxxxxxxxxxxxxxxxxxx...
   ```

   string을 realm 명, 토큰을 apijwt Argument의 Value를 참조하도록 합니다.

   ```nginx
   auth_jwt "API" token=$arg_apijwt;
   ```

   참고로, $arg_name 과 $cookie_name 변수는 [ngx_http_core_module 모듈의 Embedded Variables](http://nginx.org/en/docs/http/ngx_http_core_module.html#variables)를 참조하면 됩니다. 



이 내용을 바탕으로 NGINX Plus Ingress Controller에서 JWT 테스트를 진행할 예정입니다.
