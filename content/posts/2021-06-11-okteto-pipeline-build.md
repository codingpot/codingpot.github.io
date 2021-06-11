---
title : "Okteto 파이프라인 개요, okteto build, pr12er 서버용 Dockerfile 분석"
date : 2021-06-11
tags:
    - okteto
---

> 코딩냄비 프로젝트 중 `pr12er`는 TensorFlow Korea의 논문을 읽고/리뷰하는 모임 PR12에서 촬영된 동영상을 큐레이션하는 프로젝트입니다. 개략적으로 프론트엔드는 `Flutter`, 백엔드는 `GO`로 작성되었으며, 이 둘간의 인터페이스는 `gRPC/protobuf`로 구성되어있습니다. 특히 `pr12er` 프로젝트의 백엔드 서버는 `PR`이 `Merge` 됨과 동시에 `Okteto`가 제공하는`k8s` 에 배포되는 `CD` 루틴을 탑니다.

이 글은 `Okteto` 에 배포하기위한 파이프라인을 분석하는 총 X 편의 시리즈물 중 첫 번째입니다.

1. [`Okteto` 파이프라인 개요, `okteto build`, `pr12er` 서버용 `Dockerfile` 분석](https://codingpot.github.io/cicd/okteto-pipeline-build/)
2. [Okteto에 gRPC용 Deployment, Service, Ingress 설정]()
3. ................

## `Okteto` 파이프라인의 개요
말 그대로 `Okteto`에 원하는 애플리케이션을 배포하는 일련의 과정(파이프라인)을 정의하는 방법입니다. 일반적으로 `okteto-pipeline.yaml` 이라는 파일로 작성되며, 그 과정은 다음과 같습니다.

1.애플리케이션에 대한 도커 이미지를 빌드한 후 `Okteto` 레지스트리에 해당 이미지를 올리는 과정 
2.`kubectl` 명령어로 `Deployment`, `Service` 등을 `k8s` 에 설정

### `pr12er`에 적용된 `Okteto` 파이프라인 명세서
다음은 `pr12er` 에서 정의한 `okteto-pipeline.yaml`이며, 이번 게시글에서는 그 중 첫 번째인 `okteto build` 명령어를 분석합니다.
```yaml
deploy:
  - okteto build -t okteto.dev/codingpot-pr12er-server:${OKTETO_GIT_COMMIT} -f ./server/deploy/Dockerfile server
  - for file in k8s/kkweon-okteto/*.yaml; do envsubst < $file | kubectl apply -f -; done
  - kubectl set image deployment server server=okteto.dev/codingpot-pr12er-server:${OKTETO_GIT_COMMIT}
  - kubectl rollout restart deployment grafana-agent
  - kubectl rollout status deployment server && kubectl rollout status deployment grafana-agent
```

// TODO 전체적인 설명 (매우 간단히)

### `okteto build` 명령어
[공식문서](https://okteto.com/docs/reference/cli#build)에 따르면 `okteto build` 명령어는 다음과 같은 일을 합니다. 이 내용을 가지고, 약간의 옵션 플래그를 더한 `pre12er`용 `okteto-pipeline.yaml`의 `okteto build` 가 하는일을 살펴보죠.
> `Dockerfile`로부터 이미지를 빌드하여 `Okteto`의 레지스트리에 등록

```yaml
deploy:
  - okteto build -t okteto.dev/codingpot-pr12er-server:${OKTETO_GIT_COMMIT} -f ./server/deploy/Dockerfile server
```

**`-t` 옵션**
- 빌드될 이미지명과 태그를 설정합니다. 여기서 설정된 이름으로 `Okteto` 레지스트리에 등록됩니다.

**`-f` 옵션**
- 빌드시 사용할 `Dockerfile`의 위치(path)를 명시합니다.

**마지막 `server`**
- `Dockerfile`가 빌드할 디렉터리 대상을 지정합니다.

이를 종합적으로 풀어서 설명하면, `server` 라는 디렉터리를 참조하여 `./server/deploy/`에 위치한 `Dockerfile`로 `okteto.dev/codingpot-pr12er-server:${OKTETO_GIT_COMMIT}` 라는 이름의 이미지를 빌드하여 `Okteto` 레지스트리에 등록하라는 뜻이됩니다.

### `pr12er` 서버용 `Dockerfile`
그렇다면 `Dockerfile`은 어떻게 생겼을지도 살펴보겠습니다. 이는 `Okteto`와는 무관하지만, `GO` 언어를 사용한 `gRPC/protobuf` 프로젝트를 위한 기본 골격을 파악하는데 도움이 될 수 있습니다.

```dockerfile
FROM golang:1.16-alpine AS builder

RUN apk --no-cache add ca-certificates && GRPC_HEALTH_PROBE_VERSION=v0.4.2 && \
  wget -qO/bin/grpc_health_probe https://github.com/grpc-ecosystem/grpc-health-probe/releases/download/${GRPC_HEALTH_PROBE_VERSION}/grpc_health_probe-linux-amd64 && \
  chmod +x /bin/grpc_health_probe

WORKDIR /src/
COPY go.mod .
COPY go.sum .
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags "-w -s" -o server cmd/server/main.go

FROM gcr.io/distroless/static:nonroot

COPY --from=builder /src/server .
COPY --from=builder /bin/grpc_health_probe /bin/grpc_health_probe

ENTRYPOINT ["./server"]
```

우선 `FROM`이라는 지시자는 새로운 빌드 스테이지를 초기화하며, 이어지는 지시자가 적용될 대상 베이스 이미지로 설정합니다. 상기 `Dockerfile`에는 `FROM` 지시자가 두 개 등장하는 데, 이는 빌드 스테이지가 여러개임을 뜻합니다. **첫 번째** `FROM`의 이미지에서 작업된 내역을 **두 번째** `FROM`으로 전달하여 최종 이미지는 **두 번째**의것이 되는것이죠. 이 방식은 주로 애플리케이션을 컴파일/빌드하기 위한 여러가지 디펜던시를 담아 애플리케이션의 컴파일/빌드를 수행한 다음, 해당 애플리케이션의 실행 가능한 바이너리 파일만을 **경량** 이미지에 옮겨 이미지의 사이즈를 최소화하는 데 많이 활용됩니다.

- `golang:1.16-alpine` 이미지는 [`alpine`](https://hub.docker.com/_/alpine) 이라는 리눅스 배포판에 기초하여 `GO` 언어까지가 함께 설치된 것입니다. 
- [`gcr.io/distroless/static:nonroot`](https://github.com/GoogleContainerTools/distroless) 는 오로지 구동될 애플리케이션과 런타임 디펜던시만을 포함하고, 기타 일반적인 리눅스에서 제공하는 다양한 패키지(쉘 포함)를 제외한 초경량 이미지입니다. 이에 대한 더 자세한 설명은 게시글 하단의 레퍼런스를 참고하세요.

그렇다면 두 `FROM` 사이에서 첫 번째 `FROM`이 해야할 일은 명확합니다. 애플리케이션을 빌드한 바이너리 파일, 그리고 그 애플리케이션을 구동하는 데 필요한 런타임 디펜던시를 구축하는 것이죠. 그렇게 구축된것을 두 번째 `FROM`에 심어주기 위함입니다.

**`WORKDIR /src/`**
- 첫 번째 `FROM` 이미지에서 작업할 디렉터리를 지정합니다. 

**`COPY go.mod .`, `COPY go.sum .`**
- 프로젝트에 필요한 모듈 디펜던시와 각 디펜던시에 대한 체크섬이 명시된 두 파일을 현재 작업 디렉터리로 복사합니다.
- `COPY` 지시자의 첫 번째 인자는 앞서 지정된 `server` 라는 디렉터리를 기준으로 복사될 파일명을 명시하며, 두 번째 인자는 파일이 복사되어 저장될 이미지내 디렉터리를 명시합니다(즉 `WORKDIR`).

**`RUN go mod download`**
- 디펜던시에대한 명세서만 있을 뿐, 현재 이미지에는 해당 디펜던시가 없기 때문에, 명세서를 참조하여 모듈을 다운로드 받습니다.

**`COPY . .`**
- 디펜던시 외 프로젝트를 구동하는데 필요한 모든 파일을 복사합니다.

**`RUN ... go build ... -o server cmd/server/main.go`**
- 이후 `go build` 명령어로 `cmd/server/main.go` 라는 파일을 바이너리 파일로 빌드하고, 그 바이너리 파일의 이름을 `server`라고 명명합니다.

이제 구동될 애플리케이션을 빌드하였으니, 실제 애플리케이션 구동에 필요없는 파일들(소스 파일 등)을 제외하고 두 번째 `FROM` 이미지로 복사합니다. 이 과정은 실제 이미지가 올라갔을 때 최초로 실행되는 지점인 `ENTRYPOINT` 지시자 이전까지에 해당합니다.

**`COPY --from=builder /src/server .`**
- `--from=builder` 라는 플래그가 새롭게 추가된 `COPY` 지시자 입니다. 앞서 첫 번째 `FROM` 지시자에서 `as builder` 라고 적은것은 나중에 해당 빌드 스테이지를 참조하기위한 별칭을 지정한 것입니다. 
- 따라서 `builder` 스테이지에서 `go build`로 만들어진 바이너리 파일인 `server`를 현재 이미지에 복사한다는 뜻이됩니다. 이때 `/src/server`인 이유는 첫 번째 스테이지에서 작업 디렉터리가 `/src/`로 지정되어, 만들어진 `server` 바이너리 파일이 `/src/` 디렉터리내 위치하기 때문입니다.

**`COPY --from=builder /bin/grpc_health_probe /bin/grpc_health_probe`**
- `grpc_health_probe`는 `gRPC` 서버의 실행여부를 판단하는 데 쓰이는 유틸리티 프로그램입니다. 이 프로그램은 첫 번째 빌드 스테이지 중 `RUN` 지시자에서 `wget`을 통해 다운로드 받은것을 그대로 복사한 것입니다. 
- 두 번째 스테이지에서 다운로드 받지 **못한** 이유는 초경량 이미지여서 `wget` 명령어조차 들어있지 않기 때문입니다.

## 참고자료
- **Okteto**
  - https://okteto.com/
- **Golang Images**
  - https://hub.docker.com/_/golang
- **Distroless Image**
  - https://github.com/GoogleContainerTools/distroless
- **Best practices for building containers**
  - https://cloud.google.com/architecture/best-practices-for-building-containers
- **Distroless Containers: Hype or True Value?**
  - https://hackernoon.com/distroless-containers-hype-or-true-value-2rfl3wat