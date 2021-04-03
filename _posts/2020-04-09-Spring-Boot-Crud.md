---
title: "Spring Boot CRUD using Redis for session"

categories:
  - Post
tags:
  - Post
---

웹 개발 중 클라이언트 개발을 주로 해왔지만 웹 개발자로서 서버 개발 능력 또한 갖추어야 한다고 생각해 자바 `Spring Boot`를 활용한 `CRUD API`를 구현했다.

`Express.js`가 아닌 `Spring Boot`를 선택한 이유로는 대학 생활에 가장 즐겁게 배웠던 프로그래밍 언어였으며 우리나라의 대부분 서비스 서버는 자바로 구현되어 있기 때문이다.

또한 개인적으로 웹 `Front-end` 개발의 길을 걷게 한 안드로이드 앱 개발 강의를 들을 때도 자바를 사용하여 개발했기 때문에 애정이 있다.

**백문이 불여 일견**이라 하여 기본적인 개념만 습득하고 바로 개발을 시작하여 중간 중간 개념이 부족한 부분은 찾아가며 구현하는 방식으로 진행했다.

## 목표

1. 스프링 부트를 활용, `CRUD API`를 구현해보고 스프링 부트 활용 능력 향상
2. `Session`을 서버 인스턴스가 아닌 `Redis` 데이터베이스에 저장 및 관리

## Spring Boot 프로젝트 구성

### 환경

- H2 Database
- Spring-Data-JPA
- Spring-OAuth2-Client
- Spring-Data-Redis
- Spring-Session-Data-Redis

### 디렉토리 구조

```
|- /config
|- /domain
|- /service
|- /web
|- Application.java
```

- **`config`** - 스프링 부트 웹 서버 환경 설정 클래스 모음
- **`domain`** - 데이터베이스와 연결할 `Entity` 클래스 모음
- **`service`** - `Transaction`과 `Domain`의 순서를 보장하는 클래스 모음
- **`web`** - `Controller`와 `dto` 클래스 모음

> 🚨 service 계층에서 비즈니스 로직을 처리해야 한다고 많은 사람들이 생각한다. 이는 `트랜잭션 스크립트 방식`이라고 하는데 하나의 서비스에서 많은 모델을 읽어 로직을 각각 작성함으로써 서비스의 **복잡도는 매우 높아지며 중복되는 코드는 개발하면 할 수록 많아진다.** 때문에 비즈니스 로직은 도메인 클래스에 작성하여 응집도를 높이고 서비스 클래스는 트랜잭션과 도메인의 순서만 보장하여 복잡도를 낮춘다.

### Web

`web` 디렉토리에는 `Controller`와 `Dto` 클래스를 작성했다. 여기서 `Dto`는 데이터 송수신을 위한 클래스로 요청 및 응답에 해당하는 데이터 객체 모델이다.

> `Domain` 클래스와 `Dto` 클래스는 유사하다는 점이 있어 구분해야 할까? 의문점이 들 수 있다. 하지만 `Domain` 클래스는 데이터베이스와 가장 밀접한 핵심 클래스로 변경 시 여러 클래스에 영향을 끼칠 수 있다. 때문에 자주 변경될 수 있는 `Dto` 클래스를 통해 필요한 정보만 가질 수 있는 객체를 만들어 변경이 자주 일어날 수 있는 View에 활용한다.

### Controller

`@RestController` 어노테이션을 활용하여 요청 및 응답 Dto 객체를 통해 데이터의 송수신을 담당한다.

## Redis 활용
