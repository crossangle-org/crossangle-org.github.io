---
title:  "Block 데이터 처리를 위해 랭기지를 만든 이유"
categories:
  - backend
tags:
  - blockchain
  - bigdata
toc: true
toc_icon: desktop
toc_label: "목차"
toc_sticky: true
---
<div style="text-align: right;"><span style="font-size: 13px;">작성자: 송재영</span></div>



## Block 데이터 처리를 위해 랭기지를 만든 이유

블록체인의 Block 정보는 유형자산 또는 무형자산의 데이터를 가지고 있으며, 사실상 가치를 지닌 모든 것들이 블록체인 네트워크 상에서 추적되고 거래됩니다. 이러한 과정은 공개적으로 처리되어 투명한 정보를 제공합니다. 그러나 문제는 데이터의 수가 상상을 초월할 정도로 많으며, 현재 수십에서 수백개의 블록체인이 전 세계적으로 존재한다는 것입니다.

이 글에서는 쟁글의 Explorer 스쿼드 백엔드에서 수많은 블록체인 블록 일부 데이터를 정제하는 데 사용되는 기술, 전략 및 체계에 대해 간략하게 소개하겠습니다.

## Block의 데이터들

블록체인에 대해 잘 모르고 다음 데이터들을 보고 있으면 무슨 데이터들인가 모르실것 같습니다.

>그림1 polygon(POL) Block의 Transaction 데이터
![그림1 polygon(POL) Block의 Transaction 데이터](http://upload.techblog.xangle.io.s3.amazonaws.com/2023/07-18/Untitled.png)



위 그림은 블록체인 네트워크인 메인넷에서 1차 가공된 Block Transaction 정보입니다. 

간략하게 설명하면 다음과 같은 데이터 인데....

### blockHash

블록체인에 묶인 블록의 unique한 hash 값이다.

### blockNumber

블록체인에 묶인 increment counting number 의 hash 값이다.

### timestamp

블록체인에 묶인 시간 hash 값이다.

### from

블록체인을 요청한 주소 hash 값이다.

### to

블록체인을 요청 받은 주소 hash 값이다.

눈치 채셨겠지만 모든 블록체인에 묶인 Block 데이터들은 해시 값으로 표현됩니다. 해당 데이터만으로는 컴퓨터가 아닌 인간이 의미를 알기 어려운 데이터 입니다.

따라서 데이터의 2차 가공 및 정제 처리가 필요합니다.

## 데이터의 가공 및 정재 처리

아래 그림은 데이터 가공 및 정제 처리를 위해 블록의 각종 데이터의 해시 값을 DB에서 가져온 뒤, 애플리케이션에서 디코딩하는 예제 코드입니다.

>그림2 Block 데이터를 Decode 처리1
![그림2 Block 데이터를 Decode 처리1](http://upload.techblog.xangle.io.s3.amazonaws.com/2023/07-18/Untitled%201.png)


>그림3 Block 데이터를 Decode 처리2
![그림3 Block 데이터를 Decode 처리2](http://upload.techblog.xangle.io.s3.amazonaws.com/2023/07-18/Untitled%202.png)



너무 간단하기도 하지만 문제는 지금부터 입니다..

## 너무 다양한 Case by Case

위에서 개발한 내용대로 처리가 되면 얼마나 좋을까... 문제는 블록체인의 종류가 너무 많다는 것입니다.

블록체인 데이터마다 해석 방법이 다르며, 블록체인 네트워크 메인넷에서 가져온 원시 데이터 구조도 차이가 있습니다.

아래 그림은 위키에서 검색한 토큰 및 블록체인의 종류입니다.

>그림4 토큰 및 블록체인 종류
![그림4 토큰 및 블록체인 종류](http://upload.techblog.xangle.io.s3.amazonaws.com/2023/07-18/Untitled%203.png)



한 가지 블록체인을 해석 하더라도, 블록에 포함된 Transaction 데이터는 Contract ABI에 따라 데이터 해석 위치와 방법이 달라집니다. 또한, 몇 개의 Contract ABI를 운 좋게 얻더라도, 수십 개 또는 수백 개의 다른 Contract에 대한 블록 데이터를 처리하려면 코드를 개발해야 합니다.

# Block 분석 및 데이터 가공&정재 방법

예를들면 다음과 같습니다.

>그림5 polygon transaction input data
![그림5 polygon transaction input data](http://upload.techblog.xangle.io.s3.amazonaws.com/2023/07-18/Untitled%204.png)



다음과 같은 Block Transaction Input data가 있다고 가정했을 때, 해석하는 방법의 종류는 여러 가지가 있지만 자주 사용하는 방법은 다음과 같습니다.

- Transaction을 처리한 Contract ABI를 역순으로 하는 코드를 만든다.
- ABI의 method 정보가 유효한(유명한) method라면 역순하는 로직을 찾아 해석 처리를 개발한다.
- 각각 input data의 자릿수를 계산하여 특성에 맞는 코드를 개발한다.

우선 input data를 분석하면...

>그림6 transaction input data 분석결과
![그림6 transaction input data 분석결과](http://upload.techblog.xangle.io.s3.amazonaws.com/2023/07-18/Untitled%205.png)



다음과 같이 특정 자리수마다 데이터를 잘라서 다른 split 데이터를 디코딩해야 한다는 결론이 나왔습니다.

디코딩된 결과는 다음과 같습니다.

>그림7 transaction input data decode 결과
![그림7 transaction input data decode 결과](http://upload.techblog.xangle.io.s3.amazonaws.com/2023/07-18/Untitled%206.png)



드디어 뭔가 의미 있는 데이터들이 나오기 시작했습니다. (여기서 무언가를 더 해야 되긴 하지만..)

이처럼 분석과 분석된 내용대로 코드를 만들게 되면 비로소 Block에 가시적 극소수 데이터를 얻을수 있습니다.

## 코드 less의 필요성

위에 설명한대로 수많은 블록체인 & 수많은 contract에 대한 logic을 코드로 구현하기에는 무리가 있었습니다.

분석은 사람이 한다 해도 분석한 내용까지 코드 logic으로 만들기에는 어마어마한 인력이 필요했기 때문입니다.

## 자동 디코딩 처리 및 코드 less 처리 기술

변환 처리에는 다음과 같은 기술이 사용됩니다.

- Java class Library
- Jayway JSONPath

**Java Class Library**

java class library는 JVM(java virtual machine)이 runtime 시 호출 할수 있는 동적 로드 가능토록 처리합니다. block을 정제 하여 처리 하는 application 자체가 java srping 기반 service로 동작 하고 있었기에 JCL(java class library)를 통해 code less를 개발 하게 되었습니다.

**Jayway JSONPath**

JSONPath는 XPath와 유사한 구문을 사용하여 JSON 데이터를 쿼리하기 위한 표현식 언어 입니다. 

Jayway JsonPath는 이러한 JSONPath 표현식을 해석하고 실행하여 JSON 데이터에서 원하는 정보를 찾아내는 기능을 제공하며 이를 통해 JSON 데이터의 특정 필드, 배열 요소, 중첩 구조 등을 쉽게 탐색하고 조작할 수 있습니다. 

쟁글에서 가지고 있는 데이터들은 이기종 DB 에 담겨 표준화 할수 없지만, 원시 데이터를 추출하는데에는 문제가 없었다. 원시데이터를 json로 변환 후 처리를 하면 되기에 기술적으로 적절성과 접근성 안정성을 고려하여 jayway를 선택하였습니다.

## 컨셉

자동 디코딩 및 code less의 컨셉은 다음과 같았습니다.

- 코딩을 모르는 분석팀에서 분석한 결과 logic 처리.
- 쉽고 간단한 문법을 통해 맵핑 정보를 만든다.

## Sequence flow

>그림8 자동 변환 library 시퀀스 플로우
![그림8 자동 변환 library 시퀀스 플로우](http://upload.techblog.xangle.io.s3.amazonaws.com/2023/07-18/Untitled%207.png)



1. admin에서 새로운 Block이나 Contract의 새로운 형태의 데이터 포맷에 대한 payload를 저장
(payload는 백엔드팀에서 만든 문법으로 해당 문법을 익힌 상태에서의 기획자, 개발자 등등 누구나 만들수 있도록 개발)
2. user가 api 호출
3. api에서 db로 Block Transaction 정보를 요청
4. Block Transaction 정보를 받음
5. 해당 데이터를 그대로 쓸수 있는지 새로운 payload에 접목하여 변환 처리 해야하는지 확인
6. 자동 디코딩 Library에 변환 데이터 요청
7. admin을 통해 저장 된 db에서 payload정보를 요청
8. payload정보 가져오기
9. 변환 데이터 로직 실행
10. 변환데이터 반환
11. api 결과값 생성
12. complete!!

## Payload 문법

변환 Libarary의 핵심인 Payload 문법은 다음과 같이 설계&개발 하였습니다.

- trim (빈값 없애기)
- map (맵핑처리)
- value (값처리)
- default (기본값처리)
- join (복수의 다른값과 합치기)
- delimiter (다른값 사이처리)
- dateformat (날짜 변환처리)
- decode (hex 값 decode)
- encode (hash 처리)
- prefix (접두사처리)
- postfix (접미사처리)
- nodes (배열처리)
- merge (두개의 다른 level의 맵핑값을 합치기)

## Library 개발

거의 프로그래밍 랭기지를 만드는것으로 생각 되어 머리가 아팠지만.. 변환 library를 만들지 않을시 더 머리가 아파질것 같았습니다.

각 문법과 문법 사이에 logic이 수행되어야 했고, 문법과 문법은 끝없이 이어져도 정상 작동 해야 했습니다.

예를들어 다음과 같은 Payload 문법도 가능해야 했습니다.

```json
{
  "merge": {
    "prefix": "?",
    "postfix": "",
    "delimiter": "&",
    "nodes": [
      {
        "merge": {
          "delimiter": "=",
          "nodes": [
            {
              "map": {
                "name": "gasPrice"
              }
            },
            {
              "map": {
                "value": "statusType"
              }
            }
          ]
        }
      },
      {
        "merge": {
          "delimiter": "=",
          "nodes": [
            {
              "map": {
                "name": "details[1]::labels::name"
              }
            },
            {
              "map": {
                "value": "value"
              }
            }
          ]
        }
      },
      {
        "map": {
          "default": "endTextTest"
        }
      }
    ]
  }
}
```

위의 조건들을 충족 시키기 위해 composite 처리를 위한 설계를 진행하였으며, composite는 다음 문법을 실행하는 매개체 역할을 하고, leaf는 상단의 composite Payload 문법을 수행하는 역할을 하도록 개발되었습니다.

또한 구동 방식은 B-tree 탐색 구조로 Payload 문법의 순서를 보장 하여 정상 수행토록 설계 되었습니다.

>그림9 Library composite 구조
![그림9 Library composite 구조](http://upload.techblog.xangle.io.s3.amazonaws.com/2023/07-18/Untitled%208.png)



## 완성 된 예제

Payload 설정을 통해 간단한 4가지 예제를 살펴보면..

>그림10 변환 설정 파일 Payload
![그림10 변환 설정 파일 Payload](http://upload.techblog.xangle.io.s3.amazonaws.com/2023/07-18/Untitled%209.png)



test_1_mapping 문법내용대로 변환해야할 데이터(테스트용)의 구조의 어떤 곳을 참조 하는지 보도록 합시다.

위의 payload의 문법 대로라면 deatails[1] 즉 details의 두번째 List 정보에서 name 정보를 맵핑해야 함으로

**“transaction2”** 값이 나와야 합니다.

>그림11 실제 Block data (테스트용) 1
![그림11 실제 Block data (테스트용) 1](http://upload.techblog.xangle.io.s3.amazonaws.com/2023/07-18/Untitled%2010.png)



payload의 test_2_mapping & test_3_trim의 데이터를 보도록 합시다

test_2_mapping은 payload의 문법대로 timeStamp 정보를 맵핑 하여 

**“1684148482”**

test_3_trim은  method의 정보를 맵핑 하기 전에 trim을 통한 빈문자열을 지워야합니다.

**“Start”**

>그림12 실제 Block data (테스트용) 2
![그림12 실제 Block data (테스트용) 2](http://upload.techblog.xangle.io.s3.amazonaws.com/2023/07-18/Untitled%2011.png)



마지막 예제인 test_4_join은 details의 배열 전체에서 hash 값 사이에 “,”을 넣어 순서대로 맵핑 하여

**“0x11111,0x22222”** 가 나와야 합니다.

>그림13 실제 Block data (테스트용) 3
![그림13 실제 Block data (테스트용) 3](http://upload.techblog.xangle.io.s3.amazonaws.com/2023/07-18/Untitled%2012.png)



Library를 통한 결과값은 다음과 같이 정상적으로 맵핑 되어 출력 되는것을 볼수 있습니다.

>그림14 mapping 완료 된 결과값
![그림14 mapping 완료 된 결과값](http://upload.techblog.xangle.io.s3.amazonaws.com/2023/07-18/Untitled%2013.png)



## 결과

자동 디코딩 Library 개발 이전에는 분석한 내용에 따라 코드를 개발하여 로직을 만들어, 다양한 블록체인 및 ABI 없는 Contract에 대처하고 있었으나, 

현재는 Payload에 따른 간단한 디코딩 및 매핑 작업만으로 데이터 정제를 진행하고 있습니다.

대용량 데이터에 대한 부담 없이 새로운 Contract를 발견 하여 분석을 진행 후 엔지니어 뿐만 아니라 비 엔지니어까지 간단하게 설정 한 데이터를 토대로 logic을 수행하게 되었으며, 이로인한 리소스 절약 및 데이터 처리 속도에 지대한 발전을 이루어 냈습니다.

## 마치며

쟁글의 Explorer 스쿼드는 블록체인 관련 대용량 데이터를 핸들링 하고 서비스 하고 있습니다. 다양한 결합도 높은 기술을 적용하여 고가용성, 고성능, 동시성 트래픽 처리를 통해 사용자에게 서비스를 제공 하고 있으며, 쟁글의  견고한 인프라 위에서 높은 기술력을 자랑하고 있습니다.

국내에서는 이정도의 블록체인 트래픽 처리 시스템과 그에 따른 대용량 데이터 처리, 견고한 프로젝트 structure, 아키텍쳐 시스템 설계 지식 및 말도 안되는 트래픽과 데이터등을 처리함으로 나타나는 다양한 트러블 슈팅 경험은 다른회사에서 겪지 못할것이라고 자신합니다.