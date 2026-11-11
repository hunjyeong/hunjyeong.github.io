---
title: "Ethereum Lifecycle"
date: 2026-03-22 12:24:00 +0900
categories: [Blockchain, Ethereum]
tags: [github Pages, githubBlog, blog, github, blockchain]
media_subpath: /assets/img/posts/ethereum-life-cycle/
---

> 이 글은 2026년 11월 기준 Ethereum 메인넷(Fusaka 업그레이드 적용 상태)을 기준으로 한다. Fusaka는 Fulu + Osaka의 줄임말로, Ethereum의 업그레이드 이름은 Consensus Layer 업그레이드 이름(Fulu)과 Execution Layer 업그레이드 이름(Osaka)을 합쳐서 만든다. 다음 업그레이드인 Glamsterdam(Gloas + Amsterdam)은 2026년 10월 6일 Sepolia 테스트넷에 활성화되었다. Hoodi 테스트넷 활성화는 10월 26일로 예정되었다가 10월 22일에 연기되었고, 새 Hoodi 일정과 메인넷 활성화 일정은 정해지지 않았다([Ethereum Foundation 공지](https://blog.ethereum.org/2026/10/22/glamsterdam-hoodi-delay)). Glamsterdam이 메인넷에 적용되면 3장의 PBS(Proposer-Builder Separation) 구조, Slot 내 타이밍, Engine API(Application Programming Interface) 메서드 버전이 달라진다. 1.1과 2.1의 가스 수치도 달라진다([EIP-2780](https://eips.ethereum.org/EIPS/eip-2780), [EIP-7976](https://eips.ethereum.org/EIPS/eip-7976), [EIP-7981](https://eips.ethereum.org/EIPS/eip-7981)). 포함될 EIP 목록은 확정 전이므로 바뀔 수 있다. PBS 구조의 변경은 ePBS(enshrined PBS)라고 부르며 [EIP-7732](https://eips.ethereum.org/EIPS/eip-7732)에 정의되어 있다. EIP(Ethereum Improvement Proposal)는 Ethereum 프로토콜 변경을 제안하는 문서이다.

<br />
<br />

**Ethereum(이더리움)**은 블록과 트랜잭션 데이터를 검증할 수 있는 소프트웨어를 실행하는 컴퓨터들로 구성된 분산 네트워크이다. 컴퓨터가 Node가 되려면 이 소프트웨어(Client)를 실행해야 한다.

![client](node.webp)
_출처: [https://ethereum.org/developers/docs/nodes-and-clients](https://ethereum.org/developers/docs/nodes-and-clients/#:~:text=Simplified%20diagram%20of%20a%20coupled%20execution%20and%20consensus%20client)_

### What are Nodes?

Node는 Ethereum Client 소프트웨어를 실행하는 컴퓨터이다. 다른 Node와 P2P(Peer-to-Peer)로 연결되어 블록과 트랜잭션을 검증하고, 저장하고, 전파한다.

Node는 저장하는 데이터의 범위에 따라 세 가지로 나뉜다.

- Full Node: 동기화 이후 들어오는 블록을 모두 직접 실행해 검증한다. 최근 상태만 유지하고 오래된 상태는 삭제한다.
- Archive Node: Full Node의 데이터에 더해 과거의 모든 상태를 보관한다.
- Light Node: 블록 헤더만 저장한다. 필요한 데이터는 Full Node에 요청하고, 헤더의 루트 값으로 직접 검증한다.

### What are Clients?

Client는 Ethereum 프로토콜 명세를 구현한 소프트웨어이다. 명세는 하나이지만 여러 팀이 서로 다른 언어로 구현한 Client가 존재한다. PoS(Proof-of-Stake) 전환 이후 하나의 Node는 Execution Client와 Consensus Client를 함께 실행해야 한다. 두 Client는 Engine API라는 로컬 RPC(Remote Procedure Call)로 연결되며, Consensus Client가 Execution Client에 요청을 보내는 방향으로 동작한다.

- **Execution Client** (EL Client, Execution Engine)
    - Execution Layer(실행 계층)를 구현한 프로그램이다.
        - 예: Geth, Nethermind, Besu, Erigon, Reth 등
    - P2P 네트워크에서 트랜잭션을 수신하여 Mempool(블록에 포함되기 전의 트랜잭션을 보관하는 공간)에 저장하고 다른 Node에 전파한다.
    - 트랜잭션을 EVM(Ethereum Virtual Machine)에서 실행하고, 계정 잔고와 스토리지 등 Ethereum의 최신 상태와 데이터베이스를 관리한다.
    - JSON(JavaScript Object Notation)‑RPC API를 제공해서 외부(지갑, DApp(Decentralized Application) 등)에서 상태 조회와 트랜잭션 전송이 가능하게 한다.
    - Consensus Client의 요청(Engine API)에 따라 Execution Payload를 생성하고 검증한다. Execution Payload는 블록에 담기는 트랜잭션 목록과 그 실행 결과이다.

- **Consensus Client** (CL Client, Beacon Node)
    - Consensus Layer(합의 계층)와 PoS 합의 알고리즘을 구현한 프로그램이다.
        - 예: Lighthouse, Prysm, Teku, Nimbus, Lodestar, Grandine 등
    - Execution Client와는 별도의 P2P 네트워크에서 Beacon Block과 Attestation(Validator의 투표)을 수신하고 전파한다.
    - Beacon State를 관리하며, 이 상태로부터 각 Slot(12초 단위의 시간 구간)의 Proposer(블록 제안자)와 Committee(투표를 맡은 Validator 집합)를 계산한다.
    - 수집한 Attestation으로 Fork Choice Rule(포크 선택 규칙)인 LMD-GHOST(Latest Message Driven Greediest Heaviest Observed SubTree)를 실행하여 체인의 Head(최신 블록)를 결정한다.
    - 블록의 실행 결과를 직접 계산하지 않고, Engine API로 Execution Client에 Execution Payload의 생성과 검증을 요청한다.
    - Casper FFG(Friendly Finality Gadget) 규칙에 따라 Checkpoint를 Justified(정당화) 및 Finalized(완결) 상태로 만든다.

<br />

![client](client1.webp)
_출처: [https://ethereum.org/developers/docs/nodes-and-clients/node-architecture/](https://ethereum.org/developers/docs/nodes-and-clients/node-architecture/)_

<br />

실제로 트랜잭션이 네트워크에 들어와서 블록에 포함되고 확정되기까지 Ethereum 트랜잭션의 Lifecycle은 다음과 같이 진행된다.


## 1. Transaction Origination and Signing

사용자가 트랜잭션을 네트워크에 제출하기 전, 로컬 환경(지갑 앱, DApp 등)에서 수행하는 단계이다.

### 1.1 트랜잭션 객체 생성 (Transaction Object Construction)

트랜잭션을 보낼 수 있는 계정은 EOA(Externally Owned Account, 개인키로 제어하는 계정)뿐이고, 컨트랙트 계정은 트랜잭션을 보낼 수 없다. 사용자는 지갑 소프트웨어(예: MetaMask)에서 트랜잭션 객체를 생성한다. 아래는 [EIP-1559](https://eips.ethereum.org/EIPS/eip-1559)(type 2) 트랜잭션 객체의 예시이다.
 
```js
{
  from: "0xEA674fdDe714fd979de3EdF0F56AA9716B898ec8", // 서명할 계정 (서명 대상 필드는 아님)
  type: 2,                                          // EIP-1559 트랜잭션
  chainId: 1,                                       // Ethereum 메인넷
  nonce: 0,
  to: "0xac03bb73b6a9e108530aff4df5077c2b3d481e5a",
  value: 10000000000n,                              // wei 단위 (= 10 gwei)
  data: "0x",                                       // 단순 송금이므로 비어 있음
  gasLimit: 21000n,                                 // 단순 송금에 필요한 가스
  maxFeePerGas: 30000000000n,                       // 30 gwei
  maxPriorityFeePerGas: 1000000000n,                // 1 gwei
  accessList: []
}
```
 
- **from**: 트랜잭션에 서명할 발신자 주소. 지갑이나 RPC 요청에서 서명할 계정을 지정하는 용도이며, 서명되는 트랜잭션 필드에는 포함되지 않는다. 네트워크는 서명에서 발신자 주소를 복구한다(2.1 참고).
- **type**: 트랜잭션 유형([EIP-2718](https://eips.ethereum.org/EIPS/eip-2718)). 2는 EIP-1559 트랜잭션으로 현재의 표준 유형이다. 이 밖에 0(Legacy), 1([EIP-2930](https://eips.ethereum.org/EIPS/eip-2930)), 3([EIP-4844](https://eips.ethereum.org/EIPS/eip-4844) blob 트랜잭션), 4([EIP-7702](https://eips.ethereum.org/EIPS/eip-7702))가 있다.
- **chainId**: 트랜잭션이 유효한 체인의 ID(Identifier). 서명 대상에 포함되어 다른 체인에서의 재전송(Replay Attack)을 막는다.
- **nonce**: 발신자 계정의 현재 nonce와 같아야 하는 값. 0부터 시작하며, 트랜잭션이 블록에 포함될 때마다 계정 nonce가 1씩 증가하므로 같은 체인에서 동일 트랜잭션이 중복 실행되는 것을 막는다. EIP-7702의 authorization(EOA가 자신의 코드를 특정 컨트랙트에 위임한다는 서명된 승인)이 처리되거나 위임된 코드가 컨트랙트를 생성(CREATE/CREATE2)할 때도 증가하므로, 보낸 트랜잭션 수와 다를 수 있다.
- **to**: 수신자 주소. 수신자가 EOA든 컨트랙트든 value가 전송되며, 수신자에 코드가 있으면(컨트랙트, 또는 EIP-7702로 코드를 위임한 EOA) 그 코드가 실행된다. 코드 실행이 revert되면 value 전송도 취소된다. 컨트랙트를 배포하는 트랜잭션은 to를 비워 두고 data에 init code(실행 결과로 컨트랙트의 런타임 코드를 반환하는 생성 코드)를 넣는다. type 3과 type 4 트랜잭션은 to를 비울 수 없어 배포에 쓸 수 없다.
- **value**: 전송할 ETH의 양 (wei 단위. wei는 ETH의 최소 단위이며 1 ETH = 10^18 wei, 1 gwei = 10^9 wei)
- **data**: 임의의 데이터를 담는 선택 필드. 컨트랙트를 호출할 때는 앞 4bytes에 호출할 함수의 selector를, 그 뒤에 ABI(Application Binary Interface) 규칙으로 인코딩한 인자를 넣는다.
- **gasLimit**: 이 트랜잭션이 소모할 수 있는 가스의 최대 한도. 코드가 없는 주소로 ETH만 보내는 단순 송금은 21,000이 필요하다(Glamsterdam의 [EIP-2780](https://eips.ethereum.org/EIPS/eip-2780) 적용 후에는 존재하지 않는 주소로 value를 보내면 새 계정 생성 비용이 추가되고, value가 0이면 15,000으로 줄어든다. EIP-2780은 2026년 11월 기준 Last Call 상태이며 메인넷에 적용되지 않았다). 사용하지 않은 가스에 대한 요금은 청구되지 않는다.
- **maxFeePerGas** (EIP-1559): 가스당 지불할 의사가 있는 총 요금의 상한. Base Fee와 Priority Fee를 모두 포함한 값이다.
    - 포함 조건: maxFeePerGas ≥ Base Fee, maxFeePerGas ≥ maxPriorityFeePerGas, 발신자 잔고 ≥ gasLimit × maxFeePerGas + value (실제 차감액이 아니라 상한을 기준으로 잔고를 확인한다)
    - 실제 가스당 지불액 = min(maxFeePerGas, Base Fee + maxPriorityFeePerGas)
    - Base Fee는 소각되고, 나머지(Priority Fee)는 블록의 fee recipient(블록을 만든 쪽이 지정한 수수료 수령 주소)에게 지급된다. 상한과 실제 지불액의 차이는 청구되지 않는다.
- **maxPriorityFeePerGas** (EIP-1559): fee recipient에게 팁으로 지불할 가스당 최대 요금
- **accessList** ([EIP-2930](https://eips.ethereum.org/EIPS/eip-2930)): 트랜잭션이 접근할 주소와 스토리지 키 목록. 대부분 비워 둔다.

### 1.2 계정 상태 조회 및 Fee 추정 (Account State Query & Fee Estimation)

트랜잭션 서명 전 지갑은 RPC Node에 연결해 상태를 조회하고 fee를 추정한다. `eth_estimateGas`와 `eth_call`은 트랜잭션을 Node에서 시뮬레이션하므로 예상 가스 사용량과 실패 여부를 미리 확인할 수 있다. 어느 메서드를 어떤 시점에 호출하는지는 지갑 구현에 따라 다르다.

```text
1. eth_getTransactionCount(address, "pending")              → nonce 조회 (대기 중인 트랜잭션 포함)
2. eth_feeHistory(blockCount, "latest", rewardPercentiles)  → 최근 블록들의 base fee와 priority fee 분포
   또는 eth_getBlockByNumber("latest", false)                → 최신 블록의 baseFeePerGas
3. eth_maxPriorityFeePerGas()                               → Node가 제안하는 priority fee
4. eth_estimateGas({from, to, value, data})                 → gasLimit 추정
5. eth_call({from, to, value, data}, "latest")              → 시뮬레이션 실행 (상태는 변경되지 않음)
```

### 1.3 개인키 기반 트랜잭션 서명 (Transaction Signing)

먼저 서명 대상 해시를 만든다. EIP-1559 트랜잭션은 서명을 제외한 9개 필드를 정해진 순서로 RLP(Recursive Length Prefix) 인코딩하고, 그 앞에 트랜잭션 유형 바이트 `0x02`를 붙인 뒤 keccak256 해시를 계산한다.

```text
sigHash = keccak256(0x02 || RLP([chainId, nonce, maxPriorityFeePerGas, maxFeePerGas, 
                                gasLimit, to, value, data, accessList]))
```

- **RLP**: Ethereum Execution Layer에서 사용하는 데이터 직렬화(serialization) 형식. 트랜잭션 필드와 블록 헤더(stateRoot, transactionsRoot 등)를 인코딩한다. prefix 바이트가 뒤따르는 데이터의 종류(바이트열 또는 리스트)와 길이를 나타낸다.
- **Hash의 역할**: 임의 길이의 트랜잭션을 32bytes 고정 길이 값으로 요약한다. 트랜잭션 내용이 조금이라도 바뀌면 해시가 달라지므로, 같은 서명에서 원래 발신자와 다른 주소가 복구된다.

그 다음으로, sigHash를 secp256k1 개인키로 ECDSA(Elliptic Curve Digital Signature Algorithm) 서명하여 (yParity, r, s)를 생성한다.

- r: 서명 과정에서 만든 임시 곡선 점의 x좌표 (32bytes)
- s: 서명값 (32bytes)
- yParity: 복구 식별자. 임시 곡선 점의 y좌표 홀짝을 나타내며 0 또는 1이다.
    - Legacy(type 0) 트랜잭션에서는 이 값을 v라고 부르며, 27/28 또는 [EIP-155](https://eips.ethereum.org/EIPS/eip-155) 방식(chainId × 2 + 35 또는 36)을 사용한다. Type 2 트랜잭션은 chainId가 별도 필드이므로 v에 chainId를 넣지 않는다.

서명 후 서명값을 포함해 다시 RLP 인코딩하여 Signed Raw Transaction을 만든다.

```text
rawTx  = 0x02 || RLP([chainId, nonce, maxPriorityFeePerGas, maxFeePerGas,
                      gasLimit, to, value, data, accessList, yParity, r, s])
txHash = keccak256(rawTx)
```

### 1.4 노드로 트랜잭션 전송 (Transaction Submission to Node)

Signed Raw Transaction을 `eth_sendRawTransaction` JSON-RPC 호출을 통해 Ethereum EL Client에게 전송한다. 전송 대상은 사용자가 직접 운영하는 Node(Geth, Nethermind 등 EL Client) 혹은 Node 서비스 제공자(Infura, Alchemy, QuickNode 등 RPC Endpoint)이다.

<br />

## 2. Mempool Propagation and Block Construction

서명된 트랜잭션이 EL Node에 도착한 후 블록에 포함되기까지 대기하는 단계이다. EL Node는 먼저 Mempool 검증을 통과한 트랜잭션만 Mempool에 저장하고 P2P 네트워크로 전파한다. Mempool은 프로토콜이 규정한 자료구조가 아니라 각 Client가 구현한 로컬 저장소이므로, 세부 정책은 Client마다 다르다. 이 장의 용어와 기본값은 Geth를 기준으로 한다.

### 2.1 트랜잭션 1차 검증 (Initial Mempool Validation)

Node는 [`eth_sendRawTransaction`](https://ethereum.org/developers/docs/apis/json-rpc/#eth_sendrawtransaction)으로 받은 트랜잭션을 먼저 검증한다. 검증을 통과하면 트랜잭션 해시를 응답으로 반환하고, 실패하면 오류를 반환한다. 검증 항목은 다음과 같다.

1. **서명 유효성 검증 및 발신자 복구:** 
    
    트랜잭션에는 from 필드가 없으므로 Node는 서명에서 발신자 주소를 복구한다.
    
    ```text
    1. sigHash = keccak256(0x02 || RLP([chainId, nonce, maxPriorityFeePerGas, maxFeePerGas,
                                        gasLimit, to, value, data, accessList]))   // 서명 필드 제외
    2. pubKey  = ecrecover(sigHash, yParity, r, s)    // 64bytes 공개키 (x, y)
    3. sender  = keccak256(pubKey)[12:32]             // 해시의 마지막 20bytes가 주소
    ```
    
    - r과 s만으로는 공개키 후보가 여러 개 나오므로 yParity로 하나를 확정한다.
    - Node는 chainId가 현재 체인과 일치하는지, s가 곡선 위수의 절반 이하인지([EIP-2](https://eips.ethereum.org/EIPS/eip-2)) 함께 확인한다.
    
    ⇒ 별도의 "서명 검증" 단계는 없다. 복구 자체가 검증 역할을 한다. 개인키 없이 만든 서명이나 내용이 변조된 트랜잭션에서는 원래 발신자와 다른 주소가 복구되며, 그 주소에는 잔고가 없으므로 Balance 검증에서 거부된다. 이 구조에서 얻는 성질은 세 가지이다. 1) 발신자 인증: 해당 주소의 개인키 소유자만 유효한 서명을 만들 수 있다. 2) Replay Attack 방지: chainId가 서명 대상에 포함되므로 같은 트랜잭션을 다른 체인에 재전송할 수 없다. 3) 무결성: 필드가 조금이라도 바뀌면 sigHash가 달라진다. 즉, 검증은 개인키 없이 누구나 할 수 있지만 서명 생성은 개인키 소유자만 가능하다.

2. **Nonce 검증:**
    - 트랜잭션의 nonce가 발신자 계정의 현재 nonce보다 작지 않은지 확인한다.
    - 현재 계정의 nonce = 5인 경우 예시
        - `tx.nonce < 5` → 이미 사용된 nonce ⇒ 거부
        - `tx.nonce = 5` → 바로 실행 가능 ⇒ Pending
        - `tx.nonce > 5` → 앞선 nonce가 Mempool에 모두 있으면 Pending, 중간에 빈 nonce가 있으면 Queued

3. **Balance 검증:**
    - 발신자 잔고 ≥ value + (gasLimit × maxFeePerGas) 인지 확인한다.
    - Node가 보유한 최신 상태(StateDB)에서 잔고를 조회한다.

4. **Fee 및 GasLimit 검증:**
    - maxFeePerGas ≥ maxPriorityFeePerGas
    - intrinsic gas ≤ gasLimit ≤ 16,777,216
        - intrinsic gas는 실행 전에 부과되는 기본 비용으로, 21,000에 calldata와 access list 비용 등이 더해진다. gasLimit이 이보다 작으면 거부된다. Pectra(Prague + Electra) 업그레이드의 [EIP-7623](https://eips.ethereum.org/EIPS/eip-7623) 이후에는 gasLimit이 calldata floor 비용(21,000 + 10 × calldata 토큰 수)보다 작아도 거부된다.
        - 16,777,216(2^24)은 Fusaka 업그레이드의 [EIP-7825](https://eips.ethereum.org/EIPS/eip-7825)가 정한 트랜잭션당 가스 상한이다. 이 값을 넘으면 거부된다. Geth는 gasLimit이 현재 블록 가스 한도를 넘는지도 함께 확인한다.
        - 블록 가스 한도는 고정 상수가 아니며 Validator들의 설정에 따라 블록마다 조금씩 조정된다. 2026년 11월 기준 메인넷은 약 60M(6,000만) gas이다. Fusaka에 포함된 [EIP-7935](https://eips.ethereum.org/EIPS/eip-7935)가 EL Client의 기본 설정값을 60M으로 정했다.


### 2.2 Mempool 등록 (Mempool Admission)

1차 검증을 모두 통과한 트랜잭션은 Node의 로컬 Mempool에 다음과 같은 과정을 통해 추가된다. 

1. 트랜잭션 해시 계산
    - keccak256(0x02 || RLP([..., yParity, r, s])) → 트랜잭션 해시 (0x1234...)
2. Pending / Queued 분류
    - Pending: 계정의 현재 nonce부터 빈 번호 없이 이어지는 트랜잭션. 바로 실행 가능하다.
    - Queued: 앞선 nonce가 비어 있어 아직 실행할 수 없는 트랜잭션. 빈 nonce가 채워지면 Pending으로 승격된다.
3. Replacement 체크
    - 같은 발신자, 같은 nonce의 트랜잭션이 이미 있으면 수수료를 비교한다.
    - Geth 기본 설정에서는 새 트랜잭션의 maxFeePerGas와 maxPriorityFeePerGas가 모두 기존 값보다 10% 이상 높아야 교체된다(`--txpool.pricebump`). 조건을 만족하지 못하면 새 트랜잭션이 거부된다.
4. 블록 구성 시 정렬
    - 같은 계정의 트랜잭션은 nonce 순서를 유지하고, 계정 간에는 effective tip이 높은 트랜잭션이 우선한다(3.2 참고).


```text
계정의 현재 nonce = 5, 수수료 단위는 gwei, TX는 트랜잭션(Transaction)의 줄임말
 
초기 상태
  Pending: []
  Queued:  []
 
1. TX1 (nonce=5, maxFee=50, tip=20) 도착 → 1차 검증 통과
  Pending: [TX1(5)]
  Queued:  []
 
2. TX2 (nonce=7, maxFee=50, tip=15) 도착 → nonce 6이 비어 있어 실행 불가
  Pending: [TX1(5)]
  Queued:  [TX2(7)]
 
3. TX3 (nonce=5, maxFee=60, tip=25) 도착 → TX1과 같은 nonce, 두 수수료 모두 10% 이상 인상 → Replacement
  Pending: [TX3(5)]          ← TX1 삭제
  Queued:  [TX2(7)]
 
4. TX4 (nonce=6, maxFee=50, tip=15) 도착 → 빈 nonce가 채워져 TX2가 Pending으로 승격
  Pending: [TX3(5), TX4(6), TX2(7)]
  Queued:  []
 
5. TX3이 블록에 포함됨 → 계정 nonce = 6, TX3은 Mempool에서 제거
  Pending: [TX4(6), TX2(7)]
  Queued:  []
```

- **Local TX**: 해당 Node의 RPC로 직접 제출된 트랜잭션이다. 다섯 개 EL Client 모두 Local TX를 다른 Node에서 전파받은 트랜잭션(Remote TX)과 구분하며, 기본 설정에서는 Local TX도 P2P 네트워크에 전파한다. 우대 방식은 Client마다 다르다(2026년 11월 각 저장소 소스 기준).

    | Client | Local TX 처리 |
    | --- | --- |
    | Geth | Mempool에는 Remote TX와 같은 규칙으로 들어간다. 별도의 tracker가 Local TX를 기록해 두었다가 Mempool에서 빠지면 다시 제출하고, journal 파일에 저장해 재시작 후 복원한다. v1.15.0에서 이 구조로 바뀌었다. |
    | Reth | 기본 설정에서 가격 제한, 축출, 계정당 Slot 제한을 면제한다. `--txpool.no-local-transactions-propagation` 옵션을 켜면 Local TX를 전파하지 않는다(기본값은 전파). |
    | Besu | 기본 설정에서 Local TX에 우선권을 준다. `--tx-pool-no-local-priority` 옵션으로 해제한다. |
    | Nethermind | 수수료 하한 필터를 면제하고, 블록에 포함될 때까지 주기적으로 다시 전파한다. 단, maxFeePerGas가 현재 base fee의 70%(기본값) 미만이면 즉시 전파하지 않고 보관했다가 조건을 만족할 때 전파한다. |
    | Erigon | 최소 수수료 제한과 계정당 Slot 제한을 면제하고, 주기적으로 다시 전파한다. |

- **Private TX**: 공용 Mempool에 전파되지 않고 Builder에게 직접 전달되는 트랜잭션이다(2.3 참고). Local TX와는 다른 개념이다.


### 2.3 P2P Gossip 기반 전파 (P2P Gossip Propagation)

Mempool에 추가된 트랜잭션은 EL의 P2P 프로토콜(devp2p의 eth 프로토콜)을 통해 연결된 피어에게 전파된다. Node는 전체 네트워크에 직접 보내지 않고 자신의 피어에게만 보내며, 피어들이 다시 자신의 피어에게 전달한다. Geth 기준으로 피어 수의 제곱근에 해당하는 수의 피어에게는 트랜잭션 전체를 보내고, 나머지 피어에게는 트랜잭션 해시만 알린다(announcement 형식은 [EIP-5793](https://eips.ethereum.org/EIPS/eip-5793)). blob 트랜잭션과 크기가 큰 트랜잭션은 모든 피어에게 해시만 알린다. 해시를 받은 피어는 자신에게 없는 트랜잭션만 요청해서 받아 온다.
 
전파에는 시간이 걸리고 Node마다 Mempool 정책이 다르므로, 모든 Node의 Mempool이 동일하지는 않다. 또한 Builder는 공용 Mempool 외에 자신에게 직접 전달된 트랜잭션도 사용하므로, Builder마다 보는 트랜잭션 집합이 다르다.
 
- **Private RPC:** MEV(Maximal Extractable Value)는 블록에 담을 트랜잭션의 포함 여부와 순서를 조정해서 얻는 수익이다. MEV 공격(샌드위치 공격 등)을 방지하려는 사용자는 공용 Mempool이 아닌, Flashbots Protect RPC 등 Private RPC를 통해 Builder에게 직접 트랜잭션을 전달하기도 한다.

<br />

## 3. Block Proposal

Ethereum에서는 MEV로 인한 Validator 중앙화 압력을 줄이기 위해 [PBS(Proposer-Builder Separation)](https://ethereum.org/roadmap/pbs/#proposer-builder-separation) 구조가 사용된다. PBS에서는 블록을 구성하는 Builder와 블록을 네트워크에 제안하는 Proposer의 역할이 분리된다. 현재 메인넷의 PBS는 프로토콜에 포함된 기능이 아니라, Validator가 선택적으로 실행하는 외부 소프트웨어인 MEV-Boost와 Relay를 통해 구현된다. PBS를 프로토콜에 내장하는 ePBS([EIP-7732](https://eips.ethereum.org/EIPS/eip-7732))는 Glamsterdam 업그레이드에 포함되어 있으며, 메인넷에는 아직 적용되지 않았다.

Builder는 Mempool의 트랜잭션 등을 이용해 Execution Payload를 구성하고, Proposer에게 지급할 금액과 함께 입찰(Bid)을 Relay에 제출한다. Proposer는 Relay로부터 가장 높은 입찰의 블록 헤더와 입찰 금액만 전달받아 그 헤더에 서명한다. Proposer는 서명 시점에 블록 내용을 볼 수 없으며, 서명이 Relay에 전달된 뒤에야 Relay가 전체 Execution Payload를 공개하고 블록이 네트워크에 전파된다. 따라서 Proposer는 Relay가 유효한 블록을 제때 공개하고 전파할 것이라고 신뢰해야 한다.

### 3.1 블록 제안자 선정 (Proposer Selection)

Ethereum 시간 단위는 12초당 1개의 Slot으로 나뉘며, 32개의 Slot이 모여 1개의 Epoch를 구성한다. Fusaka 업그레이드([EIP-7917](https://eips.ethereum.org/EIPS/eip-7917)) 이후에는 Epoch N이 시작될 때 Epoch N+1의 모든 Slot에 배정될 Proposer가 결정되어 Beacon State에 저장된다. 즉 Proposer는 최소 1 Epoch 전에 알 수 있다. Fusaka 이전에는 해당 Epoch가 시작되는 시점에야 Proposer가 확정되었다.

Proposer는 활성화된 Validator 세트 중에서 무작위로 추출된다. 이때, Beacon Chain 프로토콜은 RANDAO(Random과 DAO(Decentralized Autonomous Organization)를 합친 이름으로, 각 Proposer가 제공한 값을 누적해 난수를 만드는 방식) 기반 난수 값을 사용하여 각 Slot의 Proposer를 선정한다. Validator로 활성화되려면 최소 32 ETH를 예치(Stake)해야 하며, 개별 Validator의 선정 확률은 해당 Validator의 Effective Balance에 비례한다. Effective Balance의 상한은 Pectra 업그레이드([EIP-7251](https://eips.ethereum.org/EIPS/eip-7251)) 이후 Withdrawal Credentials 유형에 따라 다르다. 0x00과 0x01 유형은 32 ETH, 0x02(compounding) 유형은 2,048 ETH이다.


### 3.2 실행 페이로드 구성 (Execution Payload Construction)

Proposer로 선정된 Validator는 Consensus Layer(CL)와 Execution Layer(EL)의 상호작용을 통해 블록에 포함될 Execution Payload를 확보하고 이를 Beacon Block에 포함하여 제안한다. 

Execution Payload의 생성 방식은 Validator가 MEV-Boost를 사용하는지 여부에 따라 두 가지 경로로 나뉜다. [Rated Network](https://explorer.rated.network/relays?network=mainnet&timeWindow=30d) 집계(2026년 11월 11일 조회)에서 MEV-Boost 블록은 196,137개, MEV-Boost를 쓰지 않은 블록은 19,569개로, 생성된 블록의 약 90.9%가 MEV-Boost를 통해 만들어졌다. 이 비율은 프로토콜 사양이 아니라 관측치이므로 시점에 따라 달라진다.

- **Vanilla 모드 (블록의 약 9%, MEV-Boost 미사용):**
    
    Vanilla 모드에서는 Validator의 로컬 EL Client가 직접 블록을 구성한다. 아래 Engine API 메서드 버전은 Fusaka 기준이며 포크에 따라 버전이 달라진다.
    
    1. CL → EL 블록 생성 요청 (Engine API) ⇒ 즉 “현재 Fork Choice 상태를 기준으로 새로운 블록 생성을 시작하라”
        
        CL Client는 자신이 제안할 Slot에 앞서 Engine API의 `engine_forkchoiceUpdatedV3`를 호출한다. 이때 현재 Fork Choice 상태(head, safe, finalized 블록 해시)와 함께 Payload Attributes(timestamp, prevRandao, suggestedFeeRecipient, withdrawals, parentBeaconBlockRoot)를 전달한다. withdrawals는 [EIP-4895](https://eips.ethereum.org/EIPS/eip-4895), parentBeaconBlockRoot는 [EIP-4788](https://eips.ethereum.org/EIPS/eip-4788)에서 추가된 필드이다. EL Client는 `payloadId`를 반환하고, 로컬 Mempool에서 트랜잭션을 선택하여 Execution Payload 생성을 시작한다.
        
    2. 트랜잭션 선택 전략
        
        EL은 다음 조건을 만족하도록 트랜잭션을 선택하고 정렬한다.
        
        - 트랜잭션 유효성 (maxFeePerGas ≥ baseFeePerGas 포함)
        - 같은 계정 내 nonce 순서 유지
        - 블록 gas limit 만족
        - effective priority fee 최대화
        
        → 실제 정렬 기준: effective_tip = min(maxPriorityFeePerGas, maxFeePerGas − baseFeePerGas)
        
    3. 트랜잭션 실행 (EVM Execution)
        
        선택된 트랜잭션들은 EVM에서 순차적으로 실행되며 account balance, nonce, contract storage, logs, receipts 등이 업데이트된다. 실행 도중 revert가 발생한 트랜잭션은 상태 변경은 롤백되지만 소비된 gas 비용은 유지된다.
    
    4. 블록 헤더 필드 계산
        
        모든 실행이 완료되면 EL은 실행 결과를 요약하는 값을 계산한다.
        
        - stateRoot, transactionsRoot, receiptsRoot: 각각 상태, 트랜잭션 목록, 영수증 목록에 대한 Cryptographic Commitment
        - logsBloom: 로그 검색용 Bloom filter
        - gasUsed: 블록에서 사용된 가스 합계
    
    5. EL → CL에게 Execution Payload 전달 ⇒ 즉, “결과는 다음과 같아”
        
        CL은 `engine_getPayloadV5(payloadId)`를 호출하여 완성된 Execution Payload를 받는다. 응답에는 블록의 수익(blockValue), blob 데이터(blobsBundle), execution requests가 함께 포함된다. CL은 이를 Beacon Block에 포함하여 네트워크에 전파한다.
        
- **PBS/MEV-Boost 모드 (블록의 약 91%):**
    
    MEV-Boost 환경에서는 Execution Payload 생성 책임이 외부 Builder에게 위임된다.
    
    - 외부 Builder들은 Public Mempool, Private Orderflow, 그리고 MEV Bundle을 조합하여 수익이 최대화된 Execution Payload를 생성한다.
    - 생성된 블록 후보는 Relay에게 전달되며, Relay는 여러 Builder의 입찰(bid)을 수집한다.
    - Relay는 각 블록의 전체 내용을 공개하지 않고 블록 헤더(Execution Payload Header)와 입찰 금액만을 Proposer에게 제공한다.
    - Proposer는 가장 높은 입찰의 헤더를 담은 Blinded Beacon Block에 서명하여 Relay에 전달한다.
    - Proposer의 서명이 확인되면 Relay는 전체 Execution Payload를 공개(reveal)하고 블록이 네트워크에 전파된다.
    - Proposer는 서명 전에 Payload를 볼 수 없으므로 실행 유효성을 직접 검증할 수 없다. Builder가 제출한 블록의 유효성 확인은 Relay의 역할이며, 확인 방식은 Relay 구현에 따라 다르다. Relay가 Payload를 공개하지 않거나 Payload가 유효하지 않으면 해당 Slot은 missed proposal이 된다. Proposer가 같은 Slot에 다른 블록을 다시 서명하면 이중 제안으로 Slashing(규칙 위반 시 예치한 ETH의 일부를 잃고 강제 퇴출되는 처벌) 대상이 되므로 대체 블록을 제안할 수 없다.
    
    이 과정에서 Builder는 Proposer가 Relay에 등록한 fee recipient 주소로 입찰 금액을 지급해야 한다(builder-specs). 지급 방법은 사양이 규정하지 않는다. 널리 쓰이는 방식은 Builder가 Execution Payload의 feeRecipient를 자신의 주소로 설정하여 priority fee와 MEV 수익을 받고, 블록의 마지막 트랜잭션으로 Proposer의 주소에 입찰 금액을 송금하는 것이다.
    

**Execution Payload 구조**

- Execution Payload는 Execution Layer의 블록 내용과 상태 전이 결과를 나타내며 다음 정보를 포함한다. 아래는 Deneb 이후(Fusaka 포함) 기준 구조이다.

```python
class ExecutionPayload(Container):
    parent_hash        # 이전 EL 블록 해시
    fee_recipient      # priority fee를 받는 주소 (MEV-Boost 블록에서는 보통 Builder 주소)
    state_root         # 실행 후 상태 루트
    receipts_root
    logs_bloom
    prev_randao        # 이전 블록까지 누적된 RANDAO mix (EIP-4399)
    block_number
    gas_limit          # 블록 가스 한도 (2026년 11월 기준 메인넷 약 60M)
    gas_used
    timestamp
    extra_data
    base_fee_per_gas
    block_hash         # 이 EL 블록의 해시
    transactions       # 트랜잭션 목록
    withdrawals        # 출금 목록 (EIP-4895, Shanghai/Capella)
    blob_gas_used      # EIP-4844 (Cancun/Deneb)
    excess_blob_gas    # EIP-4844 (Cancun/Deneb)
```

### 3.3 Beacon Block 생성 및 브로드캐스트 (Beacon Block Assembly & Broadcast)

Proposer는 Execution Payload를 확보한 이후 CL에서 Beacon Block을 생성하고 이를 네트워크 전체에 전파한다.

아래 순서는 Vanilla 경로 기준이다. MEV-Boost 경로에서는 Proposer가 Execution Payload Header만으로 Blinded Beacon Block을 만들어 서명하고, 전체 Payload는 그 뒤에 Relay가 공개한다.

1. Execution Payload 수신 및 검증
    
    CL Client는 EL Client로부터 Execution Payload를 전달받는다. CL은 실행 결과 자체를 재계산하지 않으며, 대신 Consensus Layer 관점에서 Execution Payload가 Beacon State와 일치하는지 확인한다. 이 검사는 블록을 수신한 모든 Node도 동일하게 수행한다.
    
    - parent_hash가 Beacon State에 기록된 직전 Execution Payload의 block_hash와 일치하는지 확인
    - prev_randao가 Beacon State의 현재 RANDAO mix와 일치하는지 확인
    - timestamp가 해당 Slot의 시작 시각과 정확히 일치하는지 확인
    - blob commitment 개수가 블록당 상한을 넘지 않는지 확인
    
    실행 상태 전이의 유효성 검증은 블록을 받은 각 Node의 EL이 독립적으로 수행한다.
    
2. Beacon Block 생성
    
    검증이 완료되면 CL Client는 Execution Payload를 포함한 Beacon Block을 생성한다.
    
    - Beacon Block 구조 (Electra 이후, Fusaka 포함 기준)
        
        ```python
        class BeaconBlock(Container):
            slot                      # Slot 번호 (Slot 길이 12초)
            proposer_index            # Proposer의 Validator 인덱스
            parent_root               # 이전 Beacon Block의 루트
            state_root                # 이 블록 적용 후 Beacon State의 루트
            body: BeaconBlockBody
 
        class BeaconBlockBody(Container):
            randao_reveal             # 현재 Epoch 번호에 대한 Proposer의 BLS(Boneh-Lynn-Shacham) 서명. RANDAO mix에 섞인다
            eth1_data                 # deposit 컨트랙트 상태에 대한 투표 (EIP-6110 이후 레거시)
            graffiti                  # Proposer가 넣는 임의 32bytes
            proposer_slashings
            attester_slashings
            attestations              # 이전 Slot들에서 수집한 Attestation
            deposits                  # 레거시 방식 deposit (EIP-6110 이후 execution_requests로 대체)
            voluntary_exits
            sync_aggregate            # sync committee 집계 서명
            execution_payload         # EL 블록 (3.2의 ExecutionPayload)
            bls_to_execution_changes  # Withdrawal Credentials 변경 (Capella)
            blob_kzg_commitments      # blob에 대한 KZG(Kate-Zaverucha-Goldberg) commitment (EIP-4844, Deneb)
            execution_requests        # EL에서 발생한 deposit, 출금, 통합 요청 (EIP-7685, Electra)
        ```
        
        - execution_payload: EL에서 발생한 상태 전이 결과를 CL 블록에 연결하는 역할을 수행한다.
        - state_root: Beacon State 전이 결과에 대한 cryptographic commitment이다.

3. Beacon Block 서명
    
    Proposer는 생성된 Beacon Block에 대해 BLS 서명(BLS12-381 곡선)을 수행한다. 이 서명은 해당 Slot의 정당한 Proposer가 블록을 생성했음을 증명한다.
    
    - 서명 대상: `compute_signing_root(block, domain)`. 블록의 `hash_tree_root`와 도메인(DOMAIN_BEACON_PROPOSER, 포크 버전, genesis validators root로 계산)을 묶어 해시한 값이다. 도메인이 포함되므로 같은 서명을 다른 용도나 다른 체인에서 재사용할 수 없다.
    - 서명 결과: `SignedBeaconBlock { message: BeaconBlock, signature: BLSSignature }`

4. P2P 네트워크 전파 (Block Gossip)
    
    서명이 완료된 `SignedBeaconBlock`은 libp2p 기반 GossipSub 프로토콜을 통해 네트워크에 전파된다. 블록이 참조하는 blob 데이터는 블록과 별도로 전파된다. Fusaka의 [EIP-7594](https://eips.ethereum.org/EIPS/eip-7594)(PeerDAS, Peer Data Availability Sampling) 이후에는 data column sidecar 형태로 전파된다.
    
    블록을 수신한 다른 Validator들은 다음 과정을 수행한다.
    
    1. Consensus Layer 규칙 검증
    2. Execution Layer에서 Execution Payload 재실행 및 검증
    3. 해당 Slot의 Committee에 배정된 Validator는 Attestation 생성 및 전파
    
    이 과정은 네트워크 전체가 동일한 블록 상태에 합의하도록 만드는 첫 단계가 된다.

<br />

![client2](client2.webp)
_출처: [https://ethereum.org/developers/docs/networking-layer/#when-consensus-client-is-block-producer](https://ethereum.org/developers/docs/networking-layer/#when-consensus-client-is-block-producer)_


여기서 Execution Layer와 Consensus Layer의 역할이 명확히 분리되어 있음을 확인할 수 있다.
 
- **Execution Layer (EL)**: 상태 전이와 트랜잭션 실행을 담당하며, Execution Payload를 생성하고 블록에 포함된 트랜잭션 실행 결과의 유효성을 검증한다.
- **Consensus Layer (CL)**: 블록 제안, 검증, 그리고 합의 규칙을 담당하며 Beacon Block이 합의 프로토콜 규칙을 만족하는지 검증한다.

CL은 트랜잭션 실행을 직접 수행하지 않으며, Execution Payload의 내부 상태 전이 결과를 재계산하지 않는다. 대신 각 Node(Validator 포함)는 자신의 Execution Layer Client를 통해 Execution Payload를 독립적으로 재실행하고 실행 결과의 유효성을 확인한다.

이러한 구조에서 네트워크의 신뢰성은 특정 EL 구현을 신뢰하는 데서 발생하는 것이 아니라, 블록을 받는 모든 Full Node가 독립적으로 실행 검증을 수행하고, Validator들이 동일한 상태 전이에 합의하는 과정에서 보장된다. 즉, 실행 검증은 블록을 받는 모든 Full Node에 의해 Trustless하게 이루어진다.

EL과 CL은 Engine API라는 표준 인터페이스로 분리되어 있어, 어떤 EL 구현이든 어떤 CL 구현과 조합할 수 있다. 각 계층에서 여러 구현이 함께 쓰이는 것을 [Client Diversity](https://ethereum.org/developers/docs/nodes-and-clients/client-diversity/)라고 한다.

<br />

## 4. Block Validation and Consensus

다른 Validator(검증자)들은 P2P 네트워크를 통해 수신한 `SignedBeaconBlock`을 독립적으로 검증하고, 해당 블록에 대한 Attestation을 생성한다. 이 Attestation들은 이후 Fork Choice Rule에 반영되어 네트워크의 Canonical Head(합의된 체인의 최신 블록)를 결정하는데 사용된다.

### 4.1 검증 위원회 선정 (Committee Assignment)

Committee(위원회)는 특정 Slot에서 블록에 대한 Attestation 투표를 수행하도록 배정된 Validator 집합이다. 즉, Proposer가 블록을 생성하는 역할을 수행하는 반면, Committee에 속한 Validator들은 해당 블록과 체인의 상태를 독립적으로 검증하고 투표(Attest)를 수행한다.

Ethereum PoS는 시간을 Epoch 단위(1 Epoch = 32 Slot, 6.4분)로 구분한다. 각 Epoch의 Committee 배정은 [RANDAO](https://ethereum.org/developers/docs/consensus-mechanisms/pos/block-proposal/#random-selection) 랜덤성(randao mix)을 seed로 하여 결정된다. Epoch N의 seed에는 Epoch N−2가 끝난 시점의 randao mix가 사용되므로, Committee 배정은 최소 1 Epoch 전에 알 수 있다. 이 seed로 전체 활성 Validator 집합을 pseudo-random하게 셔플(Shuffle)하여 Slot별 Committee에 나눈다.

Proposer는 Committee 셔플로 정해지지 않는다. 같은 randao mix에서 Proposer 전용 seed를 따로 만들고, Effective Balance에 비례하는 확률로 Slot마다 1명을 뽑는다. 결과적으로 한 Slot에는 Proposer 1명과 여러 개의 Committee가 배정된다.

모든 활성 Validator는 한 Epoch 동안 정확히 하나의 Slot, 하나의 Committee에 배정된다. 즉 각 Validator는 Epoch당 한 번 Attestation을 수행한다.

하나의 Slot에는 여러 개의 Committee가 존재할 수 있다. Slot당 Committee 개수는 Committee당 최소 128명(TARGET_COMMITTEE_SIZE)을 목표로 다음과 같이 계산되며, 최대 64개(MAX_COMMITTEES_PER_SLOT)로 제한된다.

```text
committees_per_slot = max(1, min(64, 활성 Validator 수 / 32 / 128))
```

활성 Validator 수가 늘어나면 먼저 Committee 개수가 증가한다. 활성 Validator 수가 262,144명(= 32 × 64 × 128)을 넘으면 개수는 64개로 고정되고, 그 이후에는 Committee의 크기가 커진다. 메인넷은 이 기준을 넘은 상태이므로 Committee 크기는 약 (활성 Validator 수 / 2,048)명이다. 2026년 11월 11일 기준 활성 Validator는 812,294명([beaconcha.in](https://beaconcha.in/validators/queues), Epoch 488,403)이므로 Committee당 약 397명이다. 활성 Validator 수는 2025년 5월 31일의 1,055,301명에서 감소했다. Pectra(2025년 5월 7일 메인넷 활성화)의 EIP-7251로 여러 Validator를 하나로 통합할 수 있게 된 것이 원인으로 추정된다.

Slot의 각 Committee는 64개의 Attestation Gossip Subnet 중 하나에 매핑되며, Attestation은 해당 subnet에서 전파되고 집계된다. 이러한 Committee 구조는 네트워크 메시지 부하를 분산하고 Attestation의 검증 및 전파를 병렬화하기 위해 설계되었다.

```text
slot
 ├ proposer (1)
 ├ committee 0
 ├ committee 1
 ├ committee 2
 └ ...
```

### 4.2 블록 수신 및 검증 (Block Reception and Validation)

네트워크의 Validator Node는 P2P 네트워크를 통해 `SignedBeaconBlock`을 수신하면, 먼저 Consensus Layer(CL)에서 블록의 기본 구조와 Proposer의 서명을 포함한 합의 규칙 검증을 수행한다.

CL은 블록의 Slot, parent root, Proposer index, RANDAO reveal, 그리고 포함된 consensus operations(attestations, slashings 등)이 프로토콜 규칙을 만족하는지 검증한다. 이후 CL은 블록에 포함된 `execution_payload`를 Execution Layer(EL) Client에 Engine API의 `engine_newPayloadV4`(Fusaka 기준) 호출을 통해 전달하여 실행 검증을 요청한다.

EL Client는 Payload에 포함된 트랜잭션들을 EVM에서 재실행(Re-execution)하여 상태 전이(State Transition)를 계산하고, 트랜잭션 실행 결과의 유효성, state root 일치 여부, gas 사용량 및 블록 실행 규칙 준수 여부를 검증한다. 실행 검증이 성공하면 EL은 `VALID`를, 실패하면 `INVALID`를 응답한다. EL이 아직 동기화 중이어서 판단할 수 없으면 `SYNCING` 또는 `ACCEPTED`를 응답하며, 이 경우 CL은 블록을 임시로 받아들이되(optimistic sync) 그 블록에 대해 Attestation을 수행하지 않는다.

블록이 blob을 참조하는 경우에는 해당 blob 데이터가 네트워크에서 가용한지(data availability)도 확인한다. CL 검증, EL 검증, 데이터 가용성 확인을 모두 통과한 블록만이 유효한 블록(Valid Block)으로 인정되어 Fork Choice Store에 추가되고, 이후 Fork Choice Rule에서 Canonical Chain의 후보로 고려된다.

### 4.3 증명 생성 및 전파 (Attestation Creation & Propagation)

해당 Slot의 Committee에 속한 Validator는 자신이 관측한 체인 상태에 대해 Attestation 메시지를 생성한다. 생성 시점은 해당 Slot의 유효한 블록을 수신한 시점과 Slot 시작 후 4초(Slot의 1/3)가 지난 시점 중 빠른 쪽이다. 4초까지 블록이 도착하지 않으면 Validator는 그 시점에 자신이 Head로 판단한 이전 블록에 투표한다. Glamsterdam 사양에서는 이 기한이 Slot의 25%(3초)로 바뀐다.

Attestation은 Fork Choice 투표와 Finality 투표를 동시에 수행하는 Stake 기반(stake-weighted) 합의 메시지이다. 각 Attestation에는 다음과 같은 핵심 정보가 포함된다.

- **beacon_block_root**: Validator가 현재 Head로 판단한 Beacon Chain 블록
- **Source Checkpoint**: Validator의 관점에서 가장 최근에 Justified 된 Checkpoint
- **Target Checkpoint**: Attestation이 속한 현재 Epoch의 Checkpoint

`beacon_block_root`는 [LMD-GHOST](https://arxiv.org/pdf/2003.03052) Fork Choice Rule에 사용된다. 각 Validator의 최신 Attestation은 Stake 가중치로 집계되며, 가장 많은 Stake의 지지를 받은 체인이 Canonical Head로 선택된다. 새로운 유효 블록이 기존 Head를 확장하는 경우, 해당 블록은 자연스럽게 새로운 Head 후보가 된다.

한편 Source와 Target 투표는 [Casper FFG](https://arxiv.org/pdf/1710.09437) 합의에 사용되며, Epoch 경계에서의 Justification과 Finality를 결정한다.

생성된 Attestation 메시지는 Validator의 서명과 함께 해당 Committee에 대응되는 Attestation Gossip Subnet을 통해 네트워크로 전파되며, 이후 Aggregation 과정을 거쳐 블록에 포함된다.

```python
class AttestationData(Container):
    slot                 # Attestation을 수행하는 Slot
    index                # Electra(EIP-7549) 이후 항상 0. Committee는 Attestation.committee_bits로 표시
    beacon_block_root    # LMD-GHOST: "이 블록을 Head로"
    source: Checkpoint   # FFG: 가장 최근 justified checkpoint
    target: Checkpoint   # FFG: 현재 epoch의 checkpoint
```

### 4.4 증명 집계 (Attestation Aggregation)

각 Committee에서는 일부 Validator가 Aggregator 역할을 수행한다. Aggregator는 프로토콜에 의해 직접 지정되는 것이 아니라 확률적으로 스스로 선택(Self-Selection)된다. 각 Validator는 Slot 번호에 대한 자신의 BLS 서명(Selection Proof)을 만들고, 그 해시값이 조건을 만족하면 Aggregator가 된다.

```text
modulo        = max(1, Committee 크기 / 16)        // TARGET_AGGREGATORS_PER_COMMITTEE = 16
is_aggregator = hash(selection_proof)의 앞 8bytes 정수 % modulo == 0
```

개별 Validator가 선택될 확률은 약 16 / Committee 크기이므로, Committee 크기와 관계없이 Committee당 평균 16명의 Aggregator가 선택된다.

Aggregator는 동일 Committee 내 Validator들이 생성하여 전파한 Attestation들을 수집한 뒤, 동일한 Attestation Data(블록 및 Checkpoint 정보)를 공유하는 메시지들을 하나의 Aggregated Attestation으로 결합한다. 이 과정에서는 BLS signature aggregation이 사용되어 다수의 Validator 서명을 하나의 집계 서명으로 압축할 수 있다. Aggregator는 Slot 시작 후 8초(Slot의 2/3) 시점에 집계 결과를 전파한다. Glamsterdam 사양에서는 이 시점이 Slot의 50%(6초)로 바뀐다.

집계된 Attestation에는 다음 정보가 포함된다.
 
- 공통 Attestation 데이터(Attestation data)
- aggregation bits (참여 Validator 비트필드)
- Committee bits (어느 Committee의 투표인지 표시, Electra 이후)
- 집계된 단일 BLS 서명

Electra([EIP-7549](https://eips.ethereum.org/EIPS/eip-7549)) 이후에는 AttestationData에서 Committee 번호가 빠졌으므로, 같은 Slot의 서로 다른 Committee 투표도 Attestation Data가 같으면 블록에 넣을 때 하나로 합칠 수 있다.

이러한 aggregation 메커니즘은 네트워크에서 전파되는 Attestation 메시지 수를 줄이고, 블록에 포함되어야 하는 서명 데이터의 크기를 감소시켜 전체 데이터 전파 비용을 낮춘다. 이를 통해 수십만 명 이상의 Validator가 참여하는 환경에서도 네트워크 부하를 효과적으로 제어하며 안정적인 확장성을 유지할 수 있도록 설계되었다.

![attestation](attestation.webp)
_출처: [https://ethereum.org/developers/docs/consensus-mechanisms/pos/attestations/#attestation-inclusion-lifecycle](https://ethereum.org/developers/docs/consensus-mechanisms/pos/attestations/#attestation-inclusion-lifecycle)_


Validator는 Epoch당 한 번만 Attestation을 수행하지만, 수십만 명 이상의 Validator가 존재하는 환경에서는 Slot마다 전체의 1/32에 해당하는 Validator가 투표한다. 이 개별 Attestation을 모두 그대로 블록에 포함하려 할 경우 네트워크 대역폭과 블록 크기 측면에서 심각한 병목이 발생할 수 있다. Aggregation은 여러 Validator의 투표를 하나의 메시지로 압축함으로써 이러한 문제를 해결한다.

또한 RANDAO 기반 무작위 분할을 통해 Validator들은 Slot별로 여러 Committee로 나뉘어 동작하며, 각 Committee 내부에서만 Attestation을 생성하고 집계한다. 이를 통해 전체 네트워크의 통신 부담을 분산시키면서도 보안성과 합의 정확성을 동시에 유지할 수 있다.

### 4.5 포크 선택 규칙 적용 (Fork Choice — LMD-GHOST)

각 Validator는 자신이 P2P 네트워크를 통해 관측한 블록과 Attestation을 기반으로 LMD-GHOST Fork Choice Rule을 실행하여 현재 Beacon Chain의 Head를 결정한다.

LMD-GHOST는 각 Validator의 가장 최근 Attestation(latest message) 만을 고려하여 Stake 가중치를 계산한다. 알고리즘은 가장 최근에 Justified 된 Checkpoint 블록(5.1 참고)부터 시작하여 다음 과정을 반복한다. 현재 블록의 자식 블록들 중 가장 많은 Stake(weighted latest attestations)을 받은 블록을 선택하고, 해당 경로를 따라 내려가며 더 이상 자식이 없을 때의 블록을 Head로 선택한다. 즉, 가장 많은 Validator의 최신 지지를 받고 있는 체인이 Canonical Chain으로 선택된다. 여기에 더해, 해당 Slot의 Attestation 기한 전에 도착한 그 Slot의 블록에는 Proposer boost라는 임시 가중치가 더해진다. 크기는 한 Slot에 배정된 전체 Committee Stake의 40%이며 그 Slot 동안만 적용된다. Genesis가 아니라 Justified Checkpoint에서 시작하는 것이 LMD-GHOST와 Casper FFG가 결합되는 지점이다.

다음 Slot의 Proposer는 자신이 관측한 Aggregated Attestation들을 Beacon Block에 포함시킬 수 있으며, 이를 통해 네트워크 전체가 Validator 투표 결과를 빠르게 동기화할 수 있다. Slot N의 Attestation이 포함될 수 있는 가장 빠른 블록은 Slot N+1의 블록이다. 가장 늦게는 Attestation이 속한 Epoch의 다음 Epoch 마지막 Slot 블록까지 포함될 수 있다([EIP-7045](https://eips.ethereum.org/EIPS/eip-7045)).

이 단계에서 특정 트랜잭션이 포함된 블록은 Fork Choice Rule에 의해 체인의 일부(Head 체인)로 간주되지만, 아직 Casper FFG에 의해 Finalized 된 상태는 아니다. 따라서 이후 더 많은 Stake의 Attestation이 다른 체인을 지지할 경우 체인 재구성(Reorganization)이 발생할 수 있다.

```text
Slot N
 ├ 0초   Proposer가 Block N 전파
 ├ ~4초  Slot N Committee: Block N 검증 (CL 규칙 + EL 재실행) → Attestation 생성 및 subnet 전파
 │         ├ beacon_block_root → LMD-GHOST (Head) 투표
 │         └ source, target    → Casper FFG (Justify/Finalize) 투표
 └ 8초   Aggregator가 Aggregated Attestation 전파

Slot N+1
 └ 0초   다음 Proposer가 Slot N의 Attestation을 Block N+1에 포함 (가장 빠른 경우)

→ Block N은 Fork Choice 상의 Head 체인에 속하지만 아직 확정된 상태가 아니다 (reorg 가능)
```
 
```text
// Slot 내 타이밍 예시 (Slot N이 12:00:00에 시작한다고 가정)

12:00:00            Slot N 시작. Proposer A가 Block N 전파
12:00:00 ~ 12:00:04 각 Node가 Block N 수신 및 검증 (CL 규칙 + EL 재실행)
12:00:04            Slot N Committee의 Attestation 생성 기한 (블록을 먼저 검증했다면 그 시점에 생성)
12:00:08            Aggregator가 집계 결과 전파
12:00:12            Slot N+1 시작. Proposer B가 Block N+1 전파 (Slot N의 Attestation 포함 가능)
```

위 타이밍은 Fusaka 기준이다. Glamsterdam 사양에서는 Attestation 기한이 3초(Slot의 25%), 집계 전파 시점이 6초(Slot의 50%)로 바뀐다.

<br />

## 5. Justification and Finalization

이 단계에서는 LMD-GHOST Fork Choice Rule을 통해 선택된 체인의 Head(확률적 합의 상태)에 대해, Casper FFG를 사용하여 체인의 일부를 경제적으로 되돌릴 수 없는 상태(Finality)로 확정한다.

LMD-GHOST는 현재 네트워크에서 가장 많은 Validator의 최신 지지를 받는 체인을 선택하는 Fork Choice 규칙이며, Casper-FFG는 Epoch 경계에서 Validator 투표를 집계하여 체인의 특정 지점을 Justification 및 Finalization 하는 역할을 수행한다.

### 5.1 체크포인트 정당화 (Checkpoint Justification)

Ethereum PoS는 Casper FFG를 통해 블록에 대한 Finality를 제공한다.
 
Beacon Chain의 시간 구조는 다음과 같다.
 
- Slot: 12초
- Epoch: 32 Slot (6.4분)

각 Epoch의 첫 번째 Slot에 해당하는 블록은 [Checkpoint](https://ethereum.org/glossary/#checkpoint)로 간주되며, Finality 투표는 Checkpoint 단위로 수행된다. 첫 번째 Slot에 블록이 없으면 그 이전의 가장 최근 블록이 해당 Epoch의 Checkpoint가 된다.

```text
Epoch 0: Slot 0  ← checkpoint
Epoch 1: Slot 32 ← checkpoint
Epoch 2: Slot 64 ← checkpoint
Epoch 3: Slot 96 ← checkpoint
...
```
 
Validator의 Attestation에는 Head 투표뿐 아니라 Checkpoint 간 관계를 나타내는 다음 정보가 포함된다.

- **Source Checkpoint**: Validator의 관점에서 가장 최근에 Justified 된 Checkpoint
- **Target Checkpoint**: Attestation이 속한 현재 Epoch의 Checkpoint

Validator는 source Checkpoint에서 target Checkpoint까지 이어지는 체인을 지지한다는 의미의 투표를 수행한다. 전체 활성 Stake의 2/3 이상(supermajority)이 동일한 source → target 투표를 수행하면 해당 target Checkpoint는 Justified 상태가 된다. 이 집계는 Epoch이 끝날 때 수행되는 Epoch 처리 과정에서 Beacon State에 반영된다.

### 5.2 체크포인트 완결 (Checkpoint Finalization)

Checkpoint Finalization은 연속된 Epoch 간의 Justification 관계를 통해 발생한다. 

어떤 Checkpoint C_n이 Justified 상태이고, 그 다음 Epoch의 Checkpoint C_{n+1}이 C_n을 source로 하여 Justified 되면, 이전 Checkpoint C_n은 Finalized 상태가 된다. 이것이 가장 일반적인 경우이며, 명세에는 한 Epoch을 건너뛴 경우를 다루는 규칙도 추가로 정의되어 있다.

```text
Epoch n justified
Epoch n+1 justified (source = n)
→ Epoch n finalized
```
 
이는 Validator들이 두 Epoch에 걸쳐 동일한 체인을 지속적으로 지지했음을 의미하며, Checkpoint C_n의 블록과 그 이전의 체인은 합의적으로 확정된다.

Finalized 상태 이후 해당 체인을 되돌리기 위해서는 전체 Stake의 최소 1/3 이상이 상충된 투표를 수행해야 하며, 이는 Casper FFG 규칙에 의해 Slashing을 유발한다. 따라서 Finalized 체인을 재구성(reorganization)하는 공격은 경제적으로 매우 높은 비용을 요구한다.

### 5.3 상태 확정 (Canonical State Finality)

특정 블록이 포함된 체인의 Checkpoint가 Finalized 상태에 도달하면, 해당 블록까지의 상태 전이(state transition)는 합의적으로 확정된 것으로 간주된다.

블록의 execution 결과(state root)는 블록이 생성될 때 이미 Beacon Block에 암호학적으로 커밋되어 있다. Finalization이 추가하는 것은 이 커밋을 되돌릴 수 없다는 보장이다. Finalized 이후 이를 되돌리기 위해서는 전체 Validator Stake의 최소 1/3 이상이 Slashing을 감수해야 한다. 따라서 스마트 컨트랙트 상태, ETH 잔액, storage 변경 등 해당 시점까지의 모든 상태 변경은 실질적으로 되돌릴 수 없는 상태가 된다.

블록이 포함된 이후 Finality에 도달하는 시간은 그 블록이 Epoch 내 몇 번째 Slot에 있는지에 따라 달라진다. Checkpoint가 Finalized 되면 그 Checkpoint 블록까지만 확정되므로, Epoch 중간에 있는 블록은 다음 Epoch의 Checkpoint가 Finalized 되어야 확정된다. 아래는 모든 Slot에 블록이 있고, 2/3 이상이 정상적으로 투표하며, Justification과 Finalization이 Epoch 종료 시점에 반영된다고 가정한 계산이다.

```text
가정: 트랜잭션이 Epoch N의 k번째 Slot(0 ≤ k ≤ 31) 블록에 포함됨

k = 0 (해당 블록이 Epoch N의 checkpoint인 경우)
  Epoch N 종료:   Checkpoint N Justified
  Epoch N+1 종료: Checkpoint N+1 Justified → Checkpoint N Finalized
  소요 시간: 64 Slot = 12.8분

k ≥ 1 (해당 블록이 Checkpoint N 뒤에 있는 경우)
  Epoch N+1 종료: Checkpoint N+1 Justified
  Epoch N+2 종료: Checkpoint N+2 Justified → Checkpoint N+1 Finalized (해당 블록 포함)
  소요 시간: (96 − k) Slot = 13.0분 (k = 31) ~ 19.0분 (k = 1)
```

즉 정상적인 네트워크 환경에서 Finality까지 걸리는 시간은 최소 12.8분, 최대 19.0분이다. Validator 참여율이 2/3에 미치지 못하면 Finality는 그만큼 지연된다.


**왜 LMD-GHOST와 FFG를 함께 사용하는가 (Safety vs Liveness 분리)**

Ethereum의 PoS 합의 프로토콜([Gasper](https://ethereum.org/developers/docs/consensus-mechanisms/pos/gasper/), GHOST와 Casper를 합친 이름)은 단일 알고리즘이 아니라, 서로 다른 역할을 수행하는 두 메커니즘인 LMD-GHOST Fork Choice Rule과 Casper FFG(Finality gadget)의 결합으로 구성된다. 이는 블록체인 합의에서 서로 상충하기 쉬운 두 목표인 liveness(지속적인 블록 생성)와 safety(확정성 보장)를 분리하여 달성하기 위한 설계이다.

LMD-GHOST는 네트워크에서 발생하는 포크 상황에서 어떤 체인을 따라야 하는지를 결정하는 Fork Choice 알고리즘으로, Validator들의 최신 Attestation 가중치를 기반으로 가장 많은 지지를 받은 체인을 선택한다. 이를 통해 네트워크 지연이나 일시적인 분기 상황에서도 블록 생성이 계속 진행되도록 하여 높은 liveness를 보장한다. 즉, 네트워크가 완전히 안정적이지 않더라도 체인은 계속 성장할 수 있다.

반면 Casper FFG는 블록을 단순히 선택하는 것이 아니라 특정 Checkpoint 블록을 Justified 및 Finalized 상태로 승격시켜 되돌릴 수 없는 확정성을 제공한다. 전체 Stake의 최소 2/3 이상이 투표해야 Finality가 성립하며, 이 이후 블록을 되돌리려면 대규모 Slashing이 발생해야 하므로 경제적으로 매우 어려워진다. 따라서 FFG는 체인의 안전성(safety)을 담당하는 계층으로 동작한다. 

이 두 메커니즘은 서로 보완적인 관계를 가진다. LMD-GHOST는 빠르게 체인의 Head를 선택하여 네트워크 진행성을 유지하지만, 새로운 정보가 들어오면 선택이 변경될 수 있어 단독으로는 강한 안전성을 제공하지 않는다. 반대로 FFG는 높은 안전성을 제공하지만 네트워크가 불안정할 경우 finalization을 일시적으로 멈출 수 있어 liveness가 제한될 수 있다. Ethereum은 이 특성을 이용해 LMD-GHOST가 블록 생산과 체인 진행을 담당하고, FFG가 뒤따라 블록을 확정(finalize)하는 구조를 채택하였다. 

결과적으로 Gasper는 다음과 같은 역할 분리를 통해 안전성(safety)과 진행성(liveness)을 동시에 확보한다.

- LMD-GHOST → 빠른 fork 선택 및 체인 진행 (liveness 중심)
- Casper FFG → 경제적 확정성 제공 (safety 중심)

이와 같은 설계는 네트워크 지연, Validator 일부 오프라인 상황, 혹은 공격 시나리오에서도 체인이 멈추지 않으면서도 장기적으로는 되돌릴 수 없는 합의를 형성하도록 만든다. 즉, Ethereum PoS 합의는 하나의 알고리즘으로 모든 문제를 해결하기보다, 진행성과 안전성을 계층적으로 분리하여 결합한 하이브리드 합의 구조라고 볼 수 있다.