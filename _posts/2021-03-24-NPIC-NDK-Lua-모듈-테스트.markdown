---
layout: post
title:  "NPIC + NDK + Lua(+@) 모듈 테스트"
date:   2021-03-24 13:14:00 +0900
categories: nginx-ingress-controller lua
---

이 포스트는 nginx.com에 있는 NPIC(NGINX+ Ingress Controller) 이미지 빌드 방법에,
필요한 다이나믹 모듈을 추가하는 테스트에 대한 기록을 작성했습니다.

## NPIC + NDK + Lua 모듈 테스트


### 1. 임시 폴더 생성
```bash
mkdir -p /var/tmp/npic/
cd /var/tmp/npic/
```

### 2. nginxinc git clone으로 NPIC에 필요한 파일을 복제
```bash
yum -y install git
git clone https://github.com/nginxinc/kubernetes-ingress/
```

### 3. 버전 지정 및 Dockerfile 수정해 이미지 빌드 시작
- NPIC 버전 지정
```bash
git checkout v1.10.1
```

- 기존 Dockerfile 백업 (이 테스트에서는 Dockerfile)
-- 이 테스트의 Dockerfile은 AppProtect 모듈을 사용할 수 있는 NGINX Plus를 사용했습니다.

```bash
cp /var/tmp/npic/kubernetes-ingress/build/appprotect/DockerfileWithAppProtectForPlus \
/var/tmp/npic/kubernetes-ingress/build/appprotect/DockerfileWithAppProtectForPlus.bak
```

- 기존 Dockerfile 수정

```bash
vi /var/tmp/npic+openresty/kubernetes-ingress/build/appprotect/DockerfileWithAppProtectForPlus
```
- yum -y install nginx-plus 커맨드에 아래 패키지를 추가해줍니다.
  * nginx-plus-module-ndk
  * nginx-plus-module-lua
  * nginx-plus-module-cookie-flag
  * nginx-plus-module-encrypted-session
  * nginx-plus-module-headers-more
  * nginx-plus-module-set-misc

추가한 Dockerfile은 아래 형태가 됩니다.
(이 테스트에서는 DockerfileWithAppProtectForPlus 파일의 51번 라인을 수정했습니다.)
```bash
...[omitted]
&& apt-get update && apt-get install -y nginx-plus=$NGINX_PLUS_VERSION nginx-plus-module-njs=${NGINX_NJS_VERSION} nginx-plus-module-ndk nginx-plus-module-lua nginx-plus-module        -cookie-flag nginx-plus-module-encrypted-session nginx-plus-module-headers-more nginx-plus-module-set-misc 
...[omitted]
```

### 4. NGINX+ 라이선스를 Kubernetes-ingress 디렉토리에 복사하고, 이미지 빌드를 시작합니다.
- NGINX+ 테스트 라이선스를 사용했습니다.
```bash
cp /var/tmp/nginx-repo* /var/tmp/npic/kubernetes-ingress/
```
- 이미지 빌드를 시작합니다.
```bash
make PREFIX=dddaong/nginxplus-ingress DOCKERFILE=appprotect/DockerfileWithAppProtectForPlus
```
빌드가 완료되면 Repository에 업로드를 시도하기 때문에 아래 오류 메시지는 무시하셔도 좋습니다.
```bash
docker push dddaong/nginxplus-ingress:1.10.1
The push refers to repository [docker.io/dddaong/nginxplus-ingress]
88d8672f5205: Preparing
ea7ef61b9298: Preparing
d47136687724: Preparing
c9373f5b7891: Preparing
6b531f110dfa: Preparing
c6b91badbd9f: Waiting
11858bd34312: Waiting
d96c33a7777a: Waiting
df4e940be119: Waiting
14a1ca976738: Waiting
denied: requested access to the resource is denied
make: *** [push] Error 1
```

### 5. 테스트 환경에 Image repository가 없으므로 수동으로 노드에 이미지를 로드해줍니다.

- save docker image
```bash
docker save dddaong/nginxplus-ingress:1.10.1 -o /var/tmp/npic+openresty/samsung_npic.tgz 
scp /var/tmp/npic+openresty/samsung_npic.tgz root@175.196.235.22:/var/tmp
scp /var/tmp/npic+openresty/samsung_npic.tgz root@175.196.235.23:/var/tmp
```

- load docker image @each node
```bash
docker load -i /var/tmp/samsung_npic.tgz
```

### 6. NGINX+ Ingress Controller를 사용하는데 필요한 Kubernetes 리소스들을 설치합니다.

```bash
cd deployments
```

- Apply RBAC
```bash
kubectl apply -f common/ns-and-sa.yaml
kubectl apply -f rbac/rbac.yaml
```
-(for NAP)
```bash
kubectl apply -f rbac/ap-rbac.yaml
```

- Apply Common Resources
```bash
kubectl apply -f common/default-server-secret.yaml
kubectl apply -f common/nginx-config.yaml
kubectl apply -f common/ingress-class.yaml
```

- Apply Custom Resources
```bash
kubectl apply -f common/crds/k8s.nginx.org_virtualservers.yaml
kubectl apply -f common/crds/k8s.nginx.org_virtualserverroutes.yaml
kubectl apply -f common/crds/k8s.nginx.org_transportservers.yaml
kubectl apply -f common/crds/k8s.nginx.org_policies.yaml
```

- Apply Custom Resources for tcp/udp Service
```bash
kubectl apply -f common/crds/k8s.nginx.org_globalconfigurations.yaml
kubectl apply -f common/global-configuration.yaml
```

- Apply App Protect Resources
```bash
kubectl apply -f common/crds/appprotect.f5.com_aplogconfs.yaml
kubectl apply -f common/crds/appprotect.f5.com_appolicies.yaml
kubectl apply -f common/crds/appprotect.f5.com_apusersigs.yaml
```
- Deploy Ingress Controller
```bash
kubectl apply -f deployment/nginx-plus-ingress.yaml
```

### 7. Service 리소스를 생성합니다. (둘 중 원하는 서비스 타입을 선택해 생성합니다.)
- nodePort Type
```bash
kubectl create -f service/nodeport.yaml
```

- LoadBalancer Type (On-Premise 환경에서 로드밸런서 타입은 Metal LB 사용이 필요합니다.)
```bash
kubectl apply -f service/loadbalancer.yaml
kubectl get svc -n nginx-ingress
```

### 8. Lua 모듈 테스트를 위해 nginx-ingress/nginx-config ConfigMap을 YAML을 작성합니다.


```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: nginx-ingress
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

### 9. 테스트를 위해 임시로 Lua 서버 블록을 만들어 NGINX Config 테스트, Curl 테스트를 수행합니다.
- Pod 내부 Bash Shell로 들어가 /etc/nginx/conf.d 디렉토리에 별도 conf 파일을 생성합니다.
```bash
kubectl -n nginx-ingress exec -it <nginx-ingress Pod 이름> -- bash
echo 'server {
    listen 8888;
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

### 10. Curl 테스트
- Pod 내부 Bash에서 빠져나와 Ingress Controller Pod IP를 확인합니다.
```bash
kubectl -n nginx-ingress get pod -o wide
```

- Pod가 Listening 중인 8888 Port에 대해 Curl Test를 수행합니다.
```bash
curl <Pod-IP>:8888
```
