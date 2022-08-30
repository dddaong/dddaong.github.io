---
title: "NGINX+ Ingress Controller - lua_package_path 테스트"
author: DDAONG
date:   2021-04-02
category: "NGINX+ Ingress-Controller"
layout: post

---

## NGINX Ingress Controller - lua_package_path 테스트

nginx-config ConfigMap 리소스의 main snippet으로 모듈로딩,
nginx-config ConfigMap 리소스의 http snippet으로 lua_package_path 설정,
마지막으로 Ingress 리소스를 통해 listen port, location을 작성해

File 형식의 Lua를 사용하는 테스트입니다.

#### 1. NPIC Pod에 Volume 설정

- Lua File path 사용을 위해 Volume 설정 (Deployment 리소스)

`# kubectl edit deployments.apps dddaong-nginx-ingress`

```yaml
[Ommitted]
        terminationMessagePolicy: File
        ## lines added from Here -
        volumeMounts:
        - mountPath: /var/tmp
          name: luapkg-path
        ## - to Here
      dnsPolicy: ClusterFirst
[Ommitted]
      terminationGracePeriodSeconds: 30
      ## lines added from Here -
      volumes:
      - hostPath:
          path: /var/tmp/lua
          type: DirectoryOrCreate
        name: luapkg-path
      ## - to Here
[Ommitted]
```

#### 2. Pod가 생성된 node에 hello.lua 작성

```bash
mkdir -p /var/tmp/lua/
tee -a /var/tmp/lua/hello.lua <<EOF 1>/dev/null
local _M = {}

function _M.greet(name)
 ngx.say("Greetings from ", name)
end

return _M
EOF
```

#### 3. snippet 작성해 Conf 반영

`kubectl edit cm nginx-config`

```yaml
data:
  http-snippets: |
    lua_package_path "/etc/nginx/conf.d/?.lua;;";
  main-snippets: |
    load_module /etc/nginx/modules/ndk_http_module.so;
    load_module /etc/nginx/modules/ngx_http_lua_module.so;
    load_module /etc/nginx/modules/ngx_stream_lua_module.so;
    load_module /etc/nginx/modules/ngx_http_encrypted_session_module.so;
    load_module /etc/nginx/modules/ngx_http_set_misc_module.so;
    load_module /etc/nginx/modules/ngx_http_cookie_flag_filter_module.so;
    load_module /etc/nginx/modules/ngx_http_headers_more_filter_module.so;
```

#### 4. Service 리소스 수정 및 ingress 생성

- NPIC svc에 Port 추가

`kubectl edit svc dddaong-nginx-ingress`

```yaml
  - name: lua
    nodePort: 30180
    port: 8888
    protocol: TCP
    targetPort: 8888
  - name: https
    nodePort: 30443
```

- Ingress 생성
  - Ingress가 생성되면, Pod 내 /etc/nginx/conf.d 디렉토리에 conf 파일이 생성됩니다.
  - 생성되는 conf 파일의 Config 유효성을 체크해가며 수정하면 되겠습니다.

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: lua-server
  annotations:
    nginx.org/listen-ports: "8888"
    nginx.org/server-snippets: |
      location / {
        default_type text/plain;
        content_by_lua_block {
          local hello = require "hello"
          hello.greet("a lua module")
        }
      }
spec:
  rules:
  - host: hello-lua.test
    http: ### Dummy conf below
      paths:
      - path: /lua
        backend:
          serviceName: dddaong-nginx-ingress
          servicePort: 8888
```

#### 5. 테스트 수행

`curl -H host:hello-lua.test http://<Pod IP>:8888`

`curl -H host:hello-lua.test http://<Svc ClusterIP>:8888`

`curl -H host:hello-lua.test http://<Node IP>:30180`

결과가 모두 `Greetings from a lua module` 로 나오면 설정이 잘된 것입니다.

참고로 Ingress의 Dummy conf URI로 Curl을 하면 400 Request Header Or Cookie Too Large 오류가 발생합니다.
