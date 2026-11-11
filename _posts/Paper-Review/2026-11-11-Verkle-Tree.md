---
title: "Verkle Tree"
date: 2026-11-11
categories: [Paper Review]
tags: [Blockchain, ZKP, Verkletree, MPT]
---

# Verkle Tree


Verkle Tree는 상태 트리의 부모 노드를 hash 대신 Vector Commitment로 만들어, 형제 노드 없이 값을 증명하는 트리다. 목적은 Stateless Client가 쓸 수 있을 만큼 Witness를 줄이는 것이다. 2026년 11월 기준 이더리움 메인넷은 Verkle Tree를 채택하지 않았고, 로드맵의 무게는 hash 기반 Binary Tree로 옮겨 갔다. 자세한 내용은 9장에 있다.

## 1. 배경: Stateless Client와 Witness 크기

### 용어

- **State(상태)**: 모든 계정의 잔액, nonce, 컨트랙트 코드, 스토리지를 합친 데이터다.
- **State Root(상태 루트)**: 상태 전체를 요약한 값 하나다. 블록 헤더에 들어간다.
- **Proof(증명)**: 특정 값이 State Root가 가리키는 상태에 포함되어 있음을 보이는 데이터다.
- **Witness**: 블록의 트랜잭션을 실행하는 데 필요한 상태 조각과, 그 조각에 대한 Proof를 묶은 것이다.
- **Stateless Client**: 상태 데이터베이스를 저장하지 않고, 블록과 함께 받은 Witness로 블록을 검증하는 클라이언트다.

### 현재의 검증 방식과 문제

현재 클라이언트는 블록의 트랜잭션을 모두 다시 실행하고, 로컬 상태 트리를 갱신한 뒤, 계산한 State Root가 블록에 적힌 값과 같은지 비교한다. 트랜잭션이 어떤 계정과 스토리지를 읽을지 미리 알 수 없으므로 상태 전체를 로컬에 보관해야 한다.

