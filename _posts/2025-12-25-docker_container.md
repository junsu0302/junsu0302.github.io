---
title: "[Docker] 02-Docker Container 사용하기"
categories: [DevOps, Container]
tags: [DevOps, Container, Docker]
---

# Docker Container 사용하기

## Docker Engine

도커를 설치한 호스트 컴퓨터의 내부 실행 구조는 다음과 같다. 핵심은 도커 엔진이 컨테이너와 호스트 OS 사이의 매개체 역할을 한다는 점이다.

- **컴퓨터 (Host)**
  - **운영체제 (Host OS)**
  - **도커 엔진 (Docker Engine)**: 컨테이너를 관리하고 실행하는 주체
    - **도커 컨테이너**: 실제 애플리케이션이 동작하는 격리된 환경
    - **도커 이미지 캐시**: 다운로드한 이미지를 저장하는 로컬 저장소

우리가 도커를 사용하는 과정은 도커 엔진의 API를 호출하는 과정이다.
1. 사용자가 **도커 CLI** 명령어를 입력 (API 호출)
2. **도커 엔진**이 명령을 수신하여 작업 수행
   - **빌드 (Build)**: 애플리케이션 실행 파일과 설정값을 포함하여 도커 이미지 생성
   - **공유 (Share)**: 생성된 이미지를 레지스트리(Docker Hub 등)에 업로드 또는 다운로드
   - **실행 (Run)**: 이미지를 기반으로 컨테이너 인스턴스 생성 및 실행

이러한 **`빌드-공유-실행`** 워크플로우는 도커가 어디서든 동일하게 동작하는 이식성을 보장하는 핵심 개념이다.

## Docker Container

도커 컨테이너는 기술적으로 다음과 같이 구성된다.
- **애플리케이션**: 실제 실행할 코드 및 바이너리
- **가상 리소스**: 격리된 파일 시스템, 호스트명, IP 주소 등

도커 컨테이너는 VM과 다르게 내부에 **게스트 OS(Guest OS)를 포함하지 않는다.** 대신 도커 엔진을 통해 **호스트 OS의 리눅스 커널을 공유**하여 CPU와 메모리를 할당받는다. 이를 통해 다음과 같은 이점을 얻는다.
- **리소스 경감**: 무거운 게스트 OS가 없으므로 가볍고 부팅이 빠르다.
- **집적도 향상**: 오버헤드가 적어 하나의 서버에서 더 많은 애플리케이션을 실행할 수 있다.
- **격리성**: 리눅스 네임스페이스(Namespace)와 cgroups 기술을 통해 독립된 환경을 보장한다.


## Docker Container 실행

Docker Engine의 구조와 컨테이너의 작동 원리를 다룬다. 컨테이너는 호스트 OS의 커널을 공유하여 리소스 효율성을 극대화하며, `Build-Share-Run`의 생명주기를 가진다. 또한, 컨테이너 실행(`run`), 조회(`ls`), 삭제(`rm`) 등 핵심 명령어의 사용법과 포트 포워딩을 통한 외부 네트워크 통신 과정을 설명한다.

도커 실행은 다음과 같은 명령어를 사용하면 된다.
```md
`docker container run [OPTIONS] [IMAGE NAME]` : 컨테이너로 애플리케이션 실행

- `-i` or `--interactive` : 컨테이너와 상호작용 활성화
- `-t` or `--tty` : 터미널을 통해 컨테이너 조작(주로 -i와 함께 사용)
- `-p` or `--publish [HOST PORT]:[CONTAINER PORT]` : 호스트 PORT로 들어온 트래픽을 컨테이너 PORT로 전달
- `-d` or `--detach` : 컨테이너를 백그라운드에서 실행
```


## Docker Container 정보 확인

현재 실행 중인 도커 컨테이너의 정보는 다음과 같이 확인할 수 있다.
```md
`docker container ls` : 실행 중인 컨테이너 정보 출력
`docker container ls --all` : 모든 컨테이너 정보 출력
`docker container logs [CONTAINER ID]` : 대상 컨테이너에서 수집된 모든 로그 출력
`docker container inspect [CONTAINER ID]` : 대상 컨테이너의 상세 정보 출력(IP, Volume, Network 등)
`docker container stats` : 실행 중인 컨테이너 상태 확인(CPU, Memory)
```

## Docker Container 삭제

도커 컨테이너 사용이 끝나면 다음과 같은 명령어로 컨테이너를 삭제할 수 있다.
```md
`docker container rm [OPTION] [CONTAINER ID]` : 컨테이너 삭제
`--force` or `-f` : 실행 중인 컨테이너라도 명령 실행
CONTAINER ID에 $(docker container ls --all -q)입력 시 모든 컨테이너 삭제
```

## Docker Container 호스팅 과정

`docker container run -d -p 8088:80 [IMAGE NAME]` 명령어로 웹 서버를 실행했을 때, 외부에서 접속하는 원리는 다음과 같다.

환경 가정 : 
- 호스트 컴퓨터 IP(물리 네트워크 주소) : 192.168.2.150 (외부 접근 가능)
- 도커 컨테이너 IP(가상 네트워크 주소) : 172.0.5.1 (외부 접근 불가, 도커 내부망)

트래픽 흐름 :
1. 사용자가 브라우저를 통해 192.168.2.150:8088로 접속 시도
2. 도커 엔진이 8088 포트로 들어온 요청을 감지
3. 도커 엔진이 설정된 규칙(`-p 8088:80`)에 따라 트래픽을 컨테이너의 80포트로 전달
4. 포트 80을 보유한 도커 컨테이너(172.0.5.1) 내부의 웹 서버가 요청을 받아 응답

이 과정을 **포트 포워딩(Port Forwarding)**이라고 하며, 이를 통해 격리된 컨테이너 환경을 유지하면서도 필요한 서비스만 외부에 노출할 수 있다.
