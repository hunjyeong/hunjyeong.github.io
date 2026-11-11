---
title: "Solana NFT"
date: 2026-11-11
categories: [Blockchain]
tags: [Blockchain, NFT]
---

# Solana NFT 표준 정리: Token Metadata, Core, Compressed NFT

> 기준 자료: Metaplex 공식 문서와 Solana 공식 문서, Metaplex 월간 보고, 각 Wallet의 공식 자료. 출처는 본문 안에 링크로 적는다.
> 수치(비용, 수수료, 지원 현황)는 기준일의 공식 문서 값이며, Program 버전과 시점에 따라 달라질 수 있다.

## 1. 배경: Solana의 NFT는 왜 Account가 여러 개인가

### 1.1 Solana의 Account 모델

Solana에서는 모든 데이터가 Account(계정)에 저장된다. Program(프로그램, 다른 체인의 스마트 컨트랙트에 해당)은 실행 코드만 가지고, 상태 데이터는 Program이 소유한 별도의 Account에 둔다. 따라서 하나의 자산을 표현할 때도 "무엇을 저장하느냐"에 따라 Account가 나뉜다.

Account는 저장하는 데이터 크기에 비례하는 SOL을 예치해야 유지된다. 이 예치금을 Rent(임대료)라고 부른다. Account를 닫으면 예치금을 돌려받는다. Account 수가 많고 크기가 클수록 NFT 하나를 만드는 비용이 커지는 이유가 여기에 있다.

### 1.2 SPL Token의 기본 구조

