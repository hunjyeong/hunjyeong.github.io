---
title: "Verkle Tree"
date: 2026-11-10
last_modified_at: 2026-11-26
categories: [Paper Review]
tags: [Blockchain, Ethereum, Verkle Tree, Merkle Patricia Trie, Stateless Client, Commitment]
math: true
media_subpath: /assets/img/posts/verkle-tree/
---

>
> 1. **Verkle Tree**
> 2. [Verkle Tree and the Ethereum Roadmap]({% post_url Paper-Review/2026-11-10-Verkle-Tree-Roadmap %})

> 관련 글: [Multiproof]({% post_url Cryptography/2026-11-10-Multiproof %})

### Paper Information

| Item | Details |
| --- | --- |
| Title | Verkle Trees |
| Author | John Kuszmaul (Mentor: Alin Tomescu) |
| Venue | MIT(Massachusetts Institute of Technology) PRIMES(Program for Research In Mathematics, Engineering, and Science) 2018 Computer Science Conference |
| Link | [Paper](https://math.mit.edu/research/highschool/primes/materials/2018/Kuszmaul.pdf) |

---

### TL;DR

Verkle Tree는 Merkle Tree의 Parent Node를 Hash 대신 Vector Commitment로 만든 Tree다. Child 하나를 증명할 때 Sibling이 필요 없으므로, Width(폭)를 넓혀 Tree Depth(깊이)를 줄여도 Proof가 커지지 않는다.

Ethereum은 전체 State를 저장하지 않고 Witness로 Block을 검증하는 Stateless Client를 위해, 2021년부터 State Tree를 Verkle Tree로 바꾸는 방안을 연구했다. 이 방안을 정의한 EIP(Ethereum Improvement Proposal, Ethereum의 표준 제안 문서)인 [EIP-6800](https://eips.ethereum.org/EIPS/eip-6800)은 2026년 11월 현재 Stagnant(진행이 멈춘 상태)다. [ethereum.org](https://ethereum.org/roadmap/statelessness/)는 Hash 기반 Binary Tree를 쓰는 EIP-7864를 현재 제안으로 적는다. [EIP-7864](https://eips.ethereum.org/EIPS/eip-7864)는 Verkle Tree를 쓰지 않는 이유로 Elliptic Curve(타원곡선) 암호에 Quantum Resistance(양자 내성)가 없다는 점을 든다. 이 경위는 [2편]({% post_url Paper-Review/2026-11-10-Verkle-Tree-Roadmap %})에서 다룬다.

<br />

## Merkle Tree

### What is a Merkle Tree?

Merkle Tree는 Child(자식) Node들의 Hash를 이어 붙여 다시 Hash한 값을 Parent(부모) Node로 삼는 Tree다. 맨 위 Node인 Root의 Hash를 Merkle Root라 하고, 이 값 하나가 전체 Data를 대표한다. Node 하나가 가질 수 있는 최대 Child 수를 Width라 한다. 이 글은 Width가 2인 Binary Merkle Tree를 기준으로 설명한다.

### Merkle Tree Structure

1. Data를 Hash하여 Leaf Node 생성

    - Leaf Node: h1, h2, h3, h4

2. Node를 Left부터 두 개씩 묶고, 두 Hash를 (Left, Right) 순서로 이어 붙여 Hash하여 Parent Node 생성

    - Parent Node: h5 = hash(h1, h2), h6 = hash(h3, h4)

3. Node가 하나 남을 때까지 2를 반복. 남은 Node의 Hash가 Merkle Root

    - Merkle Root: h7 = hash(h5, h6)

![merkle-tree.png](merkle-tree.png)

- Level: Root까지의 Edge(간선) 수가 같은 Node의 모음. (그림에서 3개)

- Tree Depth: Root에서 Leaf까지의 Edge 수. (그림에서 2)

- Node 수가 홀수일 때 짝 없는 Node의 처리 규칙은 구현마다 다름. 이 글은 Leaf 개수가 2의 거듭제곱인 경우만 다룸

#### Key Properties

Data가 바뀌었는데 Merkle Root가 그대로이려면 Path(경로) 어딘가에서 Hash Collision(충돌)이 일어나야 한다. Hash Function(해시 함수)의 Collision Resistance(충돌 저항성) 때문에 이는 사실상 불가능하므로, Merkle Root 하나만 신뢰할 수 있으면 전체 Data를 검증할 수 있다.

단, Leaf와 Internal Node(내부 Node)를 같은 방식으로 Hash하면 Internal Node의 Input(입력), 즉 두 Child Hash를 이어 붙인 값을 Leaf Data로 내세워 Collision 없이 통과시킬 수 있다. 그래서 Certificate Transparency 표준인 RFC(Request for Comments) 6962는 Leaf와 Internal Node에 서로 다른 Prefix(접두어)를 붙여 Hash한다 (0x00 / 0x01).

### Merkle Proof

Merkle Proof는 Data 하나가 i번째 Leaf에 있음을 보이는 Data다.

- Prover(증명자): Merkle Proof를 만들어 보내는 쪽

- Verifier(검증자): Merkle Proof를 확인하는 쪽. 신뢰할 수 있는 Merkle Root를 알고 있음

Parent가 같은 Node를 Sibling(형제)이라 한다. Parent의 Hash를 계산하려면 두 Child의 Hash와 순서가 필요하다.

**Proof 구성**

- Root를 제외한 Level마다 Sibling의 Hash와 그 위치(Left/Right)

- 위치 대신 Leaf Index를 보내도 됨

    - Leaf Index: Left부터 0으로 센 Leaf의 위치

    - Index를 Tree Depth만큼의 자릿수로 Binary(2진수)로 쓰면, 왼쪽 Bit부터 차례로 Root 바로 아래 Level부터 Leaf Level까지 Path 위 Node가 Left(0)인지 Right(1)인지를 나타냄. Sibling은 그 반대쪽

**Example**: Data1이 첫 번째 Leaf(Index 0)에 있음을 증명한다.

1. Prover가 Data1과 (h2, Right), (h6, Right)를 보낸다.

2. Verifier가 Data1의 Hash를 계산한다. → h1

3. Verifier가 hash(h1, h2)를 계산한다. → h5

4. Verifier가 hash(h5, h6)를 계산한다. → Merkle Root와 같으면 Accept

Data2, Data3, Data4는 보내지 않는다.

Leaf와 Internal Node를 구분해 Hash하지 않는 구현에서는 Verifier가 Tree Depth를 알고, Proof Length(Proof 길이)가 그와 같은지 확인해야 한다.

### Proof Size

Leaf가 $n$개일 때:

- Tree Depth: $\log_2 n$

- Proof의 Hash: Root를 제외한 Level마다 1개씩, $\log_2 n$개

- Verifier의 Hash 계산: Leaf Data 1회 + Sibling Hash 하나당 1회, $\log_2 n + 1$회

- Leaf가 $2^{32}$개이고 Hash가 32 Bytes일 때 Sibling Hash의 크기: 32 × 32 = 1,024 Bytes, 약 1 kB. 위치 정보 32 Bits는 별도

**⇒ Verifier는 Root 하나만 저장하고도, Data 하나와 짧은 Proof로 포함 여부를 확인할 수 있다. 대신 Level마다 Sibling이 필요하다는 점이 뒤에서 문제가 된다.**

<br />

## Merkle Patricia Trie (MPT)

### What is a Merkle Patricia Trie?

[Merkle Patricia Trie(MPT)](https://ethereum.org/developers/docs/data-structures-and-encoding/patricia-merkle-trie/)는 Merkle Tree와 Patricia Trie를 결합한 자료구조로, Key-Value Data를 효율적으로 저장, 탐색, 수정하고 Data의 무결성을 암호학적으로 검증할 수 있다.

Merkle Tree는 Data의 위치가 Leaf Index로 정해져서 Key로 Value를 찾을 수 없다. Ethereum은 Key로 Value를 찾아야 하므로 MPT를 쓴다.

- **Merkle Tree**: Hash로 Data의 무결성을 검증하는 Tree다. Data가 하나라도 바뀌면 Merkle Root가 바뀌므로, Merkle Root 하나로 전체 Data를 검증할 수 있다.

- **Patricia Trie**: Trie에서 분기 없이 이어지는 Path를 Node 하나로 압축한 구조다. Tree Depth와 Node 수가 줄어든다.

    - Trie (Prefix Tree): Key를 문자 단위로 나누고, Root에서부터 한 문자씩 따라 내려가며 Value를 찾는 Tree다. Key가 곧 Root에서 Value까지의 Path이고, 공통 Prefix를 가진 Key들은 그 Prefix만큼 같은 Path를 공유한다.

![patricia-trie.png](patricia-trie.png)

즉, MPT는 Patricia Trie의 Path 압축 및 탐색 방식에 Merkle Tree의 Hash 기반 검증 기능을 결합한 구조다.

#### MPT in Ethereum

Ethereum에서 Transaction을 실행하고 State(상태)를 관리하는 계층을 Execution Layer(실행 계층)라고 한다. Execution Layer의 Trie는 모두 MPT다. Ethereum의 MPT는 일반적인 구조를 일부 수정한 것이어서 Modified Merkle Patricia Trie라고 부른다.

- **Keccak-256**: Ethereum이 사용하는 Hash Function. 32 Bytes(256 Bits) 길이의 Hash를 출력함

- **[Recursive Length Prefix(RLP)](https://ethereum.org/developers/docs/data-structures-and-encoding/rlp/)**: Byte 배열과 그 중첩 목록을 하나의 Byte 배열로 변환하는 Encoding 방식. Execution Layer에서 Data의 Serialization(직렬화)에 사용함

| Trie | Key | Value | Merkle Root 저장 위치 |
| --- | --- | --- | --- |
| State Trie | `Keccak-256(Address)` | `RLP(Account State)` | Block Header의 `stateRoot` |
| Storage Trie | `Keccak-256(Storage Slot)` | `RLP(Storage Value)` | Account State의 `storageRoot` |
| Transaction Trie | `RLP(Transaction Index)` | Encoding된 Transaction | Block Header의 `transactionsRoot` |
| Receipt Trie | `RLP(Transaction Index)` | Encoding된 Receipt | Block Header의 `receiptsRoot` |
| Withdrawals Trie | `RLP(목록 내 위치)` | `RLP(Withdrawal)` | Block Header의 `withdrawalsRoot` |

State Trie는 Address(주소)로 Account(계정)의 State를 찾는 Trie다. Account State는 nonce(Account가 보낸 Transaction 수), balance(잔액), storageRoot, codeHash의 네 값이다. Storage Trie는 Account마다 하나씩 있고, Transaction Trie, Receipt Trie, Withdrawals Trie는 Block마다 하나씩 만든다. Withdrawals Trie는 Block에 담긴 Withdrawal(출금) 목록으로 만들며, [Transaction Trie와 같은 방식으로 구성한다](https://eips.ethereum.org/EIPS/eip-4895). Key는 Withdrawal이 가진 `index` 값이 아니라 목록 안에서의 위치다. 이 글은 State Trie를 기준으로 설명하며, State Trie의 Merkle Root를 State Root라 한다.

### Node Types

Ethereum의 MPT는 Key를 Nibble(4 Bits, 16진수 한 자리) 단위로 나누어 Path를 구성한다. Nibble 하나는 0부터 f까지 16가지 값을 가지므로 한 Node의 Child는 최대 16개, 즉 Width는 16이다. State Trie의 Key는 32 Bytes이므로 Path는 64 Nibble이다.

| Node | 형태 | 역할 |
| --- | --- | --- |
| NULL | Empty Byte String (RLP: 0x80) | Node가 없음을 나타낸다 |
| Branch Node | 17-item `[v0 ... v15, vt]` | Nibble 0~f에 해당하는 자식 16개와 Value 1개 |
| Extension Node | 2-item `[encodedPath, key]` | 분기 없는 공통 경로를 압축하여 저장하고 다음 Node를 참조 |
| Leaf Node | 2-item `[encodedPath, value]` | Key의 남은 경로와 해당 Value를 저장 |

아래에서는 Branch Node에서 Nibble x에 해당하는 항목을 항목 x라 한다. `vt`는 Key가 그 Branch Node에서 끝날 때의 Value다. State Trie는 Key 길이가 모두 같아 Branch Node에서 끝나는 Key가 없으므로 `vt`는 항상 비어 있다.

**Hex-Prefix(HP) Encoding**

Extension Node와 Leaf Node는 둘 다 항목이 2개라서 형태만으로는 구분할 수 없다. 또 Path는 Nibble 단위인데 저장은 Byte 단위라서, Nibble 개수가 홀수인 Path는 그대로 저장할 수 없다. 예를 들어 Nibble `1` 하나와 Nibble `0`, `1` 두 개는 모두 Byte `01`로 저장되어 구분되지 않는다.

[HP Encoding](https://ethereum.org/developers/docs/data-structures-and-encoding/patricia-merkle-trie/#optimization)은 Path 앞에 Flag Nibble 하나를 붙여 이 두 문제를 해결한다. Flag는 Node 종류와 Path 길이의 홀짝을 나타낸다. Path 길이가 짝수이면 Flag 뒤에 Padding Nibble `0`을 하나 더 붙여, 전체 Nibble 개수를 항상 짝수로 만든다. 이렇게 만든 값이 `encodedPath`다.

| Flag | Node | Path 길이 | 예시 Path | Encoding 결과 |
| --- | --- | --- | --- | --- |
| `0` | Extension Node | 짝수 | `0 1 2 3 4 5` | `00 01 23 45` |
| `1` | Extension Node | 홀수 | `1 2 3 4 5` | `11 23 45` |
| `2` | Leaf Node | 짝수 | `0 f 1 c b 8` | `20 0f 1c b8` |
| `3` | Leaf Node | 홀수 | `f 1 c b 8` | `3f 1c b8` |

### How MPT Works

**Node의 참조 방식**

Parent가 Child를 담는 방식은 Child를 RLP로 Encoding한 결과의 길이에 따라 [달라진다](https://ethereum.org/developers/docs/data-structures-and-encoding/patricia-merkle-trie/).

- 32 Bytes 미만: Child 자체를 Parent 안에 그대로 넣는다.

- 32 Bytes 이상: Child의 RLP Encoding 결과를 Keccak-256으로 Hash한 값을 넣는다.

Root Node는 길이와 상관없이 Hash하며, 이 값이 Merkle Root다.

**탐색 과정**

Key를 Nibble 열로 바꾸고, Root Node에서 시작해 Node 종류에 따라 내려간다.

- Branch Node: 다음 Nibble 하나에 해당하는 항목의 Child로 이동한다.

- Extension Node: `encodedPath`의 Path가 남은 Nibble의 앞부분과 같으면 그 길이만큼 건너뛰고 `key`가 가리키는 Child로 이동한다.

- Leaf Node: `encodedPath`의 Path가 남은 Nibble 전체와 같으면 `value`가 찾는 Value다.

- Path가 다르거나 빈 항목(NULL)에 도달하면 그 Key는 Trie에 없다.

![mpt-st.png](mpt-st.png)

**Example**

그림은 Key 4개를 담은 State Trie다. 설명을 위해 Key를 7 Nibble로, Value를 잔액으로 줄였다.

| Key | Value |
| --- | --- |
| a711355 | 45.0 ETH |
| a77d337 | 1.00 WEI |
| a7f9365 | 1.1 ETH |
| a77d397 | 0.12 ETH |

그림의 표기는 Node Types 표와 다음과 같이 대응한다.

- Extension Node: `prefix`와 `shared nibble(s)`를 합친 것이 `encodedPath`, `next node`가 `key`

- Leaf Node: `prefix`와 `key-end`를 합친 것이 `encodedPath`

- `prefix`는 HP Encoding의 Flag다. 그림은 Padding Nibble을 생략했다.

아래에서는 Extension Node를 위에서부터 E1, E2, Branch Node를 위에서부터 B1, B2라 한다. Leaf Node는 Value가 45.0 ETH, 1.00 WEI, 0.12 ETH, 1.1 ETH인 순서로 L1, L2, L3, L4라 한다.

- 네 Key가 모두 a7로 시작하므로 Root Node는 `shared nibble(s)`가 a7인 Extension Node E1이다. 그림의 `stateRoot`가 State Root이고, E1의 Hash다.

- 세 번째 Nibble이 1, 7, f로 갈리므로 Branch Node B1이 온다. B1의 항목 1은 L1, 항목 f는 L4를 가리키고, 두 Leaf Node는 남은 Nibble 1355와 9365를 `key-end`로 가진다.

- a77로 시작하는 두 Key는 다음 두 Nibble d3도 같으므로 B1의 항목 7은 Extension Node E2를 가리킨다. 그다음 Nibble이 3과 9로 갈리므로 Branch Node B2가 온다. B2의 항목 3은 L2, 항목 9는 L3을 가리키고, 두 Leaf Node는 마지막 Nibble 7을 `key-end`로 가진다.

- a77d397을 찾으면 a7(E1) → 7(B1) → d3(E2) → 9(B2) → 7(L3) 순서로 Node 5개를 거쳐 0.12 ETH를 얻는다. a711355는 a7(E1) → 1(B1) → 1355(L1)로 Node 3개만 거친다.

- Flag는 Path 길이가 짝수인 E1과 E2가 0, L1과 L4가 2이고, 홀수인 L2와 L3이 3이다.

#### Key Properties

- (Key, Value)의 집합이 같으면 넣은 순서와 상관없이 Trie의 모양과 Merkle Root가 같다.

- Value 하나가 바뀌면 그 Key의 Path 위 Node만 Hash가 바뀌고, 이 변화가 Root Node까지 이어져 Merkle Root가 바뀐다.

- Root Node에서 Leaf Node까지 거치는 Node 수가 Key마다 다르다.

### Merkle Proof in MPT

MPT의 Merkle Proof는 Root Node부터 Key의 Path를 따라 만나는 [Node 전부](https://eips.ethereum.org/EIPS/eip-1186)다. 각 Node를 RLP로 Encoding한 값을 보내며, Parent 안에 그대로 들어 있는 Node는 따로 보내지 않는다. Verifier는 신뢰할 수 있는 State Root를 알고 있다.

1. 첫 Node의 Hash가 State Root와 같은지 확인한다.

2. Key의 Nibble을 따라 내려가며, 현재 Node에서 Child를 가리키는 항목의 Hash가 다음 Node의 Hash와 같은지 확인한다.

3. Leaf Node의 Path가 남은 Nibble 전체와 같으면 그 Value를 Accept한다.

**Example**

위 그림에서 Key a77d397의 Value가 0.12 ETH임을 증명한다. hash(X)는 X를 RLP로 Encoding한 값의 Keccak-256 Hash이고, 모든 Node의 RLP가 32 Bytes 이상이라고 가정한다.

1. Prover가 E1, B1, E2, B2, L3을 보낸다.

2. Verifier가 hash(E1)이 State Root와 같은지 확인한다. E1의 `shared nibble(s)` a7이 Key의 앞부분과 같고, E1의 `next node`가 hash(B1)과 같은지 확인한다.

3. B1의 항목 7이 hash(E2)와 같은지 확인한다.

4. E2의 `shared nibble(s)` d3이 Key의 다음 부분과 같고, E2의 `next node`가 hash(B2)와 같은지 확인한다.

5. B2의 항목 9가 hash(L3)과 같은지 확인한다.

6. L3의 `key-end` 7이 남은 Nibble과 같다. → 0.12 ETH를 Accept

Branch Node의 Hash를 계산하려면 그 Node의 항목이 모두 필요하다. 그래서 Binary Merkle Tree에서는 Level마다 Sibling의 Hash가 1개였지만, MPT에서는 Branch Node 하나마다 Sibling의 Hash가 최대 15개 들어간다. 위 예시에서 B1을 보내면 L1과 L4의 Hash가, B2를 보내면 L2의 Hash가 함께 간다.

Key가 없다는 것도 같은 방식으로 증명한다. Path를 따라가다 빈 항목을 만나거나, Extension Node나 Leaf Node의 Path가 Key와 다름을 보이면 된다.

### Proof Size

Key가 $n$개이고 고르게 퍼져 있으며, Path 위 Branch Node의 항목 16개가 모두 찼다고 가정한 근삿값이다. Extension Node와 Leaf Node의 크기, RLP의 길이 표시는 넣지 않았다.

- Branch Node의 Level 수: $\log_{16} n = \frac{1}{4}\log_2 n$. Binary Merkle Tree의 1/4

- Level 하나의 Sibling Hash: 15개 × 32 Bytes = 480 Bytes

- Proof의 크기: $480 \times \log_{16} n = 120 \times \log_2 n$ Bytes. Vitalik Buterin의 2024년 글 [The Verge](https://vitalik.eth.limo/general/2024/10/23/futures4.html)의 식이다.

- Key가 $2^{32}$개일 때: 120 × 32 = 3,840 Bytes. 같은 개수의 Binary Merkle Tree(1,024 Bytes)의 3.75배

- 실제 Proof는 Branch Node를 통째로 보내므로, Verifier가 직접 계산할 수 있는 Path 위 Child의 Hash까지 들어가 Level마다 Hash가 16개(512 Bytes)다. 이 기준으로는 [$128 \times \log_2 n$ Bytes](https://hackmd.io/@vbuterin/BylgAm7ZF)이고, Key가 $2^{32}$개일 때 4,096 Bytes로 Binary Merkle Tree의 4배다.

- Verifier의 Hash 계산: Proof의 Node 하나당 1회

이후 이 글의 크기 비교는 Sibling만 세는 앞의 식(Level마다 15개)을 기준으로 한다.

**⇒ MPT는 Key로 Value를 찾을 수 있는 Merkle Tree다. Width가 16이라 Level 수는 Binary Merkle Tree의 1/4이지만, Level마다 Sibling Hash가 최대 15개 필요해 Proof는 약 4배로 커진다.**

<br />

## State Growth Problem

### State Growth

Ethereum Full Node는 Block을 검증하기 위해 전체 State를 저장한다. [Ethereum Research의 실험](https://ethresear.ch/t/what-if-we-only-kept-1-year-of-active-state/23980)에서 Geth(go-ethereum, Execution Layer를 구현한 Client 중 하나) v1.16.5의 Database에 담긴 State는 359.03 GB였다. Account와 Storage 각각의 Snapshot(Geth가 Trie와 별도로 두는, Leaf를 평탄하게 펼친 저장 구조)과 Trie Node를 합한 값이다.

State가 커질수록 Node 운영에 필요한 Disk 공간과 Hardware 사양도 커진다. 일반 사용자가 직접 Node를 운영하기 어려워지면, Block을 스스로 검증하는 Node가 줄어들 수 있다.

### → Stateless Client로 State 저장 부담 없애기

Stateless Client는 전체 State를 저장하지 않는다. 대신 Block마다 그 Block을 실행하는 데 필요한 State와, 그 State가 State Root에 속함을 보이는 Proof를 함께 받아 검증한다. 이 묶음을 Witness라 한다.

- **Stateless Client**: 자신이 가진 State Root, 받은 Block, 그 Block의 Witness로 Block을 검증하고 State Root를 갱신함

- **Block Producer**: Witness를 만들어야 하므로 여전히 전체 State가 필요함. Block 생성에는 전체 State가 필요하고 검증에는 필요 없는 이 방식을 [Weak Statelessness](https://ethereum.org/roadmap/statelessness/)라 한다.

Witness는 Block과 함께 Network로 전파된다. 그래서 Block 하나에 주어진 12초 구간인 Slot 안에 전파될 만큼 작아야 한다.

**⇒ Stateless Client는 State를 저장하는 문제를 Witness를 전송하는 문제로 바꾼다. 이 방식이 동작하려면 Witness가 한 Slot 안에 전파될 만큼 작아야 한다.**

## Witness Size Problem

### Witness Size in MPT

MPT의 Witness에는 접근한 Key마다 Merkle Proof가 들어간다. Proof에는 Path 위 Branch Node마다 Sibling Hash가 최대 15개 들어가고, Account와 Storage가 늘어 Key가 많아지면 Path 위 Branch Node가 늘어 Proof도 길어진다.

아래 추정에 나오는 Gas는 Ethereum이 연산과 State 접근에 매기는 비용 단위다. Block 하나에 담을 수 있는 Gas에는 상한이 있으므로, 상한을 접근 하나의 Gas로 나누면 Block 하나가 할 수 있는 최대 접근 수가 나온다.

- **Account 하나**: [EIP-6800](https://eips.ethereum.org/EIPS/eip-6800)에 따르면 Account 접근 하나의 Witness는 평균 3 kB에 가깝고, 최악의 경우 그 세 배다.

- **Block 하나, EIP-6800의 추정 (Block Gas 15,000,000)**: 최악의 경우 접근 6,000번(접근당 2,500 Gas)에 접근당 평균 3 kB를 곱해 약 18 MB다. EIP-6800은 이 크기가 12초 Slot 안에 전파하기에 너무 크다고 적는다.

- **Block 하나, [The Verge](https://vitalik.eth.limo/general/2024/10/23/futures4.html)의 추정 (Block Gas 30,000,000, Code 포함)**: 최악의 경우 Account 읽기 하나에 2,400 Gas로 계산해 30,000,000 ÷ 2,400 × (5 × 480 + 24,000) = 330,000,000 Bytes, 약 330 MB다.

330 MB를 나누어 보면 두 가지 원인이 드러난다.

1. **Code 전체**: Account State에는 Contract Code(Smart Contract의 실행 Code) 전체의 Hash인 codeHash만 들어 있다. 그래서 Code의 일부만 실행해도, 그 Code가 codeHash와 일치하는지 확인하려면 최대 약 24,000 Bytes인 Code 전체를 받아야 한다. 접근 12,500번 × 24,000 Bytes = 300 MB로, 330 MB의 대부분이다.

2. **Sibling Hash**: Code를 뺀 부분은 접근 12,500번 × 2,400 Bytes = 30 MB다. 접근 하나의 Proof가 2,400 Bytes(5 × 480)인 것은 Level 하나가 Sibling 15개 × 32 Bytes = 480 Bytes이고, 접근이 많으면 Proof끼리 위쪽 Node가 겹쳐서 Level 8개(Key $2^{32}$개 기준)가 아니라 5개만 세기 때문이다.

**⇒ Witness를 줄이려면 두 가지가 필요하다. Level마다 보내는 Sibling을 줄이는 Tree 구조, 그리고 Code를 잘게 나눠 Tree에 넣어 실행한 부분만 증명할 수 있게 하는 구조다.**

<br />

## Solution 1: Binary Trie

Guillaume Ballet과 Vitalik Buterin이 2020년에 낸 [EIP-3102](https://eips.ethereum.org/EIPS/eip-3102)(Stagnant)는 MPT를 Width 2인 Binary Trie로 바꾸자고 제안했다. EIP-3102는 Binary Trie의 Proof가 Hexary(16진) Trie보다 약 4배 작다고 적는다. Account Trie와 Storage Trie를 하나로 합쳐, 여러 변경을 Proof 하나로 증명할 수 있게 하는 내용도 담았다.

Width가 $k$인 Hash 기반 Tree에서 Key 하나의 Path 크기는 $32 \times (k - 1) \times \log_k n$ Bytes다. Level 수는 $\log_k n$으로 줄지만, Level마다 Sibling이 $k - 1$개 필요하다. $n = 2^{32}$일 때 Width별 크기는 다음과 같다 (직접 계산한 값).

| Width $k$ | Level 수 | Level마다 Sibling | Path 크기 |
| --- | --- | --- | --- |
| 2 (Binary Trie) | 32 | 1개 | 1,024 Bytes |
| 4 | 16 | 3개 | 1,536 Bytes |
| 16 (MPT) | 8 | 15개 | 3,840 Bytes |
| 256 | 4 | 255개 | 32,640 Bytes |

Width를 넓히면 Level 수가 줄어드는 것보다 Sibling이 늘어나는 것이 더 커서 Path가 오히려 길어진다. Width 2일 때 가장 작다. Vitalik Buterin도 [Hash 기반 Tree 중에서는 Binary Tree가 이미 이론적 최적에 가깝다](https://hackmd.io/@vbuterin/BylgAm7ZF)고 적는다.

**한계**

- **더 줄일 수 없음**: Hash 기반 Tree로는 Key 하나의 Path를 $32 \times \log_2 n$ Bytes 아래로 줄일 수 없다. 같은 글은 Binary Tree의 Proof도 여전히 너무 크다고 본다.

- **Block 하나의 최악의 경우**: Code를 Chunk(일정한 길이로 자른 조각)로 나눠 Tree에 넣는다고 가정해도 30,000,000 ÷ 2,400 × 32 × (32 - 14 + 8) = 10,400,000 Bytes, 약 10.4 MB다 ([The Verge](https://vitalik.eth.limo/general/2024/10/23/futures4.html)). 32는 Tree Depth, 14는 접근 약 $2^{14}$번이 함께 쓰는 위쪽 Level, 8은 Chunk 안의 Value까지 내려가는 Proof의 길이다.

**⇒ Hash로 만든 Parent는 Child 전부가 있어야 확인할 수 있으므로, Hash 기반 Tree는 Level마다 Sibling을 보내야 한다. Width를 어떻게 정해도 이 한계를 넘을 수 없다. Proof를 더 줄이려면 Sibling 없이 Child 하나를 증명하는 방법이 필요하다.**

<br />

## Solution 2: Verkle Tree

**Verkle Tree**는 John Kuszmaul이 2018년 MIT PRIMES 연구에서 제안한 Tree다. 이름은 Very short Merkle Tree를 줄인 것이다 (Ver(y short Mer)kle). [논문](https://math.mit.edu/research/highschool/primes/materials/2018/Kuszmaul.pdf)의 아이디어는 한 문장으로 정리된다. Merkle Tree에서 Parent는 Child들의 Hash이고, Verkle Tree에서 Parent는 Child들의 Vector Commitment다.

- Vector Commitment 하나로 전체 Data를 Commit하면 Proof는 $O(1)$이지만, 논문이 쓴 방식에서는 만드는 시간이 $O(n^2)$이다. Verkle Tree는 Data를 Width $k$씩 나눠 Vector Commitment를 Tree로 쌓는다.

- Width $k$인 Verkle Tree의 Proof 크기는 $O(\log_k n)$이다. Width $k$인 Merkle Tree는 $O(k \log_k n)$이다.

- 대가로 Tree를 만드는 시간은 $O(kn)$이다. Width $k$인 Merkle Tree는 $O(n)$이다. Width가 클수록 Proof는 작아지고 계산은 늘어난다.

- 논문은 Catalano와 Fiore의 Vector Commitment로 구현했다. Ethereum에 제안된 설계는 다른 Commitment를 쓴다 (Step 1, Step 4).

### 핵심 아이디어

Hash 기반 Tree는 Parent를 Child들의 Hash로 만든다. Verifier가 Parent를 다시 계산하려면 Child 전부가 필요하므로 Proof에는 Level마다 Sibling이 들어간다. Verkle Tree는 이 부분을 다음처럼 바꾼다.

- **Parent: Child들의 Vector Commitment** → Child 하나가 Parent에 속함을 나머지 Child 없이 짧은 증명으로 보인다. 이 증명을 Opening Proof라 한다 (Step 1).

- **Width: 256** → Sibling을 보내지 않으므로 Width를 넓혀도 Level마다 들어가는 Proof Data가 늘지 않는다. 그만큼 Tree Depth가 줄어든다 (Step 2).

- **여러 Opening Proof: 하나로 묶음** → 묶은 증명을 Multiproof라 한다 (Step 3). 증명할 Value가 많아도 Opening Proof 부분의 크기는 일정하다.

Verifier는 Root의 Commitment(State Root) 하나만 신뢰하고, Path 위의 Commitment와 Multiproof로 Value를 확인한다.

| 항목 | MPT | Binary Trie | Verkle Tree (Ethereum 제안) |
| --- | --- | --- | --- |
| Parent의 값 | Child들의 Hash | Child들의 Hash | Child들에 대한 Vector Commitment |
| Width | 16 | 2 | 256 |
| Tree Depth | 약 $\log_{16} n$ | 약 $\log_2 n$ | 약 $\log_{256} n$ |
| Level마다 Proof에 들어가는 것 | Sibling Hash 최대 15개 (480 Bytes) | Sibling Hash 1개 (32 Bytes) | Commitment 1개 (32 Bytes) |
| Key 하나의 Path 크기 ($n = 2^{32}$) | 3,840 Bytes | 1,024 Bytes | 128 Bytes + Multiproof |
| 여러 Key를 증명할 때 | Key마다 Sibling Hash가 늘어남 | Key마다 Sibling Hash가 늘어남 | 모든 Opening Proof를 Multiproof 하나로 묶음 |

### Verkle Tree 단계별 구성

#### Step 1: Hash 대신 Vector Commitment

**→ 문제: Hash로 만든 Parent는 Child 전부가 있어야 확인할 수 있음**

- Parent가 hash(Child 0, ..., Child $k-1$)이면, Child 하나만으로는 Parent를 다시 계산할 수 없다.

- 그래서 Merkle Proof에는 Level마다 Sibling이 들어가고, Width를 넓힐수록 Sibling이 늘어난다.

**⇒ 해결: Parent를 Child들의 Vector Commitment로 만듦**

**Commitment Scheme**

Commitment Scheme은 값을 Commitment라는 짧은 값 하나로 고정해 두고, 나중에 그 값을 열어 확인시키는 암호학적 방법이다. 값을 봉투에 넣어 밀봉해 건네는 것과 같다.

- Binding: 한 번 만든 Commitment를 다른 값의 Commitment인 것처럼 열 수 없다.

- Hiding: Commitment만 보고는 값을 알 수 없다.

Vector Commitment는 순서가 있는 값의 목록인 Vector $(v_0, \ldots, v_{n-1})$ 전체를 Commitment 하나로 만든다. Vector의 값 하나하나를 Element(원소)라 한다.

- Opening: Commitment $C$의 Index $i$에 있는 Element가 $v_i$임을 밝히는 것

- Opening Proof: Opening이 맞음을 보이는 증명. 나머지 Element 없이 확인할 수 있다.

Verkle Tree가 쓰는 성질은 Binding과 짧은 Opening Proof다. Hiding은 쓰지 않는다. Ethereum의 State는 공개된 Data이고, [EIP-6800](https://eips.ethereum.org/EIPS/eip-6800)의 Commitment 식에도 값을 감추는 Random 항이 없다.

Hash도 Vector 전체를 고정한다는 점에서는 Vector Commitment로 볼 수 있다. 다만 Element 하나를 열려면 나머지 Element가 전부 필요하다. Verkle Tree가 Hash를 바꾸는 이유가 이 차이다.

**Polynomial Commitment**

Vector Commitment를 만드는 한 방법이 Polynomial Commitment다.

- Element $v_0, \ldots, v_{n-1}$에 대해 $f(i) = v_i$를 만족하는 Polynomial $f$를 만든다. 주어진 점들을 지나는 Polynomial을 구하는 Lagrange Interpolation(라그랑주 보간)을 쓴다.

- 그러면 "Index $i$의 Element가 $v_i$다"는 "$f(i) = v_i$다"와 같은 말이 된다. 그래서 Polynomial Commitment에서는 Opening Proof를 Evaluation Proof라고도 부른다.

- $f(z) = y$이면 $f(x) - y$는 $(x - z)$로 나누어떨어지고, 그 역도 성립한다. 따라서 Quotient(몫) $q(x) = \dfrac{f(x) - y}{x - z}$가 Polynomial로 존재한다는 것이 $f(z) = y$의 증거가 된다.

**Example**: Kate-Zaverucha-Goldberg(KZG) Commitment의 흐름을 작은 수로 줄인 것이다. 실제 암호 연산이 아니라 원리를 보이기 위한 계산이다.

- Vector는 (40, 2, 4)다.

- Prover는 Polynomial을 공개하지 않고 "Index 2의 Element는 4다"를 보이려 한다.

- Setup: Random 값 $r = 7$을 정한다.

1. **Commit**: $f(0) = 40$, $f(1) = 2$, $f(2) = 4$를 만족하는 $f$를 만들고, Commitment $C$를 계산한다.

    $$
    f(x) = 20x^2 - 58x + 40
    $$

    $$
    C = f(r) = 20 \times 7^2 - 58 \times 7 + 40 = 614
    $$

2. **Opening**: $f(2) = 4$에 대한 Quotient $q$를 만들고, Opening Proof $\pi$를 계산한다.

    $$
    q(x) = \frac{f(x) - 4}{x - 2} = \frac{20x^2 - 58x + 36}{x - 2} = 20x - 18
    $$

    $$
    \pi = q(r) = 20 \times 7 - 18 = 122
    $$

3. **Verify**: $C - f(2) = \pi \times (r - 2)$가 성립하는지 확인한다.

    $$
    614 - 4 = 610 = 122 \times 5
    $$

![polynomial-commitment.png](polynomial-commitment.png)

이 예시가 줄인 부분은 다음과 같다.

- 실제로는 $r$을 아무도 모른다. KZG는 Trusted Setup(신뢰 설정)이라는 준비 절차에서 $r$의 거듭제곱을 Elliptic Curve Point 안에 숨긴 공개 값을 만들고 $r$을 폐기한다.

- Commitment와 Opening Proof는 수가 아니라 Elliptic Curve Point다. 3단계의 등식은 Pairing이라는 Elliptic Curve 연산으로 $r$을 모르는 채 확인한다.

- $r$이 알려지면 안전하지 않다. 위 예시에서 $r = 7$을 아는 Prover는 거짓 값 14에 대해 $\pi = (614 - 14) / 5 = 120$을 만들어 등식을 통과시킬 수 있다 (직접 계산한 값).

Why 이 구조로 Step 1의 문제가 해결되는가?

- Verifier가 받은 것은 $C$, $f(2)$, $\pi$뿐이다. 나머지 Element인 40과 2는 받지 않는다.

- 거짓 값은 통과하지 못한다. $f(2) = 5$라고 주장하면 $f(x) - 5 = 20x^2 - 58x + 35$는 $x - 2$로 나누어떨어지지 않는다(나머지 -1). Polynomial $q$를 만들 수 없다.

- → Parent를 Child들의 Vector Commitment로 만들면, Child 하나가 Parent에 속함을 Sibling 없이 증명할 수 있다.

#### Step 2: Width를 256으로 넓히기

**→ 문제: Level마다 Proof가 들어가므로 Level 수를 줄여야 함**

- Step 1 이후에도 Path의 Level마다 Commitment 1개와 Opening Proof 1개가 필요하다.

- Hash 기반 Tree에서는 Width를 넓히면 Sibling이 늘어 Level 수를 줄일 수 없었다 (Solution 1).

**⇒ 해결: Width를 256으로 넓혀 Tree Depth를 줄임**

Sibling을 보내지 않으므로 Width를 넓혀도 Level마다 들어가는 Proof Data는 그대로다. Ethereum에 제안된 Verkle Tree의 Width는 256이고, 1,024가 낫다는 의견도 있었다. [Width의 상한은 Proof를 만드는 비용이 정한다](https://vitalik.eth.limo/general/2021/06/18/verkle.html). Proof를 만들 때 Node마다 Width에 비례하는 연산이 들기 때문이다.

**Structure**

원리를 보이기 위해 단순화한 구조다. EIP-6800의 실제 구조는 Step 5와 Step 6에서 설명한다.

- Inner Node: Child 256개의 Commitment를 Element로 가진다. Key를 1 Byte씩 따라 Child를 고른다. MPT의 Branch Node에 해당한다.

- Leaf Node: Key의 남은 부분과 Value를 가진다.

- Node의 값은 자기 Element들에 대한 Commitment다. Root의 Commitment가 State Root가 된다.

- Inner Node는 Key가 갈리는 지점까지만 생긴다.

**Example 1**: Account a1, a2, a3의 Key 첫 Byte가 각각 0x73, 0x7b, 0xd5다. Key의 첫 Byte가 Root에서의 Index가 된다 (0x73 = 115, 0x7b = 123, 0xd5 = 213).

![verkle-structure.png](verkle-structure.png)

```
Root (Inner Node)  C_r = Commit(C_0, ..., C_255)
├─ Index 0x73 → Leaf Node  C_115 = Commit(a1의 남은 Key와 Value)
├─ Index 0x7b → Leaf Node  C_123 = Commit(a2의 남은 Key와 Value)
└─ Index 0xd5 → Leaf Node  C_213 = Commit(a3의 남은 Key와 Value)
```

- 세 Key가 첫 Byte에서 모두 갈리므로 Inner Node는 Root 하나뿐이다.

- Width가 16인 MPT라면 a1과 a2가 첫 Nibble 7을 공유하므로 Branch Node가 하나 더 필요하다. Width가 넓으면 같은 Key 집합에서 Tree Depth가 줄어든다.

**Example 2**: Tree Depth가 더 깊어지는 경우다. Example 1과 다른 Account 집합이고, Key를 5 Bytes로 줄였다.

| Account | Key | Value |
| --- | --- | --- |
| a1 | 0x0a1b2c3d4e | Nonce 4, Balance 10 ETH |
| a2 | 0x0a62c01234 | Nonce 2, Balance 1 ETH |
| a3 | 0x0a62cd5f61 | Nonce 3, Balance 0.5 ETH |
| a4 | 0x0ac4e34567 | Nonce 5, Balance 4 ETH |
| a5 | 0x0ac4ea6765 | Nonce 7, Balance 3 ETH |

![verkle-structure-deep.png](verkle-structure-deep.png)

- 다섯 Key가 첫 Byte 0a를 공유하므로 Root의 Index 0x0a 아래에 Inner Node가 생긴다.

- 둘째 Byte는 1b, 62, c4로 갈린다. 1b는 a1뿐이므로 바로 Leaf Node가 된다.

- a2와 a3은 62를, a4와 a5는 c4를 공유한다. 각각 Inner Node가 하나 더 생기고 셋째 Byte에서 갈린다 (c0와 cd, e3와 ea).

- Root를 포함해 a1의 Path에는 Inner Node가 2개, 나머지 Path에는 3개 있다.

- Commitment의 아래 첨자는 Parent 안에서의 Index를 10진수로 적은 것이다 (0x0a = 10, 0x1b = 27, 0x62 = 98, 0xc4 = 196).

**검증 방법**

Verifier는 Root의 Commitment $C_r$을 알고 있다. Path의 각 Level에서 "Parent의 Commitment를 Index $z$에서 Opening하면 Element가 $y$다"를 확인한다.

- $z$: Key가 정하는 Child의 위치

- $y$: Child의 Commitment(실제로는 Commitment를 Scalar로 바꾼 값. Scalar는 Step 4, 변환은 Step 6에서 설명). 마지막 Level에서는 저장된 Value

[Proof는 Path 위의 Commitment와, 각 Commitment를 다음 Commitment에 잇는 짧은 증명이다](https://vitalik.eth.limo/general/2021/06/18/verkle.html). Sibling은 들어가지 않는다.

**Example**: Example 1에서 a1의 Value를 증명한다.

![verkle-proof.png](verkle-proof.png)

1. Prover가 a1의 Value, $C_{115}$, Opening Proof $\pi_r$, $\pi_{115}$를 보낸다.

2. Verifier가 $C_r$을 Index 115에서 Opening한 Element가 $C_{115}$인지 $\pi_r$로 확인한다.

3. $C_{115}$를 Opening한 Element가 a1의 Value인지 $\pi_{115}$로 확인한다. → Accept

- a2와 a3에 관한 Data는 보내지 않는다. Width가 16인 MPT라면 같은 증명에 a2의 Leaf Node와 a3의 Leaf Node의 Hash가 Sibling으로 들어간다.

- 단순화를 위해 남은 Key의 확인은 생략했다. $C_{115}$는 남은 Key와 Value의 Commitment이므로, 이 Leaf Node가 a1의 것임을 보이려면 남은 Key의 자리도 Opening해야 한다. 실제 구조에서는 Stem(Step 5)을 Opening해 이를 확인한다.

Why 이 구조로 Step 2의 문제가 해결되는가?

- Level마다 들어가는 Proof Data는 Commitment 1개와 Opening Proof 1개로, Width와 무관하다.

- Level 수는 $\log_{256} n = \frac{1}{2}\log_{16} n$으로 MPT의 절반이다. $n = 2^{32}$이면 MPT는 8 Level, Verkle Tree는 4 Level이다.

- Hash 기반 Tree가 Width 256에서 Level마다 Sibling 255개를 보내야 했던 것(Solution 1의 표에서 32,640 Bytes)과 달리, Verkle Tree는 Width를 넓힌 만큼 Proof가 줄어든다.

#### Step 3: Multiproof로 Opening Proof 묶기

**→ 문제: Opening Proof가 Opening 수만큼 늘어남**

- Block 하나는 수천 개의 Value에 접근한다. Opening은 대략 (Key 개수) × (Key당 Opening 수)만큼 생긴다.

- Example 1에서 a1, a2, a3을 모두 증명하면 Commitment 3개($C_{115}$, $C_{123}$, $C_{213}$)와 Opening Proof 6개(Root에서 3개, Leaf Node에서 3개)가 필요하다. 남은 Key의 확인까지 세면 Leaf Node에서 6개, 모두 9개다.

![verkle-multi-proof.png](verkle-multi-proof.png)

**⇒ 해결: 모든 Opening Proof를 Multiproof 하나로 묶음**

Multiproof는 여러 Opening Proof를 크기가 고정된 증명 하나로 묶는다. [Dankrad Feist의 방식](https://dankradfeist.de/ethereum/2021/06/18/pcs-multiproofs.html)은 다음과 같다. Opening이 $m$개이고, $i$번째 Opening은 "Commitment $C_i$의 Polynomial $f_i$가 $f_i(z_i) = y_i$를 만족한다"는 주장이다.

1. 모든 $C_i$, $z_i$, $y_i$를 Hash해 Random 값 $r$을 만든다.

2. Prover가 Quotient를 $r$의 거듭제곱으로 묶은 Polynomial $g$를 만들고, 그 Commitment $D$를 보낸다.

    $$
    g(X) = \sum_{i=0}^{m-1} r^i \, \frac{f_i(X) - y_i}{X - z_i}
    $$

    주장이 틀린 항은 나누어떨어지지 않는다. $r$은 주장들이 정해진 뒤에 Hash로 정해지므로, Prover가 틀린 항의 나머지끼리 서로 지워지도록 $r$을 고를 수 없다. 따라서 주장이 하나라도 틀리면 $g$는 무시할 수 있는 확률을 빼고 Polynomial이 되지 않는다.

3. $r$과 $D$를 Hash해 Random 값 $t$를 만든다. $t$는 Verifier가 $g$를 확인할 점이다.

4. Verifier가 $C_i$들로부터 $E = \sum_i \dfrac{r^i}{t - z_i} C_i$를, $y_i$들로부터 $y = \sum_i \dfrac{r^i \, y_i}{t - z_i}$를 직접 계산한다. $E$는 Polynomial $h(X) = \sum_i \dfrac{r^i}{t - z_i} f_i(X)$의 Commitment다.

5. Prover가 "$E - D$의 Polynomial을 $t$에서 Opening하면 $y$다"에 대한 Opening Proof 하나를 보낸다. $h(t) - g(t) = y$이기 때문이다.

자세한 과정은 [Multiproof 글]({% post_url Cryptography/2026-11-10-Multiproof %})에 있다.

Why 이 구조로 Step 3의 문제가 해결되는가?

- Proof는 $D$와 Opening Proof 하나다. Opening 개수 $m$과 무관하다.

- 4단계에서 Commitment끼리 더할 수 있어야 한다. KZG와 뒤의 Pedersen Commitment 모두 이 성질이 있다.

- → Key가 늘어나도 커지는 것은 Path 위의 Commitment와 Key, Value 등 Key마다 따로 필요한 Data뿐이고, Opening Proof 부분은 커지지 않는다.

#### Step 4: Pedersen Commitment와 IPA(Inner Product Argument)

**→ 문제: KZG를 쓰려면 Trusted Setup이 필요함**

Step 1~3은 원리를 KZG로 설명했다. KZG는 Trusted Setup이 필요하고, 검증에 Pairing을 쓴다. Ignacio Hagopian의 [Anatomy of a Verkle proof](https://ihagopian.com/posts/anatomy-of-a-verkle-proof)에 따르면 Verkle Tree는 원래 KZG를 쓸 계획이었다가 IPA로 바뀌었고, 그 이유는 역사적인 이유라고만 적혀 있다.

**⇒ 해결: Trusted Setup이 필요 없는 Pedersen Commitment와 IPA를 씀**

EIP-6800 본문은 Pedersen Commitment를 정하고, IPA는 EIP-6800 본문이 아니라 Verkle Proof 형식이 정한다.

Elliptic Curve Point는 서로 더할 수 있고 정수를 곱할 수 있다. 곱하는 정수를 Scalar라 한다. Pedersen Commitment는 미리 정해 둔 Elliptic Curve Point $G_0, \ldots, G_{255}$에 Element $v_i$를 각각 곱해 모두 더한 Point다. 이 Point들을 Basis Point(기저점)라 한다.

$$
C = v_0 G_0 + v_1 G_1 + \cdots + v_{255} G_{255}
$$

- Binding: [Basis Point끼리의 Discrete Log(이산 로그) 관계가 알려져 있지 않으면 성립한다](https://dankradfeist.de/ethereum/2021/07/27/inner-product-arguments.html).

- Trusted Setup: 필요 없다.

- Additively Homomorphic(덧셈 동형): 두 Commitment를 더하면 두 Vector의 합에 대한 Commitment가 된다. 이 성질이 Multiproof(Step 3)와 Tree의 갱신을 가능하게 한다.

- Opening Proof: IPA로 만든다. IPA는 Commitment에 담긴 Vector $a$와 공개된 Vector $b$의 Inner Product(내적)가 $y$임을 증명한다. Vector를 Polynomial의 값 $f(0), \ldots, f(255)$로 보면, $f(z)$는 $z$로부터 계산한 $b$와의 Inner Product로 쓸 수 있다.

EIP-6800 구조에서의 값은 다음과 같다.

- Elliptic Curve: Bandersnatch. Basis Point 256개 (EIP-6800)

- Commitment의 크기: 32 Bytes (Verkle Proof 형식에서 Elliptic Curve Point 하나의 크기)

- Element 하나에 넣을 수 있는 값: [252 Bits까지](https://blog.ethereum.org/2021/12/02/verkle-tree-structure). Scalar가 가질 수 있는 값의 집합인 Scalar Field의 크기가 253 Bits인 소수이기 때문이다.

| 항목 | KZG | Pedersen Commitment와 IPA |
| --- | --- | --- |
| Trusted Setup | 필요 | 불필요 |
| Opening Proof | Elliptic Curve Point 1개 | Elliptic Curve Point $2 \log_2 256 = 16$개와 Scalar 1개 |
| 검증 비용 | Pairing 등식 1개 (Pairing 2회) | Width에 비례하는 Elliptic Curve 연산 |
| 암호학적 가정 | Bilinear Group(Pairing을 계산할 수 있는 Group) 위의 t-Strong Diffie-Hellman(t-SDH) | Discrete Log |

Why 이 구조로 Step 4의 문제가 해결되는가?

- Pedersen Commitment는 Basis Point만 정하면 되므로 Trusted Setup이 없다.

- 대가로 IPA는 KZG보다 Opening Proof가 크고 검증 비용이 크다. 그러나 Multiproof가 모든 Opening을 하나로 묶으므로, IPA의 Opening Proof는 Witness 하나에 한 개만 들어가고 한 번만 검증한다.

- Multiproof의 크기: Anatomy of a Verkle proof가 적은 Verkle Proof 형식에서 $D$ 1개, IPA의 Point 16개, Scalar 1개다. 각 32 Bytes이므로 18 × 32 = 576 Bytes다 (형식에서 직접 계산한 값).

- 다른 자료는 Commitment 방식이 다를 수 있다. Vitalik Buterin의 2021년 글은 수치를 KZG(Commitment 48 Bytes) 기준으로 든다.

#### Step 5: Key를 Stem과 Suffix로 나누기

**→ 문제: 한 Account의 Value가 여러 Trie에 흩어지고, Code는 통째로 증명해야 함**

- MPT에서 Account State는 State Trie의 Leaf 하나이고, Storage는 Account마다 따로 있는 Storage Trie에 있다. Code는 Tree에 들어 있지 않고 Account State에 codeHash만 있다.

- 그래서 Account 하나의 Balance, Storage Slot(Contract Storage에서 Value 하나가 들어가는 자리) 몇 개, Code 일부를 읽는 Block은 Trie마다 따로 Path를 증명하고, Code는 전체를 보내야 한다.

**⇒ 해결: Account, Storage, Code를 Tree 하나에 넣고, Key를 Stem과 Suffix로 나눔**

EIP-6800은 Account, Storage, Code를 모두 하나의 Tree에 넣는다. Key와 Value는 모두 32 Bytes다. Key는 앞 31 Bytes인 Stem과 마지막 1 Byte인 Suffix로 나뉜다.

- Stem: Tree 안에서의 Path를 정한다. Address와 Tree Index를 이어 붙여 Pedersen Hash에 넣은 결과의 앞 31 Bytes다.

    - Tree Index: 한 Account의 Value를 256개씩 묶은 그룹의 번호

    - Pedersen Hash: Pedersen Commitment를 Hash Function처럼 쓴 것

- Suffix: 같은 Stem 아래에 있는 Value 256개 중 하나를 고른다.

Address 자체가 Path가 되는 것이 아니다. Stem이 Hash의 결과이므로 Key는 Tree 전체에 고르게 퍼진다. EIP-6800은 Tree 하나로 합치는 이유로 Key-Value 저장소로 다루기 쉬운 단순함, State가 Tree 전체에 고르게 퍼지는 균일함, Account 정보와 Code가 Storage와 같은 구조에 있어 확장하기 쉬운 점을 든다. Vitalik Buterin과 Dankrad Feist의 [Verkle Tree EIP 초안](https://notes.ethereum.org/@vbuterin/verkle_tree_eip)도 Account 정보, Code, Storage를 Key-Value Tree 하나에 넣는 방식을 정의한다.

Account의 첫 번째 Stem(Tree Index 0)에는 다음 Value가 들어간다.

| Suffix | 내용 | 상수 |
| --- | --- | --- |
| 0 | Version, Code Size, Nonce, Balance를 Value 하나에 묶음 | BASIC_DATA_LEAF_KEY = 0 |
| 1 | Code Hash | CODE_HASH_LEAF_KEY = 1 |
| 64~127 | Storage Slot 0~63 | HEADER_STORAGE_OFFSET = 64 |
| 128~255 | Code의 Chunk 0~127 | CODE_OFFSET = 128 |

- 나머지 Storage Slot과 Chunk는 다른 Tree Index의 Stem으로 간다.

- Code는 31 Bytes씩 나뉘고, 앞에 1 Byte를 붙인 32 Bytes Chunk로 Tree에 들어간다. 앞 1 Byte는 그 조각의 첫머리에서 PUSH 명령의 Data인 Byte의 수다. PUSH는 뒤따르는 Byte를 Data로 읽는 명령이다.

Why 이 구조로 Step 5의 문제가 해결되는가?

- 같은 Stem에 속한 Value는 Path를 공유한다. Nonce와 Balance, 앞쪽 Storage Slot, 앞쪽 Code Chunk를 함께 읽어도 Path 위의 Commitment는 Witness에 한 번만 들어간다.

- EIP-6800은 Storage Slot을 이렇게 묶는 것을, 함께 접근하는 Slot의 Witness를 효율적으로 만들기 위한 최적화라고 적는다.

- Code가 Chunk 단위로 Tree에 있으므로, MPT와 달리 실행한 Chunk만 증명할 수 있다.

#### Step 6: Extension Node와 C1, C2로 Value 담기

**→ 문제: Stem 아래 Value 256개를 Commitment 하나에 그대로 넣을 수 없음**

- Element 하나에는 252 Bits까지만 넣을 수 있는데 Value는 32 Bytes(256 Bits)다.

- Value가 0인 자리와 Value가 쓰인 적 없는 자리를 구분해야 한다.

- Inner Node는 Key가 갈리는 지점까지만 생기므로, Path 끝의 Node가 어느 Stem의 것인지 확인할 수 있어야 한다.

**⇒ 해결: Stem마다 Extension Node를 두고, Value를 16 Bytes씩 나눠 C1과 C2에 담음**

| Node | Commitment의 대상 | 역할 |
| --- | --- | --- |
| Inner Node | Child 256개의 Commitment | Stem을 1 Byte씩 따라 Child를 고른다 |
| Extension Node | (1, Stem, C1, C2). 나머지 Element 252개는 0 | Stem 하나와 그 아래 Value 256개를 대표한다 |

```
Inner Node → ... → Extension Node  Commit(1, Stem, C1, C2)
                     ├─ C1 = Commit(Suffix 0~127인 Value들의 조각 256개)
                     └─ C2 = Commit(Suffix 128~255인 Value들의 조각 256개)
```

- Extension Node의 첫 Element 1은 이 Node가 Extension Node임을 나타내는 표시다.

- Extension Node는 MPT의 Extension Node와 이름만 같다. MPT에서는 갈라지지 않는 Path 구간을 잇는 Node이고, 여기서는 Stem 하나에 속한 Value 256개를 담는 Node다. Stem도 남은 부분이 아니라 31 Bytes 전체를 가진다.

- Step 2의 단순화한 구조에서 Leaf Node라고 부른 것이 실제 구조에서는 Extension Node와 C1, C2에 해당한다.

- Parent가 Child의 Commitment를 담을 때는 Scalar로 변환해 Element에 넣는다. Commitment는 Elliptic Curve Point이고 Element는 Scalar이기 때문이다.

- Value 하나를 16 Bytes씩 두 조각으로 나눠 Element 두 개에 넣는다. Value 128개가 Element 256개를 채우므로 Commitment가 두 개 필요하다. C1은 Suffix 0~127, C2는 Suffix 128~255를 담는다.

- 아래쪽 조각에는 $2^{128}$을 더한다. Value가 없는 자리의 Element는 0이다.

Why 이 구조로 Step 6의 문제가 해결되는가?

- 조각 하나는 16 Bytes(128 Bits)이고 $2^{128}$을 더해도 129 Bits이므로 Element 하나에 들어간다.

- Value가 0이어도 아래쪽 조각은 $2^{128}$이 되어 0이 아니다. 그래서 [Value가 0인 경우와 Value가 쓰인 적 없는 경우를 구분할 수 있다](https://blog.ethereum.org/2021/12/02/verkle-tree-structure).

- Extension Node가 31 Bytes Stem 전체를 가지므로, Stem을 Opening하면 이 Node가 찾는 Key의 것인지 확인할 수 있다. 다른 Stem이면 Key가 없다는 증거가 된다.

### Final Verkle Tree (EIP-6800)

EIP-6800과 Verkle Proof 형식이 정하는 최종 구조는 다음과 같다.

1. **Key**: Stem(31 Bytes) + Suffix(1 Byte). Stem은 Address와 Tree Index의 Pedersen Hash 앞 31 Bytes

2. **Inner Node**: Child 256개의 Commitment(Scalar로 변환)를 Element로 가진 Pedersen Commitment. Stem을 1 Byte씩 따라 내려간다

3. **Extension Node**: Commit(1, Stem, C1, C2). C1과 C2는 Value 128개씩을 16 Bytes 조각 256개로 담는다

4. **Commitment**: Bandersnatch 위의 Pedersen Commitment, Basis Point 256개, 32 Bytes

5. **Witness 생성**: 전체 State를 가진 Block Producer가 Path 위의 Commitment와 Multiproof(IPA)를 만든다

6. **검증**: Stateless Client가 Witness로 Tree의 일부를 다시 만들고, Opening 목록을 직접 계산해 Multiproof를 이전 Block의 State Root에 대해 확인한다

7. **Update**: Value가 바뀌면 Path 위의 Commitment를 바뀐 분량만 더해 갱신한다

| 항목 | Step 2의 단순화한 구조 | EIP-6800과 Verkle Proof 형식 |
| --- | --- | --- |
| Commitment | KZG의 흐름으로 설명 | Pedersen Commitment와 IPA |
| Key | Account마다 Key 하나 | Stem(31 Bytes)과 Suffix(1 Byte) |
| Tree의 끝 | Leaf Node | Extension Node와 그 아래 C1, C2 |
| Opening Proof의 저장 | 다루지 않음 | Node에 저장하지 않고, Witness를 만들 때 Multiproof 하나로 계산 |

**⇒ Account의 Value를 Stem 단위로 모은 Tree 위에서, Path 위의 Commitment와 Multiproof 하나로 접근한 모든 Value를 증명한다.**

### Verkle Tree 동작 예시

#### Value 하나의 Proof

Example 1에서 Key의 첫 Byte를 Stem, 둘째 Byte를 Suffix로 보고 a1(Key 0x7340, Stem 0x73, Suffix 0x40)을 증명하면 Opening이 여섯 개 필요하다. [Anatomy of a Verkle proof](https://ihagopian.com/posts/anatomy-of-a-verkle-proof)가 나열하는 Opening 구성을 이 예시에 적용한 수다.

| 대상 | Opening하는 Index | 확인하는 것 | 개수 |
| --- | --- | --- | --- |
| Root의 Commitment | 0x73 | a1의 Extension Node의 Commitment | 1 |
| Extension Node의 Commitment | 0, 1, 2 | Extension Node 표시 1, Stem, C1 | 3 |
| C1 | 128, 129 | Value의 아래쪽 조각과 위쪽 조각 | 2 |

- Path에 Inner Node가 더 있으면 Inner Node마다 Opening이 1개씩 늘어난다.

- Suffix 0x40은 10진수로 64이므로 Value는 C1에 들어 있고, 조각의 Index는 2 × 64 = 128과 129다. Suffix가 128 이상이면 Extension Node의 Index 3에서 C2를 열고, C2 안의 Index는 2 × (Suffix - 128)과 그다음이다.

- 같은 Stem에서 C1에 속한 다른 Value를 함께 읽으면 앞의 Opening 4개는 공유한다.

- Opening 여섯 개의 Opening Proof는 모두 Multiproof 하나에 묶인다.

#### Key가 없음을 증명

- Path가 Inner Node의 빈 자리에서 끝나는 경우: 그 Index의 Element가 0임을 Opening한다.

- Path가 다른 Stem의 Extension Node에서 끝나는 경우: 그 Extension Node의 표시 1과 Stem을 Opening해 Stem이 다름을 보인다.

- Stem은 있지만 그 Suffix에 Value가 쓰인 적이 없는 경우: C1 또는 C2에서 그 Suffix의 조각 두 자리가 0임을 Opening한다.

#### Witness

Witness는 Verifier가 State 없이 Block을 검증하는 데 필요한 Data다. Anatomy of a Verkle proof가 적은 형식은 다음으로 이루어진다.

- 접근한 Stem과 Suffix, 그리고 Value

- Path 위의 Commitment. Root는 넣지 않고, Path가 겹치면 한 번만 넣는다.

- Stem마다 Path의 깊이와 Extension 상태(빈 자리, 다른 Stem, 있음)를 적은 1 Byte

- Key가 없음을 보일 때 쓰는 다른 Stem

- Multiproof 하나: $D$와 IPA의 Opening Proof (Elliptic Curve Point 16개, Scalar 1개)

Opening의 $z$와 $y$는 보내지 않는다. Verifier가 위 Data로 Tree의 일부를 다시 만들고, (Commitment, Index, Element)의 목록을 직접 계산한다. Root의 Commitment는 Verifier가 알고 있는 이전 Block의 State Root다.

#### Update

Value가 바뀔 때마다 Commitment를 처음부터 다시 계산하면 Node마다 Element 256개에 대한 연산이 필요하다. Verkle Tree는 Path 위의 Commitment만, 바뀐 분량만 더해서 갱신한다.

Index $i$의 Element가 $v_i$에서 $v_i'$로 바뀌면 새 Commitment는 다음과 같다. Pedersen Commitment가 Additively Homomorphic이기 때문에 가능하다.

$$
C' = C + (v_i' - v_i)\,G_i
$$

- 나머지 Element 255개는 읽을 필요가 없다.

- Elliptic Curve Point에 Scalar를 곱하는 연산(이하 Scalar Multiplication) 1회와 Point의 덧셈 1회로 끝난다.

- Opening Proof는 Node에 저장하지 않는다. 그래서 갱신할 때 고칠 Opening Proof가 없다.

Path를 따라 올라가는 순서는 다음과 같다.

1. Value가 바뀌면 Extension Node의 C1 또는 C2를 위 식으로 갱신한다. Value는 두 조각으로 나뉘어 있으므로 바뀐 조각마다 한 번씩 적용한다.

2. C1 또는 C2가 바뀌었으므로 Extension Node의 Commitment를 같은 식으로 갱신한다.

3. Parent인 Inner Node는 Child의 Commitment를 Scalar로 바꾼 값을 Element로 가진다. 새 Scalar와 이전 Scalar의 차이로 Parent를 같은 식으로 갱신한다.

4. Root까지 3을 반복한다.

**Example**: Example 2에서 a1이 a4에게 3 ETH를 보낸다. 수수료를 받는 Account의 변경은 생략한다. 그림은 단순화한 구조로 그린 것이다.

![verkle-update.png](verkle-update.png)

- 바뀌는 Value: a1(Nonce 4 → 5, Balance 10 → 7 ETH), a4(Balance 4 → 7 ETH)

- a1의 Path: a1의 Leaf Node → Index 0x0a 아래 Inner Node → Root

- a4의 Path: a4의 Leaf Node → Index 0xc4 아래 Inner Node → Index 0x0a 아래 Inner Node → Root

- 두 Path가 만나는 Index 0x0a 아래 Inner Node와 Root는 두 변경을 한 번에 반영한다.

- a2, a3, a5의 Leaf Node와 Index 0x62 아래 Inner Node는 그대로다.

- 실제 구조에서는 Leaf Node 자리에서 C1 또는 C2, Extension Node의 Commitment 순으로 갱신한 뒤 같은 Path를 올라간다.

#### MPT와 비교: Key 하나를 증명할 때 Verifier가 받는 것

| | MPT (a77d397 예시) | Verkle Tree (a1 예시, EIP-6800 형식) |
| --- | --- | --- |
| Path 위의 Node | Node 5개 전체 (E1, B1, E2, B2, L3) | Commitment 2개 (a1의 Extension Node, C1) |
| Sibling | B1에 든 L1, L4의 Hash, B2에 든 L2의 Hash | 없음 |
| 확인 방법 | Node마다 Hash를 계산해 Parent의 항목과 비교 | Opening 6개를 Multiproof 하나로 확인 |
| 그 밖의 Data | 없음 (남은 Key와 Value는 L3 안에 있음) | Stem, Value, 깊이와 Extension 상태 1 Byte |

### Proof Size 비교

Verkle Tree의 Proof가 MPT보다 작은 이유는 세 가지로 정리된다.

- Width가 256이므로 Tree Depth가 약 $\log_{16} n$에서 약 $\log_{256} n$으로 줄어든다 (Step 2).

- Level마다 Sibling Hash 최대 15개 대신 Commitment 1개만 들어간다 (Step 1).

- Opening Proof는 Key 수와 무관하게 Multiproof 하나다 (Step 3).

#### Path Size of One Key

Key 하나의 Path 크기다. Hash와 Commitment는 32 Bytes다. 세 식은 [The Verge](https://vitalik.eth.limo/general/2024/10/23/futures4.html)에 있다. 3,840 Bytes와 1,024 Bytes는 글에 적힌 값이고, 128 Bytes는 식에 대입한 값이다.

| Tree | 식 | $n = 2^{32}$일 때 |
| --- | --- | --- |
| MPT (Width 16) | $32 \times 15 \times \log_{16} n$ | 3,840 Bytes |
| Binary Trie (Width 2) | $32 \times \log_2 n$ | 1,024 Bytes |
| Verkle Tree (Width 256) | $32 \times \log_{256} n$ | 128 Bytes |

Verkle Tree에는 여기에 Multiproof 하나(576 Bytes)가 더해진다. Value $m$개를 증명할 때 Width가 $k$인 Hash 기반 Tree는 $O(mk \log_k n)$, Verkle Tree는 $O(m \log_k n)$에 크기가 고정된 Multiproof를 더한 값이다.

#### Proof Size of 1,000 Values

$n$ = 1,000,000, 증명할 Value $m$ = 1,000개일 때의 계산이다 (직접 계산한 값).

- Tree Depth: MPT는 $\log_{16} 10^6 \approx 4.98$이므로 5, Verkle Tree는 $\log_{256} 10^6 \approx 2.49$이므로 3으로 놓는다.

- MPT는 Level마다 Sibling Hash 15개(각 32 Bytes)를 넣는다. 실제 Proof처럼 Branch Node 전체(Hash 16개)를 세면 1,000 × 16 × 5 × 32 = 2,560,000 Bytes다.

- Verkle Tree(Step 2의 단순화한 구조)는 Level마다 Commitment 1개(32 Bytes)를 넣고, Multiproof 하나(576 Bytes)를 더한다.

- Verkle Tree(EIP-6800)는 Path 끝에 Extension Node의 Commitment와 C1 또는 C2가 함께 들어가므로 Key당 Commitment가 하나 더 많다. Root를 포함해 Inner Node가 3개면 Root를 뺀 Inner Node 2개, Extension Node, C1 또는 C2로 Key당 4개다.

| Tree | 계산 | 크기 |
| --- | --- | --- |
| MPT | 1,000 × 15 × 5 × 32 Bytes | 2,400,000 Bytes (2.4 MB) |
| Verkle Tree (단순화한 구조) | 1,000 × 3 × 32 Bytes + 576 Bytes | 96,576 Bytes (약 97 kB) |
| Verkle Tree (EIP-6800) | 1,000 × 4 × 32 Bytes + 576 Bytes | 128,576 Bytes (약 129 kB) |

- MPT와 비교하면 단순화한 구조는 약 25배, EIP-6800 구조는 약 19배 작다.

- 세 값 모두 근삿값이며 상한이 아니다.

    - Path가 가정보다 긴 Key가 있다. Key가 Random이면 MPT에서 앞 5 Nibble을 다른 Key와 공유할 확률은 $1 - e^{-10^6/16^5} \approx 0.61$이다. 따라서 약 61%의 Key가 Branch Node를 6개 이상 거친다. Verkle Tree에서 앞 3 Bytes를 공유할 확률은 $1 - e^{-10^6/256^3} \approx 0.058$이다. 따라서 약 6%의 Key가 Inner Node를 4개 이상 거친다.

    - 실제 크기를 줄이는 요인도 있다. 여러 Key가 위쪽 Level의 Node를 공유하면 같은 Hash나 Commitment는 한 번만 넣는다. MPT의 아래쪽 Branch Node는 항목 16개가 모두 차 있지 않다.

- Key와 Value 자체의 크기는 넣지 않았다.

#### 출처의 수치

- [Vitalik Buterin의 2021년 글](https://vitalik.eth.limo/general/2021/06/18/verkle.html): Width 256, Commitment와 Opening Proof가 48 Bytes인 KZG, Key가 가장 고르게 채워진 Tree를 가정한다. $2^{32}$ 크기의 Tree에서 Key 1개는 320 Bytes, Key 1,000개는 107,744 Bytes, Key 10,000개는 937,472 Bytes다.

    - Key 1,000개일 때 Key당 약 108 Bytes로, Key 1개일 때의 320 Bytes보다 훨씬 작다. 위쪽 Level의 Commitment를 여러 Key가 공유하고 Multiproof가 하나뿐이기 때문이다.

    - 같은 글의 보정: Key가 Random이면 Tree Depth가 약 0.6 늘어 Key당 약 30 Bytes가 더 든다. 32 Bytes Commitment를 쓰면 크기가 약 3분의 1 줄어든다.

    - 같은 글의 비교: Verkle Tree의 Proof는 Width 16인 MPT보다 20~30배 이상, 이상적인 Binary Merkle Tree보다 6~8배 작다.

- [ethereum.org](https://ethereum.org/roadmap/verkle-trees/)는 Leaf 1,000개 기준으로 MPT의 Witness 약 3.5 MB(7 Level), Verkle Tree의 Witness 약 150 kB(4 Level)를 들고 약 23배 작다고 적는다.

- Tree Depth: Random인 Key $n$개를 Width $d$인 Tree에 넣으면 [Path 위 Inner Node의 수는 평균 약 $\log_d n + 2/3$](https://dankradfeist.de/ethereum/2021/06/18/verkle-trie-for-eth1.html)이다. 수치 실험에서 얻은 값이다. $n = 2^{30}$, $d = 2^{10}$이면 약 3.67이다.

#### Witness Size of One Block

- Account 하나: 평균 약 200 Bytes ([EIP-6800](https://eips.ethereum.org/EIPS/eip-6800)). MPT는 평균 약 3 kB다.

- 평균적인 Block 하나: 수백 kB 규모의 Proof Data ([The Verge](https://vitalik.eth.limo/general/2024/10/23/futures4.html)). 계산값이 아니라 대략적인 규모다.

최악의 경우는 The Verge의 추정이다 (Block Gas 30,000,000, 접근당 2,400 Gas).

| Tree | Code 처리 | 최악의 Block Witness |
| --- | --- | --- |
| MPT | Code 전체 | 약 330 MB |
| Binary Trie | Chunk로 나눠 Tree에 넣음 | 약 10.4 MB |
| Verkle Tree | Chunk로 나눠 Tree에 넣음 | 약 1.3 MB |
| Verkle Tree, 공격자가 Path를 길게 만든 경우 | Chunk로 나눠 Tree에 넣음 | 약 2.6 MB |
| Verkle Tree, 인접한 위치에 연속 접근 (접근당 200 Gas) | Chunk로 나눠 Tree에 넣음 | 약 4.8 MB |

- Verkle Tree의 1.3 MB는 30,000,000 ÷ 2,400 × 32 × (32 - 14 + 8) ÷ 8 = 1,300,000 Bytes다. Binary Trie의 식을 Width 256에 맞춰 8로 나눈 것이다.

- 2.6 MB는 공격자가 앞부분이 같은 Address를 찾아내 Path를 약 두 배로 길게 만드는 경우다.

- 4.8 MB는 인접한 위치의 접근에 200 Gas만 매기는 규칙([EIP-4762](https://eips.ethereum.org/EIPS/eip-4762))에서 30,000,000 ÷ 200 × 32로 계산한 값이다.

### 검증과 생성 비용

- **검증**: [Value $m$개를 증명할 때 크기 $m$의 Multiexponentiation 1회와 약 $4m$회의 Field 연산이 대부분이다](https://vitalik.eth.limo/general/2021/06/18/verkle.html). Multiexponentiation은 여러 Elliptic Curve Point에 각각 Scalar를 곱해 모두 더하는 연산이고, Multiproof의 $E$를 계산하는 데 쓴다. 같은 글은 Value 수천 개의 검증이 100 ms 안에 끝난다고 적는다. 2021년의 추정이고 측정 환경은 적혀 있지 않다.

- **생성**: Multiproof는 [Polynomial 전체를 가지고 있을 때에만 효율적으로 만들 수 있다](https://dankradfeist.de/ethereum/2021/06/18/pcs-multiproofs.html). Path 위 각 Node의 Element 256개가 모두 필요하다는 뜻이다. 그래서 State 전체를 가진 Node가 Witness를 만든다.

- **갱신**: Value 하나를 고치면 Scalar Multiplication이 C1 또는 C2에서 최대 2회, Extension Node에서 1회, Path 위 Inner Node마다 1회 든다. Inner Node가 3개면 최대 6회다 (구조에서 직접 센 값). Vitalik Buterin의 2021년 글은 약 4회로 적는다. Leaf에서 Root까지 Commitment마다 1회씩, Root를 포함해 센 값이다. 같은 글에 따르면 Basis Point의 배수를 미리 계산해 두면 더 빨라진다.

### 한계

- **Witness 생성에 전체 State 필요**: Multiproof를 효율적으로 만들려면 Path 위 Node의 Element 전부가 필요하다. Stateless Client는 State 없이 검증할 수 있지만, Block Producer는 여전히 전체 State를 가져야 한다 (Weak Statelessness).

- **Width의 상한**: Proof를 만드는 비용이 Width에 비례하므로 Width를 무한정 넓힐 수 없다.

- **Opening Proof의 검증 비용**: IPA는 검증에 Width에 비례하는 연산이 든다. Multiproof로 Witness당 한 번만 검증해 이 비용을 줄인다.

- **Quantum Resistance 없음**: Pedersen Commitment의 Binding은 Discrete Log 문제의 어려움에 기댄다. [The Verge](https://vitalik.eth.limo/general/2024/10/23/futures4.html)는 Verkle Tree가 Quantum Computer에 취약하다고 적는다.

- **SNARK로 증명하기 어려움**: The Verge에 따르면 Verkle Proof의 검증에는 256 Bits 연산이 많이 들어가서, 검증 과정 자체를 SNARK(Succinct Non-interactive Argument of Knowledge)로 증명하기 어렵다.

- **State Root 갱신 속도**: The Verge는 State Root 갱신이 Verkle Tree보다 Hash 기반 Tree에서 빠르다고 적는다.

### 결론

Verkle Tree는 Merkle Tree의 Hash를 Vector Commitment로 바꾼 Tree다. Hash로 만든 Parent는 Child 전부가 있어야 확인할 수 있어서, Merkle Proof에는 Level마다 Sibling이 들어가고 Width를 넓힐수록 Proof가 커진다. Binary Trie가 Hash 기반 Tree로는 최선에 가깝지만 여전히 크다. Vector Commitment로 만든 Parent는 Child 하나를 Sibling 없이 짧은 Opening Proof로 증명할 수 있다. 그래서 Width를 256으로 넓혀 Tree Depth를 줄이고, 여러 Opening Proof를 Multiproof 하나로 묶는다. EIP-6800은 그 위에 Trusted Setup이 없는 Pedersen Commitment와 IPA를 쓰고, Stem과 Suffix로 함께 읽는 Value를 한 Path에 모으고, Code를 Chunk로 나눠 Tree에 넣었다.

그 결과 Key 하나의 Path는 3,840 Bytes(MPT)에서 128 Bytes로 줄고, Block 하나의 최악의 Witness는 Code를 Chunk로 나눈 효과를 포함해 약 330 MB에서 약 1.3~2.6 MB로 줄어든다. 대가는 Witness 생성 비용과, Quantum Computer에 취약한 Elliptic Curve 암호에 기대는 구조다. The Verge는 Verkle Tree와 함께 Hash 기반 Binary Tree의 Proof를 STARK(Scalable Transparent Argument of Knowledge)로 압축하는 방식을 비교한다. Verkle Tree가 채택되지 않고 Hash 기반 Binary Tree가 다시 제안된 경위는 [2편]({% post_url Paper-Review/2026-11-10-Verkle-Tree-Roadmap %})에서 다룬다.