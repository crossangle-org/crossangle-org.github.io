---
title:  "모노레포 멀티모듈 가이드"
categories:
  - Backend
tags:
  - Spring Boot
  - Gradle
  - Java
toc: true
toc_icon: desktop
toc_label: "목차"
toc_sticky: true
---
<div style="text-align: right;"><span style="font-size: 13px;">Data Analytics Platform 이현민</span></div>

안녕하세요! 크로스앵글의 백엔드 개발자 이현민입니다. 

오늘은 제가 모노레포 멀티모듈의 장단점과 Xangle ERP Performance Analytics 프로덕트 팀에서 진행한 간단한 모듈링 구성을 구현하는 코드를 공유해드리고, 나아가 심화적인 모듈링 구성법과 관리 팁을 전해드리고자 합니다.

# PA에서 멀티모듈을 도입한 이유

Performance Analytics(이하 PA) 팀에는 서비스 api와 스케줄러 프로젝트가 존재했었습니다. 여기에 어드민 api 프로젝트가 들어오게 되었고 추후 다른 프로젝트들도 생길것으로 예상되었습니다. 

이렇게 프로젝트마다 다른 레포지토리로 분리해서 관리할 경우 공통 코드에 대해 중복작업이 필요해지고 일관성있게 관리해줘야합니다. 예를 들어 DB 테이블에 칼럼이 하나 추가되었다고 하면 모든 프로젝트를 돌면서 수정하고 각각 PR을 올려서 머지해줘야 하죠. 

그런데 개발자도 사람이다보니 일을 하다보면 이런 수정 작업을 누락하게 될 수 있고 이는 이슈로 이어집니다. 또한 코드가 여러 프로젝트에 산재해있다보니, 처음엔 동일했던 코드가 각자 다르게 변화할 가능성이 커지고, 이런 상황이 온다면 유지보수 난이도가 높아질 것입니다. 

그래서 PA팀에선 3개의 모듈을 하나의 레포지토리로 합쳐서 멀티모듈 구조로 진행하고 있습니다.

Entity, Enum, DB 설정, 예외코드 등 공통적으로 가져가야할 내용들을 core모듈에 넣어놓고, 각 하위 모듈에서 core모듈을 의존하는 간단한 구조의 모듈링 구조입니다. 모든 프로젝트에서 같은 entity와 enum타입을 바라보게 되기 때문에 중복 코드가 줄어들고, 한 곳에서 관리하기 때문에 일관성 유지가 쉬워집니다. 각 app 모듈은 하위모듈들을 포함해서 실행 가능한 빌드파일을 만들 수 있고 각각 다른 서버 혹은 다른 포트에서 독립적으로 실행이 가능해집니다.

![mtmd-01](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2024/08-02/mtmd-01.png)

# 멀티모듈 설정법

멀티모듈을 구성하는 것 자체는 굉장히 쉽습니다. 저를 한 번 따라해 보시죠!

## 모듈 생성

먼저 gradle 프로젝트를 생성하고, `root 디렉토리에서 우클릭` > `New` > `Module`을 눌러 새 모듈을 생성할 수 있습니다. 하위 모듈도 gradle 프로젝트로 생성해줘야겠죠.

root 디렉토리에 있던 src 디렉토리는 사용하지 않을테니 제거해도 무방합니다.

![mtmd-02](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2024/08-02/mtmd-02.png)
![mtmd-03](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2024/08-02/mtmd-03.png)

## setting.gradle

루트 프로젝트의 setting.gradle 파일에 아래와 같이, 루트 프로젝트에 포함된 모듈들을 명시하는 내용을 추가해줍니다.
```groovy
rootProject.name = 'multi-module-example'
include 'module-core', 'module-service', 'module-admin'
```

## build.gradle

build.gradle은 아래와 같이 설정하면 됩니다.

### jar, bootJar

