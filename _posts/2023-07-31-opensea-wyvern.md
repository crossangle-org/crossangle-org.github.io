---
title:  "OpenSea 거래 데이터 분석(Wyvern Protocol)"
categories:
  - backend
tags:
  - Blockchain
  - NFT
  - Data
  - OpenSea
toc: true
toc_icon: desktop
toc_label: "목차"
toc_sticky: true
---
<div style="text-align: right;"><span style="font-size: 13px;">Token Engineering 서현준</span></div>

# 들어가며

---

쟁글에서는 블록체인 데이터를 분석하여 블록체인 관련 종사자 및 블록체인에 관심이 있는 사람들에게 의미 있는 데이터를 제공하기 위해 블록체인의 다양한 데이터를 분석하고 있습니다.

이번 글에서는 NFT 마켓 플레이스 중 세계 최대 거래소인 OpenSea의 거래 데이터를 분석하며, 각 메인넷별 NFT 거래 데이터 지표를 만들었던 경험들 중에서 OpenSea 초기 거래 프로토콜인 Wyvern Protocol의 거래 데이터를 분석하는 방법을 소개하겠습니다.

이번 글에서는 데이터 분석을 위한 기본 지식 설명과 Wyvern 프로토콜을 통한 거래 데이터 분석에 대해 설명하고자 합니다.

들어가기에 앞서, OpenSea가 현재 지원하는 메인넷과 거래 프로토콜의 종류를 살펴볼 예정입니다.

본 글에서 사용된 OpenSea 데이터 분석에 활용된 데이터는 Ethereum 기준으로 작성되었습니다.

# OpenSea 마켓 플레이스에서 지원하는 메인넷

---

OpenSea는 초기 Ethereum 거래를 지원했으며, 다른 메인넷도 비슷한 시기에 추가되거나 시간이 지남에 따라 하나씩 메인넷 지원이 추가되었습니다. 현재는 Arbitrum, Avalanche, BNB Chain, Ethereum, Klaytn, Optimism, Polygon, Solana, Zora 등의 메인넷에서 초기에는 Wyvern Protocol, 현재는 Seaport Protocol을 통해 거래가 이루어지고 있습니다.

# OpenSea 마켓 플레이스의 거래 프로토콜

---

OpenSea는 2018년 6월 12일(UTC+0) OpenSea Deployer를 통해 Wyvern Protocol을 출시하였으며, 2018년 6월 13일(UTC+0) 첫 거래가 시작되었습니다. 이후, 2022년 2월 18일(UTC+0) EIP712 서명을 통한 거래 데이터 투명성 확보 및 취약점 극복을 위해 새로운 버전의 Wyvern Protocol이 업데이트되었습니다.

그 후, NFT 거래가 활발해지면서 수수료 문제가 논란이 되었고, OpenSea는 가스비 최적화 작업을 통해 새로운 거래 프로토콜을 발표하게 되었습니다. Seaport Protocol은 바이트 코드 레벨에서의 가스비 최적화 처리로 가스비 절감을 통해 최적화가 이루어졌으며, 2022년 6월 11일(UTC+0) OpenSea Deployer를 통해 새로운 거래 프로토콜로 발표되었습니다. OpenSea의 새 거래 프로토콜인 Seaport는 2022년 6월 12일(UTC+0) 첫 거래가 시작되었습니다.

이후 다양한 기능 업데이트를 통해 현재(2023년 7월 31일(UTC+0)) 기준으로 Seaport 1.5 버전까지 출시되었습니다. 현재는 Seaport Protocol을 계속 향상시키면서 Seaport Protocol을 통해 거래가 이루어지고 있습니다.

**표 1. OpenSea 프로토콜(Contract)별 거래 일자**

| Name | Address | Created by Tx Hash | Created Date(UTC+0) | Start Trade Date(UTC+0) | End Trade Date(UTC+0) |
| --- | --- | --- | --- | --- | --- |
| OpenSea: Wyvern Exchange (v1) | 0x7be8076f4ea4a4ad08075c2508e481d6c946d12b | 0xebe6bd9d1da992b9a6f816564eafef42e9bf68d3876fae22556cc63da6fc0f66 | 2018-06-12 | 2018-06-13 | 2022-02-25 |
| OpenSea: Wyvern Exchange2.3 (v2) | 0x7f268357a8c2552623316e2562d90e642bb538e5 | 0xb387cc66173783ef9faef775d4b7eaaff3fdd47e765239d5ffb7633ec0be665b | 2022-02-01 | 2022-02-18 | 2022-08-01 |
| OpenSea: seaport1.1 (v3) | 0x00000000006c3852cbef3e08e8df289169ede581 | 0x34a28a0111a238347fe30a6c91b00437f8438357427c5e759a2f7577afe65144 | 2022-06-11 | 2022-06-12 | continue |
| OpenSea: seaport1.2 | 0x00000000000006c7676171937c444f6bde3d6282 | 0xadc7728fd231c4d6118ca2409d4fceef5be4028ef9049b0dffc2ae9c326e9f00 | 2023-01-31 | 2023-02-09 | 2023-02-21 |
| OpenSea: seaport1.3 | 0x0000000000000ad24e80fd803c6ac37206a45f15 | 0xd3d4aec88ee9feedd880eca79642b0a1ecf1a678f1e89b07c760c1a667b7206c | 2023-02-16 | 2023-02-16 | 2023-02-16 |
| OpenSea: seaport1.4 | 0x00000000000001ad428e4906ae43d8f9852d0dd6 | 0x4f5eae3d221fe4a572d722a57c2fbfd252139e7580b7959d93eb2a8b05b666f6 | 2023-02-18 | 2023-02-21 | continue |
| OpenSea: seaport1.5 (v4) | 0x00000000000000adc04c56bf30ac9d3c0aaf14dc | 0xa7f75a8b23c0f3150b7116aae930101f42996cb35e3e596a646156926350933a | 2023-04-26 | 2023-04-27 | continue |

