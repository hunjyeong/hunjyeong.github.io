---
title: "Verkle Tree"
date: 2026-11-11
last_modified_at: 2026-11-23
categories: [Paper Review]
tags: [Blockchain, Ethereum, Verkle Tree, Merkle Patricia Trie, Stateless Client, Commitment]
math: true
media_subpath: /assets/img/posts/verkle-tree/
---

> 관련 글: [Multiproof]({% post_url Cryptography/2026-11-11-Multiproof %})

Verkle Tree는 Merkle Tree의 부모 Node를 Hash 대신 **Vector Commitment**로 만든 트리다. 이름은 Vector Commitment와 Merkle Tree를 합친 것이다. 어떤 Value가 트리에 들어 있음을 보이는 Proof가 Merkle Tree보다 훨씬 작다. 2018년 John Kuszmaul이 [논문](https://math.mit.edu/research/highschool/primes/materials/2018/Kuszmaul.pdf)에서 처음 제안했고, Ethereum은 2021년부터 Account 데이터 전체를 담는 State Tree를 Verkle Tree로 바꾸는 방안을 추진했다.

이 정의를 이해하려면 Merkle Tree가 무엇인지, Ethereum이 지금 쓰는 Merkle Patricia Trie(이하 MPT)가 무엇인지부터 알아야 한다. 그래서 다음 순서로 설명한다.

1. Merkle Tree는 무엇이고, Proof는 어떻게 만드는가
2. Ethereum은 State를 어떻게 저장하는가 (MPT)
3. MPT의 무엇이 문제여서 Ethereum은 State Tree를 바꾸려 했는가
4. Verkle Tree는 Proof를 어떻게 줄이는가
5. Ethereum 제안의 실제 구조는 어떤가, 원 논문과 무엇이 다른가

여러 증명을 하나로 묶는 Multiproof는 [별도 글](/posts/multiproof/)에서 다루므로 여기서는 결과만 쓴다.

2026년 11월 23일 기준 Ethereum Mainnet은 Verkle Tree를 채택하지 않았다. 논의의 중심은 Hash 기반 Binary Tree로 옮겨 갔다. 이유와 경과는 글 끝의 "한계와 현재 상태"에 적었다.

이 글에서 Proof와 Witness의 크기는 B(Byte), kB, MB로 적는다.

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

크기: Leaf가 $n$개면 깊이가 약 $\log_2 n$이므로 Hash도 약 $\log_2 n$개임. Leaf가 $2^{30}$개이고 Hash가 32 B면 약 1 kB임

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

## Ethereum이 State Tree를 바꾸려 한 이유

### State가 커지면 생기는 문제

Block을 검증하는 Node는 Block의 Transaction을 모두 다시 실행하고, 계산한 State Root가 Block에 적힌 값과 같은지 비교한다. Transaction이 어떤 Account와 Storage를 읽을지 미리 알 수 없으므로 State 전체를 저장하고 있어야 한다. Vitalik Buterin의 [2024년 글](https://vitalik.eth.limo/general/2024/10/23/futures4.html)은 Node가 저장하는 State를 수백 GB로, 증가 속도를 연 약 30 GB로 적었다.

- State가 커질수록 일반 사용자가 Node를 직접 운영하기 어려워진다.
- 검증에 참여하는 Node가 줄어들면 소수의 Node에 권한이 집중될 수 있다.

**⇒ State 전체를 저장하지 않고도 Block을 검증할 수 있어야 한다.**

### Stateless Client와 Witness

Client는 Ethereum Protocol을 실행하는 Software다. **Stateless Client(상태 비저장 클라이언트)**는 State를 저장하지 않고, Block과 함께 받은 **Witness**로 Block을 검증한다. Witness는 두 가지를 묶은 것이다.

- Block의 Transaction을 실행하는 데 필요한 State 조각
- 그 조각이 State Root가 가리키는 State에 들어 있음을 보이는 Proof

Verkle Tree가 겨냥한 것은 Weak Statelessness(약한 상태 비저장)다. Block을 만드는 Node는 Witness를 만들어야 하므로 여전히 State 전체를 갖고, 나머지 Node만 State 없이 검증한다. 어떤 Node도 State 전체를 갖지 않는 Strong Statelessness(강한 상태 비저장)와 구분된다. 두 개념은 [ethereum.org](https://ethereum.org/roadmap/statelessness/)에 정리되어 있다.

### Witness 크기의 조건

Witness는 Block과 함께 Network로 전파된다. Ethereum에서 Block 하나가 제안되는 시간 단위인 Slot은 12초다. Witness는 한 Slot 안에 받아서 처리할 수 있을 만큼 작아야 한다. 현재 구조인 MPT로는 이 조건을 맞추기 어렵다.

아래 표를 읽는 데 필요한 용어는 두 가지다. Ethereum Improvement Proposal(이하 EIP)은 Protocol 변경을 제안하는 문서이고 번호로 구분한다. Gas는 연산과 State 접근에 매기는 비용 단위이고, Gas Limit은 Block 하나가 쓸 수 있는 Gas의 상한이다.

| 항목 | MPT에서의 크기 | 가정 | 출처 |
| --- | --- | --- | --- |
| 접근 1회당 Witness | 평균 약 3 kB, 최악의 경우 그 약 3배 | | [EIP-6800](https://eips.ethereum.org/EIPS/eip-6800) |
| Block Witness | 약 18 MB | Gas Limit 15,000,000, 접근 1회당 2,500 Gas, Block당 접근 6,000회 | [EIP-6800](https://eips.ethereum.org/EIPS/eip-6800) |
| Block Witness, 최악 | 약 330 MB | Gas Limit 30,000,000, 접근 1회당 2,400 Gas, 접근마다 Contract Code 24,000 B 포함 | [2024년 글](https://vitalik.eth.limo/general/2024/10/23/futures4.html) |

Block Witness의 수치는 각 글이 쓰인 시점의 Gas Limit을 전제로 한다. Gas Limit이 바뀌면 비례해서 바뀐다. 2026년에 제안된 [EIP-8297](https://eips.ethereum.org/EIPS/eip-8297)은 Gas Limit 60,000,000과 Code 크기 한도 64 KiB를 기준으로 같은 최악의 경우를 약 1.8 GB로 계산한다.

반면 Verkle Tree에서는 Witness가 Account당 평균 약 200 B로 줄어든다고 EIP-6800은 적는다. 2024년 글은 Block 하나를 검증하는 데 드는 Proof 데이터를 수백 kB로 적는다.

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

왜 Width를 바꾸는 것만으로는 해결되지 않는가?

- Width를 키우면 깊이는 줄지만 형제 수가 더 빨리 늚. $k=256$이면 Level마다 형제 Hash가 최대 255개이고, $N=2^{32}$일 때 32 × 255 × 4 = 32,640 B (위 식에 대입한 계산)
- Hash 기반 트리에서는 반대로 Width 2가 가장 유리함. $k=2$인 Binary Merkle Tree는 $32 \times \log_2 N$ B이고, $N=2^{32}$일 때 1,024 B
- Width 2로도 Block 하나의 Witness는 최악의 경우 약 10.4 MB임 (뒤의 "Verkle Tree의 Proof Size" 참고)

MPT에는 문제가 하나 더 있음. Code가 트리 밖에 있어, Code 접근을 증명하려면 Code 전체(최대 24,000 B)를 Witness에 넣어야 함

**⇒ Hash를 쓰는 한 형제를 보내야 한다는 제약은 사라지지 않는다. Proof를 줄이려면 트리 구조, 그중에서도 부모 Node를 만드는 방법 자체를 바꿔야 한다.**

<br />

## Verkle Tree

### 핵심 아이디어: Hash 대신 Vector Commitment

**문제 상황: Hash로 만든 부모는 자식 전부를 알아야 검증할 수 있음. 그래서 Level마다 형제 Node가 Proof에 들어감**

**해결 방법: 부모 Node를 자식들의 Vector Commitment로 만듦. 형제 없이, 경로 위의 Commitment와 짧은 증명만으로 검증함**

효과는 두 가지임

1. **형제 Node가 필요 없음**: 자식 하나가 부모에 속함을 보이는 데 나머지 자식들이 필요하지 않음
2. **깊이를 줄일 수 있음**: 형제를 보내지 않으므로 Width를 키워도 Level당 Proof 데이터가 늘지 않음. Width를 키우면 깊이만 줄어듦

Ethereum에 제안된 Verkle Tree의 Width는 256임. 1,024로 올리자는 의견도 있었음([Vitalik Buterin의 2021년 글](https://vitalik.eth.limo/general/2021/06/18/verkle.html)). Width의 상한은 Proof를 만드는 비용이 정함. Node마다 Width에 비례하는 연산이 들기 때문임

| 항목 | MPT | Verkle Tree (Ethereum 제안) |
| --- | --- | --- |
| 부모 Node의 값 | 자식들의 Hash | 자식들에 대한 Vector Commitment |
| Width | 16 | 256 |
| 깊이 | 약 $\log_{16} N$ | 약 $\log_{256} N$ |
| Level당 Proof 데이터 | 형제 Hash 최대 15개 (480 B) | Commitment 1개 (32 B) |
| 여러 Key를 검증할 때 | Key마다 형제 Hash 추가 | 모든 증명을 Multiproof 1개로 묶음 |

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

2. **Evaluation**: $x=2$일 때 $f(2)=4$와 Evaluation Proof $\pi=122$ 생성

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

- Verifier가 받은 것은 $C$, $y$, $\pi$뿐임. 나머지 항목 40과 2는 받지 않음 → 형제가 필요 없는 이유
- 거짓 값은 통과하지 못함. $y=5$를 주장하면 $f(x)-5=20x^2-58x+35$는 $x-2$로 나누어떨어지지 않음(나머지 -1). 다항식 $q$를 만들 수 없음

주의: 이 예시가 줄인 부분

- 실제로는 $r$을 아무도 모름. KZG는 Trusted Setup(신뢰 설정)이라는 준비 절차에서 $r$의 거듭제곱을 타원곡선 점 안에 숨긴 공개 매개변수를 만들고 $r$을 폐기함
- Commitment와 Opening Proof는 수가 아니라 타원곡선 점임. 3단계의 등식은 Pairing이라는 타원곡선 연산으로 $r$을 모르는 채 확인함
- $r$이 알려지면 안전하지 않음. 위 예시에서 $r=7$을 아는 Prover는 거짓 값 $y=14$에 대해 $\pi=(614-14)/5=120$을 만들어 등식을 통과시킬 수 있음 (출처에 없는 계산)

### Ethereum 제안의 Commitment: Pedersen Commitment와 IPA

위 예시는 KZG의 흐름임. Ethereum 제안([EIP-6800](https://eips.ethereum.org/EIPS/eip-6800))은 다른 방식인 **Pedersen Commitment**와 **Inner Product Argument(내적 증명, 이하 IPA)**를 씀

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
- Commitment 크기: 32 B (Anatomy of a Verkle proof가 적은 Witness 형식)
- 한 항목에 넣을 수 있는 값: 252 Bit까지. Scalar가 가질 수 있는 값의 집합인 Scalar Field의 크기가 253 Bit 소수이기 때문임 (Ethereum Foundation Blog)

| 항목 | KZG | Pedersen Commitment와 IPA |
| --- | --- | --- |
| Trusted Setup | 필요 | 불필요 |
| Opening Proof | 타원곡선 점 1개 | 타원곡선 점 $2\log_2 256 = 16$개와 Scalar 1개 |
| 검증 비용 | Pairing 1회 | Width에 비례하는 타원곡선 연산 |
| 암호학적 가정 | Bilinear Group | Discrete Log |

- IPA는 KZG보다 Opening Proof가 크고 검증 비용이 큼. 대신 Multiproof로 모든 Opening을 하나로 묶으므로, IPA의 Opening Proof는 Witness 하나에 한 개만 들어가고 한 번만 검증함
- 비교는 Dankrad Feist의 [Inner Product Arguments](https://dankradfeist.de/ethereum/2021/07/27/inner-product-arguments.html)에 있음. Scalar 1개는 Ignacio Hagopian의 [Anatomy of a Verkle proof](https://ihagopian.com/posts/anatomy-of-a-verkle-proof)가 적은 Witness 형식에 있음
- 주의: 다른 자료는 Commitment 방식이 다를 수 있음. Dankrad Feist의 [Verkle trie for Eth1 state](https://dankradfeist.de/ethereum/2021/06/18/verkle-trie-for-eth1.html)는 KZG 기준으로 쓰였고, Vitalik Buterin의 2021년 글은 수치를 KZG(Commitment 48 B) 기준으로 듦

### Verkle Tree의 구조

원리를 보이기 위해 단순화한 구조임. 실제 제안의 구조는 뒤의 "Ethereum 제안(EIP-6800)의 실제 구조"에 있음

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

### 여러 Value의 검증과 Multiproof

**문제 상황: 증명할 Value가 많아지면 Opening Proof도 그만큼 늘어남**

- Block 하나는 수천 개의 Value에 접근함. Opening이 대략 (Key 개수) × (Key당 Opening 수)만큼 생김
- 예시 1에서 a1, a2, a3를 모두 검증하면 Commitment 3개($C_{115}$, $C_{123}$, $C_{213}$)와 Opening Proof 6개(Root에서 3개, Leaf Node에서 3개)가 필요함

![verkle-multi-proof.png](verkle-multi-proof.png)

**해결 방법: Multiproof로 모든 Opening Proof를 크기가 고정된 증명 하나로 묶음**

- 원리: Opening마다 "주장이 맞을 때만 다항식이 되는 식" $\dfrac{f_i(X)-y_i}{X-z_i}$를 만들고, 무작위 계수를 곱해 모두 더한 다항식 하나를 만듦. 이 다항식 하나에 대한 증명만 확인하면 모든 Opening이 한 번에 검증됨
- 자세한 과정은 [Multiproof 글](/posts/multiproof/)과 Dankrad Feist의 [PCS multiproofs using random evaluation](https://dankradfeist.de/ethereum/2021/06/18/pcs-multiproofs.html)에 있음
- 크기는 Opening 개수와 무관하게 고정임. Vitalik Buterin의 2021년 글은 방식에 따라 128~1,000 B로 적음
- KZG에서 Dankrad Feist의 방식은 타원곡선 점 2개이고 96 B임. Vitalik Buterin의 2021년 글의 표는 Commitment 1개, 수 1개, 증명 2개인 176 B로 계산되어 있음
- EIP-6800에 맞춰 제안된 Witness 형식에서는 576 B임. Commitment 1개와 IPA의 Opening Proof(타원곡선 점 16개, Scalar 1개)이고 원소마다 32 B임 (Anatomy of a Verkle proof의 형식에서 계산한 값)

Multiproof를 적용한 Witness의 구성

- 접근한 Key와 Value
- 각 경로 위의 Commitment. Root는 제외하고, 경로가 겹치면 한 번만 넣음
- Multiproof 하나

$z$와 $y$는 보내지 않음. Verifier가 Key, Value, 경로의 Commitment로부터 직접 계산함

**⇒ Key가 늘어날 때 Proof에서 커지는 부분은 경로 위의 Commitment뿐이다.**

비용은 다음과 같음 (Vitalik Buterin의 2021년 글)

- **검증**: Value $m$개를 증명할 때 크기 $m$의 Multiexponentiation 1회와 약 $4m$회의 Field 연산이 대부분임. Multiexponentiation은 여러 타원곡선 점에 각각 Scalar를 곱해 모두 더하는 연산이고, Field 연산은 큰 소수를 법으로 하는 덧셈과 곱셈임. 글은 Value 수천 개의 검증이 100 ms 안에 끝난다고 적음. 2021년의 추정이고 측정 환경은 적혀 있지 않음
- **생성**: 경로 위 각 Node의 항목 256개가 모두 필요함. 그래서 State 전체를 가진 Block 생성 Node가 Witness를 만듦

### Verkle Tree의 Proof Size

Key 하나의 경로 크기임. $N=2^{32}$, Hash와 Commitment는 32 B임. 식은 2024년 글에 있음. MPT와 Binary Merkle Tree의 값은 글에 적혀 있고, Verkle Tree의 128 B는 식에 대입한 값임

| 트리 | 식 | $N=2^{32}$일 때 |
| --- | --- | --- |
| MPT (Width 16) | $32 \times 15 \times \log_{16} N$ | 3,840 B |
| Binary Merkle Tree (Width 2) | $32 \times \log_2 N$ | 1,024 B |
| Verkle Tree (Width 256) | $32 \times \log_{256} N$ | 128 B |

- Verkle Tree에는 여기에 Multiproof 하나가 더해짐. Key가 몇 개든 하나임
- Value $m$개를 증명할 때 MPT는 $O(mk \log_k N)$, Verkle Tree는 $O(m \log_k N)$에 고정 크기 Multiproof를 더한 값임

Block 하나의 최악 Witness임. 2024년 글의 계산이고, Gas Limit 30,000,000과 접근 1회당 2,400 Gas(Block당 최대 12,500회 접근)를 가정함

| 트리 | 최악의 Witness | 비고 |
| --- | --- | --- |
| MPT | 약 330 MB | 접근마다 Code 전체(최대 24,000 B) 포함 |
| Binary Merkle Tree, Code를 트리에 포함 | 약 10.4 MB | |
| Verkle Tree | 약 1.3 MB | 공격자가 긴 공통 앞부분을 가진 Address를 만들면 약 2.6 MB |
| Verkle Tree, 인접 접근만 한 경우 | 약 4.8 MB | 인접 접근 1회당 200 Gas, 접근마다 32 B를 가정 |

마지막 행의 식은 30,000,000 / 200 × 32임. 글에는 4,800,800 B로 적혀 있으나 식을 계산하면 4,800,000 B임

### MPT와 Verkle Tree의 Proof Size 비교

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

- Vitalik Buterin의 2021년 글의 표임. Width 256, Commitment와 Opening Proof가 48 B인 KZG, Key가 고르게 분포한 트리를 가정함. Key와 Value 자체는 포함하지 않음. 단위는 B임

    | 트리 크기 | Key 1개 | Key 1,000개 | Key 10,000개 |
    | --- | --- | --- | --- |
    | $2^{24}$ | 272 | 59,792 | 457,616 |
    | $2^{32}$ | 320 | 107,744 | 937,472 |

- $2^{32}$ 크기의 트리에서 Key 1,000개일 때 Key당 약 108 B임. Key 1개일 때의 320 B보다 훨씬 작음. 상위 Level의 Commitment를 여러 Key가 공유하고 Multiproof가 하나뿐이기 때문임
- 같은 글의 보정: Key가 무작위로 분포하면 깊이가 약 0.6 늘어 Key당 약 30 B가 더 듦. 32 B Commitment를 쓰면 크기가 약 3분의 1 줄어듦
- 같은 글의 결론: Verkle Tree의 Proof는 Width 16인 MPT보다 20~30배 이상, 이상적인 Binary Merkle Tree보다 6~8배 작음
- [ethereum.org](https://ethereum.org/roadmap/verkle-trees/)는 Value 1,000개 기준으로 MPT의 Witness 약 3.5 MB(7 Level), Verkle Tree의 Witness 약 150 kB(4 Level)를 들고 약 23배 작다고 적음

<br />

## Ethereum 제안(EIP-6800)의 실제 구조

앞의 구조는 단순화한 것임. 실제 제안인 [EIP-6800](https://eips.ethereum.org/EIPS/eip-6800)은 Account, Storage, Code를 모두 하나의 트리에 넣음. Key와 Value는 모두 32 Byte임

이 장은 2026년 11월 23일에 확인한 EIP-6800 본문을 기준으로 함. 이전 시점의 자료는 배치가 다를 수 있음. 예를 들어 Anatomy of a Verkle proof는 Balance와 Nonce를 서로 다른 Suffix에 두는 이전 배치로 설명함

### Key의 구성: Stem과 Suffix

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
- Gas 비용에도 반영함. 같은 그룹 안에서 인접한 Storage Slot에 접근하는 비용을 2,100 Gas에서 200 Gas로 낮춤. 세부 규칙은 [EIP-4762](https://eips.ethereum.org/EIPS/eip-4762)가 정함

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

## 원 논문과 Ethereum 제안의 차이

### Kuszmaul의 Verkle Trees 논문

Verkle Tree를 처음 제시한 글은 John Kuszmaul의 [Verkle Trees](https://math.mit.edu/research/highschool/primes/materials/2018/Kuszmaul.pdf)(2018)다. Massachusetts Institute of Technology(MIT)의 PRIMES 프로그램에서 나온 12쪽 논문이고, 아이디어는 멘토 Alin Tomescu가 제공했다고 적혀 있다.

논문의 내용은 다음과 같다. $n$은 Leaf의 수이고 $k$는 Width다.

- **문제**: Width가 2인 Merkle Tree의 Proof 크기는 $O(\log_2 n)$이다. Leaf가 $2^{30}$개이고 Hash가 256 Bit면 약 1 kB다. Width가 $k$인 Merkle Tree는 Level마다 형제 $k-1$개가 필요해 $O(k \log_k n)$으로 오히려 커진다.
- **설계**: Merkle Tree와 같은 방식으로 트리를 만들되 Hash 대신 Vector Commitment를 쓴다. Opening Proof는 크기가 고정이므로 Proof 크기는 $O(\log_k n)$이다. Width 2인 Merkle Tree보다 $\log_2 k$배 작다. $k=1024$면 10배, $k=32$면 5배다.
- **비용**: Node를 만들 때 자식마다 Opening Proof를 미리 계산한다. 그래서 트리 구성에 $O(kn)$, Value 하나 갱신에 $O(k \log_k n)$이 든다. $k$는 Proof 크기와 계산량을 맞바꾸는 값이다.
- **구현**: Catalano와 Fiore의 Vector Commitment를 썼다. Bilinear Group과 Computational Diffie-Hellman 가정에 기반한다. Leaf가 $2^{14}$개인 트리에서 Leaf 1개당 구성 시간은 $k=1024$일 때 110.1 ms, $k=32$일 때 8.4 ms였다.

| 구조 | 트리 구성 | Value 하나 갱신 | Proof 크기 |
| --- | --- | --- | --- |
| Merkle Tree | $O(n)$ | $O(\log_2 n)$ | $O(\log_2 n)$ |
| Width $k$인 Merkle Tree | $O(n)$ | $O(k \log_k n)$ | $O(k \log_k n)$ |
| Vector Commitment | $O(n^2)$ | $O(n)$ | $O(1)$ |
| Width $k$인 Verkle Tree | $O(kn)$ | $O(k \log_k n)$ | $O(\log_k n)$ |

### Ethereum 제안이 바꾼 것

Ethereum 제안은 논문의 구조에서 세 가지를 바꿨다.

- **Vector Commitment가 다르다.** Pedersen Commitment와 IPA를 쓴다.
- **Opening Proof를 Node에 저장하지 않는다.** Block이 접근한 Value에 대해서만 그때 계산한다. 그래서 Value 하나를 갱신하는 비용이 Width에 비례하지 않는다.
- **여러 Opening Proof를 Multiproof 하나로 묶는다.** 논문은 Opening Proof를 묶는 방법을 다루지 않는다.

논문의 비교 대상은 Width 2인 Merkle Tree다. Ethereum의 현재 트리는 Width가 16이어서 줄어드는 폭이 더 크다.

<br />

## 한계와 현재 상태

### Verkle Tree의 한계

앞의 세 가지는 Vitalik Buterin의 [2024년 글](https://vitalik.eth.limo/general/2024/10/23/futures4.html)이 Verkle Tree의 단점으로 든 것이고, 뒤의 두 가지는 같은 글의 수치와 설명에서 읽은 것임

- **Quantum Resistance(양자 내성)가 없음**: Commitment가 타원곡선에 기반하므로 양자 컴퓨터에 취약함. 도입하더라도 나중에 다시 교체해야 함
- **SNARK 안에서 증명하기 어려움**: Succinct Non-interactive Argument of Knowledge(이하 SNARK)는 계산이 올바르게 수행됐음을 짧은 증명으로 보이는 방식임. Verkle Tree의 Proof를 검증하는 계산에는 256 Bit 연산이 많아, 이를 SNARK로 증명하려면 큰 오버헤드나 전용 구조가 필요함
- **State Root 갱신이 Hash 기반 트리보다 느림**: State 전체를 저장하는 Node가 Chain을 따라잡는 Sync 시간이 길어짐
- **최악의 Witness는 여전히 MB 단위임**: 앞의 표에서 약 1.3~4.8 MB임
- **Width에 상한이 있음**: 트리가 너무 넓어지면 Proof 계산이 비효율적이 됨

같은 글이 든 Verkle Tree의 장점은 세 가지임. Proof 생성이 빠르고, 구현이 성숙했고, Witness를 다음 Block에 맞춰 갱신할 수 있음

### 대안: STARK로 증명하는 Binary Hash Tree

Binary Merkle Tree를 쓰고, Merkle Proof 전체를 Scalable Transparent Argument of Knowledge(이하 STARK)로 증명해 Proof를 대체하는 방식임. STARK는 SNARK와 같은 목적의 증명 방식이고, 이 방식은 타원곡선 없이 Hash 함수에만 의존함

아래는 2024년 10월 시점의 비교임. 보수적 Hash는 오래 쓰여 검증된 Hash 함수이고, 공격적 Hash는 증명에 유리하도록 새로 설계되어 검증이 덜 된 Hash 함수임

| 방식 | Proof 크기 | 보안 가정 | 최악의 증명 생성 시간 |
| --- | --- | --- | --- |
| Verkle Tree | 데이터 + 약 100~2,000 kB | 타원곡선 (Quantum Resistance 없음) | 1초 미만 |
| STARK + 보수적 Hash (SHA256, BLAKE) | 데이터 + 약 100~300 kB | 보수적 Hash 함수 | 10초 초과 |
| STARK + 공격적 Hash (Poseidon, Ajtai) | 데이터 + 약 100~300 kB | 덜 검증된 새 Hash 함수 | 1~2초 |

같은 글은 최악의 경우 약 660,000번의 Hash를 증명해야 하고, 보수적 Hash는 당시 초당 약 10,000~30,000번을 증명할 수 있다고 적음

### 현재 상태

2026년 11월 23일 기준이다.

- **EIP-6800**: Stagnant 단계로 표시되어 있다. Stagnant는 Draft, Review, Last Call 단계의 EIP가 6개월 이상 활동이 없을 때 옮겨지는 단계다. 2026년 3월 6일에는 consensus-specs 저장소에서 EIP-6800 사양이 [제거](https://github.com/ethereum/consensus-specs/pull/4984)됐다.
- **EIP-7864**: 2025년 1월에 제안된 Hash 기반 Unified Binary Tree이고, [Draft 단계](https://eips.ethereum.org/EIPS/eip-7864)다. 제안 동기는 타원곡선의 Quantum Resistance 우려와 증명 시스템의 발전이다. Hash 함수는 확정되지 않았다. 참조 구현은 BLAKE3를 쓰고, Keccak과 Poseidon2가 후보다.
- **EIP-8297**: 2026년 6월에 EIP-7864의 저자들에 두 명이 더해진 저자진이 제안한 [Partitioned Binary Tree](https://eips.ethereum.org/EIPS/eip-8297)이고 Draft 단계다. Key의 첫 Byte로 Account Header, Code, Storage 구역을 나누고, Code를 Account가 아니라 Code Hash 기준으로 저장한다. go-ethereum은 2026년 11월 10일에 "EIP-8297이 EIP-7864를 대체한다"는 설명과 함께 EIP-7864와 EIP-4762의 구현을 [제거](https://github.com/ethereum/go-ethereum/pull/36091)했다.
- **이어지는 것**: Stateless Client라는 목표는 그대로다. 두 Binary Tree 제안은 하나의 트리에 모두 넣는 구조, 함께 읽는 Value를 같은 Stem에 묶는 방식, Code를 31 Byte Chunk로 나누는 방식처럼 Commitment 방식과 무관한 EIP-6800의 설계를 이어받았다. 달라지는 것은 부모 Node를 만드는 방법과 Proof를 줄이는 방법이다. EIP-8297은 Gas 비용 체계를 바꾸지 않는다고 적는다.
- **Ethereum Foundation**: 2026년 2월 18일 글에서 State 구조의 장기 방향을 [Binary Tree와 Statelessness](https://blog.ethereum.org/2026/02/18/protocol-priorities-update-2026)로 적었다.
- **보도**: [The Block](https://www.theblock.co/post/391681/vitalik-buterin-lays-out-a-two-part-plan-to-overhaul-ethereums-execution-layer-from-the-ground-up)은 2026년 3월, Verkle Tree가 2026년 Hard Fork 후보로 거론됐으나 양자 컴퓨팅 우려로 2024년 중반부터 Binary Tree가 다시 주목받았다고 보도했다. [The Defiant](https://thedefiant.io/newsletter/defi-daily/ethereum-bets-its-technical-future-on-privacy-and-quantum-defense)는 2026년 8월, Ethereum Foundation의 Roadmap 초안에서 Verkle Tree 자리에 EIP-7864의 Unified Binary Tree가 들어갔다고 보도했다.
- **일정**: State Tree 변경이 들어갈 Hard Fork가 확정됐다는 자료는 찾지 못했다. 2026년 10월 Sepolia Testnet에서 활성화된 Glamsterdam 업그레이드의 [EIP 목록](https://blog.ethereum.org/2026/10/16/glamsterdam-hoodi-announcement)에 State Tree 변경은 없다. Ethereum Research의 2026년 9월 [의견 글](https://ethresear.ch/t/how-hegota-can-influence-the-state-roadmap/25895)은 Partitioned Binary Tree로의 전환을 Glamsterdam 다음 업그레이드인 Hegotá보다 뒤로 예상한다.
- **주의**: [ethereum.org](https://ethereum.org/roadmap/verkle-trees/)의 Verkle Tree 문서는 Binary Tree를 언급하지 않고, Verkle Tree를 여전히 Stateless Client로 가는 단계로 설명한다. 같은 문서는 MPT를 Leaf를 둘씩 묶어 Hash하는 트리로 설명하지만 MPT의 Width는 16이다. Witness 크기가 Leaf 개수와 무관하게 고정이라고도 적지만 고정인 것은 Multiproof의 크기다.

<br />

## 결론

Stateless Client가 되려면 Witness가 작아야 하고, Witness 크기는 State Tree의 Proof 크기가 정한다. MPT는 Hash로 부모 Node를 만들기 때문에 Level마다 형제 Hash를 보내야 하고, Proof가 (Width - 1) × 깊이에 비례한다. Verkle Tree는 부모 Node를 Vector Commitment로 만들어 형제를 보내지 않는다. 형제가 없으므로 Width를 256으로 키워 깊이를 줄일 수 있고, 여러 Opening Proof는 Multiproof 하나로 묶인다. 그 결과 Key 하나의 경로 크기는 3,840 B에서 128 B로, Value 1,000개의 Witness는 약 3.5 MB에서 약 150 kB로 줄어든다.

이 구조는 Kuszmaul의 논문이 제시했고, Ethereum 제안은 Commitment를 Pedersen Commitment와 IPA로 바꾸고 Stem과 Suffix로 Value를 묶는 배치를 더했다. 대가는 타원곡선에 대한 의존이다. Quantum Resistance가 없고 SNARK로 증명하기 어렵다는 점 때문에, 2026년 11월 기준 Ethereum의 논의는 Hash 기반 Binary Tree로 옮겨 가 있다.