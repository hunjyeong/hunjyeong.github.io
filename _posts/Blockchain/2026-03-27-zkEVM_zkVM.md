---
title: "EVM/zkEVM/zkVM"
date: 2026-03-27
categories: [Blockchain]
tags: [Blockchain, Ethereum, EVM, zkEVM, zkVM]
math: true
media_subpath: /assets/img/posts/zkevm/
---

참고 문서

[https://decipher-zkvm.gitbook.io/decipher-zkvm-research](https://decipher-zkvm.gitbook.io/decipher-zkvm-research)

## Ethereum Virtual Machine (EVM)

### What is EVM?

EVM은 Ethereum 노드의 Execution Client(실행 클라이언트)가 공통으로 따르는 실행 환경으로, 스마트 컨트랙트의 실행과 트랜잭션 처리를 담당하는 Execution Layer(실행 계층)의 핵심을 이룬다. Execution Client(Geth, Nethermind, Besu, Erigon, Reth 등)는 각자 EVM을 구현하지만 동일한 공식 사양(하드포크별 EIP(Ethereum Improvement Proposal)와 실행 명세 [`execution-specs`](https://github.com/ethereum/execution-specs); Ethereum Yellow Paper는 초기 형식 명세)을 따른다.

EVM은 **동일한 입력이 주어질 경우 항상 동일한 계산 결과를 생성**하도록 설계되어 있다. 이러한 **Deterministic Execution**(결정론적 실행)은 모든 노드가 같은 블록을 독립적으로 실행해 같은 결과를 얻는 근거가 되며, Ethereum의 검증 방식이 성립하는 기반이다.

Ethereum은 The Merge(2022년 9월) 이후 합의와 실행을 분리한 두 개의 Layer(계층) 구조를 가진다. 각 노드는 Consensus Client와 Execution Client를 함께 실행하며, 두 Client는 Engine API(Application Programming Interface)로 통신한다.

- **Consensus Layer(CL, 합의 계층):** Block Proposer(블록 제안자) 선정, LMD-GHOST(Latest Message Driven Greedy Heaviest Observed Sub-Tree) 기반 Fork Choice(체인 선택), Casper FFG(Friendly Finality Gadget) 기반 Finality(최종성) 결정
- **Execution Layer(EL, 실행 계층):** 트랜잭션 실행, State Transition 계산, 실행 결과 생성 및 검증

Consensus Layer의 참여자는 Validator(검증자)다. Validator는 Ether를 Stake(예치)하고 블록 제안과 투표를 수행한다. EVM은 Execution Layer에서 트랜잭션의 코드 실행을 담당한다. Execution Layer는 블록을 직접 실행하고, 그 결과가 블록 헤더에 기록된 실행 결과 값과 일치하는지 확인해 블록의 실행 유효성을 판단한다.

### World State and State Transition Function (STF)

**EVM은 State(상태)를 직접 저장하지 않는다.** EVM은 트랜잭션이 컨트랙트를 호출하거나 생성할 때 바이트코드를 실행하여, 코드 실행에 따른 상태 변화를 계산한다. EVM이 읽고 쓰는 대상은 Ethereum의 **World State**(전역 상태)이다.
World State는 주소에서 계정 상태로의 매핑이며, 각 계정은 Externally Owned Account(EOA, 외부 소유 계정) 또는 Contract Account(CA, 컨트랙트 계정)로 구분된다. 각 계정은 **Nonce, Balance, Storage Root**(`storageRoot`), **Code Hash**(`codeHash`)를 가지며, 스마트 컨트랙트 코드와 컨트랙트 Storage는 각각 이 해시와 루트로 연결된다. EIP-7702(Pectra) 이후에는 EOA도 Delegation Indicator(위임 표시자, `0xef0100 || address`)를 코드로 가질 수 있다.

World State는 Modified Merkle Patricia Trie(MPT) 구조로 표현되며, 이 Trie의 Root Hash는 블록 헤더의 **State Root**로 기록된다. World State의 값이 하나라도 바뀌면 Root Hash도 바뀌므로, **State Root는 해당 시점 World State 전체를 대표한다.** 실제 디스크 저장 방식은 Client마다 다르다.

State Root와 Merkle Proof(특정 값에서 Root Hash까지의 경로에 있는 Trie Node들)를 이용하면, 전체 상태 데이터를 보유하지 않은 노드도 특정 계정이나 Storage 값이 그 상태에 포함되어 있는지 효율적으로 검증할 수 있다. 다만 State Root는 결과 상태가 무엇인지만 나타낸다. 그 상태가 올바른 실행으로 만들어졌는지는 같은 규칙으로 블록을 다시 실행해야 확인할 수 있다. 이 규칙이 State Transition Function이다.

**State Transition Function**

Ethereum은 단순한 거래 원장이 아니라 트랜잭션 기반의 **Distributed State Machine**(분산 상태 머신)으로 동작한다. 네트워크는 Genesis State(초기 상태)에서 시작해 각 블록의 트랜잭션을 순서대로 실행하며 상태를 갱신한다. 이 갱신 규칙을 **State Transition Function**(상태 전이 함수)이라고 한다. Yellow Paper는 State Transition Function을 트랜잭션 단위와 블록 단위로 나누어 정의한다. 블록 단위 함수가 트랜잭션 단위 함수를 반복 호출하는 계층 구조다. 아래에서 $\sigma$는 이전 상태, $\sigma^\prime$는 결과 상태, $T$는 트랜잭션, $B$는 블록이다.

- **트랜잭션 단위**: $\sigma^\prime = \Upsilon(\sigma, T)$

$\Upsilon$은 트랜잭션 하나를 상태에 적용한다. $\Upsilon$이 곧 EVM 실행은 아니다. EVM의 코드 실행은 Yellow Paper에서 $\Xi$로 따로 정의되며, $\Upsilon$은 $\Xi$를 포함한 트랜잭션 처리 전체를 담당한다. 처리 순서는 다음과 같다.

    1. 유효성 검사: 서명, Nonce, 잔액, Gas Limit을 확인한다.
    2. 선차감: 발신자 잔액에서 Upfront Gas 비용(실행 전에 Gas Limit만큼 미리 차감하는 수수료)을 차감하고, Nonce를 1 증가시킨다.
    3. 실행: Value를 이체하고, 수신자에 코드가 있으면 EVM이 그 코드를 실행한다($\Xi$).
    4. 정산: 사용하지 않은 Gas를 환급한다. Priority Fee(블록 생성자에게 주는 추가 수수료)는 Fee Recipient(수수료 수령 주소)에 지급하고, Base Fee(EIP-1559에 따라 블록마다 정해지는 기본 수수료)는 소각한다.

2단계의 선차감과 Nonce 증가는 실행이 실패해도 되돌려지지 않는다. 따라서 블록에 포함된 트랜잭션은 코드 실행이 실패하더라도 Nonce와 잔액을 바꾼다. 즉 **블록에 포함된 모든 트랜잭션은 상태 변화를 일으킨다.**

- **블록 단위**: $\sigma^\prime = \Pi(\sigma, B)$

$\Pi$는 블록 전체를 상태에 적용한다. 블록 $B$는 블록 헤더 값(블록 번호, Timestamp, Base Fee, Fee Recipient 등), 순서화된 트랜잭션 목록, Withdrawal 목록으로 이루어진다. $\Pi$는 트랜잭션 순서대로 $\Upsilon$을 반복 적용하며, 앞 트랜잭션의 결과 상태가 다음 트랜잭션의 입력 상태가 된다.

```text
σ₀ = 블록 시작 처리(σ)
σ₁ = Υ(σ₀, T₀)
σ₂ = Υ(σ₁, T₁)
...
σₙ = Υ(σₙ₋₁, Tₙ₋₁)
σ' = 블록 종료 처리(σₙ)
```

트랜잭션 실행의 앞뒤에는 블록 단위 처리가 있다. 블록 단위 처리는 특정 트랜잭션에 속하지 않고 Protocol이 직접 수행하며, 주로 Consensus Layer와 Execution Layer가 정보를 주고받는 데 쓰인다.

- **블록 시작 처리:** Protocol은 트랜잭션을 실행하기 전에 System Call(시스템 호출)로 특정 시스템 컨트랙트를 직접 호출해 블록 정보를 기록한다. System Call은 트랜잭션이 아니므로 서명과 수수료가 없다. Cancun 이후 EIP-4788은 부모 Beacon Block Root(Consensus Layer 블록의 Root Hash)를 기록하고, Prague 이후 EIP-2935는 부모 블록 Hash를 기록한다. 컨트랙트는 이 값을 읽어 Consensus Layer 정보나 과거 블록 Hash를 확인할 수 있다.
- **블록 종료 처리:** 모든 트랜잭션을 실행한 뒤에는 두 방향의 처리가 일어난다.
    - **Consensus Layer에서 Execution Layer로:** Shanghai 이후 Withdrawal(인출)을 처리한다. Validator가 Stake한 Ether를 출금하는 처리로, 대상 주소의 잔액을 늘린다. EVM을 실행하지 않으며 Gas도 쓰지 않는다.
    - **Execution Layer에서 Consensus Layer로:** Prague 이후 요청(Requests)을 모아 Consensus Layer에 전달한다. 예치 요청(EIP-6110)은 Deposit Contract의 Log에서 모으고, 출금 요청(EIP-7002)과 통합 요청(EIP-7251)은 System Call로 모은다.

The Merge 이후 Execution Layer에는 블록 보상이 없다. Validator 보상은 Consensus Layer에서 처리된다. Priority Fee 지급과 Base Fee 소각은 블록 단위가 아니라 각 트랜잭션의 $\Upsilon$ 안에서 일어난다.

블록 실행 전 상태 $\sigma$를 Pre-State, 실행 후 상태 $\sigma^\prime$를 Post-State라고 한다. Pre-State의 Root Hash는 부모 블록의 State Root이고, Post-State의 Root Hash는 해당 블록의 State Root다. 그 사이의 $\sigma_1, \ldots, \sigma_n$은 계산 중에만 존재하며 블록에 기록되지 않는다. Byzantium(EIP-658) 이후 Receipt에도 트랜잭션별 중간 State Root 대신 성공 여부(status)만 들어간다. 따라서 상태 전이의 검증은 블록 단위로 이루어진다.

**State Transition Function은 결정론적이다.** 같은 Pre-State, 같은 블록, 같은 Hard Fork 규칙이 주어지면 어떤 노드에서도 같은 Post-State, 즉 같은 State Root가 계산된다.

```text
Identical Pre-State
+ Identical Block (Transaction Order, Withdrawals, Header Fields)
+ Identical State Transition Rules (Same Fork)
= Identical stateRoot
```

결국 **블록체인은 각 블록의 트랜잭션과, 그 실행 결과 상태를 대표하는 State Root를 연속적으로 기록한 구조**다. 상태 전체는 블록에 기록되지 않는다.


### Transaction and Block Execution Pipeline

State Transition 중 EVM이 담당하는 부분은 $\Upsilon$ 안의 코드 실행($\Xi$)이다. 스마트 컨트랙트는 Solidity나 Vyper와 같은 고급 언어로 작성되며 컴파일 과정을 통해 EVM 바이트코드로 변환된다. EVM 사양은 Program Counter가 가리키는 Opcode를 하나씩 실행하는 의미론으로 정의되며, Client는 이를 Interpreter(인터프리터) 등으로 구현한다.

EVM의 실행 모델은 **Machine State(Program Counter, Stack, Memory, 가용 Gas)와 실행 대상 코드**로 구성되며, 실행 중 계정의 Storage에 접근한다. Cancun 이후에는 트랜잭션 종료 시 초기화되는 Transient Storage(EIP-1153)도 사용할 수 있다.


![evm](evm.png)
_출처: [https://ethereum.org/developers/docs/evm/](https://ethereum.org/developers/docs/evm/)_

- **Program Counter(PC):** 현재 실행 중인 바이트코드 위치(Offset)를 가리키는 포인터로, Opcode가 실행되면 다음 명령 위치로 이동한다. `PUSH1`~`PUSH32`는 뒤따르는 Immediate(즉시값) Byte 수만큼 더 이동한다(`PUSH0`은 즉시값이 없다). `JUMP`는 Stack에서 꺼낸 목적지로 이동하고, `JUMPI`는 조건 값이 0이 아닐 때만 이동한다. 목적지는 반드시 `JUMPDEST`여야 하며, 그렇지 않으면 예외가 발생해 실행이 중단된다.
- **Stack:** 256bit(32byte) 워드를 저장하는 LIFO(Last In, First Out, 후입선출) 구조로, 최대 1024개의 요소를 저장할 수 있다. 각 Opcode는 Stack에서 정해진 개수의 항목을 꺼내고 정해진 개수의 결과를 넣는다(0개일 수 있음). `STOP`, `JUMPDEST`처럼 Stack을 사용하지 않는 Opcode도 있다. 연산은 주로 최상위 값을 대상으로 하며, DUP 계열은 위에서 16번째 항목까지, SWAP 계열은 위에서 17번째 항목까지 접근할 수 있다.
- **Memory:** Byte 단위로 주소가 지정되는 휘발성 선형 공간으로, 연산 중간값, ABI(Application Binary Interface) 인코딩 및 디코딩, 반환값 등 임시 데이터에 사용된다. Call Frame(호출 프레임, 메시지 호출 또는 컨트랙트 생성 하나) 단위로 빈 상태로 새로 할당되며 해당 호출이 끝나면 폐기된다. 32byte 워드 단위로 확장되며, 확장 비용은 크기에 따라 이차적으로 증가한다.
- **Storage:** 계정별 영구 저장소로, 256bit 키를 256bit 값에 대응시키는 매핑이다. 상태 변수, 매핑, 배열 등이 저장되며 `SLOAD`와 `SSTORE` Opcode로 접근한다. 트랜잭션이 끝난 뒤에도 World State에 남으므로 접근 비용이 높고, 특히 값 변경에는 높은 Gas 비용이 부과된다.
- **Code:** 실행 대상 EVM 바이트코드로, 호출된 계정의 `codeHash`에 대응하는 코드 또는 컨트랙트 생성 시의 Init Code(초기화 코드)이다. 실행 중 Read-Only(읽기 전용)로 취급되며, Program Counter는 이 코드를 따라 Opcode를 읽고 실행한다. `DELEGATECALL`이나 EIP-7702 위임에서는 다른 계정의 코드를 현재 계정의 Storage와 Balance 맥락에서 실행한다.

**Gas**

Machine State의 가용 **Gas**는 계산 자원의 남용을 방지하기 위한 EVM의 비용 모델을 이룬다. Gas는 연산, Storage 접근, 데이터 크기 같은 자원 사용량을 근사한 비용 단위다. 모든 Opcode에는 사전에 정의된 Gas 비용이 존재하며, Memory 확장이나 Storage 접근 이력(EIP-2929의 Cold/Warm 구분 등)에 따라 추가 비용이 붙는 Opcode도 있다.

실행 중 Gas가 소진되면 **Out-of-Gas(가스 부족) 예외**가 발생하여 해당 Call Frame의 실행이 즉시 중단된다. 이 경우 **그 Call Frame에서 발생한 상태 변경은 모두 되돌려지며(Revert), 그 프레임에 할당된 Gas는 모두 소모된다.** 반면 Byzantium 이후의 `REVERT` Opcode로 실행을 명시적으로 되돌리면, 상태 변경은 되돌려지지만 남은 Gas는 반환된다.

하위 호출이 실패하면 상위 호출은 실패 결과를 받고 실행을 계속할 수 있다. Tangerine Whistle(EIP-150) 이후 하위 호출에는 남은 Gas의 최대 63/64만 전달되므로, 하위 호출이 Gas를 모두 소진해도 상위 호출에는 Gas가 남는다. 최상위 호출에서 예외가 발생하면 트랜잭션의 실행 결과가 되돌려지지만, **트랜잭션은 블록에 포함되고 수수료 지불과 Nonce 증가는 남는다.** 이러한 메커니즘은 무한 Loop 실행과 DoS(Denial of Service, 서비스 거부 공격)를 방지한다.


### What is Opcode?

- https://www.evm.codes/

![opcode](opcode.png)
_출처: [https://ethereum.org/developers/docs/evm/#opcodes](https://ethereum.org/developers/docs/evm/#opcodes)_

Opcode(Operation Code)는 **"이 명령이 어떤 연산을 수행해야 하는지"를 지정하는 숫자 코드**이다. 하나의 Instruction(명령어)은 보통 Opcode + Operand(연산 대상/주소)로 이루어지며, 이 중 Opcode가 연산 종류(ADD, JUMP 등)를, Operand가 그 연산에 쓰일 데이터나 주소를 담는다.

CPU(Central Processing Unit, 중앙 처리 장치)에서 `ADD`, `SUB`, `LOAD`, `STORE`, `JMP` 등은 명령어의 Mnemonic(니모닉)이며, 실제 기계어 명령어는 Opcode 필드와 Operand 필드로 구성된 특정 Bit 패턴으로 인코딩된다. 제어 장치는 Memory에서 명령어를 읽어(Fetch) Opcode Bit를 해석(Decode)하고, 그에 맞는 제어 신호 또는 마이크로 연산 시퀀스를 생성해 실행한다.

EVM에서는 스마트 컨트랙트가 바이트코드로 표현되고, 이 바이트코드는 **1byte Opcode와 일부 Opcode 뒤에 붙는 Immediate**로 구성된다. 예를 들어 0x01은 `ADD`, 0x55는 `SSTORE`처럼 각 1byte 값이 Stack 조작, 산술 연산, Storage 접근, 컨트랙트 호출 등 특정 동작을 의미한다. **EVM은 Stack 기반 머신이**므로 대부분의 Opcode는 Operand를 명령어 안에 갖지 않고 Stack에서 꺼내 사용한다. Immediate를 갖는 대표적인 예외는 `PUSH1`(0x60)부터 `PUSH32`(0x7F)까지로, Opcode 뒤에 1~32byte의 Immediate가 붙는다. 따라서 바이트코드의 모든 Byte가 Opcode인 것은 아니며, Immediate 안에 포함된 0x5B는 `JUMPDEST`로 인정되지 않는다.

계산 관련 Opcode(`ADD`, `MUL`, `AND`, `OR` 등)는 산술 연산과 비트 연산을 수행하고, 제어 흐름 Opcode(`JUMP`, `JUMPI`)는 무조건 분기와 조건 분기를 수행하며 이를 조합해 Loop를 구현한다. Storage와 Memory Opcode(`SLOAD`/`SSTORE`, `MLOAD`/`MSTORE`)는 각각 Storage와 Memory에 대한 읽기와 쓰기를 담당한다. 환경 및 블록 정보 Opcode(`BALANCE`, `BLOCKHASH` 등)는 계정 잔액, 블록 해시 같은 실행 환경 정보를 조회하고, 호출 및 생성 Opcode(`CALL`, `CREATE` 등)는 외부 호출과 컨트랙트 생성을 수행한다. 
해시 계산은 `KECCAK256` Opcode가 담당한다. 반면 ECDSA(Elliptic Curve Digital Signature Algorithm, 타원곡선 디지털 서명 알고리즘) 서명자 주소 복구(`ecrecover`)와 BN254 곡선 연산 등은 Opcode가 아니라 특정 주소의 Precompile로 제공되며, `CALL` 계열 Opcode로 호출한다.

#### 블록 실행 결과

블록의 모든 처리가 끝나면 Execution Layer는 결과를 블록 헤더에 기록한다. 블록 헤더에는 **실행해야 얻을 수 있는 결과값**과 **블록 입력을 요약한 값**이 함께 들어 있으므로 둘을 구분해야 한다.

| 구분 | 헤더 값 | 계산 방법 |
|---|---|---|
| 실행 결과 | State Root | Post-State의 MPT Root Hash |
| 실행 결과 | Receipts Root | 각 트랜잭션 Receipt로 만든 Trie의 Root Hash |
| 실행 결과 | Logs Bloom | 모든 Log를 요약한 2048bit 값. 특정 Log의 포함 가능성을 빠르게 확인하는 데 쓴다 |
| 실행 결과 | Gas Used | 블록의 모든 트랜잭션이 사용한 Gas 합계 |
| 실행 결과 | Requests Hash (Prague 이후) | 요청 처리 결과의 Hash |
| 입력 요약 | Transactions Root | 블록의 트랜잭션 목록으로 만든 Trie의 Root Hash |
| 입력 요약 | Withdrawals Root (Shanghai 이후) | 블록의 Withdrawal 목록으로 만든 Trie의 Root Hash |

Receipt(영수증)는 트랜잭션 하나의 실행 결과 기록으로, 성공 여부, 누적 Gas 사용량, Log, Logs Bloom을 담는다. Log는 컨트랙트가 `LOG0`~`LOG4` Opcode로 남기는 이벤트 기록이다.

**입력 요약 값은 실행하지 않아도 계산할 수 있고, 실행 결과 값은 블록을 실행해야만 얻을 수 있다.** 따라서 입력 요약 값으로는 블록 내용이 변조되지 않았는지를, 실행 결과 값으로는 실행이 올바른지를 확인한다.

### End-to-End Block Verification Process

1. **트랜잭션 수신과 Mempool 사전 검증**

    Mempool(트랜잭션 대기 공간)은 노드가 블록에 포함되기 전의 트랜잭션을 보관하는 공간이다. 노드는 트랜잭션을 받으면 자원 낭비를 막기 위해 Mempool에 넣기 전에 사전 검증을 수행한다. 아래는 EIP-1559 Type-2 트랜잭션 기준이며, Mempool 정책은 Client마다 다를 수 있다.

    ```text
    1. 서명 검증: (y_parity, r, s)로 발신자 주소를 복원하고, Chain ID가 네트워크와 일치하는지 확인
    Malleability 방지: s ≤ secp256k1n / 2 (EIP-2)
    2. Nonce 확인: nonce ≥ 발신자 계정의 현재 Nonce
    3. Gas 확인: gasLimit ≥ max(Intrinsic Gas, Calldata Floor 비용) (EIP-7623, Pectra 이후)
                gasLimit ≤ 2^24 (EIP-7825, Fusaka 이후)
                maxFeePerGas ≥ maxPriorityFeePerGas
    4. 잔액 확인: balance ≥ value + gasLimit × maxFeePerGas
    ```

    **Intrinsic Gas**는 코드 실행 전에 트랜잭션 자체에 부과되는 기본 Gas다. 기본 21,000 Gas에 Calldata, Access List, 컨트랙트 생성 비용이 더해진다. Intrinsic Gas는 `gasLimit` 안에 포함되므로 잔액 조건에 따로 더하지 않는다.

    **Mempool 검증은 블록 유효성의 일부가 아니다.** 블록에서 실행할 때는 위 항목을 모두 다시 확인하며, 다음 조건이 추가된다.

    - Nonce가 계정의 현재 Nonce와 **정확히** 일치해야 한다.
    - `maxFeePerGas ≥` 해당 블록의 `baseFee`여야 한다.
    - `gasLimit`이 블록의 남은 Gas 이하여야 한다.
    - 발신자 계정에 Code가 없어야 한다(EIP-3607). Pectra 이후 Delegation Indicator를 가진 EOA는 예외다.

2. **블록 생성자의 블록 실행**

    블록은 각 Slot(12초 단위의 시간 구간)마다 Block Proposer로 선정된 Validator가 제안한다. 블록의 실행 내용(Execution Payload)은 Proposer의 Execution Client가 직접 구성하거나, Protocol 외부 시스템인 MEV-Boost를 통해 Builder(블록 빌더)가 구성한다. MEV는 Maximal Extractable Value(최대 추출 가능 가치)의 약자다. Glamsterdam의 ePBS(enshrined Proposer-Builder Separation, EIP-7732)가 적용되면 Builder의 역할이 Protocol 안으로 들어온다.
    
    블록 생성자는 Pre-State에서 $\Pi$를 다음 순서로 실행한다.

    1. **블록 시작 처리:** EIP-4788(Cancun 이후)과 EIP-2935(Prague 이후)의 시스템 호출을 수행한다.
    2. **트랜잭션별 실행($\Upsilon$):**
        1. 4-1의 블록 규칙을 확인한다. 실패하면 그 트랜잭션은 블록에 포함될 수 없다.
        2. 발신자 잔액에서 `gasLimit × effectiveGasPrice`를 선차감하고 Nonce를 1 증가시킨다. **이 두 변경은 실행이 실패해도 되돌려지지 않는다.**
            - `effectiveGasPrice = min(maxFeePerGas, baseFee + maxPriorityFeePerGas)`
        3. Checkpoint(되돌림 기준점)를 만든다.
        4. Intrinsic Gas를 차감한다.
        5. `value`를 이체한다.
        6. 실행 대상에 따라 처리한다.
            - 수신자에 Code가 있으면 EVM이 실행한다.
            - Code가 없으면 이체만 일어난다.
            - 수신자 주소가 비어 있으면 Init Code를 실행해 새 컨트랙트를 만든다.
        7. 최상위 Call Frame이 예외나 `REVERT`로 끝나면 Checkpoint로 되돌린다. 이체와 Storage 변경은 사라지고, 선차감과 Nonce 증가는 남는다.
        8. Gas를 정산한다.
            - 환급액은 `min(환급 카운터, gasUsed / 5)`다(EIP-3529, London 이후).
            - Pectra 이후 최종 `gasUsed = max(gasUsed − 환급액, Calldata Floor 비용)`이다(EIP-7623).
            - 발신자는 `(gasLimit − 최종 gasUsed) × effectiveGasPrice`를 돌려받는다.
            - Fee Recipient는 `(effectiveGasPrice − baseFee) × gasUsed`를 받고, `baseFee × gasUsed`는 소각된다.
        9. Receipt를 생성한다.
    3. **블록 종료 처리:** Withdrawal(Shanghai 이후)을 처리하고, 요청(Prague 이후)을 모은다.
    4. **헤더 기록:** 3장의 "블록 실행 결과" 값을 계산해 블록 헤더에 기록한다.

3. **검증 노드의 Re-execution**

    검증 노드의 Consensus Client는 블록을 받으면 Engine API의 `engine_newPayload`로 Execution Payload를 Execution Client에 전달한다. **Execution Client는 블록 생성자가 제공한 결과를 신뢰하지 않는다.** 다음 순서로 직접 확인한다.

    1. **헤더 사전 검사:** 실행 전에 헤더 값을 확인한다.
        - 부모 Hash와 블록 번호가 올바른지, Timestamp가 부모 블록보다 큰지 확인한다.
        - Gas Limit 변화가 부모 대비 1/1024 미만인지 확인한다.
        - Base Fee가 부모 블록으로부터 계산한 값과 같은지 확인한다.
        - Fusaka 이후에는 EIP-7934의 블록 크기 제한도 확인한다.
    2. **입력 요약 확인:** Execution Payload의 트랜잭션 목록과 Withdrawal 목록으로 Transactions Root와 Withdrawals Root를 계산해 헤더를 재구성하고, 그 Hash가 블록 Hash와 같은지 확인한다.
    3. **Re-execution(재실행):** 부모 블록의 State Root가 가리키는 Pre-State에서 4-2와 같은 절차로 블록을 다시 실행한다.
    4. **실행 결과 비교:** 계산한 실행 결과 값이 블록 헤더 값과 모두 일치하는지 확인한다.

    $$\mathrm{Root}(\Pi(\sigma, B)) \stackrel{?}{=} \texttt{stateRoot}$$

    State Root 외에 Receipts Root, Logs Bloom, Gas Used, Requests Hash도 일치해야 한다. **하나라도 다르면 블록은 무효다.** Execution Client는 결과(`VALID` 또는 `INVALID`)를 Consensus Client에 돌려준다.

    Re-execution은 동기화를 마친 Full Node가 수행한다. Light Client는 블록을 실행하지 않고, 블록 헤더와 Merkle Proof로 필요한 값만 확인한다.

4. **Attestation과 Finality**

    각 Validator는 Epoch(32 Slot, 약 6.4분)마다 배정된 Slot 하나에서 Attestation(투표)을 보낸다. Attestation에는 두 가지 투표가 함께 들어 있다.

    - **Head 투표:** 자신이 보는 체인의 최신 블록에 대한 투표로, LMD-GHOST가 사용한다.
    - **Checkpoint 투표:** Source Checkpoint와 Target Checkpoint에 대한 투표로, Casper FFG가 사용한다.

    Validator는 Execution Client가 유효하다고 판단한 블록이 포함된 체인에만 투표한다.

    Finality는 블록 하나가 아니라 **Checkpoint 단위**로 결정된다. Checkpoint는 각 Epoch 첫 Slot의 블록이며, 그 Slot이 비어 있으면 그 이전의 가장 최근 블록이다.

    1. 전체 Stake의 2/3 이상이 투표한 Checkpoint는 **Justified(정당화)** 상태가 된다.
    2. Justified Checkpoint를 Source로 삼아 바로 다음 Epoch의 Checkpoint가 Justified되면, 앞의 Checkpoint는 **Finalized(최종 확정)** 상태가 된다.
    3. Finalized Checkpoint와 그 이전 블록은 모두 확정된다.

    정상적인 경우 Checkpoint 블록은 약 2 Epoch(12.8분) 뒤에 Finalized된다. Epoch 중간에 있는 블록은 다음 Epoch의 Checkpoint가 Finalized될 때 함께 확정되므로, 블록 위치에 따라 최대 약 3 Epoch(19.2분)가 걸린다.

5. **설계상 비용과 제약**

    Re-execution 방식은 검증의 독립성을 보장하지만, 검증하는 노드 수만큼 같은 계산이 반복되는 **Computational Redundancy**(계산 중복)가 발생한다. 이 반복은 Validator 단위가 아니라 노드 단위로 일어난다. 하나의 노드에 여러 Validator를 연결해 운영할 수 있으므로 Validator 수는 노드 수보다 훨씬 많다. Validator를 운영하지 않는 Full Node도 같은 계산을 수행한다.

    이 설계는 이더리움의 핵심 원칙인 **"Don't Trust, Verify"** 철학에 기반한다. 각 노드가 독립적으로 다시 실행하므로, 블록 생성자가 잘못된 State Transition을 담은 블록을 제안해도 정직한 노드는 Re-execution 과정에서 이를 발견하고 거부한다.

    **누구나 일반적인 하드웨어로 Re-execution을 할 수 있도록 블록당 실행량은 Block Gas Limit으로 제한된다.** Block Gas Limit은 고정된 값이 아니다. Block Proposer가 블록마다 부모 블록 대비 1/1024 미만 범위에서 조정하며, Validator들은 통상 일반 하드웨어가 Slot 시간 안에 검증을 마칠 수 있는 수준을 고려해 목표값을 정한다.

    일부 노드에 장애가 발생해도 다른 노드가 검증과 블록 제안을 계속하므로 체인은 진행된다. 다만 전체 Stake의 1/3을 초과하는 Validator가 오프라인이 되면 2/3 조건을 채울 수 없으므로 Finality가 멈추고, 온라인 Validator에 의한 블록 생성만 계속된다. Finality가 4 Epoch(약 25.6분) 넘게 성립하지 않으면 Inactivity Leak(비활성 누수)이 시작된다. Inactivity Leak은 오프라인 Validator의 Stake를 점차 줄이며, 온라인 Validator의 비율이 2/3를 회복하면 Finality가 재개된다.


<br />
<br />

## zkEVM

zkEVM(Zero-Knowledge Ethereum Virtual Machine)은 EVM 실행이 올바르게 수행되었음을 암호학적 증명으로 검증하는 시스템이다. 대표적인 사용처는 L2 ZK 롤업으로, 롤업은 "Off-chain Execution(오프체인 실행) + L1 On-chain Proof Verification(온체인 증명 검증)" 구조로 L2 트랜잭션을 처리한다. 이더리움 L1에서도 Global Re-execution(네트워크 전역 재실행) 기반 검증을 증명 검증으로 대체하는 방향이 연구되고 있지만, 아직 메인넷에는 도입되지 않았다(아래 '이더리움 L1의 zkEVM 도입 현황' 참고). zkEVM은 이더리움과 동일하게 State Transition을 검증의 기본 단위로 사용한다. 이더리움과 완전히 동일한 규칙을 증명하는 zkEVM(Type 1)은 State Transition Function $$\Pi(S, T) = S'$$ 자체를 변경하지 않는다. 다시 말해, 주어진 초기 상태 $$S$$에 대해 트랜잭션 집합 $$T$$를 적용했을 때 최종 상태 $$S'$$로 전이되는 Execution Semantics(실행 의미론)는 Ethereum의 EVM과 동일하게 유지하되, 그 전이가 올바르게 수행되었는지를 확인하는 검증 메커니즘만 암호학적 증명으로 대체하는 것이다. 다만 구현에 따라 상태 트리, 해시 함수, Gas 규칙 등을 바꾼 zkEVM도 있다(Type 2~4). 이 구조에서 zkEVM은 "주어진 초기 상태 $$S$$에서 트랜잭션 집합 $$T$$를 적용하여 최종 상태 $$S'$$로의 전이가 EVM 규칙을 정확히 따랐다"는 사실만을 간결한 증명으로 요약해 제출하며, Verifier(검증자)는 중간 실행 과정을 다시 계산하지 않고도 State Transition의 정당성을 검증할 수 있다. 여기서 활용되는 핵심 성질은 정보를 숨기는 영지식성보다, 증명이 짧고 빠르게 검증된다는 Succinctness(간결성)이다. 롤업은 Data Availability(데이터 가용성)를 위해 트랜잭션 데이터를 L1에 별도로 게시한다.

### zkEVM을 통한 트랜잭션 검증 방식

zkEVM에서는 소수의 Prover(증명자)가 블록 또는 트랜잭션 배치를 Off-chain에서 실행하고, EVM 바이트코드를 Opcode 단위로 순차적으로 실행하는 과정에서 발생하는 상태 변화를 추적하여 Execution Trace를 생성한다. 이 Execution Trace에는 각 스텝별 Stack 값, Memory 접근, Storage 읽기와 쓰기, 호출과 반환(Call/Return) 등의 세부적인 중간 상태가 포함되며, 이러한 세부 정보는 On-chain에 공개되지 않는다. EVM의 실행 규칙은 미리 수학적 제약으로 변환(Arithmetization, 산술화)되어 Polynomial Constraints(다항식 제약)로 표현되고, Execution Trace는 이 제약을 만족해야 하는 Witness(증인) 데이터가 된다. 즉, "ADD Opcode면 Stack 두 값을 더해야 한다" 같은 규칙을 방정식으로 미리 만들어 두고, Prover는 자신이 만든 Trace가 이 제약 조건들을 모두 만족한다는 증명 $$\pi$$를 생성한다.

Verifier(L2 롤업에서는 L1의 검증 스마트 컨트랙트)는 블록 내 트랜잭션을 다시 실행할 필요 없이, Public Input(공개 입력: 이전 State Root $$S_{\text{root}}$$, 트랜잭션 배치 Commitment, 이후 State Root $$S'_{\text{root}}$$)과 제출된 증명 $$\pi$$만 수학적으로 검증함으로써 실행의 정합성(Correctness)을 확인한다. 증명이 유효한 경우, $$T$$를 적용한 State Transition $$S \to S'$$가 EVM 규칙을 정확히 준수했음이 암호학적으로 보장되며, 이 검증 과정의 비용은 동일한 트랜잭션 집합을 모든 노드가 전부 다시 실행하는 비용에 비해 매우 작도록 설계된다. 그 결과, 기존 이더리움의 "Execution × N 노드" 구조와 비교했을 때, zkEVM 기반 구조에서는 "Execution × Prover(소수) + Verification × N 노드" 형태로 바뀌어 네트워크 전반에서 발생하던 계산 중복을 크게 줄이면서도 State Transition의 안전성과 무결성을 유지할 수 있다.

zkEVM은 강력한 보안과 확장성을 제공하지만, 현실적인 제약도 존재한다. 먼저, 증명 생성 비용이 높다. EVM의 복잡한 로직을 Circuit(회로, 수학적 제약의 집합)으로 표현하면 Execution Trace가 비대해지며, 이를 증명하는 데 많은 연산 자원과 시간이 소모된다. 제약식의 종류는 EVM 규칙으로 고정되어 있고, 실행 스텝이 늘어나면 Trace의 행 수가 실행 스텝 수에 비례해 늘어난다. 같은 연산이라도 Re-execution보다 증명 생성이 훨씬 느리고 비싸며, 그 배수는 증명 시스템, 하드웨어, 작업량에 따라 크게 달라진다. 이더리움 재단(EF)은 2025년 말 L1 블록을 슬롯 시간 안에 증명하는 Real-Time Proving(실시간 증명)에 도달했다고 발표했지만, 이를 위해서는 GPU 클러스터 수준의 장비가 필요하다. 특히 Keccak-256 해시와 ECDSA 서명 검증(`ecrecover`) 같은 무거운 연산이 많은 트랜잭션은 "Gas는 저렴하지만 증명은 매우 비싼 Prover Killer"가 된다. 또한, 증명 방식 사이에는 개발 편의성과 성능 사이의 Trade-off가 있다. RISC Zero와 같은 zkVM 계열은 일반 프로그램을 그대로 증명할 수 있어 범용성이 높지만, CPU 명령어 단위로 증명하므로 네이티브 실행 대비 증명 생성 오버헤드가 크다. 반대로 gnark나 ZoKrates 같은 Circuit 기반 SNARK 도구는 특정 계산에 맞춘 Circuit으로 증명 생성 효율을 높일 수 있지만, Circuit을 직접 설계해야 하고 로직이 바뀔 때마다 Circuit을 다시 만들어야 하며, Groth16을 쓰면 Circuit마다 Trusted Setup(신뢰 설정)도 다시 해야 한다. 더해서 단일 Prover(또는 소수) 구조는 Liveness 위험을 가진다. 즉, Sequencer나 Prover를 운영하는 주체에 문제가 생기면 상태 확정이 지연된다. 예를 들어 Polygon zkEVM은 2024년 3월 L1 Reorg를 Synchronizer가 제대로 처리하지 못해 증명할 수 없는 배치가 생기면서 약 10시간 동안 블록 생성이 멈춘 바 있다. 이 장애의 원인은 증명 생성 속도가 아니라 운영 소프트웨어의 오류였다.

### zkEVM 검증 상세 과정

![zkevm](zkevm.svg)
_출처: [https://zkevm.ethereum.foundation/blog/zkevm-security-overview](https://zkevm.ethereum.foundation/blog/zkevm-security-overview)_


1. **Execution Trace 생성**
    
    먼저 Sequencer(또는 Executor)가 트랜잭션 배치를 실행하여 EVM State Transition을 계산한다. zkEVM은 단순히 최종 결과 상태만 계산하지 않고, 실행 과정 전체를 Execution Trace 형태로 기록한다. 이 Execution Trace는 각 Opcode 실행 단계에서의 상태 변화를 시간 순서대로 저장한 것으로, Program Counter(PC), 실행 Opcode, Stack 값, Memory와 Storage 접근, Gas 소비량 등이 포함된다. 실제 구현은 매 행에 Memory와 Storage 전체를 복사하지 않고, 해당 스텝의 접근 기록만 남긴 뒤 별도 테이블과 Lookup으로 일관성을 확인한다.

    
    ```text
    trace[t] = {
      PC_t: Program Counter
      opcode_t: 실행된 opcode
      stack_t: 이 스텝에서 읽고 쓴 Stack 값
      memory_t: 이 스텝의 Memory 접근 (주소, 값, 읽기/쓰기)
      storage_t: 이 스텝의 Storage 접근 (키, 이전 값, 새 값)
      gas_t: 남은 Gas
    }
    ```
    
    즉, zkEVM은 결과(State Root)뿐 아니라 "어떻게 그 결과에 도달했는가"를 증명하기 위한 Witness를 수집한다.
    
    예를 들어 바이트코드 `60 02 60 03 01 00`(PUSH1 2, PUSH1 3, ADD, STOP)은 다음과 같이 기록된다. Stack은 실행 후 상태이며 오른쪽이 최상단이다.
    
    ```text
    //예시
    step 0: PC=0, opcode=PUSH1 0x02, stack=[2]
    step 1: PC=2, opcode=PUSH1 0x03, stack=[2, 3]
    step 2: PC=4, opcode=ADD,        stack=[5]
    step 3: PC=5, opcode=STOP
    ```

    `PUSH1`은 Immediate 1바이트를 포함해 2바이트를 차지하므로 다음 PC는 2다.
    
2. **Arithmetization: 실행 규칙의 수학적 변환**
    
    Execution Trace는 State Machine(상태 기계)으로 해석되며, 각 단계가 EVM 규칙을 위반하지 않았음을 검증할 수 있도록 EVM의 실행 규칙을 미리 Polynomial Constraints로 변환해 둔다(Arithmetization). 아래 예시는 개념을 보여주기 위해 단순화한 것이다.
    
    - 예시
        
        ```text
        ADD opcode 규칙:
        stack_new[0] = (stack_old[0] + stack_old[1]) mod 2^256
        
        → 다항식 제약식 (개념):
        stack_new[0] − (stack_old[0] + stack_old[1]) + carry × 2^256 = 0,  carry ∈ {0, 1}
        
        증명 시스템의 유한체(예: BN254 스칼라체는 약 254비트)는 2^256보다 작으므로,
        실제 구현은 256비트 값을 작은 조각(limb)으로 나누고
        조각별 자리올림(carry)과 범위 확인(range check)을 제약으로 추가한다.
        ```
        
        ```text
        SSTORE 규칙:
        storage_new[slot] = value
        
        → 다항식 제약식 (개념):
        storage_new[slot] − value = 0
        
        실제로는 이 한 줄로 부족하다. 어느 계정의 Storage인지, 이전 값이 실제 상태에
        있던 값인지(Merkle proof), 접근 순서, Gas와 revert 규칙, 최종 State Root까지
        연결하는 제약이 함께 필요하다.
        ```
    
    일반화하면 모든 인접한 실행 단계는 다음 조건을 만족해야 한다.
    
    $$
    P_i\big(\mathrm{trace}[t],\ \mathrm{trace}[t+1]\big) = 0 \quad \text{for all } i \text{ and } t \in [0, N-1]
    $$
    
    Memory와 Storage 접근의 일관성, 값의 범위처럼 인접한 두 행만으로 표현하기 어려운 조건은 Lookup이나 Permutation 논증으로 따로 확인한다.
    
3. **Proof Generation**
    
    Prover는 Execution Trace(Witness)와 제약식을 바탕으로 증명을 생성한다. 대규모 Execution Trace를 처리하기 위해 STARK 계열 증명을 쓰는 구현이 많지만, 증명 시스템은 구현마다 다르다. STARK는 Trusted Setup이 필요 없고 대형 Trace 처리에 유리하지만, 증명 크기가 크다는 단점이 있다. 이를 보완하기 위해 증명을 검증하는 계산을 다시 증명하는 Recursion(재귀 증명)이나 Aggregation으로 여러 증명을 하나로 합치고, 최종적으로는 L1에서 저비용으로 검증 가능하도록 작은 크기의 SNARK(예: Groth16)로 감싸는(Wrapping) 구성이 흔하다.
    
    증명이 보장하는 내용은 모든 $$t \in [0, N-1]$$에 대해 $$\mathrm{trace}[t] \to \mathrm{trace}[t+1]$$ 전이가 EVM 규칙을 준수하고, $$\mathrm{trace}[0]$$에서 시작하는 상태가 $$S_{\text{root}}$$와, $$\mathrm{trace}[N]$$까지 실행한 결과 상태가 $$S'_{\text{root}}$$와 일치한다는 것이다. 즉, 전체 실행이 올바른 State Transition이었음이 하나의 증명으로 요약된다.
    
4. **Verification**
    
    마지막으로 생성된 증명과 핵심 상태 정보가 L1의 Verifier 스마트 컨트랙트에 제출되며, 일반적으로 제출되는 데이터는 다음과 같다.
    
    - Pre-State Root ($$S_{\text{root}}$$)
    - Post-State Root ($$S'_{\text{root}}$$)
    - Transaction Batch 데이터 또는 DA 커밋
    - ZK Proof ($$\pi$$)
    
    검증 컨트랙트는 Pre-State Root가 현재 승인된 루트와 같은지 확인하고, 트랜잭션을 다시 실행하지 않고 제출된 증명 $$\pi$$의 수학적 타당성만을 검증한다. 검증이 성공하면 해당 배치의 State Transition은 EVM 규칙을 만족한 것으로 간주되며, 새로운 State Root가 On-chain에 기록된다. 이 상태가 되돌릴 수 없게 확정되는 시점은 검증 트랜잭션이 포함된 L1 블록이 Finalized 상태가 되는 시점이다.
    
    ```text
    Sequencer / Prover:
    1. TX 실행 → batch trace 생성
    2. ZK proof π 생성
    3. (S_root, S'_root, 배치 Commitment, π) L1 제출
    
    L1 Verifier 컨트랙트:
    1. π 검증
    2. S'_root 기록 → L1 Finality 이후 State Transition 확정
    ```

<br />
<br />

## zkVM

zkVM(Zero-Knowledge Virtual Machine)은 특정 로직을 위한 전용 Circuit을 직접 설계하는 대신, RISC-V와 같은 ISA(Instruction Set Architecture, 명령어 집합) 자체를 증명 가능한 가상 머신으로 구현한 범용 플랫폼이다.

zkEVM이 "무엇을 증명하는가(EVM 실행)"를 가리키는 말이라면, zkVM은 "어떤 방식으로 증명하는가(범용 ISA의 실행 증명)"를 가리키는 말이다. zkVM은 Rust, C/C++ 등 고수준 언어로 작성되어 해당 ISA로 컴파일된 프로그램의 실행 무결성을 증명하는 '범용 컴퓨터'의 역할을 수행한다(지원 언어는 zkVM의 툴체인마다 다르다). 쉽게 말해 zkEVM은 "EVM이라는 특정 상태 머신이 올바르게 실행되었는가?"를 증명하고, zkVM은 "일반적인 프로그램이 정의된 ISA 규칙에 따라 올바르게 실행되었는가?"를 증명한다. 두 개념은 배타적이지 않다. zkVM 위에서 EVM 실행 프로그램(예: 이더리움 블록 실행 로직)을 돌리면 그 자체로 zkEVM이 된다. 이때 증명은 프로그램이 올바르게 실행되었음을 보장하며, 프로그램이 이더리움 규칙을 정확히 구현했는지는 별도로 검증해야 한다.

### zkVM을 통한 트랜잭션 검증 방식

개발자는 복잡한 영지식 Circuit 설계 지식 없이도 일반적인 프로그램 로직을 통해 트랜잭션 검증 기능을 구현할 수 있다. 예를 들어, Rust로 작성된 검증 로직을 zkVM 환경에서 구동하면, VM 내부의 각 명령어(Instruction) 실행 단계가 미리 제약으로 변환해 둔 ISA 규칙을 만족한다는 암호학적 증명이 만들어진다. 이를 통해 특정 입력값에 대해 프로그램이 정해진 규칙대로 올바른 결과값을 산출했음을 보장할 수 있다. Prover가 zkVM 내에서 검증 프로그램을 실행하여 실행 결과와 그 정합성을 요약한 하나의 증명을 생성하면, Verifier는 해당 증명만을 검증(Verification)함으로써 실행 결과를 신뢰할 수 있게 된다.

zkVM은 범용적인 검증 환경을 제공한다는 점에서 높은 유연성을 가진다. 트랜잭션 검증 로직을 일반 프로그램 형태로 구현할 수 있기 때문에 복잡한 로직을 비교적 쉽게 증명 시스템에 통합할 수 있다. 하지만 이러한 범용성은 동시에 상당한 성능 비용을 동반한다. zkVM은 일반적인 CPU 명령어 수준에서 프로그램 실행을 증명하기 때문에, 단일 트랜잭션 검증을 위해서도 매우 긴 Execution Trace가 생성될 수 있다. 이는 결과적으로 증명 생성 비용과 증명 생성 시간을 크게 증가시키는 요인이 된다.

특히 범용 zkVM 시스템인 RISC Zero와 같은 플랫폼은 높은 범용성을 제공하는 대신, 네이티브 실행 대비 상당한 증명 생성 오버헤드를 가진다. 이를 줄이기 위해 SHA-256, Keccak, 타원곡선 연산 같은 무거운 연산은 전용 가속 Circuit(Precompile)로 처리한다. 최근에는 L2 롤업과 이더리움 L1 실행 증명 연구 모두 zkVM 기반으로 옮겨가는 추세다. Scroll은 2025년 4월 Euclid 업그레이드에서 기존 halo2 Circuit을 폐기하고 OpenVM 기반 Prover로 전환했고, Taiko는 TEE(SGX) 증명과 RISC Zero, SP1 증명을 조합한 다중 증명을 사용한다.

### zkVM 검증 상세 과정

1. **Program Execution**

    Prover는 트랜잭션 검증 로직이 담긴 프로그램(예: Rust 기반 트랜잭션 검증기)을 zkVM 환경 내에서 실행한다. 이 프로그램은 입력 데이터 검증, State Transition 규칙 적용, 새로운 State Root 계산 등의 작업을 수행한다. zkVM은 프로그램의 실행 과정을 CPU 명령어(Instruction) 단위로 세분화하여 기록한다. 이때 PC(Program Counter), 레지스터 상태 변화, Memory Read/Write 로그 등이 포함된 Execution Trace가 생성된다. Rust나 C++ 같은 고수준 언어의 코드 한 줄은 여러 CPU 명령어로 컴파일되므로, 복잡한 로직일수록 Execution Trace가 길어진다. Trace 길이는 실행된 명령어 수에 비례해 선형으로 늘어난다.

2. **Proof Generation**
    
    기록된 Trace가 해당 CPU의 연산 규칙(Instruction Rules)을 완벽히 준수했음을 암호학적으로 입증하는 단계이다. zkEVM과 마찬가지로 CPU의 State Transition 규칙을 미리 Polynomial Constraints로 정의해 두고(Arithmetization), Trace가 이를 만족함을 증명한다. 이후 대규모 Trace를 빠르게 처리할 수 있고 Trusted Setup이 필요 없는 STARK 기반 초기 증명을 생성하며, 이를 Recursion 기술로 결합하여 하나의 증명으로 합친다. 최종적으로 L1(이더리움 등) On-chain 검증 비용을 최소화하기 위해 고정된 작은 크기를 갖는 SNARK 증명(예: Groth16)으로 변환(Wrapping)하여 결과물을 도출한다. RISC Zero, SP1 등이 이 구성을 쓰지만 모든 zkVM의 공통 구조는 아니다.

3. **Verification**

    Verifier(Off-chain 프로그램 또는 스마트 컨트랙트)는 원본 프로그램을 다시 실행하지 않고, 제출된 증명의 타당성만을 수학적으로 확인한다. RISC Zero를 예로 들면 Verifier는 [Image ID(실행한 프로그램을 식별하는 값), Journal(프로그램이 공개한 출력), Seal(증명)]의 세 가지 요소가 서로 일치하는지 확인한다. 입력은 기본적으로 비공개이며, 공개하려는 입력은 프로그램이 Journal에 기록해야 한다. 증명 검증이 성공하면, 해당 프로그램이 주어진 입력에 대해 수정되거나 왜곡되지 않은 채 원래 설계된 로직대로 실행되어 결과값을 산출했음이 암호학적으로 보장된다.

<br />

### zkEVM과 zkVM 비교

| **구분** | **zkEVM** | **zkVM** |
| --- | --- | --- |
| **목적** | EVM 실행 증명 (L2 롤업, 이더리움 L1 실행 증명 연구) | 범용 프로그램의 실행 증명 (zkVM 위에서 EVM을 실행해 zkEVM을 구현하는 데도 쓰임) |
| **최적화** | 전용 Circuit 방식은 EVM 연산에 맞춰 Circuit을 설계 | CPU 명령어 단위로 증명하므로 오버헤드가 크고, 무거운 연산은 가속 Circuit으로 보완 |
| **복잡도** | 전용 Circuit 방식은 EVM 업데이트 시 Circuit을 수정해야 함 | 프로그램 로직만 바꾸고 다시 컴파일하면 되어 유지보수가 쉬움 |
| **대표 프로젝트** | 전용 Circuit: Linea (Polygon zkEVM은 2026년 7월 종료). zkVM 기반: Scroll(OpenVM), Taiko(TEE(SGX) + RISC Zero, SP1) | RISC Zero, Succinct(SP1), OpenVM, Jolt |
- **zkEVM**에서 Verifier는 "이더리움 블록 안의 트랜잭션들이 상태를 정확히 바꿨다"는 것을 확인한다.
- **zkVM**에서 Verifier는 "어떤 프로그램(Image ID)이 주어진 입력값을 가지고 실행되어 이 결과값을 냈다"는 사실을 확인한다. 이 프로그램이 이더리움 블록 실행 로직이라면 zkEVM처럼 동작하는 것이고, 게임 엔진이라면 게임 결과의 무결성을 증명하는 것이 된다.


### RISC Zero zkVM

RISC Zero는 범용 프로그램의 실행 과정을 ZKP(Zero-Knowledge Proof, 영지식 증명)로 검증할 수 있도록 설계된 zkVM이다. 기존 ZKP 시스템이 특정 연산을 위해 맞춤형 Circuit을 직접 설계하는 방식과 달리, RISC Zero는 일반적인 CPU 실행 모델 자체를 증명 가능한 형태로 구현하여 프로그램 실행의 정합성을 검증한다.

RISC Zero의 핵심 아이디어는 오픈소스 CPU 아키텍처인 RISC-V(RV32IM)를 ZKP 시스템 내부의 가상 머신으로 구현하고, 해당 CPU 위에서 실행된 프로그램의 모든 명령어가 올바르게 수행되었음을 증명하는 것이다. 즉, 특정 스마트 컨트랙트 연산을 개별적으로 증명하는 것이 아니라, "주어진 프로그램이 특정 입력에 대해 RISC-V 명령어 집합의 규칙에 따라 정상적으로 실행되었다"는 사실 자체를 증명한다.

RISC Zero zkVM은 프로그램을 실행한 후, 실행의 정확성을 보장하는 암호학적 증명 객체인 Receipt를 생성한다.

- Journal: 프로그램 실행 결과로 산출된 Public Output(공개된 데이터)
- Seal: 해당 결과값이 특정 프로그램(Image ID)에 의해 올바르게 도출되었음을 입증하는 암호학적 증명

RISC Zero는 연산의 주도권을 Prover에게 부여하고, Verifier는 최소한의 자원으로 결과를 확정하는 Verifiable Computing(검증 가능한 연산) 모델을 따른다. 따라서 Verifier는 프로그램을 다시 실행하지 않고도, 증명(Receipt) 검증만으로 실행 결과의 정당성을 확인할 수 있다.

- 기존 실행 모델: Input → Program Execution → Output
- RISC Zero: Input → Program Execution (RISC-V) → Output + Zero-Knowledge Proof

- https://dev.risczero.com/api/zkvm/

RISC Zero zkVM은 '증명될 대상(Guest)'과 '증명을 관리하는 실행자(Host)'로 명확히 역할이 분리되어 작동한다.
 
- **Guest**: 증명 대상 핵심 프로그램 (Rust/C++ 작성 → RISC-V ELF 컴파일)
- **Host**: Guest 실행 감독(입력 전달, 실행기 구동) → 증명(Receipt) 생성 요청 및 검증. 증명은 로컬 장비나 원격 증명 서비스에서 생성한다.

전체 프로세스는 다음과 같이 진행된다.

1. **Guest 프로그램 설계**

    Rust/C++ 등 지원 언어로 증명하고자 하는 로직(스마트 컨트랙트 검증, Merkle Proof 검증, State Transition(잔고 / Nonce) 등)을 작성한다. 개발자는 Circuit을 정의하지 않는다.

    - 입력: `env::read()` → Host로부터 데이터를 전달받음 (읽은 값은 기본적으로 비공개)
    - 출력: `env::commit()` → 증명과 함께 외부에서 확인할 수 있는 공개 결과물인 Journal에 기록

    실제 Guest 예시:
    ```rust
    #![no_main]
    
    use risc0_zkvm::guest::env;
    
    risc0_zkvm::guest::entry!(main);
    
    pub fn main() {
        // Host에서 입력 받기 (비공개)
        let a: u64 = env::read();
        let b: u64 = env::read();
        
        // 자명하지 않은 인수인지 확인 (0과 1 제외)
        if a < 2 || b < 2 {
            panic!("Trivial factors");
        }
        
        // 계산 (오버플로우 체크)
        let product = a.checked_mul(b).expect("Overflow");
        
        // 결과 공개 (Journal)
        env::commit(&product);
    }
    ```

2. **Guest 컴파일 및 ImageID 생성**

    작성된 코드는 일반적인 CPU가 아닌 zkVM이 해석 가능한 RISC-V ELF 바이너리 형태로 컴파일된다. 보통 `methods` 크레이트의 빌드 스크립트(`methods/build.rs`, `risc0-build`)가 컴파일하며, 재현 가능한 빌드가 필요하면 `cargo risczero build`를 쓴다.
 
    ```text
    Rust/C++
    ↓
    risc0-build (또는 cargo risczero build)
    ↓
    RISC-V RV32IM ELF 바이너리 생성
    ↓
    ELF를 zkVM의 초기 Memory 이미지로 적재 (1 kB 페이지 단위, 진입점 주소도 Memory의 정해진 위치에 기록)
    ↓
    페이지 Merkle tree 구성 (Poseidon2 해시)
    ↓
    ImageID = Merkle tree의 루트 다이제스트 (32바이트)
    ```
    
    - **ImageID**: Guest의 초기 Memory 상태 전체를 커밋한 Merkle Root로, Verifier가 "어떤 프로그램이 실행되었는지" 식별하는 데 사용된다. Guest 코드가 바뀌면 ImageID도 바뀐다.

3. **zkVM 실행 및 Execution Trace 생성**

    프로그램이 실제로 구동되면서 발생하는 모든 연산 과정을 수학적으로 증명 가능한 데이터 형태로 기록한다. 이 과정은 Host 환경의 Executor가 담당하며, 물리적 CPU가 아닌 가상화된 RISC-V 인터프리터를 통해 바이너리를 실행하며 시작된다. 

    VM은 내부의 Interpreter Loop(실행 루프)를 돌며 각 명령어가 처리되는 과정을 기록한다. 개념적으로는 매 사이클마다 명령어 실행 전후의 상태 변화를 기록한다고 볼 수 있다. 다만 실제로 전체 Memory와 레지스터를 매 사이클 복사하지는 않으며, 그 사이클에서 읽고 쓴 값을 기록하고 Memory 일관성은 별도의 논증으로 확인한다.

    - **Pre-execution:** 명령 실행 전 상태
        - 실행 중인 명령어의 Memory 주소인 Program Counter(PC), 읽는 레지스터 값(정수 레지스터는 x0~x31의 32개이며 x0은 항상 0), 읽는 Memory 값 등
    - **Opcode 실행:** RISC-V RV32IM 명령어(RV32I 40개 + M 확장 8개, 총 48개: ADD, LW(Load), SW(Store), JAL(Jump) 등) 중 하나
    - **Post-execution:** 명령 실행 후 발생한 변화
        - 연산 결과가 쓰인 레지스터($$\Delta\,\text{Register}$$), Memory에 쓴 주소와 값($$\Delta\,\text{Memory}$$), 다음 PC 등
    
    이렇게 시간 순서대로 나열된 사이클별 데이터는 Execution Trace라 불리는 거대한 행렬 구조의 테이블이 된다. 이 테이블에서 각 행(Row)은 개별 실행 사이클을 의미하며, 각 열(Column)은 PC, Opcode, 레지스터 값, Memory 접근 정보 등을 나타낸다. 이 테이블의 모든 데이터는 이후 Arithmetization 단계를 거쳐 Polynomial Constraints 조건을 만족하는지 검증받게 된다.

    - Execution Trace 테이블 (개념 예시, x10 = 0x2000으로 가정):
    ```text
    | Cycle | PC    | 명령            | 레지스터 쓰기 | MemAddr | Mem 쓰기 |
    |-------|-------|-----------------|---------------|---------|----------|
    | 0     | 0x100 | addi x5, x0, 5  | x5 = 5        | -       | -        |
    | 1     | 0x104 | addi x6, x0, 3  | x6 = 3        | -       | -        |
    | 2     | 0x108 | add  x7, x5, x6 | x7 = 8        | -       | -        |
    | 3     | 0x10c | sw   x7, 0(x10) | -             | 0x2000  | 8        |
    ```

    x0에는 값을 쓸 수 없고, ADD 같은 산술 명령은 Memory를 쓰지 않는다. PC는 분기가 없으면 4씩 증가한다. 실제 사이클 수는 Memory 페이지 로딩(Page-In) 등으로 명령 수와 정확히 같지는 않다.
 
    프로그램의 실행 복잡도가 높아져 연산량이 너무 많아지는 경우, 단일 Trace 테이블로 처리하기에는 Memory 부하가 커질 수 있다. 이를 해결하기 위해 RISC Zero는 실행 과정을 여러 개의 Segment 단위로 분할하여 Trace를 생성하는 Continuations 기술을 적용한다. 이렇게 분할된 각 Segment는 독립적인 증명으로 생성된 뒤, 나중에 Recursion을 통해 하나의 최종 증명(Receipt)으로 합쳐진다.


4. **Arithmetization(산술화)**

    생성된 Execution Trace는 STARK Prover에 의해 수학적 증명으로 변환되는 Arithmetization 과정을 거친다. Trace 테이블의 각 열에 기록된 값들을 다항식의 평가값(Evaluation)으로 간주하고, 보간(실제 구현은 NTT 사용)을 통해 해당 값들을 지나는 다항식을 복원한다. "연산 결과의 정확성", "명령어 주소의 논리적 연속성" 등의 검증 규칙은 Trace의 인접 행 사이 제약(전이 제약)과 첫 행, 마지막 행 제약(경계 제약)으로 표현하며, 이러한 표현 방식을 AIR(Algebraic Intermediate Representation)라고 한다. Guest 프로그램이 정상 실행되었다면 모든 제약 조건 다항식이 Trace의 정의역에서 0이 된다.

    - **AIR Constraints** (정상 실행이면 모두 0):
        - PC Constraint: 분기나 점프가 아니면 $$PC_{n+1} = PC_n + 4$$, 분기나 점프가 실행되면 $$PC_{n+1} = \text{target}_n$$(목적지)
            - RV32IM 명령어는 4바이트이므로 순차 실행 시 PC는 4씩 증가하고, 분기와 점프에서는 목적지로 이동해야 한다.
        - Register Constraint: $$\text{rd}_{n+1} = \mathrm{ALU}(\text{rs1}_n, \text{rs2}_n, \text{imm}) \bmod 2^{32}$$, $$\text{rd} = x_0$$이면 값 변화 없음
            - ADD 명령이 실행되었다면, 결과 레지스터 값은 두 원본 레지스터 값의 합($$\bmod 2^{32}$$)과 같아야 한다.
        - Memory Constraint: 어떤 주소에서 읽은 값 = 그 주소에 마지막으로 쓴 값
            - Memory 접근 기록을 주소와 시간 순으로 정렬한 Permutation 논증 등으로 확인한다.
        - Decoding / Range Constraint: 명령 워드가 유효한 RV32IM 명령으로 디코딩되고, 레지스터와 Memory 값이 32비트 범위 안에 있음


5. **STARK Prover (FRI)**

    Prover는 Trace 다항식들에 커밋한 뒤, FRI(Fast Reed-Solomon Interactive Oracle Proof of Proximity) 프로토콜로 커밋한 값들이 낮은 차수의 다항식에 가깝다는 것(저차 근접성)을 증명한다. 수백만 행 이상의 Trace 다항식을 직접 검증하는 것은 불가능하기 때문에, FRI는 다항식을 반복적으로 '접는(Folding)' 방식을 취한다. 일반적인 설명에서는 짝수 항과 홀수 항을 분리하고 Verifier의 랜덤값을 혼합하여 차수를 절반씩 줄이며, RISC Zero는 한 번에 16배씩 접는다. Verifier는 무작위 위치를 정해진 횟수만큼 조회해(RISC Zero는 50회) 전체 실행 Trace가 제약을 만족하는 저차 다항식에서 나왔음을 높은 확률로 확인한다. RISC Zero Datasheet 기준 Segment 하나의 STARK 증명(Seal) 크기는 200 kB 남짓이다($$2^{15}$$ 사이클에서 201.3 kB, $$2^{16}$$ 사이클에서 213 kB).

    Segment 크기는 `segment_limit_po2`로 2의 거듭제곱 단위로 정하며, 기본값은 $$2^{20}$$(1,048,576) 사이클이다. risc0-zkp는 $$2^{13}$$(약 8천)~$$2^{24}$$(약 1,677만) 사이클을 정의하지만, 기본 검증 설정이 받아들이는 최대 Segment 크기는 $$2^{22}$$다. $$2^{23}$$과 $$2^{24}$$는 기본값이 아닌 검증 설정을 따로 써야 한다. 각 Segment는 독립적으로 STARK 증명을 생성하며, 이는 GPU 가속이나 Boundless 같은 분산 증명 시장을 통해 병렬로 처리될 수 있다. 이전에 쓰이던 Bonsai는 RISC Zero가 운영한 중앙화된 호스팅 증명 서비스였으며, 2025년 12월 종료되었다. 이후 별도의 Recursion Prover(재귀 증명자)가 Segment 증명을 Recursion Circuit용 증명으로 변환(`lift`)하고 둘씩 합치는(`join`) 과정을 반복해, 수많은 Segment 증명을 하나의 Succinct Receipt로 병합한다.

    기본적인 STARK 증명은 생성 속도가 빠르지만 증명 데이터의 크기가 커서 이더리움 On-chain 검증 시 많은 Gas 비용을 발생시킨다. 이를 최적화하기 위해 RISC Zero는 Succinct Receipt를 검증하는 로직을 Groth16 Circuit으로 구현하여, 이더리움 검증에 최적화된 Groth16(SNARK) 형식으로 변환(Wrapping)한다. 이 과정을 거치면 증명 크기가 BN254 기준 256바이트(압축 시 128바이트)로 줄어든다. On-chain 검증 비용은 20만 Gas대다. 이 Groth16 Wrapping Circuit은 Trusted Setup이 필요하며, RISC Zero가 세레머니를 진행했다.

    최종 결과물인 Receipt는 아래 구조체로 구성된다.
    
    ```rust
    pub struct Receipt {
        pub inner: InnerReceipt,       // 내부 증명 (Composite, Succinct, Groth16 등)
        pub journal: Journal,          // 공개 출력 (commit 결과)
        pub metadata: ReceiptMetadata, // SDK 버전 등 메타데이터 (암호학적으로 보증되지 않음)
    }
    ```
    Composite Receipt는 Segment 증명을 모두 담아 실행 길이에 비례해 커지고, Succinct Receipt는 Recursion으로 합친 상수 크기의 STARK 증명이며, Groth16 Receipt는 이를 감싼 Groth16 증명이다.

6. **검증 (Verification)**

    검증의 핵심은 "프로그램을 다시 실행하지 않고도 실행 결과가 진실임을 확인하는 것"이다. 이를 위해 Verifier는 기대하는 `ImageID`, Receipt의 `Seal`(증명), `Journal` 세 가지 요소를 대조한다. ImageID는 Receipt에 별도 필드로 들어 있지 않으며, 증명이 담은 Claim(주장)의 시작 상태 다이제스트가 Verifier가 넘긴 ImageID와 같은지 확인한다.

    - **Off-chain 검증:** Rust SDK 기반 검증
        - Off-chain 검증은 주로 서버 간 통신, 클라이언트 앱, 또는 다른 체인의 노드(Substrate 등)에서 증명을 확인할 때 사용된다.
        - RISC Zero Rust SDK의 `receipt.verify(image_id)` 인터페이스를 호출한다.
        - 일반 CPU에서 빠르게 검증되며, 검증 시간은 Receipt 형태에 따라 다르다.
        - `InnerReceipt`가 Composite, Succinct, Groth16 중 무엇이든 SDK가 형태에 맞는 검증 알고리즘을 수행하고, 프로그램이 정상 종료했는지와 Journal이 증명된 출력과 일치하는지도 함께 확인한다.
    - **On-chain 검증:** 이더리움 같은 블록체인 상의 스마트 컨트랙트(Verifier)가 증명을 검증
        - 이더리움과 같은 블록체인 상의 스마트 컨트랙트가 Verifier가 되는 방식으로, 'Trustless(무신뢰)' 환경에서 연산 결과를 체인에 반영할 때 필수적이다.
        - Solidity Verifier 컨트랙트의 `verify(bytes seal, bytes32 imageId, bytes32 journalDigest)` 함수를 호출한다.
        - 검증 프로세스:
            1. `imageId`와 `journalDigest`로 검증할 Claim을 재구성 → 프로그램 무결성과 공개 출력 연결
            2. `seal`(Groth16 증명) 검증 → 실행 Trace 유효성 확인
            3. 애플리케이션이 Journal 내용(이전 State Root, 배치 번호 등)을 기대값과 비교
        - On-chain 검증기는 STARK를 직접 검증하지 않고 Groth16으로 감싼 증명을 검증하며, 검증 비용은 원래 계산의 크기와 관계없이 일정하다.

⇒ **트랜잭션 검증 로직의 동작 원리**

Guest 프로그램 내부에는 서명 확인, 잔고 체크, Nonce 검증 등 복잡한 비즈니스 로직이 포함된다. 검증 단계에서 하나라도 실패할 경우 `panic!` 또는 `assert!`로 즉시 중단되게 만들 수 있다. 중단된 실행으로는 정상 종료를 증명하는 Receipt를 얻을 수 없으며(`receipt.verify`는 정상 종료만 받아들인다), 모든 검증을 통과했을 때만 마지막 단계인 `env::commit(new_state_root)`이 실행된다. 따라서 Verifier는 Receipt를 확인하는 것만으로 "해당 검증 로직을 모두 통과한 정당한 트랜잭션만이 새로운 상태값(new_state_root)을 생성했다"는 사실을 신뢰하게 된다. 다만 이더리움 State Transition을 증명하는 Guest라면, 서명이나 Nonce가 틀린 트랜잭션(블록을 무효로 만듦)과 실행 중 Revert된 트랜잭션(유효한 트랜잭션이며 수수료와 Nonce 증가가 반영됨)을 구분해야 한다. Revert된 트랜잭션까지 panic으로 처리하면 유효한 블록을 증명할 수 없다.

**장점**

- 특정 DSL(Domain-Specific Language, 도메인 전용 언어)을 학습할 필요 없이 일반적인 Rust/C++로 로직을 작성할 수 있어 기존 라이브러리를 많이 재사용할 수 있다. 다만 zkVM 대상 환경의 제약이 있고, ECDSA나 해시 같은 암호 연산은 가속 Circuit을 쓰도록 패치된 크레이트를 쓰는 것이 좋다.
- 트랜잭션 포맷이 변경되거나 수수료 계산 방식 같은 규칙이 바뀌어도 Circuit을 새로 설계할 필요 없이 Guest 코드만 수정하여 다시 컴파일하면 된다. 이때 새로운 ImageID가 생성되므로, 검증 쪽(On-chain 컨트랙트 등)의 허용 ImageID도 갱신해야 업그레이드가 완료된다.
- 복잡한 반복문(for, while)과 조건문(if)을 일반 코드처럼 사용할 수 있어, 방대한 State Transition 로직을 직관적으로 구현할 수 있다. 증명 비용은 실제 실행한 사이클 수에 비례한다.
- On-chain에서는 `verify(seal, imageId, journalDigest)` 인터페이스 하나로, 계산 크기와 관계없이 일정한 비용(Groth16 검증, 20만 Gas대)으로 트랜잭션 배치의 실행 결과를 확정할 수 있다. 배치 데이터를 L1에 게시하는 비용은 별도다.
- STARK 증명 단계는 Trusted Setup이 필요 없다. On-chain 검증을 위한 Groth16 Wrapping 단계에는 Trusted Setup이 필요하지만, 이 설정은 Wrapping Circuit 하나에 대한 것이라 Guest를 바꿀 때마다 다시 할 필요는 없다.

**단점**

- CPU 명령어(Instruction) 단위로 증명이 이루어지므로, 특정 연산에 맞춰 설계한 전용 Circuit보다 증명 오버헤드가 크다.
- 가상 머신 위에서 명령어를 해석하고 기록하는 과정 때문에 네이티브 실행보다 증명 비용이 크게 늘어나며(배수는 작업량, 가속 Circuit 사용 여부, 장비에 따라 다르다), 블록 시간 안에 증명하려면 GPU 등 고성능 장비나 Boundless 같은 증명 시장이 필요하다.
- 단순 송금이나 1:1 토큰 스왑처럼 연산량이 매우 적은 경우에는 Segment 최소 크기, Recursion, Wrapping 같은 고정 비용 때문에 해당 연산에만 특화된 작은 Circuit보다 비용이 더 들 수 있다.
- 블록 타임이 매우 짧은 고성능 체인(예: 1초 미만)에서는 블록 타임 안에 증명을 생성하기 어렵다.


<br />
<br />

## Circuit 기반 SNARK 도구
 
zkVM이 범용 ISA의 실행을 증명하는 것과 달리, 아래 도구들은 증명할 로직을 개발자가 직접 산술 제약(Circuit)으로 작성한다.

### gnark

https://github.com/Consensys/gnark

gnark는 Consensys가 개발한 Go 기반 zk-SNARK 라이브러리로, 별도의 언어를 배울 필요 없이 Go의 struct와 전용 API(`frontend.API`)를 활용해 산술 Circuit을 설계할 수 있게 해준다. 즉, Go 안에 내장된 DSL로 제약을 작성하는 방식이다. gnark는 Frontend(Circuit 정의)와 Backend(증명 생성)를 분리한 계층 구조를 가지며, 작성된 Go 코드를 R1CS(Groth16용) 또는 Plonkish(SCS, PLONK용) 제약 시스템으로 컴파일한다.

Go의 타입 시스템, 테스트 프레임워크, CI/CD를 그대로 사용할 수 있어 일반 개발 워크플로우에서 ZKP를 자연스럽게 통합할 수 있다. gnark는 BN254, BLS12-381, BLS12-377, BW6-761 곡선을 지원하며, Solidity 검증기 내보내기는 주로 BN254를 대상으로 한다.

gnark는 범용 Circuit 라이브러리로, 특정 연산의 증명부터 전체 EVM State Transition까지 표현할 수 있다. 실제로 Linea ZK 롤업의 Prover가 gnark를 사용한다. Circuit 방식에서는 개발자가 증명할 로직을 직접 산술 제약으로 설계하므로, 특정 연산에 맞춰 제약 수를 줄이는 최적화가 가능하다. 예를 들어 Keccak 해시(EVM에서는 `KECCAK256` Opcode)나 ECDSA 서명 검증(EVM에서는 `ecrecover` Precompile) 같은 연산을 `gnark/std`의 가젯으로 Circuit 안에서 직접 제약으로 구성할 수 있다.

1. **Circuit 설계**
    gnark Circuit 설계는 Go의 `struct`로 표현되며, 개발자는 증명 과정에서 공개될 Public Witness와 Prover만 알고 있어야 하는 Secret Witness를 구조체 태그로 구분한다. 필드는 기본적으로 Secret이며, 공개할 필드에 `gnark:",public"`(쉼표 필요)를 붙인다.
    
    - **Public Witness**: Verifier가 아는 값
        - ex. 이전 상태 Root, 새 상태 Root, 블록 헤더 정보 등
    - **Secret Witness**: 증명에 공개되지 않는 값
        - ex. 트랜잭션들, 각 트랜잭션에 대한 Merkle Proof, 서명 등

    ```go
    type CubicCircuit struct {
        // x는 secret witness (태그의 "x"는 이름이며, 공개 지정이 없으므로 secret)
        X frontend.Variable `gnark:"x"`
        // Y는 public witness (Verifier도 알고 있는 Public Input)
        Y frontend.Variable `gnark:",public"`
    }
    ```

2. **Define 메서드에서 제약 조건 작성**

    - `Define(api frontend.API) error` 메서드 내에서, 덧셈, 곱셈, 비교 등 논리적 제약 조건을 작성한다. `api.Add`, `api.Mul`과 같은 산술 연산을 통해 $$x^3 + x + 5 = y$$와 같은 방정식을 구성하거나, `gnark/std` 라이브러리의 가젯을 호출하여 Merkle Proof 검증 및 EdDSA, ECDSA 서명 검증과 같은 복잡한 로직을 Circuit 안에 포함시킨다.

    ```go
    func (circuit *CubicCircuit) Define(api frontend.API) error {
        // x³ + x + 5 == y 를 증명하는 Circuit
    
        // x² 계산
        x2 := api.Mul(circuit.X, circuit.X)
        // x³ 계산
        x3 := api.Mul(x2, circuit.X)
        // x³ + x 계산
        sum := api.Add(x3, circuit.X)
        // x³ + x + 5 계산
        result := api.Add(sum, 5)
    
        // 제약 조건: result == Y 를 단언(assert)
        api.AssertIsEqual(result, circuit.Y)
    
        return nil
    }
    ```

3. **컴파일 (Compilation)**

    정의된 Go 코드는 `frontend.Compile()` 을 통해 수학적 시스템인 R1CS(Rank-1 Constraint System) 또는 Plonkish 형식으로 변환된다. 이때 이더리움 Precompile 환경과의 호환성을 위해 주로 BN254 곡선을 사용하며, Circuit의 산술은 BN254의 스칼라체 $$\mathbb{F}_r$$(약 254비트 소수 $$r$$) 위에서 이루어진다. 곡선 방정식 $$y^2 = x^3 + 3$$은 기저체 $$\mathbb{F}_p$$ 위의 식으로, Circuit 산술이 이루어지는 스칼라체와는 다른 체다.
    
    설계된 Circuit은 수학적 처리가 가능한 형태로 변환되기 위해 Flattening(평탄화) 과정을 거친다. 이는 복잡한 수식($$x^3 + x + 5 = y$$)을 하나의 곱셈 연산만 포함된 간단한 수식들의 집합으로 분해하는 작업이다. 모든 연산은 "입력1 × 입력2 = 출력" 형태로 쪼개지며, R1CS에서는 최종적으로 평탄화된 각 식이 $$A \times B = C$$ 형식의 제약 조건으로 변환된다. 이때 $$A, B, C$$는 Circuit 내의 모든 변수 $$v_i$$(입력값, 중간값, 상수 $$v_0 = 1$$ 포함)의 Linear Combination(선형 결합)으로 구성된다. 
    
    $$
    \begin{aligned}
    A &= \alpha_0 v_0 + \alpha_1 v_1 + \cdots + \alpha_n v_n \\
    B &= \beta_0 v_0 + \beta_1 v_1 + \cdots + \beta_n v_n \\
    C &= \gamma_0 v_0 + \gamma_1 v_1 + \cdots + \gamma_n v_n
    \end{aligned}
    $$

    이때 $$A, B, C$$ 내부의 덧셈(Linear Combination)은 R1CS에서 별도의 제약을 추가하지 않는다. 따라서 최대한 많은 덧셈을 $$A \times B = C$$의 괄호 안으로 병합하여, 전체 곱셈 횟수를 최소화하는 것이 R1CS Circuit 설계의 핵심 최적화 요소가 된다. PLONK의 Plonkish 제약에서는 게이트의 입력 수가 제한되어 있어 덧셈에도 게이트가 필요하다.

    일반적인 프로그래밍 언어와 달리 산술 Circuit에서는 입력에 따라 실행 경로를 바꿀 수 없으므로, 조건문과 반복문을 수학적인 제약 조건으로 치환해야 한다. 조건문(if)은 `api.Select(condition, a, b)` 함수로 대체된다(조건이 첫 번째 인자). 이 함수는 `condition`이 1(참)이면 `a`를, 0(거짓)이면 `b`를 선택하며, 내부적으로 `b + condition × (a − b)`로 계산되어 곱셈 1개로 Circuit에 삽입된다. 현재 gnark는 R1CS 빌더와 PLONK용 SCS 빌더 모두 Select 안에서 `condition`이 0 또는 1이라는 Boolean 제약을 함께 추가한다. 반복문(for)은 컴파일 타임에 Unrolling(펼치기) 방식으로 처리된다. `Define()` 안의 Go for 문은 Circuit을 컴파일할 때 실행되어 반복마다 제약을 생성하므로, 반복 횟수가 컴파일 시점에 정해져 있으면 for 문을 그대로 쓰면 된다. 입력 개수가 정해진 경우에는 고정 크기 배열을 struct 필드로 정의한다. 이는 Circuit 컴파일 시 모든 제약 수가 정적으로 확정되어야 하는 Circuit 기반 SNARK의 특성 때문이다. 

    이 모든 것이 평탄화되어 Go 코드가 다수의 $$A \times B = C$$ 제약으로 변환된다. 결과적으로 Prover는 이 방정식 시스템을 유한체 위에서 정확히 만족하는 Witness를 알고 있음을 증명하고, Verifier는 로직을 다시 실행하는 대신 증명과 Verifying Key(검증 키)로 페어링 등식을 확인하여 계산의 정합성을 입증한다.

4. **Trusted Setup**

    컴파일이 완료되면 선택한 증명 시스템(Groth16 또는 PLONK)에 따라 설정을 수행한다. 

    - **Groth16 Trusted Setup**
        
        컴파일 완료 후 Groth16의 Trusted Setup은 `groth16.Setup(cs)` 함수를 호출하여 진행된다. 이 과정에서 Circuit별 Proving Key(pk)와 Verifying Key(vk)를 생성하며, 내부적으로 Toxic Waste(비밀 난수: $$\alpha, \beta, \gamma, \delta, \tau$$ 등)를 사용해 Structured Reference String을 만든다. 이 값을 아는 사람은 증명을 위조할 수 있으므로 반드시 파기해야 하며, 대규모 프로젝트에서는 MPC(Multi-party Computation) 세레머니를 통해 수행된다. 참여자 중 한 명만 정직하게 파기해도 안전하다.
    
    - **PLONK Trusted Setup**
        
        반면 PLONK의 Trusted Setup은 KZG Polynomial Commitment 기반의 Universal SRS를 전제로 하며, gnark에서는 `plonk.Setup(ccs, srs, srsLagrange)`처럼 SRS를 인자로 넘긴다. 각 Circuit마다 새 세레머니를 하는 대신 사전에 MPC로 생성된 공통 SRS를 재사용한다. 다만 Circuit마다 Proving Key와 Verifying Key를 만드는 전처리는 필요하므로, Circuit을 업데이트하면 키는 다시 생성해야 한다(새 세레머니는 필요 없다).
    

    ⇒ 주요 차이점은 Groth16이 Circuit별 설정으로 상수 크기 증명($$\mathbb{G}_1$$ 점 2개 + $$\mathbb{G}_2$$ 점 1개, BN254 기준 비압축 256바이트, 압축 128바이트)과 빠른 검증($$e(\alpha, \beta)$$를 미리 계산하면 페어링 3번)을 제공하는 반면, PLONK는 범용 설정으로 유연하지만 증명이 더 크고(gnark의 BN254 Solidity 직렬화 기준 768바이트: $$\mathbb{G}_1$$ 점 9개와 스칼라 6개이며, Commitment 커스텀 게이트를 쓰면 게이트당 96바이트가 추가된다) 검증에 KZG 확인(페어링 2번과 여러 번의 $$\mathbb{G}_1$$ 스칼라 곱)이 필요하다는 것이다.


5. **증명 생성 (Proving)**
    
    Prover는 Assignment(실제 데이터)를 Circuit에 주입하여 Witness를 생성하고, 이를 Proving Key와 함께 백엔드에 전달한다. gnark에서는 `frontend.NewWitness(&assignment, field)`로 Witness를 생성한 후 `groth16.Prove(cs, pk, witness)` 또는 `plonk.Prove(cs, pk, witness)`를 호출한다. 이 단계에서 gnark는 두 가지 핵심 연산을 수행한다.

    - **MSM (Multi-Scalar Multiplication)**: 타원 곡선 점들의 거대한 Linear Combination($$P = \sum_{i} r_i G_i$$)을 계산한다. 증명 생성에서 가장 무거운 연산 중 하나이며, gnark(gnark-crypto)는 Pippenger의 Bucket Method(버킷 방식)와 멀티코어 병렬화로 이를 계산한다.
    - **FFT (Fast Fourier Transform)**: 계수 표현과 점(Point) 표현 간 변환을 수행한다. 유한체 위에서는 NTT(Number Theoretic Transform)로 계산한다.

    gnark의 강점은 CPU 병렬화와 Memory 최적화를 통한 빠른 증명 생성이며, Groth16에 대해서는 GPU 가속(ICICLE)도 실험적으로 지원된다.

6. **검증 (Verification)**

    생성된 증명(Proof)은 Verifying Key(vk)와 Public Witness를 사용해 `groth16.Verify(proof, vk, publicWitness)` 또는 `plonk.Verify(proof, vk, publicWitness)`로 확인된다. Groth16의 경우 $$e(\alpha, \beta)$$를 미리 계산해 두면 페어링 3번으로 검증이 완료되며, Off-chain에서는 빠르게 처리된다. On-chain에서는 비용을 Gas로 측정한다.

    gnark는 `vk.ExportSolidity(f)` 함수로 Verifying Key를 Solidity 스마트 컨트랙트로 자동 변환해준다. 생성된 `Verifier.sol`은 이더리움의 `ecPairing` Precompile(주소 0x08)을 직접 호출하며, 페어링 4쌍 확인에 181,000 Gas(EIP-1108 기준)가 들고 Public Input당 약 6~7천 Gas가 더해져 일반적으로 20만~30만 Gas로 증명을 검증한다.


    ```go
    func main() {
        // 예시를 위해 오류 처리를 생략했다
    
        // --- 컴파일 (Compile) ---
        // Circuit을 R1CS (Rank-1 Constraint System)로 컴파일
        var circuit CubicCircuit
        ccs, _ := frontend.Compile(ecc.BN254.ScalarField(), r1cs.NewBuilder, &circuit)
    
        // --- 셋업 (Setup) ---
        // Groth16 증명 시스템의 proving key, verifying key 생성
        pk, vk, _ := groth16.Setup(ccs)
    
        // --- Witness 생성 ---
        // x=3 일 때 y = 3³ + 3 + 5 = 35
        assignment := CubicCircuit{
            X: 3,
            Y: 35,
        }
        witness, _ := frontend.NewWitness(&assignment, ecc.BN254.ScalarField())
        publicWitness, _ := witness.Public()
    
        // --- 증명 (Prove) ---
        proof, _ := groth16.Prove(ccs, pk, witness)
    
        // --- 검증 (Verify) ---
        err := groth16.Verify(proof, vk, publicWitness)
        if err != nil {
            panic("증명 검증 실패!")
        }
        // 검증 성공: Y=35를 만족하는 X를 Prover가 알고 있음을, X를 공개하지 않고 증명
    }
    ```

**장점**

- gnark의 강점은 최적화된 증명 생성 속도이다. 하부 라이브러리 gnark-crypto는 MSM과 FFT 같은 핵심 연산을 저수준에서 최적화하고 병렬 처리한다.
- 별도의 언어(예: Circom의 전용 문법)를 학습할 필요 없이 Go struct와 API로 Circuit을 설계하며, 기존 Go 테스트 프레임워크와 CI/CD를 그대로 활용할 수 있다.
- 이더리움 메인넷에서 ZKP를 검증하려면 Gas 비용이 중요하다. `gnark`는 이더리움 Precompile에 최적화된 BN254 곡선을 지원하며, 검증 로직을 Solidity 컨트랙트로 자동 변환(Export)해주는 기능을 기본 제공한다. Groth16 검증은 20만 Gas대 비용으로 가능하다.
- 하나의 Circuit 정의로 Groth16과 PLONK라는 두 가지 주요 증명 시스템을 모두 사용할 수 있다(빌더만 바꾼다). 상황에 따라 증명 크기가 작은 Groth16이나, Universal Setup이 가능한 PLONK를 선택할 수 있는 유연성을 제공한다.

**단점**
 
- `if`는 `api.Select()`로, 입력에 따라 길이가 바뀌는 반복은 상한과 Selector(선택자) 방식으로 변환해야 하며, 제약 수 최소화를 의식한 설계가 필수이다.
- EVM 전체 상태 전환(Account Balance, Storage, Merkle Root)도 Circuit으로 표현할 수 있지만(Linea 사례), Circuit을 직접 설계해야 하므로 개발과 감사 부담이 크다.
- 증명이 가장 작고 검증이 빠른 Groth16을 사용하려면 각 Circuit마다 고유한 Trusted Setup이 필요하다. 로직이 조금만 바뀌어도 키를 다시 생성하고 관리해야 하므로 운영상의 부담이 크다. (PLONK를 사용하면 공통 SRS를 재사용하므로 이 문제는 완화된다.)

### RISC Zero와 gnark 비교
 
⇒ 둘 다 "어떤 상태에서 어떤 트랜잭션 집합을 적용했더니, 새로운 상태가 나왔다"는 **State Transition의 정당성**을 증명하는 데 쓸 수 있다. 단, 그렇게 증명하도록 Guest 프로그램이나 Circuit을 설계해야 한다.
 
- **입력**: 이전 상태(Commitment, 비공개 가능), 트랜잭션, 서명, Merkle Proof
- **출력**: 새 상태(Commitment), 트랜잭션 실행 결과
- **증명**: 이 입력들을 가지고 정의된 검증 로직을 실행했는데, 모든 검증에 성공했고 해당 출력이 나왔다는 사실

**차이점: 검증 로직을 어디서 어떻게 표현하느냐?**
 
- **Risc Zero**: zkVM 방식 (소프트웨어 중심)
    - 일반적인 **Rust/C++ 코드**로 작성함
    - "이 프로그램을 실행했을 때 이런 결과가 나왔다"는 **'CPU의 Execution Trace'** 자체를 증명함
    - RISC‑V 프로그램(Guest 코드)로 검증 로직 작성 → zkVM이 그 실행 전체를 증명
- **gnark**: Circuit 방식 (수학적 제약 중심)
    - 산술 제약 조건(API)으로 작성함
    - "이 방정식 시스템을 만족하는 값(Witness)을 알고 있다"는 '수학적 관계'를 직접 증명함

| **항목** | **RISC Zero** | **gnark** |
| --- | --- | --- |
| **추상화 레벨** | zkVM (RISC-V RV32IM 실행 증명) | Circuit DSL (Go struct + API) |
| **증명 시스템** | STARK + Recursion + Groth16 Wrapping(On-chain용) | Groth16, PLONK (KZG) |
| **프로그래밍 언어** | RISC-V로 컴파일 가능한 언어 (Rust, C/C++) | Go로 Circuit 정의 및 증명 |
| **용도** | 임의 프로그램 실행 증명, zkVM 기반 롤업 | 특정 기능 Circuit (Rollup/Bridge/Light Client), zkEVM Prover(Linea) |
| **개발자 경험** | "Guest 작성 → `risc0-build`로 컴파일" | "제약 조건 설계 → `api.Mul()` 등" |
| **검증 로직 표현** | RISC-V Guest 코드 (반복문, 조건문 자유) | Circuit 제약 (`frontend.API`, 평탄화된 산술식) |
| **증명 대상** | "프로그램 전체 실행 Trace" | "제약 시스템을 만족하는 Witness" |
| **트랜잭션 검증** | 코드에서 라이브러리 사용 (가속 Circuit용 패치 크레이트 권장) | `gnark/std` 가젯 (ECDSA/EdDSA/Merkle/Keccak) |
| **On-chain 검증** | `verify(seal, imageId, journalDigest)` Solidity Verifier | `vk.ExportSolidity()`로 만든 Solidity Verifier |
| **변경 내성** | 코드 변경 → ImageID 갱신 (On-chain 허용 목록 포함) | Circuit 변경 → 키 재생성 (Groth16은 Circuit별 Trusted Setup도 다시) |
| **성능 튜닝** | RISC-V Instruction 수, Memory 접근, 가속 Circuit | 제약 수 최소화, 곡선/백엔드 선택 |


### ZoKrates
 
ZoKrates는 이더리움 환경을 겨냥한 zk-SNARK 툴체인으로, 자체 DSL을 통해 복잡한 산술 Circuit 설계를 추상화함으로써 개발자 접근성을 높인다. 개발자가 ZoKrates 언어(중괄호 블록을 쓰는 문법, 예: `def main(private field a) -> field { return a * a; }`)로 계산 로직을 기술하면, 툴체인은 이를 AST(추상 구문 트리)로 변환한 뒤 if, for Loop 및 중첩 수식을 기본 연산 단위로 분해하는 Flattening 과정을 수행한다. 분해된 연산들은 최종적으로 $$A \times B = C$$ 형식의 R1CS(Rank-1 Constraint System) 제약 조건으로 변환되며, 이 과정에서 코드상의 변수들은 행렬의 인덱스로 매핑된다.
 
컴파일 이후의 Setup(설정) 단계에서는 R1CS Circuit을 입력받아 Proving Key(증명 키)와 Verifying Key를 생성한다. 이때 Groth16(G16) 백엔드를 사용할 경우 'Toxic Waste'라 불리는 비밀 난수가 생성되며, 증명의 무결성을 위해 이는 생성 직후 반드시 파괴되어야 한다. ZoKrates는 G16 외에도 GM17, Marlin 증명 체계를 지원하며(백엔드는 bellman, ark), PLONK는 지원하지 않는다. 이를 통해 개발자는 필요에 따라 상수 크기의 증명을 제공하는 G16을 선택하거나, 공통 설정(`universal-setup`)을 재사용할 수 있는 범용 설정 방식인 Marlin을 선택할 수 있다. Marlin도 Circuit별 Setup 단계는 거친다.
 
실제 증명 생성 단계에서는 개발자가 제공한 입력값을 바탕으로 Circuit의 모든 중간 계산값을 산출하는 Witness 생성 과정이 선행된다. 이후 Proving Key와 Witness 데이터를 Prover 알고리즘에 입력하면, 대규모 연산인 MSM(Multi-scalar Multiplication)과 FFT(Fast Fourier Transform)를 거쳐 최종적으로 수백 바이트 크기의 증명이 생성된다. 특히 ZoKrates는 Verifying Key를 기반으로 이더리움에서 즉시 실행 가능한 Solidity 검증 컨트랙트를 자동으로 생성함으로써, 암호학적 세부 구현 없이도 On-chain 검증 워크플로를 구축할 수 있게 한다.
 
이러한 CLI 기반의 자동화는 개발 편의성을 크게 향상시키지만, 제약 수는 연산 종류에 따라 크게 달라진다. 해시나 비트 연산은 코드 한 줄이 많은 R1CS 제약으로 확장되고, 덧셈은 추가 제약이 거의 없다. 증명 생성 과정의 CPU와 Memory 부담은 Prover가 지며, Verifier는 작은 증명만 확인한다. 따라서 ZoKrates의 한계는 검증 노드의 실시간 처리 성능이 아니라 증명 생성 비용과 Circuit을 직접 설계하는 방식에 있다.


<br />
<br />


## Loop 관련
 
### EVM
 
- Loop = 비교 Opcode(LT, GT 등) + 조건부 점프(JUMPI) 패턴
    
```text
    예시 (EVM 어셈블리, Stack은 오른쪽이 최상단):
    0   PUSH1 0x00   [i]            i = 0
    2   JUMPDEST     [i]            loop:
    3   PUSH1 0x0a   [i, 10]
    5   DUP2         [i, 10, i]
    6   LT           [i, c]         c = (i < 10)
    7   ISZERO       [i, !c]
    8   PUSH1 0x11   [i, !c, 17]
    10  JUMPI        [i]            !c ≠ 0 (i ≥ 10) 이면 17로 점프 (Loop 탈출)
    11  PUSH1 0x01   [i, 1]         (Loop 본문 자리)
    13  ADD          [i+1]          i = i + 1
    14  PUSH1 0x02   [i+1, 2]
    16  JUMP         [i+1]          loop로 점프 (PC 뒤로)
    17  JUMPDEST     [i]            end:
```
    
- 특징:
    - 실제 CPU처럼 자유롭게 실행
    - PC가 2→3→...→16→2→... 로 반복 (Loop 한 번에 11단계 실행)
    - 검증: 모든 노드가 Re-execution
    - 제약: 없음 (Gas Limit만)
    - 비용: 실행량 (반복 $$N$$번이면 전체 $$11N + 9$$단계, $$N = 10$$이면 119단계)
- Loop가 1번이든 100만 번이든, 노드는 그만큼 Opcode를 실행해야 한다.

### zkEVM (trace 기반)

- 동일 Opcode 실행 → Execution Trace 생성
    
    ```text
    trace[t]:   PC=6,  opcode=LT,     stack [i, 10, i] → [i, c]
    trace[t+1]: PC=7,  opcode=ISZERO, stack [i, c] → [i, !c]
    trace[t+2]: PC=8,  opcode=PUSH1 0x11
    trace[t+3]: PC=10, opcode=JUMPI,  cond=!c → cond=0이면 PC_next=11, cond≠0이면 PC_next=17
    ```
        
    ```text
    Arithmetization 규칙 (모든 스텝, 개념):
    IF opcode=JUMPI ∧ cond≠0 THEN PC_next − dest = 0  (그리고 code[dest] = JUMPDEST)
    IF opcode=JUMPI ∧ cond=0 THEN PC_next − (PC + 1) = 0
    ```
    
- 특징:
    - 가변 Loop 가능 (Trace 길이만 늘어남)
    - ZK: "각 스텝의 PC/Stack/Memory 규칙 준수"를 증명
    - **Trace 길이** $$N \propto$$ **Loop 횟수 × Loop당 Opcode 수**
        - **⇒ 따라서 Prover가 Trace → ZK 증명 생성하는 부분에서 오래 걸림**
        - 예시: Loop 100만 번 = Trace 약 1,100만 행 (Loop당 11단계) → 증명 시간 증가

- **결론 조건**
    - 이 규칙과 Stack, Gas, Memory 등 나머지 제약이 모든 Execution Trace에서 성립하고, 시작과 종료 상태가 Public Input과 맞으면 "이 Loop는 정당하게 실행되었다"고 결론내림
    - ZK는 Loop 횟수를 신경쓰지 않고, 각 스텝의 규칙 준수만 확인하므로 가변 Loop를 자연스럽게 처리 가능
- 예시
    - **동작 방식** (위 EVM 예시, N=10)
        ```text
        trace[0]:   PC=0,  PUSH1 0x00                 stack=[0]
        trace[1]:   PC=2,  JUMPDEST                   (i=0 반복 시작)
        trace[4]:   PC=6,  LT      0<10 → c=1
        trace[5]:   PC=7,  ISZERO  → !c=0
        trace[7]:   PC=10, JUMPI   cond=0 → PC_next=11 (계속)
        trace[9]:   PC=13, ADD     i=1
        trace[11]:  PC=16, JUMP    → PC_next=2 (뒤로 점프)
        trace[12]:  PC=2,  JUMPDEST                   (i=1 반복 시작)
        ...
        trace[100]: PC=2,  JUMPDEST                   (i=9 반복 시작)
        trace[106]: PC=10, JUMPI   cond=0 → PC_next=11
        ...
        trace[111]: PC=2,  JUMPDEST                   (i=10, 마지막 확인)
        trace[114]: PC=6,  LT      10<10 → c=0
        trace[117]: PC=10, JUMPI   cond=1 → PC_next=17 (탈출)
        trace[118]: PC=17, JUMPDEST                   stack=[10]
        ```
    - **ZK가 확인하는 것**
        - 모든 $$t$$에 대해:
            1. $$\text{opcode}_t$$에 따른 Stack/Memory 변화 규칙 준수
            2. 다음 PC 규칙: $$PC_{t+1} = f(PC_t, \text{opcode}_t, \text{cond}_t)$$
    - ZK는 Loop라는 개념보다는 PC 점프 패턴이 Opcode 규칙을 지켰는지만 확인하면 됨
        
        ⇒ 즉, Loop 횟수 $$N \in \{1,\ 10,\ 10^6\}$$에 상관없이 
        
        1. 동일한 제약 공식 사용
        2. Trace 길이만 $$N$$에 비례해서 증가
        3. 각 스텝마다 같은 규칙만 확인

- Loop가 1번이든 100만 번이든 상관 없이, "이 규칙을 모든 스텝에서 지켰다"만 보여주면 되기 때문에 가변 Loop가 자연스럽게 처리됨. 단, 100만 번 돌면 Trace가 길어지고, Prover 쪽 증명 시간이 길어지는 게 차이
- 단, Trace 방식에도 증명 하나가 담을 수 있는 길이에 상한이 있어, zkVM은 긴 실행을 Segment로 나누고 Recursion으로 합침

### Circuit 기반 ZKP (gnark, ZoKrates, Circom)
 
- **Circuit = 고정 크기 산술식**
    - 프로그램 → 컴파일 → R1CS/PLONK 제약 집합 (고정 크기)
    - 프로그램 전체를 한 번에 거대한 다항식/제약 세트로 만들기 때문에, Loop/리스트 길이가 바뀌면 Circuit 모양이 바뀜
    - 즉, Secret Input에 따라 Loop 길이가 달라지면 "Circuit 크기"를 미리 정할 수 없음
- 고정 상한 + Padding 방식으로 우회 가능
    - 즉, Max Bound(최대 횟수)를 정해두고 남는 단계는 Selector로 비활성화하는 Padding 기법 사용
    - 빈 자리에 0을 넣기만 하면 안 되고, Selector가 0 또는 1인지, 활성 단계가 앞쪽에 연속하는지, 활성 단계 수가 N인지를 제약해야 함
    - 비용: 실제 $$N = 10$$, $$\text{MAX} = 1000$$ → Circuit은 항상 1000단계 분량이므로 단계 수 기준 100배 (990번의 무의미한 단계가 발생하며, 이는 실제 단계 수의 99배). 증명 시간이 정확히 100배가 되는 것은 아님

우회 방식 예시
 
- 컴파일 타임 Unroll 또는 Padding 필수
    
```text
    1. 고정 Loop (Circom):
    for (var i = 0; i < 10; i++) { ... }  // 반복 횟수는 컴파일 시 상수
    
    2. 가변 Loop (padding):
    for (i = 0; i < MAX; i++) {          // MAX = 1000 (컴파일 시 상수)
      s_i = (i < secret_N)               // Selector: 비교 Circuit으로 계산하고 Boolean 제약
      state = s_i × 실제작업(state) + (1 − s_i) × state
    }
```
    
- Recursion이나 Folding 기법으로 단계별 증명을 이어 붙이면 단일 고정 Circuit의 제약을 피할 수 있음

<br />
<br />

## 이더리움 L1의 zkEVM 도입 현황

이더리움 L1에서 zkEVM으로 블록 실행을 검증하는 방안이 연구되고 있다. 이더리움 재단(EF)은 「zkEVM Security Overview」(2026-01-14)에서 특정 구현체의 버그나 암호학적 취약점에 대비해, Consensus Client(합의 클라이언트)가 서로 다른 여러 zkEVM 증명(Execution Client 구현과 zkVM 양쪽의 다양성 포함)을 확인한 뒤에 블록을 받아들이는 Multiproofs(다중 증명) 전략을 제시했다. 현재 이더리움의 Execution Client(Geth, Besu, Nethermind 등)가 다양하게 존재하듯, 증명 쪽에서도 여러 zkVM과 Execution Client 조합을 활용해 다양성을 확보하려는 접근이다. Multiproofs 구조는 특정 구현체에서 치명적인 버그가 발생하더라도 다른 시스템과의 교차 검증을 통해 잘못된 블록이 받아들여지는 것을 방지한다. 실제로 Scroll은 OpenVM 전환 직후 OpenVM 1.0.0 Circuit의 Soundness(건전성) 버그로 긴급 업그레이드를 했고(2025-04-25), Taiko는 SGX Enclave 서명 키가 공개 GitHub 저장소에 노출되어 위조 증명으로 브리지에서 약 170만 달러를 탈취당했다(2026-06-22).

다만 작성 시점(2026-11-11) 이더리움 메인넷은 Multiproofs를 채택하지 않았고, 블록은 여전히 Re-execution으로 검증된다. 첫 단계로 제안된 EIP-8025(Optional Execution Proofs)는 Draft 상태이며, 참여를 원하는 Validator만 증명을 만들고 검증하는 선택적 방식이다. 합의 유효성 규칙을 바꾸지 않고, 증명을 검증하는 노드도 Re-execution을 계속하며 증명은 추가 신호로만 쓰인다. EIP-8025는 Glamsterdam 다음 하드포크인 Hegotá의 포함 대상으로 제안되었지만, 2026년 10월 15일 ACDC #189에서 Hegotá에 포함하지 않기로(DFI, Declined for Inclusion) 결정되었다.

이러한 방식의 비용은 역할별로 나눠 봐야 한다. 증명 생성은 Prover가 담당하며 GPU 클러스터 수준의 장비가 필요하다. zkVM 방식은 모든 실행 과정을 Polynomial Constraints로 증명하므로 네이티브 실행보다 훨씬 많은 연산이 든다. 반면 증명 검증은 블록의 계산량과 거의 무관하게 작아서, 여러 증명을 검증해도 검증 노드의 부담은 크지 않다. 실제 위험은 증명 생성이 소수의 고성능 운영자에게 집중되는 것과, 증명이 늦게 도착할 때 Attestation이나 Finality가 지연될 수 있다는 점이다. 결과적으로 STARK 기반 zkVM 접근법은 검증 비용을 낮추는 대신 증명을 만드는 단계에 큰 비용을 전가하며, 이 부담은 Validator 노드가 아니라 Prover에게 있다.
