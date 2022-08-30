---
title: "F5 Telemetry Streaming with Prometheus & Grafana Test"
author: DDAONG
date: 2022-08-23
category: "BIG-IP"
layout: post
---

## F5 Telemetry Streaming with Prometheus & Grafana Test

F5 BIG-IP와 Prometheus 및 Grafana를 사용한 모니터링 테스트
[UDF 세션 자료](https://github.com/GBond/f5-ts-with-prometheus-udf)
[github 자료](https://github.com/mdditt2000/prometheus)

### 0. 개요
모니터링 솔루션이 Metric을 수집하는 방법을 Pull / Push 방식으로 구분할 수 있습니다.

*Support for the Prometheus pull consumer is available in TS 1.12.0 and later.*
*14.0.x 이전 버전에서는 아래 cmd로 프레임워크를 Enable하는 과정이 필요합니다.*
```bash
touch /var/config/rest/iapps/enable
```

#### Pull 방식 : 
- 모니터링 솔루션이 먼저 요청을 보내 데이터 수집
- Request에 대한 인증 처리를 통해 보안 측면에서 유리

#### Push 방식 : 
- 각 서버의 Agent가 메트릭 데이터를 수집해 서버로 전달
- 이벤트 생성 빈도가 높은 시스템 모니터링에 유리



### F5 Telemetry Streaming 활성화

F5network Github TS 에서 RPM파일을 다운로드 받습니다. 아래 링크를 사용하면 편리합니다.
[Github.com - f5-telemetry-streaming](https://github.com/F5Networks/f5-telemetry-streaming/releases)
다운로드 받은 파일은 iApps - Package Management LX 메뉴에서 업로드 및 설치를 수행합니다.

설치를 완료하고 나면, Telemetry Streaming은 HTTP Endpoint를 노출합니다.
설치가 잘 되었는지, 아래 URI로 GET Request를 보내어 확인할 수 있습니다

```bash
# Variables
BIGIP="192.168.1.245"
BIGIPUSER="admin"
BIGIPPASS="password"
CREDS="${BIGIPUSER}:${BIGIPPASS}"

# TS TEST CMD
curl -sku $CREDS https://${BIGIP}/mgmt/shared/telemetry/info
```

Declare POST Request를 통해, TS Endpoint를 선언해줄 수 있습니다.
[EndPoint Declaration 샘플](https://clouddocs.f5.com/products/extensions/f5-telemetry-streaming/latest/declarations.html#examples) 및 [Prometheus Pull Consumer 설정 샘플](https://clouddocs.f5.com/products/extensions/f5-telemetry-streaming/latest/pull-consumers.html#prometheus-pull-consumer) 을 제공합니다.

```bash
# Prometheus Declaration sample
cat <<EOF> prom_pull_declare.json
{
    "class": "Telemetry",
    "Prom_Poller": {
        "class": "Telemetry_System_Poller",
        "interval": 0
    },
    "My_System": {
        "class": "Telemetry_System",
        "enable": "true",
        "systemPoller": "Prom_Poller"
    },
    "Prom_Pull_Consumer": {
        "class": "Telemetry_Pull_Consumer",
        "type": "Prometheus",
        "systemPoller": "Prom_Poller"
    }
}
EOF
```

```bash
# Send a POST Declare Request
curl -sku $CREDS -X POST https://${BIGIP}/mgmt/shared/telemetry/declare -H "Content-type: application/json" -d @prom_pull_declare.json

# Pull Consumer TEST CMD
curl -sku $CREDS https://${BIGIP}/mgmt/shared/telemetry/pullconsumer/Prom_Pull_Consumer
```



### Prometheus 설치
Prometheus는  HTTP Protocol 기반의 Pull 방식으로 Time-series(시계열) 데이터를 수집합니다.

 이 테스트에서는 앞서 설정한 Pull Consumer Endpoint 
 "https://${BIGIP}/mgmt/shared/telemetry/pullconsumer/Prom_Pull_Consumer"를 통해 
 모니터링용 시계열 데이터를 수집하게 됩니다.

Prometheus 이외에도 F5 Telemetry Streaming을 사용할 수 있는 툴은 다양합니다.
[CloudDocs](https://clouddocs.f5.com/products/extensions/f5-telemetry-streaming/latest/using-ts.html)에서 TS와 연동 가능한 툴의 리스트 확인할 수 있습니다.

#### Prometheus on Kubernetes

아래 커맨드를 kubernetes Master에서 수행합니다.

```shell
cat <<EOF > prom_ns_rbac.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  labels:
    kubernetes.io/metadata.name: monitoring
  name: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - nodes/proxy
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups:
  - extensions
  resources:
  - ingresses
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: default
  namespace: monitoring
EOF

kubectl apply -f prom_ns_rbac.yaml
```


```shell
cat <<EOF > prom_configmap_bigip.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-server-conf
  labels:
    name: prometheus-server-conf
  namespace: monitoring
data:
  prometheus.rules: |-
    groups:
    - name: devopscube demo alert
      rules:
      - alert: High Pod Memory
        expr: sum(container_memory_usage_bytes) > 1
        for: 1m
        labels:
          severity: slack
        annotations:
          summary: High Memory Usage
  prometheus.yml: |-
    global:
      scrape_interval: 5s
      evaluation_interval: 5s
    rule_files:
      - /etc/prometheus/prometheus.rules
    alerting:
      alertmanagers:
      - scheme: http
        static_configs:
        - targets:
          - "alertmanager.monitoring.svc:9093"

    scrape_configs:
      - job_name: 'BIGIP - TS'
        scrape_timeout: 30s
        scrape_interval: 30s
        scheme: https

        tls_config:
          insecure_skip_verify: true

        metrics_path: '/mgmt/shared/telemetry/pullconsumer/Prom_Pull_Consumer'
        basic_auth:
          username: '${BIGIPUSER}'
          password: '${BIGIPPASS}'
        static_configs:
        - targets: ['${BIGIP}']

      - job_name: cis
        scrape_interval: 10s
        metrics_path: '/metrics'
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_namespace]
            action: replace
            target_label: k8s_namespace
          - source_labels: [__meta_kubernetes_pod_name]
            action: replace
            target_label: k8s_pod_name
          - source_labels: [__address__]
            action: replace
            regex: ([^:]+)(?::\d+)?
            replacement: ${1}:8080
            target_label: __address__
          - source_labels: [__meta_kubernetes_pod_label_app]
            action: keep
            regex: k8s-bigip-ctlr
EOF

kubectl apply -f prom_configmap_bigip.yaml
```

```shell
cat <<EOF > prom_deploy_svc.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-deployment
  namespace: monitoring
  labels:
    app: prometheus-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus-server
  template:
    metadata:
      labels:
        app: prometheus-server
    spec:
      containers:
        - name: prometheus
          image: prom/prometheus
          args:
            - "--config.file=/etc/prometheus/prometheus.yml"
            - "--storage.tsdb.path=/prometheus/"
          ports:
            - containerPort: 9090
          volumeMounts:
            - name: prometheus-config-volume
              mountPath: /etc/prometheus/
            - name: prometheus-storage-volume
              mountPath: /prometheus/
      volumes:
        - name: prometheus-config-volume
          configMap:
            defaultMode: 420
            name: prometheus-server-conf
        - name: prometheus-storage-volume
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: prometheus-service
  namespace: monitoring
  annotations:
      prometheus.io/scrape: 'true'
      prometheus.io/port:   '9090'
spec:
  selector:
    app: prometheus-server
  type: NodePort
  ports:
    - port: 8080
      targetPort: 9090
      nodePort: 30000
EOF

kubectl apply -f prom_deploy_svc.yaml
```



### Grafana 설치
Grafana는 데이터의 시각화를 담당합니다. 

[Grafana.com - Documents](https://grafana.com/docs/grafana/latest/setup-grafana/installation/kubernetes/)

```bash
cat <<EOF > grafana_pv_pvc.yaml
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: grafana-pv
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: local-storage
  local:
    path: /home/volume
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - kube-worker.local
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: grafana-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: local-storage
EOF

kubectl apply -f grafana_pv_pvc.yaml
```
```bash
cat <<EOF > grafana_deploy_svc.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: grafana
  name: grafana
spec:
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      securityContext:
        fsGroup: 472
        supplementalGroups:
          - 0
      containers:
        - name: grafana
          image: grafana/grafana:8.4.4
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 3000
              name: http-grafana
              protocol: TCP
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /robots.txt
              port: 3000
              scheme: HTTP
            initialDelaySeconds: 10
            periodSeconds: 30
            successThreshold: 1
            timeoutSeconds: 2
          livenessProbe:
            failureThreshold: 3
            initialDelaySeconds: 30
            periodSeconds: 10
            successThreshold: 1
            tcpSocket:
              port: 3000
            timeoutSeconds: 1
          resources:
            requests:
              cpu: 250m
              memory: 750Mi
          volumeMounts:
            - mountPath: /var/lib/grafana
              name: grafana-pv
      volumes:
        - name: grafana-pv
          persistentVolumeClaim:
            claimName: grafana-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: grafana
spec:
  ports:
    - port: 3000
      protocol: TCP
      targetPort: http-grafana
  selector:
    app: grafana
  sessionAffinity: None
  type: NodePort
EOF

kubectl apply -f grafana_deploy_svc.yaml
```


