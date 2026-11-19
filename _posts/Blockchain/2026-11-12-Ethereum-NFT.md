---
title: "Ethereum NFT"
date: 2026-11-12 01:00:00 +0900
last_modified_at: 2026-11-18
categories: [Blockchain]
tags: [Blockchain, NFT]
---

>
> 1. [NFT]({% post_url Blockchain/2026-11-12-NFT %})
> 2. **Ethereum NFT**
> 3. [Solana NFT]({% post_url Blockchain/2026-11-12-Solana-NFT %})
>

## Ethereum Core Concepts

### Accounts and Smart Contracts

Ethereum은 모든 상태를 [Account](https://ethereum.org/developers/docs/accounts/)에 저장하고, 각 Account는 20 Byte Address(주소)로 구분된다. Account는 두 종류다.

- **Externally Owned Account (EOA)**: Private Key(개인키)로 제어하는 Account다. Address는 Public Key(공개키)를 Keccak-256으로 Hash한 값의 마지막 20 Byte다.

- **Contract Account (CA)**: Blockchain에 배포된 Smart Contract다. Private Key가 없고 Code(배포된 실행 코드)가 동작을 결정한다. 이하 Contract라고 줄여 쓴다.

Contract는 Code와 Storage를 같은 Account 안에 가진다. [Solana]({% post_url Blockchain/2026-11-12-Solana-NFT %})에서는 Program이 상태를 갖지 않고 별도의 Data Account에 상태를 두지만, Ethereum에서는 Contract 하나가 로직과 상태를 모두 가진다. 그래서 Ethereum의 NFT(Non-Fungible Token, 대체 불가능 토큰)는 NFT마다 Account가 생기지 않고, Contract 하나의 Storage 안에 Collection 전체의 소유 기록이 들어간다.

Private Key를 보관하고 서명하는 프로그램을 Wallet(지갑)이라고 하며, 이 글에서는 사용자의 EOA Address를 Wallet Address라고도 쓴다.

### Transactions and Gas

상태를 바꾸려면 [Transaction](https://ethereum.org/developers/docs/transactions/)(트랜잭션)을 보낸다. Transaction은 EOA가 서명한 명령이다. Contract는 Transaction을 시작할 수 없고, 받은 Transaction을 처리하는 도중에만 다른 Contract를 호출할 수 있다.

- `to`가 Contract이면 그 Contract의 Code가 실행된다.
- 호출할 함수와 인자는 `input data`에 담는다. 앞 4 Byte가 Function Selector(함수 선택자)이고, 함수 시그니처를 Keccak-256으로 Hash한 값의 앞 4 Byte다.
- 실행 중 Revert(되돌림)가 일어나면 그 Transaction이 만든 상태 변경은 모두 취소된다.

[Gas](https://ethereum.org/developers/docs/gas/)(가스)는 연산과 저장에 드는 작업량을 재는 단위다. 수수료는 "사용한 Gas × Gas당 가격"이고, Transaction이 실패해도 낸다. ETH를 보내기만 하는 Transaction은 21,000 Gas를 쓴다. 연산 가운데 Storage에 값을 쓰는 비용이 가장 크다.

Ethereum에는 Solana의 Rent-Exempt Minimum 같은 저장 예치금이 없다. NFT를 저장하는 비용은 Storage에 값을 쓰는 Transaction의 Gas로 낸다.

### Storage and Events

Contract의 Storage는 256 Bit Key와 256 Bit 값의 쌍으로 이루어진 저장 공간이다. 한 칸을 Slot이라고 한다. Solidity의 `mapping`은 Key마다 Slot 하나를 쓴다([Layout of State Variables in Storage](https://docs.soliditylang.org/en/latest/internals/layout_in_storage.html)). 예를 들어 `mapping(uint256 tokenId => address)`는 TokenID마다 Slot 하나에 주소를 저장한다.

Event는 Contract가 실행 중에 남기는 기록이다. Storage와 달리 Contract가 다시 읽을 수 없고, 외부 프로그램이 상태 변화를 추적하는 데 쓴다. Event를 모아 조회용 데이터를 만드는 프로그램을 Indexer(인덱서)라고 한다.

### Token과 ERC

#### Token Contract

Ethereum의 Token은 별도의 Account가 아니라 Contract의 Storage에 있는 장부 기록이다. Fungible Token(대체 가능 토큰) 표준인 [ERC-20](https://eips.ethereum.org/EIPS/eip-20)은 "주소 → 잔액"을 기록한다. 같은 종류의 Token은 서로 구분되지 않으므로 수량만 있으면 된다. NFT는 Token 하나하나가 고유하므로 "어느 Token을 누가 가졌는지"를 기록해야 한다.

#### ERC와 EIP

ERC는 Ethereum Request for Comments의 줄임말이다. Ethereum의 변경 제안 문서인 EIP(Ethereum Improvement Proposal) 가운데 Application 수준의 표준을 다루는 분류다([EIP-1](https://eips.ethereum.org/EIPS/eip-1)).

ERC가 정하는 것은 구현 코드가 아니라 Interface다. Interface는 Contract가 외부에 제공해야 하는 함수와 Event의 목록이다. 모든 Contract가 같은 Interface를 따르므로, Wallet이나 Marketplace(마켓플레이스, NFT를 사고파는 서비스)는 Contract마다 내부 구현을 알지 못해도 같은 방법으로 Owner(소유자)를 조회하고 전송을 요청할 수 있다.

#### ERC-165

[ERC-165](https://eips.ethereum.org/EIPS/eip-165)는 Contract가 어떤 Interface를 구현하는지 조회하는 표준이다. 외부에서 `supportsInterface(bytes4 interfaceID)`를 호출해 확인한다. Interface Identifier는 4 Byte 값으로, Interface에 속한 모든 함수의 Function Selector를 XOR(Exclusive OR)한 결과다. ERC-721과 ERC-1155 모두 ERC-165 구현을 요구한다.

이 글에서 On-Chain은 Blockchain 위의 Contract Storage 또는 Code에 있다는 뜻이고, Off-Chain은 Blockchain 밖에 있다는 뜻이다.

<br />

## ERC NFT Standards

Ethereum에서 NFT를 만드는 표준은 ERC-721과 ERC-1155 두 가지다. 둘의 차이는 Contract가 장부에 무엇을 기록하는지에 있다. ERC-721은 "TokenID → Owner"를, ERC-1155는 "(ID, 주소) → 수량"을 기록한다.

표준은 Interface만 정하고 저장 구조는 정하지 않는다. 이 글에서 저장 구조와 동작은 널리 쓰이는 Smart Contract Library(라이브러리)인 [OpenZeppelin Contracts](https://docs.openzeppelin.com/contracts/api/token/erc721) 5.6.1의 구현을 기준으로 적는다.

| 표준 | 제안일(Created) | 다루는 Token | NFT 상태를 저장하는 곳(OpenZeppelin 5.6.1 기준) | NFT 식별자 | 승인 단위 | Mint Gas(이 글의 측정) |
|---|---|---|---|---|---|---|
| ERC-721 | [2018-01-24](https://eips.ethereum.org/EIPS/eip-721) | NFT | Contract Storage의 `mapping` 4개. `_owners[tokenId]`가 Owner를 직접 기록한다 | (Contract 주소, TokenID) | TokenID 1개(`approve`) 또는 전체(`setApprovalForAll`) | 62,164 ~ 96,364 |
| ERC-1155 | [2018-06-17](https://eips.ethereum.org/EIPS/eip-1155) | Fungible Token, NFT, Semi-Fungible Token | Contract Storage의 `mapping` 2개. `_balances[id][account]`가 수량을 기록한다. Owner를 직접 기록하는 곳은 없다 | (Contract 주소, ID). 그 ID의 총 공급량이 1일 때 NFT다 | 전체(`setApprovalForAll`)만 | 54,010 |

Mint(발행)는 Token을 새로 만드는 동작이다. Gas 값은 이 글의 예시 Contract를 실행해 측정한 Transaction 전체 사용량이며, 기본 21,000 Gas를 포함한다. 측정 환경은 OpenZeppelin Contracts 5.6.1, Solidity 0.8.38(Optimizer 끔, EVM Version `cancun`), `@ethereumjs/vm` 10.1.3(Cancun Hardfork 설정)이다. ERC-721의 값은 조건에 따라 달라진다. Contract의 첫 Mint는 96,364, 받는 주소의 첫 NFT는 79,264, 받는 주소가 이미 NFT를 가졌으면 62,164였다. ERC-1155의 값은 받는 주소가 그 ID를 처음 가 때의 값이다. ETH로 환산한 비용은 Gas 가격에 따라 달라지므로 적지 않았다.

<br />

## ERC-721

### What is ERC-721?

ERC-721은 Ethereum에서 NFT를 다루는 방법을 정한 표준이다. 사양 문서는 [EIP-721](https://eips.ethereum.org/EIPS/eip-721)이고 상태는 Final이다.

ERC-721 Contract 하나가 NFT Collection 하나다. Collection에 속한 NFT는 모두 같은 Contract의 Storage에 기록되고, TokenID로 구분된다. Solana Token Metadata처럼 Collection 소속을 따로 검증하는 절차가 없다. 어떤 NFT가 어느 Collection에 속하는지는 Contract 주소로 정해진다.

| 구분 | 내용 |
| --- | --- |
| 표준이 정하는 것 | NFT의 Owner 조회, 전송, 전송 권한 승인에 쓰는 함수와 Event |
| 표준이 정하지 않는 것 | NFT를 새로 만드는 Mint와 없애는 Burn(소각)의 방법, Contract 내부의 저장 구조, Metadata의 저장 위치 |

**장점**

- 가장 먼저 자리 잡은 NFT 표준이라 Wallet, Marketplace, 도구의 지원 범위가 넓다.
- 구조가 단순하다. NFT 하나는 (Contract 주소, TokenID) 쌍이고, `ownerOf(tokenId)` 호출 한 번으로 Owner를 On-Chain에서 조회한다.
- 전송 권한을 TokenID 1개 단위로 줄 수 있다.
- 받는 주소가 미리 준비할 것이 없다. Transfer(전송)는 같은 Contract의 Storage 값만 바꾸므로 새 Account를 만들지 않는다.

**단점**

- 표준 전송 함수는 한 번에 NFT 하나만 옮긴다. 여러 개를 옮기려면 Transaction을 여러 번 보내거나 별도 Contract를 써야 한다.
- Mint마다 Storage 쓰기가 필요해 NFT 수에 비례해 Gas가 든다. 이 글의 예시에서 Mint 한 번은 62,164 ~ 96,364 Gas였다.
- `transferFrom`은 수신자가 NFT를 다룰 수 있는지 확인하지 않는다. NFT를 내보낼 기능이 없는 Contract로 보내면 꺼낼 수 없게 될 수 있다.
- Royalty(로열티, NFT가 다시 팔릴 때 원 창작자에게 가는 수수료)를 강제할 수 없다.
- 필수 Interface만으로는 "이 주소가 가진 TokenID 전체"를 조회할 수 없다.
- Contract 하나에 Fungible Token을 함께 담을 수 없다.
- Metadata가 바뀌지 않는다는 것을 표준이 보장하지 않는다.

### ERC-721의 On-Chain 구성 요소

| 구성 요소 | 표준에서의 지위 | 설명 |
| --- | --- | --- |
| TokenID(Token Identifier) | 필수 | NFT를 식별하는 번호 |
| Owner | 필수 | NFT를 소유한 주소 |
| Approved Address(승인된 주소) | 필수 | NFT 1개의 전송 권한을 받은 주소 |
| Operator(운영자) | 필수 | Owner의 모든 NFT의 전송 권한을 받은 주소 |
| Name, Symbol | 선택 | NFT Collection의 이름과 약칭 |
| TokenURI | 선택 | Metadata 파일의 위치를 가리키는 URI(Uniform Resource Identifier, 통합 자원 식별자) |

필수 요소는 소유권에 관한 것이다. Name, Symbol, TokenURI는 EIP-721이 선택 사항으로 둔 Metadata 확장에 속한다.

**1. TokenID**

ERC-721 Contract 안의 NFT는 각각 `uint256` 타입의 식별자를 가진다. 이 글에서는 이 식별자를 TokenID라고 부른다.

- TokenID는 하나의 Contract 안에서 고유하다.
- (Contract 주소, TokenID) 쌍은 하나의 Ethereum Blockchain 안에서 고유하다.
- TokenID는 Contract가 존재하는 동안 바뀌면 안 된다.
- 호출자는 TokenID에 특정 규칙이 있다고 가정하면 안 된다. 예를 들어 0부터 순서대로 증가한다고 가정할 수 없다.

NFT의 Non-Fungibility(대체 불가능성)는 (Contract 주소, TokenID) 쌍에서 나온다. 예를 들어 [PudgyPenguins Contract](https://etherscan.io/token/0xbd3531da5cf5857e7cfaa92426877b022e612cf8?a=777)의 TokenID 777은 같은 Contract의 다른 TokenID와도, 다른 Contract의 TokenID 777과도 다른 NFT다. Solana Token Metadata에서 Mint Account Address가 하는 역할을 이 쌍이 한다.

**2. Owner**

해당 TokenID를 소유한 주소다. EOA일 수도 있고 Contract일 수도 있다. `ownerOf(tokenId)`로 조회한다.

**3. Approved Address와 Operator**

NFT 하나를 전송할 수 있는 주체는 Owner를 포함해 세 종류다.

- Approved Address: Owner가 특정 TokenID 하나에 대해 전송 권한을 준 주소다. TokenID 하나에 주소 하나만 지정된다.
- Operator: Owner가 자신의 모든 NFT에 대해 전송 권한을 준 주소다. Owner 한 명이 여러 Operator를 둘 수 있다.

Approved Address와 Operator는 Owner가 직접 Transaction을 보내지 않아도 제3자 Contract가 NFT를 옮길 수 있게 한다. Marketplace가 판매자의 NFT를 구매자에게 넘기는 동작이 이 권한 위에서 이루어진다.

**4. Name과 Symbol**

Name은 NFT Collection의 이름이고 Symbol은 약칭이다. Contract 하나에 한 쌍이 있다. 예를 들어 PudgyPenguins Contract의 Name은 `PudgyPenguins`, Symbol은 `PPG`다.

**5. TokenURI**

TokenURI는 TokenID에 대응하는 URI다. Contract의 `tokenURI(tokenId)` 함수가 반환하며, Metadata 파일의 위치를 가리킨다.

TokenURI 문자열 자체는 Contract가 저장하거나 계산해서 반환하는 On-Chain 값이다. Off-Chain에 있는 것은 그 URI가 가리키는 Metadata 파일이다. Blockchain이 보장하는 것은 소유권 기록과 URI 문자열까지다.

TokenURI를 만드는 방식은 두 가지다.

*Base URI와 TokenID를 이어 붙이는 방식*

TokenID마다 완전한 URI를 저장하지 않고, 공통 접두사인 Base URI 하나만 두고 조회 시점에 TokenID를 이어 붙인다.

- Base URI: `https://mycollection.com/metadata/`
- TokenID: `100`
- TokenURI: `https://mycollection.com/metadata/100`

OpenZeppelin 5.6.1의 기본 `ERC721`이 이 방식이다. 구분자를 자동으로 넣지 않으므로 Base URI가 `/`로 끝나야 한다.

*TokenID별로 URI를 저장하는 방식*

TokenID마다 다른 URI를 Contract Storage에 기록한다. OpenZeppelin의 [`ERC721URIStorage`](https://docs.openzeppelin.com/contracts/api/token/erc721) 확장이 이 방식이다. `_setTokenURI(tokenId, uri)`로 값을 저장한다.

두 방식의 차이는 비용과 유연성이다. 이어 붙이는 방식은 TokenID마다 저장할 값이 없으므로 Mint 비용이 작지만 URI 형태가 규칙에 묶인다. 저장하는 방식은 TokenID마다 임의의 URI를 줄 수 있지만 문자열 저장 비용이 든다. [OpenZeppelin 문서](https://docs.openzeppelin.com/contracts/api/token/erc721)도 `ERC721URIStorage`를 더 유연하지만 더 비싼 방식으로 설명한다.

**Interface**

위 구성 요소는 Interface를 통해 조회하고 변경한다. EIP-721은 필수 Interface 하나와 선택 확장 두 개, 그리고 NFT를 받는 Contract가 구현하는 Interface 하나를 정의한다.

| Interface | 구현 의무 | Interface Identifier | 다루는 구성 요소 |
| --- | --- | --- | --- |
| ERC721 | 필수 | `0x80ac58cd` | TokenID, Owner, Approved Address, Operator |
| ERC165 | 필수 | `0x01ffc9a7` | 지원하는 Interface 조회 |
| ERC721Metadata | 선택 | `0x5b5e139f` | Name, Symbol, TokenURI |
| ERC721Enumerable | 선택 | `0x780e9d63` | 전체 NFT 목록 열거 |
| ERC721TokenReceiver | Safe Transfer를 받는 Contract에 필요 | `0x150b7a02` | NFT 수신 가능 여부 응답 |

ERC721 Interface의 Event는 `Transfer`, `Approval`, `ApprovalForAll` 세 가지다. 함수는 역할에 따라 세 묶음으로 나뉜다. 표의 Zero Address(영 주소)는 `0x0000...0000` 주소를 뜻한다.

| 묶음 | 함수 | 동작 |
| --- | --- | --- |
| 조회 | `balanceOf` | 주소가 소유한 NFT 개수를 반환한다. Zero Address를 조회하면 실패한다. |
| 조회 | `ownerOf` | TokenID의 Owner를 반환한다. 유효하지 않은 TokenID면 실패한다. |
| 전송 | `transferFrom` | NFT를 `_from`에서 `_to`로 옮긴다. |
| 전송 | `safeTransferFrom` | `transferFrom`과 같되, 수신자가 Contract이면 수신 가능 여부를 확인한다. |
| 승인 | `approve` | TokenID 하나의 Approved Address를 지정한다. |
| 승인 | `setApprovalForAll` | 호출자의 모든 NFT에 대한 Operator를 지정하거나 해제한다. |
| 승인 | `getApproved` | TokenID의 Approved Address를 반환한다. 없으면 Zero Address를 반환한다. |
| 승인 | `isApprovedForAll` | 특정 주소가 Owner의 Operator인지 반환한다. |

이 Interface에 대해 알아둘 조건은 다음과 같다.

- Mint와 Burn 함수는 Interface에 없다. 다만 NFT가 생성될 때는 `_from`이 Zero Address인 `Transfer` Event를, Burn될 때는 `_to`가 Zero Address인 `Transfer` Event를 발생시킨다.
- Contract 생성 중에는 `Transfer` Event 없이 NFT를 생성하고 할당할 수 있다. 따라서 Event만으로 소유 현황을 복원하면 누락이 생길 수 있다.
- 전송 함수와 `approve`는 사양에서 `payable`로 선언되어 있다. EIP-721은 구현이 Interface보다 엄격한 Mutability(상태 변경 가능성)를 가져도 된다고 명시한다. OpenZeppelin 5.6.1은 이 함수들을 Non-Payable로 구현한다.

Name, Symbol, TokenURI를 조회하는 ERC721Metadata Interface는 다음과 같다. `tokenURI`는 유효하지 않은 TokenID면 실패한다.

```solidity
interface ERC721Metadata /* is ERC721 */ {
    function name() external view returns (string _name);
    function symbol() external view returns (string _symbol);
    function tokenURI(uint256 _tokenId) external view returns (string);
}
```

**저장 구조**

표준은 저장 구조를 정하지 않는다. OpenZeppelin 5.6.1의 [`ERC721.sol`](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v5.6.1/contracts/token/ERC721/ERC721.sol)은 네 개의 `mapping`으로 소유권과 승인 상태를 저장한다.

```solidity
mapping(uint256 tokenId => address) private _owners;
mapping(address owner => uint256) private _balances;
mapping(uint256 tokenId => address) private _tokenApprovals;
mapping(address owner => mapping(address operator => bool)) private _operatorApprovals;
```

| Mapping | 의미 | 조회 함수 |
| --- | --- | --- |
| `_owners` | TokenID의 Owner | `ownerOf` |
| `_balances` | 주소가 가진 NFT 개수 | `balanceOf` |
| `_tokenApprovals` | TokenID의 Approved Address | `getApproved` |
| `_operatorApprovals` | Owner별 Operator 지정 여부 | `isApprovedForAll` |

`_owners[tokenId]`가 Zero Address이면 그 TokenID는 존재하지 않는 것으로 취급한다.

### ERC-721의 Off-Chain 구성 요소

TokenURI는 Off-Chain의 Metadata 파일을 가리킨다. Metadata는 NFT의 이름, 설명, 이미지 주소 같은 부가 정보다. EIP-721은 TokenURI가 "ERC721 Metadata JSON Schema"를 따르는 JSON(JavaScript Object Notation) 파일을 가리킬 수 있다고 정한다. 이 Schema가 정의하는 속성은 세 개다.

| 속성 | 타입 | 의미 |
| --- | --- | --- |
| `name` | string | 자산의 이름 |
| `description` | string | 자산의 설명 |
| `image` | string | 이미지 자원을 가리키는 URI |

실제 NFT Metadata에는 이 세 속성 외의 값이 자주 들어간다. 아래는 [OpenSea의 Media and traits 문서](https://docs.opensea.io/docs/media-and-traits)에 실린 예시다(2026년 11월 기준). `external_url`과 `attributes`는 EIP-721 Schema에 없는 속성이며, OpenSea가 화면 표시에 사용하는 확장이다.

```json
{
    "name": "Dave Starbelly",
    "description": "Friendly OpenSea Creature that enjoys long swims in the ocean.",
    "image": "https://storage.googleapis.com/opensea-prod.appspot.com/puffs/3.png",
    "external_url": "https://openseacreatures.io/3",
    "attributes": [
        { "trait_type": "Base", "value": "Starfish" }
    ]
}
```

NFT를 보여주는 서비스는 다음 순서로 동작한다.

1. Contract의 `tokenURI(tokenId)`를 호출해 TokenURI를 얻는다.
2. TokenURI가 가리키는 위치에서 Metadata 파일을 가져온다.
3. Metadata 파일을 해석해 이름, 설명, 이미지를 표시한다.

Metadata 파일은 일반적으로 IPFS(InterPlanetary File System) 또는 중앙화된 서버에 저장한다. 중앙화된 서버에 둔 파일은 서버 운영자가 내용을 바꿀 수 있고, 서버가 중단되면 조회할 수 없다.

EIP-721은 URI가 변할 수 있다고 명시한다. 사양은 집을 나타내는 NFT의 Metadata가 시간이 지나며 바뀌는 경우를 예로 든다. 반면 TokenID는 바뀌면 안 된다. 그래서 NFT의 정체성은 TokenURI나 Metadata가 아니라 (Contract 주소, TokenID) 쌍에 있다.

ERC-721에는 Solana Token Metadata의 Is Mutable 같은 표준 필드가 없다. URI가 바뀔 수 있는지는 Contract의 Code에 달려 있다. 아래 예시 Contract는 `_baseURI`가 고정된 문자열을 반환하고 이를 바꾸는 함수가 없으므로 URI 문자열이 바뀌지 않는다. 이 경우에도 URI가 가리키는 위치의 파일 내용은 저장 위치의 성질을 따른다.

### ERC-721의 동작

이 절은 EIP-721의 요구사항과, 그것을 구현한 OpenZeppelin Contracts 5.6.1의 `ERC721.sol`을 함께 설명한다. 어느 쪽 내용인지는 문장마다 구분해 적는다.

**Mint**

Mint는 존재하지 않던 TokenID에 Owner를 처음 지정하는 동작이다. OpenZeppelin의 `_mint(to, tokenId)`는 다음 순서로 동작한다.

1. `to`가 Zero Address이면 `ERC721InvalidReceiver`로 Revert한다.
2. `_balances[to]`를 1 늘리고 `_owners[tokenId]`에 `to`를 기록한다.
3. `Transfer(address(0), to, tokenId)` Event를 발생시킨다.
4. 이전 Owner가 Zero Address가 아니었다면, 즉 이미 존재하는 TokenID였다면 `ERC721InvalidSender`로 Revert한다. Revert되면 2번과 3번의 변경도 모두 취소된다.

`_safeMint`는 `_mint`를 실행한 뒤 아래 Safe Transfer의 수신자 확인을 추가로 수행한다.

**Approve**

`approve(to, tokenId)`는 호출자가 TokenID의 Owner이거나 Owner의 Operator일 때만 성공한다. Approved Address는 자신이 받은 권한을 다른 주소에 다시 `approve`할 수 없다. 성공하면 `_tokenApprovals[tokenId]`가 바뀌고 `Approval` Event가 발생한다.

`setApprovalForAll(operator, approved)`는 호출자 자신의 모든 NFT에 대해 `operator`의 권한을 켜거나 끈다. 이후에 호출자가 새로 얻는 NFT에도 적용된다. `ApprovalForAll` Event가 발생한다.

| 구분 | `approve` | `setApprovalForAll` |
| --- | --- | --- |
| 범위 | TokenID 1개 | Owner의 모든 NFT |
| 지정 가능한 주소 수 | TokenID당 1개 | 제한 없음 |
| 전송 후 | 초기화된다 | 유지된다 |

**Transfer**

EIP-721은 호출자가 Owner, Approved Address, Operator 중 하나가 아니거나, `_from`이 현재 Owner가 아니거나, `_to`가 Zero Address이거나, `_tokenId`가 유효한 NFT가 아니면 `transferFrom`이 실패해야 한다고 정한다. OpenZeppelin의 `transferFrom(from, to, tokenId)`는 다음 순서로 동작한다.

1. `to`가 Zero Address이면 `ERC721InvalidReceiver`로 Revert한다.
2. 현재 Owner를 읽고, 호출자가 Owner, Operator, Approved Address 중 하나인지 확인한다. 아니면 TokenID가 존재하지 않을 때는 `ERC721NonexistentToken`으로, 존재할 때는 `ERC721InsufficientApproval`로 Revert한다.
3. `_tokenApprovals[tokenId]`를 Zero Address로 초기화한다.
4. 이전 Owner의 `_balances`를 1 줄이고 `to`의 `_balances`를 1 늘린다.
5. `_owners[tokenId]`에 `to`를 기록한다.
6. `Transfer(이전 Owner, to, tokenId)` Event를 발생시킨다.
7. 이전 Owner가 인자 `from`과 다르면 `ERC721IncorrectOwner`로 Revert한다.

3번의 초기화에서는 `Approval` Event가 발생하지 않는다. EIP-721은 `Transfer` Event가 발생하면 해당 NFT의 Approved Address도 초기화된 것으로 본다고 정하고 있다.

**Safe Transfer**

EIP-721은 `transferFrom`을 쓸 때 수신 가능 여부를 확인할 책임이 호출자에게 있다고 명시한다. `safeTransferFrom`은 이 확인을 Contract 수준에서 수행한다.

1. `transferFrom`과 같은 방식으로 전송한다.
2. `_to` 주소에 Code가 있는지 확인한다. Code 크기가 0보다 크면 Contract로 판단한다.
3. Contract이면 `_to`의 `onERC721Received`를 호출한다.
4. 반환값이 `0x150b7a02`가 아니면 전체 전송을 실패 처리한다.

`onERC721Received`는 NFT Contract가 아니라 받는 쪽 Contract가 구현하는 ERC721TokenReceiver Interface의 함수다. `0x150b7a02`는 `bytes4(keccak256("onERC721Received(address,address,uint256,bytes)"))`의 결과다.

```solidity
interface ERC721TokenReceiver {
    function onERC721Received(
        address _operator,
        address _from,
        uint256 _tokenId,
        bytes _data
    ) external returns (bytes4);
}
```

수신자가 EOA처럼 Code가 없는 주소이면 3번과 4번은 건너뛴다. `bytes data` 인자가 있는 `safeTransferFrom`은 그 값을 `onERC721Received`에 그대로 전달한다.

이 글의 실행에서 `onERC721Received`가 없는 Contract로 `safeTransferFrom`을 호출하면 `ERC721InvalidReceiver`로 Revert되었고, 같은 Contract로 `transferFrom`을 호출하면 성공해 Owner가 그 Contract로 바뀌었다.

`safeTransferFrom`은 수신 Contract가 "받을 수 있다"고 응답했는지만 확인한다. 수신 Contract가 실제로 NFT를 다시 내보낼 수 있는지는 보장하지 않는다.

**Burn**

Burn은 TokenID의 Owner 기록을 지우는 동작이다. OpenZeppelin의 `_burn(tokenId)`는 Approved Address를 초기화하고, Owner의 `_balances`를 1 줄이고, `_owners[tokenId]`를 Zero Address로 바꾼 뒤 `Transfer(이전 Owner, address(0), tokenId)` Event를 발생시킨다. 존재하지 않는 TokenID이면 `ERC721NonexistentToken`으로 Revert한다.

`_burn`은 호출자의 권한을 확인하지 않는 내부 함수다. 외부에 Burn 기능을 열 때는 권한 확인을 직접 추가하거나 `ERC721Burnable` 확장을 사용한다.

### ERC-721의 Transfer 구조 예시

Alice가 TokenID 0인 NFT 하나를 가지고 있고, 이 NFT를 Bob에게 보낸다. Alice가 `transferFrom(Alice, Bob, 0)`을 호출하면 같은 Contract의 Storage에서 다음 값이 바뀐다.

| Storage | Transfer 전 | Transfer 후 |
| --- | --- | --- |
| `_owners[0]` | Alice | Bob |
| `_balances[Alice]` | 1 | 0 |
| `_balances[Bob]` | 0 | 1 |
| `_tokenApprovals[0]` | 지정된 주소 또는 Zero Address | Zero Address |

새로 만들어지는 Account는 없다. Bob은 이 NFT를 처음 받더라도 미리 준비할 것이 없고, Transaction 수수료는 호출자인 Alice가 낸다. Solana Token Metadata에서는 Bob의 Token Account가 없으면 먼저 만들어야 하고, Solana Core에서는 Asset Account의 `owner` 필드 하나가 바뀐다. ERC-721의 Transfer는 Account 단위가 아니라 Contract Storage의 Slot 단위로 일어난다.

**예시 Contract**

OpenZeppelin 5.6.1을 상속해 만든 최소 구성의 ERC-721 Contract다. 5.6.1의 `ERC721.sol`은 `pragma solidity ^0.8.24`를 요구한다.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {ERC721} from "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";

contract MyCollection is ERC721, Ownable {
    uint256 private _nextTokenId;

    constructor(address initialOwner)
        ERC721("MyCollection", "MYC")
        Ownable(initialOwner)
    {}

    function _baseURI() internal pure override returns (string memory) {
        return "https://mycollection.com/metadata/";
    }

    function safeMint(address to) external onlyOwner returns (uint256 tokenId) {
        tokenId = _nextTokenId++;
        _safeMint(to, tokenId);
    }
}
```

- `ERC721("MyCollection", "MYC")`: Name과 Symbol을 지정한다.
- `Ownable`: Contract 관리자를 한 명 두는 OpenZeppelin Contract다. `onlyOwner`가 붙은 함수는 관리자만 호출할 수 있다. 여기서의 관리자는 NFT의 Owner와 다른 개념이다.
- `_baseURI`: Base URI를 지정한다. `tokenURI`는 여기에 TokenID를 이어 붙인다.
- `safeMint`: TokenID를 0부터 1씩 늘려가며 Mint한다. 함수 이름과 권한은 이 Contract가 정한 것이다.

**호출 흐름**

Deployer(배포자), Alice, Bob, Carol 네 주소가 있고 모두 EOA다. Deployer가 `initialOwner`로 자신을 지정해 Contract를 배포했다. 아래 결과는 ERC NFT Standards 절에 적은 측정 환경에서 실행해 확인한 것이다.

| 순서 | 호출자 | 호출 | 결과 |
| --- | --- | --- | --- |
| 1 | Deployer | `safeMint(Alice)` | TokenID 0 생성. `Transfer(address(0), Alice, 0)` 발생 |
| 2 | 누구나 | `ownerOf(0)` | Alice |
| 3 | 누구나 | `tokenURI(0)` | `https://mycollection.com/metadata/0` |
| 4 | Alice | `approve(Bob, 0)` | `Approval(Alice, Bob, 0)` 발생. `getApproved(0)`은 Bob |
| 5 | Bob | `transferFrom(Alice, Carol, 0)` | `Transfer(Alice, Carol, 0)` 발생. `ownerOf(0)`은 Carol |
| 6 | 누구나 | `getApproved(0)` | Zero Address. 전송 시 초기화됨 |
| 7 | Bob | `transferFrom(Carol, Bob, 0)` | `ERC721InsufficientApproval(Bob, 0)`로 Revert |

5번에서 Bob은 Owner가 아니지만 Approved Address이므로 전송할 수 있다. 전송과 함께 승인이 초기화되므로 7번에서 Bob은 같은 NFT를 다시 옮길 수 없다. 4번에서 Alice가 `approve` 대신 `setApprovalForAll(Bob, true)`를 호출했더라도 7번은 실패한다. Operator 권한은 Alice의 NFT에 대한 것이고, 전송 후의 Owner는 Carol이기 때문이다.

### 확장: Enumerable, Metadata Update, Royalty, Batch Mint

#### Enumerable

필수 Interface의 `balanceOf`는 개수만 반환한다. EIP-721의 선택 확장인 ERC721Enumerable은 Owner가 가진 TokenID 목록과 전체 NFT 목록을 On-Chain에서 조회할 수 있게 한다.

```solidity
interface ERC721Enumerable /* is ERC721 */ {
    function totalSupply() external view returns (uint256);
    function tokenByIndex(uint256 _index) external view returns (uint256);
    function tokenOfOwnerByIndex(address _owner, uint256 _index) external view returns (uint256);
}
```

- `totalSupply`: Contract가 추적하는 유효한 NFT의 개수
- `tokenByIndex`: 전체 NFT 중 `_index`번째 TokenID
- `tokenOfOwnerByIndex`: 특정 Owner의 NFT 중 `_index`번째 TokenID

EIP-721은 열거 순서를 정하지 않는다. OpenZeppelin의 `ERC721Enumerable`은 전송, Mint, Burn마다 목록을 갱신하는 Storage 쓰기가 추가되며, [OpenZeppelin 문서](https://docs.openzeppelin.com/contracts/api/token/erc721)는 이 확장에 상당한 Gas 비용이 든다고 설명한다. 기본 `ERC721`은 Metadata 확장은 포함하지만 Enumerable 확장은 포함하지 않는다. 이 글의 예시 Contract에서 `supportsInterface(0x780e9d63)`은 `false`였다.

#### Metadata Update

EIP-721은 URI가 바뀔 수 있다고 하지만, 바뀌었음을 알리는 방법은 정하지 않는다. [ERC-4906](https://eips.ethereum.org/EIPS/eip-4906)은 ERC-721의 확장으로, Metadata가 바뀌었음을 외부에 알리는 `MetadataUpdate(uint256 _tokenId)` Event를 정의한다. OpenZeppelin의 `ERC721URIStorage`는 ERC-4906의 Interface인 `IERC4906`을 상속하고, `_setTokenURI`로 값을 저장할 때 이 Event를 발생시킨다.

#### Royalty

ERC-721의 전송 함수는 NFT의 Owner만 바꾸고 판매 대금을 다루지 않는다. 그래서 표준 수준에서 Royalty를 강제할 수 없다. [ERC-2981](https://eips.ethereum.org/EIPS/eip-2981)은 `royaltyInfo` 함수로 수령 주소와 금액을 알려줄 뿐 지급을 강제하지 않으며, 지급이 자발적이어야 한다고 명시한다. OpenZeppelin의 `ERC721Royalty`가 이 표준을 구현한다. Solana Token Metadata의 `seller_fee_basis_points`가 참고용 값인 것과 같은 성격이다.

#### Batch Mint

ERC-721은 NFT마다 `Transfer` Event를 요구하므로 대량 Mint에서 Event 비용이 NFT 수에 비례한다. [ERC-2309](https://eips.ethereum.org/EIPS/eip-2309)는 `ConsecutiveTransfer` Event 하나로 연속된 TokenID의 생성을 알리는 방법을 정의한다. OpenZeppelin의 `ERC721Consecutive`가 이를 구현하며, Contract 생성 시의 일괄 Mint에만 쓸 수 있다.

<br />

## ERC-1155

### What is ERC-1155?

[ERC-1155](https://eips.ethereum.org/EIPS/eip-1155)는 Contract 하나로 여러 종류의 Token을 다루는 Multi Token Standard다. Contract 하나에 Fungible Token, NFT, Semi-Fungible Token(준대체 가능 토큰, 같은 ID끼리만 서로 대체되는 Token)을 함께 담을 수 있다. 상태는 Final이다.

ERC-721의 TokenID는 NFT 하나를 가리킨다. ERC-1155의 ID는 Token 종류 하나를 가리키고, 주소마다 그 ID를 몇 개 가졌는지를 기록한다. ID마다 Metadata와 공급량을 따로 가질 수 있다.

ERC-1155에는 "이 ID는 NFT다"라고 표시하는 필드가 없다. EIP-1155는 NFT를 표현하는 방법으로 두 가지를 든다.

- 공급량으로 구분: 총 공급량이 1인 ID를 NFT로 본다.
- ID Bit 분할: 256 Bit ID의 상위 128 Bit를 Token 종류로, 하위 128 Bit를 그 종류 안의 NFT 번호로 쓴다. 사양이 예시로 든 방식이며 강제하지 않는다.

**장점**

- Contract 하나에 여러 종류의 Token을 담는다. Token 종류마다 Contract를 배포하지 않아도 된다.
- 여러 ID를 한 번에 옮기는 `safeBatchTransferFrom`과 여러 잔액을 한 번에 조회하는 `balanceOfBatch`가 표준에 있다. 이 글의 측정에서 ID 5개를 `_mintBatch`로 한 번에 Mint하면 152,919 Gas였고, 하나씩 5번 Mint하면 270,050 Gas(54,010 × 5)였다.
- 표준 전송 함수가 모두 Safe 방식이다. 받는 쪽이 Contract이면 항상 수신 가능 여부를 확인한다.
- URI 하나에 `{id}` 치환 규칙을 써서 모든 ID의 Metadata 위치를 표현할 수 있다.

**단점**

- `ownerOf`가 없다. 특정 NFT를 누가 가졌는지 On-Chain 함수로 조회할 수 없고, Event를 추적해야 한다.
- 승인은 `setApprovalForAll`뿐이다. ID 1개 단위로 전송 권한을 줄 수 없다.
- Non-Fungibility를 표준이 강제하지 않는다. 공급량을 1로 유지하는 것은 Contract의 책임이다. 이 글의 실행에서 OpenZeppelin의 기본 `ERC1155`는 이미 Mint된 ID를 다시 Mint하는 호출을 막지 않았다.
- Name과 Symbol이 표준에 없다.
- Contract가 Token을 받으려면 ERC1155TokenReceiver를 구현해야 한다. ERC-721의 `transferFrom`처럼 확인 없이 보내는 표준 함수가 없다.

### ERC-1155의 On-Chain 구성 요소

| 구성 요소 | 설명 |
| --- | --- |
| ID | Token 종류를 식별하는 `uint256` 값 |
| Balance | (ID, 주소)마다의 보유 수량 |
| Operator | Owner의 모든 Token의 전송 권한을 받은 주소 |
| URI | Metadata 파일의 위치를 가리키는 URI. 선택 확장인 ERC1155Metadata_URI에 속한다 |

**Interface**

| Interface | 구현 의무 | Interface Identifier |
| --- | --- | --- |
| ERC1155 | 필수 | `0xd9b67a26` |
| ERC165 | 필수 | `0x01ffc9a7` |
| ERC1155Metadata_URI | 선택 | `0x0e89341c` |
| ERC1155TokenReceiver | Token을 받는 Contract에 필요 | `0x4e2312e0` |

아래는 EIP-1155가 정의한 ERC1155 Interface에서 주석을 뺀 것이다.

```solidity
interface ERC1155 /* is ERC165 */ {
    event TransferSingle(address indexed _operator, address indexed _from, address indexed _to, uint256 _id, uint256 _value);
    event TransferBatch(address indexed _operator, address indexed _from, address indexed _to, uint256[] _ids, uint256[] _values);
    event ApprovalForAll(address indexed _owner, address indexed _operator, bool _approved);
    event URI(string _value, uint256 indexed _id);

    function safeTransferFrom(address _from, address _to, uint256 _id, uint256 _value, bytes calldata _data) external;
    function safeBatchTransferFrom(address _from, address _to, uint256[] calldata _ids, uint256[] calldata _values, bytes calldata _data) external;

    function balanceOf(address _owner, uint256 _id) external view returns (uint256);
    function balanceOfBatch(address[] calldata _owners, uint256[] calldata _ids) external view returns (uint256[] memory);

    function setApprovalForAll(address _operator, bool _approved) external;
    function isApprovedForAll(address _owner, address _operator) external view returns (bool);
}
```

ERC-721과 다른 점은 다음과 같다.

- 전송 함수에 수량 인자 `_value`가 있다. NFT는 `_value`를 1로 보낸다.
- `ownerOf`, `approve`, `getApproved`, 확인 없는 `transferFrom`이 없다.
- Event의 첫 인자 `_operator`는 전송을 실행한 주소다. Owner가 직접 보내면 `_operator`와 `_from`이 같다.
- Mint는 `_from`이 Zero Address인 `TransferSingle` 또는 `TransferBatch` Event로, Burn은 `_to`가 Zero Address인 Event로 알린다. EIP-1155는 Contract 생성 시 정한 초기 잔액까지 포함해, Event만으로 잔액을 계산할 수 있어야 한다고 정한다. Contract 생성 중의 Event 생략을 허용하는 ERC-721과 다른 점이다.

**수신자 확인**

EIP-1155는 `_to`가 Contract이면(Code 크기가 0보다 크면) 전송 함수가 수신 Contract의 Hook을 호출해야 한다고 정한다. EOA에는 호출하지 않는다.

| 전송 | 호출되는 함수 | 받아들일 때의 반환값 |
| --- | --- | --- |
| `safeTransferFrom` | `onERC1155Received` | `0xf23a6e61` |
| `safeBatchTransferFrom` | `onERC1155BatchReceived` | `0xbc197c81` |

다른 값을 반환하거나 함수가 없으면 전송 전체가 Revert된다. 잔액 변경과 Event 발생은 Hook 호출보다 먼저 이루어져야 한다. OpenZeppelin 5.6.1은 `_mint`와 `_mintBatch`에서도 같은 확인을 수행한다. 이 글의 실행에서 `onERC721Received`만 구현한 Contract로 ERC-1155 Token을 보내면 `ERC1155InvalidReceiver`로 Revert되었다.

**저장 구조**

OpenZeppelin 5.6.1의 [`ERC1155.sol`](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v5.6.1/contracts/token/ERC1155/ERC1155.sol)은 다음 세 값을 저장한다.

```solidity
mapping(uint256 id => mapping(address account => uint256)) private _balances;
mapping(address account => mapping(address operator => bool)) private _operatorApprovals;
string private _uri;
```

ERC-721의 `_owners`에 해당하는 값이 없다. NFT 하나를 Mint할 때 쓰는 Slot이 `_balances[id][account]` 하나뿐이어서 Mint Gas가 ERC-721보다 작다.

### ERC-1155의 Off-Chain 구성 요소

`uri(uint256 _id)`가 반환하는 URI는 Off-Chain의 JSON 파일을 가리킨다. URI에 `{id}` 문자열이 있으면 Client(클라이언트, Contract를 호출하는 쪽)가 실제 ID로 바꿔서 쓴다. 바꿔 넣는 값은 `0x` 접두사 없이 소문자 16진수로 쓰고, 앞을 0으로 채워 64자로 맞춘다.

- `uri(1)`의 반환값: `https://mycollection.com/items/{id}.json`
- Client가 조회하는 주소: `https://mycollection.com/items/0000000000000000000000000000000000000000000000000000000000000001.json`

OpenZeppelin의 기본 `ERC1155`는 모든 ID에 같은 URI를 반환하고 치환을 Client에 맡긴다. 이 글의 실행에서도 `uri(1)`은 `{id}`가 그대로 들어 있는 문자열을 반환했다. ID마다 다른 URI를 저장하려면 `ERC1155URIStorage` 확장을 쓴다.

EIP-1155의 "ERC-1155 Metadata URI JSON Schema"가 정의하는 속성은 다음과 같다.

| 속성 | 타입 | 의미 |
| --- | --- | --- |
| `name` | string | 자산의 이름 |
| `decimals` | integer | 수량을 표시할 때 쓰는 소수점 아래 자릿수 |
| `description` | string | 자산의 설명 |
| `image` | string | 이미지 자원을 가리키는 URI |
| `properties` | object | 임의의 추가 속성 |
| `localization` | object | 선택 사항. 언어별 Metadata의 위치 |

URI가 바뀌면 `URI(string _value, uint256 indexed _id)` Event로 알린다. EIP-1155는 `uri` 함수로 Token의 존재 여부를 확인하면 안 된다고 정한다.

### ERC-1155의 Transfer 구조 예시

Alice가 ID 1인 Token 1개(NFT)를 가지고 있고, 이를 Bob에게 보낸다. Alice가 `safeTransferFrom(Alice, Bob, 1, 1, "")`을 호출하면 다음 값이 바뀐다.

| Storage | Transfer 전 | Transfer 후 |
| --- | --- | --- |
| `_balances[1][Alice]` | 1 | 0 |
| `_balances[1][Bob]` | 0 | 1 |

Owner를 기록하는 값이 없으므로, 소유권 이동은 "ID 1의 수량 1이 어느 주소에 있는가"로 표현된다. 수량 1이 어느 Token Account에 있는지로 소유를 표현하는 Solana Token Metadata와 같은 방식이다. 다만 ERC-1155에서는 이 수량이 Account가 아니라 Contract Storage의 Slot에 있다.

**예시 Contract**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {ERC1155} from "@openzeppelin/contracts/token/ERC1155/ERC1155.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";

contract MyItems is ERC1155, Ownable {
    constructor(address initialOwner)
        ERC1155("https://mycollection.com/items/{id}.json")
        Ownable(initialOwner)
    {}

    function mint(address to, uint256 id, uint256 value) external onlyOwner {
        _mint(to, id, value, "");
    }

    function mintBatch(address to, uint256[] calldata ids, uint256[] calldata values) external onlyOwner {
        _mintBatch(to, ids, values, "");
    }
}
```

**호출 흐름**

ERC-721 예시와 같은 주소와 환경에서 실행해 확인한 결과다. ID 1은 NFT로, ID 2는 Fungible Token으로 쓴다.

| 순서 | 호출자 | 호출 | 결과 |
| --- | --- | --- | --- |
| 1 | Deployer | `mint(Alice, 1, 1)` | `TransferSingle(Deployer, address(0), Alice, 1, 1)` 발생 |
| 2 | Deployer | `mint(Alice, 2, 100)` | `TransferSingle(Deployer, address(0), Alice, 2, 100)` 발생 |
| 3 | 누구나 | `uri(1)` | `https://mycollection.com/items/{id}.json` |
| 4 | Alice | `safeTransferFrom(Alice, Bob, 1, 1, "")` | `TransferSingle(Alice, Alice, Bob, 1, 1)` 발생. `balanceOf(Alice, 1)`은 0, `balanceOf(Bob, 1)`은 1 |
| 5 | Alice | `safeTransferFrom(Alice, Bob, 2, 30, "")` | `balanceOf(Alice, 2)`는 70, `balanceOf(Bob, 2)`는 30 |
| 6 | Carol | `safeTransferFrom(Bob, Carol, 1, 1, "")` | `ERC1155MissingApprovalForAll(Carol, Bob)`로 Revert |
| 7 | Deployer | `mint(Alice, 1, 1)` | 성공. ID 1의 총 공급량이 2가 된다 |

7번 이후 ID 1은 Alice와 Bob이 1개씩 가진 Token이 되어 더 이상 NFT가 아니다. ID를 NFT로 쓰려면 같은 ID를 다시 Mint하지 못하게 Contract에서 막아야 한다. OpenZeppelin의 `ERC1155Supply` 확장은 ID별 `totalSupply(id)`와 `exists(id)`를 제공하므로 이 확인에 쓸 수 있다.