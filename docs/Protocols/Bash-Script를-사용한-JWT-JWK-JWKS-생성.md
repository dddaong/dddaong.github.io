---
title: "JWT - Bash Script를 사용한 JWT, JWK, JWKS 생성"
author: DDAONG
date:   2021-04-20
category: JWT
layout: post
---


## Bash Shell에서 JWK, JWT 생성

Bash Shell에서 JWT, JWK, JWKS를 생성하는 과정에 대한 내용입니다.

NGINX Plus의 JWT Validation 과정에서 Subrequest 방식을 구현하는데 필요한 환경 구성을 목적으로,
Bash Shell에서  JWK JWT 생성 테스트 과정을 기록했습니다.

#### 진행 순서

1. Bash Shell에서 PEM Private Key, Public Key 생성
2. 생성한 Key들을 기반으로  JWK를 생성
3. JWT 생성
4. (다음 포스트) Mock-Json-Server의 특정 URI에서  JWK 반환 테스트
5. (다음 포스트) NGINX 반영

- CentOS7 환경에서 RS256 알고리즘을 사용했습니다.

#### Packages required

```bash
yum -y install vim openssl epel-release && yum -y install jq
```

- vim : xxd 명령어를 지원합니다.
- openssl : Private, Public Key를 생성하고 Encode, Decode를 지원합니다.
- epel-release : jq 설치를 위해 필요합니다.
- jq : JSON 형식 Validation을 수행합니다.

------

### 전체 쉘 스크립트 요약

```bash
## Generate PEM Keys
varO='dddaong'
varOU='dddaong'
varCN='dddaong.test'

openssl genrsa -out $varCN.private.pem 1024 > /dev/null 2>&1
openssl rsa -in $varCN.private.pem -out $varCN.public.pem -pubout > /dev/null 2>&1
PEM=$( cat ${varCN}.private.pem )

## Extract Modulus From Public Key
MOD_HEX=$(openssl rsa -pubin -inform PEM -text -noout < $varCN.public.pem | grep -v -e 'Key' -e 'Modulus' -e 'Exponent' | tr -d '\n' | tr -d ' ' | tr -d ':')
MOD=$(echo $MOD_HEX | xxd -r -p | base64 | tr -d '=' | tr '/+' '_-' | tr -d '\n')


## Extract Exponent From Public Key
EXP_HEX=$(openssl rsa -pubin -inform PEM -text -noout < $varCN.public.pem | grep 'Exponent' | cut -f 2 -d '(' | cut -f 1 -d ')' )
EXP=$(printf "%06X\n" $EXP_HEX | xxd -r -p | base64 | tr -d '=' | tr '/+' '_-' | tr -d '\n')


## Generate JWK and JWKS in files
KID=$(printf "%04d" $(expr $RANDOM % 1000))
JWK='{"kty":"RSA","e":"'"${EXP}"'","use":"sig","kid":"'"${KID}"'","alg":"RS256","n":"'"${MOD}"'"}'
echo $JWK | jq > $varCN.pubkey.jwk

JWKS='{"keys":['"${JWK}"']}'
echo $JWKS | jq > $varCN.pubkey.jwks


## Generate JWT - Header and Payload
NOW=$( date +%s )
IAT="${NOW}"
EXPIRE=$((${NOW} + 3600))

HEADER_RAW='{"alg":"RS256","typ":"JWT","kid":"'"${KID}"'"}'
HEADER=$( echo -n "${HEADER_RAW}" | openssl base64 | tr -d '=' | tr '/+' '_-' | tr -d '\n' )

PAYLOAD_RAW='{"iat":'"${IAT}"',"exp":'"${EXPIRE}"',"aud":"'"${varOU}"'","sub":"'"${varCN}"'"}'
PAYLOAD=$( echo -n "${PAYLOAD_RAW}" | openssl base64 | tr -d '=' | tr '/+' '_-' | tr -d '\n' )
HEADER_PAYLOAD="${HEADER}"."${PAYLOAD}"


## Generate JWT - Signature
SIGNATURE=$( openssl dgst -sha256 -sign <(echo -n "${PEM}") <(echo -n "${HEADER_PAYLOAD}") | openssl base64 | tr -d '=' | tr '/+' '_-' | tr -d '\n' )


## Save JWT in file
JWT="${HEADER_PAYLOAD}"."${SIGNATURE}"
echo ${JWT} > $varCN.token
```

위 스크립트가 오류 없이 잘 진행되었다면, 아래 5개 파일이 생성됩니다.

