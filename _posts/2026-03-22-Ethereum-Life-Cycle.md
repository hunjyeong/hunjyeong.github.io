---
title: "Ethereum Life Cycle"
date: 2026-03-22 12:24:00 +0900
categories: [blockchain, Ethereum]
tags: [github Pages, githubBlog, blog, velog, github, blockchain]
---


이더리움(Ethereum)은 블록과 트랜잭션 데이터를 검증할 수 있는 소프트웨어를 실행하는 컴퓨터들로 구성된 분산 네트워크이다. 컴퓨터가 노드가 되려면 소프트웨어(클라이언트)를 실행해야 한다.

[https://ethereum.org/developers/docs/nodes-and-clients/#:~:text=Simplified diagram of a coupled execution and consensus client](https://ethereum.org/developers/docs/nodes-and-clients/#:~:text=Simplified%20diagram%20of%20a%20coupled%20execution%20and%20consensus%20client).

### What are Nodes?

노드는 이더리움 네트워크 참여자로, 블록과 트랜잭션을 검증, 저장, 전파하며, 클라이언트 소프트웨어를 실행하는 하드웨어(컴퓨터) 인스턴스이다. 노드는 Full node, Archive node, Light node로 나뉜다.

- Full node: 전체 블록체인 데이터 저장, 네트워크 동기화 지원
- Archive node: 풀 노드에 저장된 모든 데이터 저장, 과거 상태 기록 보관
- Light node: 블록 헤더만 저장, 경량화된 검증

### What are Clients?

클라이언트는 이더리움 프로토콜을 실제로 구현한 소프트웨어 프로그램(예: Geth, Prysm 등)을 말한다. PoS(Proof-of-Stake) 환경에서 완전한 Ethereum 노드를 운영하려면 Execution Client와 Consensus Client를 함께 실행해야 한다.

- **Execution Client** (EL client, Execution Engine, …)
    - Execution Layer(실행 계층)를 실제로 구현한 프로그램이다.
        - 예: Geth, Nethermind, Erigon, Besu 등
    - 네트워크에서 트랜잭션을 수신하고 EVM에서 실행한다.
    - 계정 잔고, 스토리지, 상태 트리 등 최신 이더리움 상태와 데이터베이스를 관리한다.
    - JSON‑RPC API를 제공해서 외부(지갑, 페이지 등)에서 상태 조회·트랜잭션 전송이 가능하게 한다.
    - Consensus Client의 요청(Engine API)에 따라 Execution Payload를 생성 및 검증한다.

- **Consensus Client** (CL client, Beacon Node, …)
    - Consensus Layer(합의 계층)와 PoS 합의 알고리즘을 구현한 프로그램이다.
        - 예: Prysm, Lighthouse, Teku, Nimbus 등
    - 슬롯 기반으로 블록 제안자(Proposer)와 검증자 위원회(Committee)를 선정한다.
    - 검증자의 투표(Attestation)를 수집하고 포크 선택 규칙(LMD-GHOST)을 수행한다.
    - Execution Client와 상호작용하여 블록의 실행 결과를 검증한다.
    - Casper FFG 규칙에 따라 체크포인트를 **정당화(Justification)** 및 **완결(Finalization)** 한다.

실제로 트랜잭션이 네트워크에 들어와서 블록에 포함되고 확정되기까지 이더리움 트랜잭션의 라이프 사이클은 다음과 같이 진행된다.

## 1. Transaction Origination and Signing

사용자가 트랜잭션을 네트워크에 제출하기 전, 로컬 환경(지갑 앱, DApp 등)에서 수행하는 단계이다.

### 1.1 트랜잭션 객체 생성 (Transaction Object Construction)

사용자(EOA)가 지갑 소프트웨어(예: MetaMask)에서 트랜잭션을 생성한다.

```tsx
{
  from: "0xEA674fdDe714fd979de3EdF0F56AA9716B898ec8",
  to: "0xac03bb73b6a9e108530aff4df5077c2b3d481e5a",
  gasLimit: "21000",
  maxFeePerGas: "300",
  maxPriorityFeePerGas: "10",
  nonce: "0",
  value: "10000000000"
}
```

- from: 거래에 서명할 발신자 주소
- to: 수신자 주소 (EOA 또는 스마트 컨트랙트 주소)
- gasLimit: 이 트랜잭션이 소모할 수 있는 가스의 최대 한도
- maxFeePerGas (EIP-1559): 지불할 가스당 최대 요금
    - maxFeePerGas = Base Fee + Priority Fee
    - Base Fee는 블록당 소각되고 Priority Fee는 Proposer에게 지급됨
- maxPriorityFeePerGas (EIP-1559): 블록 제안자(Proposer)에게 팁으로 지불할 가스당 최대 요금
- nonce: 발신자 주소에서 보낸 트랜잭션의 총 횟수(리플레이 공격 방지)
- value: 전송할 ETH의 양
- data: 스마트 컨트랙트와 상호작용할 경우, 실행할 함수와 인코딩된 파라미터

### 1.2 계정 상태 조회 및 Fee 추정 (Account State Query & Fee Estimation)

트랜잭션 생성 전 지갑은 RPC 노드에 연결해 상태 조회 및 fee를 추정한다. 사용자가 Gas 파라미터를 수정할 때마다 실시간 시뮬레이션(eth_call, eth_estimateGas)을 통해 예상 비용과 성공/실패 가능성을 보여준다.

```tsx
// 실제 RPC 호출 과정
1. eth_getTransactionCount(address, "pending") → nonce 조회
2. eth_feeHistory() 또는 eth_getBlockByNumber("latest") → base fee 확인  
3. eth_maxPriorityFeePerGas() → priority fee 제안
4. eth_estimateGas({tx}) → gasLimit 추정
5. eth_call({tx}, "latest") → 시뮬레이션 실행 (실패 여부 확인)
```

### 1.3 개인키 기반 트랜잭션 서명 (Transaction Signing)

먼저 서명 전, 트랜잭션 객체 `{nonce, gasLimit, to, value, data, maxFeePerGas, maxPriorityFeePerGas...}`를 RLP 인코딩하여 바이너리 데이터로 변환 후, keccak256 해시를 통해 32bytes의 messageHash를 생성한다. 

- RLP(Recursive Length Prefix): 이더리움에서 사용하는 데이터 직렬화(serialization) 형식. 트랜잭션 필드(nonce, gasPrice 등), 블록 헤더(stateRoot, txRoot 등)를 인코딩하며, 첫 번째 prefix 바이트에 길이와 타입 정보를 포함한다.
- 해시 역할: 100바이트 이상의 TX를 32bytes로 압축 + 변조 불가능성 보장

그 다음으로, messageHash를 secp256k1 개인키로 ECDSA 서명하여 (r, s, v)를 생성한다.

- r: 곡선 점 x좌표 (32bytes)
- s: 서명값 (32bytes)
- v: 복구 식별자 (27/28 또는 체인ID 기반)

서명 후 최종 RLP 인코딩을 통해 Signed Raw Transaction을 만든다.

```tsx
RLP([nonce, gasLimit, to, value, data, maxFeePerGas, maxPriorityFeePerGas, v, r, s])
```

### 1.4 노드로 트랜잭션 전송 (Transaction Submission to Node)

Signed Raw Transaction을 `eth_sendRawTransaction` JSON-RPC 호출을 통해 이더리움 EL Client에게 전송한다. 전송 대상은 사용자가 직접 운영하는 노드(Geth, Nethermind 등 EL Client) 혹은 노드 서비스 제공자(Infura, Alchemy, QuickNode 등 RPC Endpoint)이다.

## 2. Mempool Propagation and Block Construction

서명된 트랜잭션이 EL 노드에 도착한 후 블록에 포함되기까지 대기하는 단계이다. EL 노드는 먼저 Mempool 검증을 통과한 트랜잭션만 Mempool에 저장하고 P2P 네트워크로 전파한다. 

### 2.1 트랜잭션 1차 검증 (Initial Mempool Validation)

[`eth_sendRawTransaction`](https://ethereum.org/developers/docs/apis/json-rpc/#eth_sendrawtransaction) 응답으로 트랜잭션 해시가 반환되고, 노드는 즉시 Mempool 검증을 수행한다. 다음을 순차적으로 검증한다.

1. **서명 유효성 검증:** 
    
    네트워크에서 서명이 유효한지 검증하는 방법은 다음과 같다.
    
    1. 서명 전 RLP 재생성(v,r,s 제외)
        - RLP([nonce, gasLimit, to, value, data, maxFeePerGas, maxPriorityFeePerGas]) → keccak256 → messageHash
    2. 발신자 주소 복구
        - messageHash + (v,r,s) → ECDSA_recover() ⇒ 공개키 후보 4개 생성
        - v값으로 1개 선택 → 공개키 확정
        - keccak256()[12:] → 마지막 20bytes가 주소임
    3. 서명 검증
        - 복구된 공개키로 messageHash 서명 검증 → 유효성 확인
    
    ⇒ 이때 서명 검증이 필요한 이유는 1) 트랜잭션 필드에 from 주소가 없기 때문에 r,s,v 서명으로 발신자 주소를 복구하여 “이 사람이 정말 해당 주소의 주인인가”를 확인한다. 2) v값에 체인 ID를 포함하여 Mainnet, Testnet 등 체인별로 서명이 달라져 같은 트랜잭션을 다른 체인에 재전송하는 공격(Replay Attack)을 방지한다. 3) 트랜잭션 내용을 조금만 변경해도 RLP 해시가 변경되므로 데이터 무결성도 검증된다. 즉, 개인키 없이 누구나 검증 가능하지만, 서명 생성은 개인키 소유자만 가능한 구조이다.
    
