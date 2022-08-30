
# Arcadia and NGINX Controller Configuration

Arcadia Finance Application 설치 및 NGINX Controller ADC, APIM Config Hands-on 예제입니다.

- Test Application인 Arcadia Finance는 Docker 설치와 Kubernetes 설치를 각각 안내하며,
  Docker, Kubernetes 환경 자체의 구성 방법은 이 문서에서 다루지 않습니다.
- 이 문서에는 NGINX Controller의 Service Config에 대해 안내하며,
  Instance 추가 같은 Infrastructure 메뉴에 대해서는 다루지 않습니다.

- NGINX Controller는 버전에 큰 관계가 없으나 이 문서에서는 3.18버전을 사용합니다. 리눅스는 CentOS 7을 사용합니다.

## Arcadia Deployment

Arcadia Finance application은 Stock Budget, Money Transfer, Refer Friend 등의 기능을 4 개의 Micro Service 형식으로 구현해놓은 Lab 환경 구축용 어플리케이션입니다.

<u>아래 링크에서 디테일을 확인할 수 있습니다.</u>

- [Arcadia Finance Application Gitlab](https://gitlab.com/arcadia-application) - Arcadia Gitlab
- [f5 NGINX Controller 3.x with BIG-IP Lab](https://rtd-apim-adc-controller.docs.emea.f5se.com/en/latest/index.html) - BIG-IP와 NGINX Controller의 연동 (Arcadia App 예시)

<u>각 마이크로서비스의 상세는 아래와 같습니다.</u>

![Arcadia_Micro_Services_architecture](../imgs/Arcadia_Micro_Services_architecture.png)

| App     | URI    | Container Port : Host Port |
| ------- | ------ | -------------------------- |
| Main    | /      | 80 : 30511                 |
| Backend | /files | 80 : 31584                 |
| App2    | /api   | 80 : 30362                 |
| App3    | /app3  | 80 : 31662                 |

Host Port는 변경되어도 무방하지만, Gitlab에서 제공하는 Kubernetes Material YAML과 동일하도록 작성했습니다.

### Arcadia (Docker)

Arcadia Gitlab에서는 NGINX OSS까지 설치하도록 안내하고 있지만, NGINX Controller 및 NGINX API Gateway가 그 부분을 대체하도록 하기 위해 NGINX OSS는 설치하지 않으며, 각각의 마이크로 서비스가 별도 포트로 동작하도록 합니다.

```bash
docker network create internal

docker run -dit --net=internal -h mainapp --name=mainapp -p 30511:80 \
  registry.gitlab.com/arcadia-application/main-app/mainapp:latest

docker run -dit --net=internal -h backend --name=backend -p 31584:80 \
  registry.gitlab.com/arcadia-application/back-end/backend:latest

docker run -dit --net=internal -h app2 --name=app2 -p 30362:80 \
  registry.gitlab.com/arcadia-application/app2/app2:latest

docker run -dit --net=internal -h app3 --name=app3 -p 31662:80 \
  registry.gitlab.com/arcadia-application/app3/app3:latest
```

- Docker는 Service Discovery를 위한 Embedded DNS(127.0.0.11)을 갖고 있으며, Arcadia 각 Container Image 내부의 php 스크립트가 Embedded DNS를 사용합니다. 아래 명령어로 확인할 수 있습니다.

  ```bash
  docker exec mainapp ping app3
  ```

- 즉, Container의 name을 기준으로 참조 및 API Query 하기 때문에 Network 생성 및 `--net=internal` 플래그 지정이 반드시 필요합니다.

- 오류가 있거나 기타 Container 재시작이 필요한 경우 아래 커맨드를 사용합니다.

  ```bash
  docker stop `docker ps -a | grep -i gitlab | gawk '{ print $1 }'`
  docker rm `docker ps -a | grep -i gitlab | gawk '{ print $1 }'`
  ```

### Arcadia (Kubernetes)

Kubernetes에 Arcadia를 배포하는 방법은 간단합니다.
아래 명령어로 yaml 파일 자체를 Gitlab에서 그대로 따올 수 있습니다.

```bash
kubectl apply -f https://gitlab.com/arcadia-application/kubernetes-materials/-/blob/master/kubernetes_deployments/arcadia-all.yaml
```

참고로 YAML 파일의 자세한 내용은 아래와 같습니다.

```bash
#########################################################################################
# FILES - BACKEND
#########################################################################################
apiVersion: v1
kind: Service
metadata:
  name: backend
  labels:
    app: backend
    service: backend
spec:
  type: NodePort
  ports:
  - port: 80
    nodePort: 31584
    name: backend-80
  selector:
    app: backend
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: default
  labels:
    app: backend
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
      version: v1
  template:
    metadata:
      labels:
        app: backend
        version: v1
    spec:
      containers:
      - env:
        - name: service_name
          value: backend
        image: registry.gitlab.com/arcadia-application/back-end/backend:latest
        imagePullPolicy: IfNotPresent
        name: backend
        ports:
        - containerPort: 80
          protocol: TCP
---
#########################################################################################
# MAIN
#########################################################################################
apiVersion: v1
kind: Service
metadata:
  name: main
  namespace: default
  labels:
    app: main
    service: main
spec:
  type: NodePort
  ports:
  - name: main-80
    nodePort: 30511
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: main
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: main
  namespace: default
  labels:
    app: main
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: main
      version: v1
  template:
    metadata:
      labels:
        app: main
        version: v1
    spec:
      containers:
      - env:
        - name: service_name
          value: main
        image: registry.gitlab.com/arcadia-application/main-app/mainapp:latest
        imagePullPolicy: IfNotPresent
        name: main
        ports:
        - containerPort: 80
          protocol: TCP
---
#########################################################################################
# APP2
#########################################################################################
apiVersion: v1
kind: Service
metadata:
  name: app2
  namespace: default
  labels:
    app: app2
    service: app2
spec:
  type: NodePort
  ports:
  - port: 80
    name: app2-80
    nodePort: 30362
  selector:
    app: app2
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app2
  namespace: default
  labels:
    app: app2
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app2
      version: v1
  template:
    metadata:
      labels:
        app: app2
        version: v1
    spec:
      containers:
      - env:
        - name: service_name
          value: app2
        image: registry.gitlab.com/arcadia-application/app2/app2:latest
        imagePullPolicy: IfNotPresent
        name: app2
        ports:
        - containerPort: 80
          protocol: TCP
---
#########################################################################################
# APP3
#########################################################################################
apiVersion: v1
kind: Service
metadata:
  name: app3
  namespace: default
  labels:
    app: app3
    service: app3
spec:
  type: NodePort
  ports:
  - port: 80
    name: app3-80
    nodePort: 31662
  selector:
    app: app3
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app3
  namespace: default
  labels:
    app: app3
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app3
      version: v1
  template:
    metadata:
      labels:
        app: app3
        version: v1
    spec:
      containers:
      - env:
        - name: service_name
          value: app3
        image: registry.gitlab.com/arcadia-application/app3/app3:latest
        imagePullPolicy: IfNotPresent
        name: app3
        ports:
        - containerPort: 80
          protocol: TCP
---
```

## NGINX Controller ADC Configuration

이번 단계에서는 NGINX Controller 설정을 통해 API GW로 아래와 같이 Config를 수행합니다.

![topology](../imgs/topology.png)

Main App과 Backend는 PHP 스크립트 상, Embedded DNS를 사용하도록 URI가 구성되어, 직접 통신되어야 하므로, 실제 NGINX Controller에서 Config를 수행할 내용은, App2와 App3입니다.

### 1. Environment

가장 먼저 Environment를 생성합니다.
![step1_environment](../imgs/step1_env.png)
NGINX Controller의 Environment는 Kubernetes의 Namespace와 유사한 개념으로, 서로 다른 Environment에 속한 요소들은 서로를 참조할 수 없습니다.

### 2. Gateway

다음으로는 Gateway를 생성합니다.
Instance를 직접 참조하는 Static한 방식과, Instance Group을 참조하여 해당 Instance Group에 속하는 모든 Instance에 같은 Config를 Push하는 Dynamic한 방식을 선택할 수 있습니다.

이 테스트에서는 Static하게 Instance를 선택합니다.
![step2_gateway](../imgs/step2_gw_1.png)
![step2_gateway](../imgs/step2_gw_2.png)
![step2_gateway](../imgs/step2_gw_3.png)

Gateway에는 접근 가능한 FQDN이 필요합니다.
이 Config는 NGINX Config에서 server_name Directive로 지정됩니다.

### 3. App

다음으로는 Component가 속하게 될 App을 생성합니다.
![step3_App](../imgs/step3_app.png)

Service 구성 요소 중, 지금 생성한 App은 Components의 컨테이너 역할을 수행하며, App의 생성 자체만으로는 아무런 트래픽 제어 효과가 없습니다.
실제 트래픽 제어 및 각종 Config는 Component에서 수행하게 됩니다.

### 4. App - Component ( Main app, /)

![step4_ADC_Component_main](../imgs/step4_comp_mainapp_1.png)
![step4_ADC_Component_main](../imgs/step4_comp_mainapp_2.png)![step4_ADC_Component_main](../imgs/step4_comp_mainapp_3.png)
![step4_ADC_Component_main](../imgs/step4_comp_mainapp_4.png)
URI는 상대경로Relative Path, 절대경로Absolute Path 중 무엇을 사용해도 괜찮습니다.
테스트에서는 상대 경로를 사용합니다.
이 Config는 NGINX Config에서 Location Directive로 지정됩니다.

### 5. App - Component ( App2, /api )

![step5_ADC_Component_app2](../imgs/step5_comp_app2_1.png)
![step5_ADC_Component_app2](../imgs/step5_comp_app2_2.png)
![step5_ADC_Component_app2](../imgs/step5_comp_app2_3.png)
![step5_ADC_Component_app2](../imgs/step5_comp_app2_4.png)

### 6. App - Component ( App3, /app3 )

![step6_ADC_Component_app3](../imgs/step6_comp_app3_1.png)
![step6_ADC_Component_app3](../imgs/step6_comp_app3_2.png)
![step6_ADC_Component_app3](../imgs/step6_comp_app3_3.png)
![step6_ADC_Component_app3](../imgs/step6_comp_app3_4.png)

## NGINX Controller APIM Configuration

NGINX Controller의 Component는 App Component, API Component로 구분됩니다.

Arcadia Finance Application은 OAS3 포맷으로 API Spec을 제공합니다.

원문 OAS3 포맷의 YAML 파일은 아래와 같습니다.
출처는 [Swagger App - Arcadia-OAS3 Schema](https://app.swaggerhub.com/apis/F5EMEASSA/Arcadia-OAS3/2.0.1-schema)입니다.

**Arcadia API Specification - OAS3**

```yaml
openapi: 3.0.3
info:
  description: Arcadia OpenAPI
  title: API Arcadia Finance
  version: 2.0.1-schema
paths:
  /trading/rest/buy_stocks.php:
    post:
      summary: Add stocks to your portfolio
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/buy'
            example:
              trans_value: '312'
              qty: '16'
              company: MSFT
              action: buy
              stock_price: '198'
      responses:
        '200':
          description: 200 response
          content:
            application/json:
              example:
                status: success
                name: Microsoft
                qty: '16'
                amount: '312'
                transid: '855415223'
                
  /trading/rest/sell_stocks.php:
    post:
      summary: Sell stocks that you own
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/sell'
            example:
              trans_value: '212'
              qty: '16'
              company: MSFT
              action: sell
              stock_price: '158'
      responses:
        '200':
          description: 200 response
          content:
            application/json:
              example:
                status: success
                name: Microsoft
                qty: '16'
                amount: '212'
                transid: '658854124'

  /trading/transactions.php:
    get:
      summary: Get the latests transactions that have happened
      responses:
        '200':
          description: 200 response
          content:
            application/json:
              example:
                YourLastTransaction: MFST 2000

  /api/rest/execute_money_transfer.php:
    post:
      summary: Transfer money to a friend
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/money_transfer'
            example:
              amount: '92'
              account: '2075894'
              currency: GBP
              friend: Vincent
      responses:
        '200':
          description: 200 response
          content:
            application/json:
              example:
                name: Vincent
                status: success
                currency: GBP
                transid: '524569855'
                msg: The money transfer has been successfully completed


components:
  schemas:
    buy:
      type: object
      required:
        - trans_value
        - qty
        - company
        - action
        - stock_price
      properties:
        trans_value:
          type: integer
          format: int64
        qty:
          type: integer
          format: int64
        company:
          type: string
        action:
          type: string          
        stock_price:
          type: integer
          format: int64
    sell:
      type: object
      required:
        - trans_value
        - qty
        - company
        - action
        - stock_price
      properties:
        trans_value:
          type: integer
          format: int64
        qty:
          type: integer
          format: int64
        company:
          type: string
        action:
          type: string          
        stock_price:
          type: integer
          format: int64
    money_transfer:
      type: object
      required:
        - amount
        - account
        - currency
        - friend
      properties:
        amount:
          type: integer
          format: int64
        account:
          type: integer
          format: int64
        currency:
          type: string
        friend:
          type: string
```

### API Definition

API Definition은 NGINX Controller ADC 부분에서 App이 Component들의 컨테이너 역할을 한 것처럼, API Version을 담는 Container라고 생각하면 되겠습니다.

- API Definition은 Environment에 종속되지 않습니다.
  이는 MSA를 가정했을 때, 서로 다른 앱에서 필요한 유사한 기능을 동일한 API로 구현할 수 있도록 하기 위한 부분으로 보입니다.

### API Version

API Version은 말그대로 특정 API Definition의 Version에 대한 상세를 작성하는 부분으로 이해하면 되겠습니다.

OAS3 포맷으로 제공한 YAML 파일도 이 API Version 생성 과정에서 반영하게 됩니다.

만일 API Version의 상세에 대한 YAML 파일이 없다면, 직접 작성(Configure Manually)해도 무방합니다.

- API Version은 API Definition에 속하는 요소로서, API Definition의 특징과 비슷하게 Environment에 속하지 않습니다.

앞서 API Definition을 App - Component 관계와 비교해 설명했지만, 그와는 조금 다르게 API Version은 실제 트래픽을 제어하는 Config에 직접적인 영향을 주지 않고 한 단계가 더 있습니다.

이 다음 과정에서 생성할 Published API 부분에서, 특정 API Version을 Publish하는 과정을 거쳐 Component를 연결하게 되면 그제서야 NGINX Config로서 실제 트래픽 제어에 관여하게 됩니다.

### Published API

Published API는 앞선 과정에서 작성한 API Definition의 특정 Version을 실제 Component와 연결하는 작업으로 실제 트래픽 Config와 직접적인 관계가 있는 부분입니다.
