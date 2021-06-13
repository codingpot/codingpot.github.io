---
title: "Okteto에 gRPC용 Deployment, Service, Ingress 설정"
date: 2021-06-11
tags:
    - okteto
    - kubectl
---

> 코딩냄비 프로젝트 중 `pr12er`는 TensorFlow Korea의 논문을 읽고/리뷰하는 모임 PR12에서 촬영된 동영상을 큐레이션하는 프로젝트입니다. 개략적으로 프론트엔드는 `Flutter`, 백엔드는 `GO`로 작성되었으며, 이 둘간의 인터페이스는 `gRPC/protobuf`로 구성되어있습니다. 특히 `pr12er` 프로젝트의 백엔드 서버는 `PR`이 `Merge` 됨과 동시에 `Okteto`가 제공하는`k8s` 에 배포되는 `CD` 루틴을 탑니다.

이 글은 `Okteto` 에 배포하기위한 파이프라인을 분석하는 총 X 편의 시리즈물 중 두 번째입니다.

1. [`Okteto` 파이프라인 개요, `okteto build`, `pr12er` 서버용 `Dockerfile` 분석](https://codingpot.github.io/cicd/okteto-pipeline-build/)
2. [Okteto에 gRPC용 Deployment, Service, Ingress 설정]()
3. ................

# `pr12er`에 적용된 `Okteto` 파이프라인 명세서

```yaml
deploy:
#  - okteto build -t okteto.dev/codingpot-pr12er-server:${OKTETO_GIT_COMMIT} -f ./server/deploy/Dockerfile server
  - for file in k8s/kkweon-okteto/*.yaml; do envsubst < $file | kubectl apply -f -; done
#  - kubectl set image deployment server server=okteto.dev/codingpot-pr12er-server:${OKTETO_GIT_COMMIT}
#  - kubectl rollout restart deployment grafana-agent
#  - kubectl rollout status deployment server && kubectl rollout status deployment grafana-agent
```

## 어떤 `yaml` 파일들이 존재하나?

```shell
k8s/kkweon-okteto/
├── ingress.yaml
└── server.yaml
```

![](images/ingress-service-deployment.png)

### Ingress Annotations
[dev.okteto.com/generate-host: "true"](dev.okteto.com/generate-host)
- `Okteto` 에서 자동으로 호스트 이름을 할당하는것을 허용하고자 할 때.

[kubernetes.io/ingress.class: "nginx"](https://www.nginx.com/resources/glossary/kubernetes-ingress-controller/)
- 쿠버네티스의 Ingress 컨트롤러를 nginx로 지정
- Ingress 컨트롤러의 역할에는 외부 트래픽을 수용하여 내부 Pod 들로 로드밸런싱 해주기, 클러스터 밖의 다른 서비스와 소통이 필요한 트래픽 관리, Pods들을 모니터링하여 추가/제거에 따른 로드밸런싱 규칙 갱신 등이 있다. 

[nginx.ingress.kubernetes.io/backend-protocol: "GRPC"](https://kubernetes.github.io/ingress-nginx/examples/grpc/#grpc)
> This is the magic ingredient that sets up the appropriate nginx configuration to route http/2 traffic to our service.
- 즉 nginx의 설정을 건드려서, `http/2` 트래픽이 쿠버네티스 내부 서비스로 들어올 수 있도록 해준다. 

[nginx.ingress.kubernetes.io/ssl-redirect: "true"](https://kubernetes.github.io/ingress-nginx/examples/rewrite/)
> Indicates if the location section is accessible SSL only (defaults to True when Ingress contains a Certificate)
- SSL 로만 접근 가능함을 명시한다 (디폴트는 `true` 인데, 여기서는 명시적으로 `true` 라고 적어줌)

### Ingress Spec

### Service Metadata

### Service Spec

### Deployment Metadata

### Deployment Spec

```shell
> kubectl create deployment server --dry-run=client -o yaml
# https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#create
```

# 참고자료
- How Deployment, Service, Ingress are related in their manifest
  - https://dwdraju.medium.com/how-deployment-service-ingress-are-related-in-their-manifest-a2e553cf0ffb
- dev.okteto.com/generate-host