- PEM Keys : dddaong.test.private.pem,  dddaong.test.public.pem
- JWK, JWKS : dddaong.test.pub.jwk, dddaong.test.pub.jwks
- JWT : dddaong.test.token

------

### Base Knowledges

#### Based Knowledge : 필수 필드

RFC 문서에 따르면, Public JWK에는 Modulus와 Exponent 필드가 필수로 들어가야 한다고 합니다.
[RFC7518: Parameters a Public Key MUST Have](https://tools.ietf.org/html/rfc7518#section-6.3.1)

#### Based Knowledge : JWK 형식과 PEM 형식의 동일성 검증 - Public Key

아래는 mkjwk.org에서 생성한 JWK Public Key 및 Public Key (x.509 PEM Format) 입니다.

- JWK Public Key

```json
{
 "kty": "RSA",
 "e": "AQAB",
 "use": "sig",
 "kid": "0985",
 "alg": "RS256",
 "n": "5v2V21Q3uJOylP6RAoicdUxIuHkc9xBV0KH3H6MycWVwu9XjGdzTTBTGFFTv6tmXeM2wUjiya9b0F0Q5rX-SdzCZnE5XX_NEMr8F9fTzl1NzYO8QV79TcqEfBc3PWHogaIjF7HUn1dOYi-T1U82LRi5W0EXcZUUt43gteXvr1C00gDuIMl9DPEYyJneIxKWpWoZrO8rCPrdQ0L1iE3-fbmiwCg0yproRVoJ-dWuMcAVO5KR-Zyvs9BWc0njMEkOURHGUpyv3rcguzPyEteetuX-Iy6bqAYKncUGP2Gh_JlLIwqeBW9h-Hbs8wrhK21Dj4WK6sJIO9oTxbi0FYcJoKw"
}
```

   필수 필드인 e, n은 Non-Human-Readable 필드로, Base64url Encode 된 상태입니다.
   kty, use, kid, alg 필드는 Human-Readable 필드로서, RS256을 알고리즘 사용하겠다고 선언한다면, 고정값을 갖거나 유저가 지정한 값을 갖습니다.

 `"e": "AQAB"`는 Exponent 필드이며, `01 00 01 (65537)`을 의미합니다. 아래 명령어로 확인할 수 있습니다.
이 명령어는 문자열 `"AQAB"`를 base64 디코드, Hexdump 후에 10진수로 변환하는 과정입니다.

```bash
echo $(( 16#$(echo "AQAB" | base64 -d | xxd -p) ))
65537
```

`"n": "Some hash value...."`는 Modulus 필드이며, 위 명령어에서 생략된 부분이 추가됩니다.

```bash
MODULUS_RAW=5v2V21Q3uJOylP6RAoicdUxIuHkc9xBV0KH3H6MycWVwu9XjGdzTTBTGFFTv6tmXeM2wUjiya9b0F0Q5rX-SdzCZnE5XX_NEMr8F9fTzl1NzYO8QV79TcqEfBc3PWHogaIjF7HUn1dOYi-T1U82LRi5W0EXcZUUt43gteXvr1C00gDuIMl9DPEYyJneIxKWpWoZrO8rCPrdQ0L1iE3-fbmiwCg0yproRVoJ-dWuMcAVO5KR-Zyvs9BWc0njMEkOURHGUpyv3rcguzPyEteetuX-Iy6bqAYKncUGP2Gh_JlLIwqeBW9h-Hbs8wrhK21Dj4WK6sJIO9oTxbi0FYcJoKw

if [ `expr length ${MODULUS_RAW} % 4` == 2 ]; then MODULUS=${MODULUS_RAW}'=='; fi
if [ `expr length ${MODULUS_RAW} % 4` == 3 ]; then MODULUS=${MODULUS_RAW}'='; fi

echo $MODULUS | tr '_-' '/+' | base64 -d | xxd -p -u
E6FD95DB5437B893B294FE9102889C754C48B8791CF71055D0A1F71FA332
716570BBD5E319DCD34C14C61454EFEAD99778CDB05238B26BD6F4174439
AD7F927730999C4E575FF34432BF05F5F4F397537360EF1057BF5372A11F
05CDCF587A206888C5EC7527D5D3988BE4F553CD8B462E56D045DC65452D
E3782D797BEBD42D34803B88325F433C4632267788C4A5A95A866B3BCAC2
3EB750D0BD62137F9F6E68B00A0D32A6BA1156827E756B8C70054EE4A47E
672BECF4159CD278CC124394447194A72BF7ADC82ECCFC84B5E7ADB97F88
CBA6EA0182A771418FD8687F2652C8C2A7815BD87E1DBB3CC2B84ADB50E3
E162BAB0920EF684F16E2D0561C2682B
```

- Public Key (x.509 PEM Format)

```text
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA5v2V21Q3uJOylP6RAoic
dUxIuHkc9xBV0KH3H6MycWVwu9XjGdzTTBTGFFTv6tmXeM2wUjiya9b0F0Q5rX+S
dzCZnE5XX/NEMr8F9fTzl1NzYO8QV79TcqEfBc3PWHogaIjF7HUn1dOYi+T1U82L
Ri5W0EXcZUUt43gteXvr1C00gDuIMl9DPEYyJneIxKWpWoZrO8rCPrdQ0L1iE3+f
bmiwCg0yproRVoJ+dWuMcAVO5KR+Zyvs9BWc0njMEkOURHGUpyv3rcguzPyEteet
uX+Iy6bqAYKncUGP2Gh/JlLIwqeBW9h+Hbs8wrhK21Dj4WK6sJIO9oTxbi0FYcJo
KwIDAQAB
-----END PUBLIC KEY-----
```

예시 PEM Public Key의 시작, 끝 마커를 제외한 나머지 문자열을 사용합니다.

```bash
PEM_RAW='MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA5v2V21Q3uJOylP6RAoic
dUxIuHkc9xBV0KH3H6MycWVwu9XjGdzTTBTGFFTv6tmXeM2wUjiya9b0F0Q5rX+S
dzCZnE5XX/NEMr8F9fTzl1NzYO8QV79TcqEfBc3PWHogaIjF7HUn1dOYi+T1U82L
Ri5W0EXcZUUt43gteXvr1C00gDuIMl9DPEYyJneIxKWpWoZrO8rCPrdQ0L1iE3+f
bmiwCg0yproRVoJ+dWuMcAVO5KR+Zyvs9BWc0njMEkOURHGUpyv3rcguzPyEteet
uX+Iy6bqAYKncUGP2Gh/JlLIwqeBW9h+Hbs8wrhK21Dj4WK6sJIO9oTxbi0FYcJo
KwIDAQAB'


echo $PEM_RAW | tr -d " " | base64 -d | openssl asn1parse -inform DER -i -strparse 19
    0:d=0  hl=4 l= 266 cons: SEQUENCE
    4:d=1  hl=4 l= 257 prim:  INTEGER           :E6FD95DB5437B893B294FE9102889C754C48B8791CF71055D0A1F71FA332716570BBD5E319DCD34C14C61454EFEAD99778CDB05238B26BD6F4174439AD7F927730999C4E575FF34432BF05F5F4F397537360EF1057BF5372A11F05CDCF587A206888C5EC7527D5D3988BE4F553CD8B462E56D045DC65452DE3782D797BEBD42D34803B88325F433C4632267788C4A5A95A866B3BCAC23EB750D0BD62137F9F6E68B00A0D32A6BA1156827E756B8C70054EE4A47E672BECF4159CD278CC124394447194A72BF7ADC82ECCFC84B5E7ADB97F88CBA6EA0182A771418FD8687F2652C8C2A7815BD87E1DBB3CC2B84ADB50E3E162BAB0920EF684F16E2D0561C2682B
  265:d=1  hl=2 l=   3 prim:  INTEGER           :010001
```

base64 디코딩 후, ASN.1 형식으로 파싱해주면 위와 같이 Hex 형식의 Modulus 값, 그리고 `01 00 01` 의 Exponent 값을 확인할 수 있습니다.

- JWK 작성시 동일성 검증 내용을 바탕으로 필수 필드를 Public Key에서 추출 및 작성하고, 나머지 필드를 수기 작성해주면 되겠습니다.

#### Based Knowledge: PEM에서 추출한 Modulus (Hex Format)를 Encode하기

```bash
# 추출한 Hex 값의 변수 선언
PEM='E6FD95DB5437B893B294FE9102889C754C48B8791CF71055D0A1F71FA332716570BBD5E319DCD34C14C61454EFEAD99778CDB05238B26BD6F4174439AD7F927730999C4E575FF34432BF05F5F4F397537360EF1057BF5372A11F05CDCF587A206888C5EC7527D5D3988BE4F553CD8B462E56D045DC65452DE3782D797BEBD42D34803B88325F433C4632267788C4A5A95A866B3BCAC23EB750D0BD62137F9F6E68B00A0D32A6BA1156827E756B8C70054EE4A47E672BECF4159CD278CC124394447194A72BF7ADC82ECCFC84B5E7ADB97F88CBA6EA0182A771418FD8687F2652C8C2A7815BD87E1DBB3CC2B84ADB50E3E162BAB0920EF684F16E2D0561C2682B'

# Base64 Encode
echo $PEM | xxd -r -p | base64 | tr -d '=' | tr '/+' '_-' | tr -d '\n'
5v2V21Q3uJOylP6RAoicdUxIuHkc9xBV0KH3H6MycWVwu9XjGdzTTBTGFFTv6tmXeM2wUjiya9b0F0Q5rX-SdzCZnE5XX_NEMr8F9fTzl1NzYO8QV79TcqEfBc3PWHogaIjF7HUn1dOYi-T1U82LRi5W0EXcZUUt43gteXvr1C00gDuIMl9DPEYyJneIxKWpWoZrO8rCPrdQ0L1iE3-fbmiwCg0yproRVoJ-dWuMcAVO5KR-Zyvs9BWc0njMEkOURHGUpyv3rcguzPyEteetuX-Iy6bqAYKncUGP2Gh_JlLIwqeBW9h-Hbs8wrhK21Dj4WK6sJIO9oTxbi0FYcJoKw
```

이상의 Base Knowledge들을 사용해 PEM Public Key에서 Modulus, Exponent를 추출해 JWK 생성을 수행합니다.

------

### JWK 생성

이 과정에서는, Public Key(PEM 파일, X.509형식)에서 JWK 작성에 필요한 필드값을 구해 JWK 형식으로 작성,
jq를 사용해 Validation 및 출력을 수행합니다.

#### Openssl을 사용해 PEM Public / Private Key 생성

```bash
varO='dddaong'
varOU='dddaong'
varCN='dddaong.test'

openssl genrsa -out $varCN.private.pem 1024 > /dev/null 2>&1
openssl rsa -in $varCN.private.pem -out $varCN.public.pem -pubout > /dev/null 2>&1
PEM=$( cat ${varCN}.private.pem )
```

#### Extract Modulus (in Hex) and base64url Encode

아래 openssl 명령어를 실행해 Modulus와 Exponent를 추출합니다.
결과는 아래와 같이 Hex 형식으로 나타납니다.

```bash
openssl rsa -pubin -inform PEM -text -noout < $varCN.public.pem
Public-Key: (1024 bit)
Modulus:
    00:bc:d1:11:e3:58:48:61:e9:8d:5f:18:66:b1:64:
    5c:5f:3d:15:26:1d:d0:e6:a3:f2:73:bf:58:ca:e3:
    63:73:6c:49:5e:62:2c:db:7b:67:d1:09:ca:ba:76:
    ea:2b:33:71:f6:e1:9b:d5:6d:84:59:71:97:04:c6:
    ff:bc:dd:d0:1b:ab:e5:20:66:0c:40:b6:4a:e9:68:
    27:c4:75:cc:84:2f:bd:d3:d0:b9:e2:7f:ea:97:30:
    c5:a9:f2:af:03:73:f8:07:57:21:ea:74:c3:9d:3d:
    b6:a2:71:d9:01:66:c1:57:72:9a:4c:ab:08:ed:5c:
    7f:4a:0f:e7:e4:50:5d:48:13
Exponent: 65537 (0x10001)
```

Modulus 값을 HEX만 정제해서 Base64url 인코딩 합니다.

```bash
MOD_HEX=$(openssl rsa -pubin -inform PEM -text -noout < $varCN.public.pem | grep -v -e 'Key' -e 'Modulus' -e 'Exponent' | tr -d '\n' | tr -d ' ' | tr -d ':')
MOD=$(echo $MOD_HEX | xxd -r -p | base64 | tr -d '=' | tr '/+' '_-' | tr -d '\n')

echo $MOD
```

#### Extract Exponent (in 6 Digits Hex) and base64url Encode

Exponent는 `65537`로 결과값이 `AQAB` 일 것으로 예상되지만, 테스트 과정이므로 동일한 과정을 거쳐줍니다.
Modulus는 Zero-leading Hex 값이 모두 표시되지만, Exponent의 경우 010001이 10001로 표기되기 때문에 `echo`가 아닌 `printf` 커맨드를 사용합니다.

```bash
EXP_HEX=$(openssl rsa -pubin -inform PEM -text -noout < $varCN.public.pem | grep 'Exponent' | cut -f 2 -d '(' | cut -f 1 -d ')' )
EXP=$(printf "%06X\n" $EXP_HEX | xxd -r -p | base64 | tr -d '=' | tr '/+' '_-' | tr -d '\n')

echo $EXP
```

##### Extract Modulus and Exponent (다른 방법)

좀더 명확하게 Modulus와 Exponent의 Hex 값을 구하는 방법이 있습니다.

```bash
openssl asn1parse -in $varCN.public.pem
    0:d=0  hl=3 l= 159 cons: SEQUENCE
    3:d=1  hl=2 l=  13 cons: SEQUENCE
    5:d=2  hl=2 l=   9 prim: OBJECT            :rsaEncryption
   16:d=2  hl=2 l=   0 prim: NULL
   18:d=1  hl=3 l= 141 prim: BIT STRING
```

asn1parse를 사용해 나타나는 결과의 `BIT STRING` 위치에 Modulus, Exponent가 있습니다.
커맨드 실행 결과의 각 라인 맨 앞의 숫자는 offset을 의미합니다.
그런데, RSA Key Bit 수에 따라 이 offset 숫자가 변할 때가 있어 이 방법을 사용하지 않았습니다.

```bash
openssl asn1parse -in $varCN.public.pem -strparse 18
    0:d=0  hl=3 l= 137 cons: SEQUENCE
    3:d=1  hl=3 l= 129 prim: INTEGER           :D22FF1D7EF070A99309298BFE5E290E20BDB11C0F86E9CC6E4106D7BF0272AB3FA616C5465606F302CD7265E8AE04796A4D2A3A15D67A1ADFEFB825757AAE59DF965EEEA15CF81E08C1652D8A36A529B8BE9BC39CC9FDB7C4BF6673491D299B55202B091D3B96F8A4079E446DAB436607ACE5620301033F594B15652AAE8A4D7
  135:d=1  hl=2 l=   3 prim: INTEGER           :010001
```

#### Generate JWK, JWKS

추출한 필수 필드를 기반으로 JWKS를 작성합니다.

```bash
KID=$(printf "%04d" $(expr $RANDOM % 1000))
JWK='{"kty":"RSA","e":"'"${EXP}"'","use":"sig","kid":"'"${KID}"'","alg":"RS256","n":"'"${MOD}"'"}'
echo $JWK | jq > $varOU.pubkey.jwk

JWKS='{"keys":['"${JWK}"']}'
echo $JWKS | jq > $varOU.pubkey.jwks
```

------

### JWT 생성

#### Generate JWT

- 1시간의 Expiry time을 갖는 Token을 생성합니다.

- Signature 부분의 서명은 X.509 형식 PEM Private Key를 사용합니다.

```bash
NOW=$( date +%s )
IAT="${NOW}"
EXPIRE=$((${NOW} + 3600))

## [참고] 이하의 과정은 JWK 생성과정에서 이미 수행한 커맨드/변수 선언입니다.
varO='dddaong'
varOU='dddaong'
varCN='dddaong.test'
KID=$(printf "%04d" $(expr $RANDOM % 1000))

openssl genrsa -out $varCN.private.pem 1024 > /dev/null 2>&1
openssl rsa -in $varCN.private.pem -out $varCN.public.pem -pubout > /dev/null 2>&1
PEM=$( cat ${varCN}.private.pem )
```

``` bash
## Header
HEADER_RAW='{"alg":"RS256","typ":"JWT","kid":"'"${KID}"'"}'
HEADER=$( echo -n "${HEADER_RAW}" | openssl base64 | tr -d '=' | tr '/+' '_-' | tr -d '\n' )

## Payload
PAYLOAD_RAW='{"iat":'"${IAT}"',"exp":'"${EXPIRE}"',"aud":"'"${varOU}"'","sub":"'"${varCN}"'"}'
PAYLOAD=$( echo -n "${PAYLOAD_RAW}" | openssl base64 | tr -d '=' | tr '/+' '_-' | tr -d '\n' )

HEADER_PAYLOAD="${HEADER}"."${PAYLOAD}"
SIGNATURE=$( openssl dgst -sha256 -sign <(echo -n "${PEM}") <(echo -n "${HEADER_PAYLOAD}") | openssl base64 | tr -d '=' | tr '/+' '_-' | tr -d '\n' )

JWT="${HEADER_PAYLOAD}"."${SIGNATURE}"
echo ${JWT} > $varCN.token
```

이상으로 JWK, JWKS, JWT 생성과정을 정리합니다.