|  | core | service | admin |
| --- | --- | --- | --- |
| jar | O | X | X |
| bootJar | X | O | O |
core모듈에 대해서는 bootJar가 생성되지 않도록 하고, 실제 프로젝트모듈은 bootJar가 생성되도록 설정합니다.
- `bootJar` : core모듈은 실행가능한 애플리케이션이 아니기 때문에 비활성화합니다.
- `jar` : 일반 JAR파일로 빌드해 다른 모듈이 의존성으로 사용할 수 있도록 합니다.

### core 모듈의 build.gradle

core 모듈의 build.gradle은 아래와 같이 작성해줍니다. dependencies에 의존성을 추가하게 되면 core를 의존하는 모듈들에서도 해당 의존성을 동일하게 의존하게 됩니다.

```groovy
plugins {
    id 'java'
}

bootJar {
    enabled = false
}

jar {
    enabled = true
}

dependencies {
    // 공통 라이브러리의 의존성을 여기에 추가
}
```

### service 모듈의 build.gradle

service 모듈의 build.gradle은 아래와 같이 설정해주시고, dependencies에는 core모듈을 의존하는 구절을 추가해주고, 해당 프로젝트에 필요한 의존성들을 추가해줍니다.

```groovy
plugins {
    id 'org.springframework.boot' version '2.5.4'
    id 'io.spring.dependency-management' version '1.0.11.RELEASE'
    id 'java'
}

bootJar {
    enabled = true
}

jar {
    enabled = false
}

dependencies {
    implementation project(':module-core') // 의존하는 모듈을 추가
    implementation 'org.springframework.boot:spring-boot-starter'
}
```

## application.yml

### core 모듈의 application-core.yml

모든 프로젝트가 공유할 공통설정을 core모듈의 application-core.yml파일에 삽입해줍니다.

예시에선 DB설정을 추가해주었습니다.

```yaml
spring:
  application:
    name: module-core
  sql:
    init:
      continueOnError: true
  jpa:
    generateDdl: false
    showSql: false
    hibernate:
      ddlAuto: none
    properties:
      hibernate:
        show_sql: true
        format_sql: true
        use_sql_comments: true
        orderInserts: true
        orderUpdates: true
        dialect: org.hibernate.dialect.PostgreSQLDialect
        formatSql: true
        jdbc:
          batchSize: 1000
          lob:
            nonContextualCreation: true
  datasource:
    url: jdbc:postgresql://localhost:5432/postgres
    driver-class-name: org.postgresql.Driver
    username: postgres
    password: 
    hikari:
      schema: public
      connection-test-query: SELECT 1
      connection-init-sql: SELECT 1
      maximum-pool-size: 100
      minimum-idle: 10
      idle-timeout: 30000
      connection-timeout: 30000
      auto-commit: true
      read-only: false
      data-source-properties:
        reWriteBatchedInserts: true
      pool-name: pa-postgres-db-pool

```

### module-service 모듈의 application.yml

application-core.yml의 core profile를 include한다고 지정하면 service 모듈에선 core모듈의 yml을 참조할 수 있게 됩니다. core모듈의 yml에 DB 설정값을 넣어놓았으니, service 모듈의 yml에는 별도의 DB 설정을 안 해도 core모듈의 DB설정값을 그대로 참조할 수 있게 됩니다.

```yaml
server:
  port: 8080

spring:
  profiles:
    include: core

```

## 패키지 스캔 설정

마지막으로 Application.class에 스캔 및 활성화 설정이 필요합니다.

아래 예시의 경우, base package가 `com.multimodule.service`인데요, Entity는 `com.multimodule.core`에 존재합니다. Entity가 base package 밖에 존재하기 때문에, 자동적으로 스캔하지 않기 떄문에 직접 스캔 경로를 추가해줘야하는 것입니다. JpaRepository, Bean 또한 마찬가지로 추가 스캔 설정을 필요로합니다.

```java
package com.multimodule.service;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.autoconfigure.domain.EntityScan;
import org.springframework.data.jpa.repository.config.EnableJpaRepositories;

// 필수
@EntityScan(basePackages = {"com.multimodule.core.entity"})

// core에 repository를 둘거라면 필요
@EnableJpaRepositories(basePackages = {"com.multimodule.core.repository"})

// core에 bean을 둘거라면 필요
@SpringBootApplication(scanBasePackages = {"com.multimodule.service", "com.multimodule.core"})
// 아니라면 이거 적용
@SpringBootApplication
public class ServiceApplication {

	public static void main(String[] args) {
		SpringApplication.run(ModuleProject1Application.class, args);
	}

}

```

