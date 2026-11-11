---
title: "Verkle Tree"
date: 2026-11-11
categories: [Paper Review]
tags: [Blockchain, Ethereum, Verkle Tree, Merkle Patricia Trie, Stateless Client, Commitment]
math: true
---

## What is Verkle Tree?

Verkle Tree(버클 트리)는 Merkle Tree(머클 트리)의 부모 노드를 Hash(해시) 대신 **Vector Commitment(벡터 커밋먼트)**로 만든 트리다. 이름은 Vector Commitment와 Merkle Tree를 합친 것이다. 구조는 Merkle Tree와 같지만, 값 하나가 트리에 들어 있음을 보이는 증명이 훨씬 작다. 2018년 John Kuszmaul이 처음 제안했고, 이더리움은 2021년부터 상태 트리를 Verkle Tree로 바꾸는 작업을 진행했다.

이 글은 이더리움이 왜 상태 트리를 바꾸려 했는지, 현재 구조인 Merkle Patricia Trie(머클 패트리샤 트라이)의 증명이 왜 큰지, Verkle Tree가 그것을 어떻게 줄이는지를 순서대로 설명한다. 여러 증명을 하나로 묶는 Multiproof(다중 증명)는 [별도 글](/posts/multiproof/)에서 다루므로 여기서는 결과만 쓴다.

2026년 11월 기준 이더리움 메인넷은 Verkle Tree를 채택하지 않았다. 로드맵의 무게는 Hash 기반 Binary Tree(이진 트리)로 옮겨 갔다. 자세한 내용은 마지막 장에 있다.

## Why Ethereum Needs a Smaller Proof

### State and Witness

이더리움의 **State(상태)**는 모든 계정의 잔액, Nonce, 컨트랙트 코드, 스토리지를 합친 데이터다. 상태 전체는 **State Root(상태 루트)**라는 값 하나로 요약되어 블록 헤더에 들어간다. 어떤 값이 State Root가 가리키는 상태에 들어 있음을 보이는 데이터가 **Proof(증명)**다.

