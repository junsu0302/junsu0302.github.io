---
title: "[Docker] 04-Docker 이미지 최적화와 네트워크"
date: 2025-12-16
categories: [DevOps, Container]
tags: [DevOps, Container, Docker]
---

# Docker 이미지 최적화와 네트워크

Dockerfile의 **멀티 스테이지 빌드(Multi-stage Build)** 기능을 활용하여 빌드 도구와 런타임 환경을 분리하고, 이미지 용량을 획기적으로 줄이는 방법을 다룬다. 또한, 도커의 가상 네트워크를 생성하여 컨테이너 간에 안전하고 효율적으로 통신(DNS)하는 방법을 설명한다.

## Dockerfile 멀티 스테이지 빌드

**멀티 스테이지 빌드**는 하나의 Dockerfile 안에 여러 개의 `FROM` 절을 사용하여, **빌드(Build) 환경**과 **실행(Runtime) 환경**을 분리하는 기술이다. 이를 통해 빌드 도구(Maven, Gradle, Compiler 등)는 최종 이미지에서 제외하고, 실행에 필요한 파일만 남겨 이미지 용량을 최소화할 수 있다.

```dockerfile
# [Stage 1] Build Stage: 소스 코드를 빌드하여 실행 파일(Jar) 생성
FROM maven:3.6-jdk-11 AS builder
WORKDIR /app
COPY . .
RUN mvn package

# [Stage 2] Runtime Stage: 빌드된 결과물만 가져와서 실행
FROM openjdk:11-jre-slim
WORKDIR /app
# builder 스테이지에서 생성된 파일만 복사해 온다.
COPY --from=builder /app/target/my-app.jar .

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "my-app.jar"]
```
- 빌드 단계는 `FROM` 인스트럭션으로 시작하고, `AS` 파라미터를 통해 build-stage 이름을 붙임
- `RUN` 인스트럭션으로 빌드 중에 컨테이너 안에서 명령을 실행하고, 결과를 이미지 레이어에 저장
- `--from` 인자를 통해 앞선 빌드 단계의 파일 시스템에 있는 파일임을 명시
- `EXPOSE` 인스트럭션으로 해당 포트로 외부로 공개
- `ENTRYPOINT` 인스트럭션으로 해당 이미지로 컨테이너가 실행되면 도커가 이 인스트럭션에서 정의된 명령을 실행

Dockerfile에 멀티 스테이지를 적용하면 다음과 같은 이점을 얻을 수 있다.
- 표준화: 개발자가 로컬 컴퓨터에 빌드 도구를 설치할 필요 없이, Dockerfile 내부에 정의된 정확한 버전의 도구로 빌드가 수행
- 성능 향상: 불필요한 빌드 도구가 제거되어 이미지 크기가 줄어들며, 레이어 캐시를 효율적으로 사용하여 배포 속도 향상
- 이식성 향상: 호스트 OS의 환경(라이브러리 유무 등)에 의존하지 않고, 도커만 있다면 어디서든 동일한 빌드 결과물 획득
- 외부 공격 차단: 최종 이미지에 컴파일러나 빌드 도구, 소스 코드가 포함되지 않으므로, 공격 표면(Attack Surface)이 줄어들어 보안성이 강화

## 네트워크

여러 개의 컨테이너(예: Web Server + Database)를 함께 운영하려면 컨테이너 간의 통신망이 필요하다. 도커는 가상 네트워크를 생성하여 컨테이너들을 논리적으로 묶어주고 통신을 가능하게 한다.

도커 네트워크의 특징
- 격리: 네트워크에 연결된 컨테이너끼리만 통신이 가능하여 보안성이 높다.
- DNS (Service Discovery): 같은 네트워크 내에서는 IP 주소를 몰라도 **컨테이너 이름(Container Name)**만으로 서로 통신할 수 있다.

컨테이너 간 통신에 사용되는 도커 네트워크는 다음과 같이 생성할 수 있다.

```bash
docker network create [NETWORT_NAME]
```

생성된 도커 네트워크는 새로 컨테이너를 만들 때 `--network` 옵션을 통해 연결할 수 있다.
```bash
docker container run -dp 800:80 --network [NETWORK_NAME] [IMAGE_NAME]
```
