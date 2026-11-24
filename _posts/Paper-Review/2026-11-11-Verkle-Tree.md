---
title: "Verkle Tree"
date: 2026-11-11
last_modified_at: 2026-11-24
categories: [Paper Review]
tags: [Blockchain, Ethereum, Verkle Tree, Merkle Patricia Trie, Stateless Client, Commitment]
math: true
media_subpath: /assets/img/posts/verkle-tree/
---

> 관련 글: [Multiproof]({% post_url Cryptography/2026-11-11-Multiproof %})

### Paper Information

| Item | Details |
| --- | --- |
| Title | Verkle Trees |
| Author | John Kuszmaul |
| Mentor | Alin Tomescu |
| Venue | Massachusetts Institute of Technology(이하 MIT) PRIMES(Program for Research in Mathematics, Engineering and Science for High School Students), 2018 |
| Link | [PDF](https://math.mit.edu/research/highschool/primes/materials/2018/Kuszmaul.pdf) |

- [https://vitalik.eth.limo/general/2021/06/18/verkle.html](https://vitalik.eth.limo/general/2021/06/18/verkle.html)

Verkle Tree는 Merkle Tree의 부모 Node를 Hash 대신 **Vector Commitment**로 만든 트리다. 이름은 Vector Commitment와 Merkle Tree를 합친 것이다. 어떤 Value가 트리에 들어 있음을 보이는 Proof가 Merkle Tree보다 훨씬 작다.

논문은 고등학생 대상 연구 프로그램에서 나온 10여 쪽 분량의 글이다. 저자는 복잡도 증명과 구현, 측정을 맡았고, 아이디어는 멘토가 제공했다고 적혀 있다. Ethereum은 2021년부터 Account 데이터 전체를 담는 State Tree를 Verkle Tree로 바꾸는 방안을 추진했다. 2026년 11월 현재 이 방안은 채택되지 않았고, 같은 자리에 Hash 기반 Binary Tree가 들어와 있다.

이 글은 세 가지를 다룬다. 논문의 Verkle Tree, 그 아이디어를 Ethereum의 State Tree에 적용한 제안, 그리고 그 제안이 Ethereum Roadmap에 들어갔다가 빠진 과정이다. Ethereum 제안은 논문과 Commitment 방식도, Proof를 다루는 방식도 다르다. 그래서 어느 쪽의 내용인지 구분해 적었다. 배경으로 Merkle Tree와, Ethereum이 지금 쓰는 State Tree인 Merkle Patricia Trie(이하 MPT)를 먼저 설명한다.

여러 증명을 하나로 묶는 Multiproof의 과정은 [별도 글](/posts/multiproof/)에서 다루므로 여기서는 결과만 쓴다. Ethereum 제안과 Roadmap의 상태는 2026년 11월 24일 기준이고, 글 뒤쪽의 "Ethereum Roadmap과 Verkle Tree"에 적었다. 이 글에서 Proof와 Witness의 크기는 B(Byte), kB, MB로 적는다.

---

## Merkle Tree

### Merkle Tree가 푸는 문제

데이터를 가진 쪽이, 데이터를 갖고 있지 않은 상대에게 "이 데이터는 전체 집합에 들어 있고 바뀌지 않았다"를 보여야 하는 상황이 있다. 상대가 전체 데이터를 저장하고 있다면 직접 비교하면 되지만, 그러려면 저장 부담이 크다.

- Proof를 만들어 보내는 쪽을 **Prover(증명자)**, 받아서 확인하는 쪽을 **Verifier(검증자)**라 한다.
- 목표는 Verifier가 **짧은 값 하나만 저장하고도** 데이터 하나의 포함 여부를 확인하는 것이다.

여기에 쓰는 도구가 Hash 함수다. Hash 함수는 임의 길이의 입력을 고정 길이의 값(Hash)으로 바꾼다. SHA-256이면 출력은 256 Bit, 즉 32 B다. 같은 Hash를 내는 서로 다른 두 입력을 찾는 것은 계산상 불가능하다. 이 성질을 Collision Resistance(충돌 저항성)라 한다.

### Merkle Tree의 구조

Merkle Tree는 Hash를 트리 모양으로 쌓아 데이터 전체를 Hash 하나로 요약한다.

1. 데이터 각각을 Hash함 → 맨 아래 Node
2. 두 자식 Node의 Hash를 이어 붙여 다시 Hash함 → 부모 Node
3. Node가 하나 남을 때까지 2를 반복함 → 맨 위 Node의 Hash가 **Merkle Root**

예시: 데이터 1부터 데이터 4까지 네 개. h1부터 h4는 각 데이터의 Hash이고, h5 = hash(h1, h2), h6 = hash(h3, h4), h7 = hash(h5, h6)임

![merkle-tree.png](merkle-tree.png)

- 데이터 하나가 바뀌면 그 Node의 Hash가 바뀌고, 위로 올라가며 Merkle Root까지 모두 바뀜
- 따라서 Merkle Root 하나가 데이터 전체를 대표함
- 자식이 둘인 Merkle Tree를 Binary Merkle Tree라 함

이 글에서 트리 용어는 다음과 같이 쓴다.

| 용어 | 의미 |
| --- | --- |
| Root | 맨 위 Node |
| Leaf | 자식이 없는 맨 아래 Node |
| Level | Root에서 같은 거리에 있는 Node들의 층 |
| 깊이 | Root에서 Leaf까지 거치는 Level의 수 |
| Width | 한 Node가 가질 수 있는 자식의 최대 개수 |
| Index | Node 안에서 항목의 위치를 가리키는 번호 |

### Merkle Proof

Merkle Proof는 데이터 하나가 Merkle Root가 가리키는 집합에 들어 있음을 보이는 데이터임. Verifier는 Merkle Root만 알고 있음

예시: 위 그림에서 데이터 1이 들어 있음을 증명함

1. Prover가 데이터 1과 함께 h2, h6를 보냄
2. Verifier가 데이터 1의 Hash를 계산함 → h1
3. hash(h1, h2)를 계산함 → h5
4. hash(h5, h6)를 계산함 → 결과가 알고 있는 Merkle Root와 같으면 Accept

- h2와 h6는 경로 위 Node의 **형제 Node**임. 부모의 Hash를 계산하려면 두 자식이 모두 필요하기 때문에 보냄
- Proof는 Level마다 형제 Hash 하나임. 데이터 2, 3, 4는 보내지 않음
- 위조할 수 없음. 다른 데이터로 같은 Merkle Root를 만들려면 Hash의 충돌을 찾아야 함

크기: Leaf가 $n$개면 깊이가 약 $\log_2 n$이므로 Hash도 약 $\log_2 n$개임. Leaf가 $2^{30}$개이고 Hash가 32 B면 30 × 32 = 960 B, 약 1 kB임

**⇒ Verifier는 Root 하나만 저장하고도, 데이터 하나와 짧은 Proof로 포함 여부를 확인할 수 있다. 대신 Level마다 형제가 필요하다는 점이 뒤에서 문제가 된다.**

<br />

## Merkle Patricia Trie

### Ethereum의 State와 State Tree

Ethereum의 **State(상태)**는 모든 Account(계정)의 Balance, Nonce, Contract Code, Storage를 합친 데이터다. State는 **State Tree**라는 트리에 담기고, 트리 전체는 **State Root**라는 값 하나로 요약되어 Block Header에 들어간다. State가 조금이라도 다르면 State Root가 달라진다.

- **Key**: Account의 Address를 Keccak-256 Hash 함수에 넣은 값(32 Byte)
- **Value**: Account의 Nonce, Balance, StorageRoot, CodeHash
    - StorageRoot: 그 Account의 Storage를 담은 별도 트리의 Root Hash
    - CodeHash: Contract Code의 Hash

즉 Storage는 Account마다 별도의 트리에 들어가고, Code는 트리 밖에 있으며 CodeHash만 트리에 남는다.

왜 Merkle Tree를 그대로 쓰지 않는가?

- Merkle Tree는 데이터 목록을 요약할 뿐, Key로 Value를 찾는 구조가 아님
- State는 Address로 Account를 찾아 읽고 고치는 Key와 Value의 집합임
- 그래서 Key로 Value를 찾는 트리인 Patricia Trie에 Merkle Tree의 Hash를 결합함. 이것이 현재 Ethereum의 State Tree인 MPT임. Patricia Trie는 바로 뒤에서 설명함

[ethereum.org](https://ethereum.org/developers/docs/data-structures-and-encoding/patricia-merkle-trie/)는 MPT를 결정적이고, 암호학적으로 검증할 수 있고, 삽입과 조회와 삭제가 $O(\log n)$인 구조로 소개한다.

### Patricia Trie

Trie는 Key를 한 자리씩 따라 내려가 Value를 찾는 트리다. Key 자체가 경로가 되고, 앞부분이 같은 Key들은 그 구간의 경로를 공유한다.

단순한 Trie는 Key 한 자리마다 Level이 하나씩 생긴다. Key가 64자리면 분기가 없는 구간에도 Node가 필요하고, 조회에 64단계가 걸린다. Patricia Trie는 **분기가 없는 구간을 Node 하나로 묶어** 경로를 줄인다.

예시: Key가 Ethereum, EthereumClassic, Ethena, Bitcoin, BitcoinCash, Bittensor, BitgetToken인 Patricia Trie

![patricia-trie.png](patricia-trie.png)

- Ethe는 세 Key가 공유하는 구간임. 한 글자씩 Node 네 개를 두는 대신 하나로 묶음
- Bit 아래에서는 tensor, coin, getToken으로 갈리고, coin 아래에 Cash가 이어짐
- 분기는 Key가 갈라지는 지점에서만 생김

MPT는 Patricia Trie의 각 Node를 Hash로 참조하는 구조다. Node를 직렬화해 Keccak-256에 넣은 결과가 그 Node의 참조가 된다. 직렬화한 결과가 32 Byte보다 짧은 Node만 예외로, Hash하지 않고 부모 안에 그대로 넣는다. 부모가 자식의 Hash를 담으므로 Merkle Tree와 마찬가지로 Root의 Hash 하나가 전체를 대표한다.

### MPT의 구조

Key는 4 Bit 단위인 Nibble로 나뉘어 경로가 된다. Nibble 하나는 16진수 한 자리다. Node는 세 종류다.

| Node | 형태 | 역할 |
| --- | --- | --- |
| Branch Node | 17개 항목 `[v0 ... v15, vt]` | Nibble 0~f에 해당하는 자식 16개와 Value 1개 |
| Extension Node | 2개 항목 `[encodedPath, key]` | 분기 없는 공통 경로와 다음 Node의 참조 |
| Leaf Node | 2개 항목 `[encodedPath, value]` | 남은 경로와 Value |

Patricia Trie와 대응시키면 Branch Node가 분기 지점이고, Extension Node가 분기 없이 묶인 구간이다. Leaf Node는 더 이상 분기가 없는 나머지 경로를 통째로 가진다. State Tree의 Key는 32 Byte이므로 경로 길이는 64 Nibble이다.

예시: 설명을 위해 Key를 2 Byte로 줄임. 실제 Key는 32 Byte임

| Account | Key | Value |
| --- | --- | --- |
| a1 | 0x7340 | Nonce 2, Balance 4 ETH |
| a2 | 0x7bc6 | Nonce 3, Balance 2 ETH |
| a3 | 0xd563 | Nonce 4, Balance 5 ETH |

![mpt-structure.png](mpt-structure.png)

```
Root (Branch Node)
├─ Index 7 → Branch Node
│             ├─ Index 3 → Leaf Node [남은 경로 40, a1의 Value]
│             └─ Index b → Leaf Node [남은 경로 c6, a2의 Value]
└─ Index d → Leaf Node [남은 경로 563, a3의 Value]
```

- a1과 a2는 첫 Nibble이 7로 같음 → Root의 Index 7 아래로 함께 내려감
- 그 아래 Branch Node에서 둘째 Nibble이 3과 b로 갈림 → 각각 Leaf Node가 됨
- a3은 첫 Nibble d를 혼자 씀 → Root의 Index d가 바로 Leaf Node를 가리킴
- 각 Node의 참조는 그 Node의 Hash임. Root의 Hash가 State Root가 됨

### MPT의 검증 방법

원리는 Merkle Proof와 같음

- Verifier는 State Root만 알고 있음
- Verifier는 Leaf에서 Root까지 Hash를 다시 계산하고, 결과가 State Root와 같은지 확인함
- Branch Node의 Hash를 계산하려면 그 Node의 모든 항목이 필요함

→ 경로 위의 Branch Node마다, 경로에 속하지 않는 나머지 항목인 **형제 Node의 Hash**가 Proof에 들어감

예시: a1을 검증하는 데 필요한 것

1. a1의 Leaf Node
2. 둘째 Branch Node에서의 형제인 a2 Leaf Node의 Hash → 둘째 Branch Node의 Hash를 계산
3. Root에서의 형제인 a3 Leaf Node의 Hash → Root의 Hash를 계산해 State Root와 비교

![mpt-proof.png](mpt-proof.png)

이 예시는 Branch Node가 거의 비어 있어 형제가 Level당 1개뿐임. 실제 State Tree의 상위 Level은 항목이 거의 차 있으므로 Level당 최대 15개의 형제 Hash가 들어감. Binary Merkle Tree에서는 Level당 1개였음

**⇒ Hash는 입력 전체를 알아야 다시 계산할 수 있다. 그래서 Verifier는 경로 위의 Node마다 형제를 전부 받아야 한다.**

<br />

## Problem

### 논문이 다루는 문제

논문의 상황은 원격 저장이다. 사용자가 파일 여러 개를 원격 Server에 맡기고, 나중에 돌려받은 파일이 맡긴 파일과 같은지 확인하려 한다. 사용자는 Merkle Root 같은 짧은 요약값만 저장하고, Server는 파일과 함께 Proof를 보낸다.

- Proof는 요청할 때마다 Network로 전송된다. 트리가 크면 Proof 전송이 대역폭의 대부분을 차지할 수 있다.
- Leaf가 $2^{30}$개이고 Hash가 256 Bit면 Merkle Proof는 약 1 kB다. 파일이 작으면 파일보다 Proof가 더 큰 부담이 된다.
- 논문은 같은 문제가 생기는 곳으로 Consensus Protocol, Public Key Directory, Bitcoin 같은 암호화폐, Secure File System을 든다.

**⇒ 논문의 목표는 Proof 크기를 줄여 대역폭을 아끼는 것이다. 같은 문제가 Ethereum에서는 Block과 함께 전파되는 Witness의 크기로 나타난다.**

### State가 커지면 생기는 문제

Block을 검증하는 Node는 Block의 Transaction을 모두 다시 실행하고, 계산한 State Root가 Block에 적힌 값과 같은지 비교한다. Transaction이 어떤 Account와 Storage를 읽을지 미리 알 수 없으므로 State 전체를 저장하고 있어야 한다. Vitalik Buterin의 [2024년 글](https://vitalik.eth.limo/general/2024/10/23/futures4.html)은 Node가 저장하는 State를 수백 GB로, 증가 속도를 연 약 30 GB로 적었다.

- State가 커질수록 일반 사용자가 Node를 직접 운영하기 어려워진다.
- 검증에 참여하는 Node가 줄어들면 소수의 Node에 권한이 집중될 수 있다.

**⇒ State 전체를 저장하지 않고도 Block을 검증할 수 있어야 한다.**

### Stateless Client와 Witness

Client는 Ethereum Protocol을 실행하는 Software다. **Stateless Client(상태 비저장 클라이언트)**는 State를 저장하지 않고, Block과 함께 받은 **Witness**로 Block을 검증한다. Witness는 두 가지를 묶은 것이다.

- Block의 Transaction을 실행하는 데 필요한 State 조각
- 그 조각이 State Root가 가리키는 State에 들어 있음을 보이는 Proof

Verkle Tree가 겨냥한 것은 Weak Statelessness(약한 상태 비저장)다. Block을 만드는 Node는 Witness를 만들어야 하므로 여전히 State 전체를 갖고, 나머지 Node만 State 없이 검증한다. 어떤 Node도 State 전체를 갖지 않는 Strong Statelessness(강한 상태 비저장)와 구분된다. 두 개념은 [ethereum.org](https://ethereum.org/roadmap/statelessness/)에 정리되어 있다. 같은 문서는 Strong Statelessness를 현재 Roadmap에 들어갈 것으로 보지 않는다고 적는다.

### Witness 크기의 조건

Witness는 Block과 함께 Network로 전파된다. Ethereum에서 Block 하나가 제안되는 시간 단위인 Slot은 12초다. Witness는 한 Slot 안에 받아서 처리할 수 있을 만큼 작아야 한다. 현재 구조인 MPT로는 이 조건을 맞추기 어렵다.

아래 표를 읽는 데 필요한 용어는 두 가지다. Ethereum Improvement Proposal(이하 EIP)은 Protocol 변경을 제안하는 문서이고 번호로 구분한다. Gas는 연산과 State 접근에 매기는 비용 단위이고, Gas Limit은 Block 하나가 쓸 수 있는 Gas의 상한이다.

| 항목 | MPT에서의 크기 | 가정 | 출처 |
| --- | --- | --- | --- |
| 접근 1회당 Witness | 평균 약 3 kB, 최악의 경우 그 약 3배 | | [EIP-6800](https://eips.ethereum.org/EIPS/eip-6800) |
| Block Witness | 약 18 MB | Gas Limit 15,000,000, 접근 1회당 2,500 Gas, Block당 접근 6,000회 | [EIP-6800](https://eips.ethereum.org/EIPS/eip-6800) |
| Block Witness, 최악 | 약 330 MB | Gas Limit 30,000,000, 접근 1회당 2,400 Gas, 접근마다 Contract Code 24,000 B 포함 | [2024년 글](https://vitalik.eth.limo/general/2024/10/23/futures4.html) |

Block Witness의 수치는 각 글이 쓰인 시점의 Gas Limit을 전제로 한다. Gas Limit이 바뀌면 비례해서 바뀐다. 2026년에 제안된 [EIP-8297](https://eips.ethereum.org/EIPS/eip-8297)은 Gas Limit 60,000,000과 Code 크기 한도 64 KiB를 기준으로 같은 최악의 경우를 약 1.8 GB로 계산한다.

반면 Verkle Tree에서는 Witness가 Account당 평균 약 200 B로 줄어든다고 EIP-6800은 적는다. 2024년 글은 Block 하나를 검증하는 데 드는 Proof 데이터를 접근한 데이터에 더해 약 100~2,000 kB로 적는다.

**⇒ Witness 크기는 State Tree가 만드는 Proof의 크기가 정한다. 따라서 MPT의 Proof가 왜 큰지를 봐야 한다.**

### MPT의 Proof Size

앞의 표에 나온 크기는 MPT의 Proof 크기에서 나옴

- $N$: 트리에 저장된 Key와 Value 쌍의 개수
- $k$: Width. MPT는 $k=16$
- 깊이: 약 $\log_k N$
- Level마다 형제 Hash가 최대 $k-1$개 필요하고, Hash 하나는 32 B임

$$
\text{Proof Size} \approx 32\,\text{B} \times (k-1) \times \log_k N
$$

$k=16$이면 $32 \times 15 \times \log_{16} N = 120 \times \log_2 N$ B이고, $N=2^{32}$일 때 3,840 B임. 식과 수치는 Vitalik Buterin의 [2024년 글](https://vitalik.eth.limo/general/2024/10/23/futures4.html)과 [EIP-7864](https://eips.ethereum.org/EIPS/eip-7864)에 있음

MPT에는 문제가 하나 더 있음. Code가 트리 밖에 있어, Code 접근을 증명하려면 Code 전체(최대 24,000 B)를 Witness에 넣어야 함

**⇒ MPT의 Proof가 큰 이유는 Level마다 형제 Hash가 최대 15개 들어가기 때문이다.** 풀어야 할 질문은 네 가지다.

1. **형제 Node 없이 검증할 수 있는가.** Hash는 입력 전체를 알아야 다시 계산할 수 있다.
2. **깊이를 줄일 수 있는가.** Proof는 깊이에 비례한다.
3. **Value가 많을 때도 Proof가 작게 유지되는가.** Block 하나는 수천 개의 Value에 접근한다.
4. **갱신 비용을 감당할 수 있는가.** State Tree는 Block마다 바뀐다.

<br />

## 기존 접근

논문이 Verkle Tree와 비교하는 구조는 세 가지다. $n$은 Leaf의 수이고 $k$는 Width다. 논문은 Width를 Branching Factor라 부른다.

Vector Commitment는 값의 목록 전체를 짧은 값 하나로 고정하고, 항목 하나를 나머지 항목 없이 짧은 증명으로 확인시키는 암호학적 방법이다. 자세한 설명은 다음 장에 있다.

| 접근 | 내용 | Proof 크기 | 한계 |
| --- | --- | --- | --- |
| Binary Merkle Tree | Width가 2임. Level마다 형제 Hash 1개 | $O(\log_2 n)$ | Leaf가 $2^{30}$개면 약 1 kB. 작은 데이터를 많이 담는 트리에서는 Proof 전송이 부담임 |
| Width $k$인 Merkle Tree | Width를 키워 깊이를 $\log_2 n$에서 $\log_k n$으로 줄임 | $O(k \log_k n)$ | Level마다 형제가 $k-1$개 필요해 Proof가 오히려 커짐 |
| Vector Commitment 하나 | 트리 없이 데이터 전체를 Commitment 하나로 만듦 | $O(1)$ | Commitment와 모든 항목의 증명을 만드는 데 $O(n^2)$, 항목 하나 갱신에 $O(n)$ |

Ethereum의 수치로 보면 Width를 키우는 방법이 왜 통하지 않는지 드러난다. $N=2^{32}$이고 Hash는 32 B다. 앞의 두 행은 Vitalik Buterin의 [2024년 글](https://vitalik.eth.limo/general/2024/10/23/futures4.html)의 값이고, 마지막 행은 같은 식에 대입한 계산이다.

| Hash 기반 트리 | 식 | Key 하나의 Proof |
| --- | --- | --- |
| Width 16 (MPT) | $32 \times 15 \times \log_{16} N$ | 3,840 B |
| Width 2 (Binary Merkle Tree) | $32 \times \log_2 N$ | 1,024 B |
| Width 256 | $32 \times 255 \times \log_{256} N$ | 32,640 B |

- Width를 키우면 깊이는 줄지만 형제 수가 더 빨리 늘어난다.
- Hash 기반 트리에서는 반대로 Width 2가 가장 유리하다.
- Width를 2로 바꾸고 Code를 트리에 넣어도 Block 하나의 Witness는 최악의 경우 약 10.4 MB다. 계산의 가정은 뒤의 "Block 하나의 Witness"에 있다.

**⇒ Hash를 쓰는 한 형제를 보내야 한다는 제약은 사라지지 않는다. Vector Commitment 하나로 전체를 묶으면 형제는 없어지지만 계산이 너무 무겁다. Verkle Tree는 둘을 결합한다. 트리 구조는 그대로 두고, 부모 Node를 만드는 방법만 Vector Commitment로 바꾼다.**

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

Ethereum은 논문의 구조를 그대로 쓰지 않았다. 이 장은 State Tree를 Verkle Tree로 바꾸는 제안인 [EIP-6800](https://eips.ethereum.org/EIPS/eip-6800)과, 그 설계를 설명한 글들을 기준으로 한다. EIP-6800은 2023년 3월에 작성됐고, 본문은 2026년 11월 24일에 확인했다. 이 제안이 채택되지 않은 경위는 뒤의 "Ethereum Roadmap과 Verkle Tree"에 있다.

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
- 두 방식 모두 타원곡선의 Discrete Log 문제가 어렵다는 데 기댐. 양자 컴퓨터는 이 문제를 풀 수 있으므로 어느 쪽도 Quantum Resistance(양자 내성)가 없음. 이 점이 뒤에서 Verkle Tree가 Roadmap에서 빠지는 첫째 이유가 됨
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

**⇒ 2024년 10월에는 Verkle Tree가 "지금 준비된 방식"이었고 Binary Hash Tree는 "증명이 아직 느린 방식"이었다. 이 저울이 그 뒤 어느 쪽으로 기울었는지가 다음 장의 내용이다.**

<br />

## Ethereum Roadmap과 Verkle Tree

이 장은 Verkle Tree가 Ethereum Roadmap에 어떻게 들어갔고, 왜 빠졌고, 그 자리에 무엇이 들어왔는지를 적는다. 2026년 11월 24일 기준이다. 현재 상태는 Ethereum Foundation의 Roadmap 초안인 [Strawmap](https://strawmap.org/), [ethereum.org의 Roadmap](https://ethereum.org/roadmap/), Upgrade 진행 상황을 추적하는 [Forkcast](https://forkcast.org/)에서 확인했다.

요약하면 다음과 같다.

- Verkle Tree는 Roadmap의 **The Verge**에서 State Tree를 바꾸는 수단이었다. 2024년 2월에는 Osaka Upgrade에 넣기로 정해졌다.
- 실제로는 어느 Upgrade에도 들어가지 못했다. Osaka는 Fusaka라는 이름으로 2025년 12월에 적용됐고 Verkle Tree는 없었다.
- 빠진 이유는 세 가지다. 타원곡선 기반이라 Quantum Resistance가 없고, Hash 기반 트리의 Proof를 증명하는 기술이 빨라져 Verkle Tree의 장점이 줄었고, Roadmap의 목표가 모든 실행을 SNARK로 증명하는 쪽으로 넓어지면서 증명하기 쉬운 트리가 필요해졌다.
- 그 자리에는 Hash 기반 Binary Tree가 들어왔다. 2025년 1월의 EIP-7864(Unified Binary Tree)를 거쳐 2026년 6월의 EIP-8297, 곧 Partitioned Binary Tree(이하 PBT)로 이어졌다.
- 빠진 것은 Verkle Tree라는 수단이다. Stateless Client라는 목표와 EIP-6800의 설계 일부는 그대로 이어진다.

### Roadmap 그림에서의 위치: The Verge

Vitalik Buterin의 Roadmap은 Ethereum의 개발 과제를 The Merge, The Surge, The Scourge, The Verge, The Purge, The Splurge로 나눈다. Verkle Tree는 이 가운데 **The Verge**에 속했다. 세 시점의 그림을 차례로 본다.

**2022년 11월의 Roadmap**

![roadmap2022.jpeg](roadmap2022.jpeg)

[https://x.com/VitalikButerin/status/1588669782471368704?a=AFtNzel-piZvsIUa0TE&s=20](https://x.com/VitalikButerin/status/1588669782471368704?a=AFtNzel-piZvsIUa0TE&s=20)

- 한국 시간으로 2022년 11월 5일에 올라온 그림이다. 이때 The Scourge가 새로 들어갔다.
- [당시 보도](https://coingape.com/breaking-vitalik-buterin-releases-updated-ethereum-roadmap)에 따르면 이 개정에서 The Verge는 Verkle Tree만이 아니라 검증(Verification) 전체를 가리키는 이름으로 넓어졌고, 최종 목표로 모든 실행을 SNARK로 검증하는 Ethereum이 적혔다.
- 즉 Verkle Tree는 이 시점부터 The Verge의 전부가 아니라 첫 단계였다.

**2023년판 Roadmap**

![roadmap2023.jpeg](roadmap2023.jpeg)

[https://vitalik.eth.limo/general/2024/10/23/futures4.html](https://vitalik.eth.limo/general/2024/10/23/futures4.html)

Vitalik Buterin의 2024년 글에 실린 2023년판 Roadmap이다. 아래는 그 가운데 The Verge 부분이다.

![ethereum-roadmap-verge.png](ethereum-roadmap-verge.png)

- The Verge는 원래 State 저장 구조를 Verkle Tree로 옮기는 일을 가리켰다. 2024년 글은 범위를 넓혀, Stateless 검증에 더해 Ethereum의 모든 실행을 SNARK로 검증하는 것까지 포함한다고 적는다.
- 목표는 두 가지다. Block을 완전히 검증하는 Client와 Staking Node가 수 GB를 넘는 저장 공간을 쓰지 않게 한다. 장기적으로는 스마트워치에서도 Chain 전체를 검증할 수 있게 한다.

그림의 항목은 다음과 같이 읽는다. 그림에는 EIP 번호가 없고, 항목과 EIP의 대응은 직접 연결한 것이다.

| 그림의 항목 | 뜻 | 해당 EIP |
| --- | --- | --- |
| Verkle tree spec + impl | Verkle Tree의 사양과 구현 | EIP-6800 |
| Code chunking + gas cost updates | Code를 Chunk로 나눠 트리에 넣고, Gas 비용을 Witness에 맞게 고침 | EIP-6800, EIP-4762 |
| Transition spec + impl | MPT의 State를 Verkle Tree로 옮기는 절차 | EIP-7612, EIP-7748 |
| Verkle trees | 위 세 가지가 모여 State Tree를 교체함 | |
| SNARK for Verkle proofs, Fully SNARKed Ethereum | Verkle Proof와 실행, 합의 과정을 SNARK로 증명함 | |
| Move to quantum-safe SNARKs (eg. STARKs) | 양자 컴퓨터에 안전한 증명 방식으로 옮김 | |

**⇒ 이 그림에서 Verkle Tree는 최종 목표가 아니라 중간 단계다. 그 뒤에 모든 실행을 SNARK로 증명하는 단계와 양자 내성 증명이 놓여 있다. 뒤의 두 단계가 요구하는 조건, 곧 SNARK로 증명하기 쉬울 것과 양자 컴퓨터에 안전할 것을 Verkle Tree 자신은 충족하지 못한다. Verkle Tree가 빠지는 이유가 이미 같은 그림 안에 들어 있었던 셈이다.**

**2026년 8월의 Roadmap**

![roadmap2026.jpeg](roadmap2026.jpeg)

![roadmap2026_2.jpeg](roadmap2026_2.jpeg)

[https://x.com/VitalikButerin/status/2086824673556467938?a=U1PdudCaHuTR7kbLD7Q&s=20](https://x.com/VitalikButerin/status/2086824673556467938?a=U1PdudCaHuTR7kbLD7Q&s=20)

- 2026년 8월 10일에 Vitalik Buterin이 2023년판 Roadmap을 Strawmap과 겹쳐 비교한 그림이다.
- [The Defiant](https://thedefiant.io/newsletter/defi-daily/ethereum-bets-its-technical-future-on-privacy-and-quantum-defense)와 [Unchained](https://unchainedcrypto.com/ethereums-priorities-have-changed-a-lot-since-2023-vitalik-says-comparing-new-roadmap/)의 보도에 따르면 이 비교에서 Verkle Tree는 더 나은 방식으로 대체된 항목으로 분류됐다. 먼저 Unified Binary Tree로, 다시 PBT로 바뀌었다. State Expiry도 새로운 State 유형으로 대체됐다.
- 같은 비교에서 양자 안전성은 우선순위가 올라갔고, Verifiable Delay Function과 Ethereum Virtual Machine(이하 EVM) 개선 항목 다수는 내려갔다. 강한 Privacy 같은 항목은 2023년판에 없던 것이다.

세 시점을 나란히 놓으면 다음과 같다.

| 시점 | State Tree를 바꾸는 수단 | The Verge의 범위 |
| --- | --- | --- |
| 2022년 11월 | Verkle Tree | 검증 전체로 확장. 최종 목표는 모든 실행을 SNARK로 검증 |
| 2023년판 | Verkle Tree | Verkle Tree, 그 뒤 SNARK로 증명하는 Ethereum, 그 뒤 양자 내성 SNARK |
| 2026년 8월 | Unified Binary Tree를 거쳐 PBT | 양자 안전성과 Privacy가 앞 순위로 올라옴 |

### Verkle Tree 도입을 구성한 EIP

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

### 도입 계획: 준비, Overlay Tree, 단계적 변환

가장 어려운 부분은 트리 구조가 아니라 전환이었다. 운영 중인 Chain을 멈추지 않고 State 전체를 다른 트리로 옮겨야 한다. 계획은 네 단계였다.

**1단계. 준비 (전환 Fork 이전)**

Stateless Client가 다룰 수 없는 동작을 먼저 없앤다.

- **EIP-2935**: BLOCKHASH 명령은 Client가 최근 Block 이력을 갖고 있다고 가정한다. Stateless Client는 이력을 갖고 있지 않다. 그래서 최근 8,191개 Block의 Hash를 System Contract의 Storage에 저장해, Witness에 담을 수 있게 한다. Pectra에서 적용됐다. EIP-7709는 BLOCKHASH가 실제로 이 Storage를 읽고 그 비용을 내도록 바꾼다.
- **EIP-6780**: SELFDESTRUCT 명령은 Account의 Code와 Storage를 모두 지운다. Verkle Tree에서는 한 Account의 데이터가 서로 연결이 드러나지 않는 여러 Key에 흩어져 있어 이 동작이 불가능하다. 그래서 Contract를 만든 Transaction 안에서 호출한 경우가 아니면 잔고만 보내도록 바꿨다.
- **EIP-4762**: Gas 비용을 Witness 크기에 맞춘다. EIP는 이 변경을 Verkle Tree Fork 직전의 Fork에 넣는 것을 목표로 적는다. 개발자가 새 비용 구조에 미리 적응하게 하려는 것이다.

EIP-4762의 비용은 접근 단위가 정한다. 접근은 (Address, Sub Key, Leaf Key)로 표현된다. Sub Key까지가 Stem 하나에, Leaf Key가 Suffix에 대응한다.

| 상수 | 값 (Gas) | 부과되는 때 |
| --- | --- | --- |
| WITNESS_BRANCH_COST | 1,900 | 한 Transaction 안에서 어떤 Stem에 처음 접근할 때 |
| WITNESS_CHUNK_COST | 200 | 어떤 Value에 처음 접근할 때 |
| SUBTREE_EDIT_COST | 3,000 | 어떤 Stem 아래에 처음 쓸 때 |
| CHUNK_EDIT_COST | 500 | 어떤 Value에 처음 쓸 때 |
| CHUNK_FILL_COST | 6,200 | 비어 있던 자리에 처음 값을 쓸 때 |

한 Transaction 안에서 같은 Stem에 속한 Value를 이어서 읽으면 두 번째부터는 200 Gas만 든다. 앞에서 본 "한 Stem에 묶이는 Value"의 배치가 Gas 비용으로 이어지는 지점이다.

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
- Block당 옮기는 수의 상한(CONVERSION_STRIDE)과 시작 시점은 EIP에 값이 정해져 있지 않다(TBD).

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

### Upgrade 일정에서의 위치: Osaka에 배정됐다가 빠짐

Roadmap 그림은 방향을 보여 줄 뿐이고, 실제 적용 여부는 Upgrade마다 어떤 EIP를 넣을지 정하는 개발자 회의에서 결정된다. 그 기록을 시간순으로 적는다.

| 시점 | 일어난 일 |
| --- | --- |
| 2024년 2월 1일 | All Core Developers Execution 회의에서 Verkle Tree를 Prague 다음 Upgrade인 Osaka에 넣기로 함. 반대는 없었음 ([Galaxy의 회의 정리](https://galaxy.com/insights/research/ethereum-all-core-developers-execution-call-180)) |
| 2024년 4월 11일 | 전용 시험 Network인 verkle-gen-devnet-6이 시작됨. ethPandaOps의 [문서](https://notes.ethereum.org/@ethpandaops/verkle-gen-devnet-6)는 EIP-2935, EIP-4762, EIP-6800을 활성 상태로, EIP-7545를 아직 활성화되지 않은 상태로 적음 |
| 2024년 10월 | Vitalik Buterin의 2024년 글이 Verkle Tree, STARK와 보수적 Hash, STARK와 공격적 Hash 세 가지를 나란히 놓고 곧 결정해야 한다고 적음 |
| 2025년 1월 20일 | Hash 기반 Binary Tree 제안인 [EIP-7864](https://eips.ethereum.org/EIPS/eip-7864)가 작성됨. 저자 열 명이 EIP-6800과 같음 |
| 2025년 5월 7일 | Pectra 적용. EIP-2935 포함 |
| 2025년 6월 30일 | go-ethereum의 Guillaume Ballet이 Ethereum Community Conference에서 [Verkle Tree에서 Binary Tree로의 전환](https://ethcc.io/archives/completing-the-circle-transitioning-from-verkle-to-binary-trees-in-ethereum)을 발표함 |
| 2025년 12월 3일 | Osaka가 Fusaka라는 이름으로 적용됨. 주요 변경은 PeerDAS(Peer-to-Peer Data Availability Sampling)와 Gas Limit 상향이고 Verkle Tree는 들어가지 않음 |
| 2026년 2월 18일 | Ethereum Foundation이 2026년 우선순위 글에서 State의 장기 방향을 [Binary Tree와 Statelessness](https://blog.ethereum.org/2026/02/18/protocol-priorities-update-2026)로 적음. 이 글에 Verkle Tree는 나오지 않음 |
| 2026년 3월 1일 | Vitalik Buterin이 State Tree를 Binary Tree로 바꾸고 장기적으로 EVM도 바꾸는 계획을 밝힘 ([The Block](https://www.theblock.co/post/391681/vitalik-buterin-lays-out-a-two-part-plan-to-overhaul-ethereums-execution-layer-from-the-ground-up)) |
| 2026년 3월 6일 | consensus-specs 저장소에서 EIP-6800 사양이 [제거](https://github.com/ethereum/consensus-specs/pull/4984)됨. 사유는 EIP가 Stagnant이고 가까운 Upgrade에 들어갈 가능성이 낮다는 것 |
| 2026년 6월 11일 | [EIP-8297](https://eips.ethereum.org/EIPS/eip-8297)(PBT)이 작성됨 |
| 2026년 7월 23일 | [EIP-8347](https://eips.ethereum.org/EIPS/eip-8347)(PBT로의 State 이전 절차)이 작성됨 |
| 2026년 8월 10일 | Vitalik Buterin이 2023년판 Roadmap과 Strawmap을 비교한 그림을 올림. Verkle Tree는 대체된 항목으로 분류됨 |
| 2026년 10월 22일~31일 | 시험 Network glamsterdam-pbt-1에서 MPT를 PBT로 옮기는 시험이 진행됨 |
| 2026년 11월 10일 | go-ethereum이 EIP-7864와 EIP-4762의 구현을 [제거](https://github.com/ethereum/go-ethereum/pull/36091)함. 하루 전에는 EIP-8297 구현이 들어감 |

- Verkle Tree를 폐기한다고 못 박은 단일 결정문은 확인하지 못했다. EIP-6800의 상태도 Withdrawn이 아니라 Stagnant다. 위 표의 변화가 쌓여 사실상 Roadmap에서 빠진 것이다.
- 표에서 보이는 순서는 이렇다. Osaka 배정(2024년 2월) → 대안 비교(2024년 10월) → Binary Tree 제안(2025년 1월) → Verkle Tree 없는 Fusaka(2025년 12월) → Foundation의 방향 명시와 사양 제거(2026년 2~3월) → 구현 교체(2026년 11월).

### Roadmap에서 빠진 이유

출처들이 드는 이유는 세 가지로 모인다. 각 이유가 어느 문서에 적혀 있는지 함께 적는다.

**이유 1. Quantum Resistance가 없어서 트리를 두 번 바꿔야 한다**

- Verkle Tree의 Commitment는 타원곡선의 Discrete Log 문제에 기댄다. 양자 컴퓨터는 이 문제를 풀 수 있다.
- 2024년 글은 MPT를 Verkle Tree로 바꾸면 나중에 트리를 다시 바꿔야 한다고 적는다.
- EIP-7864는 실용적인 양자 위협을 2030년대로 보는 전문가 추정과, 2030년까지 타원곡선 암호를 단계적으로 줄이라는 National Institute of Standards and Technology의 권고를 든다. Hash 함수에만 의존하는 설계가 더 오래 간다고 보고, 자신을 아마도 마지막 State Tree가 될 구조로 적는다.
- State Tree 교체는 앞의 "도입 계획"에서 본 대로 Chain 전체의 State를 옮기는 가장 무거운 종류의 변경이다. 이것을 Verkle Tree로 한 번, 양자 내성 구조로 또 한 번 하느니 한 번에 끝내자는 것이 핵심 논리다 (출처들의 서술을 묶은 해석).
- Strawmap은 장기 목표 다섯 가지 가운데 하나로 Hash 기반 방식을 통한 양자 내성을 든다. 타원곡선 기반인 Verkle Tree는 이 목표와 맞지 않는다.
- [The Block](https://www.theblock.co/post/391681/vitalik-buterin-lays-out-a-two-part-plan-to-overhaul-ethereums-execution-layer-from-the-ground-up)은 Verkle Tree가 2026년 Hard Fork의 유력 후보로 거론됐으나, 타원곡선 암호의 양자 취약성에 대한 우려로 2024년 중반부터 Binary Tree가 다시 주목받았다고 보도했다.

**이유 2. 증명 기술이 빨라져 Verkle Tree의 장점이 줄었다**

- Verkle Tree의 주된 장점은 작은 Proof를 빨리 만든다는 것이었다. Hash 기반 Binary Tree는 Proof 자체는 크지만, 그 Proof를 STARK로 증명해 대체하면 크기 문제가 사라진다. 걸림돌은 증명 속도였다.
- 2024년 10월 시점에는 보수적 Hash를 쓰는 STARK의 증명이 10초를 넘었다. 앞의 "대안" 표가 이 시점의 비교다.
- EIP-7864와 EIP-8297은 증명 시스템의 발전으로 이 격차가 좁혀져, Verkle Tree의 주된 장점에 맞먹을 수 있다고 적는다.
- 2024년 글은 Verkle Tree가 아닌 방식을 고르면 배포가 Hard Fork 한 번쯤 늦어진다는 점을 Verkle Tree 쪽의 근거로 들었다. 그런데 Verkle Tree 자신이 Osaka에 들어가지 못했으므로 이 근거도 약해졌다 (출처에 없는 해석).

**이유 3. 목표가 모든 실행을 SNARK로 증명하는 쪽으로 넓어졌다**

- The Verge의 최종 목표는 2022년부터 모든 실행을 SNARK로 검증하는 Ethereum이었다. 실행을 증명하려면 State를 읽고 쓰는 과정, 곧 State Tree의 연산도 증명 안에 들어가야 한다.
- Verkle Tree의 Proof 검증은 256 Bit 연산이 많아 SNARK로 증명하기 어렵다(2024년 글).
- EIP-7864와 EIP-8297이 첫머리에 드는 동기가 이것이다. 현재의 MPT는 Recursive Length Prefix 부호화, Keccak Hash, 트리 안에 트리가 들어 있는 구조, 트리 밖에 있는 Code 때문에 증명하기 어렵다. Binary Tree에 증명하기 쉬운 Hash 함수를 쓰면 이 문제가 줄어든다.
- The Block의 보도에 따르면 Vitalik Buterin은 State Tree와 EVM이 증명 병목의 80% 이상을 차지한다고 보았다. Binary Tree는 Merkle Proof를 지금의 4분의 1로 줄이고, Hash 함수를 바꾸면 증명 효율이 3배에서 100배 더 나아진다고 했다. 4분의 1은 앞의 "Key 하나의 Proof Size" 표에서 3,840 B가 1,024 B로 줄어드는 것과 같은 수치다.
- Strawmap의 또 다른 장기 목표는 Zero-Knowledge EVM과 실시간 증명으로 Layer 1의 처리량을 초당 1 Gigagas까지 올리는 것이다. State Tree는 이 목표에 맞춰 고르게 됐다.

**⇒ 세 이유는 하나로 이어진다. Verkle Tree는 "Stateless Client를 위한 작은 Witness"라는 문제 하나를 가장 잘 푸는 구조였다. Roadmap이 요구하는 조건이 "양자 컴퓨터에 안전할 것"과 "SNARK로 증명하기 쉬울 것"까지 늘어나자, 세 조건을 한 번의 전환으로 충족하는 Hash 기반 Binary Tree가 선택됐다.**

잃은 것도 있다.

- Binary Tree만으로는 Witness가 Verkle Tree보다 크다. 최악의 경우 약 10.4 MB와 약 1.3 MB다. 그래서 이 방향은 Proof를 증명 시스템으로 줄이는 것을 전제로 한다.
- Guillaume Ballet의 [발표 자료](https://notes.ethereum.org/@gballet/verkle-and-binary)는 Verkle Tree가 나은 점으로 저장할 내부 Node가 적다는 것과 Proof가 작다는 것을, Binary Tree가 나은 점으로 Hash 속도, Quantum Resistance, STARK와의 호환을 든다.
- 2024년 글이 적은 대로 공격적 Hash는 검증이 덜 됐다. Hash 함수가 아직 확정되지 않은 이유다.

### Verkle Tree 자리에 들어온 것: Binary Tree

Stateless Client라는 목표는 그대로이고, Proof를 줄이는 수단이 Vector Commitment에서 "Binary Tree와 그 Proof를 증명하는 SNARK"로 바뀌었다.

| EIP | 제목 | 내용 | 상태 |
| --- | --- | --- | --- |
| [EIP-7864](https://eips.ethereum.org/EIPS/eip-7864) | Ethereum state using a unified binary tree | 2025년 1월 제안. Account, Storage, Code를 Binary Tree 하나에 넣음 | Draft |
| [EIP-8297](https://eips.ethereum.org/EIPS/eip-8297) | Partitioned Binary Tree | 2026년 6월 제안. Key의 첫 Byte로 구역(Zone)을 나눔. 0x00은 Account Header, 0x01은 Code, 0xFF는 Storage | Draft |
| [EIP-8347](https://eips.ethereum.org/EIPS/eip-8347) | Offline State Migration to the PBT | 2026년 7월 제안. MPT에서 PBT로 State를 옮기는 절차 | Draft |

![eip-8297-tree.png](eip-8297-tree.png)

그림은 EIP-8297에 실린 트리 구조다.

- **이어받은 것**: Account, Storage, Code를 하나의 트리에 넣는 구조, Value를 256개 단위의 Stem으로 묶는 방식, Account Header에 기본 데이터와 Code Hash와 앞쪽 Storage Slot 64개를 함께 두는 배치(HEADER_STORAGE_OFFSET 64), Code를 31 Byte Chunk로 나누고 PUSH 데이터 Byte 수 1 Byte를 붙이는 방식은 EIP-6800에서 온 것이다. Verkle Tree를 위해 한 설계 가운데 Commitment 방식을 뺀 나머지가 대부분 남았다.
- **달라진 것**: 부모 Node는 두 자식의 Hash다. Key는 길이가 고정되어 있지 않고 첫 Byte가 구역을 나타낸다. Code는 Account가 아니라 Code Hash를 기준으로 저장해, 같은 Code를 가진 Contract들이 Chunk를 공유한다. 64번 이후의 Storage Slot은 Storage 구역으로 간다.
- **정해지지 않은 것**: Hash 함수다. 참조 구현은 BLAKE3를 쓰고 Keccak과 Poseidon2가 후보다. Poseidon2는 증명 안에서의 성능이 좋지만 안전성 분석이 진행 중이다.
- **Gas 비용**: EIP-8297은 Gas 비용 체계를 바꾸지 않는다고 적는다. Gas 비용 조정은 별도 EIP로 다룰 예정이라고 EIP-8347이 적는다.
- **EIP-7864와 EIP-8297의 관계**: go-ethereum은 2026년 11월 10일에 "EIP-8297의 PBT가 EIP-7864의 Unified Binary Tree를 대체한다"는 설명과 함께 EIP-7864와 EIP-4762의 구현을 [제거](https://github.com/ethereum/go-ethereum/pull/36091)했다. EIP 문서상으로는 둘 다 Draft다.
- **이름에 대한 주의**: PBT는 EIP-8297의 제목대로 Partitioned Binary Tree다. 2026년 8월의 일부 기사는 이를 Poseidon Binary Tree나 Polynomial Binary Tree로 풀어 적었는데, EIP의 이름과 다르다. Hash 함수는 Poseidon으로 확정되지 않았다.

전환 방식도 달라졌다.

| 항목 | Verkle Tree 계획 (EIP-7612, EIP-7748) | PBT 계획 (EIP-8347) |
| --- | --- | --- |
| 변환하는 곳 | 합의 과정 안. 모든 Node가 Block마다 일부씩 옮김 | 합의 과정 밖. Node가 직접 변환하거나 변환된 State의 사본(Snapshot)을 내려받아 검증함 |
| 기준 | MPT의 Key 순서대로 처음부터 끝까지 | 되돌릴 수 없게 확정(Finalize)된 Block 하나(Anchor Block)의 State 전체 |
| 최신 Block까지 따라잡기 | 변환과 실행이 함께 진행됨 | Block마다 바뀐 State를 기록한 Block-Level Access List(EIP-7928)를 재생함. 다시 실행하지 않음 |
| 교체 전 확인 | 없음. Fork와 함께 시작 | 검증자들이 Block마다 PBT의 Root를 서명해 공개하는 기간을 둠. 합의에는 쓰이지 않는 관찰용임 |
| State Root 교체 | Fork 시점에 바로 교체하고 이후 변환 | 변환을 마친 뒤 Hard Fork 한 번에 교체. 그 Fork가 Finalize될 때까지 두 트리를 유지 |

- EIP-8347이 합의 과정 밖의 변환을 고른 이유는 세 가지다. 합의 규칙의 변경이 작고, State Root 하나가 항상 State 전체를 대표하고, 실행을 증명하는 쪽이 변환 단계까지 증명할 필요가 없다.
- 대가도 적혀 있다. 전환 기간에 두 트리를 함께 보관해야 하고(Mainnet 규모에서 약 300 GB 추가), 변환의 정확성이 합의로 강제되지 않고, Block마다 두 트리를 갱신해야 한다.

**⇒ Verkle Tree 계획에서 가장 무거웠던 "합의 과정 안의 변환"을 PBT 계획은 합의 과정 밖으로 뺐다.**

- **시험**: 2026년 10월 22일부터 31일까지 ethPandaOps의 시험 Network(glamsterdam-pbt-1)에서 Node 9개가 Chain을 운영하면서 MPT에서 PBT로 State를 옮겼다. Execution Client는 geth, Nethermind, Besu, erigon 네 가지였고 PBT Fork는 10월 29일이었다. 기록된 범위에서 Node 사이의 State Root 불일치는 없었다([측정 데이터 저장소](https://github.com/CPerezz/pbt-devnet-1-data)).

### 현재 일정

[ethereum.org의 Roadmap](https://ethereum.org/roadmap/)은 Upgrade를 시간순으로 보여 준다. State Tree와 관련된 내용만 옮기면 다음과 같다.

| Upgrade | 시점 | 상태 | State Tree 관련 내용 |
| --- | --- | --- | --- |
| Pectra | 2025년 5월 7일 | 적용됨 | EIP-2935 포함 |
| Fusaka | 2025년 12월 3일 | 적용됨 | 주요 변경에 없음 |
| Glamsterdam | 2026년 4분기로 표기 | 개발 중 | 주요 변경에 없음. 함께 들어가는 Block-Level Access List(EIP-7928)는 EIP-8347이 사용함 |
| Hegotá | 2027년 | 개발 중 | 주요 변경에 없음. 범위는 확정되지 않음 |

같은 문서는 앞으로 올 기술 과제의 하나로 Statelessness를 든다. State Tree를 무엇으로 바꾸는지는 이 문서에 적혀 있지 않다.

- **Glamsterdam**: 대표 변경은 Enshrined Proposer-Builder Separation(EIP-7732)과 Block-Level Access List(EIP-7928)다. Sepolia Testnet에는 2026년 10월 6일에 적용됐고, Gas 비용 변경이 기존 Contract에 미치는 영향 때문에 Hoodi Testnet 적용이 보류됐다([보도](https://www.altcoinbuzz.io/ethereum-delays-glamsterdam-testnet-upgrade-over-smart-contract-risks)). [Forkcast](https://forkcast.org/)는 Mainnet 적용 시점을 미정(TBD)으로 표시하고 자체 추정치를 2027년 1월 27일로 둔다. 발표된 날짜가 아니라 추정이다. ethereum.org의 Roadmap은 2026년 4분기로 적는다.
- **Hegotá**: Forkcast는 목표를 2027년으로 적는다. 대표 변경은 Fork-Choice Enforced Inclusion List(EIP-7805)와 Frame Transaction(EIP-8141)으로 정해졌고 나머지 범위는 논의 중이다. EIP-7709는 포함을 검토하는 목록(Considered for Inclusion)에 있다.
- **State Tree 교체**: 어느 Upgrade에도 확정되어 있지 않다. Hegotá의 EIP 목록인 [EIP-8081](https://eips.ethereum.org/EIPS/eip-8081)에는 확정, 검토, 제안, 거절 어느 목록에도 EIP-8297, EIP-8347, EIP-7864, EIP-6800이 없다. EIP-8347은 교체가 일어날 Fork에 이름(PBT_ACTIVATION_FORK)만 붙이고 시점은 정하지 않는다. 시점은 나중에 별도 EIP가 정한다고 적는다.
- **예상**: EIP-8347의 저자인 Maria Silva와 Carlos Perez가 Ethereum Research에 쓴 2026년 9월 [의견 글](https://ethresear.ch/t/how-hegota-can-influence-the-state-roadmap/25895)은 PBT로의 이전을 Hegotá 다음 Fork인 I\*로 예상한다. 공식 일정이 아니라 저자들의 예상이다. Maria Silva가 관리하는 [계획 문서](https://github.com/misilva73/pbt-planning)는 사양, 시험, Client별 구현 상황을 모아 둔다.
- **Roadmap 초안**: Ethereum Foundation은 [Strawmap](https://strawmap.org)이라는 Roadmap 초안을 그림으로 관리한다. 2029년까지 약 6개월 간격으로 Fork 일곱 개를 그리고, Glamsterdam과 Hegotá 다음 Fork는 I\*, J\* 같은 임시 이름으로 적는다. 문서 스스로 공식 계획이나 예측이 아니라 조율을 위한 초안이라고 밝힌다. 그림 안에서 State Tree가 놓인 위치는 직접 확인하지 못했다.

주의할 점은 다음과 같다.

- 일부 기사는 PBT의 Mainnet 적용을 2026년 4분기로 적는다. 이는 Glamsterdam의 목표 시점과 섞인 것으로 보인다. Glamsterdam의 범위에 State Tree 교체는 없다.
- Verkle Tree가 Fusaka나 Hegotá에 들어간다고 적은 글도 있다. EIP-6800의 상태와 두 Upgrade의 EIP 목록과 맞지 않는다.
- [ethereum.org의 Verkle Tree 문서](https://ethereum.org/roadmap/verkle-trees/)는 2026년 6월에 마지막으로 고쳐졌는데도 Binary Tree를 언급하지 않고, Verkle Tree를 여전히 Stateless Client로 가는 단계로 설명한다. [Statelessness 문서](https://ethereum.org/roadmap/statelessness/)도 같다. 같은 Verkle Tree 문서는 MPT를 Leaf를 둘씩 묶어 Hash하는 트리로 설명하지만 MPT의 Width는 16이다. Witness 크기가 Leaf 개수와 무관하게 고정이라고도 적지만 고정인 것은 Multiproof의 크기다.

<br />

## 결론

Verkle Tree는 Merkle Tree의 Hash 자리에 Vector Commitment를 넣은 트리다. 논문은 이 구조가 Proof를 Width 2인 Merkle Tree보다 $\log_2 k$배 줄이고, 대신 구성과 갱신에 $k$배의 계산을 쓴다는 것을 보였다.

Ethereum은 이 아이디어를 Stateless Client에 필요한 Witness를 줄이는 데 적용했다. Witness 크기는 State Tree의 Proof 크기가 정한다. MPT는 Hash로 부모 Node를 만들기 때문에 Level마다 형제 Hash를 보내야 하고, Proof가 (Width - 1) × 깊이에 비례한다. Verkle Tree는 부모 Node를 Vector Commitment로 만들어 형제를 보내지 않는다. 형제가 없으므로 Width를 256으로 키워 깊이를 줄일 수 있고, 여러 Opening Proof는 Multiproof 하나로 묶인다. Opening Proof를 저장하지 않고 Commitment를 덧셈으로 갱신하므로 갱신 비용은 Width에 비례하지 않는다. 그 결과 Key 하나의 경로 크기는 3,840 B에서 128 B로, Value 1,000개의 Witness는 약 3.5 MB에서 약 150 kB로 줄어든다.

Roadmap에서 Verkle Tree는 The Verge의 첫 단계였고, 2024년 2월에는 Osaka Upgrade에 배정됐다. 그러나 2026년 11월 24일 기준 Ethereum은 Verkle Tree를 채택하지 않았다. 타원곡선에 기반한 Commitment에 Quantum Resistance가 없어 트리를 두 번 바꿔야 한다는 점, 증명 시스템의 발전으로 Hash 기반 트리의 약점이 줄었다는 점, 모든 실행을 SNARK로 증명하려면 증명하기 쉬운 트리가 필요하다는 점이 이유다. 그 자리에는 Hash 기반 Binary Tree(EIP-7864를 거쳐 EIP-8297)가 들어왔고, 적용할 Upgrade는 아직 정해지지 않았다. Stateless Client라는 목표와, 함께 읽는 Value를 같은 Stem에 묶는 설계는 이어진다.