블록을 검증하는 클라이언트는 블록의 트랜잭션을 모두 다시 실행하고, 로컬 상태 트리를 갱신한 뒤, 계산한 State Root가 블록에 적힌 값과 같은지 비교한다. 트랜잭션이 어떤 계정과 스토리지를 읽을지 미리 알 수 없으므로 상태 전체를 로컬에 보관해야 한다. Vitalik Buterin의 [2024년 글](https://vitalik.eth.limo/general/2024/10/23/futures4.html)은 클라이언트가 저장하는 상태를 수백 GB로, 증가 속도를 연 약 30 GB로 적었다. 저장 부담이 커질수록 일반 하드웨어로 검증에 참여하기 어려워진다.

이 문제를 푸는 방향이 **Stateless Client(상태 비저장 클라이언트)**다. 상태 데이터베이스를 저장하지 않고, 블록과 함께 받은 **Witness(위트니스)**로 블록을 검증한다. Witness는 블록의 트랜잭션을 실행하는 데 필요한 상태 조각과, 그 조각이 State Root에 포함됨을 보이는 Proof를 묶은 것이다.

### The Requirement

Witness는 블록과 함께 네트워크로 전파되므로, 12초 Slot 안에 받아서 처리할 수 있을 만큼 작아야 한다. 현재 구조인 Merkle Patricia Trie로는 이 조건을 맞추기 어렵다. 원문이 제시한 수치는 다음과 같다.

| 항목 | 수치 | 가정 | 출처 |
| --- | --- | --- | --- |
| 접근 1회당 Witness | 평균 약 3 kB, 최악 약 3배 | | [EIP-6800](https://eips.ethereum.org/EIPS/eip-6800) |
| 블록 Witness | 약 18 MB | 블록당 접근 6,000회 | [EIP-6800](https://eips.ethereum.org/EIPS/eip-6800) |
| 블록 Witness, 최악 | 약 330 MB | Gas Limit 30M, 접근마다 컨트랙트 코드 24,000 B 포함 | [Vitalik 2024](https://vitalik.eth.limo/general/2024/10/23/futures4.html) |

수치는 각 글이 쓰인 시점의 Gas Limit을 전제로 하므로 Gas Limit이 바뀌면 비례해서 바뀐다.

Verkle Tree가 겨냥하는 것은 Weak Statelessness(약한 상태 비저장)다. 블록을 만드는 쪽은 Witness를 생성해야 하므로 여전히 전체 상태를 가지고, 나머지 노드만 상태 없이 검증한다. 모든 노드가 상태를 버리는 Strong Statelessness(강한 상태 비저장)와는 다르다. [ethereum.org](https://ethereum.org/en/roadmap/statelessness/)에 두 개념이 정리되어 있다.

정리하면 풀어야 할 문제는 "상태 트리가 만드는 Proof의 크기"다. Proof 크기는 트리 구조가 결정하므로 트리 구조를 바꿔야 한다.

## Merkle Patricia Trie

현재 이더리움의 상태 트리는 Merkle Patricia Trie(MPT)다. 두 구조를 합친 것이다.

- **Merkle Tree:** 부모 노드가 자식 노드들의 Hash를 합쳐 다시 Hash한 값을 가진다. 데이터 전체가 Merkle Root 하나로 요약된다.
- **Patricia Trie(패트리샤 트라이):** Key를 한 자리씩 따라 내려가는 트리다. 분기가 없는 구간은 노드 하나로 묶어 경로를 줄인다.

MPT는 Patricia Trie의 각 노드를 Hash로 참조하는 구조다. 노드를 Recursive Length Prefix(RLP)로 인코딩한 뒤 keccak256으로 Hash한 값이 그 노드의 참조가 된다. 자세한 명세는 [ethereum.org](https://ethereum.org/en/developers/docs/data-structures-and-encoding/patricia-merkle-trie)에 있다.

### Nodes

Key는 4비트 단위인 Nibble(니블)로 나뉘어 경로가 된다. Nibble 하나는 16진수 한 자리다.

| 노드 | 형태 | 역할 |
| --- | --- | --- |
| Branch Node | 17개 항목 `[v0 ... v15, vt]` | Nibble 0~f에 해당하는 자식 16개와 값 1개 |
| Extension Node | 2개 항목 `[encodedPath, key]` | 분기 없는 공통 경로와 다음 노드의 참조 |
| Leaf Node | 2개 항목 `[encodedPath, value]` | 남은 경로와 값 |

상태 트리에서 Key는 keccak256(address)이고 경로 길이는 64 Nibble이다. 값은 계정의 Nonce, Balance, StorageRoot, CodeHash다. 컨트랙트 스토리지는 계정마다 별도의 트리에 들어가고, 코드는 트리 밖에 있으며 CodeHash만 트리에 남는다.

### Example

설명을 위해 Key를 2바이트로 줄인 예시다. 실제 Key는 32바이트다.

| 계정 | Key | 값 |
| --- | --- | --- |
| a1 | 0x7340 | Nonce 2, Balance 4 ETH |
| a2 | 0x7bc6 | Nonce 3, Balance 2 ETH |
| a3 | 0xd563 | Nonce 4, Balance 5 ETH |

Root는 Branch Node다. a1과 a2는 첫 Nibble이 7로 같으므로 Slot 7 아래로 함께 내려간다. Slot 7 아래에 Branch Node가 하나 더 있고, 두 번째 Nibble이 3과 b로 갈리므로 각각 Leaf Node가 된다. a1의 Leaf는 남은 경로 40을, a2의 Leaf는 남은 경로 c6을 가진다. a3은 첫 Nibble d를 혼자 쓴다. Root의 Slot d가 바로 Leaf Node를 가리키고, Leaf는 남은 경로 563을 가진다.

### Verification

검증자는 Leaf에서 Root까지 Hash를 다시 계산하고, 결과가 알고 있는 State Root와 같은지 확인한다. Branch Node의 Hash를 계산하려면 그 노드의 모든 Slot 값이 필요하다. 따라서 경로 위의 Branch Node마다 나머지 Slot의 Hash, 즉 **형제 노드의 Hash**가 Proof에 들어간다.

위 예시에서 a1을 검증하려면 a1의 Leaf Node, 두 번째 Branch Node의 형제인 a2 Leaf의 Hash, Root의 형제인 a3 Leaf의 Hash가 필요하다. 이 예시는 Slot이 거의 비어 있어 형제가 Level당 1개뿐이다. 실제 상태 트리의 상위 Level은 Slot이 거의 차 있으므로 Level당 최대 15개의 형제 Hash가 들어간다.

### Proof Size

트리에 저장된 Key-Value 쌍의 개수를 $N$, 한 노드가 가질 수 있는 자식의 최대 개수(Width)를 $k$라 하면 깊이는 약 $\log_k N$이다. Level마다 형제 Hash가 최대 $k-1$개 필요하다.

$$
\text{Proof Size} \approx 32\,\text{B} \times (k-1) \times \log_k N
$$

$k=16$이면 $32 \times 15 \times \log_{16} N = 120 \times \log_2 N$ 바이트다. $N=2^{32}$일 때 약 3,840 B다. Width를 키우면 깊이는 줄지만 형제 수가 더 빨리 늘어난다. Hash 기반 트리에서는 Width 2가 Proof 크기에 가장 유리하고, Width 16은 비효율적이다. 이 계산은 Vitalik Buterin의 [Verkle trees](https://vitalik.eth.limo/general/2021/06/18/verkle.html) 글에 있다.

값이 바뀌면 그 Leaf와, Leaf에서 Root까지 경로 위에 있는 노드의 Hash만 다시 계산한다.

## Key Idea

Hash는 입력 전체를 알아야 다시 계산할 수 있다. 그래서 MPT의 검증자는 부모의 모든 자식 값을 받아야 한다.

Vector Commitment는 다르다. 값의 목록 $(v_0, \ldots, v_{n-1})$ 전체를 Commitment(커밋먼트) 하나로 만들고, "이 Commitment의 $i$번째 값은 $v_i$다"를 나머지 값 없이 짧은 증명으로 보일 수 있다. 부모 노드를 자식들의 Vector Commitment로 만들면, 검증자는 **경로 위의 Commitment와, 각 부모-자식 관계에 대한 증명만** 받으면 된다. 형제 노드는 필요 없다.

형제를 보내지 않으므로 Width를 키워도 Level당 증명 데이터가 늘지 않는다. Width를 키우면 깊이만 줄어든다. 이더리움에 제안된 Verkle Tree의 Width는 256이다. Width의 상한은 증명 생성 비용이 정한다. 증명 생성에는 노드마다 Width에 비례하는 연산이 든다.

| 항목 | Merkle Patricia Trie | Verkle Tree (이더리움 제안) |
| --- | --- | --- |
| 부모 노드의 값 | 자식들의 Hash | 자식들에 대한 Vector Commitment |
| Width | 16 | 256 |
| 깊이 | 약 $\log_{16} N$ | 약 $\log_{256} N$ |
| Level당 증명 데이터 | 형제 Hash 최대 15개 (480 B) | Commitment 1개 (32 B) |
| 여러 Key를 검증할 때 | Key마다 형제 Hash 추가 | 증명을 Multiproof 1개로 묶음 |

## Commitment Scheme

Verkle Tree가 Commitment에서 쓰는 성질은 두 가지다. 한 번 만든 Commitment는 다른 값으로 열 수 없다는 Binding(구속성)과, 특정 위치의 값을 짧은 증명으로 열 수 있다는 점이다. 이더리움의 상태 값은 공개 데이터이므로 값을 감추는 Hiding(은닉성)은 목적이 아니다. 실제로 이더리움 제안의 Commitment 식에는 값을 감추기 위한 난수 항이 없다.

### Vector Commitment from Polynomial Commitment

Vector Commitment를 만드는 한 방법이 Polynomial Commitment(다항식 커밋먼트)다. 값 $v_0, \ldots, v_{n-1}$에 대해 $f(i)=v_i$를 만족하는 다항식을 Lagrange Interpolation(라그랑주 보간)으로 만들면, "$i$번째 값이 $v_i$다"는 "$f(i)=v_i$다"와 같은 말이 된다. 다항식을 Commit하고 특정 점에서 여는 방법은 [Multiproof 글](/posts/multiproof/)의 앞부분에 있다.

### Pedersen Commitment

이더리움 제안이 쓰는 Vector Commitment는 Pedersen Commitment(페더슨 커밋먼트)다. 값 $v_i$를 고정된 타원곡선 점 $G_i$에 곱해 모두 더한 점이다.

$$
C = v_0 G_0 + v_1 G_1 + \cdots + v_{255} G_{255}
$$

기저점 $G_i$들 사이의 Discrete Log(이산 로그) 관계가 알려져 있지 않으면 Binding이 성립한다. Trusted Setup(신뢰 설정)이 필요 없다. 특정 위치를 여는 증명에는 IPA(Inner Product Argument, 내적 증명)를 쓴다.

이 Commitment는 덧셈에 대해 동형(Additively Homomorphic)이다. 두 Commitment를 더하면 두 벡터의 합에 대한 Commitment가 된다. 이 성질이 트리의 업데이트와 Multiproof를 가능하게 한다.

### What Ethereum Chose

- Commitment: Bandersnatch 곡선 위의 Pedersen Commitment, 기저점 256개. [EIP-6800](https://eips.ethereum.org/EIPS/eip-6800)
- Opening: IPA. [Ethereum Foundation 블로그](https://blog.ethereum.org/2021/12/02/verkle-tree-structure/)
- 한 Slot에 넣을 수 있는 값: Bandersnatch의 Scalar Field가 253비트 소수이므로 252비트까지
- Commitment 크기: 32 B

Vitalik Buterin과 Dankrad Feist의 설명 글은 KZG(Kate-Zaverucha-Goldberg) Commitment 표기를 쓴다. KZG는 Trusted Setup과 Pairing(페어링)이 필요하지만 Opening 증명이 군 원소 하나다. Pedersen과 IPA는 Trusted Setup이 필요 없는 대신 Opening 증명이 $2\log_2 256 = 16$개의 군 원소이고 검증 비용이 Width에 비례한다. Multiproof로 Opening을 하나로 묶어 한 번만 검증하므로 이 비용은 감당할 수 있다. 두 스킴의 비교는 Dankrad Feist의 [Inner Product Arguments](https://dankradfeist.de/ethereum/2021/07/27/inner-product-arguments.html)에 있다.

## Structure

이더리움 제안은 계정, 스토리지, 코드를 모두 하나의 Key-Value 트리에 넣는다. Key와 Value는 모두 32바이트다. 이 장은 [EIP-6800](https://eips.ethereum.org/EIPS/eip-6800)의 현재 본문을 기준으로 한다.

### Key: Stem and Suffix

Key 32바이트는 앞 31바이트인 Stem과 마지막 1바이트인 Suffix로 나뉜다.

- **Stem:** 트리 안에서의 경로를 정한다. Address와 Tree Index를 합쳐 Pedersen Hash한 값의 앞 31바이트다.
- **Suffix:** 같은 Stem 아래에 있는 256개 값 중 하나를 고른다.

Address 자체가 경로가 되는 것이 아니다. Stem이 Hash의 결과이므로 Key는 트리 전체에 고르게 퍼진다.

### Values Under One Stem

계정의 첫 번째 Stem(Tree Index 0)에는 자주 함께 읽는 값이 모여 있다.

| Suffix | 내용 | 상수 |
| --- | --- | --- |
| 0 | Version, Code Size, Nonce, Balance를 한 값에 묶음 | BASIC_DATA_LEAF_KEY = 0 |
| 1 | Code Hash | CODE_HASH_LEAF_KEY = 1 |
| 64~127 | 스토리지 Slot 0~63 | HEADER_STORAGE_OFFSET = 64 |
| 128~255 | 코드의 앞부분 Chunk | CODE_OFFSET = 128 |

나머지 스토리지 Slot과 코드 Chunk는 다른 Tree Index의 Stem으로 간다. 코드는 31바이트 Chunk로 나뉘어 트리에 들어간다. MPT에서는 코드가 트리 밖에 있어, 코드 접근을 증명하려면 코드 전체를 Witness에 넣어야 했다.

같은 Stem에 속한 값은 경로를 공유한다. Nonce와 Balance, 앞쪽 스토리지 Slot을 함께 읽어도 경로 위의 Commitment는 한 번만 Witness에 들어간다. EIP-6800은 이를 Gas 비용에도 반영해, 같은 그룹 안에서 인접한 Slot에 접근하는 비용을 2,100에서 200으로 낮춘다.

### Nodes

| 노드 | Commitment의 대상 | 역할 |
| --- | --- | --- |
| Inner Node | 자식 256개의 Commitment | Stem을 1바이트씩 따라 분기. MPT의 Branch Node에 해당 |
| Extension Node | (1, Stem, C1, C2) | 하나의 Stem과 그 아래 값 256개를 대표 |

Extension Node는 값을 두 Commitment로 나눠 담는다. C1은 Suffix 0~127의 값에 대한 Commitment이고, C2는 Suffix 128~255의 값에 대한 Commitment다. 나누는 이유는 Slot 크기다. Commitment의 한 Slot에는 252비트까지만 들어가는데 값은 32바이트(256비트)다. 그래서 값 하나를 16바이트 두 조각으로 나눠 Slot 두 개에 넣는다. 값 128개가 Slot 256개를 채우므로 Commitment가 두 개 필요하다. 아래쪽 조각에는 $2^{128}$을 더해, 값이 0인 경우와 값이 없는 경우를 구분한다.

부모가 자식 Commitment를 담을 때도 같은 제약이 있다. Commitment는 곡선 위의 점이므로, Scalar 값으로 변환한 뒤 부모의 Slot에 넣는다.

### Depth

랜덤한 Key $n$개를 Width $d$인 트리에 넣으면 평균 깊이는 $\log_d n + 2/3$이다. Dankrad Feist의 [Verkle trie for Eth1 state](https://dankradfeist.de/ethereum/2021/06/18/verkle-trie-for-eth1.html)에 유도가 있다.

### Example

MPT 예시를 Width 256으로 옮기면 다음과 같다. 설명을 위해 Key의 첫 바이트를 그대로 경로로 쓴다. 실제로는 Hash로 만든 Stem을 쓴다.

- Root는 Slot 256개를 가진 Inner Node다.
- a1(0x7340)은 Slot 0x73, a2(0x7bc6)는 Slot 0x7b, a3(0xd563)은 Slot 0xd5로 간다.
- 세 Key가 첫 바이트에서 모두 갈리므로 Root 바로 아래가 Leaf다.

MPT에서는 a1과 a2가 첫 Nibble 7을 공유해 Level이 하나 더 필요했다. Width가 넓어지면 같은 Key 집합에서 깊이가 줄어든다.

## Updating the Tree

값이 바뀌면 Leaf에서 Root까지 경로 위의 Commitment만 갱신한다. 각 Commitment는 처음부터 다시 계산하지 않고 바뀐 분량만 더한다.

Commitment는 각 값에 기저점을 곱해 더한 점이다. $i$번째 값이 $v_i$에서 $v_i'$로 바뀌면 새 Commitment는 다음과 같다.

$$
C' = C + (v_i' - v_i)\,G_i
$$

나머지 255개 값은 읽을 필요가 없다. 타원곡선 곱셈 1회와 덧셈 1회로 끝난다.

경로를 따라 올라가는 순서는 다음과 같다.

1. Leaf의 값이 바뀌면 Extension Node의 C1 또는 C2를 위 식으로 갱신한다.
2. C1 또는 C2가 바뀌었으므로 Extension Node의 Commitment를 같은 식으로 갱신한다.
3. 부모 Inner Node는 자식 Commitment를 Scalar로 바꾼 값을 Slot에 담고 있다. 새 Scalar와 이전 Scalar의 차이를 구해 부모를 같은 식으로 갱신한다.
4. Root까지 3을 반복한다.

Commitment 1개를 갱신할 때마다 타원곡선 곱셈이 1회 든다. 값 하나를 고치는 데 Root를 포함해 약 4회가 필요하다. 기저점의 배수를 미리 계산해 두면 더 빨라진다.

예를 들어 계정 a1~a5가 있는 트리에서 a1이 a4에게 3 ETH를 보내면, a1의 Nonce와 Balance, a4의 Balance가 바뀐다. 두 Leaf에서 Root까지의 Commitment가 갱신되고, 두 경로가 만나는 지점부터 Root까지는 두 변경을 한 번에 반영한다. a2, a3, a5의 Leaf와 그 위의 다른 가지는 그대로다.

트리의 노드가 저장하는 것은 Commitment다. Opening 증명은 노드에 저장하지 않고, 블록이 접근한 값에 대해 Witness를 만들 때 계산한다.

## Verification

검증자는 Root Commitment를 이미 알고 있다. 경로의 각 Level에서 다음 관계를 확인한다.

- 부모 Commitment를 Index $z$에서 열면 값이 $y$다.
- $z$는 Key가 정하는 자식의 위치다.
- $y$는 자식 Commitment를 Scalar로 바꾼 값이다. 마지막 Level에서는 저장된 값이다.

위 예시에서 a1을 검증한다면 Opening이 두 개 필요하다. Root Commitment를 Index 0x73에서 열어 a1 Leaf의 Commitment를 확인하고, Leaf Commitment를 열어 Nonce와 Balance를 확인한다. a2와 a3의 Commitment는 필요하지 않다.

블록 하나는 수천 개의 값에 접근하므로 Opening이 (Key 개수) × (깊이)만큼 생긴다. Opening마다 증명을 따로 만들면 증명 데이터가 그만큼 커진다. Multiproof는 이 Opening들을 **크기가 고정된 증명 하나**로 묶는다. 동작 원리는 [Multiproof 글](/posts/multiproof/)에 있다.

Multiproof를 적용한 Witness는 다음으로 이루어진다.

- 접근한 Key와 Value
- 각 경로 위의 Commitment. Root는 제외하고, 경로가 겹치면 한 번만
- Multiproof 하나

$z$와 $y$는 보내지 않는다. 검증자가 Key, Value, 경로의 Commitment로부터 직접 계산한다. Multiproof의 크기는 Key 개수와 무관하게 고정이며, 스킴에 따라 128~1,000 B다. 따라서 Key가 늘어날 때 Witness에서 커지는 부분은 경로 위의 Commitment뿐이다.

검증 비용은 크기 $N$의 타원곡선 Multiexponentiation 1회가 대부분이다. Multiexponentiation은 여러 점에 각각 Scalar를 곱해 모두 더하는 연산이다. 값 수천 개에 100 ms 미만으로 추정된다. 증명 생성 쪽은 경로 위 각 노드의 자식 256개가 모두 필요하다. 이더리움 설계에서는 전체 상태를 가진 블록 생성자가 Witness를 만든다. 수치는 Vitalik Buterin의 [Verkle trees](https://vitalik.eth.limo/general/2021/06/18/verkle.html)에 있다.

## Proof Size

같은 조건에서 Verkle Proof는 Hexary MPT Proof보다 20~30배 이상 작고, 이상적인 Binary Merkle Proof보다 6~8배 작다.

### One Key

$N=2^{32}$, Hash와 Commitment를 32 B로 놓은 경우다. 식은 Vitalik Buterin의 [2024년 글](https://vitalik.eth.limo/general/2024/10/23/futures4.html)에 있고, Verkle Tree의 128 B는 그 식에 $N$을 대입해 계산한 값이다.

| 트리 | 식 | $N=2^{32}$일 때 |
| --- | --- | --- |
| Hexary MPT | $32 \times 15 \times \log_{16} N$ | 3,840 B |
| Binary Merkle Tree | $32 \times \log_2 N$ | 1,024 B |
| Verkle Tree (Width 256) | $32 \times \log_{256} N$ | 128 B |

### Worst Case per Block

Gas Limit 30M, 접근 1회당 2,400 Gas를 가정한 계산이다. 블록당 최대 12,500회 접근에 해당한다.

| 트리 | 최악의 Witness | 비고 |
| --- | --- | --- |
| Hexary MPT | 약 330 MB | 코드가 트리 밖에 있어 접근마다 코드 전체(최대 24,000 B) 포함 |
| Binary Merkle Tree, 코드를 트리에 포함 | 약 10.4 MB | |
| Verkle Tree | 약 1.3 MB | 공격자가 긴 공통 Prefix를 가진 Address를 만들면 약 2.6 MB |

### Many Keys

Width 256, Commitment와 증명이 48 B인 KZG, Key가 고르게 분포한 트리를 가정한 값이다. 단위는 바이트다. 출처는 [Verkle trees](https://vitalik.eth.limo/general/2021/06/18/verkle.html)다.

| 트리 크기 | Key 1개 | Key 1,000개 | Key 10,000개 |
| --- | --- | --- | --- |
| $2^{24}$ | 272 | 59,792 | 457,616 |
| $2^{32}$ | 320 | 107,744 | 937,472 |

Key 1,000개일 때 Key당 크기가 Key 1개일 때보다 훨씬 작다. 상위 Level의 Commitment를 여러 Key가 공유하고, Multiproof가 하나뿐이기 때문이다. 원문은 두 가지 보정을 덧붙인다. 실제처럼 Key가 랜덤하게 분포하면 깊이가 약 0.6 늘어 Key당 약 30 B가 더 들고, 32 B Commitment를 쓰면 크기가 약 3분의 1 줄어든다.

[ethereum.org](https://ethereum.org/roadmap/verkle-trees)는 값 1,000개 기준으로 MPT Witness 약 3.5 MB(7 Level), Verkle Witness 약 150 kB(4 Level)를 든다. [EIP-6800](https://eips.ethereum.org/EIPS/eip-6800)은 MPT 접근당 평균 약 3 kB, Verkle Tree 계정당 평균 약 200 B로 적는다.

## Limitations and Current Status

2026년 11월 11일 기준, 이더리움 메인넷의 상태 트리는 여전히 MPT다. Verkle Tree는 테스트넷 단계까지 진행됐고, 이후 논의의 중심은 Hash 기반 Binary Tree를 제안하는 EIP-7864로 옮겨 갔다.

### Limitations

- **양자 내성이 없다.** Commitment가 타원곡선에 기반하므로 양자 컴퓨터에 취약하다. 도입하더라도 나중에 다시 교체해야 한다.
- **SNARK 안에서 증명하기 어렵다.** Verkle Proof 검증에는 256비트 연산이 많다. 이를 Succinct Non-interactive Argument of Knowledge(SNARK)로 감싸려면 큰 오버헤드나 전용 구조가 필요하다.
- **최악의 Witness는 여전히 MB 단위다.** 위 표에서 약 1.3~2.6 MB다.
- **Width에 상한이 있다.** 증명 생성 비용이 Width에 비례한다.

위 한계를 지적한 Vitalik Buterin의 [2024년 글](https://vitalik.eth.limo/general/2024/10/23/futures4.html)은 Verkle Tree의 장점도 세 가지 든다. 증명 생성이 빠르고, 구현이 성숙했고, Witness를 다음 블록에 맞춰 갱신할 수 있다.

### Alternative: STARKed Binary Hash Tree

Binary Merkle Tree를 쓰고, Merkle Proof 자체를 Scalable Transparent Argument of Knowledge(STARK)로 증명해 Proof를 대체하는 방식이다. 타원곡선을 쓰지 않고 Hash에만 의존한다. 아래는 2024년 10월 시점의 비교다.

| 방식 | Proof 크기 | 보안 가정 | 최악의 증명 생성 시간 |
| --- | --- | --- | --- |
| Verkle Tree | 데이터 + 약 100~2,000 kB | 타원곡선 (양자 내성 없음) | 1초 미만 |
| STARK + 보수적 Hash (SHA256, BLAKE) | 데이터 + 약 100~300 kB | 보수적 Hash 함수 | 10초 초과 |
| STARK + 공격적 Hash (Poseidon, Ajtai) | 데이터 + 약 100~300 kB | 덜 검증된 새 Hash 함수 | 1~2초 |

### Timeline

| 시점 | 내용 |
| --- | --- |
| 2021년 6월 | Vitalik Buterin과 Dankrad Feist가 Verkle Tree와 Multiproof 설명 글을 공개했다. |
| 2023년 3월 | [EIP-6800](https://eips.ethereum.org/EIPS/eip-6800)이 작성됐다. |
| 2024년 10월 | Vitalik Buterin이 Verkle Tree와 STARKed Binary Hash Tree를 비교하고 선택이 필요하다고 적었다. |
| 2025년 1월 | EIP-7864(Unified Binary Tree)가 제안됐다. 제안 동기는 양자 내성 우려와 SNARK, STARK 증명 시스템의 발전이다. [논의 스레드](https://ethereum-magicians.org/t/eip-7864-ethereum-state-using-a-unified-binary-tree/22611) |
| 2026년 3월 | Vitalik Buterin이 Hexary keccak MPT를 EIP-7864의 Binary Tree로 바꾸는 안을 설명했다. [The Block](https://www.theblock.co/post/391681/vitalik-buterin-lays-out-a-two-part-plan-to-overhaul-ethereums-execution-layer-from-the-ground-up)은 Verkle Tree가 2026년 하드포크 후보로 거론됐으나, 양자 컴퓨팅 우려로 2024년 중반부터 Binary Tree가 다시 주목받았다고 보도했다. |

### What Was Confirmed

- EIP-6800의 상태는 Stagnant다. EIP 페이지에서 직접 확인했다.
- EIP-7864의 상태는 Draft로 알려져 있다. EIP 원문 페이지는 직접 열지 못했고, 논의 스레드와 2차 자료로 확인했다.
- [ethereum.org](https://ethereum.org/roadmap/verkle-trees)의 Verkle Tree 문서는 2026년 6월 24일에 마지막으로 갱신됐고, Binary Tree를 언급하지 않는다. 테스트넷으로 Verkle Gen Devnet 6을 든다.
- 상태 트리 변경이 어느 하드포크에 들어갈지는 확정된 자료를 찾지 못했다.

Stateless Client라는 목표와 설계의 일부는 그대로 이어진다. EIP-7864는 계정 데이터, 스토리지 Slot, 코드 Chunk를 묶어 담는 방식처럼 Commitment 스킴과 무관한 EIP-6800의 아이디어를 가져왔다. 달라지는 것은 부모 노드를 만드는 방법과 Proof를 줄이는 방법이다.

## Summary

1. Stateless Client가 되려면 Witness가 작아야 하고, Witness 크기는 상태 트리의 Proof 크기가 정한다.
2. MPT는 Level마다 형제 Hash를 보내야 하므로 Proof가 (Width − 1) × 깊이에 비례한다.
3. Verkle Tree는 부모를 Vector Commitment로 만들어 형제를 보내지 않는다. Width를 256으로 키워 깊이를 줄이고, 여러 Opening을 Multiproof 하나로 묶는다.
4. 그 결과 Key 1개의 경로 크기가 3,840 B에서 128 B로, 값 1,000개의 Witness가 약 3.5 MB에서 약 150 kB로 줄어든다.
5. 타원곡선에 의존하므로 양자 내성이 없고, 이더리움 로드맵은 Hash 기반 Binary Tree로 방향을 바꿨다.

## References

- Vitalik Buterin, [Verkle trees](https://vitalik.eth.limo/general/2021/06/18/verkle.html), 2021-06-18.
- Dankrad Feist, [Verkle trie for Eth1 state](https://dankradfeist.de/ethereum/2021/06/18/verkle-trie-for-eth1.html), 2021-06-18.
- Dankrad Feist, [PCS multiproofs using random evaluation](https://dankradfeist.de/ethereum/2021/06/18/pcs-multiproofs.html), 2021-06-18.
- Dankrad Feist, [Inner Product Arguments](https://dankradfeist.de/ethereum/2021/07/27/inner-product-arguments.html), 2021-07-27.
- Guillaume Ballet, Dankrad Feist, [Verkle tree structure](https://blog.ethereum.org/2021/12/02/verkle-tree-structure/), 2021-12-02.
- [EIP-6800: Ethereum state using a unified verkle tree](https://eips.ethereum.org/EIPS/eip-6800), 2023-03-17.
- Vitalik Buterin, [Possible futures of the Ethereum protocol, part 4: The Verge](https://vitalik.eth.limo/general/2024/10/23/futures4.html), 2024-10-23.
- [ethereum.org, Verkle trees](https://ethereum.org/roadmap/verkle-trees), 2026-06-24 갱신.
- [ethereum.org, Statelessness](https://ethereum.org/en/roadmap/statelessness/).
- [ethereum.org, Merkle Patricia Trie](https://ethereum.org/en/developers/docs/data-structures-and-encoding/patricia-merkle-trie).
- [Ethereum Magicians, EIP-7864 논의 스레드](https://ethereum-magicians.org/t/eip-7864-ethereum-state-using-a-unified-binary-tree/22611).
- [The Block, Vitalik Buterin lays out a two-part plan to overhaul Ethereum's execution layer](https://www.theblock.co/post/391681/vitalik-buterin-lays-out-a-two-part-plan-to-overhaul-ethereums-execution-layer-from-the-ground-up), 2026-03-01.