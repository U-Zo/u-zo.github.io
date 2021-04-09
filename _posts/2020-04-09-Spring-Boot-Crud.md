---
title: "Spring Boot CRUD using Redis for session"

categories:
  - Post
tags:
  - Server
---

# Spring Boot CRUD using Redis for session

웹 개발 중 클라이언트 개발을 주로 해왔지만 웹 개발자로서 서버 개발 능력 또한 갖추어야 한다고 생각해 자바스터디에 참여했고 첫 번째 공부로 `Spring Boot`를 활용한 `CRUD API`를 구현했다.

`Express.js`가 아닌 `Spring Boot`를 선택한 이유로는 대학 생활에 가장 즐겁게 배웠던 프로그래밍 언어였으며 우리나라의 대부분 서비스 서버는 자바로 구현되어 있기 때문이다.

또한 개인적으로 웹 `Front-end` 개발의 길을 걷게 한 안드로이드 앱 개발 강의를 들을 때도 자바를 사용하여 개발했기 때문에 애정이 있다.

본문의 내용은 **스프링 부트와 AWS로 혼자 구현하는 웹서비스** 책을 기반으로 공부하고 실습해보았다.

## 목표

1. 스프링 부트를 활용, `CRUD API`를 구현해보고 스프링 부트 활용 능력 향상
2. `Session`을 서버 인스턴스가 아닌 `Redis` 데이터베이스에 저장 및 관리

## 환경 구성

- H2 Database
- Spring-Data-JPA
- Spring-OAuth2-Client
- Spring-Data-Redis
- Spring-Session-Data-Redis

## Spring Boot 게시글 CRUD APIs 구현

### 전체 디렉토리 구조

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

### Controller와 Dto

```
|- /dto
|- IndexController.java
|- PostsApiController.java
```

`web` 디렉토리에는 `Controller`와 `Dto` 클래스를 작성했다. 여기서 `Dto`는 데이터 송수신을 위한 클래스로 요청 및 응답에 해당하는 데이터 객체 모델이다.

> `Domain` 클래스와 `Dto` 클래스는 유사하다는 점이 있어 구분해야 할까? 의문점이 들 수 있다. 하지만 `Domain` 클래스는 데이터베이스와 가장 밀접한 핵심 클래스로 변경 시 여러 클래스에 영향을 끼칠 수 있다. 때문에 자주 변경될 수 있는 `Dto` 클래스를 통해 필요한 정보만 가질 수 있는 객체를 만들어 변경이 자주 일어날 수 있는 View에 활용한다.

**PostApiController**

```java
package springboot.web;

import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.*;
import springboot.service.PostsService;
import springboot.web.dto.PostsResponseDto;
import springboot.web.dto.PostsSaveRequestDto;
import springboot.web.dto.PostsUpdateRequestDto;

@RequiredArgsConstructor
@RestController
public class PostsApiController {

    private final PostsService postsService;

    @PostMapping("/api/v1/posts")
    public Long save(@RequestBody PostsSaveRequestDto requestDto) {
        return postsService.save(requestDto);
    }

    @PutMapping("/api/v1/posts/{id}")
    public Long update(@PathVariable Long id,
                       @RequestBody PostsUpdateRequestDto requestDto) {
        return postsService.update(id, requestDto);
    }

    @GetMapping("/api/v1/posts/{id}")
    public PostsResponseDto findById(@PathVariable Long id) {
        return postsService.findById(id);
    }

    @DeleteMapping("/api/v1/posts/{id}")
    public Long delete(@PathVariable Long id) {
        postsService.delete(id);

        return id;
    }
}
```

**`@RestController`**: 요청 및 응답 `Dto` 객체를 통해 `JSON` 데이터 송수신 `RestAPI Controller`로 설정

**`@___Mapping`**: url 주소를 매개 변수로 주어 요청에 따른 로직을 수행하는 메소드 작성

**Dto**

데이터 송수신을 위한 데이터 객체 모델이다. `Entity`의 일부 속성을 갖거나 요청 혹은 응답을 데이터를 해당 클래스의 객체로 전달한다.

