---
title: "Overview Lightweight Kubernetes"
author: DDAONG
---

## Lightweight Kubernetes Distribution

### [K3S](https://k3s.io/)

<img title="" src="https://cncf-branding.netlify.app/img/projects/k3s/horizontal/color/k3s-horizontal-color.png" alt="K3S_logo" width="480">

Rancher Labs의 Rancher Project로 시작한 K3S는, 2020년 SUSE에 인수되고, SUSE는 같은 해 Linux Foundation에 K3S 프로젝트를 기부해 K3S가 Opensource로 유지되도록 하겠다는 약속을 지켰습니다.

이런 연유로 K3S는 공식 CNCF Sandbox Project가 되어 개발/유지되고 있습니다.

K3S는 IoT, Edge Computing 등을 목적으로 하는 초경량의 Kubernetes 배포판입니다.

### [MicroK8S](https://microk8s.io/)

<img title="" src="https://image.chitra.live/api/v1/wps/9c5ec3e/a66e0099-aacd-443d-8918-9ca4a7ec7442/1/microk8s-580x358.png" alt="MicroK8S_Logo" width="481" data-align="inline">

Ubuntu Linux를 개발하는 Canonical의 Kubernetes팀에 의해 개발된 MicroK8s는 Ubuntu의 Snaps를 통해 간편하게 설치할 수 있습니다.

LXD, juju 등과 함께 Canonical의 클라우스 생태계를 꾸려가고 있습니다.

### [MiniKube](https://minikube.sigs.k8s.io/)

<img title="" src="https://github.com/kubernetes/minikube/raw/master/images/logo/logo.png" alt="" width="233" data-align="inline">

Kubernetes 공식 홈페이지 Tutorial에도 등장하는 Minikube는 Kubernetes SIG(Special Interested Group)의 개발로 만들어진 최초의 경량 Kubernetes입니다. (2016년에 최초 Release)

개인적으로 minikube는 공부 및 테스트 목적의 Kubernetes라는 이미지가 강하고, 실제로 minikube의 목적이 Local Kubernetes 앱 개발 환경 구성 및 Kubernetes의 모든 기능 지원이라고 밝힌 바 있습니다.


### [K0S](https://k0sproject.io/)

<img src="https://docs.k0sproject.io/v1.23.3+k0s.0/img/k0s-logo-full-color-light.svg" title="" alt="" width="322">

Docker enterprise business를 인수한 Mirantis에서 만든 초경량 배포판입니다.

K3s와 종종 비교되는데, K3s에 비해 더 경량화되었다고 하긴 어려워 보입니다.


### Compare Lightweight Kubernetes Distros

|      | K3S               | MicroK8S         | Minikube                | K0S                 |
| --- | ----------------- | ---------------- | ----------------------- | ------------------- |
| CPU | 1                 | 2                | 2                       | 1                   |
| RAM | 0.5 GB            | 2 GB             | 1 GB                    | 1 GB                |
| CNI | Flannel           | Calico, Cilium   | Calico, Cilium, Flannel | kube-router, Calico |
| CRI | containerd, CRI-O | containerd, kata | containerd, CRI-O       | containerd          |



### K3s를 선택한 이유

일반적인 경량 Kubernetes Distro의 사용 목적, 리소스 제약, 편의성(친밀도)를 기준으로 선택하게 됩니다.

Kubernetes 기반으로 Home Server를 구현할 리소스가 노트북이라 업그레이드가 쉽지 않고, 심지어 구형이기 때문에 RAM 확장이 절대 불가능할 것으로 생각되어 리소스 제약을 기준으로 K3s를 선택했습니다.

