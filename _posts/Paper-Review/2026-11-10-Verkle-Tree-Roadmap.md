---
title: "Verkle Tree and the Ethereum Roadmap"
date: 2026-11-10 00:01:00 +0900
last_modified_at: 2026-11-24
categories: [Paper Review]
tags: [Blockchain, Ethereum, Verkle Tree, Roadmap, Stateless Client, Binary Tree]
media_subpath: /assets/img/posts/verkle-tree/
---

>
> 1. [Verkle Tree]({% post_url Paper-Review/2026-11-10-Verkle-Tree %})
> 2. **Verkle Tree and the Ethereum Roadmap**

[Reference]

- [https://vitalik.eth.limo/general/2024/10/23/futures4.html](https://vitalik.eth.limo/general/2024/10/23/futures4.html)
- [https://ethereum.org/roadmap/](https://ethereum.org/roadmap/)
- [https://strawmap.org/](https://strawmap.org/)
- [https://forkcast.org/](https://forkcast.org/)
- [https://blog.ethereum.org/2026/02/18/protocol-priorities-update-2026](https://blog.ethereum.org/2026/02/18/protocol-priorities-update-2026)

---

### TL;DR

- Verkle Tree는 Roadmap의 **The Verge**에서 State Tree를 바꾸는 수단이었다. 2024년 2월에는 Osaka Upgrade에 넣기로 정해졌다.

- 실제로는 어느 Upgrade에도 들어가지 못했다. Osaka는 Consensus Layer(합의 계층)의 Upgrade인 Fulu와 합쳐 Fusaka라는 이름으로 2025년 12월에 적용됐고 Verkle Tree는 없었다.

- 빠진 이유는 세 가지다. 타원곡선 기반이라 Quantum Resistance(양자 내성)가 없고, Hash 기반 트리의 Proof를 증명하는 기술이 빨라져 Verkle Tree의 장점이 줄었고, 모든 실행을 SNARK(Succinct Non-interactive Argument of Knowledge)로 증명한다는 목표가 State Tree를 고르는 기준이 되면서 증명하기 쉬운 트리가 필요해졌다.

- 그 자리에는 Hash 기반 Binary Tree가 들어왔다. 2025년 1월의 [EIP-7864](https://eips.ethereum.org/EIPS/eip-7864)(Unified Binary Tree)를 거쳐 2026년 6월의 [EIP-8297](https://eips.ethereum.org/EIPS/eip-8297), 곧 Partitioned Binary Tree(이하 PBT)로 이어졌다. EIP는 Ethereum Improvement Proposal의 줄임말이다.

- 빠진 것은 Verkle Tree라는 수단이다. Stateless Client라는 목표와 [EIP-6800](https://eips.ethereum.org/EIPS/eip-6800)의 설계 일부는 그대로 이어진다.

## Roadmap 그림에서의 위치: The Verge

Vitalik Buterin의 Roadmap은 Ethereum의 개발 과제를 The Merge, The Surge, The Scourge, The Verge, The Purge, The Splurge로 나눈다. Verkle Tree는 이 가운데 **The Verge**에 속했다. 세 시점의 그림을 차례로 본다.

**2022 Roadmap**

![roadmap2022.jpeg](roadmap2022.jpeg)

[https://x.com/VitalikButerin/status/1588669782471368704?a=AFtNzel-piZvsIUa0TE&s=20](https://x.com/VitalikButerin/status/1588669782471368704?a=AFtNzel-piZvsIUa0TE&s=20)

- 한국 시간으로 2022년 11월 5일에 올라온 그림이다. 이때 The Scourge가 새로 들어갔다.
- [당시 보도](https://coingape.com/breaking-vitalik-buterin-releases-updated-ethereum-roadmap)에 따르면 이 개정에서 The Verge는 Verkle Tree만이 아니라 검증(Verification) 전체를 가리키는 이름으로 넓어졌고, 최종 목표로 모든 실행을 SNARK로 검증하는 Ethereum이 적혔다.
- 즉 Verkle Tree는 이 시점부터 The Verge의 전부가 아니라 첫 단계였다.

**2023 Roadmap**

![roadmap2023.jpeg](roadmap2023.jpeg)

[https://vitalik.eth.limo/general/2024/10/23/futures4.html](https://vitalik.eth.limo/general/2024/10/23/futures4.html)

Vitalik Buterin의 2024년 10월 글(이하 2024년 글)에 실린 2023년판 Roadmap이다. 아래는 그 가운데 The Verge 부분이다.

![ethereum-roadmap-verge.png](ethereum-roadmap-verge.png)

- The Verge는 원래 State 저장 구조를 Verkle Tree로 옮기는 일을 가리켰다. 범위는 앞에서 본 대로 2022년 개정에서 넓어졌다. 2024년 글은 이 넓어진 범위를, Stateless 검증에 더해 Ethereum의 모든 실행을 SNARK로 검증하는 것까지 포함한다고 설명한다.

- 목표는 두 가지다. Block을 완전히 검증하는 Client와 Staking Node가 수 GB를 넘는 저장 공간을 쓰지 않게 한다. 장기적으로는 스마트워치에서도 Chain 전체를 검증할 수 있게 한다.

그림의 항목은 다음과 같이 읽는다. 그림에는 EIP 번호가 없고, 항목과 EIP의 대응은 직접 연결한 것이다.

| 그림의 항목 | 뜻 | 해당 EIP |
| --- | --- | --- |
| Verkle tree spec + impl | Verkle Tree의 사양과 구현 | EIP-6800 |
| Code chunking + gas cost updates | Code를 Chunk로 나눠 트리에 넣고, Gas 비용을 Witness에 맞게 고침 | EIP-6800, EIP-4762 |
| Transition spec + impl | 현재 State Tree인 Merkle Patricia Trie(이하 MPT)의 State를 Verkle Tree로 옮기는 절차 | EIP-7612, EIP-7748 |
| Verkle trees | 위 세 가지가 모여 State Tree를 교체함 | |
| SNARK for Verkle proofs, Fully SNARKed Ethereum | Verkle Proof와 실행, 합의 과정을 SNARK로 증명함 | |
| Move to quantum-safe SNARKs (eg. STARKs) | 양자 컴퓨터에 안전한 증명 방식으로 옮김 | |

**⇒ 이 그림에서 Verkle Tree는 최종 목표가 아니라 중간 단계다. 그 뒤에 모든 실행을 SNARK로 증명하는 단계와 양자 내성 증명이 놓여 있다. 뒤의 두 단계가 요구하는 조건, 곧 SNARK로 증명하기 쉬울 것과 양자 컴퓨터에 안전할 것을 Verkle Tree 자신은 충족하지 못한다. Verkle Tree가 빠지는 이유가 이미 같은 그림 안에 들어 있었던 셈이다.**

**2026 Roadmap**

![roadmap2026.jpeg](roadmap2026.jpeg)

![roadmap2026_2.jpeg](roadmap2026_2.jpeg)

[https://x.com/VitalikButerin/status/2086824673556467938?a=U1PdudCaHuTR7kbLD7Q&s=20](https://x.com/VitalikButerin/status/2086824673556467938?a=U1PdudCaHuTR7kbLD7Q&s=20)

- 2026년 8월 10일에 Vitalik Buterin이 2023년판 Roadmap을 Strawmap과 겹쳐 비교한 그림이다.

- 이 비교에서 Verkle Tree는 더 나은 방식으로 대체된 항목으로 분류됐다. [The Defiant](https://thedefiant.io/newsletter/defi-daily/ethereum-bets-its-technical-future-on-privacy-and-quantum-defense)는 Verkle Tree가 EIP-7864의 Unified Binary Tree로 대체됐다고 보도했다. [Unchained](https://unchainedcrypto.com/ethereums-priorities-have-changed-a-lot-since-2023-vitalik-says-comparing-new-roadmap/)는 Unified Binary Tree를 거쳐 PBT로 바뀌었다고 보도했다.

- 두 보도 모두 State Expiry가 새로운 State 유형으로 대체됐다고 적는다.

- 같은 비교에서 양자 안전성은 우선순위가 올라갔고, Verifiable Delay Function과 Ethereum Virtual Machine(이하 EVM) 개선 항목 다수는 내려갔다. 강한 Privacy 같은 항목은 2023년판에 없던 것이다.

세 시점을 나란히 놓으면 다음과 같다.

| 시점 | State Tree를 바꾸는 수단 | 그 시점의 관련 내용 |
| --- | --- | --- |
| 2022년 11월 | Verkle Tree | The Verge가 검증 전체로 확장됨. 최종 목표는 모든 실행을 SNARK로 검증 |
| 2023년판 | Verkle Tree | The Verge 안의 순서는 Verkle Tree, 그 뒤 SNARK로 증명하는 Ethereum, 그 뒤 양자 내성 SNARK |
| 2026년 8월 | Unified Binary Tree를 거쳐 PBT | Roadmap 전체에서 양자 안전성의 우선순위가 올라가고, Privacy가 새 항목으로 들어옴 |

## Verkle Tree 도입을 구성한 EIP

Verkle Tree 도입은 EIP 하나가 아니라 여러 EIP의 묶음이었다. 상태는 각 EIP 문서와 [EIP 목록](https://eips.ethereum.org/core)에서 확인했다.

| EIP | 제목 | 역할 | 상태 |
| --- | --- | --- | --- |
| [EIP-6800](https://eips.ethereum.org/EIPS/eip-6800) | Ethereum state using a unified verkle tree | 트리 구조, Key의 배치, Code Chunk | Stagnant |
| [EIP-4762](https://eips.ethereum.org/EIPS/eip-4762) | Statelessness gas cost changes | Witness를 만드는 비용을 Gas에 반영 | Draft |
| [EIP-7612](https://eips.ethereum.org/EIPS/eip-7612) | Verkle state transition via an overlay tree | Fork 시점에 트리를 바꾸는 방식 | Stagnant |
| [EIP-7748](https://eips.ethereum.org/EIPS/eip-7748) | State conversion to Verkle Tree | 기존 State를 Block마다 조금씩 옮기는 절차 | Draft |
| [EIP-7545](https://eips.ethereum.org/EIPS/eip-7545) | Verkle proof verification precompile | Smart Contract가 State Proof를 검증할 수 있게 하는 내장 Contract | Stagnant |
| [EIP-2935](https://eips.ethereum.org/EIPS/eip-2935) | Serve historical block hashes from state | 과거 Block Hash를 State에 저장 | Final |
| [EIP-7709](https://eips.ethereum.org/EIPS/eip-7709) | Read BLOCKHASH from Storage and Update Cost | Block Hash를 State에서 읽도록 변경 | Draft |
| [EIP-6780](https://eips.ethereum.org/EIPS/eip-6780) | SELFDESTRUCT only in same transaction | Account 전체 삭제를 제한 | Final |
| [EIP-7736](https://eips.ethereum.org/EIPS/eip-7736) | Leaf-level state expiry in verkle trees | 오래 쓰이지 않은 Extension Node를 지우는 State Expiry | Stagnant |

- Draft는 작성 중인 단계, Final은 확정된 단계다. Stagnant는 Draft, Review, Last Call 단계의 EIP가 6개월 이상 활동이 없을 때 옮겨지는 단계다.
- 핵심인 EIP-6800, EIP-7612, EIP-7545가 Stagnant다. 철회를 뜻하는 Withdrawn은 아니다. 제안이 거부된 것이 아니라 더 이상 진행되지 않는 상태다.
- Final인 두 EIP는 Verkle Tree와 무관하게 적용할 수 있는 준비 작업이어서 먼저 적용됐다. State Tree가 Binary Tree로 바뀌어도 되돌릴 이유가 없는 변경이다.

## 도입 계획: 준비, Overlay Tree, 단계적 변환

가장 어려운 부분은 트리 구조가 아니라 전환이었다. 운영 중인 Chain을 멈추지 않고 State 전체를 다른 트리로 옮겨야 한다. 계획은 네 단계였다.

**1단계. 준비 (전환 Fork 이전)**

Stateless Client가 다룰 수 없는 동작을 먼저 없앤다.

- **EIP-2935**: BLOCKHASH 명령은 Client가 최근 Block 이력을 갖고 있다고 가정한다. Stateless Client는 이력을 갖고 있지 않다. 그래서 최근 8,191개 Block의 Hash를 System Contract의 Storage에 저장해, Witness에 담을 수 있게 한다. Pectra에서 적용됐다. EIP-7709는 BLOCKHASH가 실제로 이 Storage를 읽고 그 비용을 내도록 바꾼다.

- **EIP-6780**: SELFDESTRUCT 명령은 Account의 Code와 Storage를 모두 지운다. Verkle Tree에서는 한 Account의 데이터가 서로 연결이 드러나지 않는 여러 Key에 흩어져 있어 이 동작이 불가능하다. 그래서 Contract를 만든 Transaction 안에서 호출한 경우가 아니면 잔고만 보내도록 바꿨다.

- **EIP-4762**: Gas 비용을 Witness 크기에 맞춘다. EIP는 이 변경을 Verkle Tree Fork 직전의 Fork에 넣는 것을 목표로 적는다. Application 개발자가 새 비용 구조에 미리 적응하고, Client 개발자가 Database 형식을 미리 옮기게 하려는 것이다.

EIP-4762의 비용은 접근 단위가 정한다. 접근은 (Address, Sub Key, Leaf Key)로 표현된다. Sub Key까지가 Stem 하나에, Leaf Key가 Suffix에 대응한다.

| 상수 | 값 (Gas) | 부과되는 때 |
| --- | --- | --- |
| WITNESS_BRANCH_COST | 1,900 | 한 Transaction 안에서 어떤 Stem에 처음 접근할 때 |
| WITNESS_CHUNK_COST | 200 | 어떤 Value에 처음 접근할 때 |
| SUBTREE_EDIT_COST | 3,000 | 어떤 Stem 아래에 처음 쓸 때 |
| CHUNK_EDIT_COST | 500 | 어떤 Value에 처음 쓸 때 |
| CHUNK_FILL_COST | 6,200 | 비어 있던 자리에 처음 값을 쓸 때 |

한 Transaction 안에서 같은 Stem에 속한 Value를 이어서 읽으면 두 번째부터는 200 Gas만 든다. [1편]({% post_url Paper-Review/2026-11-10-Verkle-Tree %})의 "Key: Stem and Suffix"에서 본 배치가 Gas 비용으로 이어지는 지점이다.

**2단계. Fork 시점 (EIP-7612)**

- 기존 MPT를 그대로 동결한다. 더 이상 쓰지 않고 읽기만 한다.
- Verkle Tree를 MPT 위에 겹쳐 놓는다. 이 구조를 Overlay Tree라 한다.
- 새로 쓰는 값은 모두 Verkle Tree에 들어간다.
- 값을 읽을 때는 Verkle Tree를 먼저 보고, 없으면 MPT에서 읽는다.
- Block Header의 State Root는 이 시점부터 MPT의 Root가 아니라 Verkle Tree의 Root다.

**3단계. 변환 (EIP-7748)**

Transaction이 건드린 값만 Verkle Tree로 넘어가기를 기다리기에는 State가 너무 크다. 그래서 Block마다 정해진 수 이하의 값을 MPT에서 Verkle Tree로 옮긴다.

- 옮기는 단위는 Storage Slot 하나, 또는 Account의 데이터와 Code다.
- Block의 Transaction을 실행하기 전에 수행한다.
- Account를 MPT의 Key 순서대로 처리하고, 각 Account 안에서는 Storage를 먼저 옮긴 뒤 Account의 데이터와 Code를 옮긴다.
- 그 사이 Transaction이 이미 새 값을 써 둔 자리는 덮어쓰지 않는다.
- Block당 옮기는 수의 상한(CONVERSION_STRIDE)과 시작 시점(CONVERSION_START_TIMESTAMP)은 EIP에 값이 정해져 있지 않다(TBD).

**4단계. 완료**

변환이 끝나면 Overlay Tree라는 구분을 없애고 모든 접근을 Verkle Tree로 처리한다.

```
Fork 이전     State Root = MPT의 Root
              읽기와 쓰기 모두 MPT

Fork 시점     State Root = Verkle Tree의 Root                    EIP-7612
              쓰기 → Verkle Tree
              읽기 → Verkle Tree, 없으면 동결된 MPT

변환 기간     Block마다 MPT의 값 일부를 Verkle Tree로 옮김        EIP-7748

변환 완료     읽기와 쓰기 모두 Verkle Tree
```

**⇒ 이 계획은 변환을 합의 과정 안에 넣는다. 모든 Node가 Block마다 같은 값을 같은 순서로 옮겨야 State Root가 일치한다.**

## Upgrade 일정에서의 위치: Osaka에 배정됐다가 빠짐

Roadmap 그림은 방향을 보여 줄 뿐이고, 실제 적용 여부는 Upgrade마다 어떤 EIP를 넣을지 정하는 개발자 회의에서 결정된다. 그 기록을 시간순으로 적는다.

| 시점 | 일어난 일 |
| --- | --- |
| 2024년 2월 1일 | All Core Developers Execution 회의에서 Verkle Tree를 Prague 다음 Upgrade인 Osaka에 넣기로 함. 반대는 없었음 ([Galaxy의 회의 정리](https://galaxy.com/insights/research/ethereum-all-core-developers-execution-call-180)) |
| 2024년 4월 11일 | 전용 시험 Network인 verkle-gen-devnet-6이 시작됨. ethPandaOps의 [문서](https://notes.ethereum.org/@ethpandaops/verkle-gen-devnet-6)는 EIP-2935, EIP-4762, EIP-6800을 활성 상태로, EIP-7545를 아직 활성화되지 않은 상태로 적음 |
| 2024년 10월 | Vitalik Buterin의 2024년 글이 Verkle Tree, STARK(Scalable Transparent Argument of Knowledge)와 보수적 Hash, STARK와 공격적 Hash 세 가지를 나란히 놓고 곧 결정해야 한다고 적음 |
| 2025년 1월 20일 | Hash 기반 Binary Tree 제안인 [EIP-7864](https://eips.ethereum.org/EIPS/eip-7864)가 작성됨. 저자 열 명이 EIP-6800과 같음 |
| 2025년 5월 7일 | Pectra 적용. EIP-2935 포함 |
| 2025년 6월 30일 | go-ethereum의 Guillaume Ballet이 Ethereum Community Conference에서 [Verkle Tree에서 Binary Tree로의 전환](https://ethcc.io/archives/completing-the-circle-transitioning-from-verkle-to-binary-trees-in-ethereum)을 발표함 |
| 2025년 12월 3일 | Fusaka 적용. Fusaka는 Execution Layer(실행 계층)의 Osaka와 Consensus Layer의 Fulu를 합친 이름임. 주요 변경은 PeerDAS(Peer Data Availability Sampling)와 Gas Limit 상향이고 Verkle Tree는 들어가지 않음 |
| 2026년 2월 18일 | Ethereum Foundation이 2026년 우선순위 글에서 State의 장기 방향을 [Binary Tree와 Statelessness](https://blog.ethereum.org/2026/02/18/protocol-priorities-update-2026)로 적음. 이 글에 Verkle Tree는 나오지 않음 |
| 2026년 3월 1일 | Vitalik Buterin이 State Tree를 Binary Tree로 바꾸고 장기적으로 EVM도 바꾸는 계획을 밝힘 ([The Block](https://www.theblock.co/post/391681/vitalik-buterin-lays-out-a-two-part-plan-to-overhaul-ethereums-execution-layer-from-the-ground-up)) |
| 2026년 3월 6일 | consensus-specs 저장소에서 EIP-6800 사양이 [제거](https://github.com/ethereum/consensus-specs/pull/4984)됨. 사유는 EIP가 Stagnant이고 가까운 Upgrade에 들어갈 가능성이 낮다는 것 |
| 2026년 6월 11일 | [EIP-8297](https://eips.ethereum.org/EIPS/eip-8297)(PBT)이 작성됨 |
| 2026년 7월 23일 | [EIP-8347](https://eips.ethereum.org/EIPS/eip-8347)(PBT로의 State 이전 절차)이 작성됨 |
| 2026년 8월 10일 | Vitalik Buterin이 2023년판 Roadmap과 Strawmap을 비교한 그림을 올림. Verkle Tree는 대체된 항목으로 분류됨 |
| 2026년 8월 13일 | Ethereum Foundation의 연구자 Justin Drake가 Foundation이 Layer 1에서 Poseidon Hash를 쓰지 않고 SHA나 BLAKE 같은 기존 Hash로 간다고 밝힘 ([보도](https://crypto.news/ethereum-l1-drops-poseidon-in-post-quantum-move/)) |
| 2026년 10월 29일 | 시험 Network glamsterdam-pbt-1에서 MPT를 PBT로 바꾸는 Fork가 적용됨. 공개된 측정 데이터는 10월 22일부터 31일까지의 구간임 |
| 2026년 11월 10일 | go-ethereum이 EIP-7864와 EIP-4762의 구현을 [제거](https://github.com/ethereum/go-ethereum/pull/36091)함. 하루 전에는 EIP-8297 구현이 들어감 |

- Verkle Tree를 폐기한다고 못 박은 단일 결정문은 확인하지 못했다. EIP-6800의 상태도 Withdrawn이 아니라 Stagnant다. 위 표의 변화가 쌓여 사실상 Roadmap에서 빠진 것이다.

- 표에서 보이는 순서는 이렇다. Osaka 배정(2024년 2월) → 대안 비교(2024년 10월) → Binary Tree 제안(2025년 1월) → Verkle Tree 없는 Fusaka(2025년 12월) → Foundation의 방향 명시와 사양 제거(2026년 2~3월) → 구현 교체(2026년 11월).

## Roadmap에서 빠진 이유

출처들이 드는 이유는 세 가지로 모인다. 각 이유가 어느 문서에 적혀 있는지 함께 적는다.

**이유 1. Quantum Resistance가 없어서 트리를 두 번 바꿔야 한다**

- Verkle Tree의 Commitment는 타원곡선의 Discrete Log 문제에 기댄다. 양자 컴퓨터는 이 문제를 풀 수 있다.
- 2024년 글은 MPT를 Verkle Tree로 바꾸면 나중에 트리를 다시 바꿔야 한다고 적는다.
- EIP-7864는 실용적인 양자 위협을 2030년대로 보는 전문가 추정과, 2030년까지 타원곡선 암호의 사용을 중단하라는 National Institute of Standards and Technology의 권고를 든다. Hash 함수에만 의존하는 설계가 더 오래 간다고 보고, 자신을 아마도 마지막 State Tree가 될 구조로 적는다.
- State Tree 교체는 앞의 "도입 계획"에서 본 대로 Chain 전체의 State를 옮기는 가장 무거운 종류의 변경이다. 이것을 Verkle Tree로 한 번, 양자 내성 구조로 또 한 번 하느니 한 번에 끝내자는 것이 핵심 논리다 (출처들의 서술을 묶은 해석).
- Strawmap은 장기 목표 다섯 가지 가운데 하나로 Hash 기반 방식을 통한 양자 내성을 든다. 타원곡선 기반인 Verkle Tree는 이 목표와 맞지 않는다.
- [The Block](https://www.theblock.co/post/391681/vitalik-buterin-lays-out-a-two-part-plan-to-overhaul-ethereums-execution-layer-from-the-ground-up)은 Verkle Tree가 2026년 Hard Fork의 유력 후보로 거론됐으나, 타원곡선 암호의 양자 취약성에 대한 우려로 2024년 중반부터 Binary Tree가 다시 주목받았다고 보도했다.

**이유 2. 증명 기술이 빨라져 Verkle Tree의 장점이 줄었다**

- Verkle Tree의 주된 장점은 작은 Proof를 빨리 만든다는 것이었다. Hash 기반 Binary Tree는 Proof 자체는 크지만, 그 Proof를 STARK로 증명해 대체하면 크기 문제가 사라진다. 걸림돌은 증명 속도였다.
- 2024년 글은 세 방식을 다음과 같이 비교한다. 2024년 10월 시점의 값이다.

| 방식 | Proof 크기 (Data 제외) | 암호학적 가정 | 최악의 경우 증명 시간 |
| --- | --- | --- | --- |
| Verkle Tree | 약 100~2,000 kB | 타원곡선. Quantum Resistance 없음 | 1초 미만 |
| STARK와 보수적 Hash (SHA256, BLAKE 등) | 약 100~300 kB | 보수적 Hash | 10초 초과 |
| STARK와 공격적 Hash (Poseidon, Ajtai 등) | 약 100~300 kB | 비교적 새롭고 검증이 덜 된 Hash | 1~2초 |

- EIP-7864는 증명 시스템의 발전으로 필요한 증명 속도에 가까워졌을 수 있다고 적는다. EIP-8297은 Proof를 충분히 빨리 만들 수 있어 Verkle Tree의 주된 장점에 맞먹는다고 적는다.
- 2024년 글은 Verkle Tree가 아닌 방식을 고르면 배포가 Hard Fork 한 번쯤 늦어진다는 점을 Verkle Tree 쪽의 근거로 들었다. 그런데 Verkle Tree 자신이 Osaka에 들어가지 못했으므로 이 근거도 약해졌다 (출처에 없는 해석).

**이유 3. 모든 실행을 SNARK로 증명한다는 목표가 State Tree를 고르는 기준이 됐다**

- The Verge의 최종 목표는 2022년부터 모든 실행을 SNARK로 검증하는 Ethereum이었다. 목표 자체는 새로 생긴 것이 아니다. 2023년판 그림에서 이 목표는 Verkle Tree 다음 단계였다. 달라진 것은 이 목표가 State Tree를 고르는 기준으로 쓰이게 됐다는 점이다.
- 실행을 증명하려면 State를 읽고 쓰는 과정, 곧 State Tree의 연산도 증명 안에 들어가야 한다.
- Verkle Tree의 Proof 검증은 256 Bit 연산이 많아 SNARK로 증명하기 어렵다(2024년 글).
- EIP-7864가 첫머리에 드는 동기는 Block을 Validity Proof로 증명한다는 장기 목표이고, 현재의 MPT가 여기에 맞지 않는다는 것이다. 그 이유로 Recursive Length Prefix 부호화, Keccak Hash, 트리 안에 트리가 들어 있는 구조, Code가 State에 포함되지 않는다는 점을 든다. EIP-8297도 같은 이유를 들고, Code에 대해서는 Bytecode의 일부를 효율적으로 증명할 수 없다고 적는다. Binary Tree에 증명하기 쉬운 Hash 함수를 쓰면 이 문제가 줄어든다.
- The Block의 보도에 따르면 Vitalik Buterin은 State Tree와 EVM이 증명 병목의 80% 이상을 차지한다고 보았다. Binary Tree는 Merkle Proof를 지금의 약 4분의 1로 줄이고, Hash 함수를 바꾸면 증명 효율이 3배에서 100배 더 나아진다고 했다. 약 4분의 1은 [1편]({% post_url Paper-Review/2026-11-10-Verkle-Tree %})의 "Path Size of One Key" 표에서 3,840 B가 1,024 B로 줄어드는 비율과 같다. 정확히는 3.75분의 1이다 (3,840 ÷ 1,024, 직접 계산한 값).
- Strawmap의 또 다른 장기 목표는 Zero-Knowledge EVM과 실시간 증명으로 Layer 1의 처리량을 초당 1 Gigagas까지 올리는 것이다.

**⇒ 세 이유는 하나로 이어진다. Verkle Tree는 "Stateless Client를 위한 작은 Witness"라는 문제 하나를 가장 잘 푸는 구조였다. "양자 컴퓨터에 안전할 것"과 "SNARK로 증명하기 쉬울 것"은 2023년판 그림에서 Verkle Tree 다음 단계의 조건이었다. 이 두 조건을 State Tree를 교체할 때 함께 충족하기로 하면서, 한 번의 전환으로 세 조건을 겨냥하는 방식, 곧 Hash 기반 Binary Tree와 그 Proof를 증명하는 SNARK가 선택됐다.**

잃은 것도 있다.

- Binary Tree만으로는 Witness가 Verkle Tree보다 크다. 2024년 글의 계산(Block Gas 30,000,000, Code를 Chunk로 나눠 트리에 넣는 경우)에서, 공격자가 Address를 골라 만드는 경우를 빼면 Binary Tree는 약 10.4 MB, Verkle Tree는 약 1.3 MB다. 공격자가 앞부분이 같은 Address를 만들어 Path를 길게 하면 Binary Tree는 Hash 약 660,000개, Verkle Tree는 약 2.6 MB가 된다. Hash 660,000개는 32 B로 환산하면 약 21 MB다 (직접 계산한 값). Verkle Tree의 값은 [1편]({% post_url Paper-Review/2026-11-10-Verkle-Tree %})의 "Toward Verkle Tree"에도 있다. 그래서 이 방향은 Proof를 증명 시스템으로 줄이는 것을 전제로 한다.
- Guillaume Ballet의 [발표 자료](https://notes.ethereum.org/@gballet/verkle-and-binary)는 Verkle Tree가 나은 점으로 저장할 내부 Node가 적다는 것과 Proof가 작다는 것을, Binary Tree가 나은 점으로 Hash 속도, Quantum Resistance, STARK와의 호환을 든다.
- 2024년 글이 적은 대로 공격적 Hash는 검증이 덜 됐다. 2026년 8월 13일의 Justin Drake의 발표는 Layer 1에서 공격적 Hash인 Poseidon 대신 기존 Hash를 쓰는 방향이다. EIP 문서상으로는 Hash 함수가 아직 확정되지 않았다.

## Verkle Tree 자리에 들어온 것: Binary Tree

Stateless Client라는 목표는 그대로이고, Proof를 줄이는 수단이 Vector Commitment에서 "Binary Tree와 그 Proof를 증명하는 SNARK"로 바뀌었다.

| EIP | 제목 | 내용 | 상태 |
| --- | --- | --- | --- |
| [EIP-7864](https://eips.ethereum.org/EIPS/eip-7864) | Ethereum state using a unified binary tree | 2025년 1월 제안. Account, Storage, Code를 Binary Tree 하나에 넣음 | Draft |
| [EIP-8297](https://eips.ethereum.org/EIPS/eip-8297) | Partitioned Binary Tree | 2026년 6월 제안. Key의 첫 Byte로 구역(Zone)을 나눔. 0x00은 Account Header, 0x01은 Code, 0xFF는 Storage | Draft |
| [EIP-8347](https://eips.ethereum.org/EIPS/eip-8347) | Offline State Migration to the PBT | 2026년 7월 제안. MPT에서 PBT로 State를 옮기는 절차 | Draft |

![eip-8297-tree.png](eip-8297-tree.png)

그림은 EIP-8297에 실린 트리 구조다.

- **이어받은 것**: Account, Storage, Code를 하나의 트리에 넣는 구조, Value를 256개 단위의 Stem으로 묶는 방식, Account Header에 기본 데이터와 Code Hash와 앞쪽 Storage Slot 64개를 함께 두는 배치(HEADER_STORAGE_OFFSET 64), Code를 31 Byte Chunk로 나누고 PUSH 데이터 Byte 수 1 Byte를 붙이는 방식은 EIP-6800에서 온 것이다. Verkle Tree를 위해 한 설계 가운데 Commitment 방식을 뺀 나머지가 대부분 남았다.
- **달라진 것**: 부모 Node는 두 자식의 Hash다. Key의 첫 Byte가 구역을 나타내고, Key 길이는 구역마다 다르다. Account Header와 Code의 Key는 34 Byte, Storage의 Key는 66 Byte다. Code는 Account가 아니라 Code Hash를 기준으로 저장해, 같은 Code를 가진 Contract들이 Chunk를 공유한다. 64번 이후의 Storage Slot은 EIP-6800에서는 같은 Account의 다른 Stem으로 갔고, EIP-8297에서는 전용 구역인 Storage 구역으로 간다.
- **정해지지 않은 것**: Hash 함수다. EIP-8297의 참조 구현은 BLAKE3를 쓰고, 문서는 BLAKE3, Keccak, Poseidon2를 후보로 적는다. 2026년 8월 13일 Ethereum Foundation의 연구자 Justin Drake는 Foundation이 Layer 1에서 Poseidon을 쓰지 않고 SHA나 BLAKE 같은 기존 Hash로 간다고 밝혔다([보도](https://crypto.news/ethereum-l1-drops-poseidon-in-post-quantum-move/)). [다른 보도](https://cointelegraph.com/news/ethereum-foundation-drops-poseidon-post-quantum-plan)도 이를 Foundation의 공식 결정문이 아니라 연구자의 X 게시글로 전한다. 확인한 두 보도는 이 방침이 State Tree의 Hash 함수에 적용되는지를 적지 않는다. EIP-8297 문서에는 2026년 11월 24일 기준 세 후보가 그대로 적혀 있다.
- **Gas 비용**: EIP-8297은 Gas 비용 체계를 바꾸지 않는다고 적는다. Gas 비용 조정은 별도 EIP로 다룰 예정이고 그 EIP는 아직 공개되지 않았다고 EIP-8347이 적는다.
- **EIP-7864와 EIP-8297의 관계**: go-ethereum은 2026년 11월 10일에 "EIP-8297의 PBT가 EIP-7864의 Unified Binary Tree를 대체한다"는 설명과 함께 EIP-7864와 EIP-4762의 구현을 [제거](https://github.com/ethereum/go-ethereum/pull/36091)했다. EIP 문서상으로는 둘 다 Draft다.
- **이름에 대한 주의**: PBT는 EIP-8297의 제목대로 Partitioned Binary Tree다. 2026년 8월의 [한 기사](https://www.cryptotimes.io/2026/08/10/buterin-recasts-ethereum-priorities-around-quantum-safety-and-privacy/)는 이를 Polynomial Binary Tree로 풀어 적었는데, EIP의 이름과 다르다.

전환 방식도 달라졌다.

| 항목 | Verkle Tree 계획 (EIP-7612, EIP-7748) | PBT 계획 (EIP-8347) |
| --- | --- | --- |
| 변환하는 곳 | 합의 과정 안. 모든 Node가 Block마다 일부씩 옮김 | 합의 과정 밖. Node가 직접 변환하거나 변환된 State의 사본(Snapshot)을 내려받아 검증함 |
| 기준 | MPT의 Key 순서대로 처음부터 끝까지 | 되돌릴 수 없게 확정(Finalize)된 Block 하나(Anchor Block)의 State 전체 |
| 최신 Block까지 따라잡기 | 변환과 실행이 함께 진행됨 | Block마다 바뀐 State를 기록한 Block-Level Access List(EIP-7928)를 재생함. 다시 실행하지 않음 |
| 교체 전 확인 | 없음. Fork와 함께 State Root가 바뀜 | 검증자들이 Block마다 PBT의 Root를 서명해 공개하는 기간을 둠. 합의에는 쓰이지 않는 관찰용임 |
| State Root 교체 | Fork 시점에 바로 교체하고 이후 변환 | 변환을 마친 뒤 Hard Fork 한 번에 교체. 그 Fork가 Finalize될 때까지 두 트리를 유지 |

- EIP-8347이 합의 과정 밖의 변환을 고른 이유는 세 가지다. 합의 규칙의 변경이 작고, State Root 하나가 항상 State 전체를 대표하고, 실행을 증명하는 쪽이 변환 단계까지 증명할 필요가 없다.
- 대가도 세 가지가 적혀 있다. 전환 기간에 두 트리를 함께 보관해야 한다(Mainnet 규모에서 약 300 GB 추가). 변환이 Chain 밖에서 일어나므로 합의가 변환의 정확성을 확인하지 못한다. Block마다 두 트리를 갱신해야 한다.

**⇒ Verkle Tree 계획에서 가장 무거웠던 "합의 과정 안의 변환"을 PBT 계획은 합의 과정 밖으로 뺐다.**

- **시험**: ethPandaOps의 시험 Network(glamsterdam-pbt-1)에서 Node 9개가 Chain을 운영하면서 MPT에서 PBT로 State를 옮겼다. Execution Client는 geth 3개, Nethermind 2개, Besu 2개, erigon 2개였고 PBT Fork는 2026년 10월 29일이었다. [측정 데이터 저장소](https://github.com/CPerezz/pbt-devnet-1-data)에 공개된 데이터는 10월 22일부터 31일까지의 구간이고, Network의 전체 운영 기간은 저장소에 적혀 있지 않다. 표본으로 잡은 Block 높이에서 Node 사이의 PBT Root 불일치는 없었다.

## 현재 일정

[ethereum.org의 Roadmap](https://ethereum.org/roadmap/)은 Upgrade를 시간순으로 보여 준다. State Tree와 관련된 내용만 옮기면 다음과 같다.

| Upgrade | 시점 | 상태 | State Tree 관련 내용 |
| --- | --- | --- | --- |
| Pectra | 2025년 5월 7일 | 적용됨 | EIP-2935 포함 |
| Fusaka | 2025년 12월 3일 | 적용됨 | 주요 변경에 없음 |
| Glamsterdam | 2026년 4분기로 표기 | 개발 중 | 주요 변경에 없음. 함께 들어가는 Block-Level Access List(EIP-7928)는 EIP-8347이 사용함 |
| Hegotá | 2027년 | 개발 중 | 주요 변경에 없음. 범위는 확정되지 않음 |

같은 문서는 앞으로 올 기술 과제의 하나로 Statelessness를 든다. State Tree를 무엇으로 바꾸는지는 이 문서에 적혀 있지 않다.

- **Glamsterdam**: 대표 변경은 Enshrined Proposer-Builder Separation(EIP-7732)과 Block-Level Access List(EIP-7928)다. Sepolia Testnet에는 2026년 10월 6일에 적용됐다. Hoodi Testnet 적용은 보류됐다. State를 새로 만드는 비용을 다시 매기는 EIP-8037이 기존 Contract에 미치는 영향을 확인하기 위해서다([보도](https://www.altcoinbuzz.io/ethereum-delays-glamsterdam-testnet-upgrade-over-smart-contract-risks)). [Forkcast](https://forkcast.org/)는 Mainnet 적용 시점을 미정(TBD)으로 표시하고 자체 추정치를 2027년 1월 27일로 둔다. 발표된 날짜가 아니라 추정이다. ethereum.org의 Roadmap은 2026년 4분기로 적는다.
- **Hegotá**: Forkcast는 목표를 2027년으로 적고 자체 추정치를 2027년 6월 16일로 둔다. 대표 변경은 Fork-Choice Enforced Inclusion List(EIP-7805)와 Frame Transaction(EIP-8141)으로 정해졌고 나머지 범위는 논의 중이다. EIP-7709는 포함을 검토하는 목록(Considered for Inclusion)에 있다.
- **State Tree 교체**: 어느 Upgrade에도 확정되어 있지 않다. Hegotá의 EIP 목록인 [EIP-8081](https://eips.ethereum.org/EIPS/eip-8081)에는 확정, 검토, 제안, 거절 어느 목록에도 EIP-8297, EIP-8347, EIP-7864, EIP-6800이 없다. EIP-8347은 교체가 일어날 Fork에 이름(PBT_ACTIVATION_FORK)만 붙이고 시점은 정하지 않는다. 시점은 이 제안을 포함하게 될 Upgrade의 EIP 목록 문서(Hard Fork Meta EIP)가 정한다고 적는다.
- **예상**: EIP-8347의 저자 세 명 가운데 Maria Silva와 Carlos Perez가 Ethereum Research에 쓴 2026년 9월 [의견 글](https://ethresear.ch/t/how-hegota-can-influence-the-state-roadmap/25895)은 PBT로의 이전을 Hegotá 다음 Fork인 I\*로 예상한다. 공식 일정이 아니라 저자들의 예상이다. Maria Silva가 관리하는 [계획 문서](https://github.com/misilva73/pbt-planning)는 사양, 시험, Client별 구현 상황을 모아 둔다.
- **Roadmap 초안**: Ethereum Foundation은 [Strawmap](https://strawmap.org)이라는 Roadmap 초안을 그림으로 관리한다. 2029년까지 약 6개월 간격으로 Fork 일곱 개를 그리고, Glamsterdam과 Hegotá 다음 Fork는 I\*, J\* 같은 임시 이름으로 적는다. 문서 스스로 공식 계획이나 예측이 아니라 조율을 위한 초안이라고 밝힌다. 그림 안에서 State Tree가 놓인 위치는 직접 확인하지 못했다.

주의할 점은 다음과 같다.

- [일부 기사](https://cryptobriefing.com/ethereum-partitioned-binary-tree-eip-8297/)는 PBT의 Mainnet 적용을 2026년 4분기로 적는다. 이는 Glamsterdam의 목표 시점과 섞인 것으로 보인다(추정). Glamsterdam의 범위에 State Tree 교체는 없다.
- [ethereum.org의 Verkle Tree 문서](https://ethereum.org/roadmap/verkle-trees/)는 2026년 6월에 마지막으로 고쳐졌는데도 Binary Tree를 언급하지 않고, Verkle Tree를 여전히 Stateless Client로 가는 단계로 설명한다. [Statelessness 문서](https://ethereum.org/roadmap/statelessness/)도 같다. 같은 Verkle Tree 문서는 MPT를 Leaf를 둘씩 묶어 Hash하는 트리로 설명하지만 MPT의 Width는 16이다. Witness 크기가 Leaf 개수와 무관하게 고정이라고도 적지만 고정인 것은 Multiproof의 크기다.

<br />

## 결론

Roadmap에서 Verkle Tree는 The Verge의 첫 단계였고, 2024년 2월에는 Osaka Upgrade에 배정됐다. 그러나 2026년 11월 24일 기준 Ethereum은 Verkle Tree를 채택하지 않았다. 타원곡선에 기반한 Commitment에 Quantum Resistance가 없어 트리를 두 번 바꿔야 한다는 점, 증명 시스템의 발전으로 Hash 기반 트리의 약점이 줄었다는 점, 모든 실행을 SNARK로 증명한다는 목표가 State Tree를 고르는 기준이 되어 증명하기 쉬운 트리가 필요해졌다는 점이 이유다.

그 자리에는 Hash 기반 Binary Tree(EIP-7864를 거쳐 EIP-8297)가 들어왔다. 전환은 합의 과정 밖에서 State를 옮긴 뒤 Hard Fork 한 번에 State Root를 바꾸는 방식(EIP-8347)으로 계획되어 있고, 적용할 Upgrade는 아직 정해지지 않았다. Stateless Client라는 목표와, 함께 읽는 Value를 같은 Stem에 묶는 설계는 이어진다.