새로운 `Entity`를 만들어 **데이터베이스에 저장해야할 경우** `toEntity()` 메소드를 정의하여 `Entity` 객체로 **변환**한 후 데이터베이스에 **저장**한다.

아래 간략하고 대표적인 `Dto`로 `PostsSaveRequestDto.java` 코드이다.

```java
// PostsSaveRequestDto.java

package springboot.web.dto;

import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;
import springboot.domain.posts.Posts;

@Getter
@NoArgsConstructor
// Entity 클래스는 Dto 클래스와 유사함에도 Request/Response 클래스로 사용해서는 안 된다.
// 데이터베이스와 맞닿은 핵심 클래스이기 때문에.
public class PostsSaveRequestDto {

    private String title;
    private String content;
    private String author;

    @Builder
    public PostsSaveRequestDto(String title, String content, String author) {
        this.title = title;
        this.content = content;
        this.author = author;
    }

    // Entity 클래스로 변환
    public Posts toEntity() {
        return Posts.builder()
                .title(title)
                .content(content)
                .author(author)
                .build();
    }
}
```

### Domain

```
|- /posts
|- /user
|- BaseTimeEntity.java
```

`domain` 디렉토리에는 데이터베이스의 테이블 데이터에 해당하는 `Entity` 클래스와 `Respository` 클래스를 작성했다.

**BaseTimeEntity.java**

게시글이 생성되거나 수정될 때 해당 일시를 추가하여 관리할 필요가 있다면 `Spring JPA`의 `Auditing` 기능을 활용하여 간단하게 생성, 수정 일시를 추가할 수 있다.

> `Spring JPA`의 `Auditing`은 `Entity`에 발생하는 **변경사항을 감시**하여 **자동으로 값을 넣어주는 기능**이다.

`BaseTimeEntity.java`는 이러한 일시 데이터를 `Entity` 클래스에 상속시켜 자동으로 추가하게 해주는 추상 클래스이다.

```java
// /domain/BaseTimeEntity.java

package springboot.domain;

import lombok.Getter;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import javax.persistence.EntityListeners;
import javax.persistence.MappedSuperclass;
import java.time.LocalDateTime;

@Getter
@MappedSuperclass // Entity 클래스가 상속할 경우 아래 필드들도 컬럼으로 인식
@EntityListeners(AuditingEntityListener.class) // Auditing 기능 포함
public abstract class BaseTimeEntity {
    // 모든 Entity 상위 클래스로 사용
    // 생성일, 수정일 관리

    @CreatedDate
    private LocalDateTime createdDate;

    @LastModifiedDate
    private LocalDateTime modifiedDate;
}
```

- **`@MappedSuperClass`**: `Entity` 클래스가 해당 클래스를 상속할 경우 아래 필드들을 컬럼으로 인식하게 하는 어노테이션
- **`@EntityListeners(AuditingEntityListener.class)`**: `Entity`가 데이터베이스로 영속되기 전에 `Auditing` 기능을 포함하도록 함
- **`@CreatedDate`**: 해당 `Entity`가 생성된 일시로 데이터 지정 어노테이션
- **`@LastModifiedDate`**: 해당 `Entity`가 수정된 일시로 데이터 지정 어노테이션

🚨 `Auditing` 기능을 활성화하려면 Main 클래스에 `@EnableJpaAuditing` 어노테이션을 추가해야 한다.

```java
// Application.java

package springboot;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.data.jpa.repository.config.EnableJpaAuditing;

@EnableJpaAuditing // JPA Auditing 활성화
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args); // 내장 WAS 실행
    }
}
```

**Entity**

`Entity` 클래스는 데이터베이스에 가장 밀접한 클래스로 데이터베이스의 테이블에 해당한다. `Entity` 클래스에 **비즈니스 로직을 작성**하여 `Service` 계층의 **복잡도를 줄일 수 있다.**

아래는 `Posts` 테이블에 해당하는 `Entity` 클래스이다.