# 사전 배경 지식

---

본격적으로 Wyvern Protocol의 거래 데이터 분석에 앞서, 알아야 할 기본 지식을 몇 가지 설명하고자 합니다.

## Smart Contract의 Function과 Event

---

블록체인 내에서 Ethereum과 같은 경우, Smart Contract라는 블록체인 내의 프로그램을 통해 특정 행위가 실행되고 기록됩니다. 이러한 행위에는 NFT 거래, 서명 등이 포함됩니다. Smart Contract는 블록체인 내에서 특정 주소에 상주하는 코드(기능) 및 데이터(상태)의 모음입니다. 이러한 Smart Contract는 크게 Function과 Event로 구성됩니다.

Function은 Smart Contract가 수행할 수 있는 특정 작업 또는 작업을 정의하는 코드입니다. 이는 기존 프로그래밍 언어의 메서드와 같다고 볼 수 있습니다. 입력을 받고, 데이터를 처리하고, 출력을 생성할 수 있습니다. 이러한 Smart Contract의 function은 블록체인 내에서 트랜잭션을 수행 시 Smart Contract의 특정 Function을 호출하게 되면 실행됩니다.

Event는 Smart Contract 내에서 발생하는 특정 사건의 발생 또는 변경 사항을 전달하고 기록하는 방법입니다.

요약하면 Smart Contract의 Function은 Smart Contract의 동작과 작업을 정의하고, Event는 Smart Contract 내 특정 사건의 발생에 대해 외부 관찰자에게 알리는 데 사용됩니다.

<aside>
💡 NFT 거래 데이터는 Function을 통해 분석할 수도 있지만, 본 글에서는 Event 정보를 통해 NFT 거래 데이터를 분석하는 방법을 설명합니다.

</aside>

## Event

---

Event 데이터를 활용하기 위해서는 크게 2가지를 알고 있어야 합니다.

첫째로, EVM 계열의 Event Log의 형태를 알고 있어야 합니다. Event Log의 데이터를 통해 OpenSea의 Smart Contract를 사용하여 이루어진 거래 데이터만을 도출할 수 있습니다.

둘째로, NFT 거래를 위해 알아야 하는 기본적인 것으로 거래 통화 수단이 되는 ERC20과 거래의 대상이 되는 NFT의 종류인 ERC721, ERC1155의 기본 Event 종류를 알고 있어야 합니다.

셋째로, Wyvern 프로토콜의 주요 Event 형태를 파악해야 합니다.

### Event Log

Transaction Receipt 데이터의 Logs 데이터의 형태를 보면 기본적으로 아래 표와 같은 데이터를 포함하고 있습니다. 각 데이터의 의미는 아래와 같습니다.

**표 2. ethereum logs의 필드 설명**

| 필드 | 설명 |
| --- | --- |
| contract address | wyvern, seaport 등의 protocol address, nft address, token address 등 |
| topic0 | Event Name의 hex signature |
| topic1 | indexed param |
| topic2 | indexed param |
| topic3 | indexed param |
| data | indexed가 아닌 param을 하나로 합친 data |
| index | 해당 event가 발생한 tx내에서 event의 index |
| block hash | 해당 event가 발생한 tx가 담긴 block의 hash |
| block number | 해당 event가 발생한 tx가 담긴 block의 number |
| block time | 해당 event가 발생한 tx가 담긴 block의 time |
| tx hash | 해당 event가 발생한 tx의 hash |
| tx index | 해당 event가 담긴 block내에서 tx의 index |
| tx from | 해당 event가 담긴 block내에서 tx의 from address |
| tx to | 해당 event가 담긴 block내에서 tx의 to address |

### ERC20

ERC20의 Smart Contract를 살펴보면 Transfer와 Approval Event로 구성되어 있습니다.