SPL(Solana Program Library) Token Program은 Solana의 기본 Token Program이다. [Solana 문서](https://solana.com/docs/tokens)에 따르면 이 Program은 Token을 두 종류의 Account로 표현한다.

- Mint Account: Token 종류 하나를 대표한다. Supply(총 발행량), Decimals(소수점 자릿수), Mint Authority(추가 발행 권한), Freeze Authority(동결 권한)를 저장한다. Mint Account의 주소가 그 Token의 식별자가 된다.
- Token Account: 특정 Wallet(지갑)이 특정 Mint의 Token을 얼마나 가지고 있는지를 저장한다. 주요 필드는 Mint, Owner(소유자), Amount(수량)이다.

![Mint Account와 Token Account의 관계 (출처: Solana 문서)](https://solana.com/assets/docs/core/tokens/token-account-relationship-light.svg)

Wallet은 보유하려는 Mint마다 Token Account가 하나씩 필요하다. 이때 주소를 매번 따로 관리하지 않도록 ATA(Associated Token Account)를 사용한다. ATA는 Wallet 주소, Token Program 주소, Mint 주소를 Seed로 하여 Associated Token Program에서 유도한 주소에 만든 Token Account다([Solana 문서의 ATA 설명](https://solana.com/ko/docs/tokens/basics/create-token-account)). 주소 유도 방식만 정해져 있을 뿐, ATA도 일반 Token Account다.

![Associated Token Account의 주소 유도 (출처: Solana 문서)](https://solana.com/assets/docs/core/tokens/associated-token-account-light.svg)

![Wallet, ATA, Mint의 관계 (출처: Solana 문서)](https://solana.com/assets/docs/core/tokens/token-account-relationship-ata-light.svg)

이렇게 정해진 입력으로부터 결정적으로 유도되는 주소를 PDA(Program Derived Address)라고 한다. PDA는 Private Key가 없고, 주소를 유도한 Program만 해당 Account에 대해 서명할 수 있다. 이후에 나오는 Metadata Account, Master Edition Account, TreeConfig Account가 모두 PDA다.

### 1.3 SPL Token만으로는 부족한 것

NFT(Non-Fungible Token, 대체 불가능 토큰)는 SPL Token 관점에서 "Decimals가 0이고 Supply가 1인 Token"이다. 그러나 Mint Account에는 이름, 이미지 같은 정보를 담을 필드가 없다. 또한 Supply가 1이라는 사실만으로는 이후에 추가 발행되지 않는다는 보장이 없다.

Metaplex는 이 부족한 부분을 채우는 Program들을 제공한다. Solana NFT를 만들고 관리하는 데 사실상 표준으로 쓰인다.

## 2. Metaplex의 NFT 표준 세 가지

| 표준 | 출시 | 담당 Program | 핵심 구조 |
|---|---|---|---|
| Token Metadata | 2021년 8월 | Token Metadata Program | SPL Token의 Mint Account에 Metadata Account 등을 PDA로 덧붙인다 |
| Core | 2024년 3월 발표, 4월 Mainnet | MPL Core Program | NFT 하나를 Asset Account 하나로 표현한다. SPL Token을 쓰지 않는다 |
| Compressed NFT | Bubblegum V1: 2022년 11월 발표, 2023년 4월 Mainnet 가동 공표. Bubblegum V2: 2025년 5월 말 | Bubblegum Program | NFT 데이터의 Hash만 Merkle Tree(머클 트리)에 넣고, 원본 데이터는 Transaction(트랜잭션)에 남긴다 |

출시 시점의 근거는 다음과 같다. Token Metadata는 [Metaplex Whitepaper](https://whitepaper.metaplex.foundation/whitepaper.pdf)의 "2021년 8월 출시 이후" 문구를 따랐다. Core는 [Metaplex 2024년 3월 월간 보고](https://www.metaplex.com/posts/metaplex-march-round-up)에서 소개되었고, [2024년 4월 월간 보고](https://metaplex.com/posts/metaplex-april-round-up)가 4월에 Mainnet에 올라갔다고 적고 있다. Bubblegum V1은 Metaplex가 [2022년 11월 6일 발표문](https://metaplex.com/posts/expanding-digital-assets-with-compression-for-nfts)으로 공개했고, Solana 재단은 [2023년 4월 글](https://solana.com/ko/news/how-to-use-compressed-nfts-on-solana)에서 cNFT가 Mainnet-beta에서 가동 중이라고 밝혔다. Bubblegum V2는 [Metaplex 2025년 6월 월간 보고](https://metaplex-website.vercel.app/blog/articles/metaplex-june-round-up-2025)가 5월 말 출시로 적고 있다.

세 표준은 "NFT의 상태를 어디에 저장하느냐"가 다르다. Token Metadata는 여러 Account, Core는 하나의 Account, Compressed NFT는 Account가 아닌 Ledger(원장)의 Transaction 기록에 저장한다.

[Metaplex의 Token Metadata 문서](https://www.metaplex.com/docs/smart-contracts/token-metadata)는 Token Metadata를 Legacy(레거시) Program으로 분류한다. 계속 지원하지만 신규 프로젝트에는 Core를 권장한다.

## 3. Token Metadata

### 3.1 온체인 Account 구성

일반적인 NFT(Token Standard가 `NonFungible`인 경우) 하나는 다음 4개의 Account로 이루어진다. Account의 역할과 PDA Seed는 [Token Metadata 개요](https://www.metaplex.com/docs/smart-contracts/token-metadata)와 [Solana의 Metaplex 설명 문서](https://solana.com/docs/tokens/metaplex.md)를 따랐다.

| Account | 소유 Program | 주소 | 저장 내용과 역할 |
|---|---|---|---|
| Mint Account | SPL Token Program | 임의의 Keypair 주소 | NFT의 고유 식별자. Supply 1, Decimals 0 |
| Token Account | SPL Token Program | 보통 ATA (Wallet, Token Program, Mint로 유도) | 현재 소유 관계. Mint, Owner, Amount(1) |
| Metadata Account | Token Metadata Program | PDA: `["metadata", Program ID, Mint 주소]` | Name, Symbol, URI, Seller Fee Basis Points, Creators, Update Authority, Is Mutable, Token Standard, Collection 등 |
| Master Edition Account | Token Metadata Program | PDA: `["metadata", Program ID, Mint 주소, "edition"]` | 대체 불가능성의 증명. Mint Authority와 Freeze Authority를 넘겨받는다. Max Supply(복제본 발행 한도) 보관 |

Master Edition Account가 필요한 이유는 다음과 같다.

- Token Metadata Program은 Master Edition Account를 만들기 전에 Mint가 NFT의 조건(Decimals 0, Supply 1)을 만족하는지 확인한다.
- 그 뒤 Mint Authority와 Freeze Authority를 Master Edition PDA로 이전한다. 이후에는 Token Metadata Program을 거치지 않고는 누구도 추가 발행이나 동결을 할 수 없다. 권한을 없애지 않고 이전하는 이유는 [Token Metadata FAQ](https://www.metaplex.com/docs/smart-contracts/token-metadata/faq)에서 설명한다.
- 따라서 Master Edition Account의 존재가 해당 Mint가 Non-Fungible이라는 증명이 된다.

Metaplex 공식 문서는 Token Metadata의 Account 수를 "3개 이상(Mint, Metadata, Token Account)"으로 표현한다. 표준 NFT 기준으로는 Master Edition을 포함해 4개이고, 3.5절의 Programmable NFT는 Token Record가 추가되어 5개다.

참고로 2024년 10월 10일 Program 변경 이후 새로 만들어지는 Account의 크기는 Metadata 607 Byte(이전 679), Master Edition 20 Byte(이전 282), Edition 42 Byte(이전 241)다([Account Size Reduction 문서](https://www.metaplex.com/docs/smart-contracts/token-metadata/guides/account-size-reduction)).

### 3.2 오프체인 구성 요소

URI(Uniform Resource Identifier) 문자열 자체는 Metadata Account에 온체인으로 저장된다. 오프체인에 있는 것은 URI가 가리키는 대상이다.

- NFT Metadata JSON(JavaScript Object Notation) 파일: Description, Image, Attributes, Animation URL 등 확장 정보를 담는다.
- 미디어 파일: JSON의 Image 필드 등이 가리키는 이미지나 영상.

저장 위치는 Arweave, IPFS(InterPlanetary File System), 또는 중앙화 서버(예: Amazon Web Services)다. 오프체인 파일이 사라지면 온체인의 Name과 URI는 남지만 이미지와 속성은 조회할 수 없게 된다. 이 때문에 Metaplex 문서는 Arweave나 Pinning된 IPFS 같은 영구 저장소를 권장한다.

### 3.3 온체인과 오프체인의 연결

```
[Wallet: Alice]
      │ Owner
      ▼
[Token Account (ATA)]          [Metadata Account (PDA)]
  Mint ───────────────┐          Mint ─────────┐
  Owner = Alice       │          Name, Symbol  │
  Amount = 1          ▼          URI ──────────┼──▶ [JSON 파일 (오프체인)]
                 [Mint Account] ◀──────────────┘         Image ──▶ [미디어 파일]
                   Supply = 1
                   Decimals = 0
                   Mint Authority ──▶ [Master Edition Account (PDA)]
                   Freeze Authority ─▶ (동일)
```

원본 구조 그림은 [Token Metadata 개요 페이지](https://www.metaplex.com/docs/smart-contracts/token-metadata)의 NFTs 절에 대화형으로 실려 있다. Mint Account와 Metadata PDA의 연결만 보여주는 간단한 그림은 [Solana의 Metaplex 설명 문서](https://solana.com/docs/tokens/metaplex.md)에 있다.

Metadata Account와 Master Edition Account는 Mint 주소에서 유도되므로, Mint 주소만 알면 나머지 주소를 계산할 수 있다. 반대 방향인 "어떤 Wallet이 이 NFT를 가졌는가"는 해당 Mint의 Token Account 중 Amount가 1인 것을 찾아야 알 수 있다.

### 3.4 전송 구조

일반 NFT의 전송은 SPL Token Program의 Transfer Instruction(명령어)으로 처리된다. Token Metadata Program은 관여하지 않는다.

Alice가 Bob에게 NFT를 보내는 경우는 다음과 같다.

1. Bob에게 해당 Mint의 Token Account가 없으면 먼저 만든다(보통 ATA). 생성 비용인 Rent는 Transaction의 Payer가 낸다.
2. Alice의 Token Account의 Amount가 1에서 0이 된다.
3. Bob의 Token Account의 Amount가 0에서 1이 된다.

Mint Account, Metadata Account, Master Edition Account는 변하지 않는다. 각 Token Account의 Owner 필드도 변하지 않는다. 소유권 이동은 "Amount 1이 어느 Token Account에 있는가"로 표현된다.

전송 후 Alice의 Token Account는 Amount 0인 상태로 남는다. Alice가 Account를 닫으면 Rent를 돌려받고 Account는 사라진다. 따라서 "소유한 적이 있는 Wallet 수만큼 Token Account가 존재한다"는 설명은 빈 Account를 닫지 않았을 때만 성립한다.

### 3.5 파생 형태

**Print Edition(복제본)**
Master Edition을 가진 NFT(Original NFT)에서 복제본(Print NFT)을 발행할 수 있다. Print NFT는 자신만의 Mint Account와 Metadata Account를 가지며, Master Edition Account 대신 Edition Account를 가진다. Edition Account는 Edition 번호와 부모 Master Edition을 기록한다. 두 Account는 같은 Seed를 쓰므로 하나의 NFT는 둘 중 하나만 가진다.

**Collection(컬렉션)**
Token Metadata에서 Collection은 그 자체가 하나의 NFT(Collection NFT)다. 일반 NFT와 Account 구조가 같고, Metadata의 `CollectionDetails` 필드가 설정되어 있다는 점만 다르다. 개별 NFT는 Metadata의 `Collection` 필드에 Collection NFT의 Mint 주소(`Key`)와 `Verified` 값을 가진다. `Collection` 필드는 누구나 임의로 채울 수 있으므로, Collection의 Authority가 Verify Instruction에 서명해야 `Verified`가 True가 된다. Wallet과 Marketplace(마켓플레이스)는 `Verified`가 True인지 확인해야 한다([Token Metadata Collections 문서](https://www.metaplex.com/docs/smart-contracts/token-metadata/collections)).

**Programmable NFT (pNFT)**
일반 NFT는 SPL Token Program으로 직접 전송할 수 있으므로 Token Metadata Program이 규칙을 강제할 수 없다. Metadata의 Seller Fee Basis Points는 참고용일 뿐이고, Royalty(로열티)를 지급하지 않는 Marketplace를 막을 수 없었다. pNFT는 이를 해결하기 위한 선택형 표준이다([pNFT 문서](https://www.metaplex.com/docs/smart-contracts/token-metadata/pnfts)).

- pNFT의 Token Account는 SPL Token Program 수준에서 항상 Frozen(동결) 상태다. 전송, 잠금, 소각은 반드시 Token Metadata Program을 거쳐야 한다. Program은 한 Instruction 안에서 동결 해제, 작업 수행, 재동결을 처리한다.
- Token Record Account가 추가된다. PDA Seed는 `["metadata", Program ID, Mint 주소, "token_record", Token Account 주소]`다([Delegates 문서](https://www.metaplex.com/docs/smart-contracts/token-metadata/delegates)). Token Account가 항상 동결 상태이므로, 실제 잠금 여부와 Delegate(위임) 정보는 Token Record가 기록한다.
- 허용 규칙은 Token Auth Rules Program이 관리하는 Rule Set Account에 정의하고, Metadata Account가 이를 참조한다. 예를 들어 Royalty를 지키는 Program만 전송을 수행하도록 허용 목록을 둘 수 있다.

## 4. Core

### 4.1 배경과 구조

Token Metadata는 NFT 하나에 4개 이상의 Account가 필요하고, 규칙을 강제하려면 pNFT처럼 구조가 더 복잡해진다. Core는 SPL Token에 의존하지 않고 NFT 전용으로 다시 설계한 표준이다. NFT 하나를 Asset Account 하나로 표현한다([Core Asset 문서](https://www.metaplex.com/docs/smart-contracts/core/what-is-an-asset)).

### 4.2 Core Asset Account

Asset Account의 주소가 NFT의 식별자다. 저장하는 필드와 크기는 [QuickNode의 Core 가이드](https://www.quicknode.com/guides/solana-development/nfts/metaplex-core)가 Program 소스에서 정리한 표를 따랐다.

| 필드 | 크기(Byte) | 내용 |
|---|---|---|
| `key` | 1 | Account 종류를 구분하는 Discriminator |
| `owner` | 32 | 현재 소유자의 Wallet 주소 |
| `update_authority` | 33 | 수정 권한자. Type이 `None`, `Address`, `Collection` 중 하나 |
| `name` | 4 + 길이 | 이름 |
| `uri` | 4 + 길이 | 오프체인 Metadata JSON의 위치 |
| `seq` | 1 + (선택) 8 | Compression Indexing에 쓰이는 Sequence 번호 |

`update_authority`의 Type은 두 가지 역할을 겸한다.

- `Address`: 지정된 주소가 수정 권한을 가진다. Collection에 속하지 않은 Asset이다.
- `Collection`: 값이 Collection Account의 주소다. 이 Asset이 해당 Collection에 속한다는 뜻이며, 수정 권한은 Collection의 Update Authority를 따른다.
- `None`: 수정 권한자가 없다. Asset을 수정할 수 없다.

Token Metadata와 달리 소유자 주소가 Asset Account 안에 직접 들어 있다. 그래서 Token Account와 ATA가 필요 없다.

### 4.3 Plugin(플러그인)

Plugin은 Asset 또는 Collection에 동작이나 데이터를 추가하는 확장 모듈이다([Plugins Overview](https://developers.metaplex.com/core/plugins)). Plugin 데이터는 별도 Account가 아니라 같은 Asset Account 안에서 기본 필드 뒤에 붙는다. 저장 순서는 기본 데이터, Plugin Header, 각 Plugin, Plugin Registry다. Plugin을 추가하면 Account 크기와 Rent가 늘어난다.

Plugin은 Create, Transfer, Burn, Update 같은 Lifecycle Event(생명 주기 이벤트)에 개입하여 동작을 승인하거나 거부할 수 있다. 관리 주체에 따라 세 종류로 나뉜다.

| 종류 | 추가 조건 | 예시 |
|---|---|---|
| Owner Managed | Owner의 서명이 필요 | Transfer Delegate, Freeze Delegate, Burn Delegate |
| Authority Managed | Update Authority가 언제든 추가 가능 | Royalties, Update Delegate, Attributes |
| Permanent | 생성 시점에만 추가 가능. 이후 제거 불가 | Permanent Freeze Delegate, Permanent Transfer Delegate, Permanent Burn Delegate |

이외에 외부 Account의 상태에 따라 Lifecycle Event를 거부할 수 있는 Oracle, 임의 데이터를 저장하는 AppData 같은 [External Plugin](https://www.metaplex.com/docs/smart-contracts/core/external-plugins/overview)이 있다.

Token Metadata의 기능이 Core에서 대응되는 방식은 [Core와 Token Metadata의 차이 문서](https://www.metaplex.com/docs/smart-contracts/core/tm-differences)에 정리되어 있다.

| Token Metadata | Core |
|---|---|
| Mint Account, Metadata Account | Asset Account 하나로 통합 |
| Associated Token Account | 필요 없음 |
| Freeze Authority | Freeze Delegate Plugin |
| Delegate | Transfer, Burn, Update Delegate Plugin |
| Creators 배열 | Verified Creators Plugin |
| Collection의 Verified 값 | Collection 소속 관계(자동) |

### 4.4 전송 구조

Alice가 Bob에게 Core Asset을 보내면 Asset Account의 `owner` 필드가 Alice의 주소에서 Bob의 주소로 바뀐다. 새로 만들어지는 Account는 없다.

전송 시 Owner Managed Plugin(Transfer, Freeze, Burn Delegate)에 위임되어 있던 권한은 자동으로 회수된다. Authority Managed Plugin과 Permanent Plugin은 그대로 유지된다.

### 4.5 온체인과 오프체인의 연결

```
[Core Asset Account]
  key
  owner = Alice
  update_authority = Collection(주소) ──▶ [Core Collection Account]
  name
  uri ──────────────▶ [JSON 파일 (오프체인)]
  (Plugin 데이터)            Image ──▶ [미디어 파일]
```

원본 구조 그림("On Chain Asset Account Structure")은 [Core Asset 문서](https://www.metaplex.com/docs/smart-contracts/core/what-is-an-asset)에 대화형으로 실려 있다.

### 4.6 Core Collection

Core의 Collection은 Asset과 구분되는 별도의 Account 종류다([Core Collections 문서](https://www.metaplex.com/docs/smart-contracts/core/collections)). Token Metadata에서는 Collection도 일반 NFT와 같은 구조여서 둘을 구분하기 어려웠다. Collection Account에는 `owner` 필드가 없다.

| 필드 | 내용 |
|---|---|
| `key` | Account 종류를 구분하는 Discriminator |
| `update_authority` | Collection과 소속 Asset에 대한 권한자 주소 |
| `name` | Collection 이름 |
| `uri` | Collection의 오프체인 Metadata 위치 |
| `num_minted` | 이 Collection에서 지금까지 만들어진 Asset의 누적 수. 이후 제거된 것도 포함 |
| `current_size` | 현재 Collection에 속한 Asset 수 |

Asset은 `update_authority`의 Type을 `Collection`으로 두어 Collection을 참조한다. 생성 이후에도 Update Instruction으로 Asset을 Collection에 추가, 이동, 제거할 수 있다. Collection에서 제거하면 Update Authority가 다시 Wallet 주소로 돌아간다. Asset이 남아 있는 Collection은 닫을 수 없다.

Collection에 붙인 Plugin은 소속된 모든 Asset에 기본으로 적용된다. Asset이 같은 종류의 Plugin을 직접 가지고 있으면 Asset의 Plugin이 우선한다. Collection에는 Authority Managed Plugin과 Permanent Plugin만 붙일 수 있다.

- 예시: Collection에 Royalties Plugin을 2%로 설정하면 전체 Asset에 2%가 적용된다. 특정 Asset에 5% Royalties Plugin을 따로 붙이면 그 Asset만 5%가 된다.
- 주의: Collection Account 하나를 수정해 전체에 반영되는 것은 Collection 수준 Plugin(Royalty, 동결 등)이다. 개별 Asset의 `name`과 `uri`는 각 Asset Account에 따로 저장되므로, Collection의 `name`이나 `uri`를 바꿔도 개별 Asset의 값은 바뀌지 않는다. (Account 구조에서 도출한 내용이다.)

## 5. Compressed NFT (cNFT)

### 5.1 배경

Token Metadata와 Core는 NFT마다 Account를 만들기 때문에 발행 수에 비례해 Rent가 든다. 수백만 개를 발행하는 용도에는 부담이 크다. cNFT(Compressed NFT)는 NFT마다 Account를 만들지 않는다. NFT 데이터의 Hash만 Merkle Tree에 넣고, 원본 데이터는 Transaction 기록에만 남긴다. 이를 담당하는 Metaplex Program이 [Bubblegum](https://www.metaplex.com/docs/smart-contracts/bubblegum-v2)이다.

### 5.2 Merkle Tree 기본 개념

Merkle Tree는 부모 Node가 두 자식 Node의 Hash인 이진 Tree다.

- Leaf(리프): 맨 아래 Node. cNFT 하나의 데이터를 Hash한 값이다.
- Root(루트): 맨 위의 Hash 하나. Tree 전체의 무결성을 대표한다.
- Proof(증명): 특정 Leaf에서 Root까지 다시 계산하는 데 필요한 형제 Node들의 Hash 목록이다.

```
                 [Root]
               /        \
          [H01]          [H23]
         /     \        /     \
     [H0]     [H1]  [H2]     [H3]      H0 = Hash(Leaf Schema 0)
      │        │     │        │
   cNFT 0   cNFT 1  cNFT 2  cNFT 3     cNFT 0의 Proof = [H1, H23]
```

검증은 Leaf Hash에 Proof의 Hash들을 차례로 결합해 올라간 결과가 온체인의 Root와 같은지 비교하는 방식이다. Leaf가 바뀌면(전송 등) Root도 새로 계산되어 온체인에 기록된다. Leaf Path와 Proof를 그린 원본 그림은 [Concurrent Merkle Trees 문서](https://www.metaplex.com/docs/smart-contracts/bubblegum-v2/concurrent-merkle-trees)에 있다.

### 5.3 온체인 Account 구성

cNFT 자체의 Account는 없다. 대신 Tree 하나당 다음 2개의 Account가 있다. Metaplex는 이 둘을 묶어 Bubblegum Tree라고 부른다([Tree 생성 문서](https://www.metaplex.com/docs/ko/smart-contracts/bubblegum-v2/create-trees)).

**Merkle Tree Account**
Account Compression Program이 소유한다. Bubblegum V1은 SPL Account Compression Program을, V2는 이를 Fork한 MPL Account Compression Program을 사용한다. 저장 내용은 다음과 같다.

- Header: Max Depth, Max Buffer Size, Authority, 생성 Slot
- Tree 본체: 현재 Root, ChangeLog Buffer, Sequence 번호, Rightmost Proof
- Canopy(캐노피): Tree 상위 Node들의 Cache

Root만 저장하는 것이 아니라는 점에 주의한다. 각 요소의 역할은 다음과 같다.

- ChangeLog Buffer: 최근의 Root와 변경된 경로를 보관한다. 같은 Block에서 여러 변경이 일어나면 먼저 처리된 변경이 다른 Proof를 무효로 만든다. Program은 ChangeLog를 이용해 오래된 Proof를 최신 상태로 보정(Fast-Forward)한다. 보관 개수는 Max Buffer Size로 정해지고, 이를 넘는 동시 변경은 실패하여 재시도해야 한다. 이런 구조를 Concurrent Merkle Tree라고 한다.
- Rightmost Proof: 가장 오른쪽 Leaf의 Proof다. Mint는 항상 가장 오른쪽에 Leaf를 추가하므로, 이 값이 온체인에 있으면 Mint 시 Proof를 보낼 필요가 없다.
- Canopy: Tree의 상위 n개 Level을 온체인에 Cache한다. 깊이 d인 Tree에서 Transaction에 실어야 하는 Proof가 d개에서 d - n개로 줄어든다([Canopy 문서](https://www.metaplex.com/docs/smart-contracts/bubblegum-v2/merkle-tree-canopy)).

**TreeConfig Account**
Bubblegum Program이 소유하는 PDA이며 Merkle Tree Account의 주소에서 유도된다. cNFT 전용 설정을 저장한다.

- `tree_creator`: Tree 생성자
- `tree_delegate`: Mint 권한을 위임받은 주소. 없으면 생성자와 같다
- `total_mint_capacity`: Mint 가능한 최대 개수
- `num_minted`: 지금까지 Mint된 개수. Leaf의 고유성을 보장하는 Nonce로도 쓰인다
- `is_public`: True이면 누구나 Mint할 수 있다
- `is_decompressible`: Decompress 허용 여부. Bubblegum V1에서만 유효하다
- Version: 사용할 Leaf Schema 버전 (V2에서 추가)

### 5.4 Leaf의 구성

Leaf는 Leaf Schema를 Keccak-256으로 Hash한 32 Byte 값이다([Hashed NFT Data 문서](https://www.metaplex.com/docs/smart-contracts/bubblegum-v2/hashed-nft-data)).

| 필드 | 내용 | 버전 |
|---|---|---|
| `id` | Asset ID. 고정 Prefix, Merkle Tree 주소, Nonce로 유도한 PDA. cNFT의 식별자 | V1, V2 |
| `owner` | 소유자 주소 | V1, V2 |
| `delegate` | 위임받은 주소. 기본값은 소유자 | V1, V2 |
| `nonce` | Tree 안에서의 고유 번호 | V1, V2 |
| `data_hash` | Metadata(이름, URI 등)와 Seller Fee Basis Points의 Hash | V1, V2 |
| `creator_hash` | Creators 배열의 Hash | V1, V2 |
| `collection_hash` | Collection 정보의 Hash | V2 |
| `asset_data_hash` | 추가 Asset 데이터의 Hash | V2 |
| `flags` | 상태 Flag | V2 |

Asset ID는 PDA 형식의 주소지만 그 주소에 실제 Account가 존재하지는 않는다. 식별자로만 쓰인다.

### 5.5 원본 데이터가 저장되는 곳

Leaf에는 Hash만 있으므로, Hash의 원본인 Metadata와 소유자 정보는 다른 곳에서 얻어야 한다. 저장과 Indexing 구조는 [Stored NFT Data 문서](https://www.metaplex.com/docs/smart-contracts/bubblegum-v2/stored-nft-data)를 따랐다.

**Transaction (Ledger)**
cNFT의 전체 데이터는 Mint Transaction에 들어 있다. 이후의 변경도 Transaction에 기록된다. Bubblegum과 Account Compression Program은 Event를 일반 Program Log로 남기지 않고, Noop(No Operation) Program을 CPI(Cross-Program Invocation, Program 간 호출)로 호출하면서 그 Instruction Data에 Event를 싣는다. Program Log는 길이 제한으로 잘릴 수 있는데, Instruction Data는 잘리지 않고 Ledger에 남기 때문이다. 흔히 "Transaction Log에 저장된다"고 표현하지만 정확히는 Noop Instruction의 Data다. (Metaplex 문서는 이 Program을 spl-noop으로 표기한다.)

**DAS API Indexer(인덱서)**
매번 과거 Transaction을 뒤지는 것은 비현실적이므로, RPC(Remote Procedure Call) 제공자가 이 데이터를 실시간으로 Indexing해서 제공한다. 이 조회 인터페이스가 Metaplex DAS(Digital Asset Standard) API다. Metaplex의 Reference 구현은 다음 순서로 동작한다.

1. Validator에서 실행되는 Geyser Plugin(Plerkle)이 Bubblegum과 Account Compression 관련 Transaction을 수집한다.
2. 수집한 데이터를 Redis Stream에 넣는다.
3. Ingester가 이를 Parsing해 Postgres 데이터베이스에 저장한다. Asset 정보와 함께 Merkle Tree의 변경 이력도 저장하며, 이 이력으로 Proof를 만들어 준다.
4. API Process가 사용자 요청에 응답한다.

DAS API는 Solana Protocol의 일부가 아니다. RPC 제공자가 운영하는 Indexing 계층이며, 모든 RPC가 지원하지는 않는다.

**오프체인 Metadata**
다른 표준과 같다. URI가 가리키는 JSON 파일과 미디어 파일이 Arweave, IPFS, 중앙화 서버 등에 저장된다.

### 5.6 온체인과 오프체인의 연결

```
[온체인 Account]                          [Ledger]                       [오프체인]
Merkle Tree Account                       Mint, Transfer Transaction      DAS API Indexer
  Root, ChangeLog, Canopy                   Noop Instruction Data           Leaf, Proof, Metadata 저장
  Leaf = Hash(Leaf Schema)   ◀─ 검증 ─      (Leaf Schema, Metadata) ──▶    │
TreeConfig Account (PDA)                                                    ▼
  tree_creator, num_minted 등                                         사용자 조회 (getAsset, getAssetProof)
                                                                            │ URI
                                                                            ▼
                                                                      JSON 파일 ──▶ 미디어 파일
```

원본 그림은 [Bubblegum V2 개요](https://www.metaplex.com/docs/smart-contracts/bubblegum-v2)의 Metaplex DAS API 절에 대화형으로 실려 있다.

### 5.7 Mint와 전송의 동작

**Mint**

1. Bubblegum의 Mint Instruction에 Metadata(이름, URI, Creators 등)를 인자로 넣어 호출한다.
2. Bubblegum이 Leaf Schema를 만들고 Hash한다.
3. Account Compression Program이 Tree의 가장 오른쪽에 Leaf를 추가하고 Root를 갱신한다. Proof는 필요 없다.
4. Leaf Schema와 변경 내용이 Noop Instruction Data로 Ledger에 남는다.
5. Indexer가 이를 읽어 데이터베이스에 반영한다.

**전송 (Alice에서 Bob으로)**

1. Client가 DAS API의 `getAsset`, `getAssetProof`로 현재 Leaf 정보(Root, Data Hash, Creator Hash, Nonce, Index)와 Proof를 받는다([Bubblegum V2 FAQ](https://www.metaplex.com/docs/smart-contracts/bubblegum-v2/faq)).
2. Transfer Instruction에 이 값과 Proof를 넣어 보낸다. Alice가 서명한다.
3. Program이 전달받은 값으로 기존 Leaf Hash를 다시 계산하고, Proof와 결합해 온체인 Root와 일치하는지 검증한다. Proof가 조금 오래되었으면 ChangeLog Buffer로 보정한다.
4. `owner`를 Bob으로 바꾼 새 Leaf로 교체하고 Root를 갱신한다.

전송에서 바뀌는 온체인 데이터는 Merkle Tree Account의 Root(와 ChangeLog, Canopy)뿐이다. Bob을 위한 Account는 만들어지지 않는다.

### 5.8 Tree 파라미터와 비용

Tree를 만들 때 세 값을 정한다. 세 값 모두 생성 후에는 바꿀 수 없다.

- Max Depth: 최대 Leaf 수가 2^Max Depth로 정해진다.
- Max Buffer Size: ChangeLog 보관 개수. 동시 변경 허용량을 결정한다.
- Canopy Depth: 온체인에 Cache할 상위 Level 수. 추가로 필요한 크기는 (2^(n+1) - 1) × 32 Byte다.

Max Depth와 Max Buffer Size는 임의의 조합이 아니라 미리 정의된 조합 중에서 골라야 한다.

Solana Transaction의 데이터 크기는 1,232 Byte로 제한된다. Proof Node 하나가 32 Byte이므로 Proof가 길면 Transaction에 다른 Instruction을 함께 넣기 어렵다. 깊이 24를 넘는 Tree는 Canopy가 반드시 필요하다. Canopy가 클수록 Proof가 줄어 다른 Program과 조합하기 쉬워지지만 Tree의 Rent가 커진다([Canopy 문서](https://www.metaplex.com/docs/smart-contracts/bubblegum-v2/merkle-tree-canopy)).

[Tree 생성 문서](https://www.metaplex.com/docs/ko/smart-contracts/bubblegum-v2/create-trees)의 권장 설정 일부는 다음과 같다. 비용 단위는 SOL이다.

| cNFT 수 | Max Depth | Canopy Depth | Max Buffer Size | Tree 비용 | cNFT당 비용 |
|---|---|---|---|---|---|
| 16,384 | 14 | 8 | 64 | 0.3358 | 0.00002550 |
| 1,048,576 | 20 | 13 | 1024 | 8.5012 | 0.00001311 |
| 1,073,741,824 | 30 | 17 | 2048 | 72.6468 | 0.00000507 |

Tree 비용은 Tree Account를 만들 때 한 번 내는 Rent다. 표의 "cNFT당 비용"은 Tree 비용을 개수로 나눈 값보다 약 0.000005 SOL 크다. 기본 Transaction 수수료가 더해진 값으로 보이지만, 문서에 계산식이 명시되어 있지 않으므로 이 부분은 추정이다.

### 5.9 Bubblegum V1과 V2

차이는 [Bubblegum V2 FAQ](https://www.metaplex.com/docs/smart-contracts/bubblegum-v2/faq)와 [V1 Decompress 문서](https://www.metaplex.com/docs/smart-contracts/bubblegum/decompress-cnfts), [Protocol Fees 문서](https://www.metaplex.com/docs/protocol-fees)를 따랐다.

| 항목 | V1 | V2 |
|---|---|---|
| Leaf Schema | LeafSchemaV1 | LeafSchemaV2 |
| Merkle Tree 소유 Program | SPL Account Compression | MPL Account Compression |
| Collection | Token Metadata의 Collection NFT | MPL Core Collection (BubblegumV2 Plugin 필요) |
| Decompress(일반 NFT로 변환) | 가능. Token Metadata NFT로 변환되며 Mint, Metadata, Master Edition Account의 Rent가 든다 | 불가능 |
| 동결, Soulbound(양도 불가), Royalty 강제 | 없음 | Core Collection의 Plugin으로 지원 |
| Protocol Fee | Create 무료 | Create 0.00009 SOL, Transfer 0.000006 SOL |

V1 Tree와 V2 Tree는 서로 호환되지 않는다. V2의 Leaf는 `createTreeV2`로 만든 Tree에만 넣을 수 있다.

## 6. 세 표준 비교

| 항목 | Token Metadata | Core | Compressed NFT |
|---|---|---|---|
| NFT당 온체인 Account | 4개 (Mint, Token, Metadata, Master Edition). pNFT는 5개 | 1개 (Asset) | 0개. Tree당 2개 (Merkle Tree, TreeConfig) |
| NFT 식별자 | Mint Account 주소 | Asset Account 주소 | Asset ID (Account가 없는 PDA 주소) |
| 소유자가 기록되는 곳 | Token Account (Amount 1의 위치) | Asset Account의 `owner` 필드 | Leaf Schema의 `owner` (온체인에는 Hash만 존재) |
| 전송 시 변경 | 두 Token Account의 Amount. 수신자 Token Account 생성이 필요할 수 있음 | Asset Account의 `owner` | Merkle Tree Account의 Root |
| 전송을 처리하는 Program | SPL Token Program (pNFT는 Token Metadata Program) | MPL Core Program | Bubblegum Program |
| 조회 방법 | 일반 RPC로 Account 조회 | 일반 RPC로 Account 조회. DAS API도 지원 | DAS API 필요 |
| Collection | Collection NFT + Verified 값 | 전용 Collection Account | V1은 Token Metadata Collection, V2는 Core Collection |
| Royalty 강제 | 일반 NFT는 불가. pNFT는 Rule Set으로 가능 | Royalties Plugin | V2에서 Core Collection Plugin으로 가능 |
| Mint 비용 (Metaplex 문서 기준) | 약 0.022 SOL | 약 0.003 SOL | Tree Rent를 나눈 값. 100만 개 Tree 기준 개당 약 0.00001 SOL |
| Protocol Fee (Create) | 0.01 SOL | 0.0015 SOL | V1 무료, V2 0.00009 SOL |

비용에 관한 참고 사항은 다음과 같다. Mint 비용은 [Core와 Token Metadata의 차이 문서](https://www.metaplex.com/docs/smart-contracts/core/tm-differences)와 [Bubblegum V2 개요](https://www.metaplex.com/docs/smart-contracts/bubblegum-v2)의 값이고, Protocol Fee는 [Protocol Fees 문서](https://www.metaplex.com/docs/protocol-fees)의 값이다.

- Core의 Mint 비용 약 0.003 SOL의 구성은 [Core Asset 생성 문서](https://www.metaplex.com/docs/smart-contracts/core/create-asset)에 나온다. Asset Account Rent 약 0.0015 SOL, Create Protocol Fee 0.0015 SOL, Transaction 수수료 약 0.000005 SOL의 합이다. Rent는 Name, URI 길이와 Plugin 수에 따라 달라진다. 즉 이 수치는 Protocol Fee를 포함한 값이다.
- Metaplex 문서의 다른 위치에는 약 0.0029 SOL(비교표), 약 0.0037 SOL(2024년 출시 당시 보도 수치)로도 적혀 있다. 이 두 수치는 구성 요소가 명시되어 있지 않다. 공통적으로 Token Metadata 대비 80% 이상 저렴하다고 설명한다.
- Token Metadata의 약 0.022 SOL에 Protocol Fee 0.01 SOL이 포함되는지는 문서에 명시되어 있지 않다.
- cNFT의 개당 비용에는 V2의 Protocol Fee(Create 0.00009 SOL)가 포함되어 있지 않다. V2에서는 Protocol Fee가 Tree Rent의 개당 몫보다 크다.

## 7. 예시: 같은 전송을 세 표준으로 비교

Alice가 Bob에게 NFT 하나를 보낸다. Bob은 이 NFT를 처음 받는다.

**Token Metadata**
- 필요한 것: Alice의 서명, Bob의 Token Account
- 처리: Bob의 ATA 생성(Rent 발생), Alice의 Amount 1에서 0, Bob의 Amount 0에서 1
- 결과: Token Account가 2개 존재한다. Alice가 빈 Account를 닫으면 1개가 된다.

**Core**
- 필요한 것: Alice의 서명
- 처리: Asset Account의 `owner`가 Alice에서 Bob으로 변경
- 결과: Account 수 변화 없음. Owner Managed Plugin의 위임 권한은 회수된다.

**Compressed NFT**
- 필요한 것: Alice의 서명, DAS API에서 받은 Leaf 정보와 Proof
- 처리: 기존 Leaf 검증 후 `owner`가 Bob인 새 Leaf로 교체, Root 갱신
- 결과: Account 수 변화 없음. Indexer가 Transaction을 읽어 소유자를 Bob으로 갱신한다.

## 8. 장단점

**Token Metadata (2021년)**

- 장점: 가장 오래 쓰인 표준이라 Wallet, Marketplace, 도구 지원이 가장 넓다. SPL Token 기반이므로 SPL Token을 다루는 도구와 호환된다.
- 단점: NFT당 Account가 4개 이상이라 구조가 복잡하고 비용이 크다. 대량 발행 시 비용이 발행 수에 비례해 커진다. 일반 NFT는 Royalty를 강제할 수 없고, 이를 보완한 pNFT는 Account와 규칙 구조가 더 복잡하다. Metaplex가 Legacy로 분류한다.

**Core (2024년)**

- 장점: Account가 1개라 비용이 낮다. Plugin으로 동결, 위임, Royalty 강제, 속성 저장 등을 필요한 만큼만 붙일 수 있다. Collection이 전용 Account라 Collection 단위 설정이 가능하다.
- 단점: Token Metadata NFT와 온체인 구조가 다르다. SPL Token이 아니므로 Token Account를 전제로 한 도구는 그대로 쓸 수 없다. 기존 Token Metadata NFT를 Core로 옮기려면 기존 NFT를 Burn하고 Core Asset을 새로 Mint해야 한다.
- 지원 현황: [Ecosystem Support 문서](https://www.metaplex.com/docs/smart-contracts/core/ecosystem-support)(2026-01-31 갱신)는 Marketplace로 Tensor, Magic Eden, Sniper, OKX, Mallow를, Wallet으로 Solflare, Phantom, Backpack을, Explorer로 SolanaFM, Solscan을 Complete로 표기한다.

**Compressed NFT**

- 장점: NFT마다 Account를 만들지 않으므로 개당 비용이 매우 낮다. Metaplex 권장 설정 표 기준으로 Tree 하나에 2^30개(약 10억 개)까지 담을 수 있어 대규모 발행에 적합하다.
- 단점:
  - Indexer 의존: 원본 데이터는 Ledger에 있으므로 이론적으로는 과거 Transaction을 재생해 복원할 수 있다. 그러나 실제 조회와 Proof 획득은 DAS API를 지원하는 RPC에 의존한다.
  - Proof 필요: 전송, 위임, 소각 같은 변경마다 Proof를 실어야 한다. Transaction 크기를 차지해 다른 Program과의 조합이 제한될 수 있다.
  - Tree 파라미터 고정: Max Depth, Max Buffer Size, Canopy Depth는 생성 후 변경할 수 없다. 용량이 부족하면 새 Tree를 만들어야 한다.
  - 동시성 한계: 한 Block 안의 변경이 Max Buffer Size를 넘으면 일부 Transaction이 실패한다.
  - 지원 범위: 일반 NFT보다 좁다. 아래 9절 참조.

## 9. 지원 현황과 확인이 필요한 내용

**Bubblegum V1 cNFT의 Wallet과 Marketplace 지원**

| 대상 | 표시 | 전송 | 근거 |
|---|---|---|---|
| Solflare | 지원 | 지원 | [Solflare 용어집](https://www.solflare.com/glossary/compressed-nft/)이 cNFT를 "일반 NFT처럼 판매와 전송이 가능하다"고 설명 |
| Phantom | 지원 | 미확인 | [Phantom 2023년 연간 보고](https://phantom.com/learn/blog/phantom-2023-year-in-review)가 cNFT를 "First class support"로 표기. 전송 기능은 명시하지 않음 |
| Backpack | 지원 | 미확인 | [Backpack 도움말](https://support.backpack.exchange/wallet/troubleshoot/hide-spam-nfts)이 cNFT를 다루고 있음. 전송 기능은 명시하지 않음 |
| Trust Wallet | 미확인 | 미확인 | Solana NFT 지원은 확인되나 cNFT를 명시한 공식 자료를 찾지 못함 |
| MetaMask | 미확인 | 미확인 | [2025년 7월 Solana 지원 발표](https://metamask.com/news/solana-on-metamask-sol-wallet)는 SOL과 SPL Token만 언급. cNFT 언급 없음 |
| Magic Eden, Sniper | 거래 지원 | | [Metaplex 2023년 9월 월간 보고](https://www.metaplex.com/posts/metaplex-september-round-up)가 두 Marketplace의 cNFT 거래 지원을 기록 |

"미확인"은 지원하지 않는다는 뜻이 아니라 공식 자료로 확인하지 못했다는 뜻이다.

**Bubblegum V2 cNFT의 지원**

[Bubblegum V2 개요](https://www.metaplex.com/docs/smart-contracts/bubblegum-v2)의 Ecosystem Compatibility 절(2026-06-19 기준)은 Solflare, Phantom, Backpack이 V2 cNFT의 조회와 표시는 지원하지만 전송과 거래는 아직 지원하지 않으며, Magic Eden과 Tensor는 조회와 거래 모두 지원하지 않는다고 적고 있다. 같은 문서는 넓은 지원이 당장 필요하면 Core를 권장한다. 이 내용은 빠르게 바뀔 수 있으므로 사용 시점에 다시 확인해야 한다.

**확인이 필요한 내용**

- Bubblegum V2의 정확한 Mainnet 배포일: 월간 보고의 "5월 말"만 확인했고 일 단위 날짜는 찾지 못했다.
- Metaplex 외의 방식: Solana의 Token Extensions Program(Token-2022)은 TokenMetadata Extension으로 Name, Symbol, URI와 사용자 정의 Key-Value를 Mint Account에 직접 저장할 수 있다([Solana Token Extensions Metadata 문서](https://solana.com/docs/tokens/extensions/metadata.md)). 이 경우 별도의 Metadata Account가 필요 없다. 이 글의 범위는 Metaplex 표준이므로 자세히 다루지 않는다.