```java
// domain/posts/Posts.java

package springboot.domain.posts;

import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;
import springboot.domain.BaseTimeEntity;

import javax.persistence.*;

@Getter
@NoArgsConstructor
@Entity // 테이블과 링크될 클래스 명시, 언더스코어 네이밍으로 테이블 이름 매칭
public class Posts extends BaseTimeEntity {

    @Id // 해당 테이블의 PK 필드
    // 복합키를 사용할 경우 유니크 키 따로 생성 권장
    @GeneratedValue(strategy = GenerationType.IDENTITY) // PK의 생성 규칙
    // GenerationType.IDENTITY: auto_increment
    private Long id;

    @Column(length = 500, nullable = false) // 테이블의 컬럼 명시, 옵션이 필요한 경우 사용
    private String title;

    @Column(columnDefinition = "TEXT", nullable = false)
    // columnDefinition: 컬럼 타입 설정
    private String content;

    private String author;

    @Builder // 빌더 패턴 클래스 생성
    public Posts(String title, String content, String author) {
        this.title = title;
        this.content = content;
        this.author = author;
    }

    public void update(String title, String content) {
        this.title = title;
        this.content = content;
    }
}
```

- **`@Entity`**: 테이블과 링크될 클래스를 명시하는 어노테이션. `@Table` 어노테이션을 통해 테이블의 이름을 명시하지 않으면 클래스명과 동일한 이름의 테이블과 링크된다.
- **`@Id`**: 테이블의 PK 필드로 지정하는 어노테이션
- **`@GeneratedValue`**: PK의 생성 규칙을 지정하는 어노테이션, `GenerationType.IDENTITY`는 기본키 생성 규칙을 데이터베이스에 위임한다는 뜻
- **`@Column`**: 테이블의 컬럼을 명시하는 어노테이션, `length`나 `nullable` 및 `columnDefinition` 등으로 해당 컬럼 옵션을 지정
- **`@Builder`**: 빌더 패턴으로 객체를 생성할 수 있음

> `Builder` 클래스는 **기본적으로 생성자와 기능은 같다.**
> 생성자로 작성하게 되면 **자바의 다형성 특징에 의해 기입해야할 데이터에 잘못 기입하는 오류가 발생**할 수 있지만
> `Builder` 클래스 사용으로 **어떤 필드에 어떤 데이터를 기입해야 할지 명확하게 인지**할 수 있다.

## OAuth2 구글 인증

OAuth2 Client를 활용하여 구글 아이디로 사용자 인증을 받을 수 있도록 작성했다. Spring Boot의 OAuth2 Client는 OAuth2UserService 클래스를 제공하여 OAuth2 인증을 구현할 수 있도록 도와준다.

### CustomOAuth2UserService

OAuth2UserService 인터페이스를 상속받아 커스텀 OAuth2UserService 클래스를 작성했다.

```java
// config/auth/CustomOAuth2UserService

package springboot.config.auth;

import lombok.RequiredArgsConstructor;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.oauth2.client.userinfo.DefaultOAuth2UserService;
import org.springframework.security.oauth2.client.userinfo.OAuth2UserRequest;
import org.springframework.security.oauth2.client.userinfo.OAuth2UserService;
import org.springframework.security.oauth2.core.OAuth2AuthenticationException;
import org.springframework.security.oauth2.core.user.DefaultOAuth2User;
import org.springframework.security.oauth2.core.user.OAuth2User;
import org.springframework.stereotype.Service;
import springboot.domain.user.User;
import springboot.domain.user.UserRepository;
import springboot.web.dto.OAuthAttributes;
import springboot.web.dto.SessionUser;

import javax.servlet.http.HttpSession;
import java.util.Collections;

@RequiredArgsConstructor
@Service
public class CustomOAuth2UserService implements OAuth2UserService<OAuth2UserRequest, OAuth2User> {

    private final UserRepository userRepository;
    private final HttpSession httpSession;

    @Override
    public OAuth2User loadUser(OAuth2UserRequest userRequest) throws OAuth2AuthenticationException {
        OAuth2UserService<OAuth2UserRequest, OAuth2User> delegate = new DefaultOAuth2UserService();
        OAuth2User oAuth2User = delegate.loadUser(userRequest);

        // 로그인 진행 중인 서비스 식별자
        String registrationId = userRequest
                .getClientRegistration()
                .getRegistrationId();

        // OAuth2 로그인 시 키가 되는 필드 값 얻어옴 / PK 같은 필드
        String userNameAttributeName = userRequest
                .getClientRegistration()
                .getProviderDetails()
                .getUserInfoEndpoint()
                .getUserNameAttributeName();

        // OAuthAttributes: OAuth2User의 attribute를 담을 클래스
        OAuthAttributes attributes =
                OAuthAttributes.of(registrationId, userNameAttributeName, oAuth2User.getAttributes());

        User user = saveOrUpdate(attributes);

        httpSession.setAttribute("user", new SessionUser(user));

        return new DefaultOAuth2User(
                Collections.singleton(new SimpleGrantedAuthority(user.getRoleKey())),
                attributes.getAttributes(),
                attributes.getNameAttributeKey()
        );
    }

    private User saveOrUpdate(OAuthAttributes attributes) {
        User user = userRepository.findByEmail(attributes.getEmail())
                .map(entity -> entity.update(attributes.getName(), attributes.getPicture()))
                .orElse(attributes.toEntity());

        return userRepository.save(user);
    }
}
```