Vitalik Buterin의 2024년 10월 글은 클라이언트가 저장하는 상태를 수백 GB로, 증가 속도를 연 약 30 GB로 적었다. [source](https://vitalik.eth.limo/general/2024/10/23/futures4.html) 저장 부담이 커지면 일반 하드웨어로 검증에 참여하기 어려워진다.

### Stateless Client가 요구하는 조건

Stateless Client는 상태 대신 Witness를 받는다. Witness는 블록과 함께 네트워크로 전파되므로, 12초 slot 안에 받아서 처리할 수 있을 만큼 작아야 한다. [source](https://ethereum.org/roadmap/verkle-trees)

현재 구조인 Merkle Patricia Trie(MPT)로는 이 조건을 맞추기 어렵다. 원문이 제시한 수치는 다음과 같다. EIP는 Ethereum Improvement Proposal(이더리움 개선 제안)의 줄임말이다.

| 항목 | 수치 | 가정 | 출처 |
| --- | --- | --- | --- |
| MPT 접근 1회당 Witness | 평균 약 3 kB | 최악의 경우 약 3배 | [EIP-6800](https://eips.ethereum.org/EIPS/eip-6800) |
| MPT 블록 Witness | 약 18 MB | 블록당 접근 6,000회 (15M gas, 접근당 2,500 gas) | [EIP-6800](https://eips.ethereum.org/EIPS/eip-6800) |
| MPT 블록 Witness, 최악 | 약 330 MB | gas limit 30M, 접근당 2,400 gas, 접근마다 컨트랙트 코드 24,000 B 포함 | [Vitalik 2024](https://vitalik.eth.limo/general/2024/10/23/futures4.html) |

수치는 각 글이 쓰인 시점의 gas limit을 전제로 한다. gas limit이 바뀌면 값도 비례해서 바뀐다.

### 범위: Weak Statelessness

Verkle Tree가 겨냥하는 것은 Weak Statelessness(약한 상태 비저장)다. 블록을 만드는 쪽(block proposer, builder)은 Witness를 생성해야 하므로 여전히 전체 상태를 가진다. 나머지 노드만 상태 없이 검증한다. 모든 노드가 상태를 버리는 Strong Statelessness(강한 상태 비저장)와는 다르다. [source](https://ethereum.org/en/roadmap/statelessness/)

정리하면, 풀어야 할 문제는 "상태 트리가 만드는 Proof의 크기"다. Proof 크기는 트리 구조가 결정하므로 트리 구조를 바꿔야 한다.

## 2. 현재 구조: Merkle Patricia Trie

Merkle Patricia Trie(MPT)는 각 level마다 형제 노드의 hash를 Proof에 넣어야 하므로, Proof 크기가 (width − 1) × 깊이에 비례한다. width는 한 노드가 가질 수 있는 자식의 최대 개수다.

### 두 가지 구성 요소

- **Merkle Tree(머클 트리)**: 부모 노드가 자식 노드들의 hash를 합쳐 다시 hash한 값을 가진다. 데이터 전체가 Merkle Root 하나로 요약된다.
- **Patricia Trie(패트리샤 트라이)**: key를 한 자리씩 따라 내려가는 트리다. 분기가 없는 구간은 하나의 노드로 묶어 경로를 줄인다.

MPT는 Patricia Trie의 각 노드를 hash로 참조하는 구조다. 노드를 Recursive Length Prefix(RLP)로 인코딩한 뒤 keccak256으로 hash한 값이 그 노드의 참조가 된다. [source](https://ethereum.org/en/developers/docs/data-structures-and-encoding/patricia-merkle-trie)

### 노드 종류

key는 4비트 단위인 nibble로 나뉘어 경로가 된다. nibble 하나는 16진수 한 자리(0\~f)다.

| 노드 | 형태 | 역할 |
| --- | --- | --- |
| Branch node | 17개 항목 `[v0 ... v15, vt]` | nibble 0\~f에 해당하는 자식 16개와 값 1개 |
| Extension node | 2개 항목 `[encodedPath, key]` | 분기 없는 공통 경로와 다음 노드의 참조 |
| Leaf node | 2개 항목 `[encodedPath, value]` | 남은 경로와 값 |

이더리움 상태 트리에서 key는 keccak256(address)이고, 경로 길이는 64 nibble이다. 값은 계정의 nonce, balance, storageRoot, codeHash다. 컨트랙트 스토리지는 계정마다 별도의 트리에 들어가고, 코드는 트리 밖에 있으며 codeHash만 트리에 남는다.

### 예시: 계정 3개

설명을 위해 key를 2바이트로 줄인 예시다. 실제 key는 32바이트다.

| 계정 | key | 값 |
| --- | --- | --- |
| a1 | 0x7340 | nonce 2, balance 4 ETH |
| a2 | 0x7bc6 | nonce 3, balance 2 ETH |
| a3 | 0xd563 | nonce 4, balance 5 ETH |

1. Root는 branch node다. a1과 a2는 첫 nibble이 7로 같으므로 slot 7 아래로 함께 내려간다.
2. slot 7 아래에 branch node가 하나 더 있다. 두 번째 nibble이 3과 b로 갈리므로 각각 leaf node가 된다. a1의 leaf는 남은 경로 40을, a2의 leaf는 남은 경로 c6을 가진다.
3. a3는 첫 nibble d를 혼자 쓴다. Root의 slot d가 바로 leaf node를 가리키고, leaf는 남은 경로 563을 가진다.

여러 key가 긴 공통 경로를 가지면, 그 구간은 extension node 하나로 묶인다.

### 검증 방법

검증자는 leaf에서 Root까지 hash를 다시 계산하고, 결과가 알고 있는 State Root와 같은지 확인한다. Branch node의 hash를 계산하려면 그 노드의 모든 slot 값이 필요하다. 따라서 경로 위의 branch node마다 나머지 slot의 hash, 즉 형제 노드 hash가 Proof에 들어간다.

위 예시에서 a1을 검증하려면 다음이 필요하다.

- a1의 leaf node
- 두 번째 branch node의 형제: a2 leaf의 hash
- Root branch node의 형제: a3 leaf의 hash

이 예시는 slot이 거의 비어 있어 형제가 level당 1개뿐이다. 실제 상태 트리의 상위 level은 slot이 거의 차 있으므로 level당 최대 15개의 형제 hash가 들어간다.

### Proof Size

트리에 저장된 key-value 쌍의 개수를 N, width를 k라 하면 깊이는 약 log\_k N이다. level마다 형제 hash가 최대 k − 1개 필요하다.

```latex
\text{Proof Size} \approx 32\,\text{B} \times (k-1) \times \log_k N
```

k = 16이면 32 × 15 × log\_16 N = 120 × log\_2 N 바이트다. N = 2^32일 때 약 3,840 B다. [source](https://vitalik.eth.limo/general/2024/10/23/futures4.html)

width를 키우면 깊이는 줄지만 형제 수가 더 빨리 늘어난다. k = 256이면 level당 형제 hash가 최대 255개다. hash 기반 트리에서는 width 2가 Proof 크기에 가장 유리하고, width 16은 비효율적이다. [source](https://vitalik.eth.limo/general/2021/06/18/verkle.html)

### 업데이트

값이 바뀌면 그 leaf와, leaf에서 Root까지 경로 위에 있는 노드의 hash만 다시 계산한다. 나머지 노드는 그대로다.

## 3. Verkle Tree의 핵심 아이디어

Verkle Tree는 부모 노드를 자식들의 hash가 아니라 자식들에 대한 Vector Commitment로 만든다. 이름은 Vector commitment와 Merkle Tree를 합친 말이다. [source](https://ethereum.org/roadmap/verkle-trees) John Kuszmaul이 2018년 논문에서 처음 제안했다. [source](https://vitalik.eth.limo/general/2021/06/18/verkle.html)

### 형제 노드가 필요 없는 이유

hash는 입력 전체를 알아야 다시 계산할 수 있다. 그래서 MPT의 검증자는 부모의 모든 자식 값을 받아야 한다.

Vector Commitment는 "이 commitment의 i번째 값은 v다"를 나머지 값 없이 짧은 proof로 보일 수 있다. 그래서 Verkle Tree의 검증자는 경로 위의 commitment와, 각 부모-자식 관계에 대한 proof만 받는다.

&#91;embedded content: MPT proof와 Verkle proof 비교, key 0x7340의 경로\]

왼쪽 MPT는 경로가 지나는 branch node마다 나머지 slot의 hash를 보낸다. 오른쪽 Verkle Tree는 경로 위의 commitment만 보낸다. 비교를 위해 양쪽 모두 두 level로 그렸다.

### width를 넓힐 수 있는 이유

형제를 보내지 않으므로 width를 키워도 level당 proof 데이터가 늘지 않는다. width를 키우면 깊이만 줄어든다. 이더리움에 제안된 Verkle Tree의 width는 256이고, 1024를 선호하는 의견도 있었다.

width의 상한은 proof 생성 비용이 정한다. proof 생성에는 노드마다 width에 비례하는 연산이 든다. [source](https://vitalik.eth.limo/general/2021/06/18/verkle.html)

### MPT와 비교

| 항목 | Merkle Patricia Trie | Verkle Tree (이더리움 제안) |
| --- | --- | --- |
| 부모 노드의 값 | 자식들의 hash | 자식들에 대한 Vector Commitment |
| width | 16 | 256 |
| 깊이 | 약 log\_16 N | 약 log\_256 N |
| level당 proof 데이터 | 형제 hash 최대 15개 (480 B) | commitment 1개 (32\~48 B) |
| 여러 key를 검증할 때 | key마다 형제 hash 추가 | opening proof를 multiproof 1개로 묶음 |

commitment 1개의 크기는 곡선에 따라 32\~48 B다. [source](https://vitalik.eth.limo/general/2021/06/18/verkle.html) multiproof는 7장에서 다룬다.

## 4. Commitment Scheme

Verkle Tree가 Commitment Scheme(커밋먼트 스킴)에서 쓰는 성질은 두 가지다. 한 번 만든 commitment는 다른 값으로 열 수 없다는 점, 그리고 특정 위치의 값을 짧은 proof로 열 수 있다는 점이다.

### 기본 개념

- **Commit**: 값으로부터 고정 크기의 값 하나(commitment)를 만든다.
- **Open**: 나중에 값을 공개하고, 그 값이 commitment와 맞는지 확인한다.
- **Binding**: 하나의 commitment를 서로 다른 두 값으로 열 수 없다.
- **Hiding**: commitment만 보고는 값을 알 수 없다.

밀봉한 봉투에 비유할 수 있다. 봉투를 받은 사람은 내용을 볼 수 없고(hiding), 보낸 사람은 내용을 바꿀 수 없다(binding).

Verkle Tree에서 필요한 것은 binding이다. 이더리움의 상태 값은 공개 데이터이므로 hiding은 목적이 아니다. 실제로 이더리움 제안의 commitment 식에는 값을 감추기 위한 랜덤 항이 없다. [source](https://blog.ethereum.org/2021/12/02/verkle-tree-structure/)

### Vector Commitment

Vector Commitment는 값의 목록 (v\_0, ..., v\_{n-1}) 전체를 commitment 하나로 만들고, "i번째 값은 v\_i다"를 proof로 보이는 스킴이다.

Merkle Tree도 hash로 만든 Vector Commitment다. Merkle Root가 commitment이고, proof는 log\_2 n개의 hash다. Verkle Tree는 n이 커져도 proof가 커지지 않는 스킴이 필요하다.

### Polynomial Commitment

Polynomial Commitment(다항식 커밋먼트)는 다항식 f(x)를 commit하고, 특정 점 z에서 f(z) = y임을 proof로 보이는 스킴이다.

이를 Vector Commitment로 쓰는 방법은 다음과 같다. 값 v\_0, ..., v\_{n-1}에 대해 f(i) = v\_i를 만족하는 다항식을 만든다. 이 다항식은 Lagrange interpolation(라그랑주 보간)으로 유일하게 정해진다. 그러면 "i번째 값이 v\_i다"는 "f(i) = v\_i다"와 같은 말이 된다. [source](https://vitalik.eth.limo/general/2021/06/18/verkle.html)

proof의 근거는 다음 사실이다. f(z) = y이면 f(x) − y는 (x − z)로 나누어떨어진다. 그 역도 성립한다.

```latex
q(x) = \frac{f(x) - y}{x - z}
```

q(x)가 다항식으로 존재한다는 것이 곧 f(z) = y의 증거다. proof는 q(x)에 대한 commitment다.

### 단순화한 계산 예시

2장 예시의 a1 leaf가 세 값을 가진다고 하자. index 0은 남은 key 40, index 1은 nonce 2, index 2는 balance 4다. 타원곡선 대신 정수로 계산한다.

1. **다항식 만들기**: f(0) = 40, f(1) = 2, f(2) = 4를 만족하는 다항식은 f(x) = 20x² − 58x + 40이다.
2. **Commit**: 점 r = 7에서 C = f(7) = 980 − 406 + 40 = 614다.
3. **Open**: index 2의 값 y = 4를 연다. q(x) = (20x² − 58x + 36) / (x − 2) = 20x − 18이고, proof는 π = q(7) = 122다.
4. **Verify**: C − y = π × (r − z)를 확인한다. 614 − 4 = 610이고 122 × 5 = 610이므로 통과한다.

검증자는 C, y, π만 받았고 f(x)의 다른 값(40, 2)은 받지 않았다.

이 예시는 식의 구조만 보여준다. 실제 스킴과 다른 점이 두 가지 있다.

- r을 아는 사람은 거짓 값 y'에 대해서도 π' = (C − y') / (r − z)를 계산해 검증을 통과시킬 수 있다. 그래서 실제 스킴인 KZG(Kate-Zaverucha-Goldberg)에서는 이 점이 아무도 모르는 비밀값 s다. Trusted Setup에서 s를 뽑아 \[s^i\] 형태의 공개 파라미터만 남기고 s는 폐기한다.
- commitment와 proof는 정수가 아니라 타원곡선 위의 점이다. prover는 s를 모르는 채로 공개 파라미터를 조합해 점을 계산하고, verifier는 pairing으로 등식을 확인한다.

KZG의 실제 식은 다음과 같다. \[a\]는 생성원 G에 a를 곱한 점이고, f\_i는 f(x)의 i차 계수다.

```latex
C = [f(s)]_1 = \sum_i f_i\,[s^i]_1, \qquad \pi = [q(s)]_1, \qquad e(C - [y]_1,\ [1]_2) = e(\pi,\ [s - z]_2)
```

### KZG와 Pedersen + IPA

Verkle Tree에 쓸 수 있는 스킴은 두 가지가 거론된다. IPA는 Inner Product Argument(내적 증명)의 줄임말이다. [source](https://dankradfeist.de/ethereum/2021/07/27/inner-product-arguments.html)

| 항목 | KZG | Pedersen + IPA |
| --- | --- | --- |
| 보안 가정 | Bilinear group (pairing) | Discrete log |
| Trusted Setup | 필요 | 불필요 |
| Commitment 크기 | group element 1개 | group element 1개 |
| Proof 크기 | group element 1개 | group element 2 log\_2 n개 |
| 검증 비용 | pairing 1회 | group 연산 O(n) |

Pedersen commitment는 값 v\_i를 고정된 기저점 G\_i에 곱해 더한 점이다. 기저점들 사이의 discrete log 관계는 알려져 있지 않아야 한다.

```latex
C = v_0 G_0 + v_1 G_1 + \cdots + v_{n-1} G_{n-1}
```

두 스킴 모두 덧셈에 대해 동형(additively homomorphic)이다. 두 commitment를 더하면 두 벡터의 합에 대한 commitment가 된다. 이 성질이 6장의 업데이트와 7장의 multiproof를 가능하게 한다.

### 이더리움 제안이 고른 스킴

이더리움 제안은 KZG가 아니라 Pedersen + IPA를 쓴다.

- Commitment: Bandersnatch 곡선 위의 Pedersen commitment, 기저점 256개. [EIP-6800](https://eips.ethereum.org/EIPS/eip-6800)
- Opening proof: IPA. [source](https://blog.ethereum.org/2021/12/02/verkle-tree-structure/)
- 한 slot에 넣을 수 있는 값: Bandersnatch의 scalar field가 253비트 소수이므로 252비트까지다. [source](https://blog.ethereum.org/2021/12/02/verkle-tree-structure/)
- Commitment 크기: 32 B. KZG를 쓰는 경우는 48 B다. [source](https://vitalik.eth.limo/general/2021/06/18/verkle.html)

IPA는 검증 비용이 n에 비례한다는 단점이 있다. 여러 opening을 하나로 합쳐 한 번만 검증하면 이 비용을 나눠 부담할 수 있다. [source](https://dankradfeist.de/ethereum/2021/07/27/inner-product-arguments.html)

Vitalik Buterin과 Dankrad Feist의 설명 글은 KZG 표기를 쓴다. 이 문서의 수식도 KZG 표기를 따른다. 이후 장의 내용은 덧셈 동형성만 있으면 성립하므로 Pedersen + IPA에도 그대로 적용된다. [source](https://dankradfeist.de/ethereum/2021/06/18/pcs-multiproofs.html)

## 5. Verkle Tree의 구조

이더리움 제안은 계정, 스토리지, 코드를 모두 하나의 key-value 트리에 넣는다. key와 value는 모두 32바이트다. 이 장은 EIP-6800의 현재 본문을 기준으로 한다. [EIP-6800](https://eips.ethereum.org/EIPS/eip-6800)

### Key: stem과 suffix

key 32바이트는 앞 31바이트인 stem과 마지막 1바이트인 suffix로 나뉜다.

- **stem**: 트리 안에서의 경로를 정한다. address와 tree\_index를 합쳐 Pedersen hash한 값의 앞 31바이트다.
- **suffix**: 같은 stem 아래에 있는 256개 값 중 하나를 고른다.

address 자체가 경로가 되는 것이 아니라는 점에 유의한다. stem이 hash의 결과이므로 key는 트리 전체에 고르게 퍼진다.

### 한 stem 아래에 모이는 값

계정의 첫 번째 stem(tree\_index 0)에는 자주 함께 읽는 값이 모여 있다.

| suffix | 내용 | 상수 |
| --- | --- | --- |
| 0 | version, code size, nonce, balance를 한 값에 묶음 | BASIC\_DATA\_LEAF\_KEY = 0 |
| 1 | code hash | CODE\_HASH\_LEAF\_KEY = 1 |
| 64\~127 | 스토리지 slot 0\~63 | HEADER\_STORAGE\_OFFSET = 64 |
| 128\~255 | 코드의 앞부분 chunk | CODE\_OFFSET = 128 |

나머지 스토리지 slot과 코드 chunk는 다른 tree\_index의 stem으로 간다. 코드는 31바이트 chunk로 나뉘어 트리에 들어간다. MPT에서는 코드가 트리 밖에 있어, 코드 접근을 증명하려면 코드 전체를 Witness에 넣어야 했다.

같은 stem에 속한 값은 경로를 공유한다. nonce와 balance, 앞쪽 스토리지 slot을 함께 읽어도 경로 위의 commitment는 한 번만 Witness에 들어간다. EIP-6800은 이를 gas 비용에도 반영해, 같은 그룹 안에서 인접한 slot에 접근하는 비용을 2,100에서 200으로 낮춘다.

### 노드 종류

| 노드 | commitment의 대상 | 역할 |
| --- | --- | --- |
| Inner node | 자식 256개의 commitment | stem을 1바이트씩 따라 분기. MPT의 branch node에 해당 |
| Extension node | (1, stem, C1, C2) | 하나의 stem과 그 아래 값 256개를 대표 |

Extension node는 값을 두 commitment로 나눠 담는다. C1은 suffix 0\~127의 값에 대한 commitment이고, C2는 suffix 128\~255의 값에 대한 commitment다.

나누는 이유는 slot 크기다. commitment의 한 slot에는 252비트까지만 들어가는데 값은 32바이트(256비트)다. 그래서 값 하나를 16바이트 두 조각으로 나눠 slot 두 개에 넣는다. 값 128개가 slot 256개를 채우므로 commitment가 두 개 필요하다. 아래쪽 조각에는 2^128을 더해, 값이 0인 경우와 값이 없는 경우를 구분한다.

부모가 자식 commitment를 담을 때도 같은 제약이 있다. commitment는 곡선 위의 점이므로, scalar 값으로 변환한 뒤 부모의 slot에 넣는다.

### 깊이

랜덤한 key n개를 width d인 트리에 넣으면 평균 깊이는 log\_d n + 2/3다. [source](https://dankradfeist.de/ethereum/2021/06/18/verkle-trie-for-eth1.html)

### 예시: 계정 3개

2장의 예시를 width 256으로 옮기면 다음과 같다. 설명을 위해 key의 첫 바이트를 그대로 경로로 쓴다. 실제로는 위에서 설명한 대로 hash로 만든 stem을 쓴다.

- Root는 slot 256개를 가진 inner node다.
- a1(0x7340)은 slot 0x73, a2(0x7bc6)는 slot 0x7b, a3(0xd563)는 slot 0xd5로 간다.
- 세 key가 첫 바이트에서 모두 갈리므로 Root 바로 아래가 leaf다.

MPT에서는 a1과 a2가 첫 nibble 7을 공유해 level이 하나 더 필요했다. width가 넓어지면 같은 key 집합에서 깊이가 줄어든다.

## 6. 업데이트 방식

값이 바뀌면 leaf에서 Root까지 경로 위의 commitment만 갱신한다. 각 commitment는 처음부터 다시 계산하지 않고, 바뀐 분량만 더한다.

### 덧셈 동형성을 이용한 갱신

commitment는 각 값에 기저점을 곱해 더한 점이다. i번째 값이 v\_i에서 v\_i'로 바뀌면 새 commitment는 다음과 같다.

```latex
C' = C + (v_i' - v_i)\,G_i
```

나머지 255개 값은 읽을 필요가 없다. 타원곡선 곱셈 1회와 덧셈 1회로 끝난다.

기저점 G\_i는 스킴에 따라 다르다. Pedersen commitment에서는 고정된 기저점이다. KZG에서 값을 f(i) = v\_i 형태로 담는 경우에는 Lagrange 기저 다항식 L\_i(x)에 대한 commitment \[L\_i(s)\]다.

### 경로를 따라 올라가기

1. leaf의 값이 바뀌면 extension node의 C1 또는 C2를 위 식으로 갱신한다.
2. C1 또는 C2가 바뀌었으므로 extension node의 commitment를 같은 식으로 갱신한다.
3. 부모 inner node는 자식 commitment를 scalar로 바꾼 값을 slot에 담고 있다. 새 scalar와 이전 scalar의 차이를 구해 부모를 같은 식으로 갱신한다.
4. Root까지 3을 반복한다.

commitment 1개를 갱신할 때마다 타원곡선 곱셈이 1회 든다. 값 하나를 고치는 데 Root를 포함해 약 4회가 필요하다. 기저점의 배수를 미리 계산해 두면 더 빨라진다. [source](https://vitalik.eth.limo/general/2021/06/18/verkle.html)

### 예시

계정 a1\~a5가 있는 트리에서 a1이 a4에게 3 ETH를 보내는 트랜잭션이 성공했다고 하자. 계정 값은 앞 장의 예시와 별개다.

| 계정 | 바뀌는 값 |
| --- | --- |
| a1 | nonce 4 → 5, balance 10 → 7 ETH |
| a4 | balance 4 → 7 ETH |

a1의 leaf와 a4의 leaf가 바뀌고, 두 leaf에서 Root까지의 commitment가 갱신된다. 두 경로가 만나는 지점부터 Root까지는 두 변경을 한 번에 반영한다. a2, a3, a5의 leaf와 그 위의 다른 가지는 그대로다.

### 트리에 저장되는 것

트리의 노드가 저장하는 것은 commitment다. opening proof는 노드에 저장하지 않는다. proof는 블록이 접근한 값에 대해 Witness를 만들 때 계산한다.

## 7. 검증 방법과 Multiproof

Verkle proof는 경로 위의 commitment와, 모든 부모-자식 관계를 한 번에 증명하는 고정 크기의 multiproof 하나로 이루어진다.

### 값 하나를 검증할 때

검증자는 Root commitment를 이미 알고 있다. 경로의 각 level에서 다음 관계를 확인한다.

- 부모 commitment를 index z에서 열면 값이 y다.
- z는 key가 정하는 자식의 위치다.
- y는 자식 commitment를 scalar로 바꾼 값이다. 마지막 level에서는 저장된 값이다.

5장의 예시에서 a1(0x7340)을 검증한다면 두 개의 opening이 필요하다. Root commitment를 index 0x73에서 열어 a1 leaf의 commitment를 확인하고, leaf commitment를 열어 nonce와 balance를 확인한다. a2와 a3의 commitment는 필요하지 않다.

### 값 여러 개를 검증할 때의 문제

블록 하나는 수천 개의 값에 접근한다. opening마다 proof를 따로 만들면 proof 개수가 (key 개수) × (깊이)로 늘어난다. Multiproof는 이 proof들을 하나로 합친다.

### Multiproof 프로토콜

증명할 내용은 m개의 opening이다. i번째 opening은 "commitment C\_i가 나타내는 다항식 f\_i에 대해 f\_i(z\_i) = y\_i다"이다. 아래는 Dankrad Feist의 방식이다. [source](https://dankradfeist.de/ethereum/2021/06/18/pcs-multiproofs.html)

**Prover**

1. 모든 C\_i, y\_i, z\_i를 hash해 난수 r을 얻는다.
2. 각 opening의 몫을 r의 거듭제곱으로 가중해 더한 다항식 g를 만들고, 그 commitment D를 보낸다.
3. r과 D를 hash해 난수 t를 얻는다.
4. 다항식 h를 만들고, h − g를 점 t에서 여는 proof π를 보낸다.

```latex
g(X) = \sum_{i=0}^{m-1} r^i\,\frac{f_i(X) - y_i}{X - z_i}, \qquad h(X) = \sum_{i=0}^{m-1} r^i\,\frac{f_i(X)}{t - z_i}
```

**Verifier**

1. 같은 hash로 r과 t를 다시 계산한다.
2. h의 commitment E를 C\_i들로부터 직접 계산한다. commitment가 덧셈 동형이므로 가능하다.
3. y\_i, z\_i, r, t로 값 y를 계산한다.
4. E − D가 점 t에서 값 y를 가지는지 π로 확인한다.

```latex
E = \sum_{i=0}^{m-1} \frac{r^i}{t - z_i}\,C_i, \qquad y = \sum_{i=0}^{m-1} r^i\,\frac{y_i}{t - z_i}, \qquad h(t) - g(t) = y
```

KZG를 쓰는 경우 마지막 확인은 pairing 등식 하나다.

```latex
e(E - D - [y]_1,\ [1]_2) = e(\pi,\ [s - t]_2)
```

### 왜 성립하는가

어느 하나라도 f\_i(z\_i) ≠ y\_i이면 그 항의 나눗셈이 나누어떨어지지 않는다. 그러면 g는 다항식이 아니므로 prover는 g에 대한 올바른 commitment를 만들 수 없다. 난수 r은 여러 항의 오류가 서로 상쇄되지 않게 하고, 난수 t에서의 확인은 D가 실제로 g의 commitment임을 보장한다.

두 난수를 hash로 얻기 때문에 prover와 verifier가 메시지를 주고받을 필요가 없다.

### Proof에 들어가는 것

- 접근한 key와 value
- 각 경로 위의 commitment (Root 제외, 경로가 겹치면 한 번만)
- multiproof 하나

z\_i와 y\_i는 보내지 않는다. 검증자가 key, value, 경로의 commitment로부터 직접 계산한다. [source](https://vitalik.eth.limo/general/2021/06/18/verkle.html)

multiproof의 크기는 key 개수와 무관하게 고정이다. 스킴에 따라 128\~1,000 B다. [source](https://vitalik.eth.limo/general/2021/06/18/verkle.html) 위 프로토콜을 KZG로 구현하면 D와 π, 즉 group element 2개다. IPA로 구현하면 D와 IPA proof 하나다. [source](https://pkg.go.dev/github.com/crate-crypto/go-ipa)

따라서 key가 늘어날 때 Proof에서 커지는 부분은 경로 위의 commitment뿐이다.

### 비용

- **검증**: 크기 N의 타원곡선 multiexponentiation 1회(E 계산)가 대부분이다. multiexponentiation은 여러 점에 각각 scalar를 곱해 모두 더하는 연산이다. 값 수천 개에 100 ms 미만으로 추정된다. [source](https://vitalik.eth.limo/general/2021/06/18/verkle.html)
- **생성**: prover는 각 f\_i 전체, 즉 관련 노드의 모든 자식 값을 가지고 있어야 한다. 그래서 전체 상태를 가진 노드만 Witness를 만들 수 있다. [source](https://dankradfeist.de/ethereum/2021/06/18/pcs-multiproofs.html)

## 8. Proof Size 비교

같은 조건에서 Verkle proof는 hexary MPT proof보다 20\~30배 이상 작고, 이상적인 Binary Merkle proof보다 6\~8배 작다. [source](https://vitalik.eth.limo/general/2021/06/18/verkle.html)

### key 1개의 경로(branch) 크기

N = 2^32, hash와 commitment를 32 B로 놓은 경우다. 식은 원문의 것이고, Verkle Tree의 128 B는 그 식에 N을 대입해 계산한 값이다. [source](https://vitalik.eth.limo/general/2024/10/23/futures4.html)

| 트리 | 식 | N = 2^32일 때 |
| --- | --- | --- |
| Hexary MPT | 32 × 15 × log\_16 N | 3,840 B |
| Binary Merkle Tree | 32 × log\_2 N | 1,024 B |
| Verkle Tree (width 256) | 32 × log\_256 N | 128 B |

### 블록 단위 최악의 경우

gas limit 30M, 접근 1회당 2,400 gas를 가정한 계산이다. 블록당 최대 12,500회 접근에 해당한다. [source](https://vitalik.eth.limo/general/2024/10/23/futures4.html)

| 트리 | 최악의 Witness | 비고 |
| --- | --- | --- |
| Hexary MPT | 약 330 MB | 코드가 트리 밖에 있어 접근마다 코드 전체(최대 24,000 B) 포함 |
| Binary Merkle Tree, 코드를 트리에 포함 | 약 10.4 MB |  |
| Verkle Tree | 약 1.3 MB | 공격자가 긴 공통 prefix를 가진 address를 만들면 약 2.6 MB |

### key 개수에 따른 Verkle proof 크기

width 256, commitment와 proof가 48 B인 KZG, key가 고르게 분포한 트리를 가정한 값이다. 단위는 바이트다. [source](https://vitalik.eth.limo/general/2021/06/18/verkle.html)

| 트리 크기 | key 1개 | key 1,000개 | key 10,000개 |
| --- | --- | --- | --- |
| 2^24 (16,777,216) | 272 | 59,792 | 457,616 |
| 2^32 (4,294,967,296) | 320 | 107,744 | 937,472 |

원문은 두 가지 보정을 덧붙인다. 실제처럼 key가 랜덤하게 분포하면 깊이가 약 0.6 늘어 key당 약 30 B가 더 든다. 32 B commitment를 쓰면 크기가 약 3분의 1 줄어든다.

key 1,000개일 때 key당 크기가 key 1개일 때보다 훨씬 작다. 상위 level의 commitment를 여러 key가 공유하고, multiproof가 하나뿐이기 때문이다.

### 값 1,000개를 검증하는 경우

| 추정 | MPT | Verkle Tree | 가정 |
| --- | --- | --- | --- |
| [ethereum.org](https://ethereum.org/roadmap/verkle-trees) | 약 3.5 MB | 약 150 kB | MPT 7 level, Verkle 4 level |
| 기존 슬라이드의 근사 | 약 2.4 MB | 약 144 kB | N = 10^6, MPT 5 level, Verkle 3 level, commitment 48 B |

슬라이드의 근사식은 (값 개수) × (깊이) × (원소 크기)다. 경로 공유를 반영하지 않으므로 상한에 가까운 값이다. key와 value 자체의 크기도 포함하지 않는다. 두 추정은 가정한 깊이가 달라 값이 다르지만 자릿수는 같다.

### 계정 1개당 평균

EIP-6800은 Witness 크기를 MPT 접근당 평균 약 3 kB, Verkle Tree 계정당 평균 약 200 B로 적는다. [EIP-6800](https://eips.ethereum.org/EIPS/eip-6800)

## 9. 한계와 현재 상태

2026년 11월 11일 기준, 이더리움 메인넷의 상태 트리는 여전히 MPT다. Verkle Tree는 테스트넷 단계까지 진행됐고, 이후 논의의 중심은 hash 기반 Binary Tree를 제안하는 EIP-7864로 옮겨 갔다.

### Verkle Tree의 한계

- **양자 내성이 없다.** commitment가 타원곡선에 기반하므로 양자 컴퓨터에 취약하다. 도입하더라도 나중에 다시 교체해야 한다.
- **SNARK 안에서 증명하기 어렵다.** Verkle proof 검증에는 256비트 연산이 많다. 이를 Succinct Non-interactive ARgument of Knowledge(SNARK)로 감싸려면 큰 오버헤드나 전용 구조가 필요하다.
- **최악의 Witness는 여전히 MB 단위다.** 8장의 표에서 약 1.3\~2.6 MB다.
- **width에 상한이 있다.** proof 생성 비용이 width에 비례한다.

위 한계를 지적한 Vitalik Buterin의 2024년 글은 Verkle Tree의 장점도 세 가지 든다. proof 생성이 빠르고, 구현이 성숙했고, Witness를 다음 블록에 맞춰 갱신할 수 있다. [source](https://vitalik.eth.limo/general/2024/10/23/futures4.html)

### 대안: STARKed Binary Hash Tree

Binary Merkle Tree를 쓰고, Merkle proof 자체를 Scalable Transparent ARgument of Knowledge(STARK)로 증명해 proof를 대체하는 방식이다. 타원곡선을 쓰지 않고 hash에만 의존한다. 아래는 2024년 10월 시점의 비교다. [source](https://vitalik.eth.limo/general/2024/10/23/futures4.html)

| 방식 | Proof 크기 | 보안 가정 | 최악의 proof 생성 시간 |
| --- | --- | --- | --- |
| Verkle Tree | 데이터 + 약 100\~2,000 kB | 타원곡선 (양자 내성 없음) | 1초 미만 |
| STARK + 보수적 hash (SHA256, BLAKE) | 데이터 + 약 100\~300 kB | 보수적 hash 함수 | 10초 초과 |
| STARK + 공격적 hash (Poseidon, Ajtai) | 데이터 + 약 100\~300 kB | 덜 검증된 새 hash 함수 | 1\~2초 |

### 진행 경과

| 시점 | 내용 |
| --- | --- |
| 2026년 3월 | Vitalik Buterin이 hexary keccak MPT를 EIP-7864의 Binary Tree로 바꾸는 안을 설명했다. [The Block](https://www.theblock.co/post/391681/vitalik-buterin-lays-out-a-two-part-plan-to-overhaul-ethereums-execution-layer-from-the-ground-up)은 Verkle Tree가 2026년 하드포크 후보로 거론됐으나, 양자 컴퓨팅 우려로 2024년 중반부터 Binary Tree가 다시 주목받았다고 보도했다. |
| 2025년 1월 | EIP-7864(unified binary tree)가 제안됐다. 제안 동기는 양자 내성 우려와 SNARK, STARK 증명 시스템의 발전이다. [source](https://ethereum-magicians.org/t/eip-7864-ethereum-state-using-a-unified-binary-tree/22611) |
| 2024년 10월 | Vitalik Buterin이 Verkle Tree와 STARKed Binary Hash Tree를 비교하고 선택이 필요하다고 적었다. [source](https://vitalik.eth.limo/general/2024/10/23/futures4.html) |
| 2023년 3월 | EIP-6800(unified Verkle tree)이 작성됐다. [EIP-6800](https://eips.ethereum.org/EIPS/eip-6800) |
| 2021년 6월 | Vitalik Buterin과 Dankrad Feist가 Verkle Tree와 multiproof 설명 글을 공개했다. |

### 확인한 상태와 확인하지 못한 것

- EIP-6800의 상태는 Stagnant다. EIP 페이지에서 직접 확인했다.
- EIP-7864의 상태는 Draft로 알려져 있다. EIP 원문 페이지는 직접 열지 못했고, 논의 스레드와 2차 자료로 확인했다.
- ethereum.org의 Verkle Tree 문서는 2026년 6월 24일에 마지막으로 갱신됐고, Binary Tree를 언급하지 않는다. 테스트넷으로 Verkle Gen Devnet 6을 든다. [source](https://ethereum.org/roadmap/verkle-trees)
- 상태 트리 변경이 어느 하드포크에 들어갈지는 확정된 자료를 찾지 못했다.

### Verkle Tree에서 이어지는 것

목표와 설계의 일부는 그대로 이어진다. Stateless Client라는 목표는 같다. EIP-7864는 계정 데이터, 스토리지 slot, 코드 chunk를 묶어 담는 방식처럼 commitment 스킴과 무관한 EIP-6800의 아이디어를 가져왔다. [source](https://ethereum-magicians.org/t/eip-7864-ethereum-state-using-a-unified-binary-tree/22611) 달라지는 것은 부모 노드를 만드는 방법과 proof를 줄이는 방법이다.

## 10. 기존 정리에서 바로잡은 내용

기존 노트 2개와 슬라이드를 원문과 대조해 찾은 항목이다. 이 문서의 본문은 모두 바로잡은 내용을 따른다.

### 내용 오류

| 위치 | 기존 내용 | 바로잡은 내용 | 관련 장 |
| --- | --- | --- | --- |
| 노트 마지막 문단 | proof 48 B 하나만 있으면 되므로 Witness가 수 KB로 줄어든다 | Witness는 값, 경로의 commitment, multiproof로 이루어진다. 값 1,000개에 약 150 kB, 블록 최악의 경우 약 2.6 MB다. 노트 앞부분의 "수백 KB"와도 맞지 않는다 | 7, 8 |
| 슬라이드 9 | 상태를 commitment로 저장하면 값을 미리 알 수 없다 | 상태 값은 공개 데이터다. Verkle Tree가 쓰는 성질은 hiding이 아니라 binding과 짧은 opening proof다 | 4 |
| 노트, 슬라이드 18 | KZG를 전제로 하고 원소 크기를 48 B로 둠 | 이더리움 제안은 Bandersnatch 곡선의 Pedersen + IPA를 쓰고 원소는 32 B다. KZG는 설명 글의 표기다. Multiproof도 KZG 전용이 아니라 덧셈 동형 스킴에 공통으로 적용된다 | 4, 7 |
| 슬라이드 13\~15, 22\~23 | 노드가 commitment와 proof π를 함께 가지고, 업데이트 때 π도 갱신한다 | 노드는 commitment만 저장한다. proof는 Witness를 만들 때 계산한다 | 6 |
| 노트, 슬라이드 23 | Lagrange Interpolation을 사용해 commitment를 갱신한다 | 갱신은 덧셈 동형성으로 한다. Lagrange는 KZG에서 기저 \[L\_i(s)\]로 등장할 뿐이고, 갱신할 때 보간을 다시 하지 않는다 | 6 |
| 슬라이드 10\~12, 배경지식 노트 | prover가 랜덤 값 r(또는 α)에서 다항식을 평가한다 | 실제 KZG에서 이 값은 아무도 모르는 비밀값이다. prover는 공개 파라미터를 조합해 commitment를 계산한다. 값을 알면 거짓 proof를 만들 수 있다 | 4 |
| 슬라이드 4, 5 | a3 위에 자식이 하나뿐인 Branch Node3가 있다 | MPT는 자식이 하나뿐인 branch node를 만들지 않는다. Root의 slot d가 leaf를 바로 가리키고, leaf의 남은 경로는 563이다 | 2 |
| 노트 | 블록 검증에 필요한 모든 것이 블록에 포함되므로 상태 저장이 필요 없다 | 검증하는 노드에만 해당한다. 블록을 만들고 Witness를 생성하는 노드는 전체 상태가 필요하다 | 1 |
| 배경지식 노트 | Finite Field는 모든 숫자가 p로 나누어지는 범위 내에 존재한다 | 원소는 p로 나눈 나머지, 즉 0부터 p − 1까지다. 노트의 GF(13) 계산 예시 4개는 모두 맞다 |  |

### 표기와 수치 불일치

| 위치 | 문제 | 처리 |
| --- | --- | --- |
| 슬라이드 5, 14 | 본문은 a1의 key를 0x7345로, 그림의 key 표와 leaf(남은 경로 40)는 0x7340으로 적음 | 0x7340으로 통일 |
| 노트와 슬라이드 10\~12 | 단순화 예시의 수치가 다름. 노트는 f(2) = 25, C = 180, 슬라이드는 f(2) = 4, C = 614 | 슬라이드 기준으로 통일 |
| 슬라이드 25\~29 | Trusted Setup의 비밀값을 KZG 장에서는 τ, Multiproof 장에서는 s로 적음 | s로 통일 |
| 슬라이드 18 | multiproof의 크기를 48 B로 더함 | 고정분은 스킴에 따라 128\~1,000 B다. 합계에 미치는 영향은 작다 |

### 시점이 지난 내용

노트와 슬라이드는 Verkle Tree를 이더리움이 도입할 구조로 서술한다. 작성 시점에는 맞는 서술이었으나, 현재는 9장의 내용으로 바뀌었다.

### 검산 결과 맞는 것

- 슬라이드 10\~12의 단순화 예시: f(x) = 20x² − 58x + 40, C = 614, π = 122, 614 − 4 = 122 × 5
- 슬라이드 18의 근사 계산: MPT 약 2.4 MB, Verkle Tree 약 144 kB. 단, 각각 깊이 5와 3으로 올림한 값이다
- 슬라이드 28\~29의 Multiproof 수식: Dankrad Feist의 원문과 같은 구조다
- "최악의 경우 수백 MB", "상태 데이터 수백 GB": Vitalik Buterin의 2024년 글과 일치한다

## Sources

직접 열어 확인한 원문

- [Vitalik Buterin, Verkle trees](https://vitalik.eth.limo/general/2021/06/18/verkle.html), 2021년 6월 18일
- [Dankrad Feist, PCS multiproofs using random evaluation](https://dankradfeist.de/ethereum/2021/06/18/pcs-multiproofs.html), 2021년 6월 18일
- [Dankrad Feist, Verkle trie for Eth1 state](https://dankradfeist.de/ethereum/2021/06/18/verkle-trie-for-eth1.html), 2021년 6월 18일
- [Dankrad Feist, Inner Product Arguments](https://dankradfeist.de/ethereum/2021/07/27/inner-product-arguments.html), 2021년 7월 27일
- [Vitalik Buterin, Possible futures of the Ethereum protocol, part 4: The Verge](https://vitalik.eth.limo/general/2024/10/23/futures4.html), 2024년 10월 23일
- [EIP-6800: Ethereum state using a unified verkle tree](https://eips.ethereum.org/EIPS/eip-6800)
- [ethereum.org, Verkle trees](https://ethereum.org/roadmap/verkle-trees), 2026년 6월 24일 갱신

검색 결과에 표시된 본문 발췌로 확인한 자료

- [Ethereum Foundation Blog, Verkle tree structure](https://blog.ethereum.org/2021/12/02/verkle-tree-structure/), 2021년 12월 2일
- [ethereum.org, Statelessness](https://ethereum.org/en/roadmap/statelessness/)
- [ethereum.org, Merkle Patricia Trie](https://ethereum.org/en/developers/docs/data-structures-and-encoding/patricia-merkle-trie)
- [Ethereum Magicians, EIP-7864 논의 스레드](https://ethereum-magicians.org/t/eip-7864-ethereum-state-using-a-unified-binary-tree/22611)
- [The Block, Vitalik Buterin lays out a two-part plan to overhaul Ethereum's execution layer](https://www.theblock.co/post/391681/vitalik-buterin-lays-out-a-two-part-plan-to-overhaul-ethereums-execution-layer-from-the-ground-up), 2026년 3월 1일
- [go-ipa 패키지 문서](https://pkg.go.dev/github.com/crate-crypto/go-ipa)
