---
title:  "ABI 기반 데이터 인덱싱"
categories:
  - backend
tags:
  - ABI
  - Indexing
  - Blockchain
toc: true
toc_icon: desktop
toc_label: "목차"
toc_sticky: true
---
<div style="text-align: right;"><span style="font-size: 13px;">Token Engineering 이득윤</span></div>


## 들어가며

이더리움 네트워크에서 ABI는 컨트랙트에서 함수를 호출할 때, 어떤 함수를 호출할지를 지정하거나 우리가 생각한 output을 보장하기 위해서 필요합니다.

이번 글에서는 먼저 ABI에 대해서 간단하게 알아본 후에, 제가 참여했던 ABI 기반 데이터 추출 프로젝트에 대해서 다뤄보려고 합니다.

## ABI란?

ABI란 Application Binary Interface의 약어로써, contract의 인터페이스를 나타내는 JSON 데이터입니다.

ABI에 대해서 이해하기 위해서는 선행적으로 solidity 컴파일 및 배포 과정에 대한 이해가 필요합니다.
스마트 컨트랙트를 작성하면, 이더리움 네트워크상에 배포하기 전에 바이트코드로 컴파일이 필요합니다.

이 컴파일하는 과정은, human-readable한 solidity 코드를 low-level한 EVM 바이트코드로 치환하는 과정인데요, 이 과정을 거쳐야 이더리움 네트워크상에서 실행할 수 있게 됩니다.

위에서 설명한 컴파일 과정을 거치면 ABI도 함께 생성됩니다.

ABI는 function과 event에 정보 그리고 각 인풋 파라미터, 아웃풋 파라미터에 대한 정보를 포함합니다.

![Untitled](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/07-31/abi-00.png)

***그림 1) Smart Contract 컴파일 및 배포 과정***



### ABI의 활용 1

아래와 같이 솔리디티로 구현한 bar라는 function이 있다고 했을 때,

```jsx
pragma solidity ^0.4.16;

contract Foo {
function baz(uint32 x, bool y) public pure returns (bool r) { r = x > 32 || y; }
}
```

_출처: [https://solidity-kr.readthedocs.io/ko/latest/abi-spec.html](https://solidity-kr.readthedocs.io/ko/latest/abi-spec.html)_

function baz는 아래와 같은 ABI로 변환될 것입니다.

```jsx
{
    "constant": true,
    "inputs": [
      {
        "name": "x",
        "type": "uint32"
      },
      {
        "name": "y",
        "type": "bool"
      }
    ],
    "name": "baz",
    "outputs": [
      {
        "name": "r",
        "type": "bool"
      }
    ],
    "payable": false,
    "stateMutability": "nonpayable",
    "type": "function"
  }
```

우리가 baz라는 function을 **102**와 **true**라는 파라미터로 call한다고 가정하겠습니다.

우리가 call한 파라미터와 function은 아래와 같은 변환과정을 거치게 됩니다.

1.  **0xcdcd77c0**

    - 메소드 ID로서, input과 function의 이름을 조합하여 keccack함수로 hash 한 후 첫 4byte를 추출한 결과입니다.
    - **baz(uint32,bool)**
      → **0xcdcd77c0**992ec5bbfc459984220f8c45084cc24d9b6efed1fae540db8de801d2
      → **0xcdcd77c0**

1.  **0x0000000000000000000000000000000000000000000000000000000000000066**

    - 102라는 숫자를 16진수로 변환한 후 32byte까지 padding한 결과물입니다.

1.  **0x0000000000000000000000000000000000000000000000000000000000000001**
    - true를 나타내는 1을 16진수로 변환한 후 32byte까지 padding한 결과물입니다.

input 데이터는 위의 3가지 결과물을 조합해

```jsx
0xcdcd77c000000000000000000000000000000000000000000000000000000000000000660000000000000000000000000000000000000000000000000000000000000001
```

와 같이 EVM에서 이해할 수 있는 byte로 변환됩니다.

위의 input에 따라서, 연산이 진행되고, 아래와 같은 결과물이 return 됩니다.

```jsx
0x0000000000000000000000000000000000000000000000000000000000000000
```

ABI를 활용하면 return된 결과물이 false임을 알 수 있습니다.

위의 과정처럼 사람은 **102**와 **true**라는 parameter를 기입해서 function call을 했지만, ABI를 통해서 EVM이 이해할 수 있는 byte로 변환되었습니다.

위와 같이 human-readable한 input을 EVM이 이해하는 byte로 변환하는 과정을 **encoding**,

byte를 해독하여 인간이 이해할 수 있는 결과물로 변환되는 과정을 **decoding**이라고 하며,

ABI는 이더리움 생태계에서 컨트랙트들과 상호작용을 할 때, encoding과 decoding에서 사용되는 스키마의 역할을 합니다.

![Untitled](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/07-31/abi-01.png)

***그림 2) ABI의 역할***



