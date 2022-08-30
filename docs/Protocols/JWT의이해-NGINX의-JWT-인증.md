---
title: "JWT - JWT Concept"
author: DDAONG
date:   2021-04-06 14:10:00 +0900
category: "JWT"
layout: post
---


## JWT의 이해

NGINX JWT 테스트에 앞서 JWT에 대해 간략히 정리한 내용입니다.

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
- 일반적으로 `Authorization: Bearer <token>` 형태로 삽입됩니다.
- 지원하는 알고리즘을 [jwt.io Libraries](https://jwt.io/#libraries-io) 에서 확인할 수 있습니다.

------

### JWK / JWKS

JWK는 JSON Web Key의 약자로, JSON형식으로 표현된 Key Pair 또는 Key를 의미합니다.
JWKS는 이 JWK의 Set를 의미하며, 반드시 하나 이상의 JWK 배열[keys]를 멤버로 갖습니다.

> jwk에 대한 자세한 내용은 [jwks의 구조](https://docs.aws.amazon.com/ko_kr/cognito/latest/developerguide/amazon-cognito-user-pools-using-tokens-verifying-a-jwt.html)에서 확인할 수 있습니다.
> jwks에 대한 자세한 내용은 [Auth0.com JSON Web Key Sets](https://auth0.com/docs/tokens/json-web-tokens/json-web-key-sets)에서 확인할 수 있습니다.
> 표준 문서 링크 : [RFC-7517 JWT Specification](https://tools.ietf.org/html/rfc7517)

요약하자면, JWK와 JWKS의 차이는 JWKS에는 여러개의 JWK가 포함될 수 있다는 점입니다.

아래는 JWK, JWKS의 차이입니다.
JWKS의 최상단에 "Keys" Field 아래 배열의 형태로 JWK가 삽입된 형태를 눈으로 확인할 수 있습니다.

```json
### JWK (RSA Public Key)
{
    "kty": "RSA",
    "e": "AQAB",
    "use": "sig",
    "kid": "0001",
    "alg": "RS256",
    "n": "oS4lA_ePN4bmcaAM5AFnGVtzMmBBXoYxVGgNEtbhotDMkiIfhilRSH_IVktaDwdorEpQZxg66gwV0y5hZ4Nqo3wWxRVWKoVmdY0xQyPCtIHOs40FYVoc85vBU4giQnl_XqNHVnGOIZz7hiKgzzBBWiLFqgajzgvYN6QZBT0rb8OMj2u-iKQra9sHhlXBgg6W3YUE8BoPiDX2ehI35zXbsf4rI1IYeJvzEFLVcv6kHNJ7vSO0qDmVY4_m-0Kdi0KmJ_XZSNs7fRFwp1MYui9SXs-rZxrmHHES3xuTutJ5sJD6J2LgqkA8NgqhiTva-Dfyj08KVEj5RH1Af4F-bo6aBQ"
}
```

```json
### JWKS (RSA Public Key)
{ "keys":
[{
    "kty":"RSA",
    "e":"AQAB",
    "use":"sig",
    "kid":"0001",
    "alg":"RS256",
    "n":"oS4lA_ePN4bmcaAM5AFnGVtzMmBBXoYxVGgNEtbhotDMkiIfhilRSH_IVktaDwdorEpQZxg66gwV0y5hZ4Nqo3wWxRVWKoVmdY0xQyPCtIHOs40FYVoc85vBU4giQnl_XqNHVnGOIZz7hiKgzzBBWiLFqgajzgvYN6QZBT0rb8OMj2u-iKQra9sHhlXBgg6W3YUE8BoPiDX2ehI35zXbsf4rI1IYeJvzEFLVcv6kHNJ7vSO0qDmVY4_m-0Kdi0KmJ_XZSNs7fRFwp1MYui9SXs-rZxrmHHES3xuTutJ5sJD6J2LgqkA8NgqhiTva-Dfyj08KVEj5RH1Af4F-bo6aBQ"
}]
}
```

#### JWK Fields의 의미 - RSA

| Key 이름 | 의미                                                         |
| :------: | :----------------------------------------------------------- |
|   alg    | 알고리즘                                                     |
|   kty    | 키 타입                                                      |
|   use    | 키 사용 형태 (sig: signature Verification / enc: Encryption) |
|    e     | 표준 pem exponent                                            |
|    n     | 표준 pem moduluos                                            |
|   kid    | 키 식별자                                                    |

------

#### 알고리즘

앞으로의 테스트에 사용할 알고리즘은 검색하면 나오는 기술 문서에 흔히 등장하는
HS256과 RS256입니다.

##### HS256

- HS256은 HMAC with SHA-256을 의미합니다.
- 대칭키 방식으로 하나의 Key만 존재합니다.
- 전달하려는 Message에 SHA256 적용 후 대칭 키를 사용해 암호화 합니다.
- 대칭 키를 알고 있는 주체들만 유효성을 확인할 수 있습니다.

##### RS256

- RS256은 RSA Signature with SHA-256을 의미합니다.
- 비대칭키 방식으로 Private Key, Public Key가 존재합니다.
- 전달하려는 Message에 SHA256 적용 후 Private Key로 암호화 합니다.
- 유효성 검증을 공개키로 수행하므로 누구나 유효성을 확인할 수 있습니다.
- Public Key는 일반적으로 JWT를 발급한 서버에서 JWK에 정의된 방식으로 제공하게 됩니다.