2. **Nonce 검증:**
    - 트랜잭션의 논스가 발신자 계정의 현재 논스와 일치하는지 확인한다.
    - 현재 계정의 nonce = 5인 경우 예시
        - `tx.nonce = 5` → 정상 ⇒ 진행
        - `tx.nonce < 5` → 중복/재전송 ⇒ 거부
        - `tx.nonce > 5` → 미래 TX ⇒ 별도 Future Queue 저장
3. **Balance 검증:**
    - 발신자가 ETH = value + (gasLimit × maxFeePerGas) 만큼의 잔고를 보유하고 있는지 확인한다.
    - StateDB를 통해 실시간으로 잔고를 조회할 수 있다.
4. **GasLimit 검증:**
    - 21,000 ≤ gasLimit ≤ 블록 가스 리밋(30M)
        - 너무 작음 → 실행 실패 예상 ⇒ 거부
        - 너무 큼 → DoS 공격 의심 ⇒ 거부

### 2.2 Mempool 등록 (Mempool Admission)

1차 검증을 모두 통과한 트랜잭션은 노드의 로컬 Mempool에 다음과 같은 과정을 통해 추가된다. 

1. 트랜잭션 해시 계산
    - RLP([...v,r,s]) → keccak256 → TX_HASH (0x1234...)
