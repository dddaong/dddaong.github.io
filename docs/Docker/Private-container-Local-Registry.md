---
title: "Docker - Private(Local) Container Registry"
author: DDAONG
date:   2021-04-15 15:00:00 +0900
category: Docker
layout: post
---

Docker.com의 Local Docker Registry Server 문서를 기반으로 수행한 테스트 과정을 작성합니다.

## Docker Private(Local) Registry 테스트

- 이 포스트의 주된 내용은 [Deploy a registry server](ttps://docs.docker.com/registry/deploying/) 내용을 기반으로, 테스트용 Kubernetes Cluster 환경에 맞게 커스터마이즈한 [Local Docker Image Registry 생성하기] 입니다.
- Docker.com에서 기본으로 제공하는 Registry 서버는 기본값으로 Localhost에서만 접근할 수 있습니다.
  - 참조 : [Run an externally-accessible registry](https://docs.docker.com/registry/deploying/#run-an-externally-accessible-registry)

- 따라서 이 포스트는 인증서 생성 및 Self-signed Cert의 생성 및 적용 과정,
  그리고 다른 노드에서 Push/Pull 테스트까지 내용을 작성합니다.

#### Self-signed Cert 생성

```bash
varPATH=/var/tmp/registry/certs
varO='dddaong'
varOU='dddaong'
varCN='dddaong.test'

mkdir -p $varPATH
openssl req \
   -newkey rsa:4096 -nodes -sha256 -keyout /var/tmp/registry/certs/$varCN.key \
   -subj "/C=KR/ST=Seoul/L=Seoul/O=$varO/OU=$varOU Department/CN=$varCN" \
   -x509 -days 365 -out /var/tmp/registry/certs/$varCN.crt
```

#### Run Registry 컨테이너 실행

Localhost가 아닌 외부에서 Registry로 접근할 수 있도록 하려면,
생성한 인증서를 Container 내부에서 사용할 수 있도록 하는 설정이 필요합니다.

아래 명령어로 실행합니다.

```bash
# $varCN='dddaong.test'
# $varPATH=/var/tmp/registry/certs
mkdir -p /var/tmp/registry/repos/
docker run -d \
  --restart=always \
  --name registry \
  -v "$varPATH":/certs \
  -v /var/tmp/registry/repos:/var/lib/registry \
  -e REGISTRY_HTTP_ADDR=0.0.0.0:443 \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/$varCN.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/$varCN.key \
  -p 44443:443 \
  registry:2
```

- `-v` flag로 HostPath를 컨테이너 내부에서 사용할 수 있도록 볼륨을 잡아줍니다.

  - `-v "$varPATH":/certs` :
    인증서 경로로서, 외부 접근을 위해 필수로 잡아주는 것이 좋습니다.
  - `-v /var/tmp/registry/repos:/var/lib/registry`:
    Push 된 이미지가 저장되는 경로로서 필수 옵션은 아니지만, Push 테스트 시 그 결과를 쉽게 확인할 수 있습니다.

- Registry의 포트를 변경하고 싶은 경우,
  예를 들어, 44443 포트에서 443 포트로 변경하려면  `-p 44443:443`부분을  `-p 443:443` 으로 변경해주면 됩니다.
  
- 잘 동작하는지 아래 명령어로 확인해줍니다.

  ```bash
  curl -vsk https://localhost:44443
  ```

- 아래와 같이 200 OK를 받아오면 정상입니다.

```bash
  # curl -vsk https://localhost:44443
  * About to connect() to localhost port 44443 (#0)
  *   Trying ::1...
  * Connected to localhost (::1) port 44443 (#0)
  * Initializing NSS with certpath: sql:/etc/pki/nssdb
  * skipping SSL peer certificate verification
  * SSL connection using TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
  * Server certificate:
  *       subject: CN=dddaong.test,OU=dddaong Department,O=dddaong,L=Seoul,ST=Seoul,C=KR
  *       start date: Apr 15 04:03:42 2021 GMT
  *       expire date: Apr 15 04:03:42 2022 GMT
  *       common name: dddaong.test
  *       issuer: CN=dddaong.test,OU=dddaong Department,O=dddaong,L=Seoul,ST=Seoul,C=KR
  > GET / HTTP/1.1
  > User-Agent: curl/7.29.0
  > Host: localhost:44443
  > Accept: */*
  >
  < HTTP/1.1 200 OK
  < Cache-Control: no-cache
  < Date: Thu, 15 Apr 2021 04:49:22 GMT
  < Content-Length: 0
  <
  * Connection #0 to host localhost left intact
```

#### Client Docker Daemon 설정

- 각 Docker Host의 `/etc/docker/certs.d/` 디렉토리로 인증서를 복사해주는 과정이 필요합니다.

- 이 과정은 동작 중인 Registry Server의 Client가 될 Docker Host 들이
  Registry 서버에서 전달하는 인증서를 안전한 인증서로 인식하게 해주는 과정입니다.

- Registry의 모든 클라이언트 Docker Host에서 수행해야 합니다.

docker Client가 될 Host의 SSH로 접속해 아래 명령을 수행합니다.

varRegiaddr 변수에는 Registry 컨테이너가 실행 중인 Host의 IP를 작성해주면 됩니다.


```bash
varRegiaddr="xxx.xxx.xxx.xxx"
varCN="dddaong.test"
varPort="44443"

echo "$varRegiaddr    $varCN" >> /etc/hosts
mkdir -p /etc/docker/certs.d/$varCN:$varPort/
scp root@$varRegiaddr:/var/tmp/registry/certs/$varCN.crt \
  /etc/docker/certs.d/$varCN:$varPort/ca.crt
```


과정이 잘 수행되었는지 확인합니다.


```bash
tail -1 /etc/hosts && ls /etc/docker/certs.d/$varCN:$varPort/
```

이제 외부에서도 Registry를 사용할 준비가 되었습니다.



#### Registry 외부 사용 테스트

Debian 리눅스 이미지를 Docker hub에서 Pulling한 후, Registry에 Push 해보는 간단한 테스트 수행합니다.

- 먼저 Debian 리눅스 이미지를 Docker Hub에서 Pull 합니다.

```bash
docker pull debian:buster-slim
```

- hosts 파일에 dddaong.test의 IP를 작성해 주고,
   Tag 명령어로, 이미지의 Registry 정보를 작성/변경해줍니다.

```bash
docker tag debian:buster-slim dddaong.test:44443/debian:buster-slim
```

- Registry 서버로 Push를 수행해봅니다.
   결과가 아래와 같이 나타난다면 Push가 정상적으로 수행된 것입니다.

```bash
# docker push dddaong.test:44443/debian:buster-slim
   
The push refers to repository [dddaong.test:44443/debian]
7e718b9c0c8c: Pushed
buster-slim: digest: sha256:f4793ff7e177674c0b8541165f67289907aaf4391f60d71ba2a3db8a70fa76dc size: 529
```

Registry Server가 동작 중인 Host에서, Push 된 Debian 이미지를 직접 확인해봅니다.


```bash
   # -v /var/tmp/registry/repos:/var/lib/registry 지정한 경우
   ls /var/tmp/registry/repos
   
   # -v /var/tmp/registry/repos:/var/lib/registry 지정하지 않은 경우
   docker exec -t registry ls /var/lib/registry/docker/registry/v2/repositories/
```

- Pull 테스트를 위해 Docker Hub에서 가져온 Debian 리눅스 이미지를 삭제합니다.

```bash
   docker rmi debian:buster-slim
   docker rmi dddaong.test:44443/debian:buster-slim
```

- dddaong.test:44443에서 debian 리눅스를 Pull 해봅니다.

결과가 아래와 같이 나타난다면 Pull이 정상적으로 수행된 것입니다.
 
```bash
   # docker pull dddaong.test:44443/debian:buster-slim
   
   buster-slim: Pulling from debian
   f7ec5a41d630: Pull complete
   Digest: sha256:f4793ff7e177674c0b8541165f67289907aaf4391f60d71ba2a3db8a70fa76dc
   Status: Downloaded newer image for dddaong.test:44443/debian:buster-slim
```

------

Native Basic Auth, NGINX를 인증 프록시로 사용하는 방법 등은 Docker.com에서 제공하는 [링크 : Basic Auth](https://docs.docker.com/registry/deploying/#native-basic-auth), [링크: Recipes List](https://docs.docker.com/registry/recipes/)를 참고하면 되겠습니다.