### ABI의 활용 2

#### 원시데이터

블록체인에서는 block, log, transaction 등의 데이터가 발생합니다.

![Untitled](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/07-31/abi-02.png)

**그림 3) 원시데이터(Block Data) 예시**



쟁글에서는 위와 같이 여러 블록체인에서 발생하는 데이터를 실시간으로 RDB에 인덱싱하고 있습니다.

블록체인상에서 가공되지 않은 형태의 데이터를 **원시데이터**라고 부르겠습니다.

#### ABI기반 원시데이터 가공

ABI를 활용하면 EVM 상에서 실행된 수많은 event, function의 결과물들을 decoding 할 수 있게 됩니다.

이는 각 contract 별로 decoding 된 데이터를 인덱싱할 수 있다는 말이기도 합니다.

ABI의 활용1 예제를 활용해서 어떻게 ABI를 통해서 function에서 발생한 데이터를 인덱싱할 수 있는지 설명하겠습니다.

- 먼저 ABI 정보를 기반으로 테이블과 인덱스를 자동 생성합니다.
- 쟁글에서는 원시데이터를 RDB의 형태로 인덱싱하고 있으니, transaction에서 to address가 해당 컨트랙트의 주소이고, transaction data의 첫 4byte가 **0xcdcd77c0**인 데이터를 조회하면 해당 TX에서 발생한 데이터를 확인할 수 있습니다.
- 그 데이터를 ABI와 매칭시키면서 parsing 하면, 아래와 같이 ABI 기반으로 인덱싱을 할 수 있게 됩니다.

![Untitled](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/07-31/abi-03.png)

**그림 4) blur exchange call function ABI 기반 데이터 인덱싱**



call function 데이터만 인덱싱을 할 수 있는 건 아닙니다.

event에서 발생한 데이터도 인덱싱할 수 있는데요,

event에서 발생한 데이터는 쟁글이 인덱싱하고 있는 log 원시 테이블에서 확인이 가능합니다.

call function의 예시에서는 signature를 만든 후에 첫 4byte를 signature로 사용했지만,

event에서는 똑같이 signature를 만든 후, 전체를 signature로 사용합니다.

아래는 ERC20의 컨트랙트가 필수로 구현해야 하는 Transfer 이벤트의 ABI 정보입니다.

```jsx
{
    "anonymous": false,
    "inputs": [
      {
        "indexed": true,
        "name": "from",
        "type": "address"
      },
      {
        "indexed": true,
        "name": "to",
        "type": "address"
      },
      {
        "indexed": false,
        "name": "value",
        "type": "uint256"
      }
    ],
    "name": "Transfer",
    "type": "event"
  }
```

call function과 같이 text siganautre를 만들면 Transfer(address,address,uint256)이 됩니다.

text signature를 keccak256 함수를 통해 hash 하면,

**0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef** 라는 signature가 만들어지게 됩니다.

ABI를 통해서 Event에서 발생한 데이터를 인덱싱하는 과정은 다음과 같습니다.

- ABI 정보를 기반으로 테이블과 인덱스를 자동 생성합니다.
- log 테이블에서 contract_address가 해당 컨트랙트의 주소이고, topic0가 signature (**0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef**)인 topic1, topic2, topic3 및 데이터를 조회합니다.
- topic과 data를 ABI와 매칭시키면서 parsing 하면, 아래와 같이 ABI 기반으로 인덱싱을 할 수 있게 됩니다.

![Untitled](https://s3.ap-northeast-2.amazonaws.com/upload.techblog.xangle.io/2023/07-31/abi-04.png)

**그림 5) Transfer event ABI 기반 데이터 인덱싱**



## 맺으며

본 포스트에서는 ABI에 대한 개념과 ABI의 역할, 그리고 쟁글에서 활용 중인 ABI 기반 데이터 인덱싱에 대해서 알아보았습니다.

블록체인 상의 데이터는 위와 같이 무궁무진한 방식으로 활용될 수 있습니다.

블록체인에 대해서 관심이 많고 본인이 백엔드 개발자로서 대용량 데이터를 다루고 싶다면, 쟁글이 좋은 선택지가 되리라 확신합니다.