**Transfer**

특징: `indexed` 필드는 `from`, `to`

```solidity
/**
	MUST trigger when tokens are transferred, including zero value transfers.

	A token contract which creates new tokens SHOULD trigger a Transfer event with the _from address set to 0x0 when tokens are created.
*/
event Transfer(
  address indexed from,
  address indexed to,
  uint256 value
);
```

**Approval**

특징: `indexed` 필드는 `owner`, `spender`

```solidity
/**
	MUST trigger on any successful call to approve(address _spender, uint256 _value).
*/
event Approval(
  address indexed owner,
  address indexed spender,
  uint256 value
);
```

### ERC721

ERC721의 Smart Contract를 살펴보면 Transfer와 Approval, ApprovalForAll Event로 구성되어 있습니다.

**Transfer**

특징: `indexed` 필드는 `_from`, `_to`, `_tokenId`

```solidity
/// @dev This emits when ownership of any NFT changes by any mechanism.
///  This event emits when NFTs are created (`from` == 0) and destroyed
///  (`to` == 0). Exception: during contract creation, any number of NFTs
///  may be created and assigned without emitting Transfer. At the time of
///  any transfer, the approved address for that NFT (if any) is reset to none.
event Transfer(
  address indexed _from,
  address indexed _to,
  uint256 indexed _tokenId
);
```

**Approval**

특징: `indexed` 필드는 `_owner`, `_approved`, `_tokenId`

```solidity
/// @dev This emits when the approved address for an NFT is changed or
///  reaffirmed. The zero address indicates there is no approved address.
///  When a Transfer event emits, this also indicates that the approved
///  address for that NFT (if any) is reset to none.
event Approval(
  address indexed _owner,
  address indexed _approved,
  uint256 indexed _tokenId
);
```

특징: `indexed` 필드는 `_owner`, `_operator`

**ApprovalForAll**

```solidity
/// @dev This emits when an operator is enabled or disabled for an owner.
///  The operator can manage all NFTs of the owner.
event ApprovalForAll(
	address indexed _owner,
	address indexed _operator,
	bool _approved
);
```

### ERC1155

ERC1155의 Smart Contract를 살펴보면 TransferSingle과 TransferBatch, ApprovalForAll, URI Event로 구성되어 있습니다.

**TransferSingle**

특징: `indexed` 필드는 `_operator`, `_from`, `_to`

```solidity
/**
	@dev Either `TransferSingle` or `TransferBatch` MUST emit when tokens are transferred, including zero value transfers as well as minting or burning (see "Safe Transfer Rules" section of the standard).
	The `_operator` argument MUST be the address of an account/contract that is approved to make the transfer (SHOULD be msg.sender).
	The `_from` argument MUST be the address of the holder whose balance is decreased.
	The `_to` argument MUST be the address of the recipient whose balance is increased.
	The `_id` argument MUST be the token type being transferred.
	The `_value` argument MUST be the number of tokens the holder balance is decreased by and match what the recipient balance is increased by.
	When minting/creating tokens, the `_from` argument MUST be set to `0x0` (i.e. zero address).
	When burning/destroying tokens, the `_to` argument MUST be set to `0x0` (i.e. zero address).        
*/
event TransferSingle(
	address indexed _operator,
	address indexed _from,
	address indexed _to,
	uint256 _id,
	uint256 _value
);
```

**TransferBatch**

특징: `indexed` 필드는 `_operator`, `_from`, `_to`

```solidity
/**
	@dev Either `TransferSingle` or `TransferBatch` MUST emit when tokens are transferred, including zero value transfers as well as minting or burning (see "Safe Transfer Rules" section of the standard).      
	The `_operator` argument MUST be the address of an account/contract that is approved to make the transfer (SHOULD be msg.sender).
	The `_from` argument MUST be the address of the holder whose balance is decreased.
	The `_to` argument MUST be the address of the recipient whose balance is increased.
	The `_ids` argument MUST be the list of tokens being transferred.
	The `_values` argument MUST be the list of number of tokens (matching the list and order of tokens specified in _ids) the holder balance is decreased by and match what the recipient balance is increased by.
	When minting/creating tokens, the `_from` argument MUST be set to `0x0` (i.e. zero address).
	When burning/destroying tokens, the `_to` argument MUST be set to `0x0` (i.e. zero address).                
*/
event TransferBatch(
	address indexed _operator,
	address indexed _from,
	address indexed _to,
	uint256[] _ids,
	uint256[] _values
);
```

**ApprovalForAll**

특징: `indexed` 필드는 `_owner`, `_operator`

```solidity
/**
	@dev MUST emit when approval for a second party/operator address to manage all tokens for an owner address is enabled or disabled (absence of an event assumes disabled).        
*/
event ApprovalForAll(
	address indexed _owner,
	address indexed _operator,
	bool _approved
);
```

