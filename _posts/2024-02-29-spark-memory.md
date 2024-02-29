---
title:  "스파크의 메모리 구조 이론"
categories:
  - Backend
tags:
  - Spark
  - Hadoop
  - Java
toc: true
toc_icon: desktop
toc_label: "목차"
toc_sticky: true
---
<div style="text-align: right;"><span style="font-size: 13px;">Data Analytics Platform 김준영</span></div>


최근 사내 기술 세미나에서 발표된 세션을 공유하고자 합니다. 우리 팀은 대규모 블록체인 데이터셋을 처리하기 위해 Hadoop과 Spark를 활용하고 있습니다. 

이를 통해 각 스쿼드 팀에 필요한 블록체인 데이터를 효과적으로 전달하고 있습니다. 특히, 발표된 세션에서는 온체인 데이터와 같은 비정형 빅 데이터를 Spark를 이용하여 효과적으로 분석하고 쿼리하는 방법에 대해 다뤘습니다. 해당 세션의 내용을 통해 우리의 데이터 애널리틱스 플랫폼 팀이 블록체인 데이터 처리에 어떻게 접근하고 있는지에 대해 자세히 이해할 수 있습니다.

# **Spark: 대규모 데이터 처리를 위한 통합 분석 엔진**

Spark는 대규모 데이터 처리를 위한 통합 분석 엔진으로, 인메모리 병렬 분산처리 엔진으로 알려져 있습니다. 이전에는 Hadoop의 MapReduce에서 디스크 IO로 데이터를 처리했지만, 이는 낮은 처리율과 성능 문제를 야기했습니다. 이에 대한 대안으로 인 메모리 기반의 Spark가 탄생하였으며, Hadoop 생태계에서 큰 인기를 얻고 있습니다.

## **Spark의 핵심 용어**

### **Executor**

Executor는 작업을 처리하는 워커를 의미합니다. 각 Executor는 CPU 코어와 메모리를 할당받아 작업을 수행합니다.

### **Core**

Core는 CPU의 코어를 의미합니다. Spark에서는 하나의 Core가 하나의 Partition을 처리하는 단위로 사용됩니다.

### **Memory**

Memory는 메모리를 의미합니다. Spark에서는 Executor가 할당받는 메모리를 설정할 수 있습니다.

### **Partition**

Partition은 RDD나 Dataset을 구성하는 최소 단위 객체를 의미합니다. 각 Partition은 하나의 Task를 수행하며, 데이터 처리에 사용됩니다. Input Partition은 파일을 읽을 때 생성되고, Output Partition은 파일을 저장할 때 생성됩니다. Shuffle Partition은 기본적으로 200개가 생성됩니다.

## **스파크 구조**

![Untitled](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/24-02-29/Untitled+(9).png)

스파크 어플리케이션은 마스터-슬레이브 구조로 실행됩니다. 자원 관리는 스파크 컨텍스트가 담당하고, 작업은 Executor가 수행합니다. SparkContext는 Job을 submit하고 task로 변환하며, Executor는 계산 작업을 수행하고 결과를 반환합니다.

## **Spark 메모리 관리**

![Untitled](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/24-02-29/Untitled+(10).png)

### **기본 개념**

Executor는 워커 노드에서 실행되는 JVM 프로세스입니다. JVM 메모리 관리는 On-Heap Store와 Off-Heap Store로 나뉩니다.

- On-Heap (In-memory): JVM 힙에 할당되는 메모리로, GC에 의해 관리됩니다.
- Off-Heap (External-memory): JVM 외부에 직렬화된 객체가 할당되는 메모리로, GC에 바인딩되지 않습니다.

### **통합 메모리 관리**

Spark 1.6부터는 통합 메모리 관리자가 도입되어 Dynamic 메모리를 할당합니다. Storage와 Execution이 공유하는 통합 메모리 컨테이너에서 메모리가 할당되며, 사용되지 않는 Execution 메모리는 Storage 메모리로 사용될 수 있습니다.

### **Reserved 메모리 및 User Memory**

```bash
spark-shell --executor-memory 300m
```

Reserved 메모리는 시스템 용으로 예약되며, User Memory는 사용자 정의 데이터 구조와 RDD 작업에 필요한 데이터를 저장하는데 사용됩니다.

# Spark 동작 (Lazy evaluation)

![Untitled](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/24-02-29/Untitled+(11).png)

### **RDD (Resilient Distributed Dataset)**

