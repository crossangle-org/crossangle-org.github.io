---
title:  "Spring Batch + MongoDB 가이드"
categories:
  - Backend
tags:
  - Spring Batch
  - MongoDB
  - Java
toc: true
toc_icon: desktop
toc_label: "목차"
toc_sticky: true
---
<div style="text-align: right;"><span style="font-size: 13px;">Web 3 Finance 최태규</span></div>

# Spring Batch  + MongoDB 가이드

안녕하세요 Xangle에서 ERP 프로덕트를 개발하고 있는 W3F 소속 백엔드 개발자 최태규입니다.

사내 주요 DBMS로 MongoDB를 사용하고 있는데 ,이번에는 Spring Batch를 Mongodb와 어떻게 결합해서 사용하는지 설명하고, 어떠한 장점이 있는지 설명하도록 하겠습니다.

## Spring Batch의 장점

Spring Batch에는 여러가지 장점이 있습니다. 그 중에서 현재 ERP 팀과 다른 팀에서 누릴 만한 장점들을 추려봤습니다.

### 1. 메타 데이터 테이블을 통한 실행 이력 관리

![Pasted image 20240614090651.png](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2024/06-17/Pasted_image_20240614090651.png)

Spring Batch에서는 작업이 실행 될 때마다 내장 메타 데이터 스키마에다가 현재 작업의 관련된 정보들을 기록을 하게 됩니다. 

이 정보들은 내가 어떠한 작업이 얼마나 수행되었고, 실패했는지 쉽게 추적할 수 있는 좋은 가이드를 제공합니다.

현재 ERP 팀에서는 배치 작업을 수행할 때 로그를 하나의 로그 파일에만 기록을 하고 있기 때문에 특정 작업이 언제 시작되었는지 추적하기가 어렵다는 문제점이 있었습니다.

간단하게 각 테이블을 설명드리자면

`BATCH_JOB_INSTANCE` : 전체 특정 작업에 대한 내용
`BATCH_JOB_EXECUTION` : 특정 작업이 실행 되었을 때의 내용
`BATCH_JOB_EXECUTION_PARAMS` : 특정 작업이 실행 되었을때 사용한 변수들
`BATCH_JOB_EXECUTION_CONTEXT` : 특정 작업의 상태를 저장하고 있음
`BATCH_STEP_EXECUTION` : 작업이 실행 됐을 때 해당 작업의 각 단계별 실행 정보를 저장하고 있음
`BATCH_STEP_CONTEXT` : 각 단계별 실행 정보의 상태를 저장하고 있음

위와 같이 메타데이터 정보를 통해서 실행 이력들을 관리하고 살펴볼 수 있는 장점이 있습니다.

하지만 현재 Spring Batch에서는 위 메타데이터 정보를 RDB에다가만 저장을 할 수가 있다는 단점이 있습니다.

다행스럽게도 현재 기준(24년 6월 기준) Spring Batch 팀에서 메타 데이터 스키마를 MongoDB에다가 저장 할 수 있는 기능이 POC 진행 중이고 (23년 10월부터 POC 진행 중) 베타 기능에서 메타데이터 정보를 MongoDB에다가 저장 할 수 있게 되었습니다. 아마 조만간 릴리즈가 된다면 MongoDB에다가 메타 데이터를 저장 할 수도 있지 않을까합니다.