**URI**

특징: `indexed` 필드는 `_id`

```solidity
/**
	@dev MUST emit when the URI is updated for a token ID.
	URIs are defined in RFC 3986.
	The URI MUST point to a JSON file that conforms to the "ERC-1155 Metadata URI JSON Schema".
*/
event URI(
	string _value,
	uint256 indexed _id
);
```

### Wyvern Protocol

Wyvern Protocol의 Smart Contract를 살펴보면 많은 이벤트로 구성되어 있습니다. 그 중 거래 내역을 볼 수 있는 OrdersMatched에 집중하겠습니다.

**OrdersMatched**

- OrdersMatched란 거래가 수행된 내역, 즉 영수증을 의미합니다. 이를 통해 거래에 대한 히스토리를 파악할 수 있습니다.
- 특징: `indexed` 필드는 `maker`, `taker`, `metadata`

```solidity
/* Log match event. */
emit OrdersMatched(
	buyHash,
	sellHash,
	sell.feeRecipient != address(0) ? sell.maker : buy.maker,
	sell.feeRecipient != address(0) ? buy.maker : sell.maker,
	price,
	metadata
);

event OrdersMatched(
	bytes32 buyHash,
	bytes32 sellHash,
	address indexed maker,
	address indexed taker,
	uint price,
	bytes32 indexed metadata
);
```

# 본격적인 Wyvern Protocol의 거래 데이터 수집, 분석, 적재 방법

---

본격적으로 Wyvern Protocol의 거래 데이터를 수집, 분석, 적재하는 과정을 살펴보겠습니다. NFT 거래 데이터를 수집하기 위해서는 사전에 block, transaction, log 데이터가 수집되어 있어야 합니다. 해당 데이터들과 아래와 같은 수집 과정들을 살펴보고자 한다면 Dune에서 실제 Ethereum 데이터를 통해 확인할 수 있습니다.

## Wyvern Protocol의 거래 데이터 수집

---

Wyvern Protocol의 거래 데이터를 수집하기 앞서, Ethereum의 Log 데이터가 갖춰져야 합니다. Log 데이터를 토대로 특정 주기(본 글에서는 일자)별로 Wyvern Protocol의 거래 데이터를 수집합니다.

1. 특정 기간 내에 Contract Address(이하 CA)가 Wyvern Protocol의 CA이면서, topic0이 OrdersMatched인(topic0은 event name의 hex값이므로) tx_hash를 group by 하여, distinct한 tx_hash를 추출합니다.
    - 기간은 시간 단위로 스케줄링 할 수도 있고, 하루 단위로 스케줄링 할 수 있습니다. Wyvern 버전에 따라 각 CA를 통해 거래된 기간 내에 조회를 해야 합니다.
    - CA는 Wyvern Protocol의 버전에 따라 거래된 기간과 매치하여 설정합니다.
    - topic0는 Wyvern Protocol에서는 OrdersMatched로 고정합니다. 단, hex signature이므로, 본 글의 하단에 **Event Name Hex Signature**를 참고하시길 바랍니다.
    
    ex) query
    
    ```sql
    SELECT tx_hash FROM ethereum.logs
    WHERE block_time >= '2022-03-01 00:00:00' AND block_time < '2022-03-01 24:00:00'
    		AND contract_address = '0x7f268357a8c2552623316e2562d90e642bb538e5'
    		AND topic1 = '0xc4109843e0b7d514e4c093114b863f8e7d8d9a458c372cd51bfe526b588006c9'
    GROUP BY tx_hash;
    ```
    
2. 1번에서 쿼리 결과로 나온 tx_hash의 값을 반복하며 해당 tx_hash에 포함된 Logs 데이터를 가져와 데이터 분석 및 적재 단계로 넘어갑니다.
    
    ex) query
    
    ```sql
    SELECT * FROM ethereum.logs WHERE block_time >= '2022-03-01 00:00:00' AND block_time < '2022-03-01 24:00:00' AND tx_hash = '0x3de1b78d00e5afcffc1999ff94f0bc8934324620d02fa15bac018c78648b7850' ORDER BY index;
    ```
    

## Wyvern Protocol의 거래 데이터 분석 및 적재

---

NFT 거래 데이터를 저장하기 위한 테이블을 모델링하고, 이 데이터를 채우기 위한 로직을 설계해야 합니다. NFT 거래 데이터를 저장하기 위한 테이블은 dune의 초기 nft_trades 테이블을 샘플로 사용하여 설명하겠습니다.

**표 3. nft_trades 테이블 sample**

