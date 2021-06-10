+++
title = "Okteto 파이프라인(빌드)"
date = "2021-01-01"
+++

현재 코딩냄비 프로젝트 중 `pr12er` 에서 배포하는 방식을 그대로 분석한 포스트다. `pr12er` 프로젝트에서는 쿠버네티스 클러스터의 네임스페이스를 임대해주는 **Okteto** 라는 서비스를 활용해 구현된 서비스를 배포한다. 살펴봐야 할 내용이 많으므로, 하나씩 분석한다.

Okteto로 디플로이 하기위한 파이프라인 중 가장 먼저 살펴볼 것은 `okteto build` 명령어이다. `pr12er`에서는 루트 디렉터리의[okteto-pipeline.yaml](https://github.com/codingpot/pr12er/blob/main/okteto-pipeline.yaml) 파일 가장 첫 번째 라인에서 사용되며, 아래는 그 한 줄을 보여준다.

```yaml
deploy
  - okteto build -t okteto.dev/이미지명:${OKTETO_GIT_COMMIT} -f ./server/deploy/Dockerfile server
```

**okteto build** CLI의 [공식문서](https://okteto.com/docs/reference/cli#build)에 따르면, 이 명령어는 **Dockerfile**로부터 이미지를 빌드하고, 지정된 **registry**에 빌드된 이미지를 **push** 하는 역할을 수행한다.

`-t okteto.dev/이미지명:${OKTETO_GIT_COMMIT}`
- 빌드할 이미지명과 태그명을 지정한다.

`-f ./server/deploy/Dockerfile`
- 사용할 `Dockerfile`의 Path를 명시한다.

마지막 **server**
- 이미지를 빌드할 대상 디렉터리를 지정한다.
- `Dockerfile`과 GO 프로젝트가 서로다른 곳에 위치하기 때문에, 명시적으로 지정한다.