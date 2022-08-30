---
title:  "F5 BIG-IP CIS - Calico+ClusterIP 모드"
author: DDAONG
date:   2022-08-03
category: "BIG-IP"
layout: post
---

# CIS Calico + ClusterIP 모드 구성 테스트

Kubernetes Cluster는 이미 구성되어 있다고 가정합니다.

#### 설치 과정

```bash
kubectl create secret generic bigip-login -n kube-system --from-literal=username=itian --from-literal=password=itian.co.kr
kubectl create serviceaccount bigip-ctlr -n kube-system
```

```bash
echo 'apiVersion: apps/v1
kind: Deployment
metadata:
  name: k8s-bigip-ctlr-deployment
  namespace: kube-system
spec:
# DO NOT INCREASE REPLICA COUNT
  replicas: 1
  selector:
    matchLabels:
      app: k8s-bigip-ctlr-deployment
  template:
    metadata:
      labels:
        app: k8s-bigip-ctlr-deployment
    spec:
      # Name of the Service Account bound to a Cluster Role with the required
      # permissions
      containers:
        - name: k8s-bigip-ctlr
          image: "f5networks/k8s-bigip-ctlr:latest"
          env:
            - name: BIGIP_USERNAME
              valueFrom:
                secretKeyRef:
                # Replace with the name of the Secret containing your login
                # credentials
                  name: bigip-login
                  key: username
            - name: BIGIP_PASSWORD
              valueFrom:
                secretKeyRef:
                # Replace with the name of the Secret containing your login
                # credentials
                  name: bigip-login
                  key: password
          command: ["/app/bin/k8s-bigip-ctlr"]
          args: [
            # See the k8s-bigip-ctlr documentation for information about
            # all config options
            # https://clouddocs.f5.com/containers/latest/
            "--bigip-username=$(BIGIP_USERNAME)",
            "--bigip-password=$(BIGIP_PASSWORD)",
            "--bigip-url=<ip_address-or-hostname>",
            "--bigip-partition=<name_of_partition>",
            "--pool-member-type=nodeport",
            "--insecure",
            ]
      serviceAccountName: bigip-ctlr
' > k8s_cis_deployment.yaml
```

```yaml
# for use in k8s clusters only
 kind: ClusterRole
 apiVersion: rbac.authorization.k8s.io/v1
 metadata:
   name: bigip-ctlr-clusterrole
 rules:
 - apiGroups: ["", "extensions", "networking.k8s.io"]
   resources: ["nodes", "services", "endpoints", "namespaces", "ingresses", "pods", "ingressclasses"]
   verbs: ["get", "list", "watch"]
 - apiGroups: ["", "extensions", "networking.k8s.io"]
   resources: ["configmaps", "events", "ingresses/status", "services/status"]
   verbs: ["get", "list", "watch", "update", "create", "patch"]
 - apiGroups: ["cis.f5.com"]
   resources: ["virtualservers","virtualservers/status", "tlsprofiles", "transportservers", "ingresslinks", "externaldnss"]
   verbs: ["get", "list", "watch", "update", "patch"]
 - apiGroups: ["fic.f5.com"]
   resources: ["f5ipams", "f5ipams/status"]
   verbs: ["get", "list", "watch", "update", "create", "patch", "delete"]
 - apiGroups: ["apiextensions.k8s.io"]
   resources: ["customresourcedefinitions"]
   verbs: ["get", "list", "watch", "update", "create", "patch"]
 - apiGroups: ["", "extensions"]
   resources: ["secrets"]
   verbs: ["get", "list", "watch"]
 ---
 kind: ClusterRoleBinding
 apiVersion: rbac.authorization.k8s.io/v1
 metadata:
   name: bigip-ctlr-clusterrole-binding
   namespace: kube-system
 roleRef:
   apiGroup: rbac.authorization.k8s.io
   kind: ClusterRole
   name: bigip-ctlr-clusterrole
 subjects:
 - apiGroup: ""
   kind: ServiceAccount
   name: bigip-ctlr
   namespace: kube-system
```

```bash
kubectl create secret generic bigip-login -n kube-system --from-literal=username=itian --from-literal=password=itian.co.kr
```

```bash
kubectl create serviceaccount bigip-ctlr -n kube-system
```

6.2 BIG-IP System

6.2.1 AS3(3.18+) rpm Install
F5Network Github에서 다운로드하여 iApps › Package Management LX 메뉴에서 Import하여 설치
[링크](https://github.com/F5Networks/f5-appsvcs-extension/releases)

설치 확인 : restcurl -u admin:itian.co.kr /mgmt/shared/appsvcs/info

6.2.2 BIG-IP Partition for Kubernetes – 전용 파티션 생성
tmsh create auth partition CIS

6.2.3 관리자 권한의 계정
create auth user itian shell bash encrypted-password itian.co.kr partition-access add { all-partitions { role admin } }

7 Calico 연동
Route Domain에서 BGP 활성화 및 Port Lockdown 설정에서 tcp/179 추가
Kubernetes Cluster에서 Calicoctl 설치 및 BGP 연동 작업
BIG-IP에서 imish를 통해 BGP 연동 작업

[링크](https://clouddocs.f5.com/containers/latest/userguide/calico-config.html)

```bash
cat << EOF | calicoctl create -f -
apiVersion: projectcalico.org/v3
kind: BGPConfiguration
metadata:
  name: default
spec:
  logSeverityScreen: Info
  nodeToNodeMeshEnabled: true
  asNumber: 64512
EOF

cat << EOF | calicoctl create -f -
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: bgppeer-global-bigip1
spec:
  peerIP: 10.1.20.11
  asNumber: 64512
EOF

cat << EOF | calicoctl create -f -
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: bgppeer-global-bigip2
spec:
  peerIP: 10.1.20.12
  asNumber: 64512
EOF
```