| id | field | type | 설명 |
| --- | --- | --- | --- |
| 1 | block_time | timestamp | 해당 거래의 블록 생성 시간 |
| 2 | tx_hash | bytea | 해당 거래의 tx hash |
| 3 | block_number | int4 | 해당 거래의 block 번호 |
| 4 | evt_index | int4 | 해당 거래의 tx 내의 evt index |
| 5 | tx_from | bytea | 해당 거래의 tx의 from address |
| 6 | tx_to | bytea | 해당 거래의 tx의 to address |
| 7 | trace_address | _int4 | 해당 거래의 trace의 index |
| 8 | seller | bytea | nft item의 판매자 |
| 9 | buyer | bytea | nft item의 구매자 |
| 10 | nft_token_id | text | nft token의 id |
| 11 | erc_standard | text | ERC721, ERC1155 |
| 12 | nft_contract_address | bytea | nft token의 CA |
| 13 | nft_project_name | text | nft project의 이름 |
| 14 | number_of_items | int4 | 거래된 nft item 개수 |
| 15 | trade_type | text | Single Item Trade, Bundle Trade |
| 16 | nft_token_ids_array | _text | Bundle 거래인 경우 nft token의 id 리스트 |
| 17 | senders_array | _bytea | Bundle 거래인 경우 nft token의 sender 리스트 |
| 18 | recipients_array | _bytea | Bundle 거래인 경우 nft token의 receipient 리스트 |
| 19 | erc_types_array | _text | Bundle 거래인 경우 nft token의 erc types 리스트 |
| 20 | nft_contract_addresses_array | _bytea | Bundle 거래인 경우 nft token의 nft CA 리스트 |
| 21 | erc_values_array | _numeric | Bundle 거래인 경우 nft token의 erc value 리스트 |
| 22 | original_currency | text | nft item을 구매한 currency(ERC20) |
| 23 | original_currency_contract | bytea | nft item을 구매한 original currency(ERC20)의 CA |
| 24 | currency_contract | bytea | nft item을 구매한 currency(ERC20)의 CA |
| 25 | platform | text | OpenSea |
| 26 | evt_type | text | Trade |
| 27 | category | text | Buy, Sell |
| 28 | exchange_contract_address | bytea | OpenSea 거래 프로토콜의 CA(여기서는 Wyvern의 CA) |
| 29 | platform_version | text | 거래 protocol 버전 |
| 30 | original_amount_raw | numeric(54, 18) | nft item을 구매한 currency 토큰의 양 |
| 31 | original_amount | numeric(54, 18) | nft item을 구매한 currency 토큰을 해당 토큰의 decimals로 나눈 값 |
| 32 | usd_amount | numeric(54, 18)  | nft item의 usd 가격 |
| 33 | trade_id | int4 | 해당 거래의 trade id |

### 기본 메커니즘

---

NFT 데이터 분석 및 적재는 Log 데이터에서 topic0(Event Name)을 기반으로 합니다. Wyvern에서는 Transfer, TransferSingle, OrdersMatched Event를 확인해야 합니다. Wyvern CA에 대한 Log들을 분석하면 ERC721 거래에 대한 TransferBatch Event는 사용되지 않는 것을 볼 수 있습니다. 따라서 Transfer, TransferSingle, OrdersMatched Event를 기반으로 위 테이블의 데이터가 확보되면 하나의 거래 데이터를 생성하여 테이블에 적재합니다.

<aside>
💡 위의 거래 데이터 수집에서 가져온 tx hash의 Log 리스트를 Event Index 기준으로 오름차순으로 정렬하면 Approval - Transfer 또는 TransferSingle - OrdersMatched 순서대로 나온다는 것을 보장합니다.

</aside>

**외부 테이블 참조(blocks, transactions, logs)**

<aside>
<img src="https://www.notion.so/icons/sign-out_green.svg" alt="https://www.notion.so/icons/sign-out_green.svg" width="40px" /> 1. block_time
2. tx_hash
3. block_number
4. evt_index
- 1~4 data는 ethereum.logs 테이블에 있는 정보로 세팅

5. tx_from
6. tx_to
- 5~6 data는 ethereum.transactions 테이블에 있는 정보로 세팅

7. trace_address
- 7 data는 ethereum.traces 테이블에 있는 정보로 세팅

</aside>

**Transfer**

- Transfer 이벤트는 ERC20, ERC721 두 개에 모두 있기 때문에 구분할 수 있어야 합니다. Topic이 4개까지 있으면 ERC721, 3개면 ERC20으로 분류합니다.
- Topic1, Topic2, Topic3 다 비어있거나, 다 채워져 있으면 ERC721 Token으로 분류합니다.
    - ERC721 CA 같은 경우에는 공식적인 형식을 지키지 않고 event 구현 시 indexed를 빠뜨리고 구현한 케이스들이 존재합니다.
- 그렇지 않으면 ERC20 Token으로 분류합니다. ERC20이 보편적으로는 NFT item을 구매할 때 사용한 currency로 사용 되나, 예외 케이스로 ERC20 자체가 거래 되는 경우도 있습니다.

