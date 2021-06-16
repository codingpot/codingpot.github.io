---
title: "Okteto에 Deployment를 롤아웃 하기"
date: 2021-06-16
tags:
    - okteto
    - kubectl
    - kubectl-set-image
    - kubectl-rollout-restart
    - kubectl-rollout-status
---

> 코딩냄비 프로젝트 중 **pr12er**는 TensorFlow Korea의 논문을 읽고/리뷰하는 모임 PR12에서 촬영된 동영상을 큐레이션하는 프로젝트입니다. 개략적으로 프론트엔드는 **Flutter**, 백엔드는 **Go**로 작성되었으며, 이 둘간의 인터페이스는 **gRPC/protobuf**로 구성되어있습니다. 특히 **pr12er** 프로젝트의 백엔드 서버는 **PR**이 **Merge** 됨과 동시에 **Okteto**가 제공하는**k8s** 에 배포되는 **CD** 루틴을 탑니다.

이 글은 **Okteto** 에 배포하기위한 파이프라인을 분석하는 총 3편의 시리즈물 중 마지막입니다.

1. [**Okteto** 파이프라인 개요, **okteto build**, **pr12er** 서버용 **Dockerfile** 분석](https://codingpot.github.io/posts/2021-06-11-okteto-pipeline-build/)
2. [**Okteto**에 gRPC용 **Deployment**, ****Service****, **Ingress** 설정](https://codingpot.github.io/posts/2021-06-16-okteto-kubectl-apply/)
3. [**정적 yaml 파일의 설정을 동적으로 바꾸기**](https://codingpot.github.io/posts/2021-06-16-okteto-kubectl-set/)

# **pr12er**에 적용된 **Okteto** 파이프라인 명세서

```yaml
deploy:
#  - okteto build -t okteto.dev/codingpot-pr12er-server:${OKTETO_GIT_COMMIT} -f ./server/deploy/Dockerfile server
#  - for file in k8s/kkweon-okteto/*.yaml; do envsubst < $file | kubectl apply -f -; done
  - kubectl set image deployment server server=okteto.dev/codingpot-pr12er-server:${OKTETO_GIT_COMMIT}
  - kubectl rollout restart deployment grafana-agent
  - kubectl rollout status deployment server && kubectl rollout status deployment grafana-agent
```

## kubectl set image 

이미 존재하는 컨테이너의 이미지를 갱신하는 명령어로 완전한 형식은 다음과 같습니다.

```shell
> kubectl set image (-f FILENAME | TYPE NAME) \
  CONTAINER_NAME_1=CONTAINER_IMAGE_1 ... CONTAINER_NAME_N=CONTAINER_IMAGE_N
```

여기서 **TYPE** 이라는 부분에는 **pod**, **replicationcontroller**, **deployment**, **daemonset**, **replicaset** 이라는 키워드가 올 수 있습니다. 즉 이 블로그 시리즈물에서는 **Deployment** 만을 다뤘지만, 그 외에도 콘테이너 스펙을 명시할 수 있는 쿠버네티스의 객체 종류가 다양합니다. 또한 **TYPE NAME** 대신 **-f** 옵션을 사용하면 파일명 자체를 지정할 수도 있으며, 하나의 **Deployment** 에 포함 가능한 여러개의 컨테이너 명세서를 원하는 만큼 갱신할 수도 있습니다.

그러면 **pr12er** 에서 이 명령어가 어떻게 사용되었는지를 말로 풀어 설명해보겠습니다.

```shell
> kubectl set image deployment server \
  server=okteto.dev/codingpot-pr12er-server:${OKTETO_GIT_COMMIT}
```

**server** 라는 이름의 **Deployment** 객체를 대상으로 지정합니다. 그리고는 해당 **Deployment**에 정의된 컨테이너 중 **server** 라는 이름의 컨테이너를 찾아서 이미지를 **okteto.dev/codingpot-pr12er-server:${OKTETO_GIT_COMMIT}** 으로 설정합니다. 이 그림은 두 번째 게시글에서 본 yaml 파일을 참조하면 매우 명확해 집니다.

```yaml
kind: Deployment
metadata:
  # ommited
  name: server
spec:
  # ommited
  template:
    spec:
      containers:
        - image: okteto.dev/codingpot-pr12er-server
          name: server
  # ommited
```

**종류(kind)** 가 **Deployment** 이며, 부여된 **이름(metadata.name)** 은 **server** 이고, 여기에 포함된 컨테이너 중 **이름(spec.template.spec.containers.name)** 이 **server** 인 녀석의 **이미지(image)** 를 교체 대상으로 특정한 것입니다.

### 왜 필요한가?
그렇다면 **kubectl set image**와 같은 명령어는 왜 필요할까요? 

앞서 정의된 **Deployment** 에는 이미지 태그 정보가 동적으로 유연하게 삽입되지 않습니다. 하지만 소스코드가 문제 없이 빌드되고, 이에 대한 풀 리퀘스트가 정상적으로 수용된다면, 최신 버전으로 빌드된 컨테이너 이미지로 교체되어야 CI/CD가 잘 이루어진다고 볼 수 있겠죠. **kubectl set image**는 바로 이 상황을 위한 것입니다. 

한 가지 알아둘 점은 **kubectl set image** 라는 명령이 하달된다고 해서, 그 즉시 쿠버네티스가 현존하는 모든 컨테이너를 제거하고 새로운 이미지로 갈아끼워 서비스 중단이라는 사태가 발생하는것은 아닙니다. 이 명령은 _즉시 액션을 취하시오_ 라는 말이 아니라, 현재 서빙되는 것은 건들지 않은채 새 컨테이너를 담은 **Pod**을 만든 다음, 해당 **Pod**이 완전히 부팅되었을 때 교체하라는 의미 정도로 생각할 수 있습니다. 

### Okteto 에서는 필요없습니다.
**Okteto** 에서는 **okteto build** 명령을 수행하면, 내부적으로 이미지 업데이트도 함께 수행하기 때문에 명시적으로 **kubectl set image** 명령을 실행할 필요는 없습니다.

하지만 **kubectl set image** 명령을 실행한다고 해서 어떤 불이익이 발생하는것은 아닙니다. 만약 **kubectl set image**를 실행했는데, 이미 **Okteto**가 갈아끼운 이미지와 동일하다면 아무런 액션도 발생하지 않기 때문이죠. 따라서 **Okteto** 만을 사용하지 않는 우리는 기본적으로 **kubectl set image** 명령을 항상 적용하는것이 베스트 프랙티스라고 볼 수 있습니다.

## kubectl rollout restart

이미 존재하는 쿠버네티스의 자원을 재시작하는 완전한 명령어는 다음과 같습니다.

```shell
> kubectl rollout restart TYPE NAME [flags]
```

말 그대로 지정된 자원을 재시작하는 명령어로, 이미 적용된 것이 있다면 새로운 설정이 들어간 녀석으로 교체하는것이 가능합니다. **pr12er** 프로젝트에서는 **grafana-agent** 라는 이름의 **Deployment** 자원을 재시작 하는 용도로 이 명령이 사용되었습니다. 그 이유는 **grafana-agent** 에 포함된 그라파나 에이전트에 대한 설정이 들어간 **ConfigMap** 의 정보가 바뀔 수 있기 때문입니다.

이 명령 또한 **kubectl set image** 처럼, 강제로 새로 시작하라는 명령을 쿠버네티스에 하달하는것은 아닙니다. 일종의 제안을 하는것으로, 만약 이미 적용된 자원과 새롭게 재시작을 요청한 자원이 다른점이 없다면 쿠버네티스는 별다른 액션을 취하지 않습니다. 따라서 이 명령을 항상 넣어두더라도 어떠한 해도 없는 것이죠. 즉 **kubectl set image** 와 마찬가지로, 항상 넣어두는것이 베스트 프랙티스라고 볼 수 있습니다.

## kubectl rollout status

롤아웃된 상태를 체크하는 완전한 명령어는 다음과 같습니다.

```shell
> kubectl rollout status (TYPE NAME) [flags]
```

> By default 'rollout status' will watch the status of the latest rollout **until it's done.** 

이 명령은 현재 롤아웃되는 자원의 상태를 지속적으로 체크합니다. 그리고 롤아웃이 완료될 때까지 그 체크는 계속되며 완전히 롤아웃이 완료될 때까지 기다리게 됩니다. 롤아웃이 완료되었다고 판단되는 시점은 다음 세 조건을 충족했을 때입니다 [링크](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#deployment-status).
- 해당 **Deployment**에 연관된 모든 **Pod(replica에 명시된 수만큼)** 들이 새로운 버전으로 갱신됨
- 새로운 버전으로 채워진 **Pod** 들이 가용상태가 됨
- 이전 버전의 **Pod** 들이 완전히 셧다운됨 (남은 놈들이 없음)

이 명령이 실행되면 콘솔 창에는 아래와 유사한 메시지가 출력되며, 성공적으로 롤아웃이 완료됨의 여부는 **exit status** 를 검사해 보면 알 수 있습니다. **exit status** 는 **$?** 로 접근할 수 있으므로, 만약 성공적으로 롤아웃된 경우 이 값을 출력하면 **0** 이라는 값을 얻게됩니다.

```shell
Waiting for rollout to finish: 1 of 1 updated replicas are available...
deployment "server" successfully rolled out
```

## 참고자료
- **kubectl set image**
  - https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#-em-image-em-

- **kubectl rollout restart**
  - https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#-em-restart-em-

- **kubectl rollout status**
  - https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#-em-status-em-