---
title: "Verkle Tree"
date: 2026-11-11
last_modified_at: 2026-11-17
categories: [Paper Review]
tags: [Blockchain, Ethereum, Verkle Tree, Merkle Patricia Trie, Stateless Client, Commitment]
math: true
---

> 관련 글: [Multiproof]({% post_url Cryptography/2026-11-11-Multiproof %})

## What is Verkle Tree?

Verkle Tree(버클 트리)는 Merkle Tree(머클 트리)의 부모 노드를 Hash(해시) 대신 **Vector Commitment(벡터 커밋먼트)**로 만든 트리다. 이름은 Vector Commitment와 Merkle Tree를 합친 것이다. 구조는 Merkle Tree와 같지만, Value(값) 하나가 트리에 들어 있음을 보이는 Proof(증명)가 훨씬 작다. 2018년 John Kuszmaul이 [논문](https://math.mit.edu/research/highschool/primes/materials/2018/Kuszmaul.pdf)에서 처음 제안했다. 이더리움은 2021년부터 계정 데이터 전체를 담는 State Tree(상태 트리)를 Verkle Tree로 바꾸는 작업을 진행했다.

이 글은 세 가지를 순서대로 설명한다. 이더리움이 왜 State Tree를 바꾸려 했는지, 현재 구조인 Merkle Patricia Trie(MPT, 머클 패트리샤 트라이)의 Proof가 왜 큰지, Verkle Tree가 그것을 어떻게 줄이는지다. 원 논문의 설계와 이더리움 제안의 차이도 함께 적는다. 여러 증명을 하나로 묶는 Multiproof(다중 증명)는 [별도 글](/posts/multiproof/)에서 다루므로 여기서는 결과만 쓴다.

2026년 11월 17일 기준 이더리움 메인넷은 Verkle Tree를 채택하지 않았다. 로드맵의 무게는 Hash 기반 Binary Tree(이진 트리)로 옮겨 갔다. Binary Tree는 노드마다 자식이 최대 둘인 트리다. 자세한 내용은 Limitations and Current Status 장에 있다.

이 글에서 Proof와 Witness의 크기는 B(바이트), kB, MB로 적는다.

## Why Ethereum Needs a Smaller Proof

### State and Witness

이더리움의 **State(상태)**는 모든 계정의 Balance(잔액), Nonce(논스), 컨트랙트 코드, 스토리지를 합친 데이터다. Nonce는 계정이 보낸 트랜잭션의 수다. State Tree는 이 State를 담는 트리다. State 전체는 **State Root(상태 루트)**라는 값 하나로 요약되어 블록 헤더에 들어간다. 어떤 Value가 State Root가 가리키는 State에 들어 있음을 보이는 데이터가 Proof다.

클라이언트는 이더리움 프로토콜을 실행하는 소프트웨어다. 블록을 검증하는 클라이언트는 블록의 트랜잭션을 모두 다시 실행하고, 로컬 State Tree를 갱신한 뒤, 계산한 State Root가 블록에 적힌 값과 같은지 비교한다. 트랜잭션이 어떤 계정과 스토리지를 읽을지 미리 알 수 없으므로 State 전체를 로컬에 보관해야 한다. Vitalik Buterin의 [2024년 글](https://vitalik.eth.limo/general/2024/10/23/futures4.html)은 클라이언트가 저장하는 State를 수백 GB로, 증가 속도를 연 약 30 GB로 적었다. 저장 부담이 커질수록 일반 하드웨어로 검증에 참여하기 어려워진다.

이 문제를 푸는 방향이 **Stateless Client(상태 비저장 클라이언트)**다. State를 저장하지 않고, 블록과 함께 받은 **Witness(위트니스)**로 블록을 검증한다. Witness는 블록의 트랜잭션을 실행하는 데 필요한 State 조각과, 그 조각이 State Root에 포함됨을 보이는 Proof를 묶은 것이다.

### The Requirement

Witness는 블록과 함께 네트워크로 전파된다. 이더리움에서 블록 하나가 제안되는 시간 단위를 Slot(슬롯)이라 하고, 길이는 12초다. Witness는 한 Slot 안에 받아서 처리할 수 있을 만큼 작아야 한다. 현재 구조인 MPT로는 이 조건을 맞추기 어렵다.

아래 표는 MPT의 Witness가 얼마나 큰지를 두 출처의 수치로 보인 것이다. 표를 읽는 데 필요한 용어는 세 가지다. Ethereum Improvement Proposal(EIP, 이더리움 개선 제안)은 프로토콜 변경을 제안하는 문서이고 번호로 구분한다. EIP-6800은 Verkle Tree 도입을 제안한 EIP다. Gas(가스)는 연산과 State 접근에 매기는 비용 단위다. Gas Limit(가스 한도)은 블록 하나가 쓸 수 있는 Gas의 상한이다.

| 항목 | 수치 | 가정 | 출처 |
| --- | --- | --- | --- |
| 접근 1회당 Witness | 평균 약 3 kB, 최악의 경우 그 약 3배 | | [EIP-6800](https://eips.ethereum.org/EIPS/eip-6800) |
| 블록 Witness | 약 18 MB | Gas Limit 15,000,000, 접근 1회당 2,500 Gas, 블록당 접근 6,000회 | [EIP-6800](https://eips.ethereum.org/EIPS/eip-6800) |
| 블록 Witness, 최악 | 약 330 MB | Gas Limit 30,000,000, 접근 1회당 2,400 Gas, 접근마다 컨트랙트 코드 24,000 B 포함 | [Vitalik Buterin의 2024년 글](https://vitalik.eth.limo/general/2024/10/23/futures4.html) |

수치는 각 글이 쓰인 시점의 Gas Limit을 전제로 한다. Gas Limit이 바뀌면 비례해서 바뀐다.

Verkle Tree가 겨냥하는 것은 Weak Statelessness(약한 상태 비저장)다. 블록을 만드는 쪽인 블록 생성자는 Witness를 만들어야 하므로 여전히 State 전체를 가지고, 나머지 클라이언트만 State 없이 검증한다. 어떤 클라이언트도 State 전체를 갖지 않는 Strong Statelessness(강한 상태 비저장)와는 다르다. 두 개념은 [ethereum.org](https://ethereum.org/en/roadmap/statelessness/)에 정리되어 있다.

정리하면 풀어야 할 문제는 State Tree가 만드는 Proof의 크기다. Proof 크기는 트리 구조가 결정하므로 트리 구조를 바꿔야 한다. 먼저 현재 구조인 MPT의 Proof가 왜 큰지 본다.

## Merkle Patricia Trie

현재 이더리움의 State Tree는 MPT다. 설명에 앞서 트리 용어를 정한다. 맨 위 노드를 Root(루트), 자식이 없는 맨 아래 노드를 Leaf(리프)라 한다. Root에서 같은 거리에 있는 노드들의 층을 Level(레벨)이라 하고, Root에서 Leaf까지 거치는 Level의 수를 깊이라 한다.

MPT는 두 구조를 합친 것이다.

- **Merkle Tree:** 부모 노드가 자식 노드들의 Hash를 합쳐 다시 Hash한 값을 가진다. 데이터 전체가 Root의 Hash 하나로 요약된다.
- **Patricia Trie(패트리샤 트라이):** Key(키)로 Value를 찾는 트리다. Key를 한 자리씩 따라 내려가고, 분기가 없는 구간은 노드 하나로 묶어 경로를 줄인다.

MPT는 Patricia Trie의 각 노드를 Hash로 참조하는 구조다. 노드를 Recursive Length Prefix(RLP, 이더리움의 직렬화 형식)로 인코딩한 뒤 Keccak-256 Hash 함수에 넣은 결과가 그 노드의 참조가 된다. 인코딩한 결과가 32바이트보다 짧은 노드는 Hash하지 않고 부모 노드 안에 그대로 넣는다. 자세한 명세는 [ethereum.org](https://ethereum.org/en/developers/docs/data-structures-and-encoding/patricia-merkle-trie)에 있다.

### Nodes

Key는 4비트 단위인 Nibble(니블)로 나뉘어 경로가 된다. Nibble 하나는 16진수 한 자리다. 노드는 세 종류다.

| 노드 | 형태 | 역할 |
| --- | --- | --- |
| Branch Node(분기 노드) | 17개 항목 `[v0 ... v15, vt]` | Nibble 0~f에 해당하는 자식 16개와 Value 1개 |
| Extension Node(확장 노드) | 2개 항목 `[encodedPath, key]` | 분기 없는 공통 경로와 다음 노드의 참조 |
| Leaf Node(리프 노드) | 2개 항목 `[encodedPath, value]` | 남은 경로와 Value |

State Tree에서 Key는 계정의 Address(주소)를 Keccak-256으로 Hash한 값이고 경로 길이는 64 Nibble이다. Value는 계정의 Nonce, Balance, StorageRoot, CodeHash다. StorageRoot는 그 계정의 스토리지를 담은 별도 트리의 Root Hash이고, CodeHash는 컨트랙트 코드의 Hash다. 즉 컨트랙트 스토리지는 계정마다 별도의 트리에 들어가고, 코드는 트리 밖에 있으며 CodeHash만 트리에 남는다.

### Example

설명을 위해 Key를 2바이트로 줄인 예시다. 실제 Key는 32바이트다. ETH는 이더리움의 화폐 단위인 Ether의 표기다.

| 계정 | Key | Value |
| --- | --- | --- |
| a1 | 0x7340 | Nonce 2, Balance 4 ETH |
| a2 | 0x7bc6 | Nonce 3, Balance 2 ETH |
| a3 | 0xd563 | Nonce 4, Balance 5 ETH |

Root는 Branch Node다. 노드 안에서 항목의 위치를 가리키는 번호를 Index(인덱스)라 한다. a1과 a2는 첫 Nibble이 7로 같으므로 Root의 Index 7 아래로 함께 내려간다. Index 7 아래에 Branch Node가 하나 더 있고, 두 번째 Nibble이 3과 b로 갈리므로 각각 Leaf Node가 된다. a1의 Leaf Node는 남은 경로 40을, a2의 Leaf Node는 남은 경로 c6을 가진다. a3은 첫 Nibble d를 혼자 쓴다. Root의 Index d가 바로 Leaf Node를 가리키고, 이 Leaf Node는 남은 경로 563을 가진다.

### Verification

Proof를 받아 확인하는 쪽을 검증자라 한다. 검증자는 Leaf에서 Root까지 Hash를 다시 계산하고, 결과가 알고 있는 State Root와 같은지 확인한다. Branch Node의 Hash를 계산하려면 그 노드의 모든 항목이 필요하다. 따라서 경로 위의 Branch Node마다 경로에 속하지 않는 나머지 항목, 즉 **형제 노드의 Hash**가 Proof에 들어간다.

위 예시에서 a1을 검증하려면 세 가지가 필요하다. a1의 Leaf Node, 두 번째 Branch Node에서 형제인 a2 Leaf Node의 Hash, Root에서 형제인 a3 Leaf Node의 Hash다. 이 예시는 Branch Node의 항목이 거의 비어 있어 형제가 Level당 1개뿐이다. 실제 State Tree의 상위 Level은 항목이 거의 차 있으므로 Level당 최대 15개의 형제 Hash가 들어간다.

### Proof Size

트리에 저장된 Key와 Value 쌍의 개수를 $N$이라 한다. 한 노드가 가질 수 있는 자식의 최대 개수를 Width(너비)라 하고 $k$로 적는다. 깊이는 약 $\log_k N$이다. Level마다 형제 Hash가 최대 $k-1$개 필요하고, Hash 하나는 32 B다.

$$
\text{Proof Size} \approx 32\,\text{B} \times (k-1) \times \log_k N
$$

$k=16$이면 $32 \times 15 \times \log_{16} N = 120 \times \log_2 N$ B다. $N=2^{32}$일 때 3,840 B다. Width를 키우면 깊이는 줄지만 형제 수가 더 빨리 늘어난다. Hash 기반 트리에서는 Width 2가 Proof 크기에 가장 유리하고, Width 16은 불리하다. 식과 수치는 Vitalik Buterin의 [2024년 글](https://vitalik.eth.limo/general/2024/10/23/futures4.html)과 [EIP-7864](https://eips.ethereum.org/EIPS/eip-7864)의 Rationale에 있다. Vitalik Buterin의 [Verkle trees](https://vitalik.eth.limo/general/2021/06/18/verkle.html) 글은 식 없이 같은 결론을 적는다.

Value가 바뀌면 그 Leaf와, Leaf에서 Root까지 경로 위에 있는 노드의 Hash만 다시 계산한다.

## Key Idea

MPT의 Proof가 큰 이유는 Hash의 성질에 있다. Hash는 입력 전체를 알아야 다시 계산할 수 있다. 그래서 MPT의 검증자는 부모의 모든 항목을 받아야 한다.

Vector Commitment는 다르다. Vector(벡터)는 순서가 있는 값의 목록이다. Vector Commitment는 Vector $(v_0, \ldots, v_{n-1})$ 전체를 Commitment(커밋먼트)라는 짧은 값 하나로 만든다. 그리고 "이 Commitment의 Index $i$에 있는 항목은 $v_i$다"를 나머지 항목 없이 짧은 증명으로 보일 수 있다. 특정 Index의 항목을 이렇게 밝히는 것을 Opening(오프닝)이라 하고, 이때 붙이는 증명을 Opening Proof(오프닝 증명)라 한다.

부모 노드를 자식들의 Vector Commitment로 만들면, 검증자는 **경로 위의 Commitment와, 각 부모와 자식 관계에 대한 Opening Proof만** 받으면 된다. 형제 노드는 필요 없다.

형제를 보내지 않으므로 Width를 키워도 Level당 Proof 데이터가 늘지 않는다. Width를 키우면 깊이만 줄어든다. 이더리움에 제안된 Verkle Tree의 Width는 256이다. Width의 상한은 Opening Proof를 만드는 비용이 정한다. 노드마다 Width에 비례하는 연산이 들기 때문이다.

| 항목 | MPT | Verkle Tree (이더리움 제안) |
| --- | --- | --- |
| 부모 노드의 값 | 자식들의 Hash | 자식들에 대한 Vector Commitment |
| Width | 16 | 256 |
| 깊이 | 약 $\log_{16} N$ | 약 $\log_{256} N$ |
| Level당 Proof 데이터 | 형제 Hash 최대 15개 (480 B) | Commitment 1개 (32 B) |
| 여러 Key를 검증할 때 | Key마다 형제 Hash 추가 | Opening Proof 전체를 Multiproof 1개로 묶음 |

다음 장에서 이 아이디어를 처음 제시한 논문을 보고, 그 뒤에 이더리움 제안이 무엇을 바꿨는지 본다.

## The Original Paper

Verkle Tree를 처음 제시한 글은 John Kuszmaul의 [Verkle Trees](https://math.mit.edu/research/highschool/primes/materials/2018/Kuszmaul.pdf)(2018)다. Massachusetts Institute of Technology(MIT)의 PRIMES 프로그램에서 나온 12쪽 논문이고, 아이디어는 멘토 Alin Tomescu가 제공했다고 적혀 있다.

논문의 내용은 다음과 같다. $n$은 Leaf의 수이고 $k$는 Width다. Binary Merkle Tree는 Width가 2인 Merkle Tree이고, $k$-ary Merkle Tree는 Width가 $k$인 Merkle Tree다.

- **문제:** Binary Merkle Tree의 Proof 크기는 $O(\log_2 n)$이다. Leaf가 $2^{30}$개이고 Hash가 256비트면 약 1 kB다. $k$-ary Merkle Tree는 Level마다 형제 $k-1$개가 필요해 $O(k \log_k n)$으로 오히려 커진다.
- **설계:** Merkle Tree와 같은 방식으로 트리를 만들되 Hash 대신 Vector Commitment를 쓴다. Opening Proof는 크기가 고정이므로 Proof 크기는 $O(\log_k n)$이다. Binary Merkle Tree보다 $\log_2 k$배 작다. $k=1024$면 10배, $k=32$면 5배다.
- **비용:** 노드를 만들 때 자식마다 Opening Proof를 미리 계산한다. 그래서 트리 구성에 $O(kn)$, Value 하나 갱신에 $O(k \log_k n)$이 든다. $k$는 Proof 크기와 계산량을 맞바꾸는 값이다.
- **구현:** Catalano와 Fiore의 Vector Commitment를 썼다. Bilinear Group(쌍선형 군)과 Computational Diffie-Hellman이라는 암호학적 가정에 기반한다. Leaf가 $2^{14}$개인 트리에서 Leaf 1개당 구성 시간은 $k=1024$일 때 110.1 ms, $k=32$일 때 8.4 ms였다.

아래는 논문이 정리한 복잡도 표다.

| 구조 | 트리 구성 | Value 하나 갱신 | Proof 크기 |
| --- | --- | --- | --- |
| Merkle Tree | $O(n)$ | $O(\log_2 n)$ | $O(\log_2 n)$ |
| $k$-ary Merkle Tree | $O(n)$ | $O(k \log_k n)$ | $O(k \log_k n)$ |
| Vector Commitment | $O(n^2)$ | $O(n)$ | $O(1)$ |
| $k$-ary Verkle Tree | $O(kn)$ | $O(k \log_k n)$ | $O(\log_k n)$ |

이더리움 제안은 논문의 구조에서 세 가지를 바꿨다.

- Vector Commitment가 다르다. 다음 장에서 설명하는 Pedersen Commitment를 쓴다.
- Opening Proof를 노드에 저장하지 않는다. 블록이 접근한 Value에 대해서만 그때 계산한다. 그래서 Value 하나를 갱신하는 비용이 Width에 비례하지 않는다.
- 여러 Opening Proof를 Multiproof 하나로 묶는다. 논문은 Opening Proof를 묶는 방법을 다루지 않는다.

논문의 비교 대상은 Binary Merkle Tree다. 이더리움의 현재 트리는 Width가 16이어서 줄어드는 폭이 더 크다.

## Commitment Scheme

이 장은 이더리움 제안이 쓰는 Vector Commitment를 설명한다.

Verkle Tree가 Commitment에서 쓰는 성질은 두 가지다. 하나는 Binding(구속성)이다. 한 번 만든 Commitment를 다른 Vector의 Commitment인 것처럼 Opening할 수 없다는 뜻이다. 다른 하나는 특정 Index의 항목을 짧은 Opening Proof로 Opening할 수 있다는 점이다. 이더리움의 State는 공개 데이터이므로 값을 감추는 Hiding(은닉성)은 목적이 아니다. 실제로 EIP-6800의 Commitment 식에는 값을 감추기 위한 난수 항이 없다.

### Vector Commitment from Polynomial Commitment

Vector Commitment를 만드는 한 방법이 Polynomial Commitment(다항식 커밋먼트)다. 항목 $v_0, \ldots, v_{n-1}$에 대해 $f(i)=v_i$를 만족하는 다항식을 Lagrange Interpolation(라그랑주 보간)으로 만든다. Lagrange Interpolation은 주어진 점들을 지나는 다항식을 구하는 방법이다. 그러면 "Index $i$의 항목이 $v_i$다"는 "$f(i)=v_i$다"와 같은 말이 된다. 다항식의 Commitment를 만들고 특정 점에서 Opening하는 방법은 [Multiproof 글](/posts/multiproof/)의 앞부분에 있다.

### Pedersen Commitment

이더리움 제안이 쓰는 Vector Commitment는 Pedersen Commitment(페더슨 커밋먼트)다. 타원곡선 위의 점은 서로 더할 수 있고 정수를 곱할 수 있다. 곱하는 정수를 Scalar(스칼라)라 한다. Pedersen Commitment는 미리 정해 둔 타원곡선 점 $G_0, \ldots, G_{255}$에 항목 $v_i$를 각각 곱해 모두 더한 점이다. 이 점들을 기저점이라 한다.

$$
C = v_0 G_0 + v_1 G_1 + \cdots + v_{255} G_{255}
$$

어떤 기저점이 다른 기저점의 몇 배인지를 Discrete Log(이산 로그) 관계라 한다. 이 관계가 알려져 있지 않으면 Binding이 성립한다. 비밀 값으로 공개 매개변수를 만드는 준비 절차인 Trusted Setup(신뢰 설정)은 필요 없다. Opening Proof에는 Inner Product Argument(IPA, 내적 증명)를 쓴다.

이 Commitment는 Additively Homomorphic(덧셈 동형)이다. 두 Commitment를 더하면 두 Vector의 합에 대한 Commitment가 된다. 이 성질이 트리의 갱신과 Multiproof를 가능하게 한다.

### What Ethereum Chose

- Commitment: Bandersnatch라는 타원곡선 위의 Pedersen Commitment, 기저점 256개. [EIP-6800](https://eips.ethereum.org/EIPS/eip-6800)
- Opening Proof: IPA. [Ethereum Foundation 블로그](https://blog.ethereum.org/2021/12/02/verkle-tree-structure/)
- 한 항목에 넣을 수 있는 값: 252비트까지. Scalar가 가질 수 있는 값의 집합을 Scalar Field(스칼라 체)라 하는데, Bandersnatch의 Scalar Field는 크기가 253비트 소수다.
- Commitment 크기: 32 B

다른 자료를 읽을 때는 Commitment 방식이 다를 수 있다. Dankrad Feist의 설명 글은 Kate-Zaverucha-Goldberg(KZG) Commitment를 기준으로 쓰였다. Vitalik Buterin의 글은 KZG와 Bulletproof 방식을 함께 언급하고 수치는 KZG 기준으로 든다. Bulletproof 방식은 IPA를 쓰는 방식이고, 이더리움 제안의 Pedersen Commitment와 IPA가 여기에 해당한다.

두 방식의 차이는 다음과 같다. KZG는 Trusted Setup과 Pairing(페어링)이라는 타원곡선 연산이 필요하지만 Opening Proof가 군 원소 하나다. 군 원소는 여기서 타원곡선 점 하나를 뜻한다. Pedersen Commitment와 IPA는 Trusted Setup이 필요 없다. 대신 Opening Proof가 $2\log_2 256 = 16$개의 군 원소와 Scalar 1개이고, 검증 비용이 Width에 비례한다. Multiproof로 Opening Proof를 하나로 묶어 한 번만 검증하므로 이 비용은 감당할 수 있다. 두 방식의 비교는 Dankrad Feist의 [Inner Product Arguments](https://dankradfeist.de/ethereum/2021/07/27/inner-product-arguments.html)에 있다. Scalar 1개는 Ignacio Hagopian의 [Anatomy of a Verkle proof](https://ihagopian.com/posts/anatomy-of-a-verkle-proof)가 적은 Witness 형식에 있다.

## Structure

이 장은 Verkle Tree에 State를 어떻게 넣는지 설명한다. 이더리움 제안은 계정, 스토리지, 코드를 모두 하나의 트리에 넣는다. Key와 Value는 모두 32바이트다.

이 장은 2026년 11월 17일에 확인한 [EIP-6800](https://eips.ethereum.org/EIPS/eip-6800) 본문을 기준으로 한다. 이 EIP는 개정된 적이 있어, 이전 시점의 자료는 배치가 다를 수 있다. 예를 들어 2024년 1월의 Anatomy of a Verkle proof는 Balance와 Nonce를 서로 다른 Suffix에 두는 이전 배치로 설명한다.

### Key: Stem and Suffix

Key 32바이트는 앞 31바이트인 Stem(스템)과 마지막 1바이트인 Suffix(서픽스)로 나뉜다.

- **Stem:** 트리 안에서의 경로를 정한다. Address와 Tree Index를 이어 붙여 Pedersen Hash에 넣은 결과의 앞 31바이트다. Tree Index는 한 계정의 Value를 256개씩 묶은 그룹의 번호다. Pedersen Hash는 Pedersen Commitment를 Hash 함수처럼 쓴 것이다.
- **Suffix:** 같은 Stem 아래에 있는 Value 256개 중 하나를 고른다.

Address 자체가 경로가 되는 것이 아니다. Stem이 Hash의 결과이므로 Key는 트리 전체에 고르게 퍼진다.

### Values Under One Stem

계정의 첫 번째 Stem(Tree Index 0)에는 자주 함께 읽는 Value가 모여 있다. 표에 나오는 용어는 두 가지다. Storage Slot(스토리지 슬롯)은 컨트랙트 스토리지에서 Value 하나가 들어가는 자리이고 번호로 구분한다. Chunk(청크)는 코드를 일정한 길이로 자른 조각이다.

| Suffix | 내용 | 상수 |
| --- | --- | --- |
| 0 | Version, Code Size, Nonce, Balance를 Value 하나에 묶음 | BASIC_DATA_LEAF_KEY = 0 |
| 1 | Code Hash | CODE_HASH_LEAF_KEY = 1 |
| 64~127 | Storage Slot 0~63 | HEADER_STORAGE_OFFSET = 64 |
| 128~255 | 코드의 앞쪽 Chunk 128개 | CODE_OFFSET = 128 |

나머지 Storage Slot과 Chunk는 다른 Tree Index의 Stem으로 간다. 코드는 31바이트씩 나뉘고, 앞에 1바이트를 붙인 32바이트 Chunk로 트리에 들어간다. PUSH는 뒤따르는 바이트를 데이터로 읽는 명령이다. 앞 1바이트는 그 Chunk의 첫머리가 PUSH의 데이터일 때 그 데이터의 바이트 수다. MPT에서는 코드가 트리 밖에 있어, 코드 접근을 증명하려면 코드 전체를 Witness에 넣어야 했다.

같은 Stem에 속한 Value는 경로를 공유한다. Nonce와 Balance, 앞쪽 Storage Slot을 함께 읽어도 경로 위의 Commitment는 한 번만 Witness에 들어간다. EIP-6800은 이를 Gas 비용에도 반영한다. 같은 그룹 안에서 인접한 Storage Slot에 접근하는 비용을 2,100 Gas에서 200 Gas로 낮춘다. 세부 규칙은 [EIP-4762](https://eips.ethereum.org/EIPS/eip-4762)가 정한다.

### Nodes

Verkle Tree의 노드는 두 종류다.

| 노드 | Commitment의 대상 | 역할 |
| --- | --- | --- |
| Inner Node(내부 노드) | 자식 256개의 Commitment | Stem을 1바이트씩 따라 분기. MPT의 Branch Node에 해당 |
| Extension Node | (1, Stem, C1, C2) | 하나의 Stem과 그 아래 Value 256개를 대표 |

Extension Node는 MPT의 Extension Node와 이름이 같지만 역할이 다르다. MPT에서는 분기 없는 경로를 잇는 노드이고, 여기서는 Stem 하나에 속한 Value 256개를 담는 노드다. 이 글에서 Verkle Tree의 Leaf는 Extension Node에 담긴 Value 하나를 가리킨다.

Extension Node는 Value를 두 Commitment로 나눠 담는다. C1은 Suffix 0~127의 Value에 대한 Commitment이고, C2는 Suffix 128~255의 Value에 대한 Commitment다. 나누는 이유는 항목의 크기다. Commitment의 한 항목에는 252비트까지만 들어가는데 Value는 32바이트(256비트)다. 그래서 Value 하나를 16바이트 두 조각으로 나눠 항목 두 개에 넣는다. Value 128개가 항목 256개를 채우므로 Commitment가 두 개 필요하다. 아래쪽 조각에는 $2^{128}$을 더한다. Value가 0인 경우와 Value가 없는 경우를 구분하기 위해서다.

부모가 자식 Commitment를 담을 때도 같은 제약이 있다. Commitment는 타원곡선 위의 점이므로, Scalar로 변환한 뒤 부모의 항목에 넣는다.

### Depth

무작위 Key $n$개를 Width $d$인 트리에 넣으면 평균 깊이는 약 $\log_d n + 2/3$이다. 여기서 깊이는 경로 위에 있는 Inner Node의 수다. Dankrad Feist가 [Verkle trie for Eth1 state](https://dankradfeist.de/ethereum/2021/06/18/verkle-trie-for-eth1.html)에서 수치 실험으로 얻은 값이고, 유도는 실려 있지 않다. $n=2^{30}$, $d=2^{10}$이면 약 3.67이다.

EIP-6800의 구조에서는 Inner Node의 경로를 Stem이 정한다. 따라서 이 식을 적용할 때 $n$은 Stem의 개수로 보는 것이 맞다. 이 문장은 출처에 적힌 내용이 아니라 구조에서 읽은 해석이다.

### Example

MPT 예시를 Width 256으로 옮기면 다음과 같다. 설명을 위해 Key의 첫 바이트를 Stem으로, 둘째 바이트를 Suffix로 쓴다. 실제 Stem은 Hash로 만든 31바이트다.

- Root는 항목 256개를 가진 Inner Node다.
- a1(0x7340)은 Root의 Index 0x73, a2(0x7bc6)는 Index 0x7b, a3(0xd563)은 Index 0xd5로 간다.
- 세 Key가 첫 바이트에서 모두 갈리므로 Root 바로 아래에 각 Key의 Extension Node가 온다. Inner Node는 Root 하나뿐이다.

MPT에서는 a1과 a2가 첫 Nibble 7을 공유해 Level이 하나 더 필요했다. Width가 넓어지면 같은 Key 집합에서 깊이가 줄어든다.

## Updating the Tree

이 장은 Value가 바뀔 때 트리를 어떻게 갱신하는지 설명한다. Value가 바뀌면 그 Value에서 Root까지 경로 위의 Commitment만 갱신한다. 각 Commitment는 처음부터 다시 계산하지 않고 바뀐 분량만 더한다. Pedersen Commitment가 Additively Homomorphic이기 때문이다.

Commitment는 각 항목에 기저점을 곱해 더한 점이다. Index $i$의 항목이 $v_i$에서 $v_i'$로 바뀌면 새 Commitment는 다음과 같다.

$$
C' = C + (v_i' - v_i)\,G_i
$$

나머지 255개 항목은 읽을 필요가 없다. 타원곡선 점에 Scalar를 곱하는 연산 1회와 점의 덧셈 1회로 끝난다. 아래에서는 이 곱하는 연산을 타원곡선 곱셈이라 부른다.

경로를 따라 올라가는 순서는 다음과 같다.

1. Value가 바뀌면 Extension Node의 C1 또는 C2를 위 식으로 갱신한다. Value는 두 조각으로 나뉘어 항목 두 개에 들어 있으므로 바뀐 조각마다 한 번씩 적용한다.
2. C1 또는 C2가 바뀌었으므로 Extension Node의 Commitment를 같은 식으로 갱신한다.
3. 부모 Inner Node는 자식 Commitment를 Scalar로 바꾼 값을 항목으로 담고 있다. 새 Scalar와 이전 Scalar의 차이를 구해 부모를 같은 식으로 갱신한다.
4. Root까지 3을 반복한다.

바뀐 항목 하나마다 타원곡선 곱셈이 1회 든다. Value 하나를 고치면 C1 또는 C2에서 최대 2회, Extension Node에서 1회, 경로 위 Inner Node마다 1회다. Inner Node가 3개면 최대 6회다. 이 횟수는 출처에 적힌 수가 아니라 위 구조에서 센 값이다.

Vitalik Buterin의 [Verkle trees](https://vitalik.eth.limo/general/2021/06/18/verkle.html) 글은 약 4회로 적는다. 그 글은 Value가 Inner Node에 바로 달린 단순한 트리를 전제로 하고, Root를 포함한 Commitment의 수를 센 것이다. 같은 글에 따르면 기저점의 배수를 미리 계산해 두면 더 빨라진다.

예를 들어 계정 a1~a5가 있는 트리에서 a1이 a4에게 3 ETH를 보내면, a1의 Nonce와 Balance, a4의 Balance가 바뀐다. 수수료를 받는 계정의 변경은 생략한다. 두 Value에서 Root까지의 Commitment가 갱신되고, 두 경로가 만나는 지점부터 Root까지는 두 변경을 한 번에 반영한다. a2, a3, a5의 Value와 그 위의 다른 가지는 그대로다.

트리의 노드가 저장하는 것은 Commitment다. Opening Proof는 노드에 저장하지 않고, 블록이 접근한 Value에 대해 Witness를 만들 때 계산한다. 노드마다 Opening Proof를 미리 계산해 두는 Kuszmaul의 설계와 다른 점이다.

## Verification

이 장은 검증자가 Witness로 Value를 확인하는 과정을 설명한다. 검증자는 Root의 Commitment를 이미 알고 있다. 경로의 각 Level에서 다음 관계를 확인한다.

- 부모 Commitment를 Index $z$에서 Opening하면 항목이 $y$다.
- $z$는 Key가 정하는 자식의 위치다.
- $y$는 자식 Commitment를 Scalar로 바꾼 값이다. 마지막 Level에서는 저장된 Value의 조각이다.

Structure 장의 예시에서 a1을 검증한다면 Opening이 여섯 개 필요하다. Anatomy of a Verkle proof가 나열하는 Opening 구성을 이 예시에 적용한 수다.

- Root의 Commitment를 Index 0x73에서 Opening해 a1의 Extension Node Commitment를 확인한다. 1개다.
- Extension Node의 Commitment를 Index 0, 1, 2에서 Opening해 Extension 표시 값 1, Stem, C1을 확인한다. 3개다. a1의 Suffix 0x40은 10진수로 64이므로 Value는 C1에 들어 있다.
- C1을 Index 128과 129에서 Opening해 Value의 아래쪽 조각과 위쪽 조각을 확인한다. 2개다.

a2와 a3의 Commitment는 필요하지 않다.

블록 하나는 수천 개의 Value에 접근하므로 Opening이 대략 (Key 개수) × (Key당 Opening 수)만큼 생긴다. 여러 Key가 공유하는 Opening은 한 번만 센다. Opening Proof를 따로 만들면 Proof 데이터가 그만큼 커진다. Multiproof는 이 Opening Proof들을 **크기가 고정된 증명 하나**로 묶는다. 원리는 Dankrad Feist의 [PCS multiproofs using random evaluation](https://dankradfeist.de/ethereum/2021/06/18/pcs-multiproofs.html)과 [Multiproof 글](/posts/multiproof/)에 있다.

Multiproof를 적용한 Witness는 다음으로 이루어진다.

- 접근한 Key와 Value
- 각 경로 위의 Commitment. Root는 제외하고, 경로가 겹치면 한 번만 넣는다.
- Multiproof 하나

Anatomy of a Verkle proof가 적은 Witness 형식에는 Stem마다 깊이와, 그 Stem의 Extension Node가 어떤 경우인지를 나타내는 값이 더 들어간다. 경우는 비어 있음, 다른 Stem이 있음, 있음의 세 가지다. 트리에 없는 Key를 증명할 때 쓰는 다른 Stem도 들어간다.

$z$와 $y$는 보내지 않는다. 검증자가 Key, Value, 경로의 Commitment로부터 직접 계산한다.

Multiproof의 크기는 Key 개수와 무관하게 고정이다. Vitalik Buterin의 Verkle trees 글은 방식에 따라 128~1,000 B로 적는다. 위 Witness 형식에서 Multiproof는 Commitment 1개와 IPA의 Opening Proof(군 원소 16개, Scalar 1개)이고 원소마다 32 B이므로 576 B다. 이 형식은 제안 단계의 것이어서 바뀔 수 있다. 따라서 Key가 늘어날 때 Proof에서 커지는 부분은 경로 위의 Commitment뿐이다.

검증 비용은 Vitalik Buterin의 Verkle trees 글에 있다. Value $m$개를 증명할 때 검증 비용은 크기 $m$의 Multiexponentiation(다중 지수 연산) 1회와 약 $4m$회의 Field(체) 연산이 대부분이다. Multiexponentiation은 여러 타원곡선 점에 각각 Scalar를 곱해 모두 더하는 연산이다. Field 연산은 큰 소수를 법으로 하는 덧셈과 곱셈이다. $m$은 트리 크기 $N$이 아니라 증명하는 Value의 수다. 같은 글은 Value 수천 개의 검증이 100 ms 안에 끝난다고 적는다. 이 수치는 2021년 글의 추정이고 측정 환경은 적혀 있지 않다.

Opening Proof를 만드는 쪽은 경로 위 각 노드의 항목 256개가 모두 필요하다. 그래서 이더리움 설계에서는 State 전체를 가진 블록 생성자가 Witness를 만든다.

## Proof Size

이 장은 Proof가 얼마나 줄어드는지를 수치로 본다. Vitalik Buterin의 Verkle trees 글은 같은 조건에서 Verkle Tree의 Proof가 Hexary MPT의 Proof보다 20~30배 이상 작고, 이상적인 Binary Merkle Tree의 Proof보다 6~8배 작다고 적는다. Hexary는 Width가 16이라는 뜻이다.

### One Key

$N=2^{32}$, Hash와 Commitment를 32 B로 놓은 경우다. 식은 Vitalik Buterin의 [2024년 글](https://vitalik.eth.limo/general/2024/10/23/futures4.html)에 있다. Hexary MPT와 Binary Merkle Tree의 값은 글에 적혀 있고, Verkle Tree의 128 B는 식에 $N$을 대입해 계산한 값이다.

| 트리 | 식 | $N=2^{32}$일 때 |
| --- | --- | --- |
| Hexary MPT | $32 \times 15 \times \log_{16} N$ | 3,840 B |
| Binary Merkle Tree | $32 \times \log_2 N$ | 1,024 B |
| Verkle Tree (Width 256) | $32 \times \log_{256} N$ | 128 B |

세 값 모두 경로 하나의 크기다. Verkle Tree의 128 B에는 Multiproof가 들어 있지 않다.

### Worst Case per Block

같은 2024년 글이 든 계산이다. Gas Limit 30,000,000, 접근 1회당 2,400 Gas를 가정한다. 블록당 최대 12,500회 접근에 해당한다. 표에서 Prefix(접두부)는 Key의 앞부분이다.

| 트리 | 최악의 Witness | 비고 |
| --- | --- | --- |
| Hexary MPT | 약 330 MB | 코드가 트리 밖에 있어 접근마다 코드 전체(최대 24,000 B) 포함 |
| Binary Merkle Tree, 코드를 트리에 포함 | 약 10.4 MB | |
| Verkle Tree | 약 1.3 MB | 공격자가 긴 공통 Prefix를 가진 Address를 만들면 약 2.6 MB |
| Verkle Tree, 인접 접근만 한 경우 | 약 4.8 MB | 인접 접근 1회당 200 Gas, 접근마다 32 B를 가정 |

마지막 행의 식은 30,000,000 / 200 × 32다. 글에는 4,800,800 B로 적혀 있으나 식을 계산하면 4,800,000 B다.

### Many Keys

여러 Key를 함께 증명할 때의 Proof 크기다. 출처는 Vitalik Buterin의 [Verkle trees](https://vitalik.eth.limo/general/2021/06/18/verkle.html) 글이다. Width 256, Commitment와 Opening Proof가 48 B인 KZG, Key가 고르게 분포한 트리를 가정한다. 단위는 B다.

| 트리 크기 | Key 1개 | Key 1,000개 | Key 10,000개 |
| --- | --- | --- | --- |
| $2^{24}$ | 272 | 59,792 | 457,616 |
| $2^{32}$ | 320 | 107,744 | 937,472 |

이 표를 읽을 때 두 가지를 구분해야 한다.

- 표의 값은 Key와 Value 자체를 포함하지 않는다. 글의 표에서 트리 크기가 256일 때의 값은 176 B다. 이 값은 글이 말하는 Multiproof 구성(Commitment 1개, 수 1개, 증명 2개)을 48 × 3 + 32로 계산한 값과 같다.
- 현재 글의 표에는 열 제목이 보이지 않는다. 위 세 열을 Key 1개, 1,000개, 10,000개로 본 것은 계산에 따른 해석이다. 같은 가정으로 계산하면 표의 값과 0.2% 이내로 맞는다.

$2^{32}$ 크기의 트리에서 Key 1,000개일 때 Key당 크기는 약 108 B다. Key 1개일 때의 320 B보다 훨씬 작다. 상위 Level의 Commitment를 여러 Key가 공유하고, Multiproof가 하나뿐이기 때문이다. 글은 두 가지 보정을 덧붙인다. 실제처럼 Key가 무작위로 분포하면 깊이가 약 0.6 늘어 Key당 약 30 B가 더 든다. 32 B Commitment를 쓰면 크기가 약 3분의 1 줄어든다.

[ethereum.org](https://ethereum.org/roadmap/verkle-trees)는 Value 1,000개 기준으로 MPT의 Witness 약 3.5 MB(7 Level), Verkle Tree의 Witness 약 150 kB(4 Level)를 들고 약 23배 작다고 적는다. Leaf 1개당 7 × 15 × 32 = 3,360 B와 4 × 32 = 128 B로 계산하면 이 수치와 가깝다. 이 계산은 ethereum.org에 적힌 것이 아니라 이 글에서 한 것이다. [EIP-6800](https://eips.ethereum.org/EIPS/eip-6800)은 MPT 접근당 평균 약 3 kB, Verkle Tree 계정당 평균 약 200 B로 적는다.

## Limitations and Current Status

2026년 11월 17일 기준, 이더리움 메인넷의 State Tree는 여전히 MPT다. Verkle Tree는 테스트넷 단계까지 진행됐다. 이후 논의의 중심은 Hash 기반 Binary Tree를 제안하는 EIP-7864로 옮겨 갔다. 이 장은 그 이유와 경과를 적는다.

### Limitations

아래 한계는 Vitalik Buterin의 [2024년 글](https://vitalik.eth.limo/general/2024/10/23/futures4.html)이 든 것이다.

- **Quantum Resistance(양자 내성)가 없다.** Quantum Resistance는 양자 컴퓨터로도 깨지지 않는 성질이다. Commitment가 타원곡선에 기반하므로 양자 컴퓨터에 취약하다. 도입하더라도 나중에 다시 교체해야 한다.
- **SNARK 안에서 증명하기 어렵다.** Succinct Non-interactive Argument of Knowledge(SNARK)는 계산이 올바르게 수행됐음을 짧은 증명으로 보이는 방식이다. Verkle Tree의 Proof를 검증하는 계산에는 256비트 연산이 많다. 이 계산을 SNARK로 증명하려면 큰 오버헤드나 전용 구조가 필요하다.
- **최악의 Witness는 여전히 MB 단위다.** Proof Size 장의 표에서 약 1.3~4.8 MB다.
- **Width에 상한이 있다.** Opening Proof를 만드는 비용이 Width에 비례한다.
- **State Root 갱신이 Hash 기반 트리보다 느리다.** State 전체를 저장하는 클라이언트가 체인을 따라잡는 Sync(동기화) 시간이 길어진다.

같은 글은 Verkle Tree의 장점도 세 가지 든다. Proof 생성이 빠르고, 구현이 성숙했고, Witness를 다음 블록에 맞춰 갱신할 수 있다.

### Alternative: STARKed Binary Hash Tree

Binary Merkle Tree를 쓰고, Merkle Tree의 Proof를 Scalable Transparent Argument of Knowledge(STARK)로 증명해 Proof를 대체하는 방식이다. STARK는 SNARK와 같은 목적의 증명 방식이다. 이 방식은 타원곡선을 쓰지 않고 Hash 함수에만 의존한다.

아래는 2024년 10월 시점의 비교다. 보수적 Hash는 오래 쓰여 검증된 Hash 함수이고, 공격적 Hash는 증명에 유리하도록 새로 설계되어 검증이 덜 된 Hash 함수다.

| 방식 | Proof 크기 | 보안 가정 | 최악의 증명 생성 시간 |
| --- | --- | --- | --- |
| Verkle Tree | 데이터 + 약 100~2,000 kB | 타원곡선 (Quantum Resistance 없음) | 1초 미만 |
| STARK + 보수적 Hash (SHA256, BLAKE) | 데이터 + 약 100~300 kB | 보수적 Hash 함수 | 10초 초과 |
| STARK + 공격적 Hash (Poseidon, Ajtai) | 데이터 + 약 100~300 kB | 덜 검증된 새 Hash 함수 | 1~2초 |

현재 글의 표에는 열 제목이 보이지 않는다. 네 번째 열을 최악의 증명 생성 시간으로 본 것은 본문의 계산에 맞춘 해석이다. 본문은 최악의 경우 약 660,000번의 Hash를 증명해야 하고, 보수적 Hash는 초당 약 10,000~30,000번을 증명할 수 있다고 적는다.

### Timeline

Hard Fork(하드포크)는 프로토콜 규칙을 바꾸는 네트워크 업그레이드다.

| 시점 | 내용 |
| --- | --- |
| 2021년 6월 | Vitalik Buterin과 Dankrad Feist가 Verkle Tree와 Multiproof 설명 글을 공개했다. |
| 2021년 12월 | [Ethereum Foundation 블로그](https://blog.ethereum.org/2021/12/02/verkle-tree-structure/)가 EIP 초안의 트리 구조를 설명했다. |
| 2023년 3월 | 이 설계가 [EIP-6800](https://eips.ethereum.org/EIPS/eip-6800)으로 등록됐다. |
| 2024년 10월 | Vitalik Buterin이 Verkle Tree와 STARKed Binary Hash Tree를 비교하고 선택이 필요하다고 적었다. |
| 2025년 1월 | EIP-7864(Unified Binary Tree)가 제안됐다. 제안 동기는 Quantum Resistance 우려와 SNARK, STARK 증명 시스템의 발전이다. [논의 스레드](https://ethereum-magicians.org/t/eip-7864-ethereum-state-using-a-unified-binary-tree/22611) |
| 2026년 3월 | Vitalik Buterin이 Hexary Keccak MPT를 EIP-7864의 Binary Tree로 바꾸는 안을 설명했다. [The Block](https://www.theblock.co/post/391681/vitalik-buterin-lays-out-a-two-part-plan-to-overhaul-ethereums-execution-layer-from-the-ground-up)은 Verkle Tree가 2026년 Hard Fork 후보로 거론됐으나, 양자 컴퓨팅 우려로 2024년 중반부터 Binary Tree가 다시 주목받았다고 보도했다. |
| 2026년 8월 | Vitalik Buterin이 2023년 로드맵을 Ethereum Foundation의 로드맵 초안인 Strawmap과 비교했다. [The Defiant](https://thedefiant.io/newsletter/defi-daily/ethereum-bets-its-technical-future-on-privacy-and-quantum-defense)는 Verkle Tree 자리에 EIP-7864의 Unified Binary Tree가 들어갔다고 보도했다. |
| 2026년 10월 | Glamsterdam 업그레이드가 Sepolia 테스트넷에서 활성화됐다. [공지](https://blog.ethereum.org/2026/10/16/glamsterdam-hoodi-announcement)의 EIP 목록에 State Tree 변경은 없고, 메인넷 활성화 날짜는 정해지지 않았다. |

### What Was Confirmed

아래는 2026년 11월 17일에 각 출처에서 확인한 내용이다.

- EIP-6800은 Stagnant 단계로 표시되어 있다. Stagnant는 오랫동안 갱신되지 않은 EIP에 붙는 단계다.
- [EIP-7864](https://eips.ethereum.org/EIPS/eip-7864)는 Draft(초안) 단계로 표시되어 있다. Hash 함수는 확정되지 않았다. 참조 구현은 BLAKE3라는 Hash 함수를 쓰고, Keccak과 Poseidon2가 후보다.
- [ethereum.org](https://ethereum.org/roadmap/verkle-trees)의 Verkle Tree 문서는 2026년 6월 24일에 마지막으로 갱신됐고, Binary Tree를 언급하지 않는다. 테스트넷으로 Verkle Gen Devnet 6을 든다. Verkle Tree를 여전히 Stateless Client로 가는 단계로 설명한다.
- 같은 문서의 두 설명은 정확하지 않다. MPT를 Leaf를 둘씩 묶어 Hash하는 트리로 설명하지만 MPT는 Width가 16이다. Witness의 크기가 Leaf 개수와 무관하게 고정이라고 적지만 고정인 것은 Multiproof의 크기다.
- State Tree 변경이 어느 Hard Fork에 들어갈지 확정한 자료는 찾지 못했다. 다음 업그레이드인 Glamsterdam의 EIP 목록에는 없다.
- 2026년 8월 Vitalik Buterin의 게시물은 원문을 확인하지 못했다. Timeline의 해당 내용은 보도에 근거한다.

Stateless Client라는 목표와 설계의 일부는 그대로 이어진다. EIP-7864는 계정 데이터, Storage Slot, Chunk를 묶어 담는 방식처럼 Commitment 방식과 무관한 EIP-6800의 아이디어를 가져왔다. 달라지는 것은 부모 노드를 만드는 방법과 Proof를 줄이는 방법이다. 하나의 Stem에 함께 담는 Storage Slot과 Chunk의 개수는 아직 고정되지 않았다. 현재 Draft는 상수 표에 4개와 16개를, 설명 문단에 64개와 128개를 적고 있다.

## Summary

1. Stateless Client가 되려면 Witness가 작아야 하고, Witness 크기는 State Tree의 Proof 크기가 정한다.
2. MPT는 Level마다 형제 Hash를 보내야 하므로 Proof가 (Width − 1) × 깊이에 비례한다.
3. Verkle Tree는 부모를 Vector Commitment로 만들어 형제를 보내지 않는다. 이 구조는 Kuszmaul의 논문이 제시했다. 이더리움 제안은 Width를 256으로 키워 깊이를 줄이고, 여러 Opening Proof를 Multiproof 하나로 묶는다.
4. 그 결과 Key 1개의 경로 크기가 3,840 B에서 128 B로, Value 1,000개의 Witness가 약 3.5 MB에서 약 150 kB로 줄어든다.
5. 타원곡선에 의존하므로 Quantum Resistance가 없고, 이더리움 로드맵은 Hash 기반 Binary Tree로 방향을 바꿨다. EIP-7864는 아직 Draft이고 적용할 Hard Fork는 정해지지 않았다.