## 모듈링 완성 ✨

이렇게 하면, service나 admin 모듈에서 core 모듈의 클래스를 라이브러리처럼 접근 가능합니다.

아래 코드는 admin 모듈에서 PostRepository 코드입니다. import문을 보면 core 모듈의 PostEntity에 접근 가능하다는 걸 볼 수 있죠.

![mtmd-04](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2024/08-02/mtmd-04.png)

# 멀티모듈 구성 전략

지금까지 core모듈에 의존하는 간단한 방식의 모듈화에 대해 알아봤습니다. 그런데 과연 이 구조가 최선일까요? 모듈링은 구현은 쉽지만 전략을 정말 신중하고 명확하게 가져가는 것이 정말 중요합니다. 한 번 엉키게 된 의존성은 다시 풀어헤치기가 어렵기 때문이에요. 

## core 모듈에만 의존하는 구성의 문제

처음에는 core-app의 2 레이어 방식도 심플하고 괜찮게 운영이 가능합니다. 하지만 해를 거듭하며 프로젝트를 계속 진행하다보면 유지보수가 어려워집니다. 왜일까요?

각 상위 모듈에서 중복되는 내용들을 모두 core에 넣게되면, 일단 core 자체가 너무 무거워집니다. core 내부적으로 의존하는 내용들이 많아지고, 이는 꼬이고 꼬여 점점 스파게티 코드로 진화할 수 있습니다. 이렇게 한 번 꼬여버리면 다시 풀어 헤치기도 어렵죠. 풀어 헤치기 위해 한 로직을 core에서 제거하면 core내의 다른 로직들이 영향을 받아 에러가 발생할 수 있고, 의도치 않게 다른 app 모듈의 로직을 해칠 수 있기 때문이에요.

![mtmd-05](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2024/08-02/mtmd-05.png)

만약 core모듈에 의존하는 app이 2개에서 점점 늘어나 10개, 20개가 된다면 어떻게 될까요? 각 app 모듈들은 분명 자신에게 필요없는 core모듈의 일부분까지도 의존하게 될 것이고, 사용하면 안되는 클래스를 사용하게 된다던지 무분별하게 아무 클래스에 접근이 가능해집니다. 또한 core모듈의 설정에 의해 모든 application이 영향을 받는 문제가 있습니다.

**즉, 의존성과 결합도가 강해집니다.** 

모듈링의 핵심은 코드 재사용을 통해 개발생산성을 올리는 것이고, 그러기 위해선 모듈 간 책임을 명확하게 나누고, 모듈간 영향력을 최소로 하고, 각 모듈은 필요로하는 최소한의 모듈만 의존함으로써, 일관성을 지키고 독립적으로 실행시키는 것이 중요합니다.

![mtmd-06](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2024/08-02/mtmd-06.png)

## core엔 repository까지만 (Performance Aanlytics 팀 채택)

Performance Aanlytics 팀에선 core-app의 2레이어 방식을 채택했습니다. 모듈링 도입 초기인데 모듈링 구조를 너무 복잡하게 가져갔다가 관리가 어려워질까 싶어 심플한 방식을 채택한 것입니다. 

대신 core모듈이 비대해지는 것을 경계하고 의존성 스파게티를 피하기 위해 core에 넣을 대상을 명확하게 정의하기로 했습니다.

core모듈에 들어가는 내용은 아래와 같습니다.

- entity
- repository
- util
- enum
- 예외 핸들링 관련 클래스
- DB 설정

이렇게 했을 경우 타입정보와 예외코드를 일관적으로 관리할 수 있고, entity, enum 등과 같은 관련 중복 코드를 줄일 수 있습니다.