RDD는 스파크가 사용하는 핵심 데이터 모델로, 분산 저장된 요소들의 집합을 의미합니다. 이는 분산 데이터 셋으로, 병렬 처리가 가능하며 장애가 발생할 경우 스스로 복구할 수 있는 내성을 가지고 있습니다.

### **Transformation(변환)**

Transformation은 기존 RDD에서 새로운 RDD를 생성하는 함수입니다. 주로 사용되는 몇 가지Transformation 함수는 다음과 같습니다.

```bash
distinct(), filter(), map(), sortBy(), join(), union()
```

- Narrow Transformation과 Wide Transformation 사이에는 중요한 차이가 있습니다. Narrow Transformation은 간단한 작업으로, 데이터를 worker node와 교환하지 않습니다. 반면 Wide Transformation은 worker node 간에 데이터 교환, 즉 **데이터 셔플링**이 이루어지는 작업입니다.
- 데이터 셔플링은 처리 비용이 많이 드는 작업으로, DB의 Sort Merge Join과 비슷한 방식으로 동작합니다.

### **Action(동작)**

Action은 작업을 수행하는 함수입니다. 몇 가지 주요한 Action 함수는 다음과 같습니다.

```bash
count(), collect(), max(), min()

```

![어떻게 보면 java ForkJoinPool과 형태가 유사하다.](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/24-02-29/%E1%84%8C%E1%85%A6%E1%84%86%E1%85%A9%E1%86%A8+%E1%84%8B%E1%85%A5%E1%86%B9%E1%84%8B%E1%85%B3%E1%86%B7-2024-01-12-1335+(2).png)

어떻게 보면 java ForkJoinPool과 형태가 유사하다.

위 그림에서 보듯이, 스파크 작업은 RDD를 활용하여 Transformation을 거쳐 가공한 후, 실제 작업은 Action을 통해 수행됩니다. RDD는 스스로 복구할 수 있는 내성을 갖추고 있으며, 이는 상위 유형에 대한 메타 데이터를 보유함으로써 가능합니다.

Action이 이루어지기 전까지의 작업은 논리적 쿼리 계획만 만들고 실제로 메모리에 올리지 않습니다. Action이 이루어지면 DAG(Directed Acyclic Graph)를 통해 작업들이 수행됩니다. 아래 예시를 살펴보겠습니다.

![DAG의 일부분을 캡처](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/24-02-29/Untitled+(12).png)

DAG의 일부분을 캡처

위 그림은 DAG의 일부분을 캡처한 것입니다. 아래에서 실제로 Query plan을 보면 어떤 작업을 하는지 유추할 수 있습니다.

![하둡에 transfer 데이터를 저장하는 예시](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/24-02-29/Untitled+(13).png)

하둡에 transfer 데이터를 저장하는 예시

위 그림은 하둡에 데이터를 저장하는 예시입니다. 데이터 2개를 filter(transformation)하여 RDD를 생성하고, 그 후 Join을 수행합니다. 이때 사용된 조인 전략은 SortMergeJoin이며, 데이터를 정렬하는 과정이 필요하여 Shuffle이 발생합니다. Shuffle은 worker node에 있는 데이터를 섞는 Wide Transformation입니다. 이 과정이 완료된 후에는 데이터가 하둡에 저장됩니다.

### Spark Fault Tolerance: Self Recovery

RDD에 대한 내결함성을 충족하기 위해 여러 노드간에 전체 데이터가 복제된다.

# Spark Memory Leak Issue

스파크에서는 메모리 관리가 중요합니다. Spark는 JVM으로 구동되므로 메모리 관리를 알아보기 위해 JVM에 대한 이해도가 필요합니다.