2. Nonce Queue 분류
    - `tx.nonce == account.nonce` → Pending Queue
    - `tx.nonce > account.nonce` → Future Queue
3. Replacement 체크
    - 기존_TX.tip < 새_TX.tip → 교체 (EIP-1559)
    - 기존_TX.tip ≥ 새_TX.tip → 유지
4. 높은 tip 우선 정렬 
    - maxPriorityFeePerGas 내림차순

```tsx
초기 상태:
Mempool
├── Pending: 빈 상태
├── Future: 빈 상태
└── Local: 빈 상태

1. TX1 (nonce=5, tip=20gwei) 도착 → 1차 검증 통과
   ↓
Pending: [TX1 (nonce=5, tip=20gwei)]

2. TX2 (nonce=6, tip=15gwei) 도착 → 현재 nonce=5이므로 Future
   ↓
Pending: [TX1]
Future: [TX2 (nonce=6, tip=15gwei)]

3. TX3 (nonce=5, tip=25gwei) 도착 → TX1 동일 nonce, tip↑ → **Replacement**
   ↓
Pending: [TX3 (nonce=5, tip=25gwei)] ← TX1 삭제
Future: [TX2]

4. Proposer가 TX3 실행 → 블록에 포함
   ↓
Pending: [빈 상태]
Future: [TX2] ← 자동 Pending 승격
   ↓ 현재 nonce=6
Pending: [TX2 (nonce=6, tip=15gwei)]
```

- Local TX: 해당 노드에서 직접 받은 TX (Private RPC 등), 다른 노드로 전파 안함
- Public TX: P2P로 전파되어 네트워크 전체 Mempool에 공유

### 2.3 P2P Gossip 기반 전파 (P2P Gossip Propagation)

Mempool에 추가된 트랜잭션은 P2P Gossip 프로토콜을 통해 이웃 노드에 전파된다. 이때 전체 네트워크에 보내는게 아니라, 가까운 이웃 노드에게 보내고 그 이웃들이 다시 자신의 이웃에게 전달하는 Flooding 방식이다. 1-2초 내 네트워크 전체가 거의 동일한 Pending TX 풀을 공유하게 되므로, 모든 Builder가 동일한 최적 트랜잭션 세트를 보고 블록을 효율적으로 구성할 수 있다.

- **Private RPC (MEV 방지):** MEV 공격(샌드위치 공격 등)을 방지하려는 사용자는 공용 멤풀이 아닌, Flashbots Protect RPC 등 Private Relay를 통해 빌더(Builder)에게 직접 트랜잭션을 전달하기도 한다.


## 3. Block Proposal

