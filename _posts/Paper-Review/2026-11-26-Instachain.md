---
title: "Instachain: Breaking the Sharding Limits via Adjustable Quorums"
date: 2026-11-26 02:00:00 +0900
last_modified_at: 2026-11-26
categories: [Paper Review]
tags: [Blockchain, Sharding, Consensus]
math: true
---

[https://eprint.iacr.org/2022/413](https://eprint.iacr.org/2022/413)

### Paper Information

| Item | Details |
| --- | --- |
| Title | Instachain: Breaking the Sharding Limits via Adjustable Quorums |
| Authors | Mustafa Safa Ozdayi (The University of Texas at Dallas), Yue Guo (J.P. Morgan Research), Mahdi Zamani (Visa Research) |
| Venue | IACR(International Association for Cryptologic Research) Cryptology ePrint Archive, Paper 2022/413 (Preprint, 2022-03-31 등록) |
| Preliminary Version | 특허 "Blockchain sharding with adjustable quorums" ([WO2021050929A1](https://patents.google.com/patent/WO2021050929A1/en), 2021년 3월) |
| Link | [ePrint](https://eprint.iacr.org/2022/413) |

---

## BFT SMR

### What is BFT SMR?

**SMR(State-Machine Replication)**은 여러 Node가 같은 Transaction 순서에 합의하여 같은 상태를 유지하는 방법이다. **BFT(Byzantine Fault Tolerant) SMR**은 일부 Node가 프로토콜을 임의로 어기는 Byzantine Fault가 있어도, 제3자 없이 Consensus를 유지한다.

- **Honest Node**: 프로토콜을 그대로 따르는 Node
- **Corrupt Node**: Adversary가 장악한 Node. 메시지를 보내지 않거나, 잘못된 메시지나 서로 다른 메시지를 보내는 등 임의로 행동함
- **Commit**: Node가 Block을 확정하는 것. Commit한 Block은 되돌리지 않음

BFT SMR의 정확성은 두 성질로 평가한다.

- **Safety**: 모든 Honest Node가 같은 Transaction 순서를 처리하여 같은 상태를 유지함
- **Liveness**: 유효한 Transaction은 결국 처리됨

허용할 수 있는 Corrupt Node의 비율은 Network Synchrony, 즉 메시지가 언제까지 도착하는지에 대한 가정에 따라 다르다.

| Synchrony 가정 | 의미 | 두 성질을 모두 만족하기 위한 조건 |
| --- | --- | --- |
| Synchronous | 메시지가 알려진 고정 시간 Δ 안에 항상 도착함 | Corrupt Node 비율 1/2 미만 (Honest Majority) |
| Partially Synchronous | 메시지가 알려지지 않은 고정 시간 안에 도착함 | Corrupt Node 비율 1/3 미만 |

- **Quorum**: 어떤 결정을 내리는 데 필요한 최소 서명(투표) 수
- **Quorum Certificate**: 같은 메시지에 대해 Quorum 크기만큼 모은 서명. 그 메시지에 충분한 수의 Node가 동의했다는 증거

### Sync HotStuff

Instachain의 Intra-shard SMR, 즉 각 Shard 안에서 실행하는 SMR은 [Sync HotStuff](https://doi.org/10.1109/SP40000.2020.00044)(Abraham et al., IEEE(Institute of Electrical and Electronics Engineers) S&P(Symposium on Security and Privacy) 2020)를 확장한 것이다. Sync HotStuff는 Synchronous 가정에서 동작하는 BFT SMR 프로토콜이다.

- **모델**: Node n개 중 Corrupt Node f개, $f < n/2$ ($n = 2f + 1$). 메시지 지연의 상한 Δ를 알고 있다고 가정하며, 모든 메시지에 서명함
- **View**: 한 Leader가 맡는 기간. View마다 Leader가 정해지고, Leader가 Block을 제안함. Leader를 바꾸면 View 번호가 1 증가함
- **두 단계**
    - Steady-state: Leader가 Block을 제안하고 Node가 Vote하는 평상시 단계
    - View-change: 문제가 있는 Leader를 교체하는 단계
- **Quorum**: $f + 1$. 한 Block에 대한 $f + 1$개의 Vote가 그 Block의 Certificate가 됨

**Steady-state**

1. **Propose**: Leader가 Block을 제안한다.
2. **Vote**: Node는 Leader에게서 처음 받은 유효한 Block에 Vote를 Broadcast하고, 받은 제안을 다른 Node에게도 전달한다. 이때 2Δ짜리 Commit-timer를 시작한다.
3. **Commit**: Vote 후 2Δ 동안 같은 높이의 충돌하는 Block을 받지 못하면, 그 Block과 모든 조상 Block을 Commit한다.

Leader가 같은 높이에 서로 다른 Block을 제안하는 것을 **Equivocation**이라고 한다. Steady-state의 Commit 지연은 2Δ다.

Why 2Δ를 기다리면 충분한가? ([설명](https://decentralizedthoughts.github.io/2019-11-12-Sync-HotStuff))

- Node r이 시각 t에 Commit한다고 하자. r은 $t - 2\Delta$에 Vote했고, Vote는 제안을 다시 전달하는 역할도 하므로 모든 Honest Node가 $t - \Delta$까지 그 Block을 받음
- $t - \Delta$까지 충돌하는 Block을 본 Honest Node가 없으면 모든 Honest Node가 이 Block에 Vote함 → Block이 Certificate를 얻음
- 충돌하는 Block이 $t - \Delta$ 이후에 도착하면 Honest Node는 그 Block에 Vote하지 않음 (처음 받은 유효한 Block에만 Vote하므로)
- 충돌하는 Block이 $t - \Delta$ 이전에 도착했다면, 그 Block을 받은 Honest Node의 Vote가 t 이전에 r에게 도착함 → r은 Commit하지 않음

**View-change**

Leader가 Equivocation을 하거나 Block을 충분히 제안하지 않으면, Node는 Leader에게 문제가 있다고 알리는 Blame 메시지를 보낸다. $f + 1$개의 Blame이 모이면 현재 View를 끝내고 다음 View로 넘어간다. 새 View에 들어가면 Node는 Certificate를 얻은 Block(Certified Block) 중 가장 높은 Block을 새 Leader에게 보낸다.

**Quorum $f + 1$이 하는 두 가지 역할**

$n = 2f + 1$에서 Quorum $q = f + 1$은 Safety 쪽 조건과 Liveness 쪽 조건을 동시에 정한다.

- **Safety 쪽**: Corrupt Node가 f개 이하이므로, $f + 1$개 서명으로 만든 Certificate에는 Honest Node의 서명이 적어도 하나 들어 있음. Instachain의 Safety 증명은 이 성질("Corrupt Node 수 < q")과 Synchronous 가정을 함께 사용함
- **Liveness 쪽**: Honest Node가 $n - f = f + 1$개 이상이므로, Corrupt Node가 참여를 거부해도 Honest Node끼리 Certificate를 만들 수 있음 ("Honest Node 수 ≥ q")

**⇒ Safety에 필요한 조건은 "Corrupt Node 수 < q", Liveness에 필요한 조건은 "Honest Node 수 ≥ q"다. Sync HotStuff에서는 $q = f + 1$이 두 조건을 모두 "Honest Majority" 하나로 묶는다. Instachain은 이 q를 조절해 두 조건을 분리한다.**

<br />

## Blockchain Sharding

### What is Sharding?

**Sharding**은 Network의 Node를 무작위로 나누어 여러 개의 작은 그룹, 즉 **Shard**를 만들고, 각 Shard가 BFT SMR로 서로 겹치지 않는 Transaction Ledger, 즉 Blockchain을 병렬로 키우는 방법이다.

- **필요성**: SMR 프로토콜의 Message Complexity, 즉 주고받는 메시지 수는 보통 Network 크기의 제곱에 비례한다. 따라서 작은 Shard는 전체 Network보다 훨씬 빠르게 Transaction을 처리할 수 있다.
- **Scalability**: 새 Node가 들어오면 새 Shard를 만들 수 있으므로, Network 크기가 커질수록 처리량도 늘어난다.

Sharding 프로토콜에서 공통으로 쓰는 개념은 다음과 같다.

- **Epoch**: 프로토콜이 진행되는 고정된 기간. Epoch마다 Shard를 다시 구성함
- **Epoch Randomness**: 모든 Node가 동의한 무작위 비트열. Node를 Shard에 무작위로 배정하는 데 사용함
- **Reconfiguration**: Epoch마다 Node를 Shard 사이에 섞는 과정. Adversary가 특정 Shard를 장악하지 못하게 함
- **Cross-shard Transaction**: 다른 Shard에 있는 Input에 의존하는 Transaction. 원자적으로 실행하려면 다른 Shard의 상태도 바꿔야 하므로 비용이 큼. Shard 수가 많을수록 Cross-shard Transaction이 생길 가능성이 커짐

### 기존 Sharding 프로토콜

- **Elastico** (Luu et al., ACM(Association for Computing Machinery) CCS(Conference on Computer and Communications Security) 2016): 공개 Blockchain을 위한 첫 Sharded Consensus 프로토콜. Epoch마다 참여자가 직전 Blockchain 상태에서 얻은 Epoch Randomness를 바탕으로 PoW(Proof-of-Work) Puzzle을 풀고, PoW 결과의 하위 비트로 Committee(Shard와 같은 뜻)를 정함
- **OmniLedger** (Kokoris-Kogias et al., IEEE S&P 2018): Epoch 시작 시점에 Node의 최대 1/4을 장악할 수 있는 Adversary를 가정함. Epoch마다(약 하루에 한 번) 전체 Reconfiguration을 수행하고, VRF(Verifiable Random Function) 기반으로 Epoch Randomness를 만듦. Partially Synchronous 설정에서 Shard당 최소 600 Node가 필요함
- **[RapidChain](https://eprint.iacr.org/2018/460)** (Zamani et al., ACM CCS 2018): 전체 Node의 1/3까지 Corrupt인 설정. 통신, 계산, 저장을 모두 Sharding하며, 신뢰할 수 있는 주체가 공개 파라미터를 미리 만드는 Trusted Setup이 필요 없음. Synchronous 설정에서 Shard당 최소 250 Node가 필요함. Instachain이 성능 비교의 기준으로 삼는 프로토콜

<br />

## Sharding Limits Problem

기존 Sharding 프로토콜은 **모든 Shard가 Safety와 Liveness를 둘 다 만족**하도록 Shard를 구성한다. 그러려면 각 Shard 안의 Corrupt Node 비율이 Synchronous Network에서는 1/2 미만, Partially Synchronous Network에서는 1/3 미만이어야 한다.

### → 문제: Shard를 작게 만들 수 없음

- Shard는 전체 Node에서 무작위로 뽑힘 → 전체 Corrupt Node 비율이 1/3 미만이어도 특정 Shard에 Corrupt Node가 몰릴 수 있음
- Shard가 작을수록 Corrupt Node가 그 Shard의 과반을 차지할 확률이 지수적으로 커짐

전체 Node N개 중 Corrupt Node가 T개일 때, 크기 n인 Shard S의 Corrupt Node 수 $C_S$는 Hypergeometric Distribution을 따른다. Hypergeometric Distribution은 전체에서 되돌려 넣지 않고 n개를 뽑을 때, 특정 종류가 몇 개 뽑히는지에 대한 분포다. Shard의 Honest Majority가 깨질 확률은 다음과 같다.

$$
\Pr[C_S \geq \lfloor n/2 \rfloor] = \sum_{i=\lfloor n/2 \rfloor}^{n} \frac{\binom{T}{i}\binom{N-T}{n-i}}{\binom{N}{n}}
$$

Shard가 $N/n$개이므로, Epoch 하나의 실패 확률 상한은 이 값에 $N/n$을 곱한 값이다. 이 계산 방식을 Union Bound라고 하며, 여러 사건 중 하나라도 일어날 확률을 각 사건 확률의 합으로 위에서 제한한다.

$N = 2000$, $T = \lfloor N/3 \rfloor$일 때 Shard 크기에 따른 Shard 실패 확률은 다음과 같다.

| Shard 크기 n | 50 | 100 | 150 | 200 |
| --- | --- | --- | --- | --- |
| Shard 실패 확률 | $9.9 \times 10^{-3}$ | $2.9 \times 10^{-4}$ | $8.1 \times 10^{-6}$ | $1.9 \times 10^{-7}$ |

- 그래서 RapidChain은 Shard당 최소 250 Node, OmniLedger는 최소 600 Node가 필요함
- Shard마다 수백 개의 Node가 SMR을 수행해야 하고, SMR의 통신 및 계산 복잡도 때문에 Sharding으로 얻는 확장성 이득이 크게 줄어듦

**⇒ 기존 방식에서 Shard 실패 확률을 낮추는 방법은 Shard 크기를 키우는 것이다. 보안을 낮추지 않으면서 Shard를 작게 만들려면, Shard 크기가 아닌 다른 방법으로 실패 확률을 낮춰야 한다.**

<br />

## Solution: Instachain

**Instachain**은 모든 Shard에서 Safety를 유지하되 일부 Shard에서는 Liveness를 포기하여, Shard 크기를 기존보다 훨씬 작게 만드는 Sharding 프로토콜이다(Ozdayi, Guo, Zamani, 2022). 핵심은 Intra-shard SMR의 Quorum 크기 q를 조절하는 **Adjustable Quorum**이다. 논문은 Synchronous 설정에서 Shard 크기를 50~100 Node로 줄이면서(RapidChain은 200~250 Node), RapidChain보다 실패 확률이 낮고 병렬성(Blockchain을 키울 수 있는 Shard 수)이 높다고 주장한다.

논문의 기여는 세 가지다.

- **Sharding with Adjustable Quorums**: Quorum 크기를 조절해 일부 Shard의 Liveness를 완화하고, 모든 Shard의 Safety는 유지함
- **Stateless Sharded Blockchains**: Sharding에 Stateless Blockchain 모델을 처음 적용함. 주된 목적은 Safety만 만족하는 Shard에서 생기는 문제를 해결하는 것이며, Reconfiguration도 효율적으로 만듦
- **Lock-free, Client-driven Cross-shard Transactions**: UTXO(Unspent Transaction Output, 아직 소비되지 않은 Output) 모델에서 Client가 주도하는 Cross-shard Transaction 검증 프로토콜
    - 엄격한 Atomicity, 즉 여러 Shard에 걸친 처리가 모두 일어나거나 모두 일어나지 않는 성질을 포기해, Transaction에 Lock을 유지하는 비용을 없앰
    - 대신 소비된 금액을 돌려주는 Refund로 일관성을 유지함
    - 같은 Transaction을 반복 제출하는 Replay Attack에도 저항함

| | RapidChain | Instachain |
| --- | --- | --- |
| 모든 Shard가 보장하는 성질 | Safety, Liveness | Safety (Liveness는 일부 Shard만) |
| Intra-shard SMR의 Quorum | 과반 | 조절 가능한 q |
| Shard 실패 조건 | $C_S \geq \lfloor n/2 \rfloor$ | $C_S \geq q$ |
| 진행이 멈춘 Shard | 허용하지 않음 (모든 Shard가 Honest Majority여야 하며, 아니면 프로토콜 실패) | 감지 후 Shard를 닫고, 그 Shard의 Transaction을 다른 Shard로 재배정 |
| Shard 크기 ($N = 2000$ 비교) | 200 | 50 ($q = 35$) |
| Shard 수 | 10 | 40 |
| Shard 실패 확률 | $1.9 \times 10^{-7}$ | $8.6 \times 10^{-8}$ |
| Safety와 Liveness를 모두 만족하는 Shard 수 (기댓값) | 10 | 14.8 |

같은 시기의 독립 연구인 Gearbox(David et al., 2021)도 일부 Shard의 Liveness를 완화한다. 논문이 밝힌 차이는 두 가지다.

- Gearbox는 Partially Synchronous 설정과 Static Adversary(장악할 Node를 처음에 정하면 바꾸지 못함)를 가정하지만, Instachain은 Synchronous 설정과 Adaptive Adversary(실행 도중 장악할 Node를 바꿀 수 있음)를 가정함
- Gearbox는 Safety만 만족하는 Shard를 처리할 때 Network 전체가 통신해야 해서 Network가 커질수록 병목이 될 수 있는 반면, Instachain은 일반 Shard보다 조금 큰 Reference Shard 하나로 처리함

### 핵심 아이디어: 일부 Shard의 Liveness 완화

Safety와 Liveness 중 Safety는 포기할 수 없다. Safety가 깨지면 같은 Output을 두 번 쓰는 Double Spending을 막을 수 없기 때문이다. 그래서 Instachain은 Liveness를 완화한다.

Sharding이 아닌 설정에서는 둘 중 하나만 만족하는 것은 쉽지만 쓸모가 없다.

- Safety만: Node가 아무것도 하지 않으면 됨. 모두 같은 상태에서 시작하므로 다른 상태가 될 수 없음
- Liveness만: Node가 각자 Transaction을 처리하면 됨. 하지만 Node 사이에 일관된 복제가 없음

Sharding 설정에서는 다르다.

- 모든 Shard가 Safety를 만족하면 → 전체 프로토콜도 Safety를 만족함 (Inter-shard Safety)
- Liveness를 만족하는 Shard가 일부 있고, Liveness가 없는 Shard의 Transaction을 그쪽으로 옮길 수 있다면 → 전체 프로토콜도 Liveness를 만족함 (Inter-shard Liveness)

**⇒ Shard 크기를 키우는 대신 Quorum 크기를 키워 실패 확률을 낮추고, 그 대가로 생기는 "Safety만 만족하는 Shard"를 효율적으로 처리한다.**

### 모델과 가정

**Network Model**

- N개 Node가 P2P(Peer-to-Peer) Network로 연결됨. 각 Node는 일정 수의 다른 Node와 연결됨
- Sybil Attack, 즉 한 Adversary가 Identity를 여러 개 만들어 영향력을 키우는 공격에 저항하는 Identity 생성 방식을 사용함. Identity는 공개키와 개인키 쌍이며, 각 Node는 계산적으로 어려운 Puzzle을 풀어 자신이 만든 공개키를 다른 Node들에게 검증받음
- 모든 참여자의 계산 자원은 같다고 가정함
- 모든 메시지는 보낸 Node의 개인키로 서명됨
- 메시지는 Synchronous Gossip 프로토콜, 즉 Node가 받은 메시지를 이웃 Node에게 퍼뜨리는 방식으로 전파됨. Honest Node가 시각 t에 보낸 메시지는 다른 Honest Node에게 $t + \Delta$까지 도착함. 단, 메시지 순서는 보장되지 않음

**Threat Model**

- PPT(Probabilistic Polynomial-Time) Byzantine Adversary가 전체 Node의 $F < 1/3$을 장악함
- **Epoch Adaptive Adversary**(Slowly Adaptive Adversary라고도 함): 프로토콜 시작 시점이나 Epoch 사이에 장악할 Node를 고를 수 있지만, Epoch 안에서는 바꿀 수 없음. 즉 Node가 Shard에 배정된 뒤에 어떤 Node를 장악할지 고를 수 없음
- Node는 장애나 Network 지연으로 연결이 끊길 수 있지만, 어느 시점에서든 Node의 2/3 초과가 온라인 상태인 Honest Node임
- PKI(Public-Key Infrastructure)나 안전한 Broadcast Channel에 의존하지 않음
- CRH(Collision-Resistant Hash)를 위해 Random Oracle, 즉 Hash 함수를 완전한 무작위 함수처럼 다룰 수 있다는 가정을 사용함

**용어**

- Client: 프로토콜 밖에서 Network에 Transaction을 보내는 사용자
- 각 Shard는 Transaction을 Block으로 묶어 Intra-shard SMR로 처리함
- **Shard가 Block을 Commit한다**: Shard 안의 모든 Honest Node가 그 Block을 Commit함
- **Shard가 Transaction을 Commit한다**: 그 Transaction을 담은 Block을 Shard가 Commit함

### 사용하는 도구

#### Threshold Signature와 DKG

- **Threshold Signature**: 비밀키를 n개의 Share로 나누어 갖고, q개 Share로 만든 서명 조각을 모으면 하나의 공개키로 검증되는 서명이 되는 방식 (q-out-of-n). Instachain은 통신 복잡도를 줄이기 위해 Threshold BLS(Boneh–Lynn–Shacham) 서명 같은 방식을 사용함
- **DKG(Distributed Key Generation)**: Shard 구성원들이 함께 키 쌍을 만들고, 비밀키를 구성원들에게 검증 가능하게 나누어 주는 프로토콜 (예: Gennaro et al.의 방식)

Instachain에서 Quorum Certificate는 q개의 서명 Share로 만든 Threshold Signature다. 따라서 Certificate는 Shard 공개키 하나로 누구나 검증할 수 있다.

#### Accumulator와 Stateless Blockchain

**Accumulator**는 집합에 대한 Binding Commitment, 즉 한 번 만들면 다른 집합에 대한 값으로 바꿀 수 없는 짧은 값이다. 집합을 이 값 하나로 나타내며, 어떤 원소가 집합에 있음(Membership)과 경우에 따라 없음(Non-membership)을 증명할 수 있다. Instachain은 다음 기능을 제공하는 Accumulator를 가정한다. RSA(Rivest–Shamir–Adleman) Accumulator가 예다([Boneh et al.](https://eprint.iacr.org/2018/1188)).

| 기능 | 설명 |
| --- | --- |
| $\text{Add}(A, x)$ | Accumulator A에 원소 x 추가 |
| $\text{Delete}(A, x)$ | A에서 원소 x 삭제 |
| $\text{GenMemWitness}(U, x)$ | 집합 U에 x가 있다는 증거(Membership Witness) 생성 |
| $\text{GenNonMemWitness}(U, x)$ | 집합 U에 x가 없다는 증거(Non-membership Witness) 생성 |
| $\text{VerMem}(A, x, w)$ | w가 x의 올바른 Membership Witness이면 1, 아니면 0 |
| $\text{VerNonMem}(A, x, w)$ | w가 x의 올바른 Non-membership Witness이면 1, 아니면 0 |

**Stateless Blockchain**은 Node가 Blockchain 상태를 직접 저장하지 않고 Transaction을 처리하는 설계다.

- Stateful Blockchain: Node가 상태 전체를 저장함. 새 Node는 기존 Node에게서 최신 상태를 받아 동기화해야 하며, 상태가 커질수록 동기화가 오래 걸리고 저장 공간을 감당하지 못하는 Node가 생길 수 있음
- Stateless Blockchain: Block을 검증하는 Node인 Validator는 UTXO 집합 대신 그 집합의 Accumulator만 유지함. RSA Accumulator는 그룹 원소 하나 크기이므로 저장 부담이 거의 없음. 새 Node는 최신 Accumulator만 받으면 참여할 수 있음

UTXO 집합 U에 대한 처리 과정은 다음과 같다.

1. Node는 U의 Accumulator만 가지고 있음. Client는 U를 직접 알고, Transaction의 Input이 U에 있다는 Membership Proof를 붙여 제출함
2. Block을 제안하는 Validator P는 Transaction의 Membership Proof를 검증하고 Accumulator를 갱신함
3. P는 Block과 갱신된 Accumulator(예: Block Header에 포함)를 Broadcast함
4. 다른 Validator는 Block의 Transaction을 처리하며 Accumulator를 검증하고, 자신의 Accumulator를 갱신한 뒤 Block 내용은 버림

대가는 Transaction 크기 증가다. Transaction은 Input마다 그 Input이 아직 소비되지 않았다는 Membership Proof를 가져야 한다. 이를 만들려면 Client가 명시적 상태를 알아야 하므로, Client는 Network를 들으며 상태를 직접 유지하거나 상태 전체를 유지해 주는 "Semi-trusted" 주체에 의존한다. 이 주체에 대해서는 상태를 언제든 제공한다는 점(Availability)만 신뢰하고, 제공한 내용이 올바르다는 점(Validity)은 신뢰하지 않는다. 즉 이 주체가 이미 소비된 Output에 대한 Membership Proof를 만들어 낼 수는 없다.

### Instachain 단계별 동작 원리

#### Step 1: Adjustable Quorums

목적: Shard 크기를 키우지 않고 Shard 실패 확률을 낮춤

Shard S의 크기를 n, Honest Node 수를 $H_S$, Corrupt Node 수를 $C_S$라고 하자($n = H_S + C_S$). Quorum 크기 q에 따라 Shard는 세 상태 중 하나가 된다.

| 상태 | 조건 | 보장하는 성질 |
| --- | --- | --- |
| **Active Shard** | $H_S \geq q$ | Safety, Liveness |
| **Safe Shard** | $H_S < q$ 그리고 $C_S < q$ | Safety만 |
| **Corrupt Shard** | $C_S \geq q$ | 둘 다 보장하지 않음 |

- 논문은 보안 증명에서 Active Shard를 Super-honest Shard라고도 부름. 이 글에서는 Active Shard로 통일함
- 논문은 "Active"를 아직 동작 중이라는 뜻으로도 씀. 이 글에서는 닫히지 않고 동작 중인 Shard를 **Running Shard**라고 부름

Why 이 조건인가?

- Safety: $C_S < q$이면 q개 서명이 필요한 모든 Certificate에 Honest Node의 서명이 적어도 하나 들어감
- Liveness: $H_S \geq q$이면 Honest Node끼리 Certificate를 만들 수 있음
- 논문이 사용하는 q는 모두 $n/2$보다 큼($n = 50$에서 27~35, $n = 100$에서 60). $q > n/2$이면 Active와 Corrupt가 동시에 성립할 수 없음 ($H_S \geq q$이고 $C_S \geq q$이면 $n \geq 2q > n$이 되어 모순)

q를 키우면 다음과 같이 바뀐다.

- Corrupt Shard가 되려면 더 많은 Corrupt Node가 필요함 → Shard 실패 확률 감소
- Active Shard가 되려면 더 많은 Honest Node가 필요함 → Active Shard가 될 확률 감소, Safe Shard가 될 확률 증가

예: $n = 50$, $q = 35$

- Liveness: $C_S \leq 15$일 때 (Corrupt 비율 30% 이하)
- Safety: $C_S \leq 34$일 때 (Corrupt 비율 70% 미만)
- 과반 Quorum을 쓰는 RapidChain은 두 성질 모두 Corrupt 비율 50% 미만일 때 보장됨

$N = 2000$, $T = \lfloor N/3 \rfloor$, $n = 50$에서 q에 따른 확률은 다음과 같다. Safe 열은 $1 - \text{Active} - \text{Corrupt}$로 계산한 값이다.

| q | Active | Safe | Corrupt (Shard 실패) |
| --- | --- | --- | --- |
| 27 | 0.98 | 0.019 | $1.8 \times 10^{-3}$ |
| 29 | 0.93 | 0.072 | $2.4 \times 10^{-4}$ |
| 31 | 0.81 | 0.19 | $2.4 \times 10^{-5}$ |
| 33 | 0.61 | 0.39 | $1.7 \times 10^{-6}$ |
| 35 | 0.37 | 0.63 | $8.6 \times 10^{-8}$ |

- RapidChain: $n = 200$이면 Shard 10개, Shard 실패 확률 $1.9 \times 10^{-7}$
- Instachain: $n = 50$, $q = 35$이면 Shard 40개, Shard 실패 확률 $8.6 \times 10^{-8}$. 기대 Active Shard 수는 $40 \times 0.37 = 14.8$
- → Shard를 1/4 크기로 줄이면서 Shard 실패 확률은 더 낮고, 진행 가능한 Shard는 더 많음

**→ 문제: Safe Shard는 진행이 멈출 수 있음**

- Safe Shard에서 Corrupt Node가 참여를 거부하면 Honest Node만으로는 q를 채울 수 없음 → Block을 Commit할 수 없음
- 그 Shard에 배정된 Transaction은 처리되지 않을 수 있음
- Liveness가 없는 Shard를 감지하고, 그 Shard의 Transaction을 다른 Shard로 옮길 주체가 필요함

#### Step 2: Reference Shard와 Epoch 시작

**⇒ 해결: Shard들을 관리하는 Reference Shard를 둠**

Instachain의 Shard는 두 종류다.

- **Reference Shard**: Shard들을 구성하고 관리하는 특별한 Shard. 높은 확률로 Honest Majority가 되도록 일반 Shard보다 크게 만듦(성능 분석에서는 Worker Shard의 4배). 따라서 Safety와 Liveness를 모두 만족하며, 내부에서는 Honest Majority SMR(예: 기본 Sync HotStuff)을 사용함
- **Worker Shard**: Client의 Transaction을 처리하는 Shard. 이 글에서 그냥 Shard라고 하면 Worker Shard를 뜻함

Node는 Reference Shard와 Worker Shard 하나에 동시에 참여할 수 있지만, Worker Shard 두 개 이상에는 참여할 수 없다.

Reference Shard는 다음 두 Table을 관리하고 Network에 Broadcast한다.

- **Node-to-shard Table**: 어느 Node가 어느 Shard에 배정되었는지
- **Tx-to-shard Table**: 어느 Transaction이 어느 Shard에 배정되었는지

이 밖에 Epoch 사이의 Shard Reconfiguration과 다음 Epoch의 Randomness 생성도 맡는다.

**Epoch 시작 과정**

1. **Randomness**: 첫 Epoch의 Randomness는 일회성 Bootstrapping 프로토콜(예: RapidChain의 방식)로 만든다. 이후 Epoch의 Randomness는 Reference Shard가 현재 Epoch이 끝날 때 만든다.
2. **배정**: 각 Node는 Randomness로 (1) Node의 Shard 배정과 (2) Intra-shard SMR을 이끌 Leader 순서를 알게 된다.
3. **Intra-shard Key Generation**: 각 Shard가 DKG를 실행해 Shard Identity를 만들고 Reference Shard에 등록한다.
4. **Transaction 배정**: Reference Shard가 등록된 Shard에 Transaction을 배정한 Tx-to-shard Table을 Commit하고 Broadcast한다.

**Intra-shard Key Generation**

- Shard S는 DKG로 키 쌍 $(sk_S, pk_S)$를 만든다. $pk_S$는 Shard가 공유하는 공개키이고, $sk_S$는 구성원들에게 q-out-of-n으로 나뉜다.
- $H(pk_S)$를 **Shard Identity**라고 한다.
- 각 Node r은 자신의 비밀키 Share로 Shard Identity에 대한 서명 Share $\sigma_r(H(pk_S))$를 만들어 Reference Shard에 보낸다.
- Reference Shard는 q개의 Share를 모아 Threshold Signature를 만들고 $pk_S$로 검증한다. 통과하면 Tx-to-shard Table에 S를 추가한다.
- 모든 Shard의 Share를 받을 만큼 충분히 기다린 뒤, Reference Shard는 Tx-to-shard Table을 Commit하고 Broadcast한다. Shard들은 이 Table을 받은 뒤 자신에게 배정된 Transaction을 처리하기 시작한다.

이 단계를 통과한 Shard만 해당 Epoch에 Transaction을 처리할 수 있다. Safe Shard의 Corrupt Node가 침묵해 DKG를 막으면, 그 Shard는 Tx-to-shard Table에서 빠지고 구성원은 Node-to-shard Table에서 ⊥(이번 Epoch에서 닫힌 Shard를 뜻하는 특수 기호)로 배정된다. 그러면 Transaction이 그 Shard로 가지 않고 Honest Node도 Intra-shard SMR에 참여하지 않는다. 즉 그 Shard는 이번 Epoch에서 닫힌다.

**Tx-to-shard Table 구조**

Transaction을 배정할 때는 (1) 모든 Transaction이 정확히 한 Shard에 배정되어야 하고(일관성), (2) Shard 간 부하가 고르게 나뉘어야 한다.

- Tx-to-shard Table은 Complete Binary Search Tree로 구현함
- Leaf는 Shard Identity이고, Root에서 Leaf까지의 경로가 그 Shard가 맡는 Transaction Hash의 Prefix, 즉 Hash의 앞부분 비트임
- 예: 경로가 00이면 S는 Hash의 처음 2비트가 00인 Transaction을 처리함
- Shard가 새로 생기거나 닫히면 Reference Shard가 Tree를 재구성해 Broadcast함

| Epoch | Shard 수 | 각 Shard가 맡는 Hash Prefix |
| --- | --- | --- |
| e | 4 | $S_0$: 00, $S_1$: 01, $S_2$: 10, $S_3$: 11 |
| e+1 (Node 증가) | 5 | $S_0$: 000, $S_1$: 001, $S_2$: 01, $S_3$: 10, $S_4$: 11 |
| e+2 (Node 감소) | 3 | $S_0$: 00, $S_1$: 01, $S_2$: 1 |

**→ 문제: Intra-shard SMR이 Quorum q로 동작하면서 Commit의 증거를 남겨야 함**

- Safe Shard에서는 Safety, Active Shard에서는 Safety와 Liveness를 보장해야 함
- 다른 Shard가 이 Shard의 상태를 이어받거나 이 Shard의 Commit을 근거로 Transaction을 처리하려면, "이 Block이 Commit되었다"는 사실을 Shard 밖에서도 검증할 수 있어야 함

#### Step 3: Intra-shard SMR

**⇒ 해결: Sync HotStuff를 Quorum q 기준으로 바꾸고, Precommit 단계를 추가해 Commit Certificate를 남김**

**Block Structure**

높이 k의 Block은 Header와 Body로 구성된다.

$$
B_k := (B^h_k, B^b_k)
$$

- Body: Transaction들을 Merkle Tree로 묶은 것
- Header: $B^h_k := (k, e, \text{MT}, A^{\text{UTXO}}_S, H(B^h_{k-1}), H(pk_S))$
    - k: Block 높이
    - e: Block이 만들어진 Epoch
    - MT: Body의 Transaction Merkle Tree Root
    - $A^{\text{UTXO}}_S$: Shard S의 최신 UTXO 상태에 대한 Accumulator
    - $H(B^h_{k-1})$: 이전 Block Header의 Hash
    - $H(pk_S)$: Block을 만든 Shard의 Identity
- Stateless Blockchain 모델을 따르므로, Node는 Block을 Commit한 뒤 Body를 버리고 최신 Block Header와 그 Commit Certificate $CC(B_k)$만 보관함

**Quorum Certificate**

Node r은 메시지 m을 보낼 때 m의 Hash를 자신의 비밀키 Share로 서명한다. m에 대한 q개의 서명 Share가 m의 Quorum Certificate가 된다. 기술적으로는 서명 Share로 만든 Threshold Signature다.

| Certificate | 구성 | 의미 |
| --- | --- | --- |
| $BC(B_k)$, Block Certificate | $B_k$에 대한 q개의 vote | Block이 충분한 Vote를 받음 |
| $CC(B_k)$, Commit Certificate | $B_k$에 대한 q개의 precommit | Block이 Commit됨 |
| $VC(v)$, View-change Certificate | View v에 대한 q개의 blame | View v를 끝냄 |

**Steady-state** (현재 View 번호 v, Leader L, $\langle m \rangle_r$은 r의 비밀키 Share로 서명한 메시지 m)

1. **Propose**: L이 $\langle \textbf{propose}, \langle B_k \rangle_L, BC(B_{k-1}), v \rangle_L$을 Broadcast한다.
2. **Vote**: L에게서 직접, 또는 다른 Node의 Vote를 통해 처음 받은 유효한 높이 k Block에 대해, precommit-timer$_k$를 2Δ로 설정하고 $\langle \textbf{vote}, \langle B_k \rangle_L, v \rangle$를 Broadcast한다.
3. **Precommit**: precommit-timer$_k$가 0이 되었을 때, 높이 k에서 Equivocation을 감지하지 않았고 $B_k$가 $BC(B_k)$를 가지고 있으면 $\langle \textbf{precommit}, \langle B_k \rangle_L, BC(B_k), v \rangle$를 Broadcast한다.
4. **Commit**: 유효한 $BC(B_k)$를 담은 precommit 메시지를 q개 받으면, $B_k$가 아직 Commit되지 않았을 경우 $B_k$와 모든 조상 Block을 Commit한다. 받은 q개의 precommit, 즉 $CC(B_k)$를 Broadcast한다.

(3단계의 Timer는 2단계에서 설정한 precommit-timer$_k$다. 논문의 프로토콜 그림에는 이 자리에 Sync HotStuff의 이름인 commit-timer가 적혀 있다.)

**View-change** (View v의 Leader L, View v+1의 Leader L')

1. **Blame**: View v에서 $(2p + 1)\Delta$ 시간 동안 L에게서 받은 Block이 p개 미만이면 $\langle \textbf{blame}, v \rangle$를 Broadcast한다(논문은 p의 값을 따로 정하지 않는다). L이 같은 높이에 두 개 이상의 Block을 제안했으면 $\langle \textbf{blame}, v \rangle$와 두 Equivocation Block을 Broadcast한다.
2. **Quit Old View**: $\langle \textbf{blame}, v \rangle$를 q개 모으면, 이를 Broadcast하고 View v를 떠난다. 실행 중인 precommit-timer를 모두 중단하고 View v에서 더는 Vote하지 않는다.
3. **Status**: 2Δ를 기다린 뒤 View v+1에 들어간다. 유효한 Block Certificate를 가진 가장 높은 Block을 L'에게 보내고 Steady-state로 돌아간다.

**Sync HotStuff와 달라진 점**

| | Sync HotStuff | Instachain Intra-shard SMR |
| --- | --- | --- |
| Quorum | $f + 1$ ($n = 2f + 1$) | 조절 가능한 q |
| Commit 조건 | Vote 후 2Δ 동안 Equivocation이 없으면 각 Node가 Commit | 2Δ 후 BC가 있으면 Precommit, BC를 담은 Precommit이 q개 모이면 Commit |
| Commit의 증거 | 각 Node의 Timer로 결정 | q개 Precommit으로 된 $CC(B_k)$ |

Safe Shard에서는 다음과 같이 동작한다.

- Corrupt Node가 Vote나 Precommit을 하지 않으면 Honest Node 수가 q보다 작으므로 BC나 CC를 만들 수 없음 → Commit이 멈춤 (Liveness 없음)
- Corrupt Node 수도 q보다 작으므로 Corrupt Node끼리 BC나 CC를 만들 수 없음 → 충돌하는 Block을 Commit시킬 수 없음 (Safety 유지)

이 SMR이 보장하는 성질은 다음과 같다.

- **Intra-shard Safety**: Safe Shard나 Active Shard에서 Honest Node 하나가 높이 k에 $B_k$를 Commit하면, 모든 Honest Node가 결국 높이 k에 $B_k$를 Commit함
- **Intra-shard Liveness**: Active Shard에 배정된 유효한 Transaction은 결국 Commit됨
- **Provable Commits**: Commit된 모든 Block $B_k$에는 S가 그 Block을 Commit했다는 Commit Proof $\text{COM}_{B_k, S}$가 있음
    - 직접 Commit된 Block: $CC(B_k)$가 Commit Proof
    - 다른 Block이 Commit되면서 함께 Commit된 조상 Block: CC가 있는 Block에서 그 Block까지 이어지는 Hash Chain이 Commit Proof
    - Header에 Shard Identity가 있으므로 Commit Proof는 어느 Shard가 Commit했는지도 알려 줌

**→ 문제: Safe Shard의 Corrupt Node가 정상적으로 참여하다가 Epoch 중간에 멈출 수 있음**

- DKG 단계에서 멈추면 Step 2에서처럼 Shard가 등록되지 않지만, 그 뒤에 멈추면 이를 감지할 방법이 필요함
- 멈춘 Shard가 이미 Transaction을 처리해 UTXO를 만들었다면, 그 상태를 이어받을 Shard도 필요함

#### Step 4: Safe Shard Detector와 Shard Closing

**⇒ 해결: Honest Node가 Liveness 부족을 감지해 Reference Shard에 알리고, Reference Shard가 그 Shard를 닫음**

각 Node는 Epoch 동안 **Safe Shard Detector**를 실행한다. Safe Shard Detector는 Safe Shard에서만 일어날 수 있는 사건을 추적하고, 이를 감지하면 Reference Shard에 $\langle \textbf{close}, S \rangle_r$ 요청을 보낸다.

| 규칙 | Timer 시작 시점 | Timer | Timer가 0이 될 때까지 다음 중 아무것도 받지 못하면 close |
| --- | --- | --- | --- |
| Vote Inactivity | $B_k$에 Vote한 직후 | 2Δ | L의 Equivocation 증거, $VC(v)$, $BC(B_k)$ |
| Precommit Inactivity | $B_k$에 Precommit한 직후 | 2Δ | L의 Equivocation 증거, $CC(B_k)$ |
| Blame Inactivity | Equivocation 증거를 Broadcast한 직후 | 2Δ | $VC(v)$ |
| View-change Inactivity | View v를 떠난 직후 | 6Δ | 유효한 Block 제안, $VC(v+1)$ |
| View-change Bound | View-change를 $n - q + 1$번 한 직후 | 없음 | 즉시 close |

Why Active Shard에서는 이 사건들이 일어나지 않는가?

- Vote Inactivity: r이 시각 t에 Vote하면 다른 Honest Node도 $t + \Delta$까지 그 Block을 받아 Vote함(그 전에 Equivocation이나 q개 Blame을 받은 경우는 제외). Active Shard는 Honest Node가 q개 이상이므로 r은 $t + 2\Delta$까지 $BC(B_k)$를 받음
- Precommit Inactivity: 다른 Honest Node도 $t + \Delta$까지 Precommit함(Leader가 Equivocation으로 일부 Node의 Precommit을 막은 경우는 제외). 따라서 r은 $t + 2\Delta$까지 Equivocation Block이나 $CC(B_k)$를 받음
- Blame Inactivity: Equivocation 증거는 $t + \Delta$까지 모든 Honest Node에 도달하고, 모두 Blame함. 따라서 $t + 2\Delta$까지 q개 Blame, 즉 $VC(v)$가 모임
- View-change Inactivity: r이 t에 View v를 떠나면 모든 Honest Node가 $t + \Delta$까지 떠나고, 2Δ를 기다린 뒤 새 Leader에게 보고함 → 새 Leader는 $t + 4\Delta$까지 Block을 제안할 수 있음. 제안이 없으면 Honest Node가 $t + 5\Delta$에 Blame하고, r은 $t + 6\Delta$까지 $VC(v+1)$을 받음
- View-change Bound: Active Shard의 Corrupt Node는 $n - q$개 이하이므로, View-change를 $n - q$번 하기 전에 Honest Leader가 나옴

**Close 요청 기준: $n - q + 1$개**

Reference Shard는 한 Shard에서 close 요청을 $n - q + 1$개 받아야 그 Shard를 닫는다.

- Active Shard: $C_S \leq n - q$이므로 Corrupt Node만으로는 close 요청 $n - q + 1$개를 만들 수 없음. Honest Node는 Active Shard에서 close를 보내지 않음 → Adversary는 Active Shard를 닫을 수 없음
- Safe Shard: $C_S \leq q - 1$이므로 $H_S \geq n - q + 1$ → Honest Node만으로 close 요청 수를 채울 수 있음

**Shard Closing 과정**

Reference Shard의 현재 Leader $L_R$이 Shard $S_i$에서 close 요청을 $n - q + 1$개 받으면 다음을 진행한다.

1. **Table 갱신**: Node-to-shard Table에서 $S_i$의 모든 구성원을 ⊥로 배정한다. Tx-to-shard Table에서는 $S_i$를 빼고 Tree를 재구성한다. 그 결과 $S_i$가 맡던 Transaction이 Running Shard $S_j$에 배정된다고 하자.
2. **Table Commit**: $L_R$이 갱신한 Table을 close 요청들과 함께 Reference Shard에 제안하고, Reference Shard는 Honest Majority SMR로 이를 Commit한다.
3. **Table Broadcast**: Commit된 Table을 Network 전체에 Broadcast한다.
4. **Shard 닫기와 상태 전달**: $S_i$의 Node는 제안을 검증한 뒤 Intra-shard SMR 참여를 멈춘다. $S_j$의 Node는 $S_i$의 Node에게 최신 상태를 요청하고, $S_i$의 Node는 최신 Block Header로 응답한다. $S_j$는 $S_i$의 상태를 받은 뒤, 기존 SMR Instance와 별도로 독립된 SMR Instance를 실행해 $S_i$에 배정되었던 Transaction을 처리한다.

Safe Shard Detector가 보장하는 성질은 다음과 같다.

- Active Shard의 Honest Node는 close 요청을 보내지 않으므로, Active Shard는 닫히지 않음
- Safe Shard는 결국 Transaction을 처리하거나 결국 닫힘. 닫히지 않으려면 Active Shard처럼 Block을 만들거나 View-change를 계속해야 하는데, View-change 횟수는 $n - q + 1$번으로 제한되기 때문

**→ 문제: 닫힌 Shard의 상태를 효율적이고 검증 가능하게 넘겨야 함**

- $S_j$가 $S_i$의 Transaction을 이어서 처리하려면 $S_i$의 최신 상태(UTXO 집합)를 알아야 함
- 상태 전체를 옮기면 비용이 크고, 받은 상태가 실제로 $S_i$가 Commit한 상태인지도 확인할 수 있어야 함

#### Step 5: Stateless Shard와 상태 전달

**⇒ 해결: Shard 상태를 Accumulator로 압축하고, 상태 전달은 최신 Block Header 전달로 대신함**

- Block Header에 $A^{\text{UTXO}}_S$가 들어 있고, Node는 Commit 후 최신 Header와 $CC(B_k)$만 보관함
- 상태 전달 = $S_i$의 Honest Node가 최신 Block Header를 $S_j$에 보내는 것
- Header에는 Commit Certificate가 함께 있으므로 $S_j$는 이 Header가 $S_i$에서 Commit된 것인지 검증할 수 있음
- Client는 명시적 상태를 직접 유지하고, Transaction의 Input마다 $A^{\text{UTXO}}_S$ 기준 Membership Proof를 붙임

Why 이 구조로 Step 4의 문제가 해결되는가?

- 상태가 Accumulator 하나로 압축되므로 전달량이 작음 (RSA Accumulator는 그룹 원소 하나)
- Safe Shard에서도 $C_S < q$이므로 Corrupt Node끼리 CC를 만들 수 없음 → 가짜 Header를 Commit된 것처럼 보이게 할 수 없음
- 상태가 작으므로 Epoch 사이 Reconfiguration도 효율적이 됨

**→ 문제: Shard를 작게 만들수록 Cross-shard Transaction이 늘어남**

- Shard가 작아지면 Shard 수가 늘고, 다른 Shard의 Input에 의존하는 Transaction이 많아짐
- 엄격한 Atomicity를 위해 Transaction에 Lock을 유지하면 Lock 상태를 저장하는 비용이 생김

#### Step 6: Lock-free, Client-driven Cross-shard Transaction

**⇒ 해결: 입력 Shard는 Input을 바로 소비하고, Client가 그 증거를 모아 출력 Shard에 전달함**

**Transaction Format** (UTXO 모델)

Input 하나와 Output 하나로 된 Transaction을 예로 든다.

$$
T := \langle I, O \rangle
$$

- Output $O := (pk, v)$: 주소 pk에 v만큼 보냄
- Input $I := (O', sig, \text{MEM}_{\langle O', S \rangle})$
    - $O' := (ctr, pk', v')$: Shard S의 Accumulator에 들어 있는 UTXO. ctr은 Accumulator에 추가된 순서
    - sig: $pk'$로 검증되는 서명
    - $\text{MEM}_{\langle O', S \rangle}$: $A^{\text{UTXO}}_S$ 기준으로 $O'$의 Membership Proof

**Single-shard Verification**

T가 $O'$이 있는 Shard S에 배정되었다면 S 안에서 검증한다.

1. Tx-to-shard Table로 T가 S에 배정되었는지 확인
2. $v' \geq v$ 확인
3. sig가 $pk'$로 유효한지 확인
4. $\text{MEM}_{\langle O', S \rangle}$가 $A^{\text{UTXO}}_S$ 기준으로 유효한지 확인

통과하면 S는 T를 Block에 넣고, $A^{\text{UTXO}}_S$에서 $O'$을 제거하고 O를 추가한다.

**Cross-shard Verification**

$T := (\langle I_1, I_2 \rangle, O_3)$에서 $I_1$, $I_2$가 참조하는 $O_1$, $O_2$가 각각 Shard $S_1$, $S_2$에 있고, T는 $S_3$에 배정되었다고 하자.

1. Client가 T를 $S_1$과 $S_2$에 보낸다.
2. $S_1$, $S_2$는 각각 $I_1$, $I_2$를 검증하고, $(T, 1)$, $(T, 2)$를 Block에 넣은 뒤, $A^{\text{UTXO}}_{S_1}$, $A^{\text{UTXO}}_{S_2}$에서 $O_1$, $O_2$를 제거한다.
3. Client는 입력 Shard에서 **PoI(Proof-of-Inclusion)**를 받는다.

$$
\text{POI}_{\langle I_i, S_i \rangle} := (\text{MT}_{\langle (T,i), B \rangle}, B^h, \text{COM}_{\langle B, S_i \rangle})
$$

- $\text{MT}_{\langle (T,i), B \rangle}$: $(T, i)$가 Block B에 들어 있다는 Merkle Proof. Merkle Proof는 Leaf에서 Root까지 올라가는 경로의 형제 Hash 목록으로, 원소가 Merkle Tree에 들어 있음을 보임 (i는 T의 몇 번째 Input인지)
- $B^h$: Block B의 Header
- $\text{COM}_{\langle B, S_i \rangle}$: B가 $S_i$에서 Commit되었다는 Commit Proof
- 의미: $(T, i)$가 $S_i$에서 Commit되었으므로 $O_i$가 $S_i$의 상태에서 제거되었음

출력 Shard $S_3$는 PoI를 다음 순서로 검증한다.

1. $B^h$에서 $S_i$를 알아냄 (Header의 Shard Identity)
2. $\text{COM}_{\langle B, S_i \rangle}$를 $S_i$의 Identity로 검증
3. $(T, i)$의 Merkle Proof를 $B^h$ 기준으로 검증

Input의 유효성에 따라 세 경우로 나뉜다.

| 경우 | Client가 얻는 PoI | 처리 |
| --- | --- | --- |
| $I_1$, $I_2$ 모두 유효 | $\text{POI}_{\langle I_1, S_1 \rangle}$, $\text{POI}_{\langle I_2, S_2 \rangle}$ | Client가 $T_{com} := (T, \text{POI}_{\langle I_1, S_1 \rangle}, \text{POI}_{\langle I_2, S_2 \rangle})$를 $S_3$에 보냄. $S_3$는 $O_1.v_1 + O_2.v_2 = O_3.v_3$와 두 PoI를 검증한 뒤 $T_{com}$을 Block에 넣고 $A^{\text{UTXO}}_{S_3}$에 $O_3$를 추가 |
| $I_1$만 유효 | $\text{POI}_{\langle I_1, S_1 \rangle}$ | $O_1$만 제거되어 Atomicity가 깨짐. Client가 Partial Commit Transaction $T_{com} := (T, \text{POI}_{\langle I_1, S_1 \rangle})$를 $S_3$에 보내 Refund. $S_3$는 PoI를 검증한 뒤 $O'_3.v_3 = O_1.v_1$인 $O'_3$를 추가 |
| 둘 다 무효 | 없음 | 유효한 Cross-shard Transaction을 만들 수 없음 |

Why Lock-free, Client-driven인가?

- 입력 Shard는 Input이 유효하면 바로 소비(제거)함 → Lock을 걸었다가 푸는 단계가 없음
- Client가 PoI를 모아 출력 Shard에 전달함 → Shard 간 통신을 줄임
- 대가: 엄격한 Atomicity 대신, 한쪽 Input만 소비된 경우 Refund로 일관성을 유지함

**→ 문제: Replay Attack**

- [Sonnino et al.](https://arxiv.org/abs/1901.11218)은 OmniLedger와 RapidChain의 Cross-shard 프로토콜이 Replay Attack에 취약함을 보였음. 지금까지의 Instachain 프로토콜도 마찬가지임
- 공격: Adversary가 같은 $T_{com}$을 $S_3$에 여러 번 보냄 → 검증은 매번 통과하므로 $S_3$가 $O_3$를 여러 번 만듦

#### Step 7: Replay Attack 방지

**⇒ 해결: 처리한 PoI를 Accumulator로 기록하고, Client가 Non-membership Proof를 제출하게 함**

- 단순한 방법: Shard가 처리한 PoI 목록을 저장함 → 목록이 계속 커짐
- Instachain: 각 Shard S가 처리한 PoI 집합의 Accumulator $A^{\text{POI}}_S$를 유지함
- Client는 출력이 만들어질 Shard의 $A^{\text{POI}}$ 기준으로, 각 PoI의 Non-membership Proof를 함께 제출해야 함

두 Input이 모두 유효한 경우, Replay Attack에 저항하는 최종 Commit Transaction은 다음과 같다.

$$
T_{com} := (T, \text{POI}_{\langle I_1, S_1 \rangle}, \text{POI}_{\langle I_2, S_2 \rangle}, \text{NMEM}_{\text{POI}_{\langle I_1, S_1 \rangle}}, \text{NMEM}_{\text{POI}_{\langle I_2, S_2 \rangle}})
$$

- $\text{NMEM}_{\text{POI}_{\langle I_i, S_i \rangle}}$: $\text{POI}_{\langle I_i, S_i \rangle}$가 출력 Shard($S_3$)의 $A^{\text{POI}}$에 없다는 Non-membership Proof
- 논문의 식 설명에는 기준 Accumulator가 $A^{\text{POI}}_{S_i}$로 적혀 있지만, 앞의 설명("출력이 만들어질 Shard에 제출")과 보안 증명에서는 $T_{com}$을 처리하는 Shard의 $A^{\text{POI}}$를 검사함

$A^{\text{POI}}_S$를 Block Header에 넣으므로, 상태 전달 프로토콜을 바꾸지 않아도 처리한 PoI 기록이 함께 넘어간다. 최종 Block Header는 다음과 같다.

$$
B^h_k := (k, e, \text{MT}, A^{\text{UTXO}}_S, A^{\text{POI}}_S, H(B^h_{k-1}), H(pk_S))
$$

Why 이 구조로 Replay Attack이 막히는가?

- 처음 처리할 때 PoI가 $A^{\text{POI}}$에 추가됨
- 같은 $T_{com}$을 다시 보내면 그 PoI의 Non-membership Proof가 검증을 통과하지 못함
- Epoch이 바뀌어 같은 Transaction을 처리하는 Shard가 달라져도, 새 Shard는 이전 Shard가 관리하던 PoI Accumulator를 검사함 (논문의 보안 증명)

### Final Instachain Protocol

한 Epoch의 흐름은 다음과 같다.

1. **배정**: Epoch Randomness로 Node를 Shard에 배정하고 Shard별 Leader 순서를 정함
2. **Intra-shard Key Generation**: 각 Shard가 DKG로 $(sk_S, pk_S)$를 만들고, q개 서명 Share로 Shard Identity $H(pk_S)$를 Reference Shard에 등록함. 등록하지 못한 Shard는 이번 Epoch에서 닫힘
3. **Transaction 배정**: Reference Shard가 Hash Prefix 기반 Tx-to-shard Table을 Commit하고 Broadcast함
4. **Intra-shard SMR**: 각 Shard가 Quorum q로 Propose, Vote, Precommit, Commit을 반복하고, Leader가 문제를 일으키면 View-change를 수행함. Node는 최신 Header와 $CC(B_k)$만 보관함
5. **Transaction 검증**: Single-shard Transaction은 해당 Shard가 Membership Proof로 검증함. Cross-shard Transaction은 Client가 입력 Shard의 PoI와 출력 Shard의 Non-membership Proof를 모아 $T_{com}$으로 제출함
6. **Safe Shard 처리**: Honest Node가 Safe Shard Detector로 Liveness 부족을 감지해 close 요청을 보내고, $n - q + 1$개가 모이면 Reference Shard가 Shard를 닫고 Transaction과 상태(최신 Header)를 Running Shard로 넘김
7. **Epoch 종료**: Reference Shard가 다음 Epoch의 Randomness를 만들고 Shard를 재구성함

### Instachain 동작 예시

아래 예시의 Shard 구성과 금액은 설명을 위해 정한 값이다.

설정: $n = 50$, $q = 35$ → close 기준 $n - q + 1 = 16$

| Shard | $H_S$ | $C_S$ | 상태 | 이 예시에서의 결과 |
| --- | --- | --- | --- | --- |
| $S_1$ | 40 | 10 | Active ($H_S \geq 35$) | $I_1$ 처리, PoI 생성 |
| $S_2$ | 38 | 12 | Active | $I_2$ 처리, PoI 생성, 닫힌 $S_3$의 Transaction 이어받음 |
| $S_3$ | 30 | 20 | Safe ($H_S < 35$, $C_S < 35$) | Commit이 멈춤 → 닫힘 |

- $S_3$가 DKG에서 등록되려면 서명 Share 35개가 필요하므로, $S_3$의 Corrupt Node는 DKG에는 참여했다고 하자. $S_3$는 이후 얼마 동안 정상적으로 Block을 Commit해 상태(UTXO)를 가지고 있다.
- Client A는 $S_1$에 $O_1$(0.5 BTC), $S_2$에 $O_2$(0.3 BTC)를 가지고 있고, B에게 0.8 BTC를 보내려 한다. $T = (\langle I_1, I_2 \rangle, O_3)$, $O_3 = (pk_B, 0.8)$이며 T의 Hash Prefix에 따라 T는 $S_3$에 배정된다.

1. **입력 Shard 처리**

    A가 T를 $S_1$, $S_2$에 보낸다. $S_1$은 $I_1$의 서명과 Membership Proof를 검증하고 $(T, 1)$을 Block에 넣어 Commit한 뒤 $A^{\text{UTXO}}_{S_1}$에서 $O_1$을 제거한다. $S_2$도 같은 방식으로 $O_2$를 제거한다.

2. **PoI 수집**

    A는 $\text{POI}_{\langle I_1, S_1 \rangle}$, $\text{POI}_{\langle I_2, S_2 \rangle}$를 받고, 각 PoI가 $A^{\text{POI}}_{S_3}$에 없다는 Non-membership Proof를 만들어 $T_{com}$을 $S_3$에 보낸다.

3. **$S_3$에서 Commit이 멈춤**

    $S_3$의 Leader가 $T_{com}$을 담은 $B_k$를 제안한다. Honest Node 30개는 Vote하지만 Corrupt Node 20개는 침묵한다. Vote가 30개뿐이라 $q = 35$를 채우지 못하므로 $BC(B_k)$가 만들어지지 않고, Precommit과 Commit도 일어나지 않는다. Corrupt Node 20개도 35를 채우지 못하므로 다른 Block에 Certificate를 만들 수 없다(Safety 유지).

4. **감지**

    Honest Node의 Vote Inactivity Timer(2Δ)가 Equivocation 증거, $VC(v)$, $BC(B_k)$ 없이 만료되고, Honest Node 30개가 close 요청을 보낸다. 30 ≥ 16이므로 Reference Shard는 $S_3$를 닫는다. 반대로 $S_1$의 Corrupt Node 10개가 close 요청을 보내도 10 < 16이므로 $S_1$은 닫히지 않는다.

5. **Closing과 상태 전달**

    Reference Shard는 $S_3$의 구성원을 ⊥로 배정하고, Tree를 재구성해 $S_3$의 Prefix를 $S_2$에 넘긴 Table을 Commit하고 Broadcast한다. $S_2$의 Node는 $S_3$의 Honest Node에게서 최신 Block Header($A^{\text{UTXO}}_{S_3}$, $A^{\text{POI}}_{S_3}$, CC 포함)를 받고, $S_3$의 Transaction을 처리할 독립 SMR Instance를 시작한다.

6. **$T_{com}$ 처리**

    갱신된 Tx-to-shard Table에 따라 $T_{com}$은 $S_2$가 처리한다. $S_2$는 0.5 + 0.3 = 0.8, 두 PoI, 두 Non-membership Proof를 검증한 뒤 $T_{com}$을 Commit하고, $S_3$ 상태의 $A^{\text{UTXO}}$에 $O_3$를, $A^{\text{POI}}$에 두 PoI를 추가한다.

7. **Replay 시도**

    누군가 같은 $T_{com}$을 다시 보내면, 두 PoI가 이미 $A^{\text{POI}}$에 있으므로 Non-membership Proof가 검증을 통과하지 못해 거부된다.

8. **이후 B의 사용**

    B가 $O_3$를 쓰려면 $O_3$의 Membership Proof가 필요하다. 따라서 B는 명시적 상태를 직접 유지하거나 Semi-trusted 주체에 의존해야 한다.

만약 $S_2$도 Safe Shard여서 멈춘다면, 같은 과정으로 $S_2$가 닫히고 Transaction이 다시 다른 Shard로 넘어간다.

### 보안 속성

아래 성질은 Corrupt Shard가 없고 Reference Shard가 Honest Majority라는 조건에서 성립한다. 이 조건이 깨질 확률이 성능 분석의 프로토콜 실패 확률이다.

| 성질 | 내용 | 근거 |
| --- | --- | --- |
| Intra-shard Safety | Safe Shard와 Active Shard에서 Honest Node들은 같은 높이에 같은 Block을 Commit함 | $C_S < q$이므로 모든 Certificate에 Honest Node가 포함되고, 같은 높이에서 BC를 얻는 Block은 하나뿐임 |
| Intra-shard Liveness | Active Shard에 배정된 유효한 Transaction은 결국 Commit됨 | Honest Node끼리 Quorum을 만들 수 있으므로 Sync HotStuff의 Liveness를 따름 |
| Provable Commits | Commit된 모든 Block에는 Commit Proof가 있음 | CC 또는 CC가 있는 Block에서 이어지는 Hash Chain |
| Inter-shard Safety | 한 Shard가 T를 Commit하면, T와 같은 Output을 쓰는 T'를 어떤 Shard도 Commit할 수 없음 (Double Spending 불가) | 아래 두 보조 정리 |
| Inter-shard Liveness | 유효한 T는 배정된 Shard가 Commit하거나, 그 Shard가 닫히고 다른 Shard가 Commit함 | Safe Shard는 결국 처리하거나 닫히고, 높은 확률로 Active Shard가 적어도 하나 있으므로 T는 결국 Active Shard에 도달함 |

Inter-shard Safety의 두 보조 정리는 다음과 같다.

- **같은 UTXO에 대한 PoI는 하나뿐임**: UTXO O를 만든 Transaction의 Hash가 Tx-to-shard Table에서 PoI를 만들 Shard를 하나로 정하고, 그 Shard는 Intra-shard Safety에 따라 같은 UTXO에 대해 서로 다른 PoI를 만들지 않음
- **같은 PoI는 두 Commit Transaction에 쓰일 수 없음**: 같은 Epoch에서는 Hash Prefix에 따라 같은 Shard가 처리하므로 $A^{\text{POI}}$의 Non-membership Proof가 두 번째 처리를 막음. Epoch이 다르면 새 Shard가 이전 Shard가 관리하던 PoI Accumulator를 검사함

### 성능 분석

논문은 구현 없이 확률 분석으로 RapidChain과 비교한다. 비교하는 값은 Epoch의 실패 확률과 Epoch당 기대 Active Shard 수다.

**표기**

- N: Network 크기
- F: Network Resiliency, 즉 전체 Node 중 Corrupt Node의 비율
- $T = \lfloor N \times F \rfloor$: Corrupt Node 수
- n: Shard 크기
- $H_S$, $C_S$: Shard S의 Honest Node 수와 Corrupt Node 수 (Random Variable)
- m: Safety와 Liveness를 모두 만족하는 Shard 수

**RapidChain**

- 모든 Shard가 Honest Majority여야 하므로 $m_{Rapid} = N/n$
- Shard 실패 확률은 앞의 $\Pr[C_S \geq \lfloor n/2 \rfloor]$
- $Pr^{Rapid}_{fail} = \Pr[C_S \geq \lfloor n/2 \rfloor] \times N/n$ (Union Bound)

**Instachain**

$$
\Pr[H_S \geq q] = \sum_{i=q}^{n} \frac{\binom{T}{n-i}\binom{N-T}{i}}{\binom{N}{n}}, \qquad m_{Insta} = \Pr[H_S \geq q] \times N/n
$$

$$
\Pr[C_S \geq q] = \sum_{i=q}^{n} \frac{\binom{T}{i}\binom{N-T}{n-i}}{\binom{N}{n}}
$$

Reference Shard $S_{ref}$(크기 $n_{ref} > n$)는 RapidChain의 Shard처럼 Honest Majority여야 한다.

$$
\Pr[S_{ref} \text{ is corrupt}] = \sum_{i=\lfloor n_{ref}/2 \rfloor}^{n_{ref}} \frac{\binom{T}{i}\binom{N-T}{n_{ref}-i}}{\binom{N}{n_{ref}}}
$$

프로토콜은 Reference Shard나 Worker Shard 중 하나라도 실패하면 실패한다.

$$
Pr^{Insta}_{fail} = \Pr[S_{ref} \text{ is corrupt}] + \Pr[C_S \geq q] \times N/n
$$

**프로토콜 수준 비교**

- TTF(Time-to-Failure): 프로토콜이 실패할 때까지 걸리는 시간. Epoch 하나를 24시간으로 두고 계산
- RapidChain: $n = 200$ (Shard Resiliency 50%)
- Instachain: $n = 100$ (RapidChain의 절반), Reference Shard는 Worker Shard의 4배. q를 탐색해 RapidChain보다 TTF와 Active Shard 수가 모두 큰 값으로 $q = 60$을 찾음 (Shard Resiliency 60%)

| Protocol | N | F | n | Shard Resiliency | Expected TTF | Expected Active Shard 수 |
| --- | --- | --- | --- | --- | --- | --- |
| RapidChain | 1,000 | 33% | 200 | 50% | 17,128년 | 5 |
| RapidChain | 2,000 | 33% | 200 | 50% | 1,412년 | 10 |
| RapidChain | 4,000 | 33% | 200 | 50% | 312년 | 20 |
| RapidChain | 8,000 | 33% | 200 | 50% | 114년 | 40 |
| Instachain | 1,000 | 33% | 100 | 60% | 39,638년 | 9.4 |
| Instachain | 2,000 | 33% | 100 | 60% | 7,666년 | 18.8 |
| Instachain | 4,000 | 33% | 100 | 60% | 2,407년 | 37.5 |
| Instachain | 8,000 | 33% | 100 | 60% | 972년 | 74.8 |

- 모든 N에서 Instachain이 TTF가 더 길고 Active Shard 수도 약 2배임
- 예: $n = 100$, $q = 60$에서 Shard가 Active일 확률은 약 0.94 → $N = 2000$이면 $20 \times 0.94 = 18.8$
- 논문은 다른 Shard 크기와 Quorum 크기 조합으로 더 확장할 수 있을 것이라고 보지만, 이 값으로도 확장성을 보이기에 충분하다고 판단함

(검산: 표의 F는 33%로 적혀 있지만 수치는 $T = \lfloor N/3 \rfloor$일 때와 일치한다. 같은 가정으로 위 식을 다시 계산하면 RapidChain $N = 4000$은 약 321년, Instachain $N = 8000$은 약 979년이 나오고, 나머지 값은 논문과 일치한다.)

### 한계

- **구현과 실측 없음**: 처리량, 저장 공간, 지연 시간 같은 수치가 없고 확률 분석만 있음. 논문도 이 점을 인정하며 Prototype 구현을 향후 과제로 둠
- **작은 Shard의 비용**: Shard가 많아지면 Cross-shard Transaction이 늘고, Safe Shard를 감지하고 Transaction을 재배정하는 비용도 생김. 이 비용보다 병렬성 이득이 큰지 실험하는 것은 향후 과제로 남김
- **Safe Shard에 배정된 Transaction의 지연**: 감지와 Closing이 끝날 때까지 처리되지 않으며, 이어받은 Shard도 Safe Shard면 재배정이 연쇄될 수 있음
- **Atomicity 완화**: 한쪽 Input만 유효하면 Client가 Refund Transaction을 따로 만들어야 함
- **Client 부담**: Stateless 모델 때문에 Client가 명시적 상태를 유지하거나 Semi-trusted 주체에 의존해야 하고, Transaction에 Membership Proof가 붙어 크기가 커짐
- **가정**: Δ를 아는 Synchronous Network와 Epoch Adaptive Adversary를 가정한 결과임
- **자세히 나오지 않는 부분**: Epoch 사이 Reconfiguration에서 Shard 상태(Accumulator)를 새 Shard로 옮기는 구체적 절차

### 결론

Instachain은 Sharding에서 Shard 크기를 결정하던 조건, 즉 "모든 Shard가 Safety와 Liveness를 모두 만족해야 한다"는 조건을 완화했다. Sync HotStuff에서는 하나의 Quorum이 Safety 조건("Corrupt Node 수 < q")과 Liveness 조건("Honest Node 수 ≥ q")을 모두 Honest Majority로 묶었지만, Instachain은 q를 키워 Safety는 더 많은 Corrupt Node를 견디게 하고 Liveness는 일부 Shard에서만 보장한다. 그 결과 Shard 크기를 키우지 않고도 Shard 실패 확률을 낮출 수 있다. 대가로 생기는 Safe Shard는 Reference Shard, Safe Shard Detector, Stateless 모델 기반 상태 전달로 처리하고, 늘어나는 Cross-shard Transaction은 Client가 주도하는 Lock-free 방식과 PoI Accumulator로 처리한다. 논문은 이 구성이 Safety와 Liveness를 보장함을 증명하고, 확률 분석으로 RapidChain보다 작은 Shard에서 더 많은 Active Shard와 더 긴 TTF를 얻을 수 있음을 보였다. 다만 실제 처리량과 지연 시간 같은 구현 수준의 검증은 남아 있다.