![[IBM 문서](https://developer.ibm.com/articles/j-codetoheap/) 참고 자료 ](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/24-02-29/Untitled+(14).png)

[IBM 문서](https://developer.ibm.com/articles/j-codetoheap/) 참고 자료 

[IBM 문서](https://developer.ibm.com/articles/j-codetoheap/)를 참고해보면, 일반적으로 어플리케이션에서 Java의 heap size는 RAM의 25%를 차지합니다.

Java application을 실행하였을 때, 메모리 공간은 2가지 영역으로 나뉩니다.

- Native Heap, Off heap (JVM이 이 영역의 일부를 사용)
- Java Heap

Spark에서 사용하는 메모리가 Java heap을 넘어가게 되면 Heap Space Error가 발생할 것입니다.

일반적인 어플리케이션에서 heap space error가 나오기 쉽지 않습니다. 대용량 데이터를 처리하는 서버의 경우를 예시로 들자면 List<Long> 안에 많은 데이터가 담긴 경우 Long은 8바이트이므로 구동 머신(RAM)에 따라 max값은 계산할 수 있습니다. 또 다른 예시로 Garbage Collection이 제대로 동작하지 않는 경우 On-heap 메모리 부족이 발생하고 Full GC → Stop the world 가 발생할 것입니다.

Spark도 마찬가지로 Garbage Collector가 작동합니다. (Java 8 이므로 parallel GC)

![Untitled](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/24-02-29/Untitled+(16).png)

위의 그림에서 보듯이, RDD_A를 필터링하여 RDD_B를 생성하면 RDD_B는 더 이상 참조되지 않게 되어, RDD_A에 대한 Minor GC가 발생할 것입니다. 그림에서 Task들의 GC time이 2초인데, Task의 증가는 GC time의 속도 저하 요인 중 하나일 수 있습니다.

[IBM 문서](https://developer.ibm.com/articles/j-codetoheap/)에 따르면, int, Interger 등 객체 선언에 따른 메모리 오버헤드가 존재합니다. Spark 개발진들은 이 오버헤드를 없애기 위해 Binary Data로 직접 연산할 수 있는 메모리 관리자인 Unified Memory Manager를 도입했습니다.

### Data Shuffle

파티션은 RDD를 구성하고 있는 최소 단위 객체입니다. 각 파티션은 서로 다른 노드에서 분산 처리됩니다.

위에서 언급한대로 파티션의 개념은 core = 파티션 = 태스크와 같습니다.

파티션의 수를 늘리는 것은 태스크 당 필요한 메모리를 줄이고 병렬화의 정도를 늘리는 데 도움이 됩니다.

![Untitled](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/24-02-29/Untitled+(15).png)

위의 그림에서 볼 수 있듯이, 작업을 제출하면 설정된 파티션은 149개이고, 태스크는 총 149개가 예약됩니다. Executor는 128개가 담당하므로 Java 8 기준에서 관리되는 Fork Join Pool의 태스크 수는 128개로 제한됩니다.

아래 예시를 통해 파티션 수를 조절하여 메모리를 관리하는 방법을 알 수 있습니다.

```bash
spark-submit --master yarn --deploy-mode cluster --executor-cores 4 --num-executors 64 --executor-memory 16g --driver-memory 1g <>.jar <여기서부터 소스코드 인자값>
```

여기서 Executor의 총 개수는 256개이며, 각 Executor는 (executor-memory / executor-core)의 메모리를 할당받습니다. 따라서 전체 시스템에서 총 예약된 메모리는 64 * 16GB = 1024GB 즉 1TB가 됩니다.

만약 처리해야 할 데이터가 10억개로 추정되고, 파티션의 수는 80개로 설정된 경우 각 파티션은 처리할 데이터의 부분집합을 나타냅니다.

실제로 필요한 Executor의 개수는 80개뿐이지만, 전체 시스템에서 예약된 메모리는 1024GB로 상당히 큽니다. 이는 효율적인 자원 활용을 위해 고려되어야 하는 중요한 측면입니다.

또한, 데이터를 정렬하는 셔플(shuffle) 연산이 수행될 경우 메모리 사용량이 증가합니다. 이로 인해 1TB의 데이터에 대한 연산을 모두 메모리에서 처리하지 못하면 디스크에 Spill이 발생하게 되고, 이러한 상황이 지속되면 Heap Space Error가 발생할 수 있습니다.

따라서 core와 메모리 예약뿐만 아니라 작업의 파티션 수를 조절하는 것도 중요합니다.

![Untitled](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/24-02-29/Untitled+(17).png)

이에 대한 대안으로, 파티션의 수를 임의로 3000개로 조정하면 각 Executor 당 필요한 데이터 양이 감소하게 됩니다. 이를 통해 메모리 사용을 효율적으로 조절할 수 있습니다. 파티션 수를 늘리면 Executor 당 필요한 데이터 양이 감소하고, 따라서 더 효율적인 메모리 사용이 가능해집니다.

# **마무리**

이상으로 스파크의 핵심 개념과 메모리 관리, 데이터 셔플 등에 대해 살펴보았습니다. 스파크는 대규모 데이터 처리를 위한 강력한 도구로, 적절한 활용을 통해 데이터 분석 및 처리 작업을 효율적으로 수행할 수 있습니다.
