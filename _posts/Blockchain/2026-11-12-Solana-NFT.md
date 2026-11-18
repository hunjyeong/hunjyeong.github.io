---
title: "Solana NFT"
date: 2026-11-12 02:00:00 +0900
last_modified_at: 2026-11-18
categories: [Blockchain]
tags: [Blockchain, NFT]
media_subpath: /assets/img/posts/solana-nft/
---

>
> 1. [NFT]({% post_url Blockchain/2026-11-12-NFT %})
> 2. [Ethereum NFT]({% post_url Blockchain/2026-11-12-Ethereum-NFT %})
> 3. **Solana NFT**
>

## Solana Core Concepts

![solana-nft-core-concepts-light.svg](solana-nft-core-concepts-light.svg)

### Accounts and Programs

Solana는 모든 상태를 [Account](https://solana.com/docs/core/accounts)에 저장하고, 각 Account는 32바이트 Address로 구분된다. Account의 `owner` 필드에는 그 Account를 소유한 Program이 기록되며, Account의 데이터는 이 Program만 바꿀 수 있다. [Program](https://solana.com/docs/core/programs)(다른 Blockchain의 Smart Contract에 해당)도 실행 코드를 담은 Account다. Program은 상태를 갖지 않고, 상태는 별도의 Data Account에 둔다. 따라서 하나의 자산도 저장하는 데이터에 따라 여러 Account로 나뉜다.

사용자 Account의 [Address](https://solana.com/docs/core/accounts/account-structure#account-address)(주소)는 Public Key(공개키)이고, 그 사용자는 짝이 되는 Private Key(개인키)로 서명한다. 이 사용자는 Account의 `owner` 필드에 기록된 Program과 다르다. Private Key가 없는 Address인 PDA도 있다(아래 설명). Private Key를 보관하고 서명하는 프로그램을 Wallet(지갑)이라고 하며, 이 글에서는 사용자의 Account Address를 Wallet Address라고도 쓴다.

Account를 바꾸려면 [Transaction](https://solana.com/docs/core/transactions)을 보낸다. Transaction에는 하나 이상의 [Instruction](https://solana.com/docs/core/instructions)(명령어)과 변경을 승인하는 서명이 들어간다. Instruction은 호출할 Program, 사용할 Account 목록, 데이터를 지정한다. Instruction이 하나라도 실패하면 Transaction 전체가 취소된다.

Account는 데이터 크기에 비례하는 최소 Balance(잔액)를 넣어 두어야 유지된다. 이 값은 [Rent-Exempt Minimum](https://solana.com/docs/core/accounts/account-structure#:~:text=The%20runtime%20and%20RPC%20method%20names%20call%20this%20the%20%22rent%2Dexempt%20minimum%2C%22%20but%20ongoing%20rent%20collection%20is%20disabled.%20Treat%20the%20balance%20as%20a%20refundable%20account%20storage%20deposit%3A%20the%20full%20balance%20can%20be%20recovered%20when%20the%20account%20is%20closed.)이라고 불리지만 주기적으로 차감되지 않으며, Account를 닫으면 돌려받는 저장 예치금이다. 그래서 Account가 많고 클수록 NFT 하나를 만드는 비용이 커진다.

이 글에서 On-Chain(온체인)은 Blockchain 안에, Off-Chain(오프체인)은 Blockchain 밖에 있다는 뜻이다.

### SPL Token

[SPL(Solana Program Library) Token](https://solana.com/docs/tokens)은 Solana의 Token을 부르는 이름이다. Fungible Token과 NFT 모두 Token Program으로 만들고 다룬다.

![solana-nft-spl-token.svg](solana-nft-spl-token.svg)

**Token Programs**

[Token Program](https://solana.com/docs/tokens#token-programs)은 Token을 Mint하고 Transfer하는 등 Token에 대한 Instruction 로직을 모두 담은 Program이다. 기존 Token Program과, 기존 기능을 모두 포함하면서 Extension(확장 기능)을 더한 Token Extension Program(Token-2022) 두 가지가 있다. Solana의 Token은 모두 Token Program이 소유한 Data Account다.

![solana-nft-token-program-light.svg](solana-nft-token-program-light.svg)

**Mint Account**

[Mint Account](https://solana.com/docs/tokens#mint-account)는 Token 종류 하나를 나타내는 Account로, Token Program이 소유한다. Mint(발행)는 Token을 새로 만드는 동작, Transfer(전송)는 Token을 한 Token Account에서 다른 Token Account로 옮기는 동작, Burn(소각)은 Token을 없애는 동작, Freeze(동결)는 Token Account를 묶어 그 안의 Token을 Transfer하거나 Burn하지 못하게 하는 동작이다. Mint Account는 Supply(현재 총 공급량, Mint하면 늘고 Burn하면 줄어든다), Decimals(소수점 아래 자릿수), Mint Authority(새 Token을 Mint해 Supply를 늘릴 수 있는 Account), Freeze Authority(Token Account를 Freeze할 수 있는 Account)를 저장한다. Mint Authority는 Mint Account를 초기화할 때 정하며, Mint Authority가 없으면 Supply가 고정되어 더 이상 Mint할 수 없다. Token은 Mint Account의 Address로 고유하게 식별된다.

![solana-nft-mint-account-light.svg](solana-nft-mint-account-light.svg)

**Token Account**

[Token Account](https://solana.com/docs/tokens#token-account)는 특정 Mint의 Token을 특정 Owner가 얼마나 가졌는지 저장하는 Account로, Token Program이 소유한다. 주요 필드는 Mint(이 Token Account가 담는 Token), Owner(소유자, 이 Token Account의 Token을 Transfer·Burn하거나 Delegate(위임)할 수 있는 Account), Amount(현재 보유 수량)다. 여기서 Owner는 Token Account의 `data` 안에 기록된 값으로, Account 구조의 `owner` 필드(Token Program)와 다르다.

![solana-nft-token-account-light.svg](solana-nft-token-account-light.svg)

Wallet은 보유하려는 Mint마다 자신을 Owner로 지정한 Token Account를 가진다. Token Account 하나는 Mint 하나와 Owner 하나에만 연결되지만, Wallet은 같은 Mint의 Token Account를 여러 개 가질 수 있다.

![solana-nft-token-account-relationship-light.svg](solana-nft-token-account-relationship-light.svg)

**Associated Token Account (ATA)**

[Associated Token Account (ATA)](https://solana.com/docs/tokens#associated-token-account)는 특정 Wallet과 Mint의 기본 Token Account로, Token Account Address를 따로 기록해 두지 않아도 찾을 수 있게 해 준다. Associated Token Program이 Wallet Address, Token Program Address, Mint Address를 Seed(주소를 유도할 때 넣는 입력값)로 하여 Address를 유도하므로, 이 조합마다 ATA Address는 정확히 하나다. Associated Token Program은 이 Address에 Account를 만들어 줄 뿐이며, ATA도 일반 Token Account다. ATA의 최소 Balance를 내는 Payer는 ATA의 Owner와 달라도 되므로, 받는 사람 대신 ATA를 만들어 줄 수 있다.

![solana-nft-associated-token-account-light.svg](solana-nft-associated-token-account-light.svg)

### Program Derived Addresses (PDAs)

[Program Derived Address (PDA)](https://solana.com/docs/core/pda)는 ATA처럼 정해진 입력으로부터 결정적으로 유도되는 32바이트 Address다. Program ID, 선택 입력인 Seed(최대 16개, 각 32바이트 이하), Bump(1바이트 값), 고정 문자열 "ProgramDerivedAddress"를 SHA-256으로 함께 해시해 만든다. 결과가 Ed25519 곡선 위에 있으면 Private Key가 존재할 수 있으므로, 곡선 밖의 Address가 나올 때까지 Bump를 255부터 1씩 줄여 가며 다시 계산한다. 처음 성공한 Bump를 Canonical Bump라고 한다. 같은 Seed와 Program ID로는 항상 같은 Address가 나온다.

![solana-nft-pda-light.svg](solana-nft-pda-light.svg)

PDA는 곡선 밖에 있어 Private Key가 없으므로 누구도 서명을 만들 수 없다. 대신 유도에 쓰인 Program만 CPI(Cross-Program Invocation, 다른 Program 호출) 중 `invoke_signed`로 PDA를 대신해 서명할 수 있다. PDA를 유도하는 것만으로 Account가 생기지는 않으며, Program이 `invoke_signed`로 System Program의 `create_account`를 호출해 그 Address에 Account를 따로 만들어야 한다. 이후에 나오는 Metadata Account, Master Edition Account, TreeConfig Account가 모두 PDA다.

<br />

## Metaplex

[Metaplex](https://www.metaplex.com/)는 Solana 생태계에서 디지털 자산을 발행하고 관리하는 사실상의 표준 프로토콜이다. Metaplex는 프로젝트의 목적과 비용 효율성에 따라 선택할 수 있는 세 가지 NFT 표준을 제공하며, 이들은 NFT의 상태(소유권 및 메타데이터)를 저장하는 방식에서 결정적인 차이를 보인다.

| 표준 | 출시 | 담당 Program | NFT 상태를 저장하는 곳 | NFT 식별자 | 발행 비용(Metaplex 문서 기준) |
|---|---|---|---|---|---|
| Token Metadata | [2021년 (Metaplex 출시: 8월)](https://whitepaper.metaplex.foundation/whitepaper.pdf) | Token Metadata Program, SPL Token Program | 여러 Account: Mint Account, Owner의 Token Account(보통 ATA), Mint Account에서 유도한 PDA(Metadata Account, Master Edition Account) | Mint Account Address | 약 0.022 SOL |
| Core | [2024년 3월 25일 Devnet, 4월 1일 Mainnet](https://developers.metaplex.com/core/changelog)(메인넷, 실제 운영 네트워크) | MPL(Metaplex Program Library) Core Program | Asset Account 하나(소유권, Metadata URI, Plugin 데이터) | Asset Account Address | 약 0.003 SOL |
| Compressed NFT | Bubblegum V1: [2022년 11월 발표](https://www.metaplex.com/blog/articles/expanding-digital-assets-with-compression-for-nfts), [2023년 4월 Mainnet 가동 공표](https://solana.com/ko/news/how-to-use-compressed-nfts-on-solana)<br>Bubblegum V2: [2025년 5월 29일 발표](https://www.metaplex.com/blog/articles/metaplex-foundation-announces-the-next-generation-of-compressed-nfts) | MPL Bubblegum Program, Account Compression Program(V1: SPL Account Compression, V2: MPL Account Compression) | NFT별 Account 없음. NFT 데이터를 Hash한 Leaf로 Merkle Tree를 만들고, On-Chain Merkle Tree Account에는 Root 등 일부 Hash만 저장한다. Tree 변경 내역은 Transaction 기록(Ledger)에 남고, RPC Indexer가 이를 모아 NFT 데이터와 Proof를 제공한다 | Asset ID(Merkle Tree Address와 Leaf Index로 유도한 PDA, 이 Address에 Account는 없음) | 약 0.00001 SOL(큰 Tree 기준) |

<br />

## Token Metadata

### What is a Token Metadata?

Token Metadata Program의 목적은 Solana의 Fungible Token 또는 Non-Fungible Token에 추가 데이터를 붙이는 것이다. NFT 전용 Program이 아니며, NFT는 가장 큰 사용처다.

Token Metadata Program은 Mint Account의 주소에서 유도한 Program Derived Addresses (PDAs)에 Metadata Account를 만들어 Mint Account에 덧붙인다. Metadata는 Token이 무엇을 나타내는지 설명하는 정보다. Metadata Account의 URI는 Off-Chain의 JSON(JavaScript Object Notation) 파일을 가리킨다.

![solana-nft-02-tm-json-standard.png](solana-nft-02-tm-json-standard.png)

**Token Standard**

[`Token Standard`](https://www.metaplex.com/docs/smart-contracts/token-metadata/token-standard#the-token-standard-field)는 Token의 종류를 나타내는 Metadata Account의 필드다. Token Metadata Program이 자동으로 정하며 직접 수정할 수 없다.

| Standard | Description |
|---|---|
| `NonFungible` | A non-fungible token with a Master Edition |
| `FungibleAsset` | A token with metadata that can also have attributes, sometimes called Semi-Fungible |
| `Fungible` | A token with simple metadata |
| `NonFungibleEdition` | A non-fungible token with an Edition account (printed from a Master edition) |
| `ProgrammableNonFungible` | A special NonFungible token that is frozen at all times to enforce custom authorization rules |

이 글에서 일반 NFT는 `NonFungible`을 말한다. `NonFungibleEdition`과 `ProgrammableNonFungible`은 아래 "확장"에서 설명한다.

**장점**
- 2021년부터 쓰인 표준이라 대부분의 Wallet, Marketplace, 도구가 지원한다.
- NFT가 SPL Token이므로 SPL Token을 다루는 Wallet과 Program에서 그대로 다룰 수 있다.
- 하나의 Program으로 NFT뿐 아니라 Fungible Token, Semi-Fungible Token(Fungible Asset), Print Edition에도 같은 방식으로 Metadata를 붙일 수 있다.

**단점**
- NFT 하나에 Mint, Token, Metadata, Master Edition Account가 필요해 구조가 복잡하고, Mint 비용이 약 0.022 SOL로 Core(약 0.003 SOL)보다 크다.
- 받는 Wallet에 해당 Mint의 Token Account가 없으면 Transfer할 때 새로 만들어야 하고, 그 최소 Balance(약 0.002 SOL)가 든다.
- 누구나 SPL Token Program으로 직접 Transfer할 수 있어 일반 NFT는 Royalty를 강제할 수 없다. 이를 보완한 pNFT는 Token Record Account와 Rule Set이 더해져 구조가 더 복잡하다.
- Collection도 NFT와 같은 Account 구조로 표현되어, Royalty 같은 설정을 Collection 단위로 한 번에 바꿀 수 없고 NFT마다 수정해야 한다.
- On-Chain Attribute를 지원하지 않는다.
- Metaplex는 Token Metadata를 Legacy Program으로 분류하고, 새 프로젝트에는 Core를 권장한다.

### Token Metadata의 On-Chain 구성 요소

![solana-nft-01-tm-master-edition.png](solana-nft-01-tm-master-edition.png)

**1. Token Account**

누가 NFT를 가졌는지를 기록한다. 소유와 Transfer는 SPL Token의 구조를 그대로 쓰므로, Transfer도 SPL Token Program이 처리한다.

- Owner: NFT를 가진 Wallet 주소
- Amount = 1: 보유 수량. NFT는 하나뿐이므로 1이다

**2. Mint Account**

Mint Account의 주소가 NFT의 고유 식별자다. Solana의 NFT는 다음과 같은 특성을 가진 Mint Account이다.

- Supply = 1: 유통되는 Token이 하나뿐이다.
- Decimals = 0: 0.5개 같은 단위가 없다.
- Mint Authority 없음: 누구도 추가로 Mint할 수 없다. 
  - Token Metadata는 이 조건을 Mint Authority를 없애는 대신 Master Edition Account로 넘기는 방식으로 구현한다(아래 4번). 그래서 그림에는 "Mint Authority = Edition"으로 적혀 있다.

Mint Account는 Supply 같은 몇 가지 값만 가지고 있어서, 앱과 Marketplace(NFT를 사고파는 서비스)가 공통으로 이해할 수 있는 표준화된 데이터를 담을 수 없다. 이 부분을 Metadata Account가 채운다.

**3. Metadata Account**

NFT의 기본 정보(Name, Symbol, URI 등)를 담는다.

- Mint Account의 주소에서 유도한 PDA다.
- 이미지와 속성은 여기에 없고, URI가 가리키는 Off-Chain JSON 파일에 있다.

[Metadata Account Structure](https://solana.com/docs/tokens/metaplex#metadata-account-structure)는 다음과 같다.

```rust
pub struct Metadata {
    pub key: Key,                              // Account type identifier
    pub update_authority: Pubkey,              // Can update metadata
    pub mint: Pubkey,                          // Associated mint
    pub name: String,                          // Token name (max 32 chars)
    pub symbol: String,                        // Token symbol (max 10 chars)
    pub uri: String,                           // URI to off-chain JSON (max 200 chars)
    pub seller_fee_basis_points: u16,          // Royalty % (100 = 1%)
    pub creators: Option<Vec<Creator>>,        // Creator list with shares
    pub primary_sale_happened: bool,           // Primary sale flag
    pub is_mutable: bool,                      // Can metadata be updated
    pub edition_nonce: Option<u8>,             // Edition nonce
    pub token_standard: Option<TokenStandard>, // Token type
    pub collection: Option<Collection>,        // Collection info
    pub uses: Option<Uses>,                    // Use tracking
}
```

`seller_fee_basis_points`의 Royalty(로열티)는 NFT가 다시 팔릴 때 원 창작자에게 가는 수수료다.

**4. Master Edition Account**

Token Metadata Program은 NFT를 위해 [Master Edition Account](https://www.metaplex.com/docs/smart-contracts/token-metadata#:~:text=NFTs%20called%20the-,Master%20Edition%20Account,-.%20This%20account%20is)라는 PDA를 하나 더 제공한다. 이 Account의 존재가 해당 Mint가 Non-Fungible이라는 증명이 된다.

Token Metadata Program은 Master Edition Account를 만들기 전에 Mint가 위의 NFT 조건을 만족하는지 확인한다. 그 뒤 Mint Authority를 없애는 대신, Mint Authority와 Freeze Authority를 모두 Master Edition PDA로 이전한다. 그래서 Token Metadata Program을 거치지 않고는 누구도 Token을 추가로 Mint하거나 Freeze할 수 없다. Authority를 없애지 않고 이전하는 이유는 [Token Metadata FAQ](https://www.metaplex.com/docs/smart-contracts/token-metadata/faq#why-are-the-mint-and-freeze-authorities-transferred-to-the-edition-pda)에서 설명한다.

![solana-nft-MasterEdition.png](solana-nft-MasterEdition.png)

Master Edition Account의 필드는 Key(`MasterEditionV2`), Supply, Max Supply다. Supply와 Max Supply는 NFT의 복제본인 Print Edition에 쓰인다.

- Supply: 지금까지 Mint된 Print Edition 수

- Max Supply: Print Edition 수의 한도. 0이면 Print Edition을 만들 수 없고, None이면 제한 없이 만들 수 있다

### Token Metadata의 Off-Chain 구성 요소

Metadata Account의 URI는 Off-Chain의 JSON 파일을 가리킨다. On-Chain 저장 비용에 묶이지 않고 추가 데이터를 제공하기 위한 구조다. URI 문자열 자체는 Metadata Account에 On-Chain으로 저장되고, Off-Chain에 있는 것은 URI가 가리키는 대상이다.

NFT의 JSON 파일은 [Non-Fungible Standard](https://www.metaplex.com/docs/smart-contracts/token-metadata/token-standard#the-non-fungible-standard)를 따른다. 필드는 다음과 같다.

| Field | Type | Description |
|---|---|---|
| name | string | Name of the asset. |
| description | string | Description of the asset. |
| image | string | URI pointing to the asset's logo. |
| animation_url | string | URI pointing to the asset's animation. |
| external_url | string | URI pointing to an external URL defining the asset — e.g. the game's main site. |
| attributes | array | Array of attributes defining the characteristics of the asset.<br>trait_type (string): The type of attribute.<br>value (string): The value for that attribute. |
| properties | object | Additional properties that define the asset.<br>files (array): Additional files to include with the asset.<br>uri (string): The file's URI.<br>type (string): The file's type. E.g. image/png, video/mp4, etc.<br>cdn (boolean, optional): Whether the file is served from a CDN.<br>category (string): A media category for the asset. E.g. video, image, etc. |

이 형식을 따르는 JSON 파일의 예시는 다음과 같다.

```json
{
  "name": "Sample NFT #1",
  "description": "Example NFT",
  "image": "https://example.com/nft/1.png",
  "animation_url": "https://example.com/nft/1.mp4",
  "external_url": "https://example.com",
  "attributes": [
    { "trait_type": "Background", "value": "Blue" },
    { "trait_type": "Rarity", "value": "Rare" }
  ],
  "properties": {
    "files": [
      { "uri": "https://example.com/nft/1.png", "type": "image/png" },
      { "uri": "https://example.com/nft/1.mp4", "type": "video/mp4", "cdn": false }
    ],
    "category": "video"
  }
}
```

JSON 파일은 누구나 접근할 수 있는 위치라면 어디에나 둘 수 있다. Arweave, IPFS(InterPlanetary File System), 클라우드 저장소(Amazon Web Services S3, Google Cloud)가 주로 쓰인다([Core Asset 문서](https://www.metaplex.com/docs/smart-contracts/core/what-is-an-asset#off-chain-metadata)). Off-Chain 파일이 사라지면 On-Chain의 Name과 URI는 남지만 이미지와 `attributes` 같은 정보는 조회할 수 없게 된다. 이 때문에 같은 문서는 Arweave나 Pinning된 IPFS 같은 영구 저장소를 쓰라고 권한다. Pinning(피닝)은 IPFS 네트워크에 참여한 컴퓨터가 특정 데이터를 지우지 않고 계속 보관하도록 고정하는 것이다.

내용이 바뀌지 않는다는 것을 보장하려면 두 가지를 함께 써야 한다. JSON 파일을 Arweave 같은 영구 저장소에 두고, Metadata Account의 Is Mutable을 False로 바꾸는 것이다. Is Mutable이 False이면 URI, Name, Creators 같은 필드를 다시는 바꿀 수 없다.

### Token Metadata의 Transfer 구조 예시

Alice가 Bob에게 NFT를 Transfer하면 두 사람의 Token Account에서 Amount 값만 바뀐다.

Alice가 Token Metadata NFT 하나를 가지고 있고, 이 NFT를 Bob에게 보낸다. Bob은 이 NFT를 처음 받는다. 일반 NFT의 Transfer는 SPL Token Program의 Transfer Instruction으로 처리되며, Token Metadata Program은 관여하지 않는다.

1. Bob에게 해당 Mint의 Token Account가 없으면 먼저 만든다(보통 ATA). 생성 비용인 Rent는 Instruction에 지정한 Payer(비용을 내는 Account)가 낸다.
2. Alice가 서명한 Transfer Instruction으로 Alice의 Token Account의 Amount가 1에서 0이 된다.
3. Bob의 Token Account의 Amount가 0에서 1이 된다.

![solana-nft-tm-transfer-ex.png](solana-nft-tm-transfer-ex.png)

Mint Account, Metadata Account, Master Edition Account는 변하지 않는다. 각 Token Account의 Owner 필드도 변하지 않는다. 소유권 이동은 "Amount 1이 어느 Token Account에 있는가"로 표현된다.

Transfer 후 Alice의 Token Account는 Amount 0인 상태로 남는다. 그래서 NFT 하나를 두 사람이 가진 적이 있으면 Token Account도 2개가 된다. 다만 Alice가 빈 Account를 닫으면 Rent를 돌려받고 Account는 사라지므로, 이 설명은 빈 Account를 닫지 않았을 때만 성립한다.

### 확장: Print Edition, Collection, Programmable NFT

**Print Edition**

Master Edition Account는 Non-Fungibility(대체 불가능성)의 증명 외에, NFT의 복제본을 Mint하는 기능도 제공한다. 원본을 Original NFT(Master Edition NFT), 복제본을 Print NFT라고 부른다([Printing Editions 절](https://www.metaplex.com/docs/smart-contracts/token-metadata#printing-editions)). Print NFT의 Token Standard는 `NonFungibleEdition`이다.

- Print NFT는 자신만의 Mint Account와 Metadata Account를 가진다. Metadata의 내용은 Original NFT에서 복사된다.
- Print NFT는 Master Edition Account 대신 Edition Account라는 PDA를 가진다. Edition Account는 Key(`EditionV1`), Parent(부모 Master Edition의 주소), Edition(Edition 번호)을 기록한다.
- Master Edition Account와 Edition Account는 PDA의 Seed가 같다. 그래서 하나의 NFT는 둘 중 하나만 가질 수 있다.

![solana-nft-03-tm-printing-editions.png](solana-nft-03-tm-printing-editions.png)

위 그림의 "OR"는 같은 PDA 주소에 Master Edition Account와 Edition Account 중 하나만 온다는 뜻이다. Edition Account의 Parent에서 Master Edition Account로 이어진 점선은 어느 원본에서 복제되었는지를 나타낸다.

**Collection NFT**

[Collection](https://www.metaplex.com/docs/smart-contracts/token-metadata/collections)은 NFT 묶음이다. Token Metadata에서 Collection은 그 자체가 하나의 NFT(Collection NFT)다. 일반 NFT와 Account 구조가 같고, Metadata의 `collection_details` 필드가 설정되어 있다는 점만 다르다. 개별 NFT는 Metadata의 `collection` 필드에 Collection NFT의 Mint 주소(`key`)와 `verified`(검증 여부) 값을 가진다. `collection` 필드는 누구나 임의로 채울 수 있으므로, Collection의 Authority가 Verify Instruction에 서명해야 `verified`가 True가 된다. Wallet과 Marketplace는 `verified`가 True인지 확인해야 한다.

![solana-nft-collection.png](solana-nft-collection.png)

**Programmable NFT (pNFT)**

Token Metadata Program은 SPL Token Program 위에 만들어져 있다. 그래서 누구나 Token Metadata Program을 거치지 않고 Token을 Transfer할 수 있고, Token Metadata Program은 자신이 붙어 있는 Token에 규칙을 강제할 수 없다. 대표적인 문제가 Royalty다. Metadata의 Seller Fee Basis Points는 참고용 값일 뿐이어서, Royalty를 지급하지 않는 Marketplace를 막을 수 없었다. pNFT는 이를 해결하려고 도입한 선택형(Opt-In) Token Standard이며, 값은 `ProgrammableNonFungible`이다([Programmable NFTs 절](https://www.metaplex.com/docs/smart-contracts/token-metadata#pnfts), [pNFT 문서](https://www.metaplex.com/docs/smart-contracts/token-metadata/pnfts)).

- pNFT의 Token Account는 SPL Token Program 수준에서 항상 Frozen(Freeze된) 상태다. 그래서 Transfer, Lock(잠금), Burn은 반드시 Token Metadata Program을 거쳐야 한다. Program은 한 Instruction 안에서 Freeze 해제, 작업 수행, 다시 Freeze를 처리한다.

- 창작자는 작업별 허용 규칙을 Rule Set(규칙 집합) Account에 정의한다. Rule Set은 Token Auth Rules라는 별도의 Metaplex Program이 관리하고, Metadata Account의 `programmable_config`가 Rule Set Account의 주소를 가리킨다. 예를 들어 Royalty를 지키는 Program 주소만 담은 Allow List(허용 목록)를 둘 수 있다.

- Token Record Account가 추가된다. PDA Seed는 `["metadata", Program ID, Mint 주소, "token_record", Token Account 주소]`다([Delegates 문서](https://www.metaplex.com/docs/smart-contracts/token-metadata/delegates)). Token Account가 항상 Frozen 상태이므로, 실제 Lock 여부와 Delegate(위임, Owner가 다른 주소에 특정 작업 권한을 맡기는 것) 정보는 Token Record가 기록한다.

![solana-nft-04-tm-pnfts.png](solana-nft-04-tm-pnfts.png)

위 그림에서 Token Account에는 "State = Frozen"이 적혀 있고, Metadata Account의 Programmable Configs가 RuleSet Account로 이어진다. Token Record Account는 이 그림에 나오지 않는다.

<br />

## Core

### What is a Core?

[Metaplex Core](https://www.metaplex.com/docs/smart-contracts/core)는 기존의 Token Metadata를 대체하는 차세대 Solana NFT 표준이다. NFT 데이터를 Account 하나에 담는 구조로, NFT 하나를 Mint하는 비용을 Token Metadata의 약 0.022 SOL에서 약 0.003 SOL로 80% 이상 줄였다. 또한 Royalty 강제, Collection 단위 작업(Collection 전체의 Freeze, Royalty 변경, Asset 일괄 수정), 기능을 골라 붙이는 Plugin 시스템을 제공한다.

Token Metadata의 기능이 Core에서 대응되는 방식은 다음과 같다.

| Token Metadata	| Core Equivalent |
| --- | --- |
| Mint account | Asset account |
| Metadata account | Asset account (combined) |
| Associated Token Account | Not needed |
| Freeze authority | Freeze Delegate plugin |
| Update authority | Update authority (same) |
| Delegate | Transfer/Burn/Update Delegate plugins |
| Collection verified | Collection membership (automatic) |
| Creators array | Verified Creators plugin |
| Uses/utility | Plugins (custom logic) |

[출처: https://www.metaplex.com/docs/smart-contracts/core/tm-differences#tm-concept-core-equivalent](https://www.metaplex.com/docs/smart-contracts/core/tm-differences#tm-concept-core-equivalent)

 
**장점**
- NFT 하나가 Asset Account 하나라 구조가 단순하고, Mint 비용이 약 0.003 SOL로 Token Metadata(약 0.022 SOL)보다 80% 이상 낮다. 연산량(Compute Unit)도 약 17,000 CU로 Token Metadata(약 205,000 CU)보다 적다.
- Token Account가 없어 Transfer할 때 새 Account를 만들지 않고, Transaction 수수료만 든다.
- Freeze, Delegate, Royalty, Attributes 같은 기능을 Plugin으로 필요한 것만 붙인다. Royalties Plugin은 Allowlist/Denylist로 Royalty를 강제하고, Attributes Plugin으로 On-Chain Attribute를 저장할 수 있다.
- Collection이 전용 Account라 Royalty 변경이나 Freeze를 Collection 단위로 한 번에 할 수 있다. 같은 Plugin을 Asset에 붙이면 Asset의 설정이 우선한다.

**단점**
- SPL Token이 아니므로 Token Account를 전제로 한 Wallet, Marketplace, Program은 Core를 따로 지원해야 쓸 수 있다.
- 기존 Token Metadata NFT를 Core로 바로 바꿀 수 없다. 기존 NFT를 Burn하고 Core Asset을 새로 Mint해야 한다.
- Plugin은 Metaplex가 제공하는 것만 쓸 수 있고 직접 만들 수 없다. 사용자 정의 검증은 Oracle 같은 External Plugin으로 처리해야 한다.
- Plugin을 붙일수록 Account 크기가 커져 Rent가 늘어난다(대부분 Plugin당 약 0.001 SOL).
- NFT용 표준이라 Fungible Token에는 쓸 수 없고, Fungible Token은 SPL Token을 써야 한다.

### Core의 On-Chain 구성 요소

**Asset Account**

[Core Asset](https://www.metaplex.com/docs/smart-contracts/core/what-is-an-asset)은 NFT 하나를 나타내는 단일 Solana Account이다. Core는 SPL Token Program을 쓰지 않고 소유권을 Asset Account의 `owner` 필드에 직접 기록하므로, Mint Account, Token Account, ATA가 필요 없다.

![solana-nft-05-core-overview.png](solana-nft-05-core-overview.png)

Owner, Name, URI, Update Authority 같은 핵심 데이터와 Plugin 데이터를 모두 이 Account에 저장한다. Freeze, Delegate, Royalty 같은 기능은 기본 구조에 넣지 않고, 필요한 것만 Plugin(Asset에 붙이는 확장 모듈)으로 붙인다.

Asset Account의 기본 구조인 [`AssetV1`](https://github.com/metaplex-foundation/mpl-core/blob/main/programs/mpl-core/src/state/asset.rs)은 다음과 같다. 주석의 숫자는 각 필드가 차지하는 기본 Byte 크기이며, `name`과 `uri`는 길이 정보 4 Byte 뒤에 실제 문자열이 붙는다.

```rust
pub struct AssetV1 {
    /// The account discriminator.
    pub key: Key, //1
    /// The owner of the asset.
    pub owner: Pubkey, //32
    /// The update authority of the asset.
    pub update_authority: UpdateAuthority, //33
    /// The name of the asset.
    pub name: String, //4
    /// The URI of the asset that points to the off-chain data.
    pub uri: String, //4
    /// The sequence number used for indexing with compression.
    pub seq: Option<u64>, //1
}
```

여기서 `update_authority`는 Asset의 데이터를 수정할 수 있는 주체다. `update_authority`의 타입인 [`UpdateAuthority`](https://github.com/metaplex-foundation/mpl-core/blob/main/programs/mpl-core/src/state/update_authority.rs)는 다음 세 가지 중 하나다.

```rust
pub enum UpdateAuthority {
    None,               /// No update authority, used for immutability.
    Address(Pubkey),    /// A standard address or PDA.
    Collection(Pubkey), /// Authority delegated to a collection.
}
```

- `None`: Update Authority가 없다. `name`과 `uri`를 다시는 바꿀 수 없다.
- `Address`: 지정한 Address(일반 Address 또는 PDA)가 Update Authority다. 이 Asset은 Collection에 속하지 않는다.
- `Collection`: 값이 Collection Account의 Address이며, Asset이 그 Collection에 속한다는 뜻이다. 이때 Asset의 수정 권한은 Collection에 위임된다.

Collection에 속한 Asset을 `fetchAssetV1`로 조회하고, `collectionAddress`로 Collection Address를 구하면 다음과 같다.

```json
collection: '2222222222222222222222222222222'
asset: {
    key: AssetV1,
    owner: "11111111111111111111111111111111",
    updateAuthority: {
      type: 'Collection',
      address: '2222222222222222222222222222222'
    },
    name: "My Core Asset",
    uri: "https://example.com/metadata.json",
    ...
}
```

### Core의 Off-Chain 구성 요소

[Off-Chain Metadata](https://www.metaplex.com/docs/smart-contracts/core/what-is-an-asset#off-chain-metadata)의 구성은 Token Metadata와 같다. Asset Account의 `uri`가 Off-Chain JSON 파일을 가리키고, 이 JSON 파일은 Token Metadata의 [Non-Fungible Standard](https://www.metaplex.com/docs/smart-contracts/token-metadata/token-standard#the-non-fungible-standard) 형식을 따른다.

![solana-nft-06-core-off-chain-metadata.png](solana-nft-06-core-off-chain-metadata.png)

### Core의 Transfer 구조 예시

Alice가 Core Asset 하나를 가지고 있고, Asset Account의 `owner`에는 Alice의 Address가 기록되어 있다. Alice가 서명해 이 Asset을 Bob에게 Transfer하면, MPL Core Program이 같은 Asset Account의 `owner`를 Alice의 Address에서 Bob의 Address로 바꾼다.
 
![solana-nft-core-transfer-ex.png](solana-nft-core-transfer-ex.png)

새로 만들어지는 Account가 없으므로 Transaction 수수료 외에 추가 비용이 들지 않는다. Token Metadata에서는 Bob의 Token Account가 없으면 새로 만들어야 하고, Alice의 Token Account는 닫지 않으면 빈 채로 남는다.

Transfer되면 Owner Managed Plugin(Transfer, Freeze, Burn Delegate)의 Delegate 권한은 자동으로 회수되고, Authority Managed Plugin과 Permanent Plugin은 그대로 유지된다. Freeze된 Asset은 Freeze가 풀릴 때까지 Transfer할 수 없다.

### Core Plugin System

[Plugin](https://www.metaplex.com/docs/smart-contracts/core/plugins)은 Core Asset 또는 Collection에 기능을 추가하는 On-Chain 확장이다. 데이터를 저장하거나(예: Attributes), 규칙을 강제하거나(예: Royalties), 권한을 Delegate한다(예: Freeze, Transfer).

Plugin 데이터는 별도 Account가 아니라 같은 Asset Account 안에서 기본 필드 뒤에 붙는다. 저장 순서는 기본 데이터, Plugin Header, 각 Plugin, Plugin Registry(각 Plugin의 종류와 위치를 기록한 목록)다([`plugin_header.rs`](https://github.com/metaplex-foundation/mpl-core/blob/main/programs/mpl-core/src/plugins/plugin_header.rs)). Plugin을 추가하면 Account 크기와 Rent가 늘어난다.

Plugin 데이터는 별도 Account가 아니라 같은 Asset Account 안에서 기본 필드 뒤에 붙는다. 저장 순서는 기본 데이터, Plugin Header, 각 Plugin, Plugin Registry(각 Plugin의 종류와 위치를 기록한 목록)다. Plugin Header(9 Byte)는 Key와, Account 끝에 있는 Plugin Registry의 위치(Offset)를 담는다. Plugin을 추가하면 Account 크기와 Rent가 늘어난다.

```rust
pub struct PluginHeaderV1 {
    /// The Discriminator of the header which doubles as a Plugin metadata version.
    pub key: Key, // 1
    /// The offset to the plugin registry stored at the end of the account.
    pub plugin_registry_offset: usize, // 8
}
```
[출처: `plugin_header.rs`](https://github.com/metaplex-foundation/mpl-core/blob/main/programs/mpl-core/src/plugins/plugin_header.rs)

Plugin은 Create, Transfer, Burn, Plugin 추가·제거 같은 Lifecycle Event에 개입해 동작을 승인하거나 거부한다. Permanent Plugin은 강제 승인(Force Approve)도 할 수 있어, 다른 Plugin의 거부보다 우선한다. Plugin은 누가, 언제 추가할 수 있는지에 따라 세 종류로 나뉜다.

**Types of Plugins**

- Owner Managed Plugins: Asset Owner가 Transaction에 서명해야만 추가할 수 있는 플러그인
  - Transfer, Freeze, Burn Delegate

- Authority Managed Plugins: Core Asset 또는 Core Collection의 Update Authority(권한자)가 언제든지 추가하고 수정할 수 있는 플러그인
  - Royalties, Attributes, Update Delegate

- Permanent Plugins: Core Asset을 생성할 때만 추가할 수 있는 플러그인
  - Asset이 이미 존재하는 경우 Permanent 플러그인을 추가할 수 없음
  - Permanent Transfer/Freeze/Burn Delegate


Collection에는 Authority Managed Plugin과 Permanent Plugin만 붙일 수 있다.

이외에 [External Plugin](https://www.metaplex.com/docs/smart-contracts/core/external-plugins/overview)도 있다. 외부 Account의 상태에 따라 Lifecycle Event를 거부할 수 있는 Oracle, Data Authority만 쓸 수 있는 데이터 영역을 Asset에 두는 AppData가 여기에 속하며, 둘 다 Authority Managed다.

### Core Collections

[Core Collection](https://www.metaplex.com/docs/smart-contracts/core/collections)은 같은 시리즈나 세트에 속하는 Asset을 묶는 Account로, Asset과는 다른 별도의 Account 종류다. Collection Account는 Collection 이름과 이미지 URI 같은 공유 Metadata를 저장하는 표지 역할을 하며, Collection 전체에 적용할 Plugin도 가질 수 있다. Asset은 Collection 없이 단독으로 존재할 수도 있다.

Assets의 Collection 표현인 [`CollectionV1`](https://github.com/metaplex-foundation/mpl-core/blob/main/programs/mpl-core/src/state/collection.rs)은 다음과 같다.

```rust
pub struct CollectionV1 {
    /// The account discriminator.
    pub key: Key, //1
    /// The update authority of the collection.
    pub update_authority: Pubkey, //32
    /// The name of the collection.
    pub name: String, //4
    /// The URI that links to what data to show for the collection.
    pub uri: String, //4
    /// The number of assets minted in the collection.
    pub num_minted: u32, //4
    /// The number of assets currently in the collection.
    pub current_size: u32, //4
}
```

Asset Account와 달리 `owner` 필드가 없어, Collection은 Asset처럼 소유하거나 Transfer할 수 없다. `update_authority`도 Asset처럼 Type을 고르는 값이 아니라 Address(Pubkey) 하나다. `num_minted`는 지금까지 이 Collection에 Mint된 Asset의 누적 수, `current_size`는 현재 Collection에 속한 Asset 수이고, `uri`는 Collection의 Off-Chain JSON 파일을 가리킨다.

Asset과 Collection의 관계는 다음 규칙을 따른다.

- Asset은 `update_authority`를 `Collection(Collection Account Address)`로 두어 Collection을 참조한다.
- 생성 이후에도 Asset의 Update Instruction으로 Asset을 Collection에 추가, 이동, 제거할 수 있다. 제거할 때는 `update_authority`를 `Address`로 바꾼다.
- 소속을 바꾸려면 Collection의 Update Authority여야 하고, Collection에 속하지 않은 Asset을 추가할 때는 그 Asset의 Update Authority이기도 해야 한다. Collection 사이에서 옮길 때는 두 Collection 모두에 대한 권한이 필요하다. Collection에 등록된 Update Delegate도 Asset을 제거하거나, 자신이 권한을 가진 Asset을 추가할 수 있다.
- Asset이 남아 있는 Collection은 닫을 수 없다. 모든 Asset을 먼저 제거해야 한다.
- Collection에 붙인 Plugin은 소속된 모든 Asset에 적용되며, Asset이 같은 종류의 Plugin을 직접 가지고 있으면 Asset의 Plugin이 우선한다.
- Collection에는 Owner가 없으므로 Authority Managed Plugin과 Permanent Plugin만 붙일 수 있다.

Collection Account 하나에 Asset 두 개를 만들고, 첫 번째는 Alice가, 두 번째는 Bob이 가진 상황을 예로 들면 다음과 같다. 두 Asset의 `update_authority`는 `Collection(Collection Account Address)`이고, Collection Account의 `num_minted`와 `current_size`는 모두 2다.

예시는 다음과 같다.

![solana-nft-core-collection.png](solana-nft-core-collection.png)

- [Plugin Priority](https://www.metaplex.com/docs/smart-contracts/core/plugins#plugin-priority): Collection에 Royalties Plugin을 2%로 설정하면 두 Asset 모두에 2%가 적용된다. 특정 Asset에 5% Royalties Plugin을 따로 붙이면 그 Asset만 5%가 된다.

- [Updating a Core Collection](https://www.metaplex.com/docs/smart-contracts/core/collections/update): Collection Account 하나만 수정해 소속 Asset 전체에 반영되는 것은 Collection에 붙인 Plugin(Royalties, Permanent Freeze Delegate 등)과 Update Authority다. Asset의 `update_authority`가 `Collection`이면 그 Asset의 수정 권한은 Collection의 Update Authority가 가지기 때문이다. 반면 개별 Asset의 `name`과 `uri`는 각 Asset Account에 따로 저장되므로, Collection의 `name`이나 `uri`를 바꿔도 Asset의 값은 바뀌지 않는다.

<br />

## Compressed NFT (cNFT)

### What is a Compressed NFT?

Token Metadata와 Core는 NFT마다 Account를 만들어 Mint 수에 비례해 Rent가 든다. cNFT(Compressed NFT)는 NFT마다 Account를 만들지 않아 수백만 개 단위의 Mint에 적합하다. 이를 담당하는 Metaplex Program이 Bubblegum이며, 최신 버전은 [Bubblegum V2](https://www.metaplex.com/docs/smart-contracts/bubblegum-v2)다.

cNFT는 [Merkle Tree](https://www.metaplex.com/docs/smart-contracts/bubblegum-v2#merkle-trees-leaves-and-proofs) 안에서만 존재한다. Merkle Tree는 각 부모 Node가 두 자식 Node의 Hash인 Binary Tree(이진 트리)다.

- Leaf(리프): cNFT 하나의 데이터를 Hash한 값
- Root(루트): Tree 전체를 대표하는 맨 위의 Hash
- Proof(증명): Leaf에서 Root까지 경로의 형제 Node Hash 목록. 특정 Leaf가 Tree에 속함을 검증하는 데 쓴다

![solana-nft-07-bubblegum-merkle-tree.png](solana-nft-07-bubblegum-merkle-tree.png)

위 그림에서 Leaf 3의 Proof는 Leaf 4, Node 3, Node 2이며, 이들로 Root를 다시 계산해 On-Chain Root와 비교한다. cNFT가 수정되거나 Transfer되면 Leaf와 Root가 바뀐다. Merkle Tree는 NFT 데이터를 저장하지 않고도 cNFT의 존재를 검증할 수 있게 한다.

[https://github.com/metaplex-foundation/mpl-bubblegum/tree/main](https://github.com/metaplex-foundation/mpl-bubblegum/tree/main)

### Compressed NFT의 구성 요소

cNFT를 Mint하려면 먼저 On-Chain Account 두 개로 이루어진 [Bubblegum Tree](https://www.metaplex.com/docs/smart-contracts/bubblegum-v2/create-trees)를 만들어야 한다.

![solana-nft-cnft-accounts.png](solana-nft-cnft-accounts.png)

- Merkle Tree Account: V2에서는 MPL Account Compression Program이 소유한다. Leaf를 모두 저장하지 않고 다음만 저장한다.

  - Header
  - 최근 Root와 변경 경로를 담은 ChangeLog Buffer. 같은 Block 안의 동시 변경으로 Proof가 오래되면 이 기록으로 보정한다
  - 가장 오른쪽 Leaf의 Proof. Mint에 Proof가 필요 없는 이유다
  - Canopy(상위 Node Cache). Transaction에 실을 Proof를 줄인다

- TreeConfigV2 Account: Merkle Tree Address로 유도한 Bubblegum PDA다. Tree Creator와 Delegate, 최대 Mint 수, 지금까지 Mint된 수(Leaf의 Nonce로도 쓰임), Public 여부, Leaf Schema 버전을 저장한다.

- Leaf: Leaf Schema를 Keccak-256으로 Hash한 32 Byte 값이다. Leaf Schema에는 Asset ID, Owner, Delegate, Nonce, Metadata Hash, Creator Hash가 들어가고, V2는 Collection Hash 등이 추가된다([Hashed NFT Data](https://www.metaplex.com/docs/smart-contracts/bubblegum-v2/hashed-nft-data)). Asset ID는 PDA 형식의 식별자이며, 그 Address에 Account는 없다.

- Tree 파라미터: Max Depth(최대 Leaf 수 2^Max Depth), Max Buffer Size(Block당 동시 변경 수), Canopy Depth다. 셋 다 생성 후 바꿀 수 없다. 예를 들어 1,048,576개용 Tree(Depth 20, Canopy 13, Buffer 1024)의 비용은 약 8.5 SOL로, cNFT당 약 0.000013 SOL이다.

Leaf에는 Hash만 있으므로 원본 데이터는 Account가 아니라 Mint와 수정 Transaction에 기록된다. Bubblegum은 Noop Program을 CPI로 호출하고, 그 Instruction Data에 데이터를 실어 Ledger에 남긴다. 현재 상태를 알려면 이 기록을 순서대로 읽어야 하므로, 실제로는 DAS API를 지원하는 RPC Provider가 Indexing해 데이터와 Proof를 제공한다([Stored NFT Data](https://www.metaplex.com/docs/smart-contracts/bubblegum-v2/stored-nft-data)). Off-Chain Metadata(JSON, 미디어 파일)는 다른 표준과 같다.

![solana-nft-08-bubblegum-transactions.png](solana-nft-08-bubblegum-transactions.png)

![solana-nft-cnft-onoff.png](solana-nft-cnft-onoff.png)

### 예시

- Mint: Bubblegum이 Leaf Schema를 Hash해 Tree의 가장 오른쪽에 Leaf로 추가하고 Root를 갱신한다. Proof는 필요 없다.
- Alice → Bob Transfer: Client가 DAS API에서 현재 Leaf 정보와 Proof를 받아 Instruction에 넣고, Alice가 서명한다. Program은 Proof로 On-Chain Root와 일치하는지 검증한 뒤, `owner`를 Bob으로 바꾼 Leaf로 교체한다. 바뀌는 On-Chain 데이터는 Merkle Tree Account뿐이며, Bob을 위한 Account는 만들어지지 않는다.

### Bubblegum V1과 V2

V2는 새 Leaf Schema(LeafSchemaV2)와 MPL Account Compression을 쓰고, Collection으로 MPL Core Collection(BubblegumV2 Plugin 필요)을 사용한다. 이를 통해 Freeze와 Thaw, Soulbound, Permanent Transfer/Burn Delegate, Royalty 강제를 지원한다([What's New in Bubblegum V2](https://www.metaplex.com/docs/smart-contracts/bubblegum-v2#whats-new-in-bubblegum-v2)). 반면 V1에서 가능했던 Decompress(일반 NFT로 변환)는 V2에서 불가능하다. V1 Tree와 V2 Tree는 서로 호환되지 않는다. Protocol Fee는 V1의 Create가 무료이고, V2는 Create 0.00009 SOL, Transfer 0.000006 SOL이다.

### 특징과 한계

- 장점: NFT마다 Account가 없어 개당 비용이 매우 낮고(큰 Tree에서 약 0.00001 SOL), Tree 하나에 최대 2^30개를 담을 수 있다.

- 단점
  - 조회와 Proof 획득을 DAS API 지원 RPC에 의존한다.
  - 변경마다 Proof를 실어야 해 Transaction 크기를 차지한다.
  - Tree 파라미터를 바꿀 수 없다.
  - 한 Block의 변경이 Max Buffer Size를 넘으면 일부 Transaction이 실패한다.
  - 지원 범위가 좁다. [Ecosystem Compatibility](https://www.metaplex.com/docs/smart-contracts/bubblegum-v2#ecosystem-compatibility)(2026-06-19 갱신) 기준으로 Phantom, Solflare, Backpack은 V2 cNFT의 조회만 지원하고, Magic Eden과 Tensor는 아직 지원하지 않는다.