이더리움은 효율적인 MEV 처리와 검증자 중앙화를 완화하기 위해 [PBS(Proposer-Builder Separation)](https://ethereum.org/roadmap/pbs/#proposer-builder-separation) 구조를 도입하였다. PBS에서는 블록을 구성하는 역할을 수행하는 블록 빌더(Builder)와 블록을 네트워크에 제안하는 제안자(Proposer)의 역할이 분리된다.

블록 빌더는 Mempool의 트랜잭션을 이용해 Execution Payload를 포함한 블록 후보를 생성하고, 해당 블록이 가져올 수 있는 예상 수익과 함께 제안자에게 입찰(Bid)을 제출한다. 제안자는 여러 빌더가 제출한 입찰 중 가장 높은 보상을 제공하는 블록을 선택한다. 이 과정에서 제안자는 블록의 전체 실행 내용을 사전에 신뢰할 필요 없이 블록 헤더와 입찰 정보만을 기반으로 선택을 수행하며, 선택 이후 빌더로부터 Execution Payload를 전달받아 이를 검증한 뒤 네트워크에 블록을 전파한다.

### 3.1 블록 제안자 선정 (Proposer Selection)

이더리움 시간 단위는 12초당 1개의 슬롯(Slot)으로 나뉘며, 32개의 슬롯이 모여 1개의 에포크(Epoch)를 구성한다. 각 에포크가 시작되기 직전, 다음 에포크의 모든 슬롯에 배정될 제안자(Proposer)가 미리 결정된다.

제안자는 활성화된 검증자(Validator) 세트 중에서 무작위로 추출된다. 이때, Beacon Chain 프로토콜은 RANDAO 기반 난수 값을 사용하여 각 슬롯의 블록 제안자를 선정한다. 검증 권한을 얻기 위해서는 최소 32 ETH를 스테이킹해야 하며, 개별 검증자의 선정 확률은 전체 스테이킹 물량 대비 해당 검증자의 유효 지분(Effective Balance, 최대 32 ETH) 비율에 비례하여 결정된다.

### 3.2 실행 페이로드 구성 (Execution Payload Construction)

블록 제안자로 선정된 검증자는 Consensus Layer(CL)와 Execution Layer(EL)의 상호작용을 통해 블록에 포함될 Execution Payload를 확보하고 이를 Beacon Block에 포함하여 제안한다. 

Execution Payload의 생성 방식은 검증자가 MEV-Boost를 사용하는지 여부에 따라 두 가지 경로로 나뉜다.

- **Vanilla 모드 (~10% 검증자, MEV-Boost 미사용):**
    
    Vanilla 모드에서는 검증자의 로컬 EL 클라이언트가 직접 블록을 구성한다.
    
    1. CL → EL 블록 생성 요청 (Engine API) ⇒ 즉 “현재 fork choice 상태를 기준으로 새로운 블록 생성을 시작하라”
        
        슬롯이 시작되면 CL 클라이언트는 Engine API의 `engine_forkchoiceUpdatedV2` 호출을 통해 현재 fork choice 상태와 proposer 정보를 EL에 전달하고 새로운 블록 생성을 시작하도록 요청한다. EL 클라이언트는 로컬 Mempool에서 트랜잭션을 선택하여 실행 페이로드 생성을 시작한다.
        
    2. 트랜잭션 선택 전략
        
        EL은 다음 조건을 만족하도록 트랜잭션을 정렬한다.
        
        - 트랜잭션 유효성 검증
        - nonce 순서 유지
        - 블록 gas limit 만족
        - effective priority fee 최대화
        
        → 실제 정렬 기준: effective_tip = min(maxPriorityFeePerGas, maxFeePerGas − baseFeePerGas)
        
    3. 트랜잭션 실행 (EVM Execution)
        
        선택된 트랜잭션들은 EVM에서 순차적으로 실행되며 account balance, nonce, contract storage, logs, receipts 등이 업데이트된다. 실행 도중 revert가 발생한 트랜잭션은 상태 변경은 롤백되지만 소비된 gas 비용은 유지된다.
        
    4. 상태 루트(State Root) 계산
        
        모든 실행이 완료되면 EL은 다음과 같은 Cryptographic Commitment 값을 계산한다.
        
        - stateRoot
        - transactionsRoot
        - receiptsRoot
        - logsBloom
        - gasUsed
    5. EL → CL에게 Execution Payload 전달 ⇒ 즉, “결과는 다음과 같아”
        
        EL은 완성된 Execution Payload를 CL에 전달하고, CL은 이를 Beacon Block에 포함하여 네트워크에 전파한다.
        
- **PBS/MEV-Boost 모드 (90%+ 검증자, 현재 표준):**
    
    MEV-Boost 환경에서는 실행 페이로드 생성 책임이 외부 블록 빌더에게 위임된다.
    
    - 외부 빌더들은 Public Mempool, Private Orderflow, 그리고 MEV Bundle을 조합하여 수익이 최대화된 실행 페이로드를 생성한다.
    - 생성된 블록 후보는 Relay에게 전달되며, Relay는 여러 빌더의 입찰(bid)을 수집한다.
    - Relay는 각 블록의 전체 내용을 공개하지 않고 블록 헤더와 보상 정보만을 제안자에게 제공한다.
    - 제안자는 가장 높은 보상을 제시한 헤더에 서명함으로써 해당 블록을 선택한다.
    - 제안자의 서명이 확인되면 Relay는 전체 Execution Payload를 공개(reveal)한다.
    - 이후 검증자의 EL 클라이언트는 전달받은 페이로드의 실행 유효성을 검증하며, 검증에 실패할 경우 블록은 제안되지 않고 해당 슬롯은 missed proposal로 처리된다.
    - 검증이 완료되면 CL은 이를 Beacon Block에 포함하여 네트워크에 전파한다.
    
    이 과정에서 제안자는 빌더가 제시한 MEV 보상과 트랜잭션 우선 수수료(priority fee)를 획득한다.
    

Execution Payload 구조

- Execution Payload는 Execution Layer의 상태 전이 결과를 나타내며 다음 정보를 포함한다.

```tsx
ExecutionPayload {
  parentHash,
  feeRecipient,        // Proposer 수익 계정
  stateRoot,
  receiptsRoot,
  logsBloom,
  prevRandao,          // PoS 무작위성
  blockNumber,
  gasLimit: 30M,
  gasUsed,
  timestamp,
  baseFeePerGas,
  withdrawalsRoot,     // Shanghai 출금
  transactions[]       // 실행된 TX 목록
}
```

### 3.3 Beacon Block 생성 및 브로드캐스트 (Beacon Block Assembly & Broadcast)

블록 제안자는 Execution Payload를 확보한 이후 CL에서 Beacon Block을 생성하고 이를 네트워크 전체에 전파한다.

1. Execution Payload 수신 및 검증
    
    CL 클라이언트는 EL 클라이언트로부터 Execution Payload를 전달받는다. CL은 실행 결과 자체를 재계산하지 않으며, 대신 합의 계층 관점에서 최소한의 구조적 검증을 수행한다.
    
    - execution payload header 일관성 검증
    - parentHash가 현재 fork choice head와 일치하는지 확인
    - timestamp가 해당 slot 시간 범위와 일치하는지 검증
    - proposer 권한 유효성 확인
    
    실행 상태 전이의 유효성 검증은 각 검증자가 실행하는 EL이 독립적으로 수행한다.
    
2. Beacon Block 생성
    
    검증이 완료되면 CL 클라이언트는 Execution Payload를 포함한 Beacon Block을 생성한다.
    
    - Beacon Block 구조
        
        ```tsx
        BeaconBlock (Capella 이후 기준) {
          slot: uint64,                    // 12초 단위
          proposer_index: uint64,          // 제안자 ID  
          parent_root: Root,              // 이전 Beacon Block
          state_root: Root,               // Beacon State
          body: {
            randao_reveal: Bytes96,       // BLS 서명 (prevRandao)
            eth1_data: Eth1Data,          // EL deposit 루트
            **execution_payload: ExecutionPayload,  // ⭐ EL 결과**
            attestations: [],             // 이전 attestation
            deposits: [],                 // validator deposit
            voluntary_exits: [],
            sync_aggregate: SyncAggregate, // sync committee sig
          }
        }
        ```
        
        - execution_payload: EL에서 발생한 상태 전이 결과를 CL 블록에 연결하는 역할을 수행한다.
        - state_root: Beacon State 전이 결과에 대한 cryptographic commitment이다.
3. Beacon Block 서명
    
    블록 제안자는 생성된 Beacon Block 전체에 대해 BLS12-381 서명을 수행한다. 이 서명은 해당 슬롯의 정당한 제안자가 블록을 생성했음을 증명한다.
    
    - 서명 대상: hash_tree_root(BeaconBlock)
    - BLS12-381 서명 생성 결과: `SignedBeaconBlock { message: BeaconBlock, signature: BLSSignature }`
4. P2P 네트워크 전파 (Block Gossip)
    
    서명이 완료된 `SignedBeaconBlock`은 libp2p 기반 GossipSub 프로토콜을 통해 네트워크에 전파된다.
    
    블록을 수신한 다른 검증자들은 다음 과정을 수행한다.
    
    1. Consensus Layer 규칙 검증
    2. Execution Layer에서 Execution Payload 재실행 및 검증
    3. 검증 성공 시 해당 블록에 대한 Attestation 생성 및 전파
    
    이 과정은 네트워크 전체가 동일한 블록 상태에 합의하도록 만드는 첫 단계가 된다.
    

![image.png](image.png)


[https://ethereum.org/developers/docs/networking-layer/#:~:text=Network layer schematic,a new tab](https://ethereum.org/developers/docs/networking-layer/#:~:text=Network%20layer%20schematic,a%20new%20tab))

여기서 Execution Layer와 Consensus Layer의 역할이 명확히 분리되어 있음을 확인할 수 있다.

- **Execution Layer (EL)**: 상태 전이와 트랜잭션 실행을 담당하며, Execution Payload를 생성하고 블록에 포함된 트랜잭션 실행 결과의 유효성을 검증한다.
- **Consensus Layer (CL)**: 블록 제안, 검증, 그리고 합의 규칙을 담당하며 Beacon Block이 합의 프로토콜 규칙을 만족하는지 검증한다.

CL은 트랜잭션 실행을 직접 수행하지 않으며, Execution Payload의 내부 상태 전이 결과를 재계산하지 않는다. 대신 각 검증자는 자신의 Execution Layer 클라이언트를 통해 Execution Payload를 독립적으로 재실행하고 실행 결과의 유효성을 확인한다.

이러한 구조에서 네트워크의 신뢰성은 특정 EL 구현을 신뢰하는 데서 발생하는 것이 아니라, 모든 검증자가 독립적으로 실행 검증을 수행하고 동일한 상태 전이에 합의하는 과정에서 보장된다. 즉, 실행 검증은 분산된 validator 집합에 의해 Trustless하게 이루어진다.

이 구조는 Execution Correctness와 Consensus Safety를 분리함으로써, 이더리움이 서로 다른 EL 및 CL 클라이언트 구현 간의 클라이언트 다양성([Client Diversity](https://ethereum.org/developers/docs/nodes-and-clients/client-diversity/))을 유지할 수 있도록 한다.


## 4. Block Validation and Consensus

다른 검증자(Validator)들은 P2P 네트워크를 통해 수신한 `SignedBeaconBlock`을 독립적으로 검증하고, 해당 블록에 대한 Attestation을 생성한다. 이 Attestation들은 이후 포크 선택 규칙(Fork Choice Rule)에 반영되어 네트워크의 canonical head(합의된 체인의 최신 블록)를 결정하는데 사용된다.

### 4.1 검증 위원회 선정 (Committee Assignment)

위원회(Committee)는 특정 슬롯(Slot)에서 블록에 대한 Attestation 투표를 수행하도록 배정된 검증자(Validator) 집합이다. 즉, 블록 제안자(Proposer)가 블록을 생성하는 역할을 수행하는 반면, 위원회에 속한 검증자들은 해당 블록과 체인의 상태를 독립적으로 검증하고 투표(Attest)를 수행한다.

이더리움 PoS는 시간을 Epoch 단위(1 epoch = 32 slots, 약 6.4분)로 구분한다. 각 Epoch의 Validator 역할은 이전 Epoch의 상태에서 생성된 [RANDAO](https://ethereum.org/developers/docs/consensus-mechanisms/pos/block-proposal/#random-selection) 랜덤성(randao mix)을 기반으로 결정되며, 전체 활성 검증자(Active Validator) 집합을 pseudo-random하게 셔플(Shuffle)하여 슬롯별 역할이 배정된다. 이 과정에서 다음이 결정된다.

- 각 슬롯의 블록 제안자(Proposer) 1명
- 각 슬롯에 대해 Attestation을 수행할 여러 개의 Committee

하나의 슬롯에는 여러 개의 Committee가 존재할 수 있으며, 위원회 크기는 활성 검증자 수에 따라 동적으로 결정된다. 프로토콜은 Committee당 약 128명의 검증자가 배정되도록 설계되어 있으며, 활성 검증자 수가 증가할 경우 Committee 크기보다는 Committee의 개수가 증가한다. 메인넷에서는 일반적으로 수백 명 규모의 검증자가 하나의 Committee를 구성한다.

각 Committee는 서로 다른 Attestation Gossip Subnet에 매핑되어 Attestation Aggregation을 수행한다. 이러한 Committee 구조는 네트워크 메시지 부하를 분산하고 Attestation의 검증 및 전파를 병렬화하기 위해 설계되었다.

```tsx
slot
 ├ proposer (1)
 ├ committee 0
 ├ committee 1
 ├ committee 2
 └ ...
```

### 4.2 블록 수신 및 검증 (Block Reception and Validation)

네트워크의 검증자 노드는 P2P 네트워크를 통해 `SignedBeaconBlock`을 수신하면, 먼저 Consensus Layer(CL)에서 블록의 기본 구조와 제안자(Proposer)의 서명을 포함한 합의 규칙 검증을 수행한다.

CL은 블록의 슬롯(slot), parent root, proposer index, RANDAO reveal, 그리고 포함된 consensus operations(attestations, slashings 등)이 프로토콜 규칙을 만족하는지 검증한다. 이후 CL은 블록에 포함된 `execution_payload`를 Execution Layer(EL) 클라이언트에 Engine API의 `engine_newPayload` 호출을 통해 전달하여 실행 검증을 요청한다.

EL 클라이언트는 payload에 포함된 트랜잭션들을 EVM에서 재실행(Re-execution)하여 상태 전이(State Transition)를 계산하고, 트랜잭션 실행 결과의 유효성, state root 일치 여부, gas 사용량 및 블록 실행 규칙 준수 여부를 검증한다. 실행 검증이 성공하면 EL은 payload를 유효(Valid)하다고 응답하며, CL은 해당 블록을 유효한 실행 결과를 가진 블록으로 간주한다.

CL과 EL의 검증이 모두 성공한 이후에만 블록은 Fork Choice Store에 추가되며, 이후 포크 선택 규칙(Fork Choice Rule)에서 Canonical Chain의 후보로 고려된다. 이러한 합의 검증과 실행 검증을 모두 통과한 블록만이 유효한 블록(Valid Block)으로 인정된다.

### 4.3 증명 생성 및 전파 (Attestation Creation & Propagation)

블록 검증이 완료되면, 해당 슬롯의 위원회(Committee)에 속한 검증자는 자신이 관측한 체인 상태에 대해 Attestation 메시지를 생성한다. Attestation은 fork choice 투표와 finality 투표를 동시에 수행하는 지분 기반(stake-weighted) 합의 메시지이다. 각 Attestation에는 다음과 같은 핵심 정보가 포함된다.

- **beacon_block_root**: 검증자가 현재 head로 판단한 beacon chain 블록
- **Source checkpoint**: 가장 최근에 justified 된 epoch의 체크포인트
- **Target checkpoint**: attestation이 속한 현재 epoch의 체크포인트

`beacon_block_root`는 [LMD-GHOST](https://arxiv.org/pdf/2003.03052)(Latest Message Driven Greediest Heaviest Observed SubTree) 포크 선택 규칙에 사용된다. 각 검증자의 최신 Attestation은 지분 가중치로 집계되며, 가장 많은 지분의 지지를 받은 체인이 Canonical Head로 선택된다. 새로운 유효 블록이 기존 Head를 확장하는 경우, 해당 블록은 자연스럽게 새로운 Head 후보가 된다.

한편 Source와 Target 투표는 [Casper FFG](https://arxiv.org/pdf/1710.09437)(Casper the Friendly Finality Gadget) 합의에 사용되며, Epoch 경계에서의 정당화(Justification)와 최종 확정(Finality)을 결정한다.

생성된 Attestation 메시지는 검증자의 서명과 함께 해당 Committee에 대응되는 Attestation Gossip Subnet을 통해 네트워크로 전파되며, 이후 Aggregation 과정을 거쳐 블록에 포함된다.

```tsx
AttestationData {
  slot                 // 대상 슬롯
  index                // 위원회 ID
  beacon_block_root    // LMD-GHOST: "이 블록을 head로"
  source: Checkpoint   // FFG: 이전 justified epoch  
  target: Checkpoint   // FFG: 현재 epoch checkpoint
}
```

### 4.4 증명 집계 (Attestation Aggregation)

각 위원회(Committee)에서는 일부 검증자가 Aggregator 역할을 수행한다. Aggregator는 프로토콜에 의해 직접 지정되는 것이 아니라, 각 검증자가 자신의 Attestation에 대해 생성한 Selection Proof를 기반으로 확률적으로 스스로 선택(Self-Selection)된다. 선택 확률은 Committee 크기에 비례하여 조정되며, 일반적으로 하나의 Committee에서 소수의 Aggregator만 생성되도록 설계되어 있다.

Aggregator는 동일 Committee 내 검증자들이 생성하여 전파한 Attestation들을 수집한 뒤, 동일한 Attestation Data(블록 및 checkpoint 정보)를 공유하는 메시지들을 하나의 Aggregated Attestation으로 결합한다. 이 과정에서는 BLS signature aggregation이 사용되어 다수의 검증자 서명을 하나의 집계 서명으로 압축할 수 있다.

집계된 Attestation에는 다음 정보가 포함된다.

- 공통 attestation 데이터(attestation data)
- aggregation bits (참여 validator 비트필드)
- 집계된 단일 BLS 서명

이러한 aggregation 메커니즘은 네트워크에서 전파되는 attestation 메시지 수를 줄이고, 블록에 포함되어야 하는 서명 데이터의 크기를 감소시켜 전체 데이터 전파 비용을 낮춘다. 이를 통해 수십만 명 이상의 validator가 참여하는 환경에서도 네트워크 부하를 효과적으로 제어하며 안정적인 확장성을 유지할 수 있도록 설계되었다.

![https://ethereum.org/developers/docs/consensus-mechanisms/pos/attestations/#attestation-inclusion-lifecycle](attachment:5b1d730e-8fae-4f96-9e97-1e6bc1f0d108:image.png)

https://ethereum.org/developers/docs/consensus-mechanisms/pos/attestations/#attestation-inclusion-lifecycle

특히 수십만 명 이상의 validator가 존재하는 환경에서 모든 validator가 매 슬롯(12초)마다 개별 attestation을 전송하고 이를 모두 블록에 포함하려 할 경우, 네트워크 대역폭과 블록 크기 측면에서 심각한 병목이 발생할 수 있다. Aggregation은 여러 validator의 투표를 하나의 메시지로 압축함으로써 이러한 문제를 해결한다.

또한 RANDAO 기반 무작위 분할을 통해 validator들은 슬롯별로 여러 committee로 나뉘어 동작하며, 각 committee 내부에서만 attestation을 생성하고 집계한다. 이를 통해 전체 네트워크의 통신 부담을 분산시키면서도 보안성과 합의 정확성을 동시에 유지할 수 있다.

### 4.5 포크 선택 규칙 적용 (Fork Choice — LMD-GHOST)

각 검증자는 자신이 P2P 네트워크를 통해 관측한 블록과 Attestation을 기반으로 LMD-GHOST 포크 선택 규칙을 실행하여 현재 Beacon Chain의 Head를 결정한다.

LMD-GHOST는 각 Validator의 가장 최근 Attestation(latest message) 만을 고려하여 지분 가중치를 계산한다. 알고리즘은 Genesis 블록부터 시작하여 다음 과정을 반복한다. 현재 블록의 자식 블록들 중 가장 많은 지분(weighted latest attestations)을 받은 블록을 선택하고, 해당 경로를 따라 내려가며 더 이상 자식이 없을 때의 블록을 head로 선택한다. 즉, 가장 많은 validator의 최신 지지를 받고 있는 체인이 Canonical Chain으로 선택된다.

다음 슬롯의 블록 제안자(Proposer)는 자신이 관측한 Aggregated Attestation들을 Beacon Block에 포함시킬 수 있으며, 이를 통해 네트워크 전체가 Validator 투표 결과를 빠르게 동기화할 수 있다.

이 단계에서 특정 트랜잭션이 포함된 블록은 포크 선택 규칙에 의해 체인의 일부(head chain)로 간주되지만, 아직 Casper FFG에 의해 최종 확정(Finalized)된 상태는 아니다. 따라서 이후 더 많은 지분의 Attestation이 다른 체인을 지지할 경우 체인 재구성(Reorganization)이 발생할 수 있다.

```tsx
Slot N: Block 제안 ──┐
                    │ CL+EL 검증
Slot N+1: 수신 ──────┼─→ stateRoot 재실행 → Attestation 생성
                    │
                    ├─→ LMD-GHOST (Head) 투표
                    └─→ Casper-FFG (Justify/Finalize) 투표
                      ↓
집계 → Slot N+2 블록에 포함 → "체인 헤드" 확정 (probabilistic)

```

```tsx
// 실제 타이밍 예시 (네트워크 지연 200ms 가정)

Slot N   (12:00:00)  제안자A → Block N 브로드캐스트
                    ↓ 200ms 후 수신
Slot N+1 (12:00:12) 다른 노드들 Block N 검증 시작 (EVM 재실행 2초)
                    ↓ 동시 진행
Slot N+1 (12:00:12) 제안자B → Block N+1 브로드캐스트  
Slot N+1 (12:00:14) Block N 검증 완료 → Attestation 생성
Slot N+2 (12:00:24) 제안자C → Block N+2 (Attestation 포함) 브로드캐스트
                    ↓ Block N "체인 헤드" 확정!
```


## 5. Justification and Finalization

이 단계에서는 LMD-GHOST 포크 선택 규칙을 통해 선택된 체인의 head(확률적 합의 상태)에 대해, Casper FFG를 사용하여 체인의 일부를 경제적으로 되돌릴 수 없는 상태(Finality)로 확정한다.

LMD-GHOST는 현재 네트워크에서 가장 많은 validator의 최신 지지를 받는 체인을 선택하는 fork choice 규칙이며, Casper-FFG는 epoch 경계에서 validator 투표를 집계하여 체인의 특정 지점을 정당화(Justification) 및 최종 확정(Finalization) 하는 역할을 수행한다.

### 5.1 체크포인트 정당화 (Checkpoint Justification)

이더리움 PoS는 Casper FFG를 통해 블록에 대한 최종 확정성을 제공한다.

Beacon Chain의 시간 구조는 다음과 같다.

- Slot: 12초
- Epoch: 32 slots (약 6.4분)

각 epoch의 경계(epoch boundary)에 위치한 블록, 즉 해당 epoch의 첫 번째 슬롯에 해당하는 블록은 [checkpoint](https://ethereum.org/glossary/#checkpoint)로 간주되며, finality 투표는 checkpoint 단위로 수행된다.

```tsx
Epoch 0: Slot 0  ← checkpoint
Epoch 1: Slot 32 ← checkpoint
Epoch 2: Slot 64 ← checkpoint
Epoch 3: Slot 96 ← checkpoint
...
```

Validator의 Attestation에는 head vote뿐 아니라 checkpoint 간 관계를 나타내는 다음 정보가 포함된다.

- **Source checkpoint**: 가장 최근에 justified 된 epoch의 체크포인트
- **Target checkpoint**: attestation이 속한 현재 epoch의 체크포인트

Validator는 source checkpoint에서 target checkpoint까지 이어지는 체인을 지지한다는 의미의 투표를 수행한다. 전체 활성 stake의 2/3 이상(supermajority)이 동일한 source → target 투표를 수행하면 해당 target checkpoint는 Justified 상태가 된다.

### 5.2 체크포인트 완결 (Checkpoint Finalization)

Checkpoint finalization은 연속된 epoch 간의 정당화 관계(Justified)를 통해 발생한다. 

어떤 checkpoint C_n이 Justified 상태이고, 그 다음 epoch의 checkpoint C_{n+1}이 동일한 체인을 source로 하여 다시 Justified 되면, 이전 checkpoint C_n은 Finalized 상태가 된다.

```tsx
Epoch n justified
Epoch n+1 justified (source = n)
→ Epoch n **finalized**
```

이는 validator들이 두 epoch에 걸쳐 동일한 체인을 지속적으로 지지했음을 의미하며, 해당 checkpoint 이전의 체인은 합의적으로 확정된다.

Finalized 상태 이후 해당 체인을 되돌리기 위해서는 전체 스테이크의 최소 1/3 이상이 상충된 투표를 수행해야 하며, 이는 Casper FFG 규칙에 의해 슬래싱(slashing)을 유발한다. 따라서 finalized 체인을 재구성(reorganization)하는 공격은 경제적으로 매우 높은 비용을 요구한다.

### 5.3 상태 확정 (Canonical State Finality)

특정 블록이 포함된 체인의 checkpoint가 Finalized 상태에 도달하면, 해당 블록까지의 상태 전이(state transition)는 합의적으로 확정된 것으로 간주된다.

이 시점 이후 블록의 execution 결과는 state root 형태로 beacon chain에 암호학적으로 커밋되며, 이를 되돌리기 위해서는 전체 검증자 지분의 최소 1/3 이상이 슬래싱 위험을 감수해야 한다. 따라서 스마트 컨트랙트 상태, ETH 잔액, storage 변경 등 해당 시점까지의 모든 상태 변경은 실질적으로 되돌릴 수 없는 상태가 된다.

정상적인 네트워크 환경에서는 블록이 포함된 이후 최소 약 2 epoch(64 slots, 약 12.8분) 이후 finality에 도달한다. 실제 메인넷에서는 네트워크 지연 및 validator 참여율 등에 따라 일반적으로 약 13~15분 내외의 시간이 소요된다.

```tsx
Slot 0: TX → Execution Payload → Beacon Block (Proposer)
Slot 1: LMD-GHOST head (4.5장, ~12초)
↓
Epoch N 시작 (Slot 32n): Attestation 투표 시작  
Epoch N 끝 (Slot 32n+31): Checkpoint 생성
↓ ~6.4분 후: Epoch N Justified (2/3 supermajority)
↓ ~12.8분 후: Epoch N+1 Justified → **Epoch N Finalized** 🔒
```

**왜 LMD-GHOST와 FFG를 함께 사용하는가 (Safety vs Liveness 분리)**

Ethereum의 PoS 합의 프로토콜([Gasper](https://ethereum.org/developers/docs/consensus-mechanisms/pos/gasper/))은 단일 알고리즘이 아니라, 서로 다른 역할을 수행하는 두 메커니즘인 LMD-GHOST fork choice rule과 Casper FFG(finality gadget)의 결합으로 구성된다. 이는 블록체인 합의에서 서로 상충하기 쉬운 두 목표인 liveness(지속적인 블록 생성)와 safety(확정성 보장)를 분리하여 달성하기 위한 설계이다.

LMD-GHOST는 네트워크에서 발생하는 포크 상황에서 어떤 체인을 따라야 하는지를 결정하는 fork choice 알고리즘으로, validator들의 최신 attestation 가중치를 기반으로 가장 많은 지지를 받은 체인을 선택한다. 이를 통해 네트워크 지연이나 일시적인 분기 상황에서도 블록 생성이 계속 진행되도록 하여 높은 liveness를 보장한다. 즉, 네트워크가 완전히 안정적이지 않더라도 체인은 계속 성장할 수 있다.

반면 Casper FFG는 블록을 단순히 선택하는 것이 아니라 특정 체크포인트 블록을 justified 및 finalized 상태로 승격시켜 되돌릴 수 없는 확정성을 제공한다. 전체 스테이크의 최소 2/3 이상이 투표해야 finality가 성립하며, 이 이후 블록을 되돌리려면 대규모 슬래싱이 발생해야 하므로 경제적으로 매우 어려워진다. 따라서 FFG는 체인의 안전성(safety)을 담당하는 계층으로 동작한다. 

이 두 메커니즘은 서로 보완적인 관계를 가진다. LMD-GHOST는 빠르게 체인의 head를 선택하여 네트워크 진행성을 유지하지만, 새로운 정보가 들어오면 선택이 변경될 수 있어 단독으로는 강한 안전성을 제공하지 않는다. 반대로 FFG는 높은 안전성을 제공하지만 네트워크가 불안정할 경우 finalization을 일시적으로 멈출 수 있어 liveness가 제한될 수 있다. Ethereum은 이 특성을 이용해 LMD-GHOST가 블록 생산과 체인 진행을 담당하고, FFG가 뒤따라 블록을 확정(finalize)하는 구조를 채택하였다. 

결과적으로 Gasper는 다음과 같은 역할 분리를 통해 안정성과 확장성을 동시에 확보한다.

- LMD-GHOST → 빠른 fork 선택 및 체인 진행 (liveness 중심)
- Casper FFG → 경제적 확정성 제공 (safety 중심)

이와 같은 설계는 네트워크 지연, validator 일부 오프라인 상황, 혹은 공격 시나리오에서도 체인이 멈추지 않으면서도 장기적으로는 되돌릴 수 없는 합의를 형성하도록 만든다. 즉, Ethereum PoS 합의는 하나의 알고리즘으로 모든 문제를 해결하기보다, 진행성과 안전성을 계층적으로 분리하여 결합한 하이브리드 합의 구조라고 볼 수 있다.