<aside>
<img src="https://www.notion.so/icons/sign-in_green.svg" alt="https://www.notion.so/icons/sign-in_green.svg" width="40px" /> - Topic1, Topic2, Topic3 가 모두 비어있는 경우 -> data에 32바이트(64글자)씩 parsing하여 Topic1, 2, 3을 채움 → ERC721 CA 개발자가 Event에 indexed를 모두 빠뜨리고 구현한 케이스
data[2:66] -> Topic2 -> seller
data[66:130] -> Topic3 -> buyer
data[130:194] -> Topic4 -> nft token id

- Topic1, Topic2, Topic3 가 모두 채워져있는 경우
Topic1 -> seller
Topic2 -> buyer
Topic3 -> nft token id

- Topic1, Topic2, Topic3 다 비어있거나, 다 채워져있으면 ERC721 정보 세팅
8. seller <- Topic1
9. buyer <- Topic2
10. nft_token_id <- Topic3
11. erc_standard <- "ERC721"

12. nft_contract_address <- contract address
13. nft_project_name <- 별도로 nft들의 메타 정보를 저장하는 테이블 관리 필요. nft contract address에 따른 nft project name을 저장하는 데이터를 확보해두고 해당 테이블에서 조회하여 저장

14. number_of_items <- number_of_items + 1 (bundle 거래가 있을 수 있으므로 기존 데이터에 +1)
15. trade_type <- number_of_items가 1개이면 "Single Item Trade", OrdersMatched 나오기 전까지 여러 개의 Transfer가 나오면(즉, 여러개의 NFT가 거래되면) "Bundle Trade"

16. nft_token_ids_array <- nft_token_id를 append
17. senders_array <- seller를 append
18. recipients_array <- buyer를 append
19. erc_types_array <- erc_standard를 append
20. nft_contract_addresses_array <- nft_contract_address를 append
21. erc_values_array <- 필자도 dune에서 만든 해당 필드의 의미를 알 수 없음, dune에서는 '','',''... 이런식으로 들어가 있는데 의미를 모르겠음
- 16~21 data는 bundle을 위해 있는 필드, single이어도 1개가 append 됨

22. original_currency <- "ETH"
23. original_currency_contract <- "zero address", dune 기준으로 original_currency가 ETH이면 이 필드는 zero address
24. currency_contract <- "WETH의 contract address", dune 기준으로 original_currency가 ETH이면 이 필드는 WETH의 contract address, 그 밖에도 다양한 ERC20들이 나타남
- 22~24 data는 default로 ETH에 대한 data가 들어감, 이후 erc20에 대한 transfer가 나오면 erc20으로 대체됨 → ERC20 Transfer 사후 처리 참조

</aside>

**Transfer Event를 통한 data 추출 이후 작업**

- Transfer Event를 통한 데이터 입력이 끝난 후 erc_standard 필드를 검사합니다. 만약 ERC721이 아니라면(즉, ERC20), tempData에 현재 데이터를 담아두어 아래에서 활용할 수 있도록 합니다.

```go
if nftTrade.ErcStandard != "erc721" {
	tempData = append(tempData, data)
}
```

**TransferSingle**

<aside>
<img src="https://www.notion.so/icons/sign-in_green.svg" alt="https://www.notion.so/icons/sign-in_green.svg" width="40px" /> Topic1 -> Operator
Topic2 -> Seller
Topic3 -> Buyer
data[2:66] -> NFT Token ID
data[66:130] -> NFT Token Count

8. seller <- Topic2
9. buyer <- Topic3
10. nft_token_id <- data[2:66]
11. erc_standard <- "ERC1155"

12. nft_contract_address <- contract address
13. nft_project_name <- 별도로 nft들의 메타 정보를 저장하는 테이블 관리 필요. nft contract address에 따른 nft project name을 저장하는 데이터를 확보해두고 해당 테이블에서 조회하여 저장

14. number_of_items <- number_of_items + data[66:130] (bundle 거래가 있을 수 있으므로 기존 데이터에 해당 transferSingle의 item 개수)
15. trade_type <- number_of_items가 1개이면 "Single Item Trade", OrdersMatched 나오기 전까지 여러개의 Transfer가 나오면(즉, 여러개의 NFT가 거래되면) "Bundle Trade"

16. nft_token_ids_array <- nft_token_id를 append
17. senders_array <- seller를 append
18. recipients_array <- buyer를 append
19. erc_types_array <- erc_standard를 append
20. nft_contract_addresses_array <- nft_contract_address를 append
21. erc_values_array <- 필자도 dune에서 만든 해당 필드의 의미를 알 수 없음, dune에서는 '','',''... 이런식으로 들어가 있는데 의미를 모르겠음
- 16~21 data는 bundle을 위해 있는 필드, single이어도 1개가 append 됨

