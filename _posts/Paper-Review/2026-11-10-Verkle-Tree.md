---
title: "Verkle Tree"
date: 2026-11-10
last_modified_at: 2026-11-24
categories: [Paper Review]
tags: [Blockchain, Ethereum, Verkle Tree, Merkle Patricia Trie, Stateless Client, Commitment]
math: true
media_subpath: /assets/img/posts/verkle-tree/
---

>
> 1. **Verkle Tree**
> 2. [Verkle Tree and the Ethereum Roadmap]({% post_url Paper-Review/2026-11-10-Verkle-Tree-Roadmap %})

> 관련 글: [Multiproof]({% post_url Cryptography/2026-11-10-Multiproof %})

[Paper]
- [Verkle Trees (John Kuszmaul)](https://math.mit.edu/research/highschool/primes/materials/2018/Kuszmaul.pdf)

[Reference]
- [https://vitalik.eth.limo/general/2021/06/18/verkle.html](https://vitalik.eth.limo/general/2021/06/18/verkle.html)
- [https://blog.ethereum.org/2021/12/02/verkle-tree-structure](https://blog.ethereum.org/2021/12/02/verkle-tree-structure)
- [https://ethereum.org/roadmap/verkle-trees/](https://ethereum.org/roadmap/verkle-trees/)
- [https://notes.ethereum.org/@vbuterin/verkle_tree_eip](https://notes.ethereum.org/@vbuterin/verkle_tree_eip)
- [EIP-6800: Ethereum state using a unified verkle tree](https://eips.ethereum.org/EIPS/eip-6800)

---

### TL;DR

Verkle Tree는 Merkle Tree의 부모 Node를 Hash 대신 Vector Commitment로 만든 트리다. 증명할 때 형제 Node가 필요 없어 Width(폭)를 넓히고 Depth(깊이)를 줄일 수 있으므로 Proof가 작아진다. 

Ethereum은 State 전체를 저장하지 않고 Witness로 Block을 검증하는 Stateless Client를 위해 2021년부터 State Tree를 Verkle Tree로 바꾸려 했다. 2026년 11월 현재 이 방안은 채택되지 않았다. Verkle Tree는 Quantum Resistance(양자 내성)가 없어 나중에 다시 교체해야 하고, Hash Tree를 STARK(Scalable Transparent Argument of Knowledge)로 증명하는 기술이 실용화에 가까워졌다. 그래서 지금 Roadmap에서는 Hash 기반 Binary Tree가 대안으로 논의되고 있다. 이 경위는 [2편]({% post_url Paper-Review/2026-11-10-Verkle-Tree-Roadmap %})에서 다룬다.

<br />

## Merkle Tree

### What is a Merkle Tree?

Merkle Tree는 Child(자식) Node들의 Hash를 이어 붙여 다시 Hash한 값을 Parent(부모) Node로 삼는 Tree다. 맨 위 Node인 Root의 Hash를 Merkle Root라 하고, 이 값 하나가 전체 Data를 대표한다. Node 하나가 가질 수 있는 최대 Child 수를 Width라 한다. 이 글은 Width가 2인 Binary Merkle Tree를 기준으로 설명한다.

#### How It Works

1. Data마다 Hash를 구해 Leaf Node로 둔다.
2. Node를 Left부터 두 개씩 묶고, 두 Hash를 Left, Right 순서로 이어 붙여 Hash한다. 그 결과가 Parent Node다.
3. Node가 하나 남을 때까지 2를 반복한다. 남은 Node의 Hash가 Merkle Root다.

Root에서 Edge(간선) 수가 같은 Node의 모음을 Level이라 하고, Root에서 Leaf까지의 Edge 수를 Tree Depth(트리의 깊이)라 한다. Node 수가 홀수일 때 짝 없는 Node를 처리하는 규칙은 구현마다 다르다. 이 글은 Leaf 개수가 2의 거듭제곱인 경우만 다룬다.

![merkle-tree.png](merkle-tree.png)

**Example**

- Leaf: h1부터 h4. Data1부터 Data4의 Hash
- Parent Node: h5 = hash(h1, h2), h6 = hash(h3, h4)
- Merkle Root: h7 = hash(h5, h6)
- hash(a, b)는 a와 b를 이 순서로 이어 붙인 값의 Hash다.
- Level은 3개, Tree Depth는 2다.

#### Key Properties

Data가 바뀌었는데 Merkle Root가 그대로이려면 Path(경로) 어딘가에서 Hash Collision(충돌)이 일어나야 한다. Hash Function(해시 함수)의 Collision Resistance(충돌 저항성) 때문에 이는 사실상 불가능하므로, Merkle Root 하나만 신뢰할 수 있으면 전체 Data를 검증할 수 있다.

단, Leaf와 Internal Node(내부 Node)를 같은 방식으로 Hash하면 Internal Node의 Input(입력), 즉 두 Child Hash를 이어 붙인 값을 Leaf Data로 내세워 Collision 없이 통과시킬 수 있다. 그래서 서로 다른 Prefix(접두어)를 붙여 Hash한다 (RFC 6962의 0x00 / 0x01).

#### Merkle Proof

Merkle Proof는 Data 하나가 i번째 Leaf에 있음을 보이는 Data다. 만들어 보내는 쪽을 Prover(증명자), 확인하는 쪽을 Verifier(검증자)라 한다. Verifier는 신뢰할 수 있는 Merkle Root를 알고 있다.

Parent가 같은 Node를 Sibling(형제)이라 한다. Parent의 Hash를 계산하려면 두 Child의 Hash와 순서가 필요하다. 그래서 Prover는 Root를 제외한 Level마다 Sibling의 Hash와 그 위치(Left/Right)를 보낸다.

위치 대신 Leaf Index(Left부터 0으로 센 Leaf의 위치)를 보내도 된다. Index를 Tree Depth만큼의 자릿수로 Binary(2진수)로 쓰면, 각 Bit가 위 Level부터 차례로 Path 위 Node 자신이 Left(0)인지 Right(1)인지를 나타낸다. Sibling은 그 반대쪽이다.

**Example**: Data1이 첫 번째 Leaf(Index 0)에 있음을 증명한다.

1. Prover가 Data1과 (h2, Right), (h6, Right)를 보낸다.
2. Verifier가 Data1의 Hash를 계산한다. → h1
3. hash(h1, h2)를 계산한다. → h5
4. hash(h5, h6)를 계산한다. → Merkle Root와 같으면 Accept

Data2, Data3, Data4는 보내지 않는다.

Leaf와 Internal Node를 구분해 Hash하지 않는 구현에서는 Verifier가 Tree Depth를 알고, Proof Length(Proof 길이)가 그와 같은지 확인해야 한다.

#### Size & Complexity

Leaf가 $n$개일 때:

- Tree Depth: $\log_2 n$
- Proof의 Hash: Root를 제외한 Level마다 1개씩, $\log_2 n$개
- Verifier의 Hash 계산: Leaf Data 1회 + Sibling Hash 하나당 1회, $\log_2 n + 1$회
- Leaf가 $2^{30}$개이고 Hash가 32 B일 때 Sibling Hash의 크기: 30 × 32 = 960 B, 약 1 kB. 위치 정보 30 Bit는 별도

**⇒ Verifier는 Root 하나만 저장하고도, Data 하나와 짧은 Proof로 포함 여부를 확인할 수 있다. 대신 Level마다 Sibling이 필요하다는 점이 뒤에서 문제가 된다.**

<br />

## Merkle Patricia Trie (MPT)

### What is a Merkle Patricia Trie?

Merkle Tree에서 Data의 위치는 Leaf Index로 정해진다. 반면 Ethereum의 State(상태)는 Address(주소)로 Account(계정)를 찾는 Key-Value 저장소다. Key로 Value를 찾고, 넣고, 지울 수 있으면서 Merkle Root 하나로 전체를 대표하는 구조가 필요하다.

[Merkle Patricia Trie(MPT)](https://ethereum.org/developers/docs/data-structures-and-encoding/patricia-merkle-trie/)는 Merkle Tree와 Patricia Trie를 결합한 Tree다.

- **Merkle Tree**: Parent가 Child를 Child의 Hash로 가리킨다. 그래서 Root의 Hash 하나가 전체를 대표한다.

- **Patricia Trie**: Trie(트라이)는 Key를 한 조각씩 따라 내려가 Value를 찾는 Tree로, Key가 곧 Root에서 Value까지의 Path다. Patricia Trie는 Child가 하나뿐인 Node가 이어지는 구간을 Node 하나로 줄인 Trie다.

![patricia-trie.png](patricia-trie.png)

Ethereum은 Key를 4 Bit 단위인 Nibble로 자른다. Nibble 하나는 16진수 한 자리이므로 Node 하나의 Child는 최대 16개, 즉 Width는 16이다.

Ethereum의 State Trie, Storage Trie, Transaction Trie, Receipt Trie가 모두 MPT다. 이 글은 State Trie를 기준으로 설명한다.

- **Key**: Keccak-256(Address). 32 Byte이므로 64 Nibble

- **Value**: Account를 Recursive Length Prefix(RLP)로 Encoding한 값. RLP는 Ethereum의 Serialization(직렬화) 형식이고, Account는 nonce, balance, storageRoot, codeHash의 네 값이다.

Address를 Hash해서 Key로 쓰므로 모든 Key의 길이가 64 Nibble로 같고, Key가 고르게 퍼진다.


#### Node Types

| Node | 형태 | 역할 |
| --- | --- | --- |
| Branch Node | 17개 항목 `[v0 ... v15, vt]` | Nibble 0~f에 해당하는 자식 16개와 Value 1개 |
| Extension Node | 2개 항목 `[encodedPath, key]` | 분기 없는 공통 경로와 다음 Node의 참조 |
| Leaf Node | 2개 항목 `[encodedPath, value]` | 남은 경로와 Value |

Patricia Trie와 대응시키면 Branch Node가 분기 지점이고, Extension Node가 분기 없이 묶인 구간이다. Leaf Node는 더 이상 분기가 없는 나머지 경로를 통째로 가진다. State Tree의 Key는 32 Byte이므로 경로 길이는 64 Nibble이다.


#### How It Works

Node의 각 항목을 Slot(칸)이라 한다. Node를 RLP로 Encoding한 값의 Keccak-256 Hash가 그 Node의 Hash이고, Parent는 Child를 가리키는 Slot에 이 Hash를 담는다. 단, [RLP가 32 Byte보다 짧은 Node는 Hash하지 않고 Parent 안에 그대로 넣는다](https://ethereum.org/en/developers/docs/data-structures-and-encoding/patricia-merkle-trie). Root Node는 길이와 상관없이 Hash하며, 이 값이 Merkle Root다. State Trie의 Merkle Root를 State Root라 하고, Block Header에 들어간다.

`encodedPath`는 Path 앞에 Prefix Nibble 하나를 붙인 값이다 (Hex-Prefix Encoding). Prefix는 두 가지를 나타낸다.

- Node 종류: Extension Node와 Leaf Node는 둘 다 Slot이 2개라서 Prefix로 구분한다.
- Path 길이의 홀짝: Path는 Nibble 단위인데 저장은 Byte 단위라서, [Nibble 1(홀수 길이)과 Nibble 01(짝수 길이)이 둘 다 Byte 01로 저장되어 구분할 수 없다](https://ethereum.org/developers/docs/data-structures-and-encoding/patricia-merkle-trie/#optimization).

Prefix 0은 Extension Node의 짝수 길이 Path, 1은 Extension Node의 홀수 길이 Path, 2는 Leaf Node의 짝수 길이 Path, 3은 Leaf Node의 홀수 길이 Path다. 짝수 길이면 Prefix 뒤에 Nibble 0을 하나 더 붙여 Byte 단위를 맞춘다.

Key로 Value를 찾는 순서:

1. Key를 Nibble 열로 바꾼다.
2. Root Node에서 시작해 Node 종류에 따라 내려간다.
   - Branch Node: 다음 Nibble 하나에 해당하는 Slot의 Child로 간다. Slot이 비어 있으면 Key가 없는 것이다.
   - Extension Node: Path가 남은 Nibble의 앞부분과 같으면 그만큼 건너뛰고 Child로 간다. 다르면 Key가 없는 것이다.
   - Leaf Node: Path가 남은 Nibble 전체와 같으면 Value를 얻는다. 다르면 Key가 없는 것이다.
3. Child로 갈 때, Slot에 Hash가 있으면 그 Hash로 Database에서 Node를 읽고, Node가 그대로 들어 있으면 그것을 쓴다.

![mpt-st.png](mpt-st.png)

**Example**

그림은 Key 4개를 담은 State Trie다. 설명을 위해 Key를 7 Nibble로, Value를 잔액으로 줄였다. 실제 State Trie의 Key는 64 Nibble이고, Value는 RLP로 Encoding한 Account다.

| Key | Value |
| --- | --- |
| a711355 | 45.0 ETH |
| a77d337 | 1.00 WEI |
| a7f9365 | 1.1 ETH |
| a77d397 | 0.12 ETH |

그림의 표기는 Node Types의 항목과 다음과 같이 대응한다.

- Extension Node: `prefix`와 `shared nibble(s)`를 합친 것이 `encodedPath`, `next node`가 `key`
- Leaf Node: `prefix`와 `key-end`를 합친 것이 `encodedPath`

아래에서는 Extension Node를 위에서부터 E1, E2, Branch Node를 위에서부터 B1, B2라 한다. Leaf Node는 Value가 45.0 ETH, 1.00 WEI, 0.12 ETH, 1.1 ETH인 순서로 L1, L2, L3, L4라 한다.

- 네 Key가 모두 a7로 시작하므로 Root Node는 `shared nibble(s)`가 a7인 Extension Node E1이다.
- 세 번째 Nibble이 1, 7, f로 갈리므로 Branch Node B1이 온다. Slot 1은 L1, Slot f는 L4를 가리키고, 두 Leaf Node는 남은 Nibble 1355와 9365를 `key-end`로 가진다.
- a77로 시작하는 두 Key는 다음 두 Nibble d3도 같으므로 Slot 7은 Extension Node E2를 가리킨다. 그다음 Nibble 3과 9에서 Branch Node B2로 갈리고, L2와 L3은 마지막 Nibble 7을 `key-end`로 가진다.
- a77d397을 찾으면 a7(E1) → 7(B1) → d3(E2) → 9(B2) → 7(L3) 순서로 Node 5개를 거쳐 0.12 ETH를 얻는다. a711355는 a7(E1) → 1(B1) → 1355(L1)로 Node 3개만 거친다.
- Prefix는 Path 길이가 짝수인 E1과 E2가 0, L1과 L4가 2이고, 홀수인 L2와 L3이 3이다. 그림은 짝수 길이일 때 Prefix 뒤에 붙는 Nibble 0을 생략했다. 실제 `encodedPath`는 E1이 00a7, E2가 00d3, L1이 201355, L4가 209365, L2와 L3이 37이다.
- 그림의 `stateRoot`가 State Root이고, Root Node인 E1의 Hash다.


#### Key Properties

- (Key, Value)의 집합이 같으면 넣은 순서와 상관없이 Tree의 모양과 Merkle Root가 같다.
- Value 하나가 바뀌면 그 Key의 Path 위 Node만 Hash가 바뀐다.
- Merkle Tree와 마찬가지로 Hash Function의 Collision Resistance 덕분에, State Root 하나만 신뢰할 수 있으면 전체 State를 검증할 수 있다.
- Root에서 Leaf Node까지 거치는 Node 수가 Key마다 다르다.

#### Merkle Proof

MPT의 Merkle Proof는 Root Node부터 Key의 Path를 따라 만나는 Node 전부다. 각 Node를 RLP로 Encoding한 값을 보내며, Parent 안에 들어 있는 32 Byte 미만 Node는 따로 보내지 않는다. Verifier는 신뢰할 수 있는 State Root를 알고 있다.

1. 첫 Node의 Hash가 State Root와 같은지 확인한다.
2. Key의 Nibble을 따라 내려가며, 현재 Node의 Child Slot에 있는 Hash가 다음 Node의 Hash와 같은지 확인한다.
3. Leaf Node의 Path가 남은 Nibble 전체와 같으면 그 Value를 Accept한다.

**Example**: 위 그림에서 Key a77d397의 Value가 0.12 ETH임을 증명한다. hash(X)는 X를 RLP로 Encoding한 값의 Keccak-256 Hash이고, 모든 Node의 RLP가 32 Byte 이상이라고 가정한다.

1. Prover가 E1, B1, E2, B2, L3을 보낸다.
2. Verifier가 hash(E1)이 State Root와 같은지 확인한다. E1의 `shared nibble(s)` a7이 Key의 앞부분과 같고, E1의 `next node`가 hash(B1)과 같은지 확인한다.
3. B1의 Slot 7이 hash(E2)와 같은지 확인한다.
4. E2의 `shared nibble(s)` d3이 Key의 다음 부분과 같고, E2의 `next node`가 hash(B2)와 같은지 확인한다.
5. B2의 Slot 9가 hash(L3)과 같은지 확인한다.
6. L3의 `key-end` 7이 남은 Nibble과 같다. → 0.12 ETH를 Accept

Branch Node의 Hash를 계산하려면 Child Slot 16개가 모두 필요하다. 그래서 Binary Merkle Tree에서는 Level마다 Sibling이 1개였지만, MPT에서는 Branch Node 하나마다 Sibling이 최대 15개 들어간다. 위 예시에서 B1을 보내면 L1과 L4의 Hash가, B2를 보내면 L2의 Hash가 함께 간다.

Key가 없다는 것도 같은 방식으로 증명한다. Path를 따라가다 빈 Slot을 만나거나, Extension Node나 Leaf Node의 Path가 Key와 다름을 보이면 된다.

#### Size & Complexity

Key가 $n$개이고 고르게 퍼져 있으며, Path 위 Branch Node의 Slot이 모두 찼다고 가정한 근삿값이다. Extension Node와 Leaf Node의 크기는 넣지 않았다.

- Branch Node의 Level 수: $\log_{16} n = \frac{1}{4}\log_2 n$. Binary Merkle Tree의 1/4
- Branch Node 하나의 크기: Hash 16개 × 32 B = 512 B. RLP의 길이 표시는 별도
- Proof의 크기: 512 × $\log_{16} n$ = [128 × $\log_2 n$ B](https://hackmd.io/@vbuterin/BylgAm7ZF)
- Key가 $2^{30}$개일 때: 128 × 30 = 3,840 B, 약 4 kB. 같은 개수의 Binary Merkle Tree는 960 B
- Verifier의 Hash 계산: Proof의 Node 하나당 1회

Width가 $k$인 Merkle Tree에서 Proof에 들어가는 Sibling은 Level마다 $k - 1$개, 모두 $(k - 1) \times \log_k n$개다. $n = 2^{30}$일 때 $k = 2$면 30개, $k = 16$이면 15 × 7.5 = 112.5개다.

MPT는 Branch Node를 통째로 보내므로, Verifier가 직접 계산할 수 있는 Path 위 Child의 Hash까지 Level마다 16개가 들어간다. 그래서 Proof의 크기는 Sibling 15개가 아니라 Hash 16개를 기준으로 계산했다.

**⇒ MPT는 Key로 Value를 찾을 수 있는 Merkle Tree다. Width를 16으로 키워 Tree Depth는 줄었지만, Level마다 Sibling이 15개 필요해 Proof는 Binary Merkle Tree보다 커졌다.**

## Proof Size Problem

MPT의 Proof가 크다는 점은 지금의 Ethereum에서는 드러나지 않는다. Full Node(풀 노드)는 State 전체를 가지고 있어서, Block을 검증할 때 Proof를 주고받지 않기 때문이다. 이 문제는 State를 저장하지 않고 Block을 검증하려 할 때 드러난다.

#### State Growth

Block을 검증하려면 Block 안의 Transaction을 다시 실행하고, State Trie를 갱신해 State Root를 계산해야 한다. 그래서 [Full Node는 State Trie 전체를 저장한다](https://ethereum.org/roadmap/verkle-trees).

- State의 크기는 수백 GB다. Geth의 Database에서 측정한 값은 [약 359 GB](https://ethresear.ch/t/what-if-we-only-kept-1-year-of-active-state/23980)였다.
- State는 Account와 Storage가 늘어나는 만큼 계속 커진다.
- 저장 공간 요구가 커질수록 일반 사용자가 Full Node를 운영하기 어렵다. 검증에 참여하는 Node가 줄면 검증이 소수의 Node에 집중될 수 있다.

#### Stateless Client

Stateless Client(무상태 클라이언트)는 State 전체를 저장하지 않고 Block을 검증하는 Client다. State 대신 Block과 함께 오는 Witness를 쓴다.

- Witness: 그 Block의 Transaction을 실행하는 데 필요한 State 조각과, 그 조각이 State에 속함을 보이는 Proof의 묶음
- 검증: Stateless Client는 이전 Block의 State Root만 알고 있다. Witness의 Proof를 State Root로 확인한 뒤, Witness의 State 조각으로 Transaction을 실행한다.

Witness는 Block마다 Network로 전파된다. 그래서 [Block 간격인 12초 안에 안전하게 전파될 만큼 작아야 한다](https://ethereum.org/roadmap/verkle-trees). Witness가 크면 느린 Node는 제때 받지 못해 뒤처지고, 빠른 연결을 가진 Node만 검증할 수 있게 된다. State 저장 부담을 없애려다 Network 부담으로 같은 중앙화 문제가 생기는 셈이다.

#### MPT Witness Size

MPT로는 이 조건을 맞출 수 없다.

- Account 접근 한 번의 Witness: [평균 약 3 kB, 최악의 경우 그 3배](https://notes.ethereum.org/@vbuterin/verkle_tree_eip). 앞에서 계산한 3,840 B와 같은 규모다.
- Block 하나의 Witness: 최악의 경우 접근이 6,000번(15M Gas ÷ 접근당 2,500 Gas)이면 약 18 MB. 12초 안에 Peer-to-Peer(P2P) Network로 전파하기에는 너무 크다.
- Leaf 1,000개의 Witness: [약 3.5 MB](https://ethereum.org/roadmap/verkle-trees)

원인은 앞에서 본 식 $(k - 1) \times \log_k n$에 있다. Hash는 Child 전부를 입력으로 받으므로, Parent의 Hash를 확인하려면 Level마다 나머지 Child의 Hash $k - 1$개가 모두 필요하다.

- Width를 키우면 Tree Depth는 줄지만 Level마다 Sibling이 늘어난다. MPT가 이 경우다.
- Width를 2로 줄이면 Sibling은 Level마다 1개지만 Tree Depth가 4배가 된다. Proof는 MPT의 1/4로 줄지만 [그래도 충분히 작지 않다](https://hackmd.io/@vbuterin/BylgAm7ZF).

Hash로 만든 Tree에서는 Width를 어느 쪽으로 바꿔도 Tree Depth와 Sibling 수 중 하나가 커진다.

#### Toward Verkle Tree

필요한 것은 Width를 키워 Tree Depth를 줄이면서도 Sibling을 보내지 않아도 되는 구조다. Verkle Tree는 Node의 Hash를 Vector Commitment(벡터 커밋먼트)로 바꾼다. [Vector Commitment는 Width와 상관없이 한 위치의 값을 일정한 크기의 Proof로 증명한다](https://dankradfeist.de/ethereum/2021/06/18/verkle-trie-for-eth1.html). 그래서 식에서 $k - 1$이 사라지고, Width를 키울수록 Proof가 작아진다.

- Account 접근 한 번의 Witness: [평균 약 200 B](https://notes.ethereum.org/@vbuterin/verkle_tree_eip). MPT는 약 3 kB
- Leaf 1,000개의 Witness: [약 150 kB](https://ethereum.org/roadmap/verkle-trees). MPT 3.5 MB의 약 1/23

2026년 11월 기준으로 Verkle Tree를 State Tree로 도입하는 Ethereum Improvement Proposal(EIP) 6800은 Stagnant 상태다. [현재 제안은 Binary Tree를 쓰는 EIP-7864이며 Draft 상태다](https://github.com/ethereum/ethereum-org-website/pull/19844). 이 글은 Verkle Tree가 위 문제를 어떻게 푸는지를 다룬다.

**⇒ Stateless Client에는 작은 Witness가 필요하다. Hash로 만든 Tree는 Width를 키워도 줄여도 Proof가 충분히 작아지지 않는다. Verkle Tree는 Hash를 Vector Commitment로 바꿔 Width와 Proof 크기의 관계를 끊는다.**

<br />

## Verkle Tree

### 핵심 아이디어: Hash 대신 Vector Commitment

논문의 핵심은 한 가지다. Merkle Tree를 만들되 Hash 함수 자리에 Vector Commitment를 쓴다. Merkle Tree에서 부모 Node는 자식들의 Hash이고, Verkle Tree에서 부모 Node는 자식들의 Vector Commitment다.

**문제 상황: Hash로 만든 부모는 자식 전부를 알아야 검증할 수 있음. 그래서 Level마다 형제 Node가 Proof에 들어감**

**해결 방법: 부모 Node를 자식들의 Vector Commitment로 만듦. 형제 없이, 경로 위의 Commitment와 짧은 증명만으로 검증함**

효과는 두 가지임

1. **형제 Node가 필요 없음**: 자식 하나가 부모에 속함을 보이는 데 나머지 자식들이 필요하지 않음
2. **깊이를 줄일 수 있음**: 형제를 보내지 않으므로 Width를 키워도 Level당 Proof 데이터가 늘지 않음. Width를 키우면 깊이만 줄어듦

Problem의 네 질문에 대한 답은 다음과 같음. 앞의 두 가지는 논문의 답이고, 뒤의 두 가지는 Ethereum 제안의 답임

| 질문 | 해결 방법 | 설명하는 곳 |
| --- | --- | --- |
| 형제 Node 없이 검증할 수 있는가 | 부모 Node를 Vector Commitment로 만들고, 항목 하나만 Opening Proof로 확인 | Commitment Scheme과 Vector Commitment, Verkle Tree의 검증 방법 |
| 깊이를 줄일 수 있는가 | Width를 키움. 형제가 없으므로 Proof가 늘지 않음 | Verkle Tree의 구조 |
| Value가 많을 때도 Proof가 작게 유지되는가 | 모든 Opening Proof를 Multiproof 하나로 묶음 | 여러 Value의 검증과 Multiproof |
| 갱신 비용을 감당할 수 있는가 | Opening Proof를 저장하지 않고, 경로 위 Commitment만 바뀐 분량만큼 갱신 | Value가 바뀔 때의 갱신 |

Ethereum에 제안된 Verkle Tree의 Width는 256임. 1,024로 올리자는 의견도 있었음([Vitalik Buterin의 2021년 글](https://vitalik.eth.limo/general/2021/06/18/verkle.html)). Width의 상한은 Proof를 만드는 비용이 정함. Node마다 Width에 비례하는 연산이 들기 때문임

| 항목 | MPT | Verkle Tree (Ethereum 제안) |
| --- | --- | --- |
| 부모 Node의 값 | 자식들의 Hash | 자식들에 대한 Vector Commitment |
| Width | 16 | 256 |
| 깊이 | 약 $\log_{16} N$ | 약 $\log_{256} N$ |
| Level당 Proof 데이터 | 형제 Hash 최대 15개 (480 B) | Commitment 1개 (32 B) |
| 여러 Key를 검증할 때 | Key마다 형제 Hash 추가 | 모든 증명을 Multiproof 1개로 묶음 |

용어는 다음과 같음. 각 용어는 뒤에서 다시 설명함

| 용어 | 의미 |
| --- | --- |
| Commitment | 값을 고정해 둔 짧은 값 |
| Vector Commitment | 값의 목록 전체에 대한 Commitment |
| Opening | Commitment의 특정 Index에 있는 항목을 밝히는 것 |
| Opening Proof | Opening이 맞음을 보이는 증명. 논문은 Membership Proof라 부름 |
| Multiproof | 여러 Opening Proof를 하나로 묶은 증명. 논문에는 없고 Ethereum 제안에 있음 |

### Commitment Scheme과 Vector Commitment

Commitment Scheme은 값을 **Commitment**라는 짧은 값 하나로 고정해 두고, 나중에 그 값을 열어 확인시키는 암호학적 방법임. 값을 봉투에 넣어 밀봉해 건네는 것에 비유할 수 있음

| 성질 | 의미 | 봉투 비유 |
| --- | --- | --- |
| Hiding(은닉성) | Commitment만 보고는 값을 알 수 없음 | 밖에서 내용을 볼 수 없음 |
| Binding(구속성) | 한 번 만든 Commitment를 다른 값의 Commitment인 것처럼 열 수 없음 | 밀봉한 뒤에는 내용을 바꿀 수 없음 |

Verkle Tree가 쓰는 것은 **Vector Commitment**임

- Vector는 순서가 있는 값의 목록임. Vector Commitment는 Vector $(v_0, \ldots, v_{n-1})$ 전체를 Commitment 하나로 만듦
- "이 Commitment의 Index $i$에 있는 항목은 $v_i$다"를 나머지 항목 없이 짧은 증명으로 보일 수 있음
- 특정 Index의 항목을 이렇게 밝히는 것이 **Opening**, 이때 붙이는 증명이 **Opening Proof**임. Evaluation Proof라고도 부름

왜 Hiding은 필요 없는가?

- Ethereum의 State는 공개 데이터이므로 값을 감출 이유가 없음
- Verkle Tree가 쓰는 성질은 Binding과 짧은 Opening Proof임
- 실제로 EIP-6800의 Commitment 식에는 값을 감추기 위한 난수 항이 없음

Hash도 Vector Commitment로 볼 수 있음. 다만 항목 하나를 열려면 나머지 항목이 전부 필요한, Proof가 큰 Vector Commitment임

### Polynomial Commitment로 만드는 Vector Commitment

Vector Commitment를 만드는 한 방법이 Polynomial Commitment(다항식 커밋먼트)임

- 항목 $v_0, \ldots, v_{n-1}$에 대해 $f(i)=v_i$를 만족하는 다항식 $f$를 만듦. 주어진 점들을 지나는 다항식을 구하는 Lagrange Interpolation(라그랑주 보간)을 씀
- 그러면 "Index $i$의 항목이 $v_i$다"는 "$f(i)=v_i$다"와 같은 말이 됨
- 쓰는 성질: $f(z)=y$이면 $f(x)-y$는 $(x-z)$로 나누어떨어지고, 그 역도 성립함. 따라서 몫 $q(x) = \dfrac{f(x)-y}{x-z}$가 다항식으로 존재한다는 것이 $f(z)=y$의 증거가 됨

예시: Kate-Zaverucha-Goldberg(이하 KZG) Commitment의 흐름을 작은 수로 줄인 것임. 실제 암호 연산이 아니라 원리를 보이기 위한 계산임

- Vector는 (40, 2, 4)임. MPT 예시의 a1에서 남은 경로 40, Nonce 2, Balance 4를 Index 0, 1, 2의 항목으로 삼은 것임
- Prover는 다항식을 공개하지 않고 "Index 2의 항목은 4다"를 보이려 함
- Setup: Random Parameter $r=7$ 생성

1. **Commit**: 다항식 $f$의 Commitment $C=614$ 생성

    $f(0)=40$, $f(1)=2$, $f(2)=4$로부터 $f(x)$ 생성

    $$
    f(x)=20x^2-58x+40
    $$

    $$
    C=\operatorname{Commit}(f)=f(r)=20\times 7^2-58\times 7+40=614
    $$

2. **Opening**: $x=2$일 때 $f(2)=4$와 Opening Proof $\pi=122$ 생성

    $$
    \begin{aligned}
    q(x)&=\frac{f(x)-f(2)}{x-2}=\frac{20x^2-58x+40-4}{x-2}\\
    &=\frac{20x^2-58x+36}{x-2}=\frac{(20x-18)(x-2)}{x-2}=20x-18
    \end{aligned}
    $$

    $$
    \pi=\operatorname{Commit}(q)=q(r)=20\times 7-18=122
    $$

3. **Verify**: $C-f(2)=\pi\times(r-2)$가 성립하는지 확인하여 Accept 또는 Reject

    $$
    f(7)-f(2)=q(7)\times(7-2)
    $$

    $$
    614-4=610=122\times 5
    $$

![polynomial-commitment.png](polynomial-commitment.png)

- Verifier가 받은 것은 $C$, $f(2)$, $\pi$뿐임. 나머지 항목 40과 2는 받지 않음 → 형제가 필요 없는 이유
- 거짓 값은 통과하지 못함. $f(2)=5$라고 주장하면 $f(x)-5=20x^2-58x+35$는 $x-2$로 나누어떨어지지 않음(나머지 -1). 다항식 $q$를 만들 수 없음

주의: 이 예시가 줄인 부분

- 실제로는 $r$을 아무도 모름. KZG는 Trusted Setup(신뢰 설정)이라는 준비 절차에서 $r$의 거듭제곱을 타원곡선 점 안에 숨긴 공개 매개변수를 만들고 $r$을 폐기함
- Commitment와 Opening Proof는 수가 아니라 타원곡선 점임. 3단계의 등식은 Pairing이라는 타원곡선 연산으로 $r$을 모르는 채 확인함
- $r$이 알려지면 안전하지 않음. 위 예시에서 $r=7$을 아는 Prover는 거짓 값 14에 대해 $\pi=(614-14)/5=120$을 만들어 등식을 통과시킬 수 있음 (출처에 없는 계산)

### Verkle Tree의 구조

Ethereum 제안의 Width인 256을 기준으로 설명함. 원리를 보이기 위해 단순화한 구조이고, 실제 제안의 구조는 뒤의 "Ethereum 제안(EIP-6800)"에 있음

- **Inner Node**: 자식 256개의 Commitment를 항목으로 가짐. Key를 1 Byte씩 따라 분기함. MPT의 Branch Node에 해당함
- **Leaf Node**: Key의 남은 부분과 Value를 가짐
- Node의 값은 자기 항목들에 대한 Commitment임. Root의 Commitment가 State Root가 됨

예시 1: MPT 예시의 Key를 그대로 씀. Key의 첫 Byte가 Root에서의 Index가 됨 (0x73 = 115, 0x7b = 123, 0xd5 = 213)

![verkle-structure.png](verkle-structure.png)

```
Root (Inner Node)  C_r = Commit(C_0, ..., C_255)
├─ Index 0x73 → Leaf Node  C_115 = Commit(a1의 남은 Key와 Value)
├─ Index 0x7b → Leaf Node  C_123 = Commit(a2의 남은 Key와 Value)
└─ Index 0xd5 → Leaf Node  C_213 = Commit(a3의 남은 Key와 Value)
```

- 세 Key가 첫 Byte에서 모두 갈림 → Inner Node는 Root 하나뿐임
- MPT에서는 a1과 a2가 첫 Nibble 7을 공유해 Branch Node가 하나 더 필요했음

→ Width가 넓어지면 같은 Key 집합에서 깊이가 줄어듦

예시 2: 깊이가 더 깊어지는 경우임. 예시 1과 다른 Account 집합이고, Key를 5 Byte로 줄임

| Account | Key | Value |
| --- | --- | --- |
| a1 | 0x0a1b2c3d4e | Nonce 4, Balance 10 ETH |
| a2 | 0x0a62c01234 | Nonce 2, Balance 1 ETH |
| a3 | 0x0a62cd5f61 | Nonce 3, Balance 0.5 ETH |
| a4 | 0x0ac4e34567 | Nonce 5, Balance 4 ETH |
| a5 | 0x0ac4ea6765 | Nonce 7, Balance 3 ETH |

![verkle-structure-deep.png](verkle-structure-deep.png)

- 다섯 Key가 첫 Byte 0a를 공유함 → Root의 Index 0x0a 아래에 Inner Node가 생김
- 둘째 Byte는 1b, 62, c4로 갈림. 1b는 a1뿐이므로 바로 Leaf Node가 됨
- a2와 a3은 62를, a4와 a5는 c4를 공유함 → 각각 Inner Node가 하나 더 생기고, 셋째 Byte에서 갈림 (c0와 cd, e3와 ea)
- Inner Node는 Key가 갈리는 지점까지만 생김. Root를 포함해 a1의 경로에는 Inner Node가 2개, 나머지 경로에는 3개 있음
- Commitment의 아래 첨자는 부모 안에서의 Index를 10진수로 적은 것임 (0x0a = 10, 0x1b = 27, 0x62 = 98, 0xc4 = 196)

### Verkle Tree의 검증 방법

- Verifier는 Root의 Commitment $C_r$을 이미 알고 있음
- 경로의 각 Level에서 "부모 Commitment를 Index $z$에서 Opening하면 항목이 $y$다"를 확인함
    - $z$: Key가 정하는 자식의 위치
    - $y$: 자식 Commitment. 마지막 Level에서는 저장된 Value

예시: 예시 1에서 a1의 Value를 검증함

![verkle-proof.png](verkle-proof.png)

1. $C_r$을 Index 115에서 Opening → 항목이 $C_{115}$임을 확인 (Opening Proof $\pi_r$)
2. $C_{115}$를 Opening → 항목이 a1의 Value임을 확인 (Opening Proof $\pi_{115}$)

→ Prover가 보내는 것은 a1의 Value, $C_{115}$, $\pi_r$, $\pi_{115}$임. a2와 a3에 관한 데이터는 필요하지 않음

MPT에서는 같은 검증에 a2 Leaf Node의 Hash와 a3 Leaf Node의 Hash가 필요했음

<br />

## Ethereum 제안(EIP-6800)

Ethereum은 논문의 구조를 그대로 쓰지 않았다. 이 장은 State Tree를 Verkle Tree로 바꾸는 제안인 [EIP-6800](https://eips.ethereum.org/EIPS/eip-6800)과, 그 설계를 설명한 글들을 기준으로 한다. EIP-6800은 2023년 3월에 작성됐고, 본문은 2026년 11월 24일에 확인했다. 이 제안이 채택되지 않은 경위는 [2편 Verkle Tree and the Ethereum Roadmap]({% post_url Paper-Review/2026-11-10-Verkle-Tree-Roadmap %})에 있다.

### 논문과 달라진 점

| 항목 | 논문 | Ethereum 제안 |
| --- | --- | --- |
| Vector Commitment | Catalano와 Fiore의 방식 | Pedersen Commitment와 Inner Product Argument(내적 증명, 이하 IPA) |
| Width | 정하지 않음. 용도에 따라 고름 | 256 |
| Opening Proof | 트리를 만들 때 자식마다 미리 계산 | 저장하지 않음. Block이 접근한 Value에 대해 그때 계산 |
| 여러 Value의 Proof | 다루지 않음 | Multiproof 하나로 묶음 |
| Value 하나의 갱신 | $O(k \log_k n)$ | 경로 위 Commitment만 바뀐 분량만큼 갱신. Width에 비례하지 않음 |
| 담는 데이터 | 파일 같은 데이터의 목록 | Account, Storage, Code |

**⇒ 논문의 약점은 구성과 갱신에 드는 $k$배의 계산이었다. Ethereum 제안은 Opening Proof를 저장하지 않고, 덧셈으로 갱신할 수 있는 Commitment를 쓴다. 그래서 갱신 비용이 Width에 비례하지 않는다. Width에 비례하는 연산은 Proof를 만드는 시점에만 든다.**

### Pedersen Commitment와 IPA

앞의 Polynomial Commitment 예시는 KZG의 흐름임. Ethereum 제안은 다른 방식인 **Pedersen Commitment**와 **IPA**를 씀

타원곡선 위의 점은 서로 더할 수 있고 정수를 곱할 수 있음. 곱하는 정수를 Scalar라 함. Pedersen Commitment는 미리 정해 둔 타원곡선 점 $G_0, \ldots, G_{255}$에 항목 $v_i$를 각각 곱해 모두 더한 점임. 이 점들을 기저점이라 함

$$
C = v_0 G_0 + v_1 G_1 + \cdots + v_{255} G_{255}
$$

- **Binding**: 어떤 기저점이 다른 기저점의 몇 배인지(Discrete Log 관계)가 알려져 있지 않으면 성립함
- **Trusted Setup**: 필요 없음
- **Opening Proof**: IPA로 만듦([Ethereum Foundation Blog](https://blog.ethereum.org/2021/12/02/verkle-tree-structure))
- **Additively Homomorphic(덧셈 동형)**: 두 Commitment를 더하면 두 Vector의 합에 대한 Commitment가 됨. 이 성질이 트리의 갱신과 Multiproof를 가능하게 함

Ethereum 제안에서의 값

- 타원곡선: Bandersnatch. 기저점 256개 (EIP-6800)
- Commitment 크기: 32 B (Ignacio Hagopian의 [Anatomy of a Verkle proof](https://ihagopian.com/posts/anatomy-of-a-verkle-proof)가 적은 Witness 형식)
- 한 항목에 넣을 수 있는 값: 252 Bit까지. Scalar가 가질 수 있는 값의 집합인 Scalar Field의 크기가 253 Bit 소수이기 때문임 (Ethereum Foundation Blog)

| 항목 | KZG | Pedersen Commitment와 IPA |
| --- | --- | --- |
| Trusted Setup | 필요 | 불필요 |
| Opening Proof | 타원곡선 점 1개 | 타원곡선 점 $2\log_2 256 = 16$개와 Scalar 1개 |
| 검증 비용 | Pairing 1회 | Width에 비례하는 타원곡선 연산 |
| 암호학적 가정 | Bilinear Group | Discrete Log |

- IPA는 KZG보다 Opening Proof가 크고 검증 비용이 큼. 대신 Multiproof로 모든 Opening을 하나로 묶으므로, IPA의 Opening Proof는 Witness 하나에 한 개만 들어가고 한 번만 검증함
- 비교는 Dankrad Feist의 [Inner Product Arguments](https://dankradfeist.de/ethereum/2021/07/27/inner-product-arguments.html)에 있음. Scalar 1개는 Anatomy of a Verkle proof가 적은 Witness 형식에 있음
- 두 방식 모두 타원곡선의 Discrete Log 문제가 어렵다는 데 기댐. 양자 컴퓨터는 이 문제를 풀 수 있으므로 어느 쪽도 Quantum Resistance(양자 내성)가 없음. 이 점이 [2편]({% post_url Paper-Review/2026-11-10-Verkle-Tree-Roadmap %})에서 볼, Verkle Tree가 Roadmap에서 빠지는 첫째 이유가 됨
- 주의: 다른 자료는 Commitment 방식이 다를 수 있음. Dankrad Feist의 [Verkle trie for Eth1 state](https://dankradfeist.de/ethereum/2021/06/18/verkle-trie-for-eth1.html)는 KZG 기준으로 쓰였고, Vitalik Buterin의 2021년 글은 수치를 KZG(Commitment 48 B) 기준으로 듦

### 여러 Value의 검증과 Multiproof

논문은 여러 Proof를 묶는 방법을 다루지 않음. Multiproof는 Ethereum 제안에서 더해진 것임

**문제 상황: 증명할 Value가 많아지면 Opening Proof도 그만큼 늘어남**

- Block 하나는 수천 개의 Value에 접근함. Opening이 대략 (Key 개수) × (Key당 Opening 수)만큼 생김
- 예시 1에서 a1, a2, a3를 모두 검증하면 Commitment 3개($C_{115}$, $C_{123}$, $C_{213}$)와 Opening Proof 6개(Root에서 3개, Leaf Node에서 3개)가 필요함

![verkle-multi-proof.png](verkle-multi-proof.png)

**해결 방법: Multiproof로 모든 Opening Proof를 크기가 고정된 증명 하나로 묶음**

- 원리: Opening마다 "주장이 맞을 때만 다항식이 되는 식" $\dfrac{f_i(X)-y_i}{X-z_i}$를 만들고, 무작위 계수를 곱해 모두 더한 다항식 하나를 만듦. 이 다항식 하나에 대한 증명만 확인하면 모든 Opening이 한 번에 검증됨
- 자세한 과정은 [Multiproof 글](/posts/multiproof/)과 Dankrad Feist의 [PCS multiproofs using random evaluation](https://dankradfeist.de/ethereum/2021/06/18/pcs-multiproofs.html)에 있음
- 크기는 Opening 개수와 무관하게 고정임. Vitalik Buterin의 2021년 글은 방식에 따라 128~1,000 B로 적음
- KZG에서 Dankrad Feist의 방식은 타원곡선 점 2개임. 점 하나가 48 B면 96 B임 (Byte 수는 직접 계산한 값). Vitalik Buterin의 2021년 글의 표는 Commitment 1개, 수 1개, 증명 2개인 176 B로 계산되어 있음
- EIP-6800에 맞춰 제안된 Witness 형식에서는 576 B임. Commitment 1개와 IPA의 Opening Proof(타원곡선 점 16개, Scalar 1개)이고 원소마다 32 B임 (Anatomy of a Verkle proof의 형식에서 계산한 값)

Multiproof를 적용한 Witness의 구성

- 접근한 Key와 Value
- 각 경로 위의 Commitment. Root는 제외하고, 경로가 겹치면 한 번만 넣음
- Multiproof 하나

$z$와 $y$는 보내지 않음. Verifier가 Key, Value, 경로의 Commitment로부터 직접 계산함

**⇒ Key가 늘어날 때 Proof에서 커지는 부분은 경로 위의 Commitment뿐이다.**

### Key의 구성: Stem과 Suffix

앞의 "Verkle Tree의 구조"는 단순화한 것임. EIP-6800은 Account, Storage, Code를 모두 하나의 트리에 넣음. Key와 Value는 모두 32 Byte임. 이전 시점의 자료는 배치가 다를 수 있음. 예를 들어 Anatomy of a Verkle proof는 Balance와 Nonce를 서로 다른 Suffix에 두는 이전 배치로 설명함

Key 32 Byte는 앞 31 Byte인 **Stem**과 마지막 1 Byte인 **Suffix**로 나뉨

- **Stem**: 트리 안에서의 경로를 정함. Address와 Tree Index를 이어 붙여 Pedersen Hash에 넣은 결과의 앞 31 Byte임
    - Tree Index: 한 Account의 Value를 256개씩 묶은 그룹의 번호
    - Pedersen Hash: Pedersen Commitment를 Hash 함수처럼 쓴 것
- **Suffix**: 같은 Stem 아래에 있는 Value 256개 중 하나를 고름

Address 자체가 경로가 되는 것이 아님. Stem이 Hash의 결과이므로 Key는 트리 전체에 고르게 퍼짐

### 한 Stem에 묶이는 Value

Account의 첫 번째 Stem(Tree Index 0)에는 자주 함께 읽는 Value가 모여 있음. Storage Slot은 Contract Storage에서 Value 하나가 들어가는 자리이고, Chunk는 Code를 일정한 길이로 자른 조각임

| Suffix | 내용 | 상수 |
| --- | --- | --- |
| 0 | Version, Code Size, Nonce, Balance를 Value 하나에 묶음 | BASIC_DATA_LEAF_KEY = 0 |
| 1 | Code Hash | CODE_HASH_LEAF_KEY = 1 |
| 64~127 | Storage Slot 0~63 | HEADER_STORAGE_OFFSET = 64 |
| 128~255 | Code의 앞쪽 Chunk 128개 | CODE_OFFSET = 128 |

- 나머지 Storage Slot과 Chunk는 다른 Tree Index의 Stem으로 감
- Code는 31 Byte씩 나뉘고, 앞에 1 Byte를 붙인 32 Byte Chunk로 트리에 들어감. 앞 1 Byte는 그 Chunk의 첫머리가 PUSH 명령의 데이터일 때 그 데이터의 Byte 수임. PUSH는 뒤따르는 Byte를 데이터로 읽는 명령임

→ MPT와 달리 Code의 일부 Chunk만 증명할 수 있음

왜 Value를 Stem 단위로 묶는가?

- 같은 Stem에 속한 Value는 경로를 공유함. Nonce와 Balance, 앞쪽 Storage Slot을 함께 읽어도 경로 위의 Commitment는 Witness에 한 번만 들어감
- Gas 비용에도 반영함. EIP-6800은 Storage Slot 0~63의 접근 비용을 첫 Slot까지 포함해 2,100 Gas에서 200 Gas로, 그 밖의 Slot은 같은 그룹 안에서 두 번째 접근부터 200 Gas로 낮춘다고 적음. 세부 규칙은 [EIP-4762](https://eips.ethereum.org/EIPS/eip-4762)가 정함

### Node의 종류: Inner Node와 Extension Node

| Node | Commitment의 대상 | 역할 |
| --- | --- | --- |
| Inner Node | 자식 256개의 Commitment | Stem을 1 Byte씩 따라 분기함 |
| Extension Node | (1, Stem, C1, C2) | Stem 하나와 그 아래 Value 256개를 대표함 |

```
Inner Node → ... → Extension Node  Commit(1, Stem, C1, C2)
                     ├─ C1 = Commit(Suffix 0~127인 Value들의 조각 256개)
                     └─ C2 = Commit(Suffix 128~255인 Value들의 조각 256개)
```

- Extension Node는 MPT의 Extension Node와 이름만 같음. MPT에서는 분기 없는 경로 구간을 잇는 Node이고, 여기서는 Stem 하나에 속한 Value 256개를 담는 Node임. Stem도 남은 부분이 아니라 31 Byte 전체를 가짐
- 앞의 단순화한 구조에서 Leaf Node라고 부른 것이 실제 구조에서는 Extension Node와 C1, C2에 해당함
- 부모가 자식 Commitment를 담을 때는 Scalar로 변환해 항목에 넣음. Commitment는 타원곡선 점이고 항목은 Scalar이기 때문임

왜 C1과 C2 두 개로 나누는가?

- Commitment의 한 항목에는 252 Bit까지만 들어가는데 Value는 32 Byte(256 Bit)임
- 그래서 Value 하나를 16 Byte 두 조각으로 나눠 항목 두 개에 넣음
- Value 128개가 항목 256개를 채우므로 Commitment가 두 개 필요함. C1은 Suffix 0~127, C2는 Suffix 128~255를 담음
- 아래쪽 조각에는 $2^{128}$을 더함. Value가 0인 경우와 Value가 없는 경우를 구분하기 위해서임

### 트리의 평균 깊이

무작위 Key $n$개를 Width $d$인 트리에 넣으면 평균 깊이는 약 $\log_d n + 2/3$임. 여기서 깊이는 경로 위에 있는 Inner Node의 수임

- Dankrad Feist가 [Verkle trie for Eth1 state](https://dankradfeist.de/ethereum/2021/06/18/verkle-trie-for-eth1.html)에서 수치 실험으로 얻은 값이고, 유도는 실려 있지 않음
- $n=2^{30}$, $d=2^{10}$이면 약 3.67임
- EIP-6800의 구조에서는 Inner Node의 경로를 Stem이 정함. 따라서 이 식을 적용할 때 $n$은 Stem의 개수로 보는 것이 맞음 (출처에 없는 해석)

### 실제 구조에서 Value 하나의 검증

예시 1에서 Key의 첫 Byte를 Stem, 둘째 Byte를 Suffix로 보고 a1(Key 0x7340, Stem 0x73, Suffix 0x40)을 검증하면 Opening이 여섯 개 필요함. Anatomy of a Verkle proof가 나열하는 Opening 구성을 이 예시에 적용한 수임

| 대상 | Opening하는 Index | 확인하는 것 | 개수 |
| --- | --- | --- | --- |
| Root의 Commitment | 0x73 | a1의 Extension Node Commitment | 1 |
| Extension Node의 Commitment | 0, 1, 2 | Extension 표시 값 1, Stem, C1 | 3 |
| C1 | 128, 129 | Value의 아래쪽 조각과 위쪽 조각 | 2 |

- Suffix 0x40은 10진수로 64이므로 Value는 C1에 들어 있고, 조각의 Index는 2 × 64 = 128과 129임
- 같은 Stem에서 C1에 속한 다른 Value를 함께 읽으면 앞의 Opening 4개는 공유함
- Opening 여섯 개의 Opening Proof는 모두 Multiproof 하나에 묶임

### Value가 바뀔 때의 갱신

**문제 상황: Value가 바뀔 때마다 Commitment를 처음부터 다시 계산하면 Node마다 항목 256개에 대한 연산이 필요함**

**해결 방법: 경로 위의 Commitment만, 바뀐 분량만 더해서 갱신함**

Index $i$의 항목이 $v_i$에서 $v_i'$로 바뀌면 새 Commitment는 다음과 같음. Pedersen Commitment가 Additively Homomorphic이기 때문에 가능함

$$
C' = C + (v_i' - v_i)\,G_i
$$

- 나머지 255개 항목은 읽을 필요가 없음
- 타원곡선 점에 Scalar를 곱하는 연산(이하 타원곡선 곱셈) 1회와 점의 덧셈 1회로 끝남

경로를 따라 올라가는 순서

1. Value가 바뀌면 Extension Node의 C1 또는 C2를 위 식으로 갱신함. Value는 두 조각으로 나뉘어 있으므로 바뀐 조각마다 한 번씩 적용함
2. C1 또는 C2가 바뀌었으므로 Extension Node의 Commitment를 같은 식으로 갱신함
3. 부모 Inner Node는 자식 Commitment를 Scalar로 바꾼 값을 항목으로 가짐. 새 Scalar와 이전 Scalar의 차이로 부모를 같은 식으로 갱신함
4. Root까지 3을 반복함

비용

- Value 하나를 고치면 타원곡선 곱셈이 C1 또는 C2에서 최대 2회, Extension Node에서 1회, 경로 위 Inner Node마다 1회 듦. Inner Node가 3개면 최대 6회임 (구조에서 센 값이고 출처에 없는 계산)
- Vitalik Buterin의 2021년 글은 약 4회로 적음. Value가 Inner Node에 바로 달린 단순한 트리를 전제로, Root를 포함한 Commitment의 수를 센 것임. 같은 글에 따르면 기저점의 배수를 미리 계산해 두면 더 빨라짐

예시: 예시 2에서 a1이 a4에게 3 ETH를 보냄. 수수료를 받는 Account의 변경은 생략함. 그림은 단순화한 구조로 그린 것임

![verkle-update.png](verkle-update.png)

- 바뀌는 Value: a1(Nonce 4 → 5, Balance 10 → 7 ETH), a4(Balance 4 → 7 ETH)
- a1의 경로: a1의 Leaf Node → Index 0x0a 아래 Inner Node → Root
- a4의 경로: a4의 Leaf Node → Index 0xc4 아래 Inner Node → Index 0x0a 아래 Inner Node → Root
- 두 경로가 만나는 Index 0x0a 아래 Inner Node와 Root는 두 변경을 한 번에 반영함
- a2, a3, a5의 Leaf Node와 Index 0x62 아래 Inner Node는 그대로임
- 실제 구조에서는 Leaf Node 자리에서 C1 또는 C2, Extension Node의 Commitment 순으로 갱신한 뒤 같은 경로를 올라감

Opening Proof는 Node에 저장하지 않음. Block이 접근한 Value에 대해 Witness를 만들 때 계산함

<br />

## 평가

논문의 평가는 복잡도 분석과 구성 시간 측정이다. Ethereum 제안의 평가는 Proof와 Witness의 크기 계산이다. 둘을 나눠 적는다.

### 논문의 분석과 측정

복잡도는 다음과 같음. $n$은 Leaf의 수이고 $k$는 Width임

| 구조 | 트리 구성 | Value 하나 갱신 | Proof 크기 |
| --- | --- | --- | --- |
| Merkle Tree | $O(n)$ | $O(\log_2 n)$ | $O(\log_2 n)$ |
| Width $k$인 Merkle Tree | $O(n)$ | $O(k \log_k n)$ | $O(k \log_k n)$ |
| Vector Commitment | $O(n^2)$ | $O(n)$ | $O(1)$ |
| Width $k$인 Verkle Tree | $O(kn)$ | $O(k \log_k n)$ | $O(\log_k n)$ |

- Proof는 Width 2인 Merkle Tree보다 $\log_2 k$배 작음. 깊이가 $\log_2 n$에서 $\log_k n$으로 줄기 때문임
- 대가로 구성과 갱신에 $k$배의 계산이 듦. 논문의 구성은 Node를 만들 때 자식 $k$개 각각의 Opening Proof를 미리 계산해 두기 때문임
    - 트리 구성 $O(kn)$: Node 하나를 자식 $k$개의 증명까지 만드는 데 $O(k^2)$이 듦. 표의 Vector Commitment 행에 $n=k$를 넣은 값임. 맨 아래 Level에 Node가 $n/k$개이므로 $O(k^2) \times n/k = O(kn)$이고, 위 Level은 Node 수가 $k$배씩 줄어들어 합쳐도 $O(kn)$임 (논문에 없는 유도)
    - Value 하나 갱신 $O(k \log_k n)$: 경로 위 Node마다 Commitment와 자식 $k$개의 증명을 고쳐야 한다고 보면 맞는 값임 (논문에 없는 해석)

실험 설정

| 항목 | 값 |
| --- | --- |
| 구현 | C++, RELIC 라이브러리, SHA-256 |
| Vector Commitment | Catalano와 Fiore의 방식. Bilinear Group과 Computational Diffie-Hellman 가정에 기반함. 논문은 저자들이 아는 한 이 방식의 첫 구현이라고 적음 |
| 트리 형태 | Prefix Tree. Leaf는 Key와 Value의 쌍이고, Leaf의 위치는 Key의 Hash가 정함 |
| 측정 방법 | 빈 트리에서 시작해 Leaf를 하나씩 추가하며 걸린 시간을 잼 |
| 트리 크기 | Leaf $2^{14}$개 (16,384개) |

- 한꺼번에 구성하면 더 빠름. 하나씩 추가한 것은 Server가 새 Leaf를 계속 받아 트리를 갱신하는 상황을 재현하기 위해서임
- 논문은 Leaf $2^{14}$개의 결과가 더 큰 트리를 대표한다고 봄. 이 크기에서 이미 점근적 비용이 계산 시간을 지배한다는 이유임

결과

| Width $k$ | Proof 크기 (Width 2인 Merkle Tree 대비) | Leaf 1개당 구성 시간 | 전체 구성 시간 (직접 계산한 값) |
| --- | --- | --- | --- |
| 1024 | 10분의 1 | 110.1 ms | 약 1,804초 (약 30분) |
| 32 | 5분의 1 | 8.4 ms | 약 138초 |

- $k$를 1024에서 32로 낮추면 구성이 약 13배 빨라지고, Proof가 줄어드는 폭은 10배에서 5배로 낮아짐
- Proof 크기의 10배와 5배는 측정값이 아니라 $\log_2 1024 = 10$, $\log_2 32 = 5$임
- 논문의 판단: 갱신이 잦은 트리는 낮은 $k$를, 잘 바뀌지 않는 트리는 높은 $k$를 고름

**⇒ 논문이 보인 것은 Proof 크기와 계산량을 $k$로 맞바꿀 수 있다는 점이다.**

아래부터는 Ethereum 제안의 수치임. 논문의 비교 대상은 Width 2인 Merkle Tree이지만, Ethereum의 현재 트리는 Width가 16이어서 줄어드는 폭이 더 큼

### Key 하나의 Proof Size

Key 하나의 경로 크기임. $N=2^{32}$, Hash와 Commitment는 32 B임. 식은 2024년 글에 있음. MPT와 Binary Merkle Tree의 값은 글에 적혀 있고, Verkle Tree의 128 B는 식에 대입한 값임

| 트리 | 식 | $N=2^{32}$일 때 |
| --- | --- | --- |
| MPT (Width 16) | $32 \times 15 \times \log_{16} N$ | 3,840 B |
| Binary Merkle Tree (Width 2) | $32 \times \log_2 N$ | 1,024 B |
| Verkle Tree (Width 256) | $32 \times \log_{256} N$ | 128 B |

- Verkle Tree에는 여기에 Multiproof 하나가 더해짐. Key가 몇 개든 하나임
- Value $m$개를 증명할 때 MPT는 $O(mk \log_k N)$, Verkle Tree는 $O(m \log_k N)$에 고정 크기 Multiproof를 더한 값임

### Value 1,000개의 Proof Size

$N$ = 1,000,000, 증명할 Value $m$ = 1,000개일 때의 계산임 (출처에 없는 계산)

- 깊이: MPT는 $\log_{16} 10^6 \approx 4.98$이므로 5, Verkle Tree는 $\log_{256} 10^6 \approx 2.49$이므로 3으로 놓음
- MPT는 Level마다 형제 Hash 15개(각 32 B)를 넣음
- Verkle Tree는 Level마다 Commitment 1개(32 B)를 넣고, Multiproof 하나(576 B)를 더함

| 트리 | 계산 | 크기 |
| --- | --- | --- |
| MPT | 1,000 × 15 × 5 × 32 B | 2,400,000 B (2.4 MB) |
| Verkle Tree | 1,000 × 3 × 32 B + 576 B | 96,576 B (약 97 kB) |

- 약 25배 차이임
- KZG 기준(Commitment 48 B, Multiproof 96 B)으로 계산하면 1,000 × 3 × 48 + 96 = 144,096 B(약 144 kB)이고 약 17배 차이임
- 두 값 모두 상한임. 여러 Key가 상위 Level의 Node를 공유하면 같은 Hash나 Commitment는 한 번만 넣으므로 실제 크기는 더 작음. Key와 Value 자체의 크기는 포함하지 않음

출처에 적힌 수치는 다음과 같음

- Vitalik Buterin의 2021년 글의 표임. Width 256, Commitment와 Opening Proof가 48 B인 KZG, Key가 고르게 분포한 트리를 가정함. 단위는 B임

    | 트리 크기 | Key 1개 | Key 1,000개 | Key 10,000개 |
    | --- | --- | --- | --- |
    | $2^{24}$ | 272 | 59,792 | 457,616 |
    | $2^{32}$ | 320 | 107,744 | 937,472 |

- 표의 값은 경로 위 Commitment와 Multiproof만 센 것으로 읽힘. Key 1개일 때 $2^{32}$ 행의 320 B가 176 + 3 × 48과 같기 때문임. Key와 Value 자체는 들어 있지 않음 (글에 명시되어 있지 않고 값에서 역산한 해석)
- $2^{32}$ 크기의 트리에서 Key 1,000개일 때 Key당 약 108 B임. Key 1개일 때의 320 B보다 훨씬 작음. 상위 Level의 Commitment를 여러 Key가 공유하고 Multiproof가 하나뿐이기 때문임
- 같은 글의 보정: Key가 무작위로 분포하면 깊이가 약 0.6 늘어 Key당 약 30 B가 더 듦. 32 B Commitment를 쓰면 크기가 약 3분의 1 줄어듦
- 같은 글의 결론: Verkle Tree의 Proof는 Width 16인 MPT보다 20~30배 이상, 이상적인 Binary Merkle Tree보다 6~8배 작음
- [ethereum.org](https://ethereum.org/roadmap/verkle-trees/)는 Value 1,000개 기준으로 MPT의 Witness 약 3.5 MB(7 Level), Verkle Tree의 Witness 약 150 kB(4 Level)를 들고 약 23배 작다고 적음

### Block 하나의 Witness

Block 하나의 최악 Witness임. 2024년 글의 계산이고, Gas Limit 30,000,000과 접근 1회당 2,400 Gas(Block당 최대 12,500회 접근)를 가정함

| 트리 | 최악의 Witness | 비고 |
| --- | --- | --- |
| MPT | 약 330 MB | 접근마다 Code 전체(최대 24,000 B) 포함 |
| Binary Merkle Tree, Code를 트리에 포함 | 약 10.4 MB | |
| Verkle Tree | 약 1.3 MB | 공격자가 긴 공통 앞부분을 가진 Address를 만들면 약 2.6 MB |
| Verkle Tree, 인접 접근만 한 경우 | 약 4.8 MB | 인접 접근 1회당 200 Gas, 접근마다 32 B를 가정 |

- Binary Merkle Tree 행의 식은 12,500 × 32 × (32 - 14 + 8)이고, Verkle Tree 행은 같은 식을 8로 나눈 값임
- 마지막 행의 식은 30,000,000 / 200 × 32임. 글에는 4,800,800 B로 적혀 있으나 식을 계산하면 4,800,000 B임

### 검증과 생성 비용

Multiproof를 쓴 Witness의 비용임 (Vitalik Buterin의 2021년 글)

- **검증**: Value $m$개를 증명할 때 크기 $m$의 Multiexponentiation 1회와 약 $4m$회의 Field 연산이 대부분임. Multiexponentiation은 여러 타원곡선 점에 각각 Scalar를 곱해 모두 더하는 연산이고, Field 연산은 큰 소수를 법으로 하는 덧셈과 곱셈임. 글은 Value 수천 개의 검증이 100 ms 안에 끝난다고 적음. 2021년의 추정이고 측정 환경은 적혀 있지 않음
- **생성**: 경로 위 각 Node의 항목 256개가 모두 필요함. 그래서 State 전체를 가진 Block 생성 Node가 Witness를 만듦

<br />

## 한계와 주의

### 논문

- **측정 범위**: 측정한 것은 트리 구성 시간뿐임

    - Proof의 실제 크기, Proof를 만드는 시간, 검증하는 시간은 측정하지 않음. 측정에 쓴 기기의 사양도 적혀 있지 않음
    - 주의: $\log_2 k$배는 Level 수의 비임. Level마다 들어가는 데이터의 크기가 Hash 하나와 같을 때에만 Proof 크기의 비가 됨. 앞의 a1 검증 예시에서도 Level마다 Opening Proof 하나가 들고 중간 Commitment인 $C_{115}$를 함께 보냄. 논문은 증명 하나의 크기를 Hash와 비교하지 않음 (논문에 없는 판단)

- **계산 비용**: 구성과 갱신에 $k$배의 계산이 듦

    - $k=1024$에서는 Leaf 16,384개를 구성하는 데 약 30분이 걸림 (직접 계산한 값). 논문도 상대적으로 느리다고 적음
    - 논문이 든 향후 과제는 구현의 병렬화와, 갱신을 묶어 처리하는 Batching임

- **여러 Value의 증명**: 논문은 Proof를 묶는 방법을 다루지 않음. Value $m$개를 증명하면 Proof도 $m$에 비례해 늚

- **안전성**: Verkle Tree의 Proof를 위조할 수 없다는 논증이 따로 없음

    - Merkle Tree에 대해서는 위조하려면 Hash의 Collision Resistance를 깨야 한다고 설명함. Verkle Tree에 대해서는 대응하는 설명이 없음 (논문에 없는 판단)

- **서술의 불일치**: 대역폭과 계산 자원 가운데 어느 쪽이 비싼지에 대한 서술이 두 곳에서 반대임

    - 한 곳은 대역폭이 계산 자원보다 훨씬 비싸다고 쓰고, 다른 곳은 대역폭이 계산 자원보다 싸다고 씀
    - 계산을 더 써서 대역폭을 줄이는 설계이므로 앞의 서술이 논지에 맞음 (논문에 없는 판단)

### Ethereum 제안

앞의 세 가지는 Vitalik Buterin의 [2024년 글](https://vitalik.eth.limo/general/2024/10/23/futures4.html)이 Verkle Tree의 단점으로 든 것이고, 나머지는 같은 글의 수치와 앞의 설명에서 읽은 것임

- **Quantum Resistance가 없음**: Commitment가 타원곡선에 기반하므로 양자 컴퓨터에 취약함. 글은 MPT를 Verkle Tree로 바꾸면 나중에 트리를 다시 바꿔야 한다고 적음
- **SNARK 안에서 증명하기 어려움**: Succinct Non-interactive Argument of Knowledge(이하 SNARK)는 계산이 올바르게 수행됐음을 짧은 증명으로 보이는 방식임. Verkle Tree의 Proof를 검증하는 계산에는 256 Bit 연산이 많아, 이를 SNARK로 증명하려면 큰 오버헤드나 전용 구조가 필요함. 글은 이것이 Stateless 검증 자체의 문제는 아니고 그 뒤 단계에서 어려움이 된다고 적음
- **State Root 갱신이 Hash 기반 트리보다 느림**: State 전체를 저장하는 Node가 Chain을 따라잡는 Sync 시간이 길어짐
- **최악의 Witness는 여전히 MB 단위임**: 평가의 "Block 하나의 Witness" 표에서 약 1.3~4.8 MB임
- **Width에 상한이 있음**: 트리가 너무 넓어지면 Proof 계산이 비효율적이 됨
- **Block을 만드는 Node는 State 전체가 필요함**: Witness를 만들려면 경로 위 Node의 항목이 모두 필요함. 그래서 목표가 Weak Statelessness임

같은 글이 든 Verkle Tree의 장점은 세 가지임. 최악의 경우에도 Proof 생성이 1초 미만이고, 구현이 성숙했고, Witness를 다음 Block에 맞춰 갱신할 수 있음. 다만 같은 글은 Merkle Proof도 갱신할 수 있다고 적으므로 마지막 장점은 Verkle Tree만의 것이 아님

### 대안: STARK로 증명하는 Binary Hash Tree

Binary Merkle Tree를 쓰고, Merkle Proof 전체를 Scalable Transparent Argument of Knowledge(이하 STARK)로 증명해 Proof를 대체하는 방식임. STARK는 SNARK와 같은 목적의 증명 방식이고, 이 방식은 타원곡선 없이 Hash 함수에만 의존함

아래는 2024년 10월 시점의 비교임. 보수적 Hash는 오래 쓰여 검증된 Hash 함수이고, 공격적 Hash는 증명에 유리하도록 새로 설계되어 검증이 덜 된 Hash 함수임

| 방식 | Proof 크기 | 보안 가정 | 최악의 증명 생성 시간 |
| --- | --- | --- | --- |
| Verkle Tree | 데이터 + 약 100~2,000 kB | 타원곡선 (Quantum Resistance 없음) | 1초 미만 |
| STARK + 보수적 Hash (SHA256, BLAKE) | 데이터 + 약 100~300 kB | 보수적 Hash 함수 | 10초 초과 |
| STARK + 공격적 Hash (Poseidon, Ajtai) | 데이터 + 약 100~300 kB | 덜 검증된 새 Hash 함수 | 1~2초 |

- 같은 글은 최악의 경우 약 660,000번의 Hash를 증명해야 하고, 초당 약 200,000번을 증명할 수 있으면 충분하다고 적음. 보수적 Hash는 당시 일반 노트북에서 초당 약 10,000~30,000번이었음
- 글은 세 방식 가운데 하나를 고르지 않음. 곧 결정해야 한다고 적고, Verkle Tree가 아닌 방식을 고르면 배포가 Hard Fork 한 번쯤 늦어질 것으로 봄

**⇒ 2024년 10월에는 Verkle Tree가 "지금 준비된 방식"이었고 Binary Hash Tree는 "증명이 아직 느린 방식"이었다. 이 저울이 그 뒤 어느 쪽으로 기울었는지는 [2편]({% post_url Paper-Review/2026-11-10-Verkle-Tree-Roadmap %})에서 다룬다.**

<br />

## 결론

Verkle Tree는 Merkle Tree의 Hash 자리에 Vector Commitment를 넣은 트리다. 논문은 이 구조가 Proof를 Width 2인 Merkle Tree보다 $\log_2 k$배 줄이고, 대신 구성과 갱신에 $k$배의 계산을 쓴다는 것을 보였다.

Ethereum은 이 아이디어를 Stateless Client에 필요한 Witness를 줄이는 데 적용했다. Witness 크기는 State Tree의 Proof 크기가 정한다. MPT는 Hash로 부모 Node를 만들기 때문에 Level마다 형제 Hash를 보내야 하고, Proof가 (Width - 1) × 깊이에 비례한다. Verkle Tree는 부모 Node를 Vector Commitment로 만들어 형제를 보내지 않는다. 형제가 없으므로 Width를 256으로 키워 깊이를 줄일 수 있고, 여러 Opening Proof는 Multiproof 하나로 묶인다. Opening Proof를 저장하지 않고 Commitment를 덧셈으로 갱신하므로 갱신 비용은 Width에 비례하지 않는다. 그 결과 Key 하나의 경로 크기는 3,840 B에서 128 B로, Value 1,000개의 Witness는 약 3.5 MB에서 약 150 kB로 줄어든다.

Roadmap에서 Verkle Tree는 The Verge의 첫 단계였고, 2024년 2월에는 Osaka Upgrade에 배정됐다. 그러나 2026년 11월 24일 기준 Ethereum은 Verkle Tree를 채택하지 않았다. 타원곡선에 기반한 Commitment에 Quantum Resistance가 없어 트리를 두 번 바꿔야 한다는 점, 증명 시스템의 발전으로 Hash 기반 트리의 약점이 줄었다는 점, 모든 실행을 SNARK로 증명하려면 증명하기 쉬운 트리가 필요하다는 점이 이유다. 그 자리에는 Hash 기반 Binary Tree(EIP-7864를 거쳐 EIP-8297)가 들어왔고, 적용할 Upgrade는 아직 정해지지 않았다. Stateless Client라는 목표와, 함께 읽는 Value를 같은 Stem에 묶는 설계는 이어진다. 자세한 경위는 [2편]({% post_url Paper-Review/2026-11-10-Verkle-Tree-Roadmap %})에 있다.
