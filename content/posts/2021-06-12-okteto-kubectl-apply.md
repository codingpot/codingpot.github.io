---
title: "Okteto에 gRPC용 Deployment, Service, Ingress 설정"
date: 2021-06-11
tags:
    - okteto
    - kubectl
---

> 코딩냄비 프로젝트 중 **pr12er**는 TensorFlow Korea의 논문을 읽고/리뷰하는 모임 PR12에서 촬영된 동영상을 큐레이션하는 프로젝트입니다. 개략적으로 프론트엔드는 **Flutter**, 백엔드는 **GO**로 작성되었으며, 이 둘간의 인터페이스는 **gRPC/protobuf**로 구성되어있습니다. 특히 **pr12er** 프로젝트의 백엔드 서버는 **PR**이 **Merge** 됨과 동시에 **Okteto**가 제공하는**k8s** 에 배포되는 **CD** 루틴을 탑니다.

이 글은 **Okteto** 에 배포하기위한 파이프라인을 분석하는 총 X 편의 시리즈물 중 두 번째입니다.

1. [**Okteto** 파이프라인 개요, **okteto build**, **pr12er** 서버용 **Dockerfile** 분석](https://codingpot.github.io/cicd/okteto-pipeline-build/)
2. [**Okteto**에 gRPC용 **Deployment**, ****Service****, **Ingress** 설정](https://codingpot.github.io/cicd/okteto-kubectl-apply/)
3. ................

# **pr12er**에 적용된 **Okteto** 파이프라인 명세서

```yaml
deploy:
#  - okteto build -t okteto.dev/codingpot-pr12er-server:${OKTETO_GIT_COMMIT} -f ./server/deploy/Dockerfile server
  - for file in k8s/kkweon-okteto/*.yaml; do envsubst < $file | kubectl apply -f -; done
#  - kubectl set image deployment server server=okteto.dev/codingpot-pr12er-server:${OKTETO_GIT_COMMIT}
#  - kubectl rollout restart deployment grafana-agent
#  - kubectl rollout status deployment server && kubectl rollout status deployment grafana-agent
```

## 어떤 **yaml** 파일들이 존재하나?

```shell
k8s/kkweon-okteto/
├── ingress.yaml
└── server.yaml
```

현재 **/k8s/kkweon-okteto/** 디렉터리 내부에는 보다 다양한 **yaml** 파일들이 존재하지만, **gRPC** 서버를 운용하기위한 최소한의 조건은 k8s의 **Deployment**, ****Service****, **Ingress** 객체를 정의해둔 **ingress.yaml** 및 **server.yaml** 두 파일만 참조하면 기본은 이해할 수 있습니다. 우선 각 파일의 세부 사항과 서로 엮인 관계를 분석해보죠.

![](./images/ingress-**Service**-deployment.png)

## Ingress
- 서버 애플리케이션이 위치한 쿠버네티스 플랫폼은 외부로부터 차단된 고유한 영역을 가집니다. 하지만 사용자는 쿠버네티스 플랫폼 외부에  존재하는것이 보통이며, 이들은 쿠버네티스 자원으로 구축된 서버에 접근할 필요가 있습니다. **Ingress**는 쿠버네티스 외부와 내부를 엮어주는 일종의 브릿지 역할을 하는 객체입니다. 

### Ingress Annotations
[**dev.okteto.com/generate-host: "true"**](dev.okteto.com/generate-host)
- **Okteto** 에서 자동으로 호스트 이름을 할당하는것을 허용하고자 할 때 사용되는 애노테이션입니다. **Okteto** **Service**를 활용한다면 거의 반드시 설정해줘야 하므로, **true**로 설정하였습니다.

[**kubernetes.io/ingress.class: "nginx"**](https://www.nginx.com/resources/glossary/kubernetes-ingress-controller/)
- 쿠버네티스의 Ingress 컨트롤러를 nginx로 지정한다는 의미입니다.
- Ingress 컨트롤러의 역할에는 외부 트래픽을 수용하여 내부 Pod 들로 로드밸런싱하기, 클러스터 밖의 다른 **Service**와 소통이 필요한 트래픽 관리, Pods들을 모니터링하여 추가/제거에 따른 로드밸런싱 규칙 갱신 등이 있습니다. 
- 쿠버네티스가 공식적으로 지원하는 Ingress 컨트롤러로는 **AWS**, **GCE**, **nginx**가 있으며, 그 외의 써드파티에서 나온 Ingress 컨트롤러도 종류가 다양하니 요구사항에 맞는것을 찾아서 사용하면 됩니다 [(목록)](https://kubernetes.io/docs/concepts/**Service**s-networking/ingress-controllers/).

[**nginx.ingress.kubernetes.io/backend-protocol: "GRPC"**](https://kubernetes.github.io/ingress-nginx/examples/grpc/#grpc)
> This is the magic ingredient that sets up the appropriate nginx configuration to route http/2 traffic to our **Service**.
- 위는 공식문서의 설명입니다. 즉 nginx의 설정을 건드려서, **HTTP/2** 트래픽이 쿠버네티스 내부 **Service**로 들어올 수 있도록 해줍니다. gRPC를 사용하려면 **HTTP/2** 프로토콜이 반드시 필요하므로, 이 설정을 반드시 해줘야 합니다.

[**nginx.ingress.kubernetes.io/ssl-redirect: "true"**](https://kubernetes.github.io/ingress-nginx/examples/rewrite/)
> Indicates if the location section is accessible SSL only (defaults to True when Ingress contains a Certificate)
- 위는 공식문서의 설명입니다. SSL로만 접근 가능함을 명시합니다(디폴트는 **true** 인데, 여기서는 명시적으로 **true** 라고 적어줌). gRPC를 사용하려면 SSL이 반드시 필요하므로, 이 설정 또한 반드시 해줘야 합니다.

### Ingress Spec

**Ingress** 스펙은 일련의 규칙을 정의합니다. 특히 **backend** 라는 필드와, 그 이전까지의 부분들로 나누어 생각할 수 있습니다. **backend**는 말 그대로 **Ingress**를 통해 도달한 트래픽이 도달해야 할 쿠버네티스 내부의 위치를 의미합니다. 따라서 **backend**는 쿠버네티스 내부에서 목적지를 찾아가기 위한 규칙을 정의하는 필드이며, **backend** 이전까지의 필드들은 수용할 외부 트래픽의 형식을 판단하는 규칙을 정의합니다. 

```yaml
spec:
  rules:
    - host: this-name-does-not-matter
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              Service:
                name: server
                port:
                  name: grpc
```

**pr12er** 프로젝트에서 정의한 **Ingress** 스펙(규칙)을 살펴보죠.
1. **rules**에는 규칙 목록을 나열합니다.
2. 가장 먼저 **host** 이름을 정의합니다. 이곳에는 보통 호스트명이 오지만, **Okteto**의 ****dev.okteto.com/generate-host: "true"**** 애노테이션을 사용했기 때문에, **Okteto**가 내부적으로 할당합니다. 따라서 무슨 값이든 상관 없습니다.
3. 그 다음 경로를 지정합니다. **path: /** 와 **pathType: Prefix** 가 함께 쓰이면, 모든 경로의 트래픽을 수용합니다 [(참고자료)](https://kubernetes.io/docs/concepts/**Service**s-networking/ingress/#path-types). 
4. 이제 **backend**가 등장합니다. 즉 **Okteto**가 내부적으로 할당한 호스트명에 대해 모든 경로로 접근하는 트래픽은 **backend**에서 정의된 내부 목적지로 포워딩 되는것이죠.
5. **Service.name: server**는 들어온 외부 트래픽이 도달할 **Service** 이름입니다. 여기서는 **server** 이므로, 이 **Ingress**를 거쳐 들어온 외부 트래픽이 포워딩될 목적지를 파악하려면 **server** 라는 이름을 가진 **Service**를 찾아봐야 한다고 해석될 수 있습니다. **Service**에 대한 부분은 잠시 후 아래에서 확인하겠습니다. 
6. **Service.port.name: grpc**는 들어온 외부 트래픽이 내부에서 전달될 포트명을 명시합니다. ****Service**.port.number**로 직접 포트 번호를 명시할 수도 있지만, **Service.port.name**을 활용하면 특정 프로토콜에 이미 예약된 포트번호를 할당하는것도 가능합니다. 여기서는 **grpc**로 명시합니다.

## Service

**Service**는 쿠버내티스 내부에서 트래픽을 원하는 목적지 **Pod**으로 포워딩하는 역할을 하는 객체입니다. **Ingress**가 외부 트래픽을 여러 **Service**로 연결시키는것이 목적이었다면, **Service**는 특정 그룹의 **Pod**들에 대해 트래픽을 연결시켜주는 셈이죠. 

또한 **Service**는 쿠버네티스 내부의 **Pod** 들간의 연결 창구가 되기도 합니다. 나중에 보게 되겠지만, 자원 모니터링을하는 도구인 그라파나 에이전트는 서버로부터 모니터링 할 내용을 폴링/조회 하기위해 **Service**에 정의된 **http-metrics** 라는 이름의 **9093** 포트를 탑니다.

### Service Metadata

```yaml
metadata:
  labels:
    app: server
  name: server
```

**Service** 메타데이터에는 **Service** 객체를 구분할 수 있는 식별자를 기입합니다. 특히 **metadata.name**에 적힌 이름이 바로 **Ingress**의 **backend.Service.name**과 매칭이 되어야만, 해당 **Ingress**를 통해 들어온 외부 트래픽이 해당 **Service**로 연결될 수 있습니다. **pr12er** 프로젝트에서는 **server** 라는 이름을 할당했습니다. 

### Service Spec

```yaml
spec:
  selector:
    app: server
  ports:
    - name: grpc
      port: 9000
      protocol: TCP
      targetPort: grpc
    - name: http-metrics
      port: 9093
      protocol: TCP
      targetPort: http-metrics
```

**Service** 또한 **Ingress** 처럼 인터페이스를 위한 객체이므로, **Service**를 통해서 연결될 애플리케이션이 탑재된 **Pod**을 지정(매핑), 프로토콜, 포트정보와 같은것들이 명시되어야 합니다. 

**selector.app**은 **Service**를 통해서 연결될 **Deployment**를 지정합니다. 아래에서 나오겠지만,  **Deployment** 객체에 명시된 **metadata.labels.app**와 **Service**의 **selector.app**이 서로 매핑되어 있습니다. 그리고 **ports**는 **Service**가 연결을 허용하는 창구의 목록입니다. 

각 창구마다 포트의 종류가 **port** 및 **targetPort** 두 개가 존재하는데, **port**는 클러스터내 애플리케이션에 대해서 노출되는 포트이며, **targetPort**는 **Service**로 도착한 트래픽이 포워딩될 **Pod**의 포트입니다. 여기에 더해서 **NodePort**를 추가적으로 정의할 수 있으며, 이것의 역할은 클러스터간의 통신에 사용 가능한 포트를 정의하는 것입니다. 

**pr12er** 프로젝트에서 정의한 **Service** 스펙을 살펴보죠.
1. **Ingress**를 통해 들어온 트래픽은 **server**라는 이름을 가진 **Service**로 포워딩됩니다.
2. **server** 라는 이름의 **Service**는 두 개의 인터페이스를 외부로 노출시키는 역할을 합니다.
3. **grpc** 인터페이스에는 **TCP** 프로토콜이 사용되며, 해당 **Service**로 들어온 트래픽은 **targetPort:grpc** 포트로 포워딩됩니다. 
   - **port: 9000**은 클러스터 내부의 **Pod**간 통신을 위한 인터페이스입니다.
4. **http-metrics** 인터페이스에도 **TCP** 프로토콜이 사용되며, 해당 **Service**로 들어온 트래픽은 **targetPort:http-metrics** 포트로 포워딩됩니다.
   - **port:9093**은 클러스 내부의 **Pod**간 통신을 위한 인터페이스입니다. 같은 클러스터 내부에 그라파나 에이전트용 애플리케이션이 실행되고 있으며, 해당 애플리케이션은 이 포트를 통해 **pr12er** 서버로부터 모니터링 정보를 긁어가게 됩니다.

## Deployment
**Deployment**는 원하는 배포 상태가 명시된 객체로, **Pod**과 **ReplicaSet**을 함께 사용하여 해당 상태를 정의합니다. **Pod**은 배포하고자 하는 컨테이너에 대한 명세서, **ReplicaSet**은 배포될 **Pod**을 항상 몇 개나 유지할지를 정의한 명세서로 볼 수 있습니다.

### Deployment Metadata

```yaml
metadata:
  labels:
    app: server
  name: server
```

**Deployment** 의 메타데이터도 단순합니다. 단지 해당 **Deployment**를 식별할 수 있는 정보가 기입되며, 이 식별 정보는 다른 쿠버네티스 객체가 참조용으로 사용합니다. 가령 상기 **metadata.labels.app**에 들어간 **server**라는 식별자는 **Service** 객체가 트래픽을 포워딩 할 대상체를 특정하는데 활용됩니다.

### Deployment Spec

**Deployment Spect**은 내용이 길기 때문에 네 개의 파트로 나누어 설명하겠습니다. 

우선 **spec.replicas**는 **Deployment** 에서 유지되어야 할 **Pod**의 개수를 명시합니다. 

그리고 꽤 친숙한 **spec.selector** 라는 필드가 등장합니다. 여기서의 **spec.selector**의 역할은 **Deployment**에서 정의된 **spec.replicas** 라는 규칙이 적용될 **Pod**의 식별자를 찾는것입니다. 

```yaml
spec:
  replicas: 1
  selector:
    matchLabels:
      app: server
  template:
    metadata:
      labels:
        app: server
```

그 다음으로 나타나는 **spec.template**이 바로 **Pod**의 상세한 스펙을 명시하는 필드입니다. 우선 **template.metadata.labels** 은 **Pod**을 식별하기 위한 메타데이터를 명시한 것으로, 직전 **spec.selector.matchLabels**의 값과 일치하는 경우 앞서 정의한 **ReplicaSet** 규칙이 해당 **Pod**에 적용됩니다.

### Deployment Template Spec (1)

이제 **Pod**의 스펙을 살펴볼 차례입니다. **template.spec.containers**에는 **Pod**에서 구동될 컨테이너 목록에 대한 상세한 스펙을 기술합니다. **containers.image**는 실행될 도커 이미지, **containers.imagePullPolicy**는 도커 이미지를 내려받을 정책, **containers.ports**는 컨테이너가 노출할 포트 정보를 담습니다. 

```yaml
spec:
  replicas: 1
  selector:
    # ommited
  template:
    # ommited
    spec:
      containers:
        - image: okteto.dev/codingpot-pr12er-server
          imagePullPolicy: IfNotPresent
          name: server
          ports:
            - containerPort: 9000
              name: grpc
              protocol: TCP
            - containerPort: 9093
              name: http-metrics
              protocol: TCP
```

상기 **pr12er** 프로젝트에서 정의한 **Deployment** 스펙을 말로 풀어서 설명해보죠. **Deployment** 객체는 항상 **1**개의 **Pod**을 유지해야 합니다. 만약 **Pod**이 다운되어 **0**개가 된다면, 자동으로 하나를 구동시켜 **1**를 맞추도록 노력합니다. 

이렇게 구동되는 **Pod** 내부에는 **okteto.dev/codingpot-pr12er-server** 라는 이름의 도커 이미지로부터 인스턴스가 만들어진 컨테이너가 구동됩니다. 만약 해당 이미지가 없다면(**IfNotPresent**) 이미지를 내려받으려는 시도를 하며, 이미지가 이미 있다면 있는 녀석을 사용합니다. 

이 컨테이너는 **grpc**와 **http-metrics** 라는 이름의 두 포트를 노출시킵니다. 두 개 모두 **TCP** 프로토콜을 사용하며, 각각에 대해 노출되는 포트 번호는 **9000**, **9093** 입니다. 

### Deployment Template Spec (2)

**Pod**에 탑재되는 컨테이너는 여러개가 될 수 있으며, 각 컨테이너마다 사용 가능한 자원 정보를 명시해 줄 수 있습니다. 이는 **Pod**이 배포될 환경에 따라 매우 상이하게 설정될 수 있는 정보입니다. 일반적으로 배포될 환경의 공식문서를 참조하거나, `kubectl top node` 와 같은 명령어로 가용 리소스를 체크하여 기입합니다. 

```yaml
spec:
  replicas: 1
  selector:
    # ommited
  template:
    # ommited
    spec:
      containers:
        - image: okteto.dev/codingpot-pr12er-server
            # ommited
          ports:
            # ommited
          resources:
            requests:
              cpu: 10m
              memory: 10m
            limits:
              cpu: 1000m
              memory: 2Gi         
          env:
            - name: PR12ER_GRPC_PORT
              value: "9000"
            - name: PR12ER_PROMETHEUS_PORT
              value: "9093" 
```

**resources.requests**에는 요청될 최소 자원 스펙을, **resources.limits**에는 반드시 넘지 말아야 할 자원의 상한선이 명시됩니다. 또한 **cpu** 필드의 **10m**는 열 개의 밀리코어/밀리CPU 라고 읽히며, 쿠버네티스에서는 하나의 CPU를 1000m 으로 규정하고 있습니다. 따라서 CPU 사용량을 정하는 방식으로 **10m** 이라고 명시하면 1/100 만큼의 컴퓨팅 파워만을 사용한다는 뜻이며, **1000m** 이라고 명시하면 하나의 CPU를 완전히 사용하겠다는 의미입니다. 

추가적으로 **containers.env** 필드는 컨테이너 내부에서 사용될 환경변수를 설정하는 데 쓰입니다. 컨테이너에서 특정 애플리케이션 구동시 필요한 환경변수가 있다면, 여기를 통해서 정의할 수 있습니다. 

### Deployment Template Spec (3)

이제 마지막으로 **containers.readinessProbe**와 **containers.livenessProbe** 필드를 살펴볼 차례입니다. 두 필드는 모두 컨테이너의 건강 상태를 체크하고, 그 상태에 따른 조치를 취하는 작업을 정의하는 데 쓰입니다. 

```yaml
spec:
  replicas: 1
  selector:
    # ommited
  template:
    # ommited
    spec:
      containers:
        - image: okteto.dev/codingpot-pr12er-server
            # ommited
          ports:
            # ommited
          resources:
            # ommited       
          env:
            # ommited

          readinessProbe:
            exec:
              command: ["/bin/grpc_health_probe", "--addr=:9000"]
            initialDelaySeconds: 10

          livenessProbe:
            exec:
              command: ["/bin/grpc_health_probe", "--addr=:9000"]
            initialDelaySeconds: 20
            periodSeconds: 60            
```

**readinessProbe**와 **livenessProbe** 모두 **exec** 라는 하위 필드가 존재하는 데, 이 필드는 조건을 의미한다. 이 조건을 만족할 때만 특정 액션을 취하겠다는 것입니다. 그러면 각각은 어떤 액션을 취할까요?

**readinessProbe**는 정의된 조건을 만족할 때만 트래픽을 수용합니다. 여러가지 상황이 존재할 수 있으며, 일반적으로 거론되는 상황으로는 연동되는 다른 **Pod**이 살아날 때까지, 내부적으로 대규모 처리를 수행중이어서 그 작업이 종료될 때까지와 같은것들이 있습니다. 

**livenessProbe**는 정의된 조건을 만족할 때만 컨테이너를 유지합니다. 만약 여기서 정의된 조건이 실패하면 컨테이너는 삭제되고, 새로운 컨테이너가 구동됩니다. 

두 개 모두 **initialDelaySeconds**라는 하위 필드에서 초기의 상태 체크에 허용하는 지연을, 상태 체크를 얼마만큼 주기적으로 수행할지를 정의하는 **periodSeconds** 하위 필드를 가집니다. **pr12er** 프로젝트에서는 두 개 모두 단순히 **grpc_health_probe**를 통한 상태 체크를 조건으로 걸고 있지만, 이 두개를 다르게 운용해도 무관합니다. 

## 마치며
**Deployment**, **Service**, **Ingress**로 엮인 큰 그림은 외부 트래픽이 쿠버네티스라는 플랫폼 내부에서 구동중인 애플리케이션을 찾아가는 과정이라고 볼 수 있습니다. 그리고 원하는 대상을 찾는 방법은 메타데이터에 명시된 레이블과 이름만을 활용하였죠. 

모든 명세서를 수작업으로 만들어도 되겠지만, 일반적으로는 아래와 같은 명령어를 통해 템플릿을 만들 수 있습니다. 특히 템플릿에 지정된 이름은 각 메타데이터에 주입되어, 기본적인 식별값을 자동으로 채워넣어 실수를 줄일 수 있습니다.

```shell
# https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#create
> kubectl create deployment server --dry-run=client -o yaml
> kubectl create ingress server ...
> kubectl create service server ...
```

마지막으로 gRPC 애플리케이션 배포를 위해 기억해둘 것은 **Ingress** 객체 정의시 gRPC를 지원하는 **Ingress** 컨트롤러와 백엔드 프로토콜을 설정하는 것입니다. **pr12er** 프로젝트에서는 **kubernetes.io/ingress.class: "nginx"**와 **nginx.ingress.kubernetes.io/backend-protocol: "GRPC"**를 사용하였고, 라즈베리파이 같은 경우는 **kubernetes.io/ingress.class: "traefik"**과 **ingress.kubernetes.io/protocol: h2c** 이 사용될 수 있습니다. 

쿠버네티스 환경에 gRPC를 활용한 애플리케이션을 배포하고 싶다면 이를 꼭 확인하기 바랍니다.