22. original_currency <- "ETH"
23. original_currency_contract <- "zero address", dune 기준으로 original_currency가 ETH이면 이 필드는 zero address
24. currency_contract <- "WETH의 contract address", dune 기준으로 original_currency가 ETH이면 이 필드는 WETH의 contract address, 그 밖에도 다양한 ERC20들이 나타남
- 22~24 data는 default로 ETH에 대한 data가 들어감, 이후 erc20에 대한 transfer가 나오면 erc20으로 대체됨 → ERC20 Transfer 사후 처리 참조

</aside>

**OrdersMatched**

- OrersMatched Event를 만나면 tempData(Transfer Event에서 ERC721이 아니어서 저장해둔 data)가 비어있지 않다면 아래 로직을 수행합니다.
    - ERC20 Transfer 사후 처리
        - erc_standard 가 ERC721 or ERC1155 이면 tempData로 currency 관련된 정보를 추출합니다.
        
        <aside>
        <img src="https://www.notion.so/icons/sign-in_green.svg" alt="https://www.notion.so/icons/sign-in_green.svg" width="40px" /> 22. original_currency <- 별도로 contract address에 따른 token의 메타 정보를 저장하는 테이블 관리 필요. contract address에 따른 token symbol을 저장하는 데이터를 확보해두고 해당 테이블에서 조회하여 저장
        23. original_currency_contract <- contract address, dune 기준으로 original_currency가 ERC20면 이 필드는 ERC20의 contract address
        24. currency_contract <- contract address, dune 기준으로 original_currency가 ERC20면 이 필드는 ERC20의 contract address
        - 22~24 data는 default로 ETH에 대한 data가 들어감, 이후 해당 과정에서 erc20에 대한 정보로 대체됨
        
        </aside>
        
- OrdersMatched
    - OrdersMatched 건마다 bulk count를 증가시킵니다.
    - 한 Tx hash 내에 여러 개의 OrdersMatched가 있을 수 있습니다.

<aside>
<img src="https://www.notion.so/icons/sign-in_green.svg" alt="https://www.notion.so/icons/sign-in_green.svg" width="40px" /> Topic2 -> Maker
Topic3 -> Take
Topic4 -> MetaData
data[2:66] -> buyHash
data[66:130] -> sellHash
data[130:] -> raw amount

25. platform <- "OpenSea"
26. evt_type <- "Trade"
27. category <- "Buy", "Sell", Taker가 위의 transfer나 transferSingle에서 seller이면 Sell, buyer이면 Buy

28. exchange_contract_address <- contract_address
29. platform_version <- contract_address에 따라 v1, v2 구분

30. original_amount_raw <- data[130:]
31. original_amount <- original_amount_raw / 10^(erc20 token의 decimals) 로 계산
32. usd_amount <- original_amount * (block_time 때 해당 erc20의 usd 가격) 로 계산
33. trade_id <- single, bundle, bulk에 따른 trade id 값 어떻게 계산할 것인가

</aside>

<aside>
💡 **Maker와 Taker**
Maker: 거래를 만든 사람
Taker: 거래를 수락한 사람

Maker가 Seller이고, Taker가 Buyer이면 해당 거래는 Buy 입니다.
Maker가 Buyer이고, Taker가 Seller이면 해당 거래는 Sell 입니다.

</aside>

- 현재 TX Hash에 대한 로직이 모두 처리되었을 때 bulk count가 1보다 크면 현재 Tx hash의 NFT 거래 데이터의 trade type을 모두 "Bundle Trade"로 변경합니다.
- wyvern에서는 bulk 거래는 없고, 모두 bundle로 처리됩니다.

**현재 거래건(TX Hash)에 대한 데이터가 모두 채워지면 nft 거래 데이터를 nft_trades 테이블에 저장합니다.**

전체적으로 위의 수집, 분석, 적재하는 일련의 과정을 TX Hash 기준으로 반복하여, OpenSea Wyvern Protocol을 통해 이루어진 NFT 거래 데이터를 수집할 수 있습니다.

# 마치며

---

OpenSea의 Wyvern Protocol을 통한 거래 데이터 분석을 통해 OpenSea의 거래량, Top NFT, Top Trader 등의 지표들을 부가적으로 산출할 수 있었고, 이를 통해 쟁글의 애널리틱스에 다양한 지표들에 활용할 수 있었다.

쟁글의 백엔드 개발팀은 높은 블록체인 이해도를 기반으로 블록체인 데이터에 대한 다양한 리서치를 통해 이를 수집, 분석, 가공, 적재하여 다양한 서비스에 활용하고 있습니다.

쟁글의 백엔드 개발팀은 블록체인 도메인에서 블록체인 데이터의 분석 및 처리를 수행할 수 있는 경험을 가지고 있으며, 일반적인 백엔드 개발 기술도 숙련되어 있습니다. 이러한 경험 덕분에 쟁글의 백엔드 개발팀은 지속적으로 높은 수준의 목표를 달성하고 있습니다.

# 참고

---

## Trade Type

<aside>
💡 **Single Item Trade**

- ERC721 : 1개
- ERC1155 : 1개 이상

**Bundle Trade**

- 한 포장 안에 여러개의 상품이 들어있다고 생각하면 됨
- 예시 : 장바구니에 여러 NFT를 넣어서 한번에 구매
- 1건의 거래에서 여러개의 NFT가 묶여서 거래됨
- ordersmatched(wyvern) / orderfulfilled(seaport) 1개

**Bulk Purchase**

- 각각 포장된 상품을 테이프로 묶었다고 생각하면 됨
- 1개의 TX에서 별개의 거래건들이 묶여 있음
- seaport에만 있는 trade type
- orderfulfilled(seaport) 가 여러개
</aside>

## Event Name Hex Signature

- 유용한 사이트 - Event Signatures: [https://www.4byte.directory/event-signatures/](https://www.4byte.directory/event-signatures/)

| 분류 | Event Name | Hex Signature |
| --- | --- | --- |
| ERC20,ERC1155 | Approval(address,address,uint256) | 0x8c5be1e5ebec7d5bd14f71427d1e84f3dd0314c0f7b2291e5b200ac8c7c3b925 |
| ERC20,ERC721 | Transfer(address,address,uint256) | 0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef |
| ERC1155 | TransferSingle(address,address,address,uint256,uint256) | 0xc3d58168c5ae7397731d063d5bbf3d657854427343f4c083240f7aacaa2d0f62 |
| ERC1155 | TransferBatch(address,address,address,uint256[],uint256[]) | 0x4a39dc06d4c0dbc64b70af90fd698a233a518aa5d07e595d983b8c0526c8f7fb |
| ERC721,ERC1155 | ApprovalForAll(address,address,bool) | 0x17307eab39ab6107e8899845ad3d59bd9653f200f220920489ca2b5937696c31 |
| ERC1155 | URI(string,uint256) | 0x6bb7ff708619ba0610cba295a58592e0451dee2622938c8755667688daf3529b |
| Wyvern | OrdersMatched(bytes32,bytes32,address,address,uint256,bytes32) | 0xc4109843e0b7d514e4c093114b863f8e7d8d9a458c372cd51bfe526b588006c9 |

# Reference

---

## Ethereum 공식

ERC20 Token Standard: [https://eips.ethereum.org/EIPS/eip-20](https://eips.ethereum.org/EIPS/eip-20)

ERC721 NFT Standard: [https://eips.ethereum.org/EIPS/eip-721](https://eips.ethereum.org/EIPS/eip-721)

ERC1155 Multi Token Standard: [https://eips.ethereum.org/EIPS/eip-1155](https://eips.ethereum.org/EIPS/eip-1155)

Etherscan: [https://etherscan.io/](https://etherscan.io/)

dune: [https://dune.com/browse/dashboards](https://dune.com/browse/dashboards)

## OpenSea 공식

OpenSea Docs v1(wyvern): [https://docs.opensea.io/](https://docs.opensea.io/)

OpenSea Docs v2(seaport): [https://docs.opensea.io/v2.0/](https://docs.opensea.io/v2.0/)

OpenSea Github: [https://github.com/ProjectOpenSea/seaport](https://github.com/ProjectOpenSea/seaport)

OpenSea Blog: [https://opensea.io/blog/](https://opensea.io/blog)

OpenSea ERC1155 거래 지원: [https://opensea.io/blog/announcements/erc1155-marketplace/](https://opensea.io/blog/announcements/erc1155-marketplace/)

OpenSea Bundles: [https://opensea.io/blog/announcements/introducing-bundles/](https://opensea.io/blog/announcements/introducing-bundles/)

OpenSea Wyvern V1으로 Market 시작: [https://opensea.io/blog/announcements/bid-on-your-favorite-crypto-collectibles/](https://opensea.io/blog/announcements/bid-on-your-favorite-crypto-collectibles/)

OpenSea Wyvern V2 업그레이드: [https://opensea.io/blog/announcements/announcing-a-contract-upgrade/](https://opensea.io/blog/announcements/announcing-a-contract-upgrade/)

OpenSea Wyvern 2.3 Protocol: [https://opensea.io/blog/developers/wyvern-2-3-developer-upgrade-guide/](https://opensea.io/blog/developers/wyvern-2-3-developer-upgrade-guide/)

OpenSea Seaport Protocol Migration: [https://opensea.io/blog/announcements/launching-seaport-saving-the-community-millions-in-fees/](https://opensea.io/blog/announcements/launching-seaport-saving-the-community-millions-in-fees/)

OpenSea Seaport Protocol: [https://opensea.io/blog/announcements/introducing-seaport-protocol/](https://opensea.io/blog/announcements/introducing-seaport-protocol/)