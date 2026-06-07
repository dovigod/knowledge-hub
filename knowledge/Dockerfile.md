---
id: 019ea14b-73d1-72cd-8a8e-c70d36752115
name: Dockerfile
aliases:
  - dockerfile
  - 도커파일
updated_at: '2026-06-07T08:55:37.169Z'
summary: >-
  Declarative text recipe describing how to build a Docker image, instruction by
  instruction, from a base image to the final command.
sources:
  - 019ea0f5-11a0-779d-b078-08da3e430aa4
---
<!-- ⚠️  Generated from .kh.db by `kh sync`. Hand edits are detected and staged under _proposals/manual_edit/ on the next sync. -->
# Dockerfile

## Overview

A Dockerfile is the **recipe** for a [[Docker]] image. Each line is an instruction (`FROM`, `RUN`, `COPY`, `EXPOSE`, `CMD`, etc.) that the `docker build` engine executes in order to produce an immutable image, which can later be instantiated as one or more containers.

> [!note] Three-part identity
> - **Dockerfile** = recipe.
> - **Image** = the baked, immutable artifact (the mold).
> - **Container** = a running instance of the image (one pastry from the mold).

## Notes

### Common instructions
- `FROM` — base image to start from.
- `RUN` — execute a shell command at build time; result is baked into the image.
- `COPY` / `ADD` — copy files from build context into the image.
- `WORKDIR` — set working directory for following instructions.
- `ENV` — set environment variables.
- `EXPOSE` — **documentation only** (see warning below).
- `CMD` / `ENTRYPOINT` — default command when the container runs.

### EXPOSE is NOT port mapping
A frequent misconception (and explicit topic in the source conversation): `EXPOSE 3000` in a Dockerfile does NOT publish a port. It's metadata — a hint for readers and tooling. The actual host-to-container port mapping happens at run time via `docker run -p` or `docker-compose.yml`. See [[Docker port mapping]] and [[iptables NAT]] for what really makes a container reachable.

### Build vs runtime split
- **Build-time:** every Dockerfile instruction. Produces image layers cached by content hash.
- **Runtime:** `docker run` flags (`-p`, `--memory`, `--network`, env, volumes). These are *not* in the Dockerfile and can vary per launch.

> [!tip] Mental rule
> If something can change between environments (port, memory limit, secrets), it belongs at runtime — not in the Dockerfile.

---

## 한국어

### 개요

Dockerfile은 [[Docker]] 이미지를 만드는 **레시피**다. 각 줄이 하나의 지시(`FROM`, `RUN`, `COPY`, `EXPOSE`, `CMD` 등)로, `docker build` 엔진이 순서대로 실행해서 불변 이미지를 만든다. 그 이미지로 하나 이상의 컨테이너를 인스턴스화할 수 있다.

> [!note] 3요소 정체성
> - **Dockerfile** = 레시피.
> - **Image** = 구워진 불변 산물 (붕어빵 틀).
> - **Container** = 이미지를 실행한 인스턴스 (틀로 찍어낸 붕어빵 하나).

### 노트

#### 흔한 지시문
- `FROM` — 시작 베이스 이미지.
- `RUN` — 빌드 시점 셸 명령 실행. 결과가 이미지에 구워짐.
- `COPY` / `ADD` — 빌드 컨텍스트에서 이미지로 파일 복사.
- `WORKDIR` — 이후 지시문의 작업 디렉터리.
- `ENV` — 환경 변수.
- `EXPOSE` — **문서일 뿐** (아래 경고 참고).
- `CMD` / `ENTRYPOINT` — 컨테이너 실행 시 기본 명령.

#### EXPOSE는 포트 매핑이 아니다
자주 하는 오해(대화의 명시적 주제): Dockerfile의 `EXPOSE 3000`은 포트를 publish하지 **않는다**. 메타데이터일 뿐 — 독자와 도구를 위한 힌트. 실제 호스트-컨테이너 포트 매핑은 실행 시점 `docker run -p` 또는 `docker-compose.yml`에서 일어난다. 진짜로 컨테이너를 도달 가능하게 만드는 건 [[Docker port mapping]]과 [[iptables NAT]] 참고.

#### 빌드 vs 런타임 분리
- **빌드 시점:** Dockerfile의 모든 지시문. 콘텐츠 해시로 캐시되는 이미지 레이어 생성.
- **런타임:** `docker run` 플래그(`-p`, `--memory`, `--network`, env, volumes). Dockerfile에 있지 *않고* 실행할 때마다 달라질 수 있음.

> [!tip] 멘탈 규칙
> 환경마다 달라질 수 있는 것(포트, 메모리 한도, 시크릿)은 런타임에 속한다 — Dockerfile이 아님.

## Sources

- [[raw/conversations/019ea0f5-11a0-779d-b078-08da3e430aa4|019ea0f5-11a0-779d-b078-08da3e430aa4]]