모듈 구조를 도입할 때 딱 이렇게 하라고 추천드리는 건 아니지만, 요지는 core모듈은 얇게 가져가고 웬만한 건 각 프로젝트에서 중복적으로 만들고 설정하는 게 유연하게 관리 가능한 방식이라는 것입니다. 

이 예시에선 repository까지 core모듈에 뒀지만, 실제로 모듈링을 처음 도입할 땐, core 모듈에선 gradle dependency에 아무것도 넣지 않고, entity, enum과 같은 타입정보만 두는 정도로 core를 얇게 두는 방식으로 모듈링을 시작하시는 걸 추천드립니다.

일단 처음엔 얇은 core모듈로부터 모듈 구조를 적용해보고, 개발해가면서 천천히 추가 정책을 도입하는 게 안정적입니다.

그리고 중복을 제거하려는 강박을 갖지 말고, 웬만한 건 그냥 상위 모듈에 중복적으로 작성하더라도 core모듈의  결합도를 낮게 유지하는 걸 추천드립니다.

한 번 core에 들어간 코드는 이미 모든 모듈들이 의존하게 되기 때문에 다시 빼기가 어려울 수 있다는 점을 인지해주세요.

![mtmd-07](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2024/08-02/mtmd-07.png)

## 3 레이어 (core - domain - app)

core와 app 레이어 사이에 domain 레이어가 끼는 형태입니다.

소프트웨어 아키텍처에서 도메인 주도 설계(DDD)를 따르는 한 방법입니다.

<br />

**`core`**  : 순수 자바 코드인 type, util 정도만 포함합니다.

**`domain`** : entity와 repository, service를 둡니다. domain 모듈에는 애플리케이션 비즈니스와 같은 내용은 배제하고 entity CRUD 등 domain 자체의 로직인 도메인 로직만 포함되어야 합니다.

**`app`** : 하위 모듈들을 이용해 애플리케이션 비즈니스 로직을 완성합니다.

<br />

 🤨 **애플리케이션 비즈니스? 도메인 비즈니스?** 

**`애플리케이션 비즈니스(app)`** : 서비스에 대한 플로우를 제어

**`도메인 비즈니스(domain)`** : 도메인 단위에서 생성/변경/소멸의 라이프 사이클을 가짐

> 예시) 게시글 생성 요청 API
> 
> 1. 요청값 검증 (app)
> 2. 게시글 게시 요청 (app)
>     1. 게시글 데이터 생성 (domain)
>     2. 게시글 데이터 검증 (domain)
>     3. 게시글 데이터 저장 (domain)
> 3. 게시글 결과 처리 (app)
> 4. 응답 (app)

### 장점

- 각 app 모듈은 필요한 domain만 의존해서 비즈니스로직을 완성할 수 있습니다.
- 각 app 모듈에선 이미 만들어진 domain을 재사용해서 개발 생산성을 높일 수 있습니다
- 각 모듈이 domain에 대한 단일책임을 갖기 때문에 해당 모듈만 수정하면 모든 모듈에서 일관되게 수정됩니다.

### 단점

- 각 도메인을 명확하게 분리하는 게 어려울 수 있습니다. 초기 설정이 복잡할 수 있습니다.
- 각 도메인마다 필요한 의존성을 넣어주다보면 의존 관계가 복잡해질 수 있습니다.
- domain 레이어의 변경사항이 app 모듈들에 끼치는 영향을 인지해야합니다.
    - 특히 다른팀이 작업하는 모듈에 가는 영향에 대해 잘 공유해야 하므로, 의사소통 비용이 있습니다.

![mtmd-08](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2024/08-02/mtmd-08.png)

domain 모듈은 1개로 시작합니다. 섣불리 쪼개는 것을 지양하고, 일단은 큰 단위(Bounded Context)로 쪼개는 것이 복잡성을 줄입니다.