**DefaultOAuth2UserService**

표준 `OAuth 2.0 Provider`를 지원하는 `OAuth2UserService` **기본 구현체**로 생성자를 통해 OAuth2UserService 클래스의 기본 인스턴스를 반환한다.

**registrationId**

현재 로그인 진행 중인 서비스를 구분하는 코드로 여러 OAuth2 제공 서비스를 구분하는데 사용할 수 있다.

**userNameAttributeName**

OAuth2 로그인 시 유저 정보의 Key가 되는 필드값, 테이블의 PK 같은 것
구글은 `sub`라는 키로 유저의 PK를 제공한다.

**DefaultOAuth2User**

OAuth2User의 기본 구현체로 OAuth2 인증을 통해 받아온 User 정보들을 담은 객체이다.

```java
public DefaultOAuth2User(Collection<? extends GrantedAuthority> authorities,
					Map<String, Object> attributes, String nameAttributeKey) {}
```

로 정의되어 있으며 유저의 권한과 정보들, 키를 인자로 받아 OAuth2 유저 인스턴스를 반환한다.

OAuth2 인증 요청을 바탕으로 사용자의 속성들을 담는 클래스다.

## Redis 활용

Redis 데이터베이스는 웹 개발을 할 때 Session 또는 캐싱을 위한 데이터베이스로 활용할 수 있다. 본문에서는 후에 서버가 여러 대로 작동할 경우 Session 클러스터링을 위해 Redis로 Session 관리하는 방법을 공부하고 적용해보았다.

### Redis란?

Key-Value 기반의 In-Memory NoSQL 데이터베이스로 모든 데이터를 메모리에 저장하고 조회하기 때문에 빠른 읽기, 쓰기 속도를 보장한다. 때문에 웹 서버에서 활용할 때 사용자의 동작 정보 등을 캐싱하거나 Session 스토어로써 사용된다.

### Spring Boot에서 Redis를 Session Store로 사용하기

```
// resources/application.properties

spring.jpa.show-sql=true
spring.jpa.database-platform=org.hibernate.dialect.MySQL5InnoDBDialect
spring.h2.console.enabled=true

spring.profiles.include=oauth,redis
# profiles.include: application-###.properties의 ###를 사용하여 설정을 포함할 수 있다.
```

```
// resources/application-redis.properties

spring.redis.host=localhost
spring.redis.port=6379
spring.session.store-type=redis
```

`Spring Boot`에서는 `Spring Session Data Redis` 의존성 모듈을 통해 `Session`을 `redis`로 저장하도록 설정할 수 있다. 아래의 `session.sotre-type=redis`에 해당한다.

`Redis` 데이터베이스에 접속하여 읽거나 쓰기 위해서는 `client`가 필요한데
`Spring Boot Starter Data Redis` 의존성 모듈을 통해 손쉽게 사용할 수 있다.
