---
title: "NGINX+ Ingress Controller - Helm Chart 배포"
author: DDAONG
date:   2021-04-10 00:00:00 +0900
category: "NGINX Ingress-Controller"
layout: post
---

Helm 차트를 사용해 NGINX Plus Ingress Controller 배포 테스트

## Helm 차트를 사용해 NGINX Plus Ingress Controller 배포

NGINX Plus Ingress Controller (NPIC)의 이미지 빌드에는 Docker, git, GNU Make, OpenSSL, 그리고 NGINX-Plus 라이선스가 필요합니다.

#### 1. Helm / git 설치

Helm은 Kubernetes에서 리눅스의 YUM rpm, APT dpkg 같이 패키지를 관리해주는 역할을 수행합니다.
Helm의 패키지를 Chart라고 합니다.

Helm 3.5.3을 다운로드 해 압축을 풀어주고, 그 PATH 경로(/usr/local/sbin)으로 이동합니다.

```bash
curl -L https://get.helm.sh/helm-v3.5.3-linux-amd64.tar.gz | tar zxf -
mv linux-amd64/helm /usr/local/sbin
```

git 도 설치해주겠습니다.

```bash
yum -y install git
```

#### 2. NGINX Plus Ingress Controller 이미지 생성

- 이 부분은 [NPIC + NDK + Lua 모듈 테스트](https://dddaong.github.io/nginx-ingress-controller/lua/2021/03/24/NPIC-NDK-Lua-%EB%AA%A8%EB%93%88-%ED%85%8C%EC%8A%A4%ED%8A%B8.html)의 과정과 유사합니다.
  (3월31일 1.11.0버전이 릴리즈되어 1.11.0 버전을 사용합니다.)

```bash
mkdir -p /var/tmp/npic && cd /var/tmp/npic
git clone https://github.com/nginxinc/kubernetes-ingress/
cd kubernetes-ingress
git chechout v1.11.0
```

- 기존 Dockerfile 백업
  v1.11.0에서는 Dockerfile이 변경된 것으로 보입니다.
  - v1.10.0과 달리 App protect for Plus가 별도로 제공되지 않습니다.

```bash
cp /var/tmp/npic/kubernetes-ingress/build/Dockerfile \
/var/tmp/npic/kubernetes-ingress/build/Dockerfile.bak
```

- 기존 Dockerfile 수정

```bash
vi /var/tmp/npic/kubernetes-ingress/build/Dockerfile
```

- yum -y install nginx-plus 커맨드에 아래 패키지를 추가해줍니다.
  - nginx-plus-module-ndk
  - nginx-plus-module-lua
  - nginx-plus-module-cookie-flag
  - nginx-plus-module-encrypted-session
  - nginx-plus-module-headers-more
  - nginx-plus-module-set-misc

추가한 Dockerfile은 아래 형태가 됩니다.
(v1.11.0의 Dockerfile 파일의 52번 라인을 수정했습니다.)

```bash
...[omitted]
apt-transport-https libcap2-bin nginx-plus=${NGINX_PLUS_VERSION} nginx-plus-module-njs=        ${NGINX_NJS_VERSION} nginx-plus-module-ndk nginx-plus-module-lua nginx-plus-module-cookie-flag nginx-plus-module-encrypted-session nginx-plus-module-headers-more nginx-plus-module-set-misc \
...[omitted]
```

#### 3. NGINX+ 라이선스를 Kubernetes-ingress 디렉토리에 복사하고, 이미지 빌드를 시작합니다

- NGINX+ 테스트 라이선스를 사용했습니다.

```bash
cp /var/tmp/nginx-repo* /var/tmp/npic/kubernetes-ingress/
```

- 이미지 빌드를 시작합니다.
  (Dockerfile 변경에 따라 make 명령어도 v1.10.0에 비해 달라졌습니다.)

```bash
make debian-image-plus PREFIX=dddaong/nginx-plus-ingress TARGET=container
```

~~빌드가 완료되면 Repository에 업로드를 시도하기 때문에 아래 오류 메시지는 무시하셔도 좋습니다.~~
v1.11.0부터는 빌드가 완료되어도 Registry에 업로드를 시도하지 않습니다.

- 빌드과정이 완료되면 docker에서 이미지를 확인합니다. Tag가 edge로 지정되어 있는 것이 보입니다.

```bash
$ docker image ls
REPOSITORY                           TAG        IMAGE ID       CREATED              SIZE
dddaong/nginx-plus-ingress           edge       ab47ffdd0e3a   About a minute ago   136MB
```

#### 4. 테스트 환경에 Image repository가 없으므로 수동으로 노드에 이미지를 로드해줍니다

- save docker image

```bash
docker save dddaong/nginx-plus-ingress:edge -o /var/tmp/npic/dddaong-npic-edge.tgz
scp /var/tmp/npic/dddaong-npic-edge.tgz root@<workernode1>:/var/tmp
scp /var/tmp/npic/dddaong-npic-edge.tgz root@<workernode2>:/var/tmp
```

- load docker image @each node

```bash
docker load -i /var/tmp/dddaong-npic-edge.tgz
```

#### 5-1. 기존 NPIC 에 새로 빌드한 이미지 적용

Helm 차트를 사용한 NPIC 구축은 5-2에서 수행합니다.
기존에 NPIC를 삭제하기 전에, 새로 빌드한 이미지를 적용하는 테스트를 수행 했습니다.

- 이번 테스트에서는 kubectl 커맨드로 롤아웃 업데이트를 해보겠습니다.

```bash
$ kubectl -n nginx-ingress get deployments.apps
NAME            READY   UP-TO-DATE   AVAILABLE   AGE
nginx-ingress   1/1     1            1           13d
```

- Deployment 리소스의 Containers에서 항목명을 확인합니다.
  이 테스트에서 이미지를 변경할 Container 이름은 nginx-plus-ingress로 확인됩니다.

```bash
$ kubectl -n nginx-ingress describe deployments.apps nginx-ingress | grep -A 4 'Container'
  Containers:
   nginx-plus-ingress:
    Image:       samsung/nginxplus-ingress:1.10.1
    Ports:       80/TCP, 443/TCP, 8081/TCP
    Host Ports:  0/TCP, 0/TCP, 0/TCP
```

- 확인한 항목의 이미지를 kubectl 명령어로 변경해줍니다.
  - 커맨드 수행시 즉시 Rollout이 진행됩니다!!

```bash
$ kubectl -n nginx-ingress set image deployments/nginx-ingress nginx-plus-ingress=dddaong/nginx-plus-ingress:edge
deployment.apps/nginx-ingress image updated
```

- Rollout 결과를 확인합니다.

```bash
$ kubectl -n nginx-ingress rollout status deployment nginx-ingress
deployment "nginx-ingress" successfully rolled out
```

- 파드와 Deployment 상태를 확인합니다.

```bash
$ kubectl -n nginx-ingress get pod
NAME                             READY   STATUS    RESTARTS   AGE
nginx-ingress-6ff488c8c8-xtsn8   1/1     Running   0          34s
```

```bash
$ kubectl -n nginx-ingress describe deployments.apps nginx-ingress | grep -A 4 'Container'
  Containers:
   nginx-plus-ingress:
    Image:       dddaong/nginx-plus-ingress:edge
    Ports:       80/TCP, 443/TCP, 8081/TCP
    Host Ports:  0/TCP, 0/TCP, 0/TCP
```

#### 5-2.  Helm을 사용해 Kubernetes에 NPIC 설치

- 먼저 기존 NPIC를 제거합니다

```bash
kubectl delete namespaces nginx-ingress
kubectl delete ingressclasses nginx
```

- helm 차트 리소스가 있는 디렉토리로 들어갑니다.

```bash
cd /var/tmp/npic/kubernetes-ingress/deployments/helm-chart
git checkout v1.11.0
```

- values-plus.yaml 파일을 백업한 후, 파일 내 이미지 경로를 수정합니다.

```bash
cp values-plus.yaml values-plus.yaml.bak
vi values-plus.yaml
```

```
controller:
  nginxplus: true
  image:
    repository: <NPIC Repository/image name>
    tag: "edge"
```

이 테스트의 경우, dddaong/nginx-plus-ingress로 작성했습니다.

- helm을 사용해 인스톨을 수행합니다.

```bash
helm install dddaong -f values-plus.yaml .
```

"dddaong"은 임의의 릴리즈 명으로서, 원하는 이름을 사용하면 됩니다.

```
W0401 13:28:12.293718   23860 warnings.go:70] networking.k8s.io/v1beta1 IngressClass is deprecated in v1.19+, unavailable in v1.22+; use networking.k8s.io/v1 IngressClassList
W0401 13:28:12.490643   23860 warnings.go:70] networking.k8s.io/v1beta1 IngressClass is deprecated in v1.19+, unavailable in v1.22+; use networking.k8s.io/v1 IngressClassList
NAME: dddaong
LAST DEPLOYED: Thu Apr  1 13:28:11 2021
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The NGINX Ingress Controller has been installed.
```

- Custom Resource Definition들을 설치합니다.

```bash
$ kubectl apply -f crds/
customresourcedefinition.apiextensions.k8s.io/aplogconfs.appprotect.f5.com configured
customresourcedefinition.apiextensions.k8s.io/appolicies.appprotect.f5.com configured
customresourcedefinition.apiextensions.k8s.io/apusersigs.appprotect.f5.com configured
customresourcedefinition.apiextensions.k8s.io/globalconfigurations.k8s.nginx.org configured
customresourcedefinition.apiextensions.k8s.io/policies.k8s.nginx.org configured
customresourcedefinition.apiextensions.k8s.io/transportservers.k8s.nginx.org configured
customresourcedefinition.apiextensions.k8s.io/virtualserverroutes.k8s.nginx.org configured
customresourcedefinition.apiextensions.k8s.io/virtualservers.k8s.nginx.org configured
```

- 설치 결과를 확인합니다.
  helm으로 제공되는 설치 방법은 별도의 nginx-ingress namespace를 생성하지 않고, default namespace에 설치됩니다.

```bash
$ kubectl get namespaces
NAME              STATUS   AGE
default           Active   13d
kube-node-lease   Active   13d
kube-public       Active   13d
kube-system       Active   13d
```

```bash
$ kubectl get pod
NAME                                     READY   STATUS    RESTARTS   AGE
dddaong-nginx-ingress-64f6d88767-txgvw   1/1     Running   0          3m46s
fakeapi-6f89b55d54-4j8vp                 1/1     Running   0          2d16h
```

#### 6. Lua 사용을 위한 configmap을 수정

- nginx-ingress에서 사용하는 configmap을 확인합니다.

```bash
$ kubectl describe pod dddaong-nginx-ingress-64f6d88767-txgvw | grep configmap
      -nginx-configmaps=$(POD_NAMESPACE)/dddaong-nginx-ingress
      
      
$ kubectl get cm
NAME                                    DATA   AGE
dddaong-nginx-ingress                   0      8m56s
```

- configmap 수정합니다.

```bash
kubectl edit cm dddaong-nginx-ingress
```

- 기존 파일 최하단에 아래 내용을 작성해주고 저장합니다.

```yaml
data:
  main-snippets: |
    load_module modules/ndk_http_module.so;
    load_module modules/ngx_http_lua_module.so;
    load_module modules/ngx_stream_lua_module.so;
    load_module modules/ngx_http_encrypted_session_module.so;
    load_module modules/ngx_http_set_misc_module.so;
    load_module modules/ngx_http_cookie_flag_filter_module.so;
    load_module modules/ngx_http_headers_more_filter_module.so;
```

변경된 configmap이 잘 반영되었는지 확인합니다.

```bash
kubectl exec -it dddaong-nginx-ingress-64f6d88767-txgvw -- cat /etc/nginx/nginx.conf | grep load_module
load_module modules/ndk_http_module.so;
load_module modules/ngx_http_lua_module.so;
load_module modules/ngx_stream_lua_module.so;
load_module modules/ngx_http_encrypted_session_module.so;
load_module modules/ngx_http_set_misc_module.so;
load_module modules/ngx_http_cookie_flag_filter_module.so;
load_module modules/ngx_http_headers_more_filter_module.so;
```

#### 7. 테스트를 위한 웹서버 및 Lua 서버 블록 설정

- NPIC 기본동작 테스트에 사용할 web server Pod와 Service를 생성합니다.

```bash
kubectl create deployment web --image=gcr.io/google-samples/hello-app:1.0
kubectl expose deployment web --type=NodePort --port=8080
```

- Lua 테스트를 위한 서버 블록 역시 ingress-controller 파드 내부에 작성해 줍니다.

```bash
kubectl exec -it dddaong-nginx-ingress-64f6d88767-txgvw -- bash

echo 'server {
    server_name hellworld-lua.test;
    listen 80;
    location / {
      default_type text/html;
      content_by_lua_block {
        ngx.say("<p>helllo, world</p>")
      }
    }
    }
' > /etc/nginx/conf.d/lua_test.conf

nginx -s reload
```

- 테스트 편의를 위해 .spec.externalTrafficPolicy 의 값을 Local에서 Cluster로 변경해 줍니다.

```bash
kubectl edit svc dddaong-nginx-ingress
```

```yaml
... [ommitted]
spec:
  clusterIP: 10.101.120.242
  clusterIPs:
  - 10.101.120.242
  externalTrafficPolicy: Cluster
  ports:
[ommitted]...
```

- curl로 간단하게 웹 서버와 Lua 서버 블록의 파드 및 서비스를 확인합니다.

```bash
$ curl [web Pod Cluster IP]:8080
Hello, world!
Version: 1.0.0
Hostname: web-79d88c97d6-gkv29

$ curl [web Svc Cluster IP]:8080
```

```bash
$ curl [nginx ingress Pod Cluster IP]:8888
<p>helllo, world</p>

$ curl [nginx ingress Svc Cluster IP]:8888
<p>helllo, world</p>
```

#### 8. 각 서비스에 대한 Ingress 작성 및 테스트

helloworld.yaml을 아래와 같이 작성 후, Kubernetes Cluster에 적용해 줍니다.

(!주의!)
Helm 차트로 설치한 NPIC는 Default IngressClass로 설정되어 있지 않기 때문에
Ingress 리소스 생성시 .spec.ingressClassName: nginx을 반드시 지정해주거나 nginx IngressClass를 Default로 지정해줘야 합니다.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: helloworld-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
  ingressClassName: nginx
  rules:
    - host: helloworld.test
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 8080
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: helloworld-lua-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
  ingressClassName: nginx
  rules:
    - host: helloworld-lua.test
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: dddaong-nginx-ingress
                port:
                  number: 8888
```

```bash
kubectl apply -f helloworld.yaml
```

- ingress가 Service와 잘 연결 되었는지 확인합니다.

```bash
$ kubectl get ingress
NAME                     CLASS   HOSTS                 ADDRESS   PORTS   AGE
helloworld-ingress       nginx   helloworld.test                 80      112m
helloworld-lua-ingress   nginx   helloworld-lua.test             80      79m


$ kubectl describe ingress helloworld-ingress | grep -A4 Rules
Rules:
  Host             Path  Backends
  ----             ----  --------
  helloworld.test
                   /   web:8080 (192.168.225.209:8080)


$ kubectl describe ingress helloworld-lua-ingress | grep -A4 Rules
Rules:
  Host                 Path  Backends
  ----                 ----  --------
  helloworld-lua.test
                       /   dddaong-nginx-ingress:8888 (192.168.225.208:8888)
```

backends 항목에 Service 리소스의 Cluster IP가 확인된다면 잘 연결된 것입니다.

- 테스트 수행 및 결과
  아래와 같이 Hostname Base Route가 잘 동작하는 것을 확인할 수 있습니다.

```bash
$ curl -H Host:helloworld.test <k8s node IP:nodePort>
Hello, world!
Version: 1.0.0
Hostname: web-79d88c97d6-gkv29

$ curl -H Host:helloworld-lua.test <k8s node IP:nodePort>
<p>helllo, world</p>
```