참고 문서 : [spring-projects-experimental/spring-batch-experimental (github.com)](https://github.com/spring-projects-experimental/spring-batch-experimental?tab=readme-ov-file#enabling-experimental-features)

### 2. 재시작 및 에러 핸들링이 용이하다

![Untitled](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2024/06-17/Untitled.png)

현재 ERP 팀에서 배치 작업을 수행 할 때 위와 같이 MongoDB 예외가 발생하는 경우가 종종 있습니다. 만약 이렇게 에러가 발생 할 경우 해당 지갑의 상태가 고정되어서 적재가 더 이상 되지 않거나 하는 문제가 발생하고 있습니다.

이럴경우 개발자가 수동으로 해당 작업의 상태를 수동으로 변경해줘야 하는 번거로움이 있었습니다.

Spring Batch에서는 해당 예외가 발생했을때 예외 타입, 그리고 반복 정책에 따라 재시작을 할지 아니면 해당 데이터를 건너뛰는 skip도 설정 할 수 있습니다.

```java
@Bean
public Job footballJob(JobRepository jobRepository, Step playerLoad, Step gameLoad, Step playerSummarization) {
	return new JobBuilder("footballJob", jobRepository)
				.start(playerLoad)
				.next(gameLoad)
				.next(playerSummarization)
				.build();
}

@Bean
public Step playerLoad(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
	return new StepBuilder("playerLoad", jobRepository)
			.<String, String>chunk(10, transactionManager)
			.reader(playerFileItemReader())
			.writer(playerWriter())
			.build();
}

@Bean
public Step gameLoad(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
	return new StepBuilder("gameLoad", jobRepository)
			.allowStartIfComplete(true)
			.<String, String>chunk(10, transactionManager)
			.reader(gameFileItemReader())
			.writer(gameWriter())
			.build();
}

@Bean
public Step playerSummarization(JobRepository jobRepository, PlatformTransactionManager transactionManager) {
	return new StepBuilder("playerSummarization", jobRepository)
			.startLimit(2)
			.<String, String>chunk(10, transactionManager)
			.reader(playerSummarizationSource())
			.writer(summaryWriter())
			.build();
}

```

또한 예외 핸들러를 통해 해당 예외가 발생하면 처리해야 할 로직을 정의 할 수도 있습니다. 이렇게 정의를 한다면 관심사를 분리시켜서 로직이 좀 더 깔끔해지고 읽기 쉬워진다는 장점이 있습니다.

```java
@FunctionalInterface
public interface ExceptionHandler {

    /**
     * Deal with a Throwable during a batch - decide whether it should be re-thrown in the     * first place.     * @param context the current {@link RepeatContext}. Can be used to store state (via
     * attributes), for example to count the number of occurrences of a particular     * exception type and implement a threshold policy.     * @param throwable an exception.
     * @throws Throwable implementations are free to re-throw the exception
     */    void handleException(RepeatContext context, Throwable throwable) throws Throwable;

```

### 3. 트랜잭션 관리

Spring Batch에서 자동으로 배치 작업에 대해 트랜잭션 관리를 해주고 있습니다.

MongoDB에서도 레플리카셋에 대해서 트랜잭션을 지원하고 있습니다. 트랜잭션을 통해서 ACID를 지원 할 수 있습니다.

Spring Batch에서 지원하는 트랜잭션 관리에 대한 컨셉은 다음과 같습니다.

```
1   |  REPEAT(until=exhausted) {
|
2   |    TX {
3   |      REPEAT(size=5) {
3.1 |        input;
3.2 |        output;
|      }
|    }
|
|  }

```

데이터를 읽고 쓰는 과정에 전체를 트랜잭션으로 처리를 하게 됩니다. 예외 상황에서는 해당 청크 전체를 롤백하게 됩니다.

위 3개의 장점 말고도 비동기, 멀티스레딩, 모듈로 된 작업을 조립해서 새로운 작업을 만드는 등 여러 장점을 가지고 있습니다.

# Spring Batch Chunk Architecture

![Pasted image 20240614112748.png](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2024/06-17/Pasted_image_20240614112748.png)

Spring Batch의 청크 구조는 위와 같이 설계되어있습니다.

`JobRepository`에서는 `JobLauncher`, `Job` , `Step` 실시간으로 통신해서 메타데이터 테이블에 현재 실행중인 작업의 상태를 기록하게 됩니다.

배치 작업이 실행 될 때는 `JobLauncher`에서 사전 구성된 `Job`을 가져와서 실행하게 됩니다.

Job과 Step은 1:N 관계로 구성되어 있습니다. `Step`은 `Job`의 일부분이며 최소 1개 이상 존재합니다. 그리고 기본적으로 순차적으로 실행되게 됩니다.

그 다음으로 Step은 총 세개의 단계로 구성 되어있습니다. 데이터를 읽고 > 가공 하고 > 저장한다.

각각의 단계는 `ItemReader`, `ItemProcessor`, `ItemWriter` 로 실행하게 됩니다.

Step의 각 `ItemReader` , `ItemProcessor`, `ItemWriter`는 각각 인터페이스이며 여러 구현체가 존재합니다. 가장 많이 사용하는 RDBMS 형태의 `JdbcItemReader`, `JpaItemReader`등이 존재하며 `RedisItemReader` `KafkaItemReader` 들도 존재합니다.

`ItemProcessor`에서는 가져온 데이터를 가공하는 작업을 수행합니다. 데이터를 필터링 하거나 비즈니스 로직을 수행하는 단계입니다. 따로 구현체가 존재하지는 않고 개발자가 직접 인터페이스의 구현체를 개발해야 합니다.

`ItemWriter` 에서는 가공한 데이터를 저장하는 역할을 수행합니다.

다음으로는 `ItemReader`에 대해서 좀 더 자세히 알아보도록 하겠습니다.

## ItemReader

배치 성능 퍼포먼스에 가장 중요한 부분이 데이터를 어떻게 읽을것인지에 관한 부분입니다. Spring Batch에서는 총 두가지 타입의 `ItemReader`가 존재합니다

커서 기반이랑 페이징 기반 두 가지 방식으로 데이터를 읽는 방식이 있습니다. 각각의 장단점이 있으므로 상황에 맞는 방식을 고르는 것이 중요합니다.

### 커서 기반 ItemReader

이 방식은 DBMS의 `커서`를 이용하는 방식입니다. `커서`를 사용하면 데이터를 스트리밍 방식으로 실시간으로 받을 수 있다는 장점이 있습니다. 커서를 청크 사이즈만큼 이동시켜서 데이터를 읽고 처리하는 작업을 반복해서 수행하게 됩니다.

기본적으로 커서 방식은 데이터셋의 사이즈와 별개로 성능이 빠르고 일관적이다 라는 장점이 있습니다. 대규모의 데이터를 처리할 때 적합 하지만 병렬로 처리할 경우 thread-safe하지 않다는 단점이 있습니다.

MongoCursorItemReader 구현체 또한 5.1.0 버전에서 신규로 추가가 되었습니다.

### 페이징 기반 ItemReader

이 방식은 데이터베이스를 페이징 방식으로 데이터를 불러오는 방식입니다. RDBMS에서는 limit,offset 쿼리를 사용하고 MongoDB에서는 skip(), limit()을 사용해서 데이터를 가져옵니다.

페이징 기반은 skip() limi()을 사용하는 태생적인 한계덕분에 대량의 데이터를 처리 할 경우 속도가 저하됩니다.

MongoDB 레퍼런스에서는 다음과 같이 설명하고 있습니다.

> The skip() method requires the server to scan from the beginning of the input results set before beginning to return results. As the offset increases, skip() will become slower.
> 

> Skip() 메서드를 사용하려면 서버가 결과 반환을 시작하기 전에 입력 결과 집합의 시작 부분부터 검색해야 합니다. 오프셋이 증가하면 Skip()이 느려집니다.
> 

한 마디로 표현하면 처음부터 읽어서 내가 몇번째 데이터인지 파악을 해야하기때문에 시간이 오래 걸린다는 단점이 있습니다.

![Pasted image 20240614125045.png](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2024/06-17/Pasted_image_20240614125045.png)

[Add support for MongoDB as JobRepository · Issue #877 · spring-projects/spring-batch (github.com)](https://github.com/spring-projects/spring-batch/issues/877) 에 따르면 테스트 결과 페이징 기반의 `ItemReader` 는 점점 더 느려진다는 점을 알렸습니다.

그리고 한 가지 단점이 더 있습니다.

페이징 기반 `ItemReader`를 사용하면 쿼리에 항상 정렬 조건을 포함해야 합니다.

페이징을 할 때 쿼리 결과가 항상 일관된 결과를 반환하지 않는 DBMS의 경우 정렬 조건을 명시를 해야 합니다.

예시로 MongoDB에서는 쿼리의 결과가 항상 일관되지 않다는 단점이 있습니다.

MongoDB 레퍼런스 [cursor.sort() - MongoDB Manual v6.0](https://www.mongodb.com/docs/v6.0/reference/method/cursor.sort/) 에서는 다음과 같이 표현하고 있습니다.

> MongoDB does not store documents in a collection in a particular order. When sorting on a field which contains duplicate values, documents containing those values may be returned in any order.
If consistent sort order is desired, include at least one field in your sort that contains unique values. The easiest way to guarantee this is to include the _id field in your sort query.
> 

> MongoDB는 문서를 특정 순서로 컬렉션에 저장하지 않습니다. 중복 값이 포함된 필드를 정렬할 때 해당 값이 포함된 문서가 어떤 순서로든 반환될 수 있습니다.
일관된 정렬 순서가 필요한 경우 고유한 값이 포함된 필드를 정렬에 하나 이상 포함하세요. 이를 보장하는 가장 쉬운 방법은 정렬 쿼리에 _id 필드를 포함하는 것입니다.
> 

쿼리가 항상 일관된 정렬 순서를 보장하지 않으면 데이터가 중복 처리 될 수도 있다는 점이 있습니다.

페이징 기반의 `ItemReader`는 중,소규모 데이터셋을 사용할때 효율적이며 thread-safe하다는 장점이 있습니다.

## ItemWriter

`ItemWriter`에서는 가공한 데이터를 저장하는데 사용됩니다.

`MongoItemWriter`에서는 총 3가지 모드를 지원하고 있습니다. delete,upsert,insert 각각의 모드는 전부 bulk 모드로 동작합니다. 각자 용도에 따라 선택해서 사용하는 것이 좋습니다.

## 예제 샘플 코드

```java
@Document("person")
@Getter
@Setter
public class Person {

    @MongoId
    private ObjectId id;

    private String name;

    private int wage;
    }

```

```java
@Configuration
@Slf4j
public class BatchConfig {

    @Bean
    public Job mongoJob(JobRepository jobRepository, Step mongoStep){
        return new JobBuilder("mongoJob",jobRepository) // 작업 이름을 지정
                .incrementer(new RunIdIncrementer())    // 반복 실행하기 위해 설정
                .start(mongoStep)   // 작업의 단계를 구성
                .build();
    }
    @Bean
    public Step mongoStep(JobRepository jobRepository,
                          PlatformTransactionManager platformTransactionManager,
                          MongoCursorItemReader<Person> itemReader,
                          MongoItemWriter<Person> itemWriter,
                          ItemProcessor<Person,Person> itemProcessor
                          ){
        return new StepBuilder("mongoStep",jobRepository)
                .<Person,Person>chunk(100,platformTransactionManager) // 청크사이즈랑 ItemReader,ItemWriter input,output 타입을 구성한다
                .reader(itemReader) //itemReader를 설정
                .processor(itemProcessor)   //ItemProcessor를 설정
                .writer(itemWriter) //ItemWriter를 설정
                .build();
    }

    @Bean
    public MongoCursorItemReader<Person> mongoItemReader(MongoTemplate mongoTemplate){ // MongoCursorItemReader를 설정
        MongoCursorItemReader<Person> itemReader = new MongoCursorItemReader<>();
        itemReader.setBatchSize(100);
        itemReader.setTemplate(mongoTemplate);
        itemReader.setCollection("person");
        itemReader.setQuery(new Query());
        itemReader.setTargetType(Person.class);
        itemReader.afterPropertiesSet();
        return itemReader;
    }
    @Bean
    public ItemProcessor<Person,Person> personItemProcessor(){
        return item -> {
            item.setWage(item.getWage() + 100);

            log.info("이름 : {} 급여 : {} ", item.getName(),item.getWage());
            return item;
        };
    }

    @Bean
    public MongoItemWriter<Person> mongoItemWriter(MongoTemplate mongoTemplate) throws Exception {
        MongoItemWriter<Person> itemWriter = new MongoItemWriter<>();
        itemWriter.setCollection("person");
        itemWriter.setTemplate(mongoTemplate);
        itemWriter.setMode(MongoItemWriter.Mode.UPSERT);
        itemWriter.afterPropertiesSet();
        return itemWriter;
    }
}

```

## 작업을 Jenkins로 실행하기

현재 팀내에서는 스케줄러 형식의 작업을 시작할때는 HTTP 요청을 통해서 작업을 시작하고 있습니다.

1. 스케줄러 서버에서 각 시간마다 수행해야 할 작업을 해당 서버에 HTTP 요청을 보냄
2. 요청을 받은 서버는 작업을 수행

위와 같은 방식으로 스케줄 작업을 수행하고 있습니다.

위와 같은 형태로 작업을 구성했을때 겪었던 몇가지 문제점이 있었습니다.

먼저 첫번째로 로그를 하나의 인스턴스에서 관리를 하고 하나의 파일에다가 기록을 하다보니 로그를 찾기가 매우 어려웠습니다.

여러 작업이 동시에 실행되는 Core 같은 경우에 여러 작업 로그가 뒤섞여서 기록되게 됩니다.

이럴 경우 로그 찾는 작업이 쉽지가 않았습니다.

두번째로 배포를 할 때 조심스러워 진다는 단점이 있었습니다.

매번 작업이 진행중인지 아닌지 체크를 했어야 했었고 작업이 수행되지 않을때만 베포를 할 수 있었습니다.

각 프로세스에는 상태가 기록되어 있는 경우가 많은데 도중에 작업이 중단되거나 서버가 다운되면 작업이 마무리 되지 않고 종료될때가 있었습니다. 이런경우 해당 작업은 계속 멈춰있는 상태로 유지되는 경우도 많았습니다.

위 두가지 문제를 Jenkins를 활용하면 해결 할 수가 있었습니다.

### jenkins cron을 통해 작업 실행하기

![Pasted image 20240614140535.png](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2024/06-17/Pasted_image_20240614140535.png)

젠킨스에는 Build periodically라는 스케줄러가 있습니다.

이를 통해서 주기적으로 젠킨스 item을 실행시킬 수가 있습니다.

젠킨스를 실행시킬때 실행시킬 작업을 지정해야합니다.

spring batch에서는 jar를 실행시킬때 변수를 통해 실행시킬 작업의 이름을 지정하면 해당 작업만 실행시키는 것이 가능합니다.

![Pasted image 20240614140704.png](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2024/06-17/Pasted_image_20240614140704.png)

Build Steps에서 해당 jar 파일과 --job.name 변수를 지정하면 해당 작업만 실행하는것이 가능합니다.

이때 해당 jar가 새 버전이 배포가 된다면 쉘스크립트를 통해서 새로운 버전의 jar 파일을 지정해서 실행시키는 것이 가능합니다.

그렇게 된다면 기존에 실행중이던 배치 작업이 있다고 하더라도 새로운 버전의 jar를 실행시켜서 작업을 실행하면 기존 작업을 중단시키지 않아도 새로운 배치 작업이 실행이 가능합니다.

### 각각의 실행이력을 별도로 관리

![Pasted image 20240614141751.png](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2024/06-17/Pasted_image_20240614141751.png)

위와 같이 jenkins를 사용하면 별도의 실행이력이 계속해서 남게 됩니다.

그리고 각각의 jar 파일에서 기록된 로그들만 출력이 되기 때문에 작업이 여러개 동시에 실행되어도 각각의 작업에 대해서만 이력이 남기때문에 추적하기가 쉽다는 장점이 있습니다.

## 마치며

위 설정은 매우 추상적으로 설정된 개념이다 보니 정확하게 운영에 반영하려면 좀 더 디테일한 설정이 필요합니다.

또한 jenkins뿐만 아니라 Airflow나 Spring Cloud Data Flow로 배치 작업들을 관리하는 곳도 있다고 들었습니다. 최근 지인 회사에서는 Airflow를 활용해서 배치 작업들을 관리한다고 들었습니다.

Spring Cloud Data Flow는 k8s 사용이 선행적이며 클라우드 네이티브 환경에서 장점이 발휘될 수 있다고 들었습니다. SCDF 경우에는 레퍼런스도 많지 않아서 허들이 높다는 단점이 있습니다.