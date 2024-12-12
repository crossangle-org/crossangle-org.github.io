---
title:  "Nexus repository와 커스텀 라이브러리를 활용한 데이터/로직 공유"
categories:
  - Backend
tags:
  - Java
  - Spring
  - nexus
  - library
toc: true
toc_icon: desktop
toc_label: "목차"
toc_sticky: true
---
<div style="text-align: right;"><span style="font-size: 13px;">Portal 송후섭</span></div>

# Java에서 Nexus Repository와 라이브러리 사용하기

## 들어가는 말

소프트웨어 개발 세계에서 효율성과 재사용성은 핵심 원칙입니다. Java 생태계에서 이러한 원칙을 실현하는 두 가지 중요한 도구가 바로 Nexus Repository와 사용자 정의 라이브러리입니다. 이 글에서는 Nexus Repository를 사용하는 이유, 라이브러리의 중요성, 그리고 Spring을 사용하여 간단한 라이브러리를 만들고 배포하는 방법에 대해 알아보겠습니다.

## Nexus Repository는 왜 쓸까?

Nexus Repository는 소프트웨어 아티팩트를 저장, 관리, 배포하는 데 사용되는 저장소 관리자입니다. 다음과 같은 이유로 많은 기업과 개발자들이 Nexus를 선택합니다:

1. **중앙 집중식 아티팩트 관리**: 모든 라이브러리와 의존성을 한 곳에서 관리할 수 있습니다.

2. **빠른 빌드 속도**: 로컬 네트워크 내에서 아티팩트를 가져오므로 빌드 속도가 향상됩니다.

3. **버전 관리**: 아티팩트의 여러 버전을 쉽게 관리하고 롤백할 수 있습니다.

4. **보안**: 외부 저장소에 대한 접근을 제어하고, 내부 아티팩트를 안전하게 보호할 수 있습니다.

5. **프록시 및 캐싱**: 외부 저장소의 아티팩트를 캐싱하여 네트워크 사용량을 줄입니다.

### Nexus와 외부 레포지토리의 통합

Nexus Repository의 큰 장점 중 하나는 외부 오픈 소스 레포지토리와 원활하게 통합된다는 점입니다. 이를 통해 개발자들은 단일 진입점을 통해 모든 필요한 의존성에 접근할 수 있습니다.

1. **통합된 아티팩트 관리**: Nexus는 Maven Central, JCenter 등의 공개 레포지토리를 프록시로 설정할 수 있습니다. 이렇게 하면 외부 레포지토리의 아티팩트와 내부에서 개발한 아티팩트를 함께 관리할 수 있습니다.

2. **캐싱을 통한 성능 향상**: 외부 레포지토리에서 가져온 아티팩트는 Nexus에 캐시됩니다. 이로 인해 같은 아티팩트를 다시 요청할 때 외부 네트워크에 접근할 필요 없이 빠르게 제공받을 수 있습니다.

3. **네트워크 격리**: 개발 환경이 인터넷에 직접 연결되지 않은 경우에도, Nexus를 통해 필요한 외부 의존성을 제공받을 수 있습니다.

4. **의존성 제어**: 특정 외부 라이브러리의 사용을 제한하거나, 특정 버전만 사용하도록 강제할 수 있습니다. 이는 보안과 일관성 유지에 도움이 됩니다.

이러한 통합을 통해 개발자들은 내부 및 외부 라이브러리를 손쉽게 활용할 수 있으며, 동시에 조직은 사용되는 모든 의존성을 중앙에서 관리하고 통제할 수 있습니다.

## 라이브러리를 쓰는 이유?

라이브러리는 재사용 가능한 코드의 집합으로, 다음과 같은 이점을 제공합니다:

1. **코드 재사용**: 공통 기능을 여러 프로젝트에서 사용할 수 있습니다.

2. **유지보수 용이성**: 중앙에서 관리되므로 버그 수정과 기능 추가가 쉽습니다.

3. **표준화**: 일관된 코딩 규칙과 패턴을 적용할 수 있습니다.

4. **생산성 향상**: 이미 검증된 코드를 사용하여 개발 시간을 단축할 수 있습니다.

5. **모듈화**: 애플리케이션을 더 작고 관리하기 쉬운 부분으로 나눌 수 있습니다.

### 회사 내 라이브러리 사용의 이점

회사 내에서 라이브러리를 개발하고 사용하는 것은 여러 면에서 이점이 있습니다. 특히 팀 간 협업과 코드 공유 측면에서 큰 도움이 됩니다.