[참고] [BoundedContext by Martin Fowler](https://martinfowler.com/bliki/BoundedContext.html)

![mtmd-09](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2024/08-02/mtmd-09.png)

# 멀티모듈 구성 팁

지금까지 멀티 모듈 구성 예시에 대해 알아보았습니다. 

이제 멀티모듈을 직접 구성할 때 알면 좋은 팁들을 알려드릴게요!

### 하위 모듈에 일괄 적용할 사항
gradle에서 allprojects와 subprojects 키워드를 사용하면, 루트 프로젝트 혹은 하위 프로젝트들에 일괄적인 설정을 적용할 수 있습니다. 

- `allprojects` : 루트 프로젝트를 포함한 모든 프로젝트에 일괄 설정
- `subprojects` : 루트 프로젝트를 제외한 모든 프로젝트에 일괄 설정

```groovy
allprojects {
	apply plugin: 'java'
	apply plugin: 'org.springframework.boot'
	apply plugin: 'io.spring.dependency-management'

	group = 'multi.module'

	tasks.withType(JavaCompile) {
		options.encoding = "UTF-8"
		sourceCompatibility = JavaVersion.VERSION_17
		targetCompatibility = JavaVersion.VERSION_17
	}

	repositories {
		mavenCentral()
	}

	tasks.named('test') {
		useJUnitPlatform()
	}
}

subprojects {
	dependencies {
		// 공통 라이브러리 추가 부분
		implementation 'org.springframework.boot:spring-boot-starter'
		implementation 'org.springframework.boot:spring-boot-starter-web'
		implementation 'org.springframework.boot:spring-boot-starter-data-jpa'

		compileOnly 'org.projectlombok:lombok:1.18.24'
		annotationProcessor 'org.projectlombok:lombok:1.18.24'
		annotationProcessor 'org.springframework.boot:spring-boot-configuration-processor'

		testImplementation 'org.projectlombok:lombok:1.18.20'
		testImplementation 'org.springframework.boot:spring-boot-starter-test'
	}
}
```

## 간편한 패키지 스캔을 위한 팁

위에서 Application 클래스에 @EntityScan, @EnableJpaRepositories, scanBasePackages 등 스캔 관련 설정들이 적용되어야 다른 모듈들의 Bean이나 Entity 등을 스캔할 수 있다고 말씀드렸는데 기억 나시나요?

```java
package com.multimodule.projectone;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.autoconfigure.domain.EntityScan;
import org.springframework.data.jpa.repository.config.EnableJpaRepositories;

// 필수
@EntityScan(basePackages = {"com.multimodule.core.entity"})

// core에 repository를 둘거라면 필요
@EnableJpaRepositories(basePackages = {"com.multimodule.core.repository"})

// core에 bean을 둘거라면 필요
@SpringBootApplication(scanBasePackages = {"com.multimodule.projectone", "com.multimodule.core"})
// 아니라면 이거 적용
@SpringBootApplication
public class ServiceApplication {

	public static void main(String[] args) {
		SpringApplication.run(ModuleProject1Application.class, args);
	}

}

```

위와 같은 설정을 하지 않는 팁이 있는데, Application 클래스의 위치를 한 단계 높이는 것입니다.

예를들어 AdminApplication.class의 경우, com.multimodule.admin이 아닌 com.multimodule에 두는 것입니다. 그럼 com.multimodule이 base package가 되고, com.multimodule.core에 있는 Entity와 Bean, Repository가 자동으로 스캔이 되는 것입니다!

![mtmd-10](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2024/08-02/mtmd-10.png)

그럼 아래와 같이 스캔과 관련한 설정을 넣지 않아도 돼요

![mtmd-11](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2024/08-02/mtmd-11.png)

예시에선 `com.crossangle`을 base package로 삼았지만, 실제 프로젝트는 `com.crossangle.community`와 같이 한 뎁스 더 들어간 패키지를 사용하면 됩니다.

모든 모듈들은 `com.crossangle.xxx`라는 상위 공통 패키지를 갖고 어플리케이션 모듈에서는 `com.crossangle.xxx` 패키지에 Application 클래스를 배치해야 합니다.

예시

- Admin application module   : `com.crossangle.community.app.admin`
- Service application module : `com.crossangle.community.app.service`
- redis module                         : `com.crossangle.community.data.redis`
- postgresql module               : `com.crossangle.community.data.postgresql`
- core                                        : `com.crossangle.community.core`
- Admin, Service application module의 Application class위치는 `com.crossangle.community` 

## implementation, api로 모듈 접근 제한 설정하기

상위 모듈이 하위 모듈을 사용할 때, 상위모듈은 하위 모듈의 하위 모듈의 내용에 대해서는 접근이 불가능해야 의존성을 줄일 수 있습니다. 하지만 반대로 상위 모듈이 하위모듈의 하위모듈의 내용까지 접근 가능하다면 편리한 부분도 있습니다.

이러한 설정을 gradle에서 api와 implementation을 이용해서 지정할 수 있습니다.

**`api`** : 현재 모듈을 의존하는 모듈에서도 사용 가능하게 함 (추이 의존성 허용)

**`implementation`** : 현재 모듈에서만 사용 가능하게 함 (추이 의존성 불가)

아래 이미지 속 예시를 들어 설명드리겠습니다. B가 A를 api모드로 의존하고 있습니다. 그럼 B를 의존하는 C에서도 A의 내용에 접근이 가능합니다. 하지만 C의 경우 B를 implementation 모드로 의존하고 있습니다. 이러면 C를 의존하고 있는 D에서는 B와 그 하위인 A의 내용을 볼 수 없습니다. 

이런 식으로 implementation과 api를 이용해 모듈간 접근 제한을 설정할 수 있습니다.

![mtmd-12](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2024/08-02/mtmd-12.png)

**D Module**

```
implementation project(':C')
```

```
public class D {
  public void act() {
    new B() <-- compile error
  }
}
```

**C Module**

```
implementation project(':B')
```

**B Module**

```
public class B
```

implementation을 사용하면 각 모듈들에서 불필요한 추이 의존성을 방지할 수 있습니다. 그래서 최대한 api 대신 implementation을 사용하는 것이 좋습니다.

## 빈 관리

모듈 구조를 사용하다보면 빈 관리의 어려움을 마주할 수 있습니다. 가령, 하위 모듈에서 정의한 Bean을 상위모듈에서 확장하거나 변경하고싶은 경우가 있습니다. 모듈 구조에서 Bean을 관리하는 팁을 몇 가지 알려드리겠습니다.

### @ConditionalOnMissingBean 활용

이 애너테이션은, 기존재하는 Bean이 없다면 @ConditionalOnMissingBean 애너테이션이 적용된 내용에 대해 새 Bean을 생성하도록하는 애너테이션입니다. 즉, 이 애너테이션을 하위 모듈에 적용하고, 상위 모듈엔 별도 빈을 설정하는 방식을 선택하면, 하위 모듈의 빈이 default Bean 역할을 하게 됩니다. 상위 모듈에 빈이 있으면 하위 모듈의 빈이 생성되지 않고, 상위 모듈에 빈이 없다면 애너테이션 조건에 의해 하위 모듈의 빈이 생성됩니다.

- 하위 모듈

```java

@Configuration
public class LowerModuleConfig {

    @Bean
    @ConditionalOnMissingBean
    public MyService myService() {
        return new MyService() {
            @Override
            public void serve() {
                System.out.println("하위 모듈");
            }
        };
    }
}
```

- 상위 모듈

```java
@Configuration
public class UpperModuleConfig {

    @Bean
    public MyService myService() {
        return new MyService() {
            @Override
            public void serve() {
                System.out.println("상위 모듈");
            }
        };
    }
}
```

@ConditionalOnMissingBean 어노테이션은 특정 타입뿐만 아니라 이름, 어노테이션 등 다양한 조건을 설정할 수 있습니다.

```java
// 타입 기준
@Bean
@ConditionalOnMissingBean(MyService.class)
public MyService myService() {
    return new MyServiceImpl();
}

// Bean 이름 기준
@Bean
@ConditionalOnMissingBean(name = "customService")
public MyService myService() {
    return new MyServiceImpl();
}

// 애너테이션 기준
@Bean
@ConditionalOnMissingBean(annotation = CustomAnnotation.class)
public MyService myService() {
    return new MyServiceImpl();
}
```

반대로 하위모듈이 아닌 상위모듈에 @ConditionalOnMissingBean을 사용함으로써, 하위 모듈에 특정 Bean이 존재하지 않으면, 상위모듈에서 default Bean을 설정할 수 있습니다. 이러면 빈 생성의 주도권이 하위 모듈에 존재하게 되죠.

@Primary를 사용하면 하위모듈에 빈이 이미 등록되었더라도 새 빈을 만들어 그걸 사용하도록 할 수 있습니다.

@ConditionalOnBean도 기존재 빈이 있다면 새 빈을 생성할 수 있습니다. @Primary와 다른 점은 기존재 빈이 무조건 있어야 된다는 점과, 기존재 빈과는 다른 타입의 빈을 생성할 수 있다는 점입니다.

상황에 따라서 다르게 적용하면 되겠지만, 하위모듈에 @ConditionalOnMissingBean을 적용하고 상위모듈에서 빈 생성의 주도권을 가져가는 방식을 주로 사용하는 게 좋겠습니다. 빈 생성의 Trigger가 한 방향으로 흐르지 않으면 생각치도 못한 빈 생성/대체가 이뤄질 수도 있기 때문입니다. 한 마디로 예측 가능한 빈 생성을 위해서입니다.

![mtmd-13](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2024/08-02/mtmd-13.png)

### Customizer 사용

상위 모듈의 Customizer Bean을 하위 모듈의 Bean이 주입받아, 하위 모듈의 빈 생성 과정에 관여합니다.

![mtmd-14](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2024/08-02/mtmd-14.png)

아래 예시의 경우, 상위 모듈에서 Jackson2ObjectMapperBuilderCustomizer bean을 만들었고, 하위 모듈에 이 bean을 주입해, 하위 모듈의 bean 생성 과정에서 상위 모듈의 bean의 메서드가 사용되어 하위 모듈 bean 생성에 개입됩니다.

즉, 상위 모듈이 하위 모듈의 생성에 영향을 주게 된 것이죠.

```java
@Bean
public Jackson2ObjectMapperBuilderCustomizer jackson2ObjectMapperBuilderCustomizer() {
    return jacksonObjectMapperBuilder -> jacksonObjectMapperBuilder.featuresToEnable(MapperFeature.DEFAULT_VIEW_INCLUSION);
}
```

```java
@Bean
@ConditionalOnMissingBean
public Jackson2ObjectMapperBuilder jacksonObjectMapperBuilder(
        List<Jackson2ObjectMapperBuilderCustomizer> customizers) {
    Jackson2ObjectMapperBuilder builder = new Jackson2ObjectMapperBuilder();
    builder.applicationContext(this.applicationContext);
    customize(builder, customizers);
    return builder;
}

private void customize(Jackson2ObjectMapperBuilder builder,
        List<Jackson2ObjectMapperBuilderCustomizer> customizers) {
    for (Jackson2ObjectMapperBuilderCustomizer customizer : customizers) {
        customizer.customize(builder);
    }
}
```

## README 작성

다양한 모듈을 만들다보면, 네이밍만으로는 모듈의 의도와 동작방식을 파악하기 힘듭니다. 모든 모듈에 README.md를 작성하면 해당 모듈에 대해 잘 모르던 개발자도 모듈 사용법을 익히고 오용을 방지할 수 있고, 개발환경을 깔끔하게 유지할 수 있습니다.

README엔 아래 내용이 포함되면 좋습니다.

- 역할과 책임
- 실행방법(사용방법)
- 관례

# 마무리

지금까지 모노레포 멀티모듈 구성법에 대해 알아보았습니다. 멀티 모듈을 잘 사용하면 편리할 것도 같지만, 너무 적극적으로 도입했다가 복잡성과 분리에 대한 모호함에 어려움을 느낄 수 있고, 의존성 지옥을 마주할 수 있습니다. 너무 성급하게 도입하지 말고, 여러 모듈 전략을 살펴보고 천천히 도입해보는 게 중요합니다.