---
title:  "F5 BIG-IP CIS 소개"
author: DDAONG
date:   2022-08-03
category: "BIG-IP"
layout: post
---

# F5 BIG-IP Container Ingress Service 소개

#### Container Ingress Service(CIS)란

+ Kubernetes 환경에서는 내부 워크로드 서비스의 노출을 위해, Ingress Controller를 사용합니다.
+ F5 Container Ingress Service는 [Kubernetes.io의 Ingress Controller 문서](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/#additional-controllers)에 리스트업 되어 있는 Ingress Controller입니다.

#### Ingress Controller의 요건

Ingress Controller는 기본적으로 클러스터 내 다른 Service 개체들을 대표하는 Service로서, Proxy 동작을 수행하며 Kubernetes 안의 Workload 및 Ingress 등의 설정을 읽어 자기 자신의 Config에 반영하는 동작이 필요합니다.
대표적 Ingress Controller인 NGINX Ingress Controller를 기준으로 설명해보자면, NGINX Ingress Controller는 Kubernetes Cluster 내부에 NGINX Ingress Controller Pod가 존재하고, 그 Service가 다른 서비스를 대표해 외부로 노출됩니다.

<p align="center"><img src="https://github.com/dddaong/dddaong.github.io/raw/master/_posts/imgs/2022-08-03-BIG-IP-CIS-Calico01.png"/></p>

NGINX Ingress Controller는 Ingress 개체 설정에 따라 자동으로 NGINX Config로 반영합니다. 사용자가 수동으로 NGINX Config를 수정할 필요가 없습니다.
유입되는 실제 서비스 트래픽은 자동 생성된 NGINX Config에 따라 L7 Routing되어 다른 서비스로 프록시 됩니다.

<p align="center"><img src="https://github.com/dddaong/dddaong.github.io/raw/master/_posts/imgs/2022-08-03-BIG-IP-CIS-Calico02.png"/></p>

한편, BIG-IP Device는 기본적으로 Proxy 동작을 수행하기 때문에, Kubernetes Workload의 상태만 실시간으로 TMOS Config로 반영할 수 있다면, BIG-IP Device를 Ingress Controller로 사용할 수 있습니다.
F5 Container Ingress Service는 이 Kubernetes의 상태가 Config로 반영되도록 해주는 어플리케이션으로서, BIG-IP LTM을 Ingress Controller로 사용할 수 있도록 하는 일종의 미들웨어이자 Kubernetes Cluster를 대신해 TMOS와 통신해주는 인터페이스 역할의 어플리케이션입니다.
이 CIS는 TMOS에서 제공하는 F5 REST API를 통해 Workload 상태를 Config로 반영할 수 있도록 합니다.

#### F5 REST API Scheme - AS3 Extension의 사용

사용자가 직접 수동으로 F5 REST API에서 어떤 서비스를 설정한다고 가정하면, Pool 생성, Profile 생성, Virtual Server 생성 등 각 개체마다 여러 번의 API Call이 필요합니다.
또한 이미 수행한 API Call을 다시 요청하는 경우, Error 메시지를 반환하기도 하고, Kubernetes Workload를 Config로 반영하려면 기존의 REST API 스키마로는 비효율적이며 Object에 따라 사용하기 복잡한 부분이 있습니다.

그래서, CIS는 AS3 Extension이라는 F5 REST API의 확장 규격을 사용합니다.
F5 Service 설정에 필요한 모든 것을 하나의 Request에서 작성하여 Application의 전체 Config를 하나의 API Request에서 구성할 수 있도록 해줍니다.

또한 API Request가 명령형이 아닌, 선언형으로서 AS3 Extension은 Declarative(선언형) Model API Scheme을 사용해, 목표 구성을 작성하면, 그에 따라 알아서 Config를 완성하는 식으로 동작합니다. 이 방식을 통해 한 번의 API Call로 TMOS Config를 완성할 수 있도록 합니다.

+ Imperative(명령형) vs Declarative(선언형)
명령형과 선언형을 간략하게 설명하자면, 명령형Imperative은 서비스를 구성하기 위해 무엇을 할지를 직접 명령하는 방식이고, 선언형은 원하는 최종 설정을 지정하면 시스템이 그에 맞는 방법을 확인하여 수행하는 형식입니다.
좀더 구체적으로는 “What to DO”와 “What to BE”의 차이이며, 기존 REST API의 동작인 Pool 생성, Profile 생성, Policy Rule 생성, Virtual Server 생성 등 각 단계마다 동작을 지정하는 방법이 명령형Imperative model API Scheme이며, AS3는 선언형Declarative model API Scheme로서, 원하는 목표 구성을 작성하면 실제 Config 동작은 BIG-IP에서 이뤄지는 형식으로 동작합니다.

#### CIS를 사용하는 이유

Kubernetes 클러스터 내부에서 Pod로서 동작하는 여타 Ingress Controller와 Container Ingress Service와 비교하자면, CIS가 하드웨어를 사용하여 얻을 수 있는 가속의 장점, 설정에 따라 Pod까지 직접 도달할 수 있어 Connection 제어 및 다양한 Persistence의 사용, 비교적 간편한 VIP 할당 및 BIG-IP의 기능 전반을 활용할 수 있는 이점이 있습니다.

#### CIS의 Deployment options

##### Nodeport

NodePort 모드는 Kubernetes Cluster Overlay Network와 연동을 하지 않는 설치 방식입니다.
Overlay Network와 연동 과정이 필요 없기 때문에 설치가 가장 간단하며, 때문에 BIG-IP의 Pool member가 Kubernetes Node IP + Node Port가 됩니다. Node IP를 Pool member로 하기 때문에 Pod Scale in/out 등에 영향을 받지 않고, 따라서Pool Member의 변화가 거의 없습니다.
BIG-IP가 Pod까지 직접 도달하지 못하고, Service를 통해 Pod에 접근하기 때문에 Pod에 대해 Cluster 내에서 어떤 경로로 트래픽이 전달되었는지, 단일 Pod가 갖고 있는 Connection의 숫자, LB 상태 등 가시성 확보가 어렵습니다.
Pod에 대한 Load balance의 주체는 Kubernetes Service인 만큼 Kube-Proxy 또는 구성에 따라 IPVS를 통해 Pod에 대해 한번 더 LB가 이뤄지게 되며, 복잡한 iptables 테이블 때문에 약간의 성능 저하가 있을 수도 있습니다.

<p align="center"><img src="https://github.com/dddaong/dddaong.github.io/raw/master/_posts/imgs/2022-08-03-BIG-IP-CIS-Calico03.png"/></p>

##### ClusterIP

ClusterIP 모드는 BIG-IP 장비가 Kubernetes Cluster Overlay Network에 접근할 수 있도록 하는 설치 방식입니다.
BIG-IP가 Pod에 직접 접근이 가능하기 때문에 BIG-IP의 Pool member가 Pod의 IP, Port가 됩니다. Pod Scale-in/out에 Pool Member config가 직접 영향을 받으며, 비교적 명확한 가시성을 확보할 수 있습니다.
Flannel CNI와의 연동을 위해 VXLAN을 사용하거나, Calico 연동을 위해 BGP Peer 설정을 수행하거나 하는 과정이 필요합니다. (Calico CNI와 BGP 연동 시 Dynamic Routing License가 필요합니다.)
<p align="center"><img src="https://github.com/dddaong/dddaong.github.io/raw/master/_posts/imgs/2022-08-03-BIG-IP-CIS-Calico04.png"/></p>

##### NodePortLocal

Antrea CNI의 NodePortLocal 기능을 사용하는 모드로서, Antrea Agent가 제공하는 포트맵을 사용해 Kube-Proxy를 바이패스할 수 있도록 합니다. NodePort 모드와 유사하게 Kubernetes Cluster Overlay Network와 연동을 하지 않는 설치 방식입니다.
Service 개체의 Type 항목으로 NodePort를 사용하지 않고, Pod 또는 Service에 작성한 ‘NodePortLocal’ Annotation에 따라 BIG-IP로 반영됩니다. 따라서 Load Balancing의 대상인 Pod가 존재하는 Node의 IP, Port가 Pool Member가 되며,
이 정보는 Antrea Agent가 제공하는 NodePort 모드에 비해 가시성이 어느정도 확보된다고 볼 수 있습니다.

<p align="center"><img src="https://github.com/dddaong/dddaong.github.io/raw/master/_posts/imgs/2022-08-03-BIG-IP-CIS-Calico05.png"/></p>

[링크] <https://antrea.io/docs/main/>
