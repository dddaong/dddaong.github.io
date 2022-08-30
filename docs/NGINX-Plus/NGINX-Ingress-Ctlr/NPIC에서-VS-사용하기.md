---
title:  "NGINX+ Ingress Controller - VS Crd로 Lua 사용"
author: DDAONG
date:   2021-04-05
category: "NGINX+ Ingress Controller"
layout: post
---

NPIC에서 VirtualServer Crd로 Lua 사용하기

## NPIC에서 VirtualServer Crd로 Lua 사용하기

#### 1. NGINX Ingress Controller의 Virtual Server

Virtual Server는 Ingress를 대체할 수 있는 커스텀 리소스 입니다.
Kubernetes의 기본값인 Ingress 리소스에서는 지원하지 않는 NGINX의 기능을 십분 활용할 수 있도록 해줍니다.

[NGINX.com - References](https://docs.nginx.com/nginx-ingress-controller/configuration/virtualserver-and-virtualserverroute-resources/)

[NGINX Inc Github - YAML Examples](https://github.com/nginxinc/kubernetes-ingress/tree/v1.11.0/examples-of-custom-resources)

#### 2. Virtualserver YAML Sample

VirtualServer YAML 파일은 NGINX Conf에 매칭되는 형식으로 구성됩니다.

```yaml
apiVersion: k8s.nginx.org/v1
kind: VirtualServer
metadata:
  name: cafe  
spec:
  host: cafe.example.com # NGINX Conf server_name에 해당
  tls: #     NGINX Conf ssl_certificate, ssl_certificate_key에 해당
    secret: cafe-secret
  ingressClassName: #  ingressClass 선택 가능
  http-snippet: | #   NGINX Conf - HTTP Block 내 옵션 작성 가능
  server-snippet: | #  NGINX Conf - Server Block 내 옵션 작성 가능
  policies: #    Policy Crd 연동
  - name: custom-policy
  upstreams: #    NGINX Conf upstream에 해당, LB 대상 서비스 Pool
  - name: upstream-member
    service: upstream-member-svc
    port: 80
  routes: #     NGINX Conf Location Block에 해당
  - path: /svc
    action:
      pass: upstream-member-svc # NGINX Conf Proxy_pass에 해당 
  - path: ~ ^/decaf/.*\\.jpg$
    action:
      pass: coffee
  - path: = /green/tea
    action:
      pass: tea
```

- Main Snippet을 제외한 http, server Snippet (NGINX Conf 형식) 작성이 가능합니다.
- VirtualServer.spec.path 아래 Location Snippet 작성도 가능합니다.

- Policies Crd 항목에서는 AccessControl, RateLimit, JWT 인증, Multiple TLS, WAF(N+ AppProtect) 등의 설정을 지정해 사용할 수 있습니다.
  [NGINX Policy Crd Reference](https://docs.nginx.com/nginx-ingress-controller/configuration/policy-resource/)

#### 3. Test

[NGINX Ingress Controller - lua_package_path 테스트](https://dddaong.github.io/npic/lua/lua_package_path/2021/04/01/NGINX-Ingress-Controller-Lua-package-path-%ED%85%8C%EC%8A%A4%ED%8A%B8.html) 포스트의 테스트 환경에서 ingress를 VS로 대체하는 테스트입니다.

- HostPath - Volume 설정은 [NGINX Ingress Controller - lua_package_path 테스트](https://dddaong.github.io/npic/lua/lua_package_path/2021/04/01/NGINX-Ingress-Controller-Lua-package-path-%ED%85%8C%EC%8A%A4%ED%8A%B8.html)와 동일하게 유지 했습니다.

- nginx-config ConfigMap 리소스에서 main-snippet은 유지하고,
  http-snippets 항목을 삭제합니다.(생성할 VirtualServer 리소스에 작성합니다.)

  ```yaml
  #kubectl edit cm nginx-config
  
  data:
    main-snippets: |
      load_module /etc/nginx/modules/ndk_http_module.so;
      load_module /etc/nginx/modules/ngx_http_lua_module.so;
      load_module /etc/nginx/modules/ngx_stream_lua_module.so;
      load_module /etc/nginx/modules/ngx_http_encrypted_session_module.so;
      load_module /etc/nginx/modules/ngx_http_set_misc_module.so;
      load_module /etc/nginx/modules/ngx_http_cookie_flag_filter_module.so;
      load_module /etc/nginx/modules/ngx_http_headers_more_filter_module.so;
  ```

- VS로 대체할 ingress 리소스는 삭제합니다.

  ```bash
  kubectl delete ingress lua-server
  ```

- VirtualServer의 Snippet은 아래 단점 때문에 기본값으로 Disabled 되어 있습니다.
  - 복잡성이 증가합니다.
  - Snippet 내용의 유효성까지 체크하지 못하기 때문에 Ingress Controller의 Config Reload 오류를 유발할 수 있습니다.
  - Conf 파일에 직접적인 편집이 가능한 것은 잠재적으로 보안적 위협이 될 수 있다고 합니다.

따라서 아래 방법으로 enable-snippets Argument를 Enable로 수정해줍니다.

```bash
#kubectl edit deployments.apps dddaong-nginx-ingress
 
 ...[ommitted]
 spec:
  template:
    spec:
      containers:
      - args:
        - -enable-snippets=true
      [ommitted] ...
```

- VirtualServer는 Listen Port를 수정할 수 없는 것으로 보입니다.

  (TCP/UDP 서비스를 위한 TransportServer Crd의 경우에 Listen Port를 지정할 수 있는 것으로 보입니다.)
  80 (http) 포트를 사용하되, 굳이 Service 리소스의 8888 포트를 삭제하진 않고 그대로 사용했습니다.

- [NGINX Ingress Controller - lua_package_path 테스트](https://dddaong.github.io/npic/lua/lua_package_path/2021/04/01/NGINX-Ingress-Controller-Lua-package-path-%ED%85%8C%EC%8A%A4%ED%8A%B8.html) 포스트의 ingress 리소스를 대체할
  VirtualServer YAML을 아래와 같이 작성했습니다.

```yaml
apiVersion: k8s.nginx.org/v1
kind: VirtualServer
metadata:
  name: hello-lua
spec:
  http-snippets: |
    lua_package_path "/var/tmp/?.lua;;";
  host: hello-lua.test
  upstreams:
  - name: lua-server
    service: dddaong-nginx-ingress
    port: 80
  server-snippets: |
    location / {
      default_type text/plain;
      content_by_lua_block {
        local hello = require "hello"
        hello.greet("a lua module")
      }
    }
  routes:
  - path: /lua
    action:
      return:
        code: 200
        type: text/plain
        body: "Dummy Response Text\n"
```

`kubectl apply -f vs-hello-lua.yaml`로 적용해줍니다.

적용 후 생성된 Conf 파일을 확인합니다.

`kubectl exec -it [nginx-ingress Podname] -- cat /etc/nginx/conf.d/vs_default_hello-lua.conf | egrep -v ^[[:space:]]*$`

결과가 아래와 같이 나타나면 잘 적용된 것 입니다.

```nginx
nginx/conf.d/vs_default_hello-lua.conf | egrep -v  ^[[:space:]]*$
upstream vs_default_hello-lua_lua-server {
    zone vs_default_hello-lua_lua-server 256k;
    random two least_conn;
    server 192.168.225.223:80 max_fails=1 fail_timeout=10s max_conns=0;
}
lua_package_path "/var/tmp/?.lua;;";
server {
    listen 80;
    server_name hello-lua.test;
    status_zone hello-lua.test;
    set $resource_type "virtualserver";
    set $resource_name "hello-lua";
    set $resource_namespace "default";
    server_tokens "on";
    location / {
      default_type text/plain;
      content_by_lua_block {
        local hello = require "hello"
        hello.greet("a lua module")
      }
    }
    location @return_0 {
        default_type "text/plain";
        # status code is ignored here, using 0
        return 0 "Dummy Response Text
";
    }
    location /lua {
        set $service "";
        error_page 418 =200 "@return_0";
        proxy_intercept_errors on;
        proxy_pass http://unix:/var/lib/nginx/nginx-418-server.sock;
    }
}

```

#### 4. Test 수행 및 결과

```
curl -H host:hello-lua.test http://<Pod IP>
curl -H host:hello-lua.test http://<Svc ClusterIP>
curl -H host:hello-lua.test http://<Node IP>:30080
```

결과가 모두  `Greetings from a lua module` 로 나오면 성공입니다.

참고로, `curl -H host:hello-lua.test http://<Node IP>:30080/lua`로 Curl을 하면
 'Dummy Response text'라는 텍스트를 응답으로 반환합니다.

이 동작은 VirtualServer YAML의 'Action' 부분 때문인데,
Action을 작성하지 않으면 Validation에 의해 Config가 Refect되기 때문에
임의의 Location을 작성한 것입니다.
