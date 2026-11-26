---
title: "Flexible Byzantine Fault Tolerance"
date: 2026-11-26 00:00:00 +0900
categories: [Paper Review]
tags: [Blockchain, Consensus, BFT]
math: true
---

### Paper Information

| Item | Details |
| --- | --- |
| Title | Flexible Byzantine Fault Tolerance |
| Authors | Dahlia Malkhi, Kartik Nayak, Ling Ren |
| Venue | 2019 ACM(Association for Computing Machinery) SIGSAC(Special Interest Group on Security, Audit and Control) Conference on Computer and Communications Security(CCS 2019), November 11-15, 2019 in London, UK at the Hilton Metropole, 1041-1053 |
| Link | [Conference Version](https://doi.org/10.1145/3319535.3354225), [arXiv Version](https://arxiv.org/abs/1904.10067) |

---

### TL;DR
 
기존 BFT 프로토콜은 관리자가 정한 하나의 가정(Byzantine 장애 비율, Synchrony 여부)을 모든 Replica와 Client에게 똑같이 적용한다. 그래서 Client마다 다른 가정을 쓸 수 없고, 모든 장애를 Byzantine으로 취급하므로 Resilience가 Partial Synchrony에서 1/3, Synchrony에서 1/2에 묶인다.
 
Flexible BFT에서 Replica는 투표로 Block을 Certify만 하고, Commit 여부는 Client가 자신의 가정으로 판단한다. Synchrony Client는 자신이 믿는 최대 지연 $\Delta$로, Partial Synchrony Client는 자신이 정한 투표 수 기준 $q_c$로 Commit하며, 가정이 맞는 Client끼리는 항상 같은 Block을 Commit한다. 여기에 Safety만 공격하고 Liveness는 돕는 Alive-but-corrupt Fault를 더해, Byzantine과 이 장애를 합친 비율이 Partial Synchrony에서 1/3, Synchrony에서 1/2를 넘어도 견딘다. 대신 견디는 전체 장애를 늘리면 견디는 Byzantine 비율은 줄어든다.

<br />

## BFT Consensus

### What is BFT Consensus?

**BFT(Byzantine Fault Tolerance) Consensus**는 일부 서버가 임의로 잘못 동작해도 나머지 서버들이 같은 결과에 합의하도록 만드는 프로토콜이다. 장애에 강한 복제 서비스를 만드는 데 쓰이며, 최근에는 Blockchain의 알고리즘 기반으로 다시 주목받았다.

- **State Machine Replication**: 여러 서버가 같은 요청을 같은 순서로 실행해, 사용자에게 장애 없는 서버 하나처럼 보이게 하는 방식. 모든 사용자는 같은 순서로 정렬된 값의 목록을 받음

- **Replica**: 복제 서비스를 구성하는 각 서버. 전체 Replica 수는 n

- **Client**: 서비스에 요청을 보내고 결과를 사용하는 사용자

- **Byzantine Fault**: Replica가 프로토콜을 따르지 않고 임의로 동작하는 장애. 잘못된 메시지를 보내거나, Replica마다 서로 다른 메시지를 보내거나, 응답하지 않을 수 있음

    - 프로토콜을 그대로 따르는 Replica는 Honest Replica라고 함

- **비율 표기**: 이 논문은 Replica 집합의 크기를 전체 n에 대한 비율로 나타낸다. 예를 들어 m개 Replica의 집합을 "q Replica"($q = m/n$)라고 부른다. 이 글도 같은 표기를 따른다.

#### Proposer, Acceptor, Learner

Lamport의 표기를 빌리면, 복제 서비스에는 세 가지 논리적 역할이 있다.

- **Proposer**: 새 값을 보냄

- **Acceptor**: 값을 정렬된 목록에 추가함. 이 목록을 Blockchain, 목록의 각 값을 Block이라고 함

- **Learner**: 프로토콜의 Transcript를 보고 값의 목록을 결정한 뒤, State Machine에서 실행함. Transcript는 프로토콜 진행 중 주고받은 메시지 기록

복제 서비스에서 Client는 Proposer이자 Learner이고, Replica는 Acceptor다. 따라서 Safety와 Liveness도 Client를 기준으로 정의한다.

#### Safety와 Liveness

- **Safety**: 어떤 두 Learner든 같은 값의 목록을 얻음

- **Liveness**: Proposer가 제안한 값은 결국 모든 Learner가 실행함

#### Network Model

- **Synchrony**: 모든 메시지가 알려진 상한 $\Delta$ 안에 도착함

- **Partial Synchrony**: 메시지 지연의 상한이 존재하지만 그 값을 모르거나, 알려진 상한이 알 수 없는 시점 이후에만 성립함 ([Dwork, Lynch, Stockmeyer](https://dl.acm.org/doi/10.1145/42282.42283))

#### Resilience

Resilience는 프로토콜이 Safety와 Liveness를 유지하면서 견딜 수 있는 장애 Replica의 비율이다. Resilience가 최적인 기존 프로토콜은 다음 범위에서 동작한다.

- **Partial Synchrony**: Byzantine Replica가 1/3 미만 (예: PBFT, HotStuff, Tendermint)

- **Synchrony**: Byzantine Replica가 1/2 미만 (예: Dfinity, Sync HotStuff)

Byzantine Replica가 이 한계를 넘거나, Synchrony 모델에서 $\Delta$가 지켜지지 않으면 Safety와 Liveness를 보장하지 못한다.

### Byzantine Quorum

**Quorum**은 다음 단계로 넘어가는 데 필요한 최소 Replica 집합, 즉 필요한 투표 수다. Partial Synchrony BFT 프로토콜은 Byzantine Quorum System의 두 성질로 Safety와 Liveness를 보장한다.

- **Quorum Intersection**: 어떤 두 Quorum이든 적어도 하나의 Honest Replica에서 겹침 → Safety

- **Quorum Availability**: Byzantine Replica를 포함하지 않는 Quorum이 존재함 → Liveness

Byzantine Replica가 1/3 미만이면 Quorum 크기를 2/3로 둔다.

- 정확히는 f개의 장애를 견디기 위해 $3f+1$개 중 $2f+1$개를 Quorum으로 사용

- Intersection: 두 Quorum의 교집합은 최소 $2(2f+1) - (3f+1) = f+1$개. Byzantine Replica는 최대 f개이므로 교집합에 Honest Replica가 적어도 하나 있음

- Availability: Honest Replica가 $2f+1$개이므로 Honest Replica만으로 Quorum을 만들 수 있음

### PBFT: Quorum으로 본 Partial Synchrony BFT

[PBFT(Practical Byzantine Fault Tolerance)](https://pmg.csail.mit.edu/papers/osdi99.pdf)는 Partial Synchrony 모델에서 1/3 미만의 Byzantine Replica를 견디는 대표 프로토콜이다. Flexible BFT 논문은 PBFT의 단계를 Lock 개념으로 단순화해, Quorum이 각 단계에서 어떤 역할을 하는지 보여 준다.

- **View**: 하나의 Leader가 합의를 이끄는 구간. Leader에 문제가 있으면 View Change로 다음 View에 넘어가 Leader를 바꿈

- **$q_r$**: Replica가 쓰는 Quorum 크기. PBFT에서는 2/3

- **Certificate**: 같은 값 b에 대한 $q_r$개 투표의 모음. $C^{q_r}(b)$로 표기

**동작 (View마다 반복)**

1. **Propose**: Leader L이 값 b를 제안

2. **Vote 1**: View v에서 처음 받은 값 b가 안전하면(아래 Lock 규칙) b에 투표. $q_r$개 투표가 모이면 Certificate $C^{q_r}(b)$가 됨

3. **Vote 2**: $C^{q_r}(b)$를 모은 Replica는 b에 **Lock**하고, $C^{q_r}(b)$에 투표

4. **Commit**: $C^{q_r}(b)$에 대한 투표가 $q_r$개 모이면, Client는 b가 Commit되었음을 앎

Vote 1은 PBFT의 Prepare 단계, Vote 2는 Commit 단계에 대응한다.

**Lock 규칙**: 어떤 View에서 b에 Lock한 Replica는 이후 View에서 b에만 투표한다. 단, 그 View 또는 더 높은 View에서 $q_r$개 Replica가 b에 Lock되어 있지 않다는 것을 확인하면 b에서 **Unlock**한다. 이 $q_r$개 Replica는 다른 값에 Lock되어 있거나, 아무 값에도 Lock되어 있지 않을 수 있다.

**Quorum이 Safety와 Liveness를 보장하는 방식**

- **View 안의 Quorum Intersection**: Replica는 View마다 한 번만 투표한다. 서로 다른 두 값이 모두 Certificate를 얻으려면 교집합의 Honest Replica가 두 값에 모두 투표해야 하므로 불가능하다. → Vote 1이 View 안의 Safety를 보장

- **View 사이의 Quorum Intersection**: b가 Commit되었다면 Quorum 크기의 Replica가 b에 Lock되어 있다. 그중 Honest Replica가 Unlock하려면 다른 Quorum이 b에 Lock되어 있지 않다고 주장해야 한다. 두 Quorum의 교집합에 있는 Honest Replica가 거짓으로 주장해야 하므로 불가능하다. → Lock이 View 사이의 Safety를 보장

- **View 안의 Quorum Availability**: Leader가 정상이면 Honest Replica만으로 Quorum이 응답한다. → View 안의 Liveness를 보장

**⇒ PBFT의 2/3 Quorum은 Certificate 생성(Lock), Unlock, Commit 판단에 모두 같은 크기로 쓰인다. 그러나 각 단계에서 Quorum이 맡는 역할은 서로 다르다.**

### Synchronous BFT

Synchrony 모델에서는 Byzantine Replica가 1/2 미만이면 합의할 수 있다. 초기 Synchrony 프로토콜은 두 가지 방식으로 Synchrony 가정에 의존했다.

1. Replica 사이의 최대 네트워크 지연 $\Delta$를 가정함

2. **Lock Step**: 모든 Replica가 같은 시각에 같은 라운드에 있어야 함

이후 프로토콜은 이 의존을 줄여 왔다.

- [Dfinity](https://arxiv.org/abs/1805.04548)(Hanke et al.): Lock Step을 없앴지만, 모든 Replica가 $2\Delta$ 동안 멈춰 기다리는 **Blocking Wait** 단계가 남음

- [Sync HotStuff](https://eprint.iacr.org/2019/270): Leader가 정직한 정상 구간에서는 Replica의 Blocking Wait를 없앴지만, View Change 같은 나쁜 상황에서는 여전히 Blocking Wait가 필요함

**⇒ 기존 Synchrony 프로토콜에서는 Replica가 프로토콜을 실행하는 과정에서 $\Delta$를 직접 사용한다.**

<br />

## Problem Definition

### Problem 1: 하나의 가정을 모든 참여자에게 적용함

기존 BFT 설계에서는 설계자나 서비스 관리자가 먼저 가정(Byzantine 장애 비율, 타이밍 가정)을 정하고, 그 가정에 맞는 프로토콜을 설계하거나 고른다. 이 가정은 모든 Replica와 모든 Client에게 똑같이 적용된다.

- 설계 가정과 다른 환경에서는 프로토콜이 무너진다. Partial Synchrony 프로토콜은 Byzantine Replica가 1/3을 넘으면, Synchrony 프로토콜은 1/2을 넘거나 $\Delta$가 지켜지지 않으면 Safety와 Liveness를 보장하지 못한다.

- 그러나 실제 Client의 가정은 서로 다를 수 있다.

    - 더 신중한 Client는 더 높은 Resilience를 원함

    - Synchrony를 믿는 Client도 있고, 믿지 않는 Client도 있음

    - 같은 Client도 100만 달러 거래와 5달러 거래에서 다른 가정을 할 수 있음. 금액이 크면 Double Spending을 위해 공모할 Replica가 늘어날 수 있으므로, Commit 전에 더 많은 투표를 기다리려 함

    - 관찰한 사건에 따라 가정을 바꿀 수 있음. 충돌하는 값에 대한 투표를 보면(Safety 공격 시도일 수 있음) 평소보다 많은 투표를 요구하고, 비정상적으로 긴 지연을 보면(네트워크 공격일 수 있음) $\Delta$를 늘리거나 Partial Synchrony 가정으로 바꿈

- 프로토콜 구조도 Client별 가정을 막는다.

    - Synchrony 프로토콜: Replica가 $\Delta$에 맞춰 Blocking Wait를 하면 $\Delta$는 프로토콜 전체가 공유하는 값이 된다. Client마다 다른 $\Delta$를 쓸 수 없다.

    - Partial Synchrony 프로토콜: 하나의 Quorum 크기를 모든 단계에 쓰므로, Client가 Commit 기준을 따로 정할 수 없다.

- 투표 수나 Synchrony 가정이 다른 두 가지 Commit Rule을 함께 두는 프로토콜(예: Zyzzyva, Sync HotStuff)도 있다. 하지만 이 경우에도 모든 Replica와 Client가 같은 가정을 하고 같은 Commit 결정을 내린다.

### Problem 2: Byzantine Fault 모델의 Resilience 한계

Byzantine Fault 모델은 장애 Replica가 Safety와 Liveness를 모두 공격할 수 있다고 가정한다. 그러나 공격자의 동기는 두 공격에서 다르다.

- **Safety 공격**: 이득이 생길 수 있음 (예: Double Spending)

- **Liveness 공격**: 대개 이득이 없음. 오히려 서비스가 계속 동작해야 수수료 같은 보상을 얻을 수 있음

그런데도 모든 장애를 Byzantine으로 취급하면, Resilience는 Partial Synchrony에서 1/3, Synchrony에서 1/2에 묶인다.

**⇒ 공격 동기를 반영한 장애 모델로 Byzantine 모델보다 높은 장애 비율을 견디고, 프로토콜과 가정을 분리해 Client마다 자신의 가정으로 Commit을 판단할 수 있게 하는 방법이 필요하다.**

<br />

## Solution: Flexible BFT

**Flexible BFT**는 Malkhi, Nayak, Ren이 제안한 BFT 프로토콜 설계 방식이다(2019). 두 가지 축으로 구성된다.

- **Stronger Resilience**: 새로운 장애 유형인 Alive-but-corrupt Fault(a-b-c)를 도입한다. Byzantine과 Alive-but-corrupt를 합친 장애 비율이 Partial Synchrony에서 1/3, Synchrony에서 1/2를 넘어도 견딘다. Byzantine 비율은 항상 각 모델의 한계보다 작으므로 기존 한계와 모순되지 않는다.

- **Diversity**: Replica가 만든 같은 Transcript를 Client마다 자신의 가정으로 해석해 Commit을 판단한다. 가정이 맞는 두 Client는 가정이 서로 달라도 같은 값을 Commit한다.

이를 위한 기술적 핵심은 두 가지다.

1. **Synchronous BFT with Network Speed Replicas**: Commit 단계만 $\Delta$를 알면 되는 Synchrony 프로토콜. Replica는 Synchrony 가정 없이 Network Speed, 즉 정해진 시간을 기다리지 않고 메시지가 도착하는 대로 진행함

2. **Flexible Byzantine Quorums**: 기존 프로토콜에서 Quorum이 맡는 역할을 나누어, Replica의 Quorum과 Client의 Quorum을 분리함

이 글에서 "전체 장애"는 Byzantine Replica와 Alive-but-corrupt Replica를 합친 비율이다.

| | 기존 BFT | Flexible BFT |
| --- | --- | --- |
| 가정 | 설계자나 관리자가 정한 하나의 가정을 모든 참여자에게 적용 | Replica는 $q_r$ 하나만 사용, Client는 각자 $q_c$ 또는 $\Delta$를 선택 |
| Commit 판단 | 모든 참여자가 같은 규칙 사용 | Replica는 값을 Certify만 하고, Commit은 Client가 각자 판단 |
| 장애 모델 | Byzantine | Byzantine + Alive-but-corrupt |
| Resilience (Partial Synchrony) | Byzantine < 1/3 | 전체 장애 < $q_c + q_r - 1$, Byzantine ≤ $1 - q_c$ |
| Resilience (Synchrony) | Byzantine < 1/2 | 전체 장애 < $q_r$, Byzantine ≤ $1 - q_r$ |
| Replica의 $\Delta$ 사용 | Synchrony 프로토콜의 Replica는 $\Delta$를 사용 | Replica는 $\Delta$를 사용하지 않음 |

($q_r$은 위 PBFT 설명에서 정의했고, $q_c$는 아래 Step 3에서 정의한다.)

### Flexible BFT Model

**Replica와 Client의 분리**

- Replica: 프로토콜을 실행해 제안된 값을 Certify함. Commit 여부는 판단하지 않음

- Client: Replica에게서 프로토콜 상태를 받아, 자신의 가정에 따라 Commit 여부를 판단함

**Safety와 Liveness의 재정의**

- **Safety for Diverse Learners**: 가정이 맞는(서로 다를 수 있는) 두 Learner는 같은 값의 목록을 얻음

- **Liveness for Diverse Learners**: 제안된 값은 결국 가정이 맞는 모든 Learner가 실행함

**Commit의 의미**

- 같은 값을 한 Client는 Commit하고 다른 Client는 아직 Commit하지 않은 상태가 가능하며, 흔하게 발생함

- 가정이 틀린 Client는 서로 충돌하는 값을 Commit할 수 있고, 그 Commit은 나중에 뒤집힐 수 있음

- 이는 기존 BFT에도 숨어 있던 동작이다. 기존 BFT에서 관리자의 가정이 깨지면(예: Byzantine Replica가 예상보다 많음) Client들이 서로 다른 값을 Commit할 수 있고, 되돌릴 방법이 없다. 이런 의미에서 Flexible BFT는 기존 BFT를 일반화한 것이다.

- 충돌하는 Commit을 발견한 Client는 가정을 더 보수적으로 바꾸고, 새 가정으로 Commit된 값을 다시 해석함

- Bitcoin과 비슷하다. Bitcoin에서 Client는 Block이 얼마나 깊이 쌓여야(Confirmation 수) Commit할지 스스로 정한다. Commit 후 더 긴 Fork가 나타나면 그 Commit은 뒤집히고, Client는 이후 더 많은 Confirmation을 요구할 수 있다.

**장애 모델**

- Byzantine Fault와 Alive-but-corrupt Fault (Step 1)

- **Static Adversary**: 어떤 Replica가 Byzantine이고 어떤 Replica가 Alive-but-corrupt인지는 프로토콜 시작 전에 정해짐

**기타 가정**

- Hash Function, Digital Signature, PKI(Public Key Infrastructure)

- $\langle x \rangle_R$: Replica R이 서명한 메시지 x

- Replica 사이에 1:1 통신 채널이 있음

- 모든 Replica의 시계는 같은 속도로 흐름

**표기**

- B: Byzantine Replica 비율

- P: Alive-but-corrupt Replica 비율

- 전체 장애: B + P

### Flexible BFT 단계별 동작 원리

#### Step 1: Alive-but-corrupt Fault

목적: 공격자의 동기를 장애 모델에 반영해, Byzantine 모델보다 높은 장애 비율을 견딤

**Alive-but-corrupt Fault(a-b-c)의 성질**

- Safety를 깨기 위해 프로토콜을 임의로 벗어날 수 있음

- Safety를 깰 수 없으면 Liveness를 방해하지 않고, 시스템이 (안전한) 결정에 도달하도록 도움

- 근거: Safety 위반은 이득이 될 수 있지만 Liveness 방해는 대개 이득이 없고, 서비스가 계속 동작하면 수수료 같은 보상을 얻을 수 있음

**증명에서 a-b-c Replica를 다루는 방식**

- Safety 증명: a-b-c Replica를 Byzantine처럼 취급

- Safety가 증명된 뒤 Liveness 증명: a-b-c Replica를 Honest처럼 취급

→ Safety 조건은 전체 장애(B + P)를 기준으로, Liveness 조건은 B만을 기준으로 정해진다.

**다른 장애 모델과의 비교**

- **Crash Fault와의 비교**: 기존에도 Byzantine Fault와 Crash Fault를 섞은 장애 모델이 있었다. Crash Fault는 응답을 멈출 뿐 Safety를 깨지 않는다. a-b-c는 반대로 Safety를 공격하고 Liveness는 유지하므로, 논문은 a-b-c 모델을 Byzantine과 "anti-crash"를 섞은 모델로 표현한다.

- **BAR(Byzantine, Altruistic, Rational) 모델의 Rational Replica**: 이익에 따라 움직인다는 점은 비슷하지만 다음이 다르다.

    - BAR는 Rational Replica끼리, 그리고 Rational과 Byzantine Replica 사이의 공모가 없다고 가정함. a-b-c에는 이런 제약이 없음

    - BAR는 부정행위를 드러내 억제하는 방식으로 설계됨. Flexible BFT는 억제에 의존하지 않음

    - BAR는 Partial Synchrony 모델만 다룸

    - BAR는 합리성을 Game Theory로 증명함. a-b-c 모델의 Incentive 분석은 열린 문제로 남음

#### Step 2: Synchronous BFT with Network Speed Replicas

목적: Replica는 Synchrony 가정 없이 Network Speed로 실행하고, Client만 자신의 $\Delta$로 Commit을 판단함

**→ 문제: Replica가 $\Delta$에 맞춰 기다리면 모든 Client가 같은 $\Delta$를 써야 함**

- 기존 Synchrony 프로토콜의 Replica는 Lock Step이나 Blocking Wait를 위해 $\Delta$를 사용함

**⇒ 해결: Replica는 $\Delta$ 없이 진행하면서 사건이 일어난 시각만 기록하고, $\Delta$를 이용한 Commit 판단은 Client가 함**

- Replica는 Blocking Wait 없이 Network Speed로 진행함. 사실상 Partial Synchrony 프로토콜을 실행하며, 어느 시점에도 Synchrony 가정에 의존하지 않음

- 논문은 이를 Replica와 Client의 Synchrony 가정을 분리한 첫 Synchrony 프로토콜로 소개함 (저자들이 아는 범위)

- 여기서는 하나의 값에 합의하는 Single-shot Consensus로 설명함. 최종 프로토콜에서는 연속된 값에 대해 Pipelining, 즉 앞 값의 투표 단계와 다음 값의 투표 단계를 겹쳐 진행하는 방식으로 확장함

**용어**

- **Equivocation**: Leader가 같은 View에서 서로 다른 값을 제안하는 것

- **Blame**: Leader가 잘못 동작했다고 판단한 Replica가 $\langle \text{blame}, v \rangle$ 메시지를 Broadcast하는 것

**Replica 프로토콜** (View마다 Leader 하나, 예: Round Robin으로 선택)

1. **Propose**: View v의 Leader L이 값 b를 제안

2. **Vote**: View v에서 처음 받은 값 b를 Broadcast하고, Lock 규칙상 안전하면 b에 투표함. 투표는 b를 다시 제안하는 역할도 함. 동시에 다음을 기록함

    - b에 대한 투표를 $q_r$개 모으면 Certificate $C^{q_r}_v(b)$가 됨 → b에 Lock하고, 그 시각을 $\text{t-lock}_v$로 기록

    - View v에 들어간 뒤 L이 서명한 다른 값을 발견하면 → 그 시각을 $\text{t-equiv}_v$로 기록하고, $\langle \text{blame}, v \rangle$과 두 값을 Broadcast해 Leader를 Blame

    - 충분한 시간 동안 제안을 받지 못하면 → Timeout, $\langle \text{blame}, v \rangle$을 Broadcast

    - $\langle \text{blame}, v \rangle$을 $q_r$개 모으면 → 그 시각을 $\text{t-viewchange}_v$로 기록하고, 모은 Blame을 Broadcast한 뒤 View v+1로 이동

- Lock 규칙은 PBFT와 같음: b에 Lock한 Replica는 이후 View에서 b에만 투표하고, $q_r$개 Replica가 b에 Lock되어 있지 않음을 확인해야 Unlock함

- Quorum 크기: 여기서는 $q_r = 1/2$ (정확히는 1/2보다 조금 큼. 예: $2f+1$개 중 $f+1$개)

**Client Commit Rule**

$\Delta$-Synchrony를 가정하는 Client는 $q_r$개 Replica가 각각 다음을 만족하는 View v가 있다고 보고하면 b를 Commit한다.

1. b가 Certified됨: $C^{q_r}_v(b)$가 존재

2. **Undisturbed-2Δ Period**: b가 Certified된 뒤 $2\Delta$가 지나기 전까지 Equivocation이나 View Change를 관찰하지 않음

$$
\min(\text{current-time},\ \text{t-equiv}_v,\ \text{t-viewchange}_v) - \text{t-lock}_v \geq 2\Delta
$$

- $\Delta$는 Client가 믿는 최대 네트워크 지연

- Replica는 $\Delta$를 쓰지 않고 시각만 기록함. 시각 차이로 $2\Delta$ 경과를 판단하므로, 모든 Replica의 시계 속도가 같다는 가정이 필요함

**Why 이 구조로 Safety가 보장되는가?** (장애 Replica 비율 < $q_r$)

- 장애 Replica가 $q_r$보다 적으므로, Commit 조건을 보고한 $q_r$개 Replica 중 적어도 하나는 Honest Replica다. 이를 h라 하고, h가 시각 t에 조건을 만족했다고 하자.

- 시간 흐름

    - h는 $t - 2\Delta$ 이전에 b에 투표(재제안)하고 Certificate를 모아 Lock함

    - 모든 Honest Replica는 $t - \Delta$ 전에 h로부터 b를 받음

    - 다른 Honest Replica도 $t - \Delta$ 전에는 Equivocation이나 View Change를 관찰하지 않음. 관찰했다면 그 Replica가 Broadcast한 메시지가 $t$ 전에 h에 도착해 h의 조건이 깨졌을 것

- **View 안의 Safety**: 어떤 Honest Replica가 $t - \Delta$ 전에 다른 값 b'을 먼저 받았다면, 그 Replica가 Broadcast한 b'이 $t$ 전에 h에 도착해 h가 Equivocation을 관찰했을 것이다. 따라서 모든 Honest Replica가 View v에서 처음 받은 값은 b이고, b 외의 값에는 투표하지 않는다. 장애 Replica가 $q_r$보다 적으므로 모든 Certificate에는 Honest Replica의 투표가 포함되고, b'의 Certificate는 만들어질 수 없다.

- **View 사이의 Safety**: h가 $t - 2\Delta$에 보낸 $C^{q_r}_v(b)$를 모든 Honest Replica가 $t - \Delta$까지 받고 b에 Lock한다. 이후 Unlock하려면 $q_r$개 Replica가 b에 Lock되어 있지 않다고 주장해야 한다. 그중 적어도 하나는 Honest Replica이므로 거짓 주장이 필요해 불가능하다.

- $2\Delta$인 이유: 첫 번째 $\Delta$는 h의 메시지가 모든 Honest Replica에 도착하는 시간, 두 번째 $\Delta$는 다른 Replica가 그 전에 보낸 충돌 메시지가 h에 도착하는 시간이다.

**Liveness**

- Byzantine Leader가 Equivocation하거나 안전한 값을 제안하지 않으면, Honest Replica와 a-b-c Replica가 모두 Blame하여 View Change가 일어남

- 결국 Honest 또는 a-b-c Replica가 Leader가 되어 합의를 이끔

- **Quorum Availability**: $B \leq 1 - q_r$이면 $q_r$개 Replica가 Leader에게 응답할 수 있음

**a-b-c Replica를 포함한 조건**

- a-b-c가 없을 때: Safety 조건(모든 $q_r$ 집합에 Honest Replica가 포함됨)은 $B < q_r$, Quorum Availability는 $B \leq 1 - q_r$. B를 최대로 하면 $q_r \geq 1/2$

- a-b-c를 포함할 때: Safety 조건은 $B + P < q_r$, Quorum Availability는 $B \leq 1 - q_r$

- $q_r > 1/2$로 정하면 전체 장애가 1/2를 넘어도 견딤. 예: $q_r = 2/3$이면 $B + P < 2/3$, $B \leq 1/3$

**Why 이 구조로 Step 2의 문제가 해결되는가?**

- Replica 프로토콜에는 $\Delta$가 없음 → Replica는 Network Speed로 실행

- Commit은 Replica가 아니라 Client가 함 → Client마다 자신의 $\Delta$로 Commit을 판단할 수 있음

- 올바른 $\Delta$를 쓴 Client는 Safety를 보장받음

#### Step 3: Flexible Byzantine Quorums

목적: Partial Synchrony 모델에서 Replica가 쓰는 Quorum과 Client가 Commit에 쓰는 Quorum을 분리함

**→ 문제: PBFT의 2/3 Quorum은 서로 다른 역할에 같은 크기로 쓰임**

- Certificate 생성(Lock), Unlock, Commit 판단(첫 번째 투표 수, 두 번째 투표 수)에 모두 같은 크기를 씀

- 모든 참여자가 같은 크기를 쓰므로 Client가 Commit 기준을 바꿀 수 없고, Byzantine Replica가 1/3을 넘으면 무너짐

**⇒ 해결: Quorum을 역할별로 나눔**

- $q_{lck}$: Certificate를 만드는(Lock) Quorum → Replica가 사용

- $q_{ulck}$: Unlock에 필요한 Quorum → Replica가 사용

- $q_{unq}$: Certificate가 유일함을 확인하는 Quorum (첫 번째 투표 수) → Client가 사용

- $q_{cmt}$: Commit이 안전함을 확인하는 Quorum (두 번째 투표 수) → Client가 사용

**수정된 PBFT형 프로토콜**

1. **Propose**: Leader L이 값 b를 제안

2. **Vote 1**: View v에서 처음 받은 값 b가 안전하면 투표. $q_{lck}$개 투표가 Certificate $C^{q_{lck}}(b)$가 됨

3. **Vote 2**: $C^{q_{lck}}(b)$를 모은 Replica는 b에 Lock하고 $C^{q_{lck}}(b)$에 투표

4. **Commit**: Client는 b에 대한 투표 $q_{unq}$개와 $C^{q_{lck}}(b)$에 대한 투표 $q_{cmt}$개를 모으면 b를 Commit

- Unlock: $q_{ulck}$개 Replica가 b에 Lock되어 있지 않음을 확인하면 Unlock

**Flexible Byzantine Quorum 조건**

- **(a) View 안의 Flexible Quorum Intersection**

    - PBFT와 달리 두 $q_{lck}$ Certificate가 Honest Replica에서 겹칠 필요가 없음. 즉 Replica들이 서로 충돌하는 값에 Lock하는 것도 허용함

    - 대신 모든 $q_{lck}$ Quorum과 모든 $q_{unq}$ Quorum이 적어도 하나의 Honest Replica에서 겹쳐야 함

    - 의미: Client가 b에 대한 첫 번째 투표를 $q_{unq}$개 봤다면, 같은 View에서 다른 값의 Certificate는 만들어질 수 없음 (교집합의 Honest Replica가 두 값에 모두 투표해야 하므로)

    - 조건: 장애 비율 < $q_{lck} + q_{unq} - 1$

- **(b) View 사이의 Flexible Quorum Intersection**

    - Client가 b를 Commit했다면 $q_{cmt}$개 Replica가 그 View에서 b에 Lock되어 있음

    - 그중 Honest Replica가 Unlock하려면 $q_{ulck}$개 Replica가 b에 Lock되어 있지 않다고 주장해야 함

    - 따라서 모든 $q_{ulck}$ Quorum과 모든 $q_{cmt}$ Quorum이 적어도 하나의 Honest Replica에서 겹쳐야 함

    - 조건: 장애 비율 < $q_{ulck} + q_{cmt} - 1$

    - [Flexible Paxos](https://arxiv.org/abs/1608.06696)는 Paxos가 View 안에서는 겹치지 않는 Quorum을 써도 되고, View 사이에서만 교집합이 필요하다는 점을 보였다. (b)는 이를 Byzantine과 a-b-c 환경으로 옮긴 조건으로 볼 수 있다.

- **Flexible Quorum Availability**: 각 단계의 Quorum을 모두 만들 수 있어야 하므로 $B \leq 1 - \max(q_{unq}, q_{cmt}, q_{lck}, q_{ulck})$

**정리**

- Safety: 장애 비율 < $\min(q_{unq} + q_{lck} - 1,\ q_{cmt} + q_{ulck} - 1)$

- Liveness: $B \leq 1 - \max(q_{unq}, q_{cmt}, q_{lck}, q_{ulck})$

**Quorum 크기 정하기**

- 균형 잡힌 크기 $q_{lck} = q_{ulck}$, $q_{unq} = q_{cmt}$가 최적이다. 균형이 맞지 않는 조합은 $(q_{cmt}, q_{ulck})$를 $(q_{unq}, q_{lck})$로 바꿔도 Safety와 Liveness 허용 범위가 줄지 않는다.

- 따라서 $q_{lck} = q_{ulck} = q_r$, $q_{unq} = q_{cmt} = q_c$로 둔다.

- $q_c \geq q_r$: Replica는 $q_r$개 투표로 Lock하고, Client도 이 투표를 Commit 판단에 그대로 쓸 수 있다. 조건식으로 보면, $q_c < q_r$일 때 Liveness 조건은 $B \leq 1 - q_r$로 $q_c = q_r$일 때와 같지만 Safety 조건($q_c + q_r - 1$)만 낮아진다.

조건은 다음 두 가지로 줄어든다.

- **Flexible Quorum Intersection**: 전체 장애 비율 < $q_c + q_r - 1$

- **Flexible Quorum Availability**: Byzantine 비율 ≤ $1 - q_c$

**Byzantine만 있을 때**

- 최선의 선택은 $q_c = q_r = 2/3$(정확히는 $3f+1$개 중 $2f+1$개)이고, Byzantine 1/3 미만을 견딤. $q_c = q_r$을 2/3보다 크게 하면 $B \leq 1 - q_r$이 1/3보다 작아짐

- 이때 Flexible Byzantine Quorum은 기존 Byzantine Quorum과 같아짐

**a-b-c Replica를 포함할 때**

- a-b-c는 Liveness를 공격하지 않으므로, 전체 장애 $q_c + q_r - 1$ 미만을 견딤. 이 값은 1/3보다 클 수 있음

- 예: $q_r = 0.7$, $q_c = 0.8$이면 전체 장애 0.5 미만, Byzantine 0.2 이하를 견딤 (n = 100이면 전체 장애 49개 이하, 그중 Byzantine 20개 이하)

**Client Commit Rule의 분리**

- Propose, Vote 1, Vote 2는 Replica가 실행하며 $q_r$만 사용함

- Commit은 Client마다 다른 $q_c$로 판단함. 고정된 $q_r$ 하나가 지원 가능한 Client의 범위를 정함

- 큰 $q_c$: 공격자의 많은 부분이 Liveness가 아니라 Safety를 공격한다고 믿는 Client. Liveness를 양보하고 더 강한 Safety를 얻음

- 작은 $q_c$: 공격자의 많은 부분이 Liveness를 공격한다고 믿는 Client

**Why 이 구조로 Step 3의 문제가 해결되는가?**

- Replica는 $q_r$ 하나로 실행하고 Client가 $q_c$를 고름 → 같은 프로토콜에서 서로 다른 장애 가정의 Client가 공존

- a-b-c를 반영해 전체 장애 허용이 1/3을 넘을 수 있고, Byzantine 비율은 여전히 기존 한계 안에 있음

### Final Flexible BFT Protocol

최종 프로토콜은 Step 2와 Step 3을 합쳐, 하나의 Replica 프로토콜로 두 종류의 Client를 동시에 지원한다.

- Partial Synchrony Client(Partial Synchrony를 가정하는 Client): $q_c$를 선택

- Synchrony Client(Synchrony를 가정하는 Client): $\Delta$를 선택

- Replica: 파라미터 $q_r$ 하나로 Network Speed 실행

- [Casper FFG(Casper the Friendly Finality Gadget)](https://arxiv.org/abs/1710.09437)와 [HotStuff](https://arxiv.org/abs/1803.05069)처럼 투표 라운드를 Pipelining으로 처리

#### 기본 개념

- **Block 구조**: Block은 연속된 값의 체인에서 각 값이다. 체인에서의 위치를 Height라고 함

    - $B_k := (b_k, h_{k-1})$

    - $b_k$: Height k에서 제안된 값

    - $h_{k-1} := H(B_{k-1})$: 이전 Block의 Hash

    - 첫 Block은 $B_1 := (b_1, \perp)$로 이전 Block이 없음

- **유효한 Block**: (i) 이전 Block이 유효하거나 $\perp$이고, (ii) 값이 애플리케이션의 유효성 조건을 만족하며 조상 Block들과 모순되지 않음 (예: 조상 Block에 담긴 Transaction을 Double Spending하지 않음)

- **확장**: $B_k$가 $B_l$의 조상이면 $B_l$이 $B_k$를 확장한다고 함 ($l > k$)

- **Block의 Equivocation**: 두 Block이 같지 않고 어느 쪽도 다른 쪽을 확장하지 않음

- **Certificate**: $C^{q_r}_v(B_k)$는 View v에서 $q_r$개 Replica가 $h_k = H(B_k)$에 한 서명의 모음. Certificate가 있는 Block을 Certified Block이라 함

- **Certified Block의 순위**: 먼저 Certified된 View로 비교하고, 같으면 Height로 비교함

- **Locked Block**: 각 Replica는 자신이 아는 가장 높은 순위의 Certified Block에 Lock함. 모든 Block의 서명을 추적하며 Locked Block을 계속 갱신함

#### Replica Protocol

Replica는 View 단위로 진행한다. Leader는 Round Robin처럼 고정된 방식이나 무작위로 정할 수 있으며, 논문의 설명은 Round Robin($v \bmod n$이 View v의 Leader)을 가정한다. Leader가 Block을 제안하고, Replica가 투표하며, $q_r$개 투표를 받은 Block은 Certified된다. Leader는 이 Block을 확장한 다음 Block을 제안해 체인을 이어 간다. **Replica는 Block을 Certify만 하고, Commit 판단은 Client에게 맡긴다.**

**Steady State**: View v의 Leader L에 대해 다음을 반복함

1. **Propose** (Leader): $\langle \text{propose}, B_k, v, C^{q_r}_{v'}(B_{k-1}), S \rangle_L$을 Broadcast

    - $B_k$는 L이 아는 가장 높은 Certified Block을 확장해야 함

    - 정상 상태의 Honest Leader는 직전에 자신이 제안한 Block을 확장함. 이때 $v' = v$, $S = \perp$

    - View Change 직후에는 S(Status 모음, 아래 View Change 참고)에서 가장 높은 Certified Block을 찾아 확장함

2. **Vote** (모든 Replica): 유효한 제안을 받으면, 다음 중 하나일 때 제안과 투표 $\langle \text{vote}, B_k, v \rangle_R$를 Broadcast

    - (i) View v의 첫 제안이고, S 안의 가장 높은 Certified Block을 확장함

    - (ii) 이 View에서 마지막으로 제안된 Block을 확장함

    - 조상 Block을 아직 받지 못했다면, 받을 때까지 기다려 유효성을 확인한 뒤 투표함

    동시에 Client의 Commit 판단에 쓸 값을 기록함

    - **투표 수** $q_{B_k,v}$: View v에서 $B_k$가 받은 투표 수. 모든 Replica가 투표를 Broadcast하므로 $q_r$보다 커질 수 있고, 새 투표를 받을 때마다 갱신함 → Partial Synchrony Client가 사용

    - **Lock 시각** $\text{t-lock}_{k-1,v}$: $B_{k-1}$이 같은 View v에서 제안된 Block이면 $B_{k-1}$에 Lock하고 그 시각을 기록함 → Synchrony Client가 사용

    - **Equivocation 시각** $\text{t-equiv}_{j,v}$: View v에서 Height j의 Equivocation Block을 제안이나 투표를 통해 관찰한 시각 → Synchrony Client가 사용

**Pipelining 해석**

- $B_k$의 제안에는 $C^{q_r}_v(B_{k-1})$이 들어 있음

- 따라서 $B_k$에 대한 투표는 $B_k$의 첫 번째 투표(Vote 1)이면서, $B_{k-1}$의 Certificate에 대한 두 번째 투표(Vote 2) 역할을 함

- Step 3의 Vote 1과 Vote 2가 연속된 Block에 겹쳐서 진행됨

**Leader Monitoring**

- 진행이 없거나, 같은 View에서 같은 Height의 Block이 둘 이상 제안되는 등 Leader의 악의적 동작을 발견하면 $\langle \text{blame}, v \rangle$을 Broadcast함

- View v를 떠나 그 View에서의 투표와 Block 전달을 멈춤

- 진행 여부는 메시지 도착 시간의 상한을 추측하거나, View마다 늘어나는 Timeout을 써서 판단함

**View Change** (View v의 Leader L, View v+1의 Leader L′)

1. **New-view**: $\langle \text{blame}, v \rangle$을 $q_r$개 모으면 이를 Broadcast하고 View v+1로 이동. 그 시각을 $\text{t-viewchange}_v$로 기록

2. **Status**: 자신의 Locked Block $B_j$에 대해 $\langle \text{status}, v, B_j, C^{q_r}_{v'}(B_j) \rangle$을 L′에게 보내고 바로 Steady State로 전환 ($v'$: $B_j$가 Certified된 View)

- L′은 $q_r$개 Status로 S를 만들고, 그중 가장 높은 Certified Block을 확장하는 Block을 첫 제안으로 보냄

- View Change에도 Blocking Wait가 없음 → Replica는 항상 Network Speed로 실행

#### Client Commit Rules

Client는 Replica에게서 프로토콜 상태를 받아, 자신의 가정에 따라 Block의 Commit 여부를 판단한다. Commit Rule은 두 가지이며, 논문은 이를 CR1, CR2(Commit Rule 1, 2)로 부른다.

**CR1: Partially-synchronous Commit (Partial Synchrony Client)**

Block $B_k$는 다음을 만족하는 $l \geq k$와 v가 있으면 파라미터 $q_c$로 Commit된다.

- (a) $C^{q_r}_v(B_l)$와 $C^{q_r}_v(B_{l+1})$이 존재하고, $B_{l+1}$은 $B_l$과 $B_k$를 확장함 ($l = k$이면 $B_l = B_k$)

- (b) $q_{B_l,v} \geq q_c$이고 $q_{B_{l+1},v} \geq q_c$

즉 연속된 두 Block이 같은 View에서 각각 $q_c$개 이상의 투표를 받으면, 앞 Block과 그 조상이 Commit된다.

**Why CR1이 안전한가?** (전체 장애 < $q_c + q_r - 1$)

- $B_l$이 View v에서 $q_c$개 투표를 받음 → $q_c$ Quorum과 $q_r$ Certificate의 교집합에 Honest Replica가 있으므로, 같은 View에서 충돌하는 Block은 Certified될 수 없음 (Step 3의 조건 (a))

- $B_{l+1}$도 같은 View에서 $q_c$개 투표를 받음 → $B_{l+1}$에 투표한 $q_c$개 Replica는 $C^{q_r}_v(B_l)$을 받았으므로 View v가 끝날 때 $B_l$(또는 $B_l$을 확장한 Block)에 Lock되어 있음

- 다음 View의 Leader가 보여 주는 S는 $q_r$개 Status로 구성됨. Lock한 $q_c$개 집합과의 교집합에 Honest Replica가 있으므로, S에는 반드시 $B_l$ 또는 $B_l$을 확장한 Block의 Certificate가 포함됨 (Step 3의 조건 (b))

- 귀납적으로 이후 View의 Certified Block도 모두 $B_l$과 같거나 $B_l$을 확장함

**CR2: Synchronous Commit (Synchrony Client)**

Block $B_k$는 $q_r$개 Replica 각각에 대해 다음을 만족하는 $l \geq k$와 v가 있으면 $\Delta$-Synchrony 가정으로 Commit된다. (l과 v는 Replica마다 달라도 됨)

- (a) $C^{q_r}_v(B_l)$가 존재하고, $B_l$은 $B_k$를 확장함 ($l = k$이면 $B_l = B_k$)

- (b) $B_{l+1}$을 받은 뒤 Undisturbed-2Δ Period를 관찰함

$$
\min(\text{current-time},\ \text{t-equiv}_{l,v},\ \text{t-viewchange}_v) - \text{t-lock}_{l,v} \geq 2\Delta
$$

- $\text{t-equiv}_{l,v}$: View v에서 $B_l$의 Equivocation을 관찰한 시각 (없으면 $\infty$)

- $\text{t-viewchange}_v$: View v에서 v+1로 View Change가 일어난 시각 (없으면 $\infty$)

- $\text{t-lock}_{l,v}$: View v에서 $B_{l+1}$의 제안을 받아 $B_l$에 Lock한 시각

- Replica는 Client가 정한 $\Delta$에 대해 이 조건을 만족하는지 진술함. 즉 $q_r$개 Replica가 "나쁜 사건(Equivocation 또는 View Change)이 $2\Delta$ 동안 없었다"고 함께 진술하는 것

**Why CR2가 안전한가?** ($\Delta$가 올바르고, 전체 장애 < $q_r$)

- 진술한 $q_r$개 중 적어도 하나는 Honest Replica h다. h가 시각 t에 $B_l$을 확장하는 $B_{l+1}$에 투표했고, $t + 2\Delta$까지 Equivocation이나 View Change를 관찰하지 않았다고 하자.

- Replica는 투표할 때 제안도 함께 전달하므로, $\Delta$-Synchrony가 성립하면 다른 Honest Replica는 h보다 최대 $\Delta$ 늦게 제안을 받는다.

- **View 안**: $B_l$과 Equivocation인 Block $B'_{l'}$($l' \geq l$)에 어떤 Honest Replica가 $t + \Delta$ 전에 투표했다면, 그 투표가 $t + 2\Delta$ 전에 h에 도착했을 것이다. $t + \Delta$ 이후에는 Honest Replica가 이미 h로부터 $B_l$을 받아 투표했으므로 $B'_{l'}$에 투표하지 않는다. 장애 Replica가 $q_r$보다 적으므로 $B'_{l'}$은 Certified될 수 없다.

- **View 사이**: h가 $t + 2\Delta$까지 View Change를 관찰하지 않았으므로, 모든 Honest Replica는 $t + \Delta$까지 View v에 있으면서 h로부터 $B_{l+1}$을 받고 $B_l$(또는 $B_l$을 확장한 Block)에 Lock한다. 다음 View의 S($q_r$개 Status)에는 Honest Replica가 포함되므로, $B_l$ 또는 $B_l$을 확장한 Block의 Certificate가 들어간다.

**Direct Commit과 Indirect Commit**

- **Direct Commit**: CR1에서는 Block과 바로 다음 Block이 같은 View에서 $q_c$개 투표를 받은 경우. CR2에서는 어떤 Honest Replica가 다음 Block을 받은 뒤 Undisturbed-2Δ Period를 보고한 경우

- **Indirect Commit**: Direct Commit 조건은 만족하지 않지만, 그 Block을 확장한 Block이 Direct Commit되면서 함께 Commit된 경우

- CR2의 Direct Commit은 증명을 위한 개념이다. Client는 어떤 Replica가 Honest인지 알 수 없으므로, 어떤 Block이 CR2로 Direct Commit되었는지 알 수 없다.

### Safety와 Liveness

여기서 Commit Rule이 "올바르다"는 것은 Client의 가정이 실제와 맞는다는 뜻이다.

- CR1($q_c$): 전체 장애 비율 < $q_c + q_r - 1$

- CR2($\Delta$): $\Delta$가 실제 최대 메시지 지연의 상한이고, 전체 장애 비율 < $q_r$

**Lemma (Direct Commit 이후의 Certified Block)**: Client가 올바른 Commit Rule로 View v에서 $B_l$을 Direct Commit했다면, $C^{q_r}_v(B_l)$보다 순위가 낮지 않은 모든 Certified Block은 $B_l$과 같거나 $B_l$을 확장한다.

- 순위가 낮지 않음: 같은 View v에서 Height가 l 이상이거나, 더 높은 View에서 Certified됨

- View에 대한 귀납법으로 증명함. 출발점은 View v 안에서 충돌 Block이 Certified될 수 없다는 점이고, 귀납 단계는 Lock과 S를 통해 이 성질이 다음 View로 이어진다는 점이다 (위 CR1, CR2의 안전성 설명)

**Theorem (Safety)**: 올바른 Commit Rule을 가진 두 Client는 모든 Height에서 같은 Block을 Commit한다.

- Height k에서 서로 다른 Block $B_k$와 $B'_k$가 Commit되었다고 가정

- 각각 View v의 $B_l$, View v′의 $B'_{l'}$이 Direct Commit된 결과라고 하고, 일반성을 잃지 않고 $v \leq v'$로 둠 (같은 View면 $l \leq l'$)

- 위 Lemma에 의해 $B'_{l'}$은 $B_l$과 같거나 $B_l$을 확장 → Height k의 Block도 같음 → 모순

- 두 Client가 서로 다른 Commit Rule(예: 하나는 CR1, 하나는 CR2)을 써도 성립함

**Theorem (Liveness)**: 모든 Client의 Commit Rule이 올바르면, 모든 Client가 새 Block을 계속 Commit한다.

- Safety가 증명되었으므로 a-b-c Replica를 Honest처럼 취급할 수 있음

- 올바른 Commit Rule은 Byzantine 비율을 최대 $1 - q_r$까지 견딤

- Byzantine Leader가 Liveness를 방해하면 $q_r$개 Blame이 모여 View Change가 일어남

- 결국 Byzantine이 아닌(Honest 또는 a-b-c) Replica가 Leader가 됨

- View마다 Timeout을 늘리면, 결국 Byzantine이 아닌 모든 Replica가 충분히 오래 같은 View에 머묾

- 두 조건이 모두 갖춰지면 Quorum Availability에 의해 Client는 같은 View에서 Commit에 필요한 투표를 얻음

### Efficiency

- **Latency**

    - Synchrony Client: $2\Delta$ + Network Speed 라운드 몇 번. $\Delta$ 기준으로 최신 Synchrony 프로토콜(Sync HotStuff)과 같음. 다만 $\Delta$가 Client마다 다르므로 Commit Latency도 Client마다 다름

    - Partial Synchrony Client: 투표 2라운드. PBFT와 같음

- **Communication**: 모든 투표와 New-view 메시지를 모든 Replica에게 Broadcast하므로 $O(n^2)$ 메시지. PBFT, Sync HotStuff와 같음

### Flexible BFT 동작 예시

아래 수치는 설명을 위해 정한 값이다.

- Replica: n = 100, $q_r = 0.7$ → Certificate에는 서명 70개가 필요

- Client 4명

    - A: Partial Synchrony, CR1, $q_c = 0.7$

    - B: Partial Synchrony, CR1, $q_c = 0.8$

    - C: Partial Synchrony, CR1, $q_c = 0.9$

    - D: Synchrony, CR2, $\Delta$ = 1초

1. **Propose와 Vote (View v, Leader L)**

    - L이 $B_1$을 제안하고, Replica는 제안을 전달하며 투표한다. 투표가 70개 모이면 $C^{q_r}_v(B_1)$이 만들어진다.

    - L이 $C^{q_r}_v(B_1)$과 함께 $B_2 := (b_2, H(B_1))$를 제안한다. Replica는 $B_2$에 투표하고, 같은 View에서 $B_2$를 받은 시각을 $\text{t-lock}_{1,v}$로 기록하며 $B_1$에 Lock한다.

    - Replica 15개가 투표하지 않아, 어떤 Replica가 집계한 투표 수가 $q_{B_1,v} = 85$, $q_{B_2,v} = 83$이라고 하자.

2. **Client별 Commit 판단**

    같은 투표 기록을 보고 Client마다 다른 결정을 내린다.

    | Client | 판단 | 이유 |
    | --- | --- | --- |
    | A ($q_c = 0.7$) | $B_1$ Commit | 같은 View에서 85 ≥ 70, 83 ≥ 70 |
    | B ($q_c = 0.8$) | $B_1$ Commit | 같은 View에서 85 ≥ 80, 83 ≥ 80 |
    | C ($q_c = 0.9$) | 아직 Commit하지 않음 | 85 < 90 |
    | D ($\Delta$ = 1초) | 2초 경과 진술이 70개 모이면 $B_1$ Commit | 70개 이상 Replica가 각자 기록한 $\text{t-lock}_{1,v}$ 이후 2초 동안 Equivocation과 View Change가 없었다고 진술 |

    - $\Delta$ = 3초를 믿는 Client라면 같은 기록을 보고, 70개 이상 Replica가 각자의 $\text{t-lock}_{1,v}$ 이후 6초 동안 문제가 없었다고 진술할 때 Commit한다.

    - C는 같은 View에서 연속된 두 Block이 각각 90개 이상의 투표를 받을 때까지 기다린다. 15개 Replica가 계속 투표하지 않으면 C는 Commit하지 못한다.

3. **실제 장애 구성에 따른 결과**

    실제로 Byzantine Replica가 15개(투표하지 않은 Replica), a-b-c Replica가 30개라고 하자. 전체 장애는 45개다.

    | Client | Safety 한도 (전체 장애) | Liveness 한도 (Byzantine) | 결과 |
    | --- | --- | --- | --- |
    | A | 39개 이하 → 45개라 위반 | 30개 이하 → 만족 | 가정이 틀림. Commit이 뒤집힐 수 있음 |
    | B | 49개 이하 → 만족 | 20개 이하 → 만족 | 가정이 맞음 |
    | C | 59개 이하 → 만족 | 10개 이하 → 15개라 위반 | Commit한 Block은 안전하지만, Commit하지 못할 수 있음 |
    | D | 69개 이하 → 만족 | 30개 이하 → 만족 | $\Delta$가 맞다면 가정이 맞음 |

    - A가 위험한 이유: 70표 Quorum 두 개의 교집합은 최소 40개다. 장애 Replica가 45개면 교집합 전체가 장애 Replica일 수 있어, 장애 Replica가 양쪽에 모두 투표하면 서로 충돌하는 Block이 같은 View에서 각각 70표 이상을 받을 수 있다. a-b-c Replica는 이를 이용해 A를 대상으로 Double Spending을 노릴 수 있다.

    - B가 안전한 이유: 80표 집합과 70표 집합의 교집합은 최소 50개다. 장애 Replica는 45개이므로 교집합에 Honest Replica가 반드시 있다.

    - B와 D는 가정이 다르지만 둘 다 맞으므로, 같은 Height에서 같은 Block을 Commit한다 (Safety Theorem).

    - 논문의 Liveness 증명은 모든 Client의 가정이 맞다는 전제에서 이루어진다. A처럼 가정이 틀린 Client가 있을 때 a-b-c Replica가 계속 Liveness를 돕는지는 이 논문이 다루지 않는다 (아래 한계 참고).

### 파라미터 선택

Flexible BFT는 세 파라미터로 정해진다.

- $q_r$: Replica의 유일한 파라미터. 서비스 관리자가 정하며, 지원 가능한 Client 가정의 범위를 결정함

- $q_c$, $\Delta$: Client가 Block을 Commit할 때 각자 정함

#### $q_r = 2/3$에서 지원되는 Client

| Commit Rule | 파라미터 | Safety: 전체 장애 | Liveness: Byzantine |
| --- | --- | --- | --- |
| CR1 | $q_c = 2/3$ | < 1/3 | ≤ 1/3 |
| CR1 | $q_c = 3/4$ | < 5/12 | ≤ 1/4 |
| CR1 | $q_c = 5/6$ | < 1/2 | ≤ 1/6 |
| CR1 | $q_c = 1$ | < 2/3 | 0 |
| CR2 | $\Delta$ (올바른 값) | < 2/3 | ≤ 1/3 |

- CR1에서 $q_c = 2/3$은 기존 Partial Synchrony 프로토콜과 같은 설정(Byzantine 1/3, 전체 장애 1/3)이다.

- CR1에서 $q_c$를 키우면, 투표를 하나 더 요구할 때마다 견디는 Byzantine Replica는 하나 줄고 전체 장애는 하나 늘어난다. 즉 a-b-c Replica는 두 개 더 견딘다.

- CR2는 투표 수를 파라미터로 쓰지 않으므로, 올바른 $\Delta$를 쓰는 Client는 Byzantine 1/3과 전체 장애 2/3을 함께 견딘다.

#### Client가 Commit Rule을 고르는 방법

Client가 견디려는 Byzantine 비율을 B, 전체 장애 비율을 T(= B + P)라 하자. 위 조건식을 정리하면 다음과 같다.

- **CR1이 가능한 경우**: $B + T < q_r$이고 $B \leq 1 - q_r$

    - $q_c$는 $q_c \geq q_r$이면서 $T + 1 - q_r < q_c \leq 1 - B$인 범위에서 고름

    - 예: $q_r = 0.7$에서 (B, T) = (0.2, 0.49)이면 $0.79 < q_c \leq 0.8$ → $q_c = 0.8$

- **CR2가 가능한 경우** (Synchrony를 믿고 $\Delta$가 올바를 때): $T < q_r$이고 $B \leq 1 - q_r$

- 둘 다 아니면 이 $q_r$로는 그 가정을 지원할 수 없음

#### 가정이 틀린 Client와 복구

- **Synchrony Client**: $\Delta$를 너무 작게 잡고 b를 Commit하면, 충돌하는 b′도 Certified될 수 있다. Replica가 b′ 쪽 체인을 확장하면 b는 뒤집히고 Safety가 깨진다. 이를 발견한 Client는 일부 Commit을 되돌리고 $\Delta$를 늘린다.

- **Partial Synchrony Client**: Safety가 깨지면 B를 줄이고 T를 늘리는 쪽($q_c$를 키우는 쪽)으로 가정을 바꾼다. $q_c$를 채우지 못해 진행이 없으면 반대쪽($q_c$를 줄이는 쪽)으로 바꾼다.

- 실제 장애가 CR2로만 지원되는 범위($T < q_r$, $B \leq 1 - q_r$이면서 $B + T \geq q_r$)에 있으면, Safety와 Liveness를 모두 만족하는 CR1은 없다. 이 Client는 결국 CR2로 바꿔야 한다.

- a-b-c Replica는 가정이 틀린 Client를 노려 이득(예: Double Spending)을 얻으려 할 수 있다. Client가 올바른 가정으로 바꾸고 잘못된 Commit에서 복구하면, 이후의 Commit은 안전하고 최종적이다.

#### $q_r$ 선택 (서비스 관리자)

- **Partial Synchrony Client 기준**: $q_r \geq 2/3$이 $q_r < 2/3$보다 항상 낫다. $B < q_c + q_r - 1$과 $B \leq 1 - q_c$를 더하면 $B < q_r/2$가 되므로, $q_r$이 작을수록 Byzantine 허용이 줄어든다. 같은 $q_c$에서 전체 장애 허용($q_c + q_r - 1$)도 줄어든다.

- **$q_r \geq 2/3$ 범위, 그리고 Synchrony Client 기준**: 모든 면에서 더 나은 $q_r$ 값은 없다.

    - Partial Synchrony: $q_r$을 키우면 전체 장애 허용(Safety)은 늘지만, $q_c \geq q_r$이므로 $B \leq 1 - q_r$이 되어 Byzantine 허용(Liveness)은 줄어든다.

    - Synchrony: Safety는 전체 장애 < $q_r$, Liveness는 $B \leq 1 - q_r$이다.

    - 즉 $q_r$ 선택은 Safety와 Liveness 사이의 Trade-off다.

#### a-b-c 허용과 Diversity의 분리

두 성질은 서로를 강화하지만, 따로 제공할 수도 있다.

- **a-b-c 허용만**: 기존의 균일한 설정에서도 PBFT에 더 큰 Quorum을 쓰면 된다. 하지만 전체 장애 허용을 높이면 Byzantine 허용이 낮아지고, 균일한 설정에서는 모든 Client가 이를 감수해야 한다. Flexible BFT에서는 Client가 가정을 고를 수 있고, 기존 Byzantine 모델을 고를 수도 있다.

- **Diversity만** (a-b-c 없이): Diversity의 범위가 줄어든다. 예를 들어 Step 2의 프로토콜은 서로 다른 $\Delta$를 믿는 Client를 지원한다. Partial Synchrony Client는 a-b-c 외에 다른 Diversity가 발견되지 않아 균일하다.

### 한계

- **Liveness 증명의 전제**: 모든 Client의 Commit Rule이 올바르다고 가정한다. a-b-c 모델은 일부 Client의 Safety를 깰 수 있는 상황에서 a-b-c Replica가 어떻게 행동할지 정하지 않으며, 이때 Liveness 돕기를 멈출 수도 있다. 저자들은 Rational 모델에서는 가정이 틀린 Client의 Safety를 공격하면서 올바른 Client의 Liveness는 유지하는 것이 a-b-c Replica의 최선 전략일 것으로 보지만, 이 분석은 향후 과제로 남겼다.

- **시계 가정**: 모든 Replica의 시계가 같은 속도로 흐른다고 가정한다. Clock Drift에 대한 대응은 향후 과제다.

- **Incentive 분석**: a-b-c 모델에서의 Incentive 분석은 열린 문제다.

- **Byzantine 허용과의 Trade-off**: Byzantine 허용 한계 자체를 넘지는 않는다. 기존 한계를 넘는 것은 전체 장애 허용이며, 전체 장애 허용을 높이면 그만큼 Byzantine 허용이 낮아진다.

- **가정이 틀린 Client**: Commit이 뒤집힐 수 있고, a-b-c Replica가 이를 노릴 수 있다.

### 결론

Flexible BFT는 BFT 프로토콜과 장애 가정을 분리해, 서로 다른 가정을 가진 Client가 같은 Ledger를 쓰게 하는 설계 방식이다. Replica는 Block을 Certify만 하고, Commit은 Client가 자신의 가정($q_c$ 또는 $\Delta$)으로 판단한다. 가정이 맞는 Client는 가정이 서로 달라도 같은 Block을 Commit한다.

이를 위해 두 기법을 쓴다. Synchrony 모델에서는 Replica가 Network Speed로 실행하고 Commit Rule만 $\Delta$를 쓰는 프로토콜을 만들었다. Partial Synchrony 모델에서는 기존 프로토콜의 Quorum을 역할별로 나눈 Flexible Byzantine Quorums를 도입해, Replica의 $q_r$과 Client의 $q_c$를 분리했다. 여기에 Safety만 공격하는 Alive-but-corrupt Fault를 더해, 전체 장애(Byzantine + a-b-c)가 Partial Synchrony에서 1/3, Synchrony에서 1/2를 넘어도 견딘다. Byzantine 비율은 기존 한계 안에 머물며, 전체 장애 허용과 Byzantine 허용은 Client가 고르는 Trade-off 관계에 있다.