1. **효율적인 코드 공유**: 개발된 유용한 기능을 라이브러리로 만들어 다른 팀과 쉽게 공유할 수 있습니다. 이는 각 팀에서 유사한 기능을 중복 개발하는 것을 방지합니다.

2. **일관된 코드 스타일**: 공통 라이브러리 사용을 통해 회사 전체적으로 코딩 스타일과 패턴을 일관되게 유지할 수 있습니다. 이는 코드 리뷰 프로세스를 간소화하고 새로운 팀원의 온보딩을 용이하게 합니다.

3. **버전 관리의 편리성**: 라이브러리에 새 기능을 추가하거나 버그를 수정할 때, 단순히 버전을 업데이트하는 것만으로 모든 프로젝트에 변경사항을 적용할 수 있습니다.

4. **팀 간 커뮤니케이션 개선**: 라이브러리 업데이트 시 변경 사항을 공유함으로써 팀 간 자연스러운 커뮤니케이션이 촉진됩니다. 이를 통해 각 팀의 개발 현황과 해결된 문제점들을 공유할 수 있습니다.

5. **지식 및 노하우의 축적**: 시간이 지남에 따라 회사의 핵심 기술과 비즈니스 로직이 라이브러리에 축적됩니다. 이는 회사의 중요한 지적 자산이 됩니다.

6. **온보딩 시간 단축**: 새로운 팀원이 합류했을 때, 이러한 라이브러리들을 통해 회사의 코드 구조와 주요 기능들을 빠르게 파악할 수 있어 적응 시간을 크게 단축할 수 있습니다.

초기에는 라이브러리를 개발하고 관리하는 데 추가적인 노력이 필요할 수 있지만, 장기적으로는 개발 속도 향상과 코드 품질 개선에 큰 도움이 됩니다.

## 라이브러리 배포예제

쟁글에서는 각 서비스의 정보를 서로의 api를 사용하여 좀더 쉽게 공유중입니다.  API 요청부터 데이터 모델링을 하는작업이 매번 생기는 것을 라이브러리로 배포함으로 라이브러리 설치 및 사용으로 좀더 쉽게 사용을 하고 있습니다 이부분에 대한 간단한 예제를 보여드리도록 하겠습니다.

### 1. 데이터 모델링.
API를 통한 데이터를 실제적으로 class화 하여 객체를 사용할수 있도록 class에 대한 정의를 먼저 내려줬습니다.

```java

@Getter
@Setter
public class PortalTokenInfo {
    @JsonProperty("asset_id")
    private String assetId;

    @JsonProperty("name")
    private PortalMultiLangInfo name;

    @JsonProperty("symbol")
    private String symbol;

    @JsonProperty("slug")
    private String slug;

    @JsonProperty("status")
    private String status;

    @JsonProperty("summary")
    private PortalMultiLangInfo summary;

    @JsonProperty("token_description")
    private PortalMultiLangInfo tokenDescription;

    @JsonProperty("logo_url")
    private String logo_url;

    @JsonProperty("coinmarketcap")
    private PortalCoinmarketcapInfo coinmarketcap;

    @JsonProperty("global_price_type")
    private String globalPriceType;

    @JsonProperty("official_website")
    private String officialWebsite;

    @JsonProperty("category_code")
    private List<PortalMultiLangInfo> categoryCode;

    @JsonProperty("other_links")
    private List<PortalOtherLinksInfo> otherLinks;

    @JsonProperty("community")
    private List<PortalCommunityInfo> community;

}

```


### 2. 라이브러리에 로직을 만들어 줍니다.

이제 해당 클래스에 해당하는 API를 호출하여 호출한 결과를 객체화 하는 과정에 대한 예시코드입니다. 

추가적으로 라이브러리의 사용과 로깅 사용하는 위치를 파악하기위해 사용자들은 token을 발급받아서 라이브러를 사용할때 검증을 받는 과정을 추가해주었습니다.

```java

public class PortalToken{
    public static PortalTokenInfo getTokenInfo(String baseUrl, String token, String tokenId){
        TokenValidate.validate(token);
        String subUrl = "/internal/asset/about";
        String params = "asset_id=" + tokenId;
        return ModelGenerator.getModelFromResponse(PortalTokenInfo.class, PortalRequest.get(baseUrl, subUrl, params));
    }
}
```

### 3. 빌드 설정


일단 라이브러리의 명은 `setting.gradle`의 rootProject.name을 따라가게되어있습니다.

`build.gradle` 에는 설정해줄것이 조금더 있습니다.


```gradle
plugins {
    id 'java-library'
    id 'maven-publish'
    id 'org.springframework.boot' version '2.5.5'
    id 'io.spring.dependency-management' version '1.0.11.RELEASE'
}

group = 'com.crossangle'
version = '0.0.1-SNAPSHOT'

bootJar {
    enabled = false
}

jar {
    enabled = true
}

publishing {
    publications {
        mavenJava(MavenPublication) {
            from components.java
        }
    }
    repositories {
        maven {
            url = "http://localhost:8081/repository/maven-releases/"
            credentials {
                username = "admin"
                password = "admin123"
            }
        }
    }
}
```

주의: 여기서 `bootJar` 태스크를 비활성화하고 `jar` 태스크를 활성화했습니다. 이는 실행 가능한 JAR가 아닌 라이브러리 JAR를 생성하기 위함입니다. 이렇게 하면 Spring Boot 애플리케이션이 아닌 일반 Java 라이브러리로 빌드됩니다.

publish 설정으로는 올리고 싶은 레포지토리의 정보와 credential을 추가 하여줍니다.



### 4. 라이브러리 빌드 및 배포

다음 명령어로 라이브러리를 빌드하고 Nexus Repository에 배포합니다:

```bash
./gradlew build publish
```

현재 설정은 로컬 repository의 maven으로 올라가도록 되어있습니다.

## 라이브러기 적용

실제 서비스를 라이브러리가 없었다면 처음부터 http 요청과 해당에 따른 예외처리 response에 대한 모델링 등 작업이 추가가 될것입니다.

이는 사용하는 프로젝트가 많아지면 많아질수록 각각의 구현이 필요하게되고 해당 api에 대한 업데이트의 관리또한 소홀해질수 있습니다.

하지만 우리는 라이브러리를 만들었으니 요청과 예외처리 및 관리는 모두 라이브러리 한쪽의 관리로 통일하면서 사용은 좀더 간편하게 할수 있는 상황이 되었습니다. 

한번 적용예제를 보도록 하겠습니다.

### 5-1. 라이브러리 적용(설정)

이제 다른 프로젝트에서 이 라이브러리를 사용할 수 있습니다. 사용하려는 프로젝트의 `build.gradle` 파일에 다음을 추가합니다:

이때 기본설정은 기본 자바의 mavenCentral을 보지만 우리가 올린 라이브러리를 사용하기 위해서 해당 메이븐에 대한 정보를 추가적으로 설정해줍니다. 

```gradle
repositories {
    mavenCentral()
    maven {
        url "http://localhost:8081/repository/maven-releases/"
    }
}

dependencies {
    implementation 'com.crossangle:"library-name":0.0.1-SNAPSHOT'
}
```

### 5-2. 라이브러리 적용(코드)

이제 실제 service를 하나 만들어보겠습니다.

서비스는 이제 단순히 해당 외부의 함수를 사용하여 불러오는 간단한 예제를 들어봤습니다. 

라이브러리를 통해 쉽게 데이터 요청과 데이터 클래스화 등을 할수 있고 이를 통해 단순히 전달 뿐 아닌 해당 서비스에서의 추가적인 데이터와 결합하여 사용하기 편하도록 만들어주었습니다.

```java
@Service
public class PortalExternalService {
    public PortalTokenInfo getTokenAbout(String baseUrl, String token, String tokenId) {
        return PortalToken.getTokenInfo(baseUrl, token, tokenId);
    }
}
```

## 결론

Nexus Repository와 사용자 정의 라이브러리는 Java 개발 프로세스를 크게 개선할 수 있는 강력한 도구입니다. Nexus를 통해 내부 및 외부 아티팩트를 중앙에서 관리하고, 사용자 정의 라이브러리를 만들어 코드 재사용성을 높임으로써 개발 팀의 생산성과 코드 품질을 향상시킬 수 있습니다.

회사 내에서 라이브러리를 개발하고 공유하는 문화를 만들면, 팀 간 협업이 촉진되고 지식 공유가 활성화됩니다. 이는 단순히 코드를 재사용하는 것 이상의 가치를 창출합니다. 각 팀의 전문성이 라이브러리를 통해 공유되며, 이는 회사 전체의 기술력 향상으로 이어집니다.

라이브러리 개발은 처음에는 약간의 추가 작업이 필요할 수 있지만, 장기적으로 볼 때 개발 시간을 단축하고 코드 일관성을 유지하는 데 큰 도움이 됩니다. 따라서 프로젝트의 규모와 복잡성이 증가함에 따라 이러한 접근 방식을 고려해 보는 것이 좋습니다. Nexus Repository와 사용자 정의 라이브러리를 효과적으로 활용하면, 더 효율적이고 유지보수가 쉬운 소프트웨어를 개발할 수 있습니다.