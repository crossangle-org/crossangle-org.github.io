---
title:  "Elasticsearch Index sorting 을 활용한 성능 개선"
categories:
  - backend
tags:
  - Elasticsearch
  - Index Sorting
toc: true
toc_icon: desktop
toc_label: "목차"
toc_sticky: true
---
<div style="text-align: right;"><span style="font-size: 13px;">작성자: 조원익</span></div>

# 성능 테스트 목표!

**Transaction List API 의 성능 테스트**를 하고자 합니다.

성능은 무조건 높으면 높을수록 좋다. 하지만 목표치가 없다면 무한 뺑뺑이만 돌뿐이다. 그렇기에 목표로 하는 성능을 잡아 보겠습니다.

- VUser: 최대 10,000 명
- TPS: 500 이상

# 테스트에 진행될 Transaction 데이터란?

간단히 말하면 입출금 거래내역을 의미합니다. 현재 시점 기준으로 Polygon Mainnet의 Transaction 총 건수는 약 28억 건 정도입니다. (참고링크: **[PolygonScan](https://polygonscan.com/txs)**) 

현재 이 테스트에서는 약 4억건의 데이터로 테스트 예정입니다.

Transaction 데이터를 서비스하기 위해 Elasticsearch를 사용하고 있으며, 방대한 데이터를 **정렬**하여 제공하는 과정에서 성능 한계를 극복하고자 Index Sorting 기술을 적용하고 그에 따른 성능의 변화를 소개해 보고자 합니다.

Transaction Data 는 아래처럼 생겼습니다.

![Untitled](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/07-31/es-00.png)

# **Index Sorting이란?**

Elasticsearch에서 Index Sorting은 색인된 데이터를 디스크에 더 효율적으로 저장하는 방법입니다. 

일반적으로 Elasticsearch는 데이터를 색인한 후 Lucene 엔진에서 기본적으로 랜덤한 순서로 디스크에 저장합니다. 하지만 Index Sorting을 적용하면 **특정 필드를 기준으로 데이터를 정렬하여 디스크에 저장**할 수 있습니다. 이렇게 하면 특정 필드를 기준으로 정렬된 상태로 데이터를 조회할 때 디스크에서 읽어오는 속도가 향상되어 성능이 향상됩니다.

# Index sorting 적용방법

- Elasticsearch docs ([ElasticsearchGuide](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-index-sorting.html))
- 데이터 적재 후에는 index sorting 을 적용할수 없기 때문에 반드시! index 생성시에 걸어주셔야 합니다.

단일 필드 index sorting (**현재 테스트 적용 방식**)

```json
PUT polygon-transaction
{
  "settings": {
    "index": {
      "sort.field": "seqNo", 
      "sort.order": "desc"
    }
  },
  "mappings": {
    "properties": {
      "seqNo": {
        "type": "long"
      }
    }
  }
}
```

복수 필드 index sorting

```json
PUT polygon-transaction
{
  "settings": {
    "index": {
      "sort.field": ["timestamp", "index"], 
      "sort.order": "desc"
    }
  },
  "mappings": {
    "properties": {
      "timestamp": {
        "type": "date"
      },
      "index": {
        "type": "int"
      }
    }
  }
}
```

# Elasticsearch 구성

## 하드웨어 구성

- 64core / 128G

![Untitled](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/07-31/es-01.png)

## Shard, Replica 구성

- Shard: 8
- Replica: 1

# 전 vs 후 성능 비교

## 테스트 방법

- nGrinder 를 사용하여 [VUser=1000, VUser=6000, Vuser=10000] 으로 3번 진행하였습니다.
- Index document 의 수는 약 **4억건** 기준입니다.

![Untitled](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/07-31/es-02.png)

- size=[25, 50, 100], page=[1~10000] 랜덤하게 요청하였습니다.
- 요청쿼리 예시
    
    ```json
    // index sorting 이 적용되지 않은 필드 쿼리
    GET polygon-transaction/_search
    {
      "sort": [
        {
          "unixTimeStamp": {
            "order": "desc"
          }
        }
      ],
      "size": 25,
      "search_after": [1626866517]
    }
    
    // index sorting 이 적용된 필드 쿼리
    GET polygon-transaction/_search
    {
      "sort": [
        {
          "seqNo": {
            "order": "desc"
          }
        }
      ],
      "size": 25,
      "search_after": [493466043]
    }
    ```
    

## [전] Index Sorting 이 적용되지 않은 필드 성능 테스트

- VUser: 1000명
- **TPS: 351**

![Untitled](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/07-31/es-03.png)

- VUser: 6000명
- **TPS: 357**

![Untitled](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/07-31/es-04.png)

- VUser: 10000명
- **TPS: 360**

![Untitled](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/07-31/es-05.png)

## [후] Index Sorting 이 적용된 필드 성능 테스트

- VUser: 1000명
- **TPS: 741**

![Untitled](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/07-31/es-06.png)

- VUser: 6000명
- **TPS: 744**

![Untitled](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/07-31/es-07.png)

- VUser: 10000명
- **TPS: 772**

![Untitled](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/07-31/es-08.png)

## 테스트 결과

![Untitled](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/07-31/es-09.png)

**356 TPS → 745 TPS** (**109% 상승!**) 

![Untitled](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/07-31/es-10.png)

# 결론

목표로 한 TPS:500 은 달성 하였지만 그래프가 생각보다 예쁘지는 않았습니다. 그럼에도 Elasticsearch Index Sorting은 방대한 양의 데이터를 처리하는 경우 성능 향상에 큰 도움을 주는 기술인 것은 명확한것 같습니다. 특히 **정렬된 데이터를 자주 조회해야 하는 상황**에서는 Index Sorting을 적용하여 응답 시간을 최적화할 수 있습니다.

하지만 Index Sorting을 적용하는 것에는 몇 가지 단점도 고려해야 할 것 같습니다.

1. **Index 크기 증가**: Index Sorting을 사용하면 특정 필드를 기준으로 정렬하여 데이터를 저장하므로 인덱스 크기가 증가할 수 있습니다. 따라서 디스크 공간을 더 많이 사용하게 되며, 이는 저장 용량에 대한 추가적인 고려가 필요합니다.
2. **색인 속도 저하**: Index Sorting은 색인 작업을 수행할 때 데이터를 특정 필드 기준으로 정렬하여 저장하는데, 이로 인해 색인 속도가 느려질 수 있습니다. 특히 대량의 데이터를 색인하는 경우 Index Sorting이 적용되는 시간이 늘어나게 됩니다.
3. **정렬 필드 변경 어려움**: Index Sorting을 적용하면 데이터를 특정 필드 기준으로 정렬하여 저장하게 되므로, 정렬에 사용되는 필드를 변경하려면 새로운 인덱스를 생성하고 데이터를 다시 색인해야 합니다. 이는 데이터의 변경이 빈번하게 발생하는 경우에 유의해야 할 점입니다.

따라서 Index Sorting을 적용하기 전에 인덱스의 크기, 데이터의 쿼리 패턴, 색인 작업 빈도 등을 고려하여 장단점을 비교하여야 합니다. 특히 데이터의 쿼리 패턴이 자주 변경되거나 색인 작업이 빈번하게 발생하는 경우 Index Sorting의 적용 여부를 신중하게 결정해야 할 것 같습니다.