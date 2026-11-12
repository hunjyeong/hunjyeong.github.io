---
title: "EVM, zkEVM, zkVM: Execution and Verification"
date: 2026-03-27 00:00:00 +0900
categories: [Blockchain]
tags: [Blockchain, Ethereum, EVM, zkEVM, zkVM]
math: true
media_subpath: /assets/img/posts/zkevm/
---

>
> 1. **EVM, zkEVM, zkVM: Execution and Verification**
> 2. [RISC Zero zkVM: Architecture and Proving Process]({% post_url Blockchain/2026-03-27-RISC-Zero-zkVM %})
> 3. [Circuit-Based SNARK Tools: gnark and ZoKrates]({% post_url Blockchain/2026-03-27-gnark-ZoKrates %})
>

참고 문서

[https://decipher-zkvm.gitbook.io/decipher-zkvm-research](https://decipher-zkvm.gitbook.io/decipher-zkvm-research)

## Ethereum Virtual Machine (EVM)

### What is EVM?

EVM(Ethereum Virtual Machine)은 Ethereum 노드의 Execution Client(실행 클라이언트)가 공통으로 따르는 실행 환경이다. 스마트 컨트랙트의 실행과 트랜잭션 처리를 담당하는 Execution Layer(실행 계층)의 핵심을 이룬다. Execution Client(Geth, Nethermind, Besu, Erigon, Reth 등)는 각자 EVM을 구현하지만 동일한 공식 사양을 따른다. 공식 사양은 하드포크별 EIP(Ethereum Improvement Proposal)와 실행 명세 [`execution-specs`](https://github.com/ethereum/execution-specs)이며, Ethereum Yellow Paper는 EVM의 형식 명세다. 현재 Client 구현의 기준은 `execution-specs`다.

EVM은 **동일한 입력이 주어질 경우 항상 동일한 계산 결과를 생성**하도록 설계되어 있다. 이러한 **Deterministic Execution(결정론적 실행)**은 모든 노드가 같은 블록을 독립적으로 실행해 같은 결과를 얻는 근거가 되며, Ethereum의 검증 방식이 성립하는 기반이다.

Ethereum은 The Merge(2022년 9월) 이후 합의와 실행을 분리한 두 개의 Layer(계층) 구조를 가진다. 각 노드는 Consensus Client와 Execution Client를 함께 실행하며, 두 Client는 Engine API(Application Programming Interface)로 통신한다.

- **Consensus Layer(CL, 합의 계층):** Block Proposer(블록 제안자) 선정, LMD-GHOST(Latest Message Driven Greedy Heaviest Observed Sub-Tree) 기반 Fork Choice(체인 선택), Casper FFG(Friendly Finality Gadget) 기반 Finality(최종성) 결정

- **Execution Layer(EL, 실행 계층):** 트랜잭션 실행, State Transition(상태 전이) 계산, 실행 결과 생성 및 검증

Consensus Layer의 참여자는 Validator(검증자)다. Validator는 Ether를 Stake(예치)하고 블록 제안과 투표를 수행한다. EVM은 Execution Layer에서 트랜잭션의 코드 실행을 담당한다. Execution Layer는 블록을 직접 실행하고, 그 결과가 블록 헤더에 기록된 실행 결과 값과 일치하는지 확인해 블록의 실행 유효성을 판단한다.

### World State and State Transition Function

**EVM은 State(상태)를 직접 저장하지 않는다.** EVM은 트랜잭션이 컨트랙트를 호출하거나 생성할 때 바이트코드를 실행하여, 코드 실행에 따른 상태 변화를 계산한다. EVM이 읽고 쓰는 대상은 Ethereum의 **World State(전역 상태)**이다.

World State는 주소에서 계정 상태로의 매핑이며, 각 계정은 Externally Owned Account(EOA, 외부 소유 계정) 또는 Contract Account(CA, 컨트랙트 계정)로 구분된다. 각 계정은 **Nonce, Balance, Storage Root**(`storageRoot`), **Code Hash**(`codeHash`)를 가지며, 스마트 컨트랙트 코드와 컨트랙트 Storage는 각각 이 Hash와 Root로 연결된다. EIP-7702(Pectra) 이후에는 EOA도 Delegation Indicator(위임 표시자, `0xef0100 || address`)를 코드로 가질 수 있다.

World State는 Modified Merkle Patricia Trie(MPT) 구조로 표현되며, 이 Trie의 Root Hash는 블록 헤더의 **State Root**로 기록된다. World State의 값이 하나라도 바뀌면 Root Hash도 바뀌므로, **State Root는 해당 시점 World State 전체를 대표한다.** 실제 디스크 저장 방식은 Client마다 다르다.

State Root와 Merkle Proof(특정 값에서 Root Hash까지의 경로에 있는 Trie Node들)를 이용하면, 전체 상태 데이터를 보유하지 않은 노드도 특정 계정이나 Storage 값이 그 상태에 포함되어 있는지 효율적으로 검증할 수 있다. 다만 State Root는 결과 상태가 무엇인지만 나타낸다. 그 상태가 올바른 실행으로 만들어졌는지는 같은 규칙으로 블록을 다시 실행해야 확인할 수 있다. 이 규칙이 State Transition Function이다.

#### State Transition Function

Ethereum은 단순한 거래 원장이 아니라 트랜잭션 기반의 **Distributed State Machine(분산 상태 머신)**으로 동작한다. 네트워크는 Genesis State(초기 상태)에서 시작해 각 블록의 트랜잭션을 순서대로 실행하며 상태를 갱신한다. 이 갱신 규칙을 **State Transition Function(상태 전이 함수)**이라고 한다. Yellow Paper는 State Transition Function을 트랜잭션 단위와 블록 단위로 나누어 정의한다. 블록 단위 함수가 트랜잭션 단위 함수를 반복 호출하는 계층 구조다. 아래에서 $\sigma$는 이전 상태, $\sigma^\prime$는 결과 상태, $T$는 트랜잭션, $B$는 블록이다.

- **트랜잭션 단위**: $\sigma^\prime = \Upsilon(\sigma, T)$

$\Upsilon$은 트랜잭션 하나를 상태에 적용한다. $\Upsilon$이 곧 EVM 실행은 아니다. EVM의 코드 실행은 Yellow Paper에서 $\Xi$로 따로 정의되며, $\Upsilon$은 $\Xi$를 포함한 트랜잭션 처리 전체를 담당한다. 처리 순서는 다음과 같다.

    1. 유효성 검사: 서명, Nonce, 잔액, Gas Limit을 확인한다.
    2. 선차감: 발신자 잔액에서 Upfront Gas 비용(`gasLimit × effectiveGasPrice`, 실행 전에 미리 차감하는 수수료)을 차감하고, Nonce를 1 증가시킨다.
    3. 실행: Value를 이체하고, 수신자에 코드가 있으면 EVM이 그 코드를 실행한다($\Xi$).
    4. 정산: 사용하지 않은 Gas를 환급한다. Priority Fee(블록 생성자에게 주는 추가 수수료)는 Fee Recipient(수수료 수령 주소)에 지급하고, Base Fee(EIP-1559에 따라 블록마다 정해지는 기본 수수료)는 소각한다.

2단계의 선차감과 Nonce 증가는 실행이 실패해도 되돌려지지 않는다. 따라서 블록에 포함된 트랜잭션은 코드 실행이 실패하더라도 Nonce와 잔액을 바꾼다. 즉 **블록에 포함된 모든 트랜잭션은 상태 변화를 일으킨다.**

- **블록 단위**: $\sigma^\prime = \Pi(\sigma, B)$

$\Pi$는 블록 전체를 상태에 적용한다. 블록 $B$는 블록 헤더 값(블록 번호, Timestamp, Base Fee, Fee Recipient 등), 순서화된 트랜잭션 목록, Withdrawal(인출) 목록으로 이루어진다. $\Pi$는 트랜잭션 순서대로 $\Upsilon$을 반복 적용하며, 앞 트랜잭션의 결과 상태가 다음 트랜잭션의 입력 상태가 된다.

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
    - **Consensus Layer에서 Execution Layer로:** Shanghai 이후 Withdrawal을 처리한다. Validator의 잔액(보상 또는 Stake한 Ether)을 Execution Layer의 대상 주소로 보내는 처리로, 대상 주소의 잔액을 늘린다. EVM을 실행하지 않으며 Gas도 쓰지 않는다.
    - **Execution Layer에서 Consensus Layer로:** Prague 이후 Requests(요청)를 모아 Consensus Layer에 전달한다. 예치 요청(EIP-6110)은 Deposit Contract의 Log에서 모으고, 출금 요청(EIP-7002)과 통합 요청(EIP-7251)은 System Call로 모은다.

The Merge 이후 Execution Layer에는 블록 보상이 없다. Validator 보상은 Consensus Layer에서 처리된다. Priority Fee 지급과 Base Fee 소각은 블록 단위가 아니라 각 트랜잭션의 $\Upsilon$ 안에서 일어난다.

블록 실행 전 상태 $\sigma$를 Pre-Sta\te, 실행 후 상태 $\sigma^\prime$를 Post-State라고 한다. Pre-State의 Root Hash는 부모 블록의 State Root이고, Post-State의 Root Hash는 해당 블록의 State Root다. 그 사이의 $\sigma_1, \ldots, \sigma_n$은 계산 중에만 존재하며 블록에 기록되지 않는다. Byzantium(EIP-658) 이후 Receipt(영수증)에도 트랜잭션별 중간 State Root 대신 성공 여부(status)만 들어간다. 따라서 상태 전이의 검증은 블록 단위로 이루어진다.

**State Transition Function은 결정론적이다.** 같은 Pre-State, 같은 블록, 같은 Hard Fork 규칙이 주어지면 어떤 노드에서도 같은 Post-State, 즉 같은 State Root가 계산된다.

```text
Identical Pre-State
+ Identical Block (Transaction Order, Withdrawals, Header Fields)
+ Identical State Transition Rules (Same Fork)
= Identical stateRoot
```

결국 **블록체인은 각 블록의 트랜잭션과, 그 실행 결과 상태를 대표하는 State Root를 연속적으로 기록한 구조**다. 상태 전체는 블록에 기록되지 않는다.


### EVM Execution Model

State Transition 중 EVM이 담당하는 부분은 $\Upsilon$ 안의 코드 실행($\Xi$)이다. 스마트 컨트랙트는 Solidity나 Vyper와 같은 고급 언어로 작성되며 컴파일 과정을 통해 EVM 바이트코드로 변환된다. EVM 사양은 Program Counter가 가리키는 Opcode를 하나씩 실행하는 의미론으로 정의되며, Client는 이를 Interpreter(인터프리터) 등으로 구현한다.

EVM의 실행 모델은 **Machine State(Program Counter, Stack, Memory, 가용 Gas)와 실행 대상 코드**로 구성되며, 실행 중 계정의 Storage에 접근한다. Cancun 이후에는 트랜잭션 종료 시 초기화되는 Transient Storage(EIP-1153)도 사용할 수 있다.


![evm](evm.png)
_출처: [https://ethereum.org/developers/docs/evm/](https://ethereum.org/developers/docs/evm/)_

- **Program Counter(PC):** 현재 실행 중인 바이트코드 위치(Offset)를 가리키는 포인터로, Opcode가 실행되면 다음 명령 위치로 이동한다. `PUSH1`~`PUSH32`는 뒤따르는 Immediate(즉시값) Byte 수만큼 더 이동한다(`PUSH0`은 즉시값이 없다). `JUMP`는 Stack에서 꺼낸 목적지로 이동하고, `JUMPI`는 조건 값이 0이 아닐 때만 이동한다. 목적지는 반드시 `JUMPDEST`여야 하며, 그렇지 않으면 예외가 발생해 실행이 중단된다.

- **Stack:** 256bit(32byte) 워드를 저장하는 LIFO(Last In, First Out, 후입선출) 구조로, 최대 1024개의 요소를 저장할 수 있다. 각 Opcode는 Stack에서 정해진 개수의 항목을 꺼내고 정해진 개수의 결과를 넣는다(0개일 수 있음). `STOP`, `JUMPDEST`처럼 Stack을 사용하지 않는 Opcode도 있다. 연산은 주로 최상위 값을 대상으로 하며, DUP 계열은 위에서 16번째 항목까지, SWAP 계열은 위에서 17번째 항목까지 접근할 수 있다.

- **Memory:** Byte 단위로 주소가 지정되는 휘발성 선형 공간으로, 연산 중간값, ABI(Application Binary Interface) 인코딩 및 디코딩, 반환값 등 임시 데이터에 사용된다. Call Frame(호출 프레임, 메시지 호출 또는 컨트랙트 생성 하나) 단위로 빈 상태로 새로 할당되며 해당 호출이 끝나면 폐기된다. 32byte 워드 단위로 확장되며, 확장 비용은 크기에 따라 이차적으로 증가한다.

- **Storage:** 계정별 영구 저장소로, 256bit 키를 256bit 값에 대응시키는 매핑이다. 상태 변수, 매핑, 배열 등이 저장되며 `SLOAD`와 `SSTORE` Opcode로 접근한다. 트랜잭션이 끝난 뒤에도 World State에 남으므로 접근 비용이 높고, 특히 값 변경에는 높은 Gas 비용이 부과된다.

- **Code:** 실행 대상 EVM 바이트코드로, 호출된 계정의 `codeHash`에 대응하는 코드 또는 컨트랙트 생성 시의 Init Code(초기화 코드)이다. 실행 중 Read-Only(읽기 전용)로 취급되며, Program Counter는 이 코드를 따라 Opcode를 읽고 실행한다. `DELEGATECALL`이나 EIP-7702 위임에서는 다른 계정의 코드를 현재 계정의 Storage와 Balance 맥락에서 실행한다.

#### Gas

Machine State의 가용 **Gas**는 계산 자원의 남용을 방지하기 위한 EVM의 비용 모델을 이룬다. Gas는 연산, Storage 접근, 데이터 크기 같은 자원 사용량을 근사한 비용 단위다. 모든 Opcode에는 사전에 정의된 Gas 비용이 존재하며, Memory 확장이나 Storage 접근 이력(EIP-2929의 Cold/Warm 구분 등)에 따라 추가 비용이 붙는 Opcode도 있다.

실행 중 Gas가 소진되면 **Out-of-Gas(가스 부족) 예외**가 발생하여 해당 Call Frame의 실행이 즉시 중단된다. 이 경우 **그 Call Frame에서 발생한 상태 변경은 모두 되돌려지며(Revert), 그 프레임에 할당된 Gas는 모두 소모된다.** 반면 Byzantium(EIP-140) 이후의 `REVERT` Opcode로 실행을 명시적으로 되돌리면, 상태 변경은 되돌려지지만 남은 Gas는 반환된다.

하위 호출이 실패하면 상위 호출은 실패 결과를 받고 실행을 계속할 수 있다. Tangerine Whistle(EIP-150) 이후 하위 호출에는 남은 Gas의 최대 63/64만 전달되므로, 하위 호출이 Gas를 모두 소진해도 상위 호출에는 Gas가 남는다. 최상위 호출에서 예외가 발생하면 트랜잭션의 실행 결과가 되돌려지지만, **트랜잭션은 블록에 포함되고 수수료 지불과 Nonce 증가는 남는다.** 이러한 메커니즘은 무한 Loop 실행과 DoS(Denial of Service, 서비스 거부 공격)를 방지한다.

### What is Opcode?

![opcode](opcode.png)
_출처: [https://ethereum.org/developers/docs/evm/#opcodes](https://ethereum.org/developers/docs/evm/#opcodes)_

Opcode(Operation Code)는 **"이 명령이 어떤 연산을 수행해야 하는지"를 지정하는 숫자 코드**이다. 하나의 Instruction(명령어)은 보통 Opcode + Operand(연산 대상/주소)로 이루어지며, 이 중 Opcode가 연산 종류(ADD, JUMP 등)를, Operand가 그 연산에 쓰일 데이터나 주소를 담는다. EVM의 전체 Opcode 목록과 Gas 비용은 [evm.codes](https://www.evm.codes/)에서 확인할 수 있다.

CPU(Central Processing Unit, 중앙 처리 장치)에서 `ADD`, `SUB`, `LOAD`, `STORE`, `JMP` 등은 명령어의 Mnemonic(니모닉)이며, 실제 기계어 명령어는 Opcode 필드와 Operand 필드로 구성된 특정 Bit 패턴으로 인코딩된다. 제어 장치는 Memory에서 명령어를 읽어(Fetch) Opcode Bit를 해석(Decode)하고, 그에 맞는 제어 신호 또는 마이크로 연산 시퀀스를 생성해 실행한다.

EVM에서는 스마트 컨트랙트가 바이트코드로 표현되고, 이 바이트코드는 **1byte Opcode와 일부 Opcode 뒤에 붙는 Immediate**로 구성된다. 예를 들어 0x01은 `ADD`, 0x55는 `SSTORE`처럼 각 1byte 값이 Stack 조작, 산술 연산, Storage 접근, 컨트랙트 호출 등 특정 동작을 의미한다. **EVM은 Stack 기반 머신**이므로 대부분의 Opcode는 Operand를 명령어 안에 갖지 않고 Stack에서 꺼내 사용한다. Immediate를 갖는 대표적인 예외는 `PUSH1`(0x60)부터 `PUSH32`(0x7F)까지로, Opcode 뒤에 1~32byte의 Immediate가 붙는다. 따라서 바이트코드의 모든 Byte가 Opcode인 것은 아니며, Immediate 안에 포함된 0x5B는 `JUMPDEST`로 인정되지 않는다.

계산 관련 Opcode(`ADD`, `MUL`, `AND`, `OR` 등)는 산술 연산과 비트 연산을 수행하고, 제어 흐름 Opcode(`JUMP`, `JUMPI`)는 무조건 분기와 조건 분기를 수행하며 이를 조합해 Loop를 구현한다. Storage와 Memory Opcode(`SLOAD`/`SSTORE`, `MLOAD`/`MSTORE`)는 각각 Storage와 Memory에 대한 읽기와 쓰기를 담당한다. 환경 및 블록 정보 Opcode(`BALANCE`, `BLOCKHASH` 등)는 계정 잔액, 블록 Hash 같은 실행 환경 정보를 조회하고, 호출 및 생성 Opcode(`CALL`, `CREATE` 등)는 외부 호출과 컨트랙트 생성을 수행한다.

해시 계산은 `KECCAK256` Opcode가 담당한다. 반면 ECDSA(Elliptic Curve Digital Signature Algorithm, 타원곡선 디지털 서명 알고리즘) 서명자 주소 복구(`ecrecover`)와 BN254 곡선 연산 등은 Opcode가 아니라 특정 주소의 Precompile로 제공되며, `CALL` 계열 Opcode로 호출한다.

### Block Execution Results

블록의 모든 처리가 끝나면 Execution Layer는 결과를 블록 헤더에 기록한다. 블록 헤더에는 **실행해야 얻을 수 있는 결과값**과 **블록 입력을 요약한 값**이 함께 들어 있으므로 둘을 구분해야 한다.

| 구분 | 헤더 값 | 계산 방법 |
|---|---|---|
| 실행 결과 | State Root | Post-State의 MPT Root Hash |
| 실행 결과 | Receipts Root | 각 트랜잭션 Receipt로 만든 Trie의 Root Hash |
| 실행 결과 | Logs Bloom | 모든 Log를 요약한 2048bit 값. 특정 Log의 포함 가능성을 빠르게 확인하는 데 쓴다 |
| 실행 결과 | Gas Used | 블록의 모든 트랜잭션이 사용한 Gas 합계 |
| 실행 결과 | Requests Hash (Prague 이후) | 블록에서 모은 Requests의 Hash(EIP-7685) |
| 입력 요약 | Transactions Root | 블록의 트랜잭션 목록으로 만든 Trie의 Root Hash |
| 입력 요약 | Withdrawals Root (Shanghai 이후) | 블록의 Withdrawal 목록으로 만든 Trie의 Root Hash |

Receipt는 트랜잭션 하나의 실행 결과 기록으로, 성공 여부, 누적 Gas 사용량, Log, Logs Bloom을 담는다. Log는 컨트랙트가 `LOG0`~`LOG4` Opcode로 남기는 이벤트 기록이다.

**입력 요약 값은 실행하지 않아도 계산할 수 있고, 실행 결과 값은 블록을 실행해야만 얻을 수 있다.** 따라서 입력 요약 값으로는 블록 내용이 변조되지 않았는지를, 실행 결과 값으로는 실행이 올바른지를 확인한다.

### End-to-End Block Verification Process

1. **Transaction Reception and Mempool Pre-validation**
    
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

    **Intrinsic Gas**는 코드 실행 전에 트랜잭션 자체에 부과되는 기본 Gas다. 기본 21,000 Gas에 Calldata, Access List, 컨트랙트 생성 비용이 더해지며, Pectra 이후 Type-4 트랜잭션은 Authorization List 비용도 더해진다. Intrinsic Gas는 `gasLimit` 안에 포함되므로 잔액 조건에 따로 더하지 않는다.

    **Mempool 검증은 블록 유효성의 일부가 아니다.** 블록에서 실행할 때는 위 항목을 모두 다시 확인하며, 다음 조건이 추가된다. 아래에서 이 조건을 **블록 포함 규칙**이라고 부른다.

    - Nonce가 계정의 현재 Nonce와 **정확히** 일치해야 한다.
    - `maxFeePerGas ≥` 해당 블록의 `baseFee`여야 한다.
    - `gasLimit`이 블록의 남은 Gas 이하여야 한다.
    - 발신자 계정에 Code가 없어야 한다(EIP-3607). Pectra 이후 Delegation Indicator를 가진 EOA는 예외다.

2. **Block Execution by the Block Producer**
    
    블록은 각 Slot(12초 단위의 시간 구간)마다 Block Proposer로 선정된 Validator가 제안한다. 블록의 실행 내용(Execution Payload)은 Proposer의 Execution Client가 직접 구성하거나, Protocol 외부 시스템인 MEV-Boost를 통해 Builder(블록 빌더)가 구성한다. MEV는 Maximal Extractable Value(최대 추출 가능 가치)의 약자다. Glamsterdam의 ePBS(enshrined Proposer-Builder Separation, EIP-7732)가 적용되면 Builder의 역할이 Protocol 안으로 들어온다. 2026년 11월 기준 Glamsterdam은 Mainnet 활성화 전이며, 활성화 일정은 확정되지 않았다.
    
    블록 생성자는 Pre-State에서 $\Pi$를 다음 순서로 실행한다.

    1. **블록 시작 처리:** EIP-4788(Cancun 이후)과 EIP-2935(Prague 이후)의 System Call을 수행한다.

    2. **트랜잭션별 실행($\Upsilon$):**

        1. 블록 포함 규칙을 확인한다. 실패하면 그 트랜잭션은 블록에 포함될 수 없다.
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

    3. **블록 종료 처리:** Withdrawal(Shanghai 이후)을 처리하고, Requests(Prague 이후)를 모은다.

    4. **헤더 기록:** "Block Execution Results"의 실행 결과 값을 계산해 블록 헤더에 기록한다.

3. **Re-execution by Verifying Nodes**

    검증 노드의 Consensus Client는 블록을 받으면 Engine API의 `engine_newPayload`로 Execution Payload를 Execution Client에 전달한다. **Execution Client는 블록 생성자가 제공한 결과를 신뢰하지 않는다.** 다음 순서로 직접 확인한다.

    1. **헤더 사전 검사:** 실행 전에 헤더 값을 확인한다.
        - 부모 Hash와 블록 번호가 올바른지, Timestamp가 부모 블록보다 큰지 확인한다.
        - Gas Limit 변화가 부모 대비 1/1024 미만인지 확인한다.
        - Base Fee가 부모 블록으로부터 계산한 값과 같은지 확인한다.
        - Fusaka 이후에는 EIP-7934의 블록 크기 제한도 확인한다.

    2. **입력 요약 확인:** Execution Payload의 트랜잭션 목록과 Withdrawal 목록으로 Transactions Root와 Withdrawals Root를 계산해 헤더를 재구성하고, 그 Hash가 블록 Hash와 같은지 확인한다.

    3. **Re-execution(재실행):** 부모 블록의 State Root가 가리키는 Pre-State에서 "Block Execution by the Block Producer"와 같은 절차로 블록을 다시 실행한다.

    4. **실행 결과 비교:** 계산한 실행 결과 값이 블록 헤더 값과 모두 일치하는지 확인한다.

    $$\mathrm{Root}(\Pi(\sigma, B)) \stackrel{?}{=} \texttt{stateRoot}$$

    State Root 외에 Receipts Root, Logs Bloom, Gas Used, Requests Hash도 일치해야 한다. **하나라도 다르면 블록은 무효다.** Execution Client는 결과(`VALID`, `INVALID`, 또는 아직 판단할 수 없을 때 `SYNCING`)를 Consensus Client에 돌려준다.

    Re-execution은 동기화를 마친 Full Node가 수행한다. Light Client는 블록을 실행하지 않고, 블록 헤더와 Merkle Proof로 필요한 값만 확인한다.

4. **Attestation and Finality**

    각 Validator는 Epoch(32 Slot, 약 6.4분)마다 배정된 Slot 하나에서 Attestation(투표)을 보낸다. Attestation에는 두 가지 투표가 함께 들어 있다.

    - **Head 투표:** 자신이 보는 체인의 최신 블록에 대한 투표로, LMD-GHOST가 사용한다.
    - **Checkpoint 투표:** Source Checkpoint와 Target Checkpoint에 대한 투표로, Casper FFG가 사용한다.
    
    Validator는 Execution Client가 유효하다고 판단한 블록이 포함된 체인에만 투표한다.

    Finality는 블록 하나가 아니라 **Checkpoint 단위**로 결정된다. Checkpoint는 각 Epoch 첫 Slot의 블록이며, 그 Slot이 비어 있으면 그 이전의 가장 최근 블록이다.

    1. 전체 Stake의 2/3 이상이 투표한 Checkpoint는 **Justified(정당화)** 상태가 된다.
    2. Justified Checkpoint를 Source로 삼아 바로 다음 Epoch의 Checkpoint가 Justified되면, 앞의 Checkpoint는 **Finalized(최종 확정)** 상태가 된다.
    3. Finalized Checkpoint와 그 이전 블록은 모두 확정된다.

    정상적인 경우 Checkpoint 블록은 약 2 Epoch(12.8분) 뒤에 Finalized된다. Epoch 중간에 있는 블록은 다음 Epoch의 Checkpoint가 Finalized될 때 함께 확정되므로, 블록 위치에 따라 최대 약 3 Epoch(19.2분)가 걸린다.

#### Design Costs and Constraints

Re-execution 방식은 검증의 독립성을 보장하지만, 검증하는 노드 수만큼 같은 계산이 반복되는 **Computational Redundancy(계산 중복)**가 발생한다. 이 반복은 Validator 단위가 아니라 노드 단위로 일어난다. 하나의 노드에 여러 Validator를 연결해 운영할 수 있으므로 Validator 수는 노드 수보다 훨씬 많다. Validator를 운영하지 않는 Full Node도 같은 계산을 수행한다.

이 설계는 Ethereum의 핵심 원칙인 **"Don't Trust, Verify"** 철학에 기반한다. 각 노드가 독립적으로 다시 실행하므로, 블록 생성자가 잘못된 State Transition을 담은 블록을 제안해도 정직한 노드는 Re-execution 과정에서 이를 발견하고 거부한다.

**누구나 일반적인 하드웨어로 Re-execution을 할 수 있도록 블록당 실행량은 Block Gas Limit으로 제한된다.** Block Gas Limit은 고정된 값이 아니다. Block Proposer가 블록마다 부모 블록 대비 1/1024 미만 범위에서 조정하며, Validator들은 통상 일반 하드웨어가 Slot 시간 안에 검증을 마칠 수 있는 수준을 고려해 목표값을 정한다.

일부 노드에 장애가 발생해도 다른 노드가 검증과 블록 제안을 계속하므로 체인은 진행된다. 다만 전체 Stake의 1/3을 초과하는 Validator가 오프라인이 되면 2/3 조건을 채울 수 없으므로 Finality가 멈추고, 온라인 Validator에 의한 블록 생성만 계속된다. Finality가 4 Epoch(약 25.6분) 넘게 성립하지 않으면 Inactivity Leak(비활성 누수)이 시작된다. Inactivity Leak은 오프라인 Validator의 Stake를 점차 줄이며, 온라인 Validator의 비율이 2/3를 회복하면 Finality가 재개된다.


<br />
<br />

## Zero-Knowledge Ethereum Virtual Machine (zkEVM)

### What is zkEVM?

zkEVM은 EVM 실행이 올바르게 수행되었음을 암호학적 증명으로 검증할 수 있게 하는 시스템이다. Re-execution 방식에서는 모든 노드가 블록을 다시 실행하여 결과를 확인한다. zkEVM에서는 소수의 Prover(증명자)가 실행하고 **그 실행이 올바르다는 증명을 만들며, 나머지는 증명만 검증한다.**

대표적인 사용처는 L2(Layer 2) ZK(Zero-Knowledge) Rollup이다. Rollup은 L1(Layer 1, Ethereum Mainnet) 밖에서 트랜잭션을 실행하고(Off-chain Execution), 그 실행의 증명을 L1의 스마트 컨트랙트가 검증하는(On-chain Proof Verification) 구조다. Ethereum L1에서도 Re-execution을 증명 검증으로 대체하는 작업이 진행 중이다.

### Types of zkEVM
 
zkEVM은 EVM과의 호환 수준에 따라 나뉜다. 아래는 Vitalik Buterin이 2022년에 제시한 [분류](https://vitalik.eth.limo/general/2022/08/04/zkevm.html)다.

| Type | 호환 수준 | 바뀌는 부분 |
|---|---|---|
| Type 1 | Ethereum과 완전히 동일 | 없음. Ethereum 블록 자체를 증명할 수 있다 |
| Type 2 | EVM과 동일 | 상태 트리, 해시 함수 등 내부 데이터 구조 |
| Type 2.5 | EVM과 동일하되 Gas 비용 변경 | 증명 비용이 큰 연산의 Gas 비용 |
| Type 3 | EVM과 거의 동일 | 구현이 어려운 일부 Opcode나 Precompile 제외 |
| Type 4 | 고급 언어 수준 호환 | Solidity 등을 다른 VM용 코드로 컴파일한다. EVM 바이트코드를 실행하지 않는다 |

Type 번호가 클수록 증명 효율은 높아지지만 Ethereum과의 호환성은 낮아진다. Ethereum L1 검증에 쓸 수 있는 것은 Ethereum과 같은 규칙을 증명하는 Type 1뿐이다.

### What Is Proven

**Type 1 zkEVM은 State Transition Function을 바꾸지 않는다.** 블록 단위 전이 $\sigma' = \Pi(\sigma, B)$의 실행 규칙은 EVM과 같다. 바뀌는 것은 그 전이가 올바른지 확인하는 방법뿐이다.

Prover가 제출하는 증명 $\pi$는 다음 명제를 증명한다.

> Pre-State Root에서 시작해 블록(또는 배치)을 EVM 규칙대로 실행하면 Post-State Root가 된다.

Verifier(증명 검증자)는 Public Input(공개 입력)과 $\pi$만으로 이 명제를 검증한다. Public Input은 Pre-State Root, 블록 또는 배치의 Commitment(트랜잭션 데이터를 대표하는 Hash), Post-State Root 등이다. **Verifier는 중간 실행 과정을 다시 계산하지 않는다.**

### Key Properties

여기서 핵심 성질은 정보를 숨기는 Zero-Knowledge(영지식성)가 아니라 **Succinctness(간결성)**이다. 실행량이 늘어도 증명 크기와 검증 시간은 거의 늘지 않으며, 검증은 실행을 다시 하는 것보다 훨씬 짧게 끝난다. 이름에 "zk"가 들어 있지만, 대부분의 zkEVM은 실행 데이터를 숨기는 것을 목적으로 하지 않는다.

증명은 실행이 올바르다는 사실만 보장하고, 트랜잭션 데이터 자체는 담지 않는다. 그래서 Rollup은 누구나 상태를 재구성할 수 있도록 트랜잭션 데이터를 L1에 따로 게시한다. 이를 **Data Availability(DA, 데이터 가용성)**라고 한다. Cancun(EIP-4844) 이후에는 주로 Blob(일정 기간 뒤 삭제되는 별도 데이터 공간)에 게시한다.

증명 시스템은 프로그램을 직접 다루지 못한다. 그래서 EVM 실행 규칙을 미리 Polynomial Constraints(다항식 제약식)로 바꿔 둔다. 이 변환을 **Arithmetization(산술화)**이라고 한다. 예를 들어 "ADD Opcode면 Stack의 두 값을 더해야 한다"는 규칙을 방정식으로 만들어 둔다.

Prover는 실제 실행 과정을 기록한 Execution Trace(실행 기록)를 만들고, 이 기록이 모든 제약식을 만족함을 증명한다. 이때 Execution Trace는 제약식에 대입되는 **Witness(증인)**, 즉 증명에 쓰이지만 공개되지 않는 값이 된다.


### End-to-End Block Verification Process

![zkevm](zkevm.svg)
_출처: [https://zkevm.ethereum.foundation/blog/zkevm-security-overview](https://zkevm.ethereum.foundation/blog/zkevm-security-overview)_

아래는 L2 ZK Rollup 기준이다. Sequencer(트랜잭션 순서를 정해 배치를 만드는 주체)가 배치를 만들고, Prover가 증명하며, L1의 Verifier 컨트랙트가 검증한다.

1. **Execution Trace Generation**

    Sequencer(또는 Executor)가 트랜잭션 배치를 실행해 State Transition을 계산한다. zkEVM은 최종 결과 상태만 계산하지 않고, **실행 과정 전체를 Execution Trace로 기록한다.** Execution Trace는 각 Opcode 실행 단계의 상태 변화를 시간 순서대로 저장한 것이다.

   ```text
    trace[t] = {
    PC_t:      Program Counter
    opcode_t:  실행된 Opcode
    stack_t:   이 스텝에서 읽고 쓴 Stack 값
    memory_t:  이 스텝의 Memory 접근 (주소, 값, 읽기/쓰기)
    storage_t: 이 스텝의 Storage 접근 (키, 이전 값, 새 값)
    gas_t:     남은 Gas
    }
    ```
    
    실제 구현은 매 행에 Memory와 Storage 전체를 복사하지 않는다. 해당 스텝의 접근 기록만 남기고, 일관성은 별도 테이블과 Lookup(조회 논증)으로 확인한다.

    예를 들어 바이트코드 `60 02 60 03 01 00`(`PUSH1 2`, `PUSH1 3`, `ADD`, `STOP`)은 다음과 같이 기록된다. Stack은 실행 후 상태이며 오른쪽이 최상단이다.

    ```text
    step 0: PC=0, opcode=PUSH1 0x02, stack=[2]
    step 1: PC=2, opcode=PUSH1 0x03, stack=[2, 3]
    step 2: PC=4, opcode=ADD,        stack=[5]
    step 3: PC=5, opcode=STOP
    ```
    
    `PUSH1`은 Immediate 1byte를 포함해 2byte를 차지하므로 다음 PC는 2다.
    
2. **Arithmetization**
    
    Execution Trace를 State Machine의 실행 기록으로 보고, 각 단계가 EVM 규칙을 지켰는지 제약식으로 확인한다. 아래 예시는 개념을 보여주기 위해 단순화한 것이다.

    ```text
    ADD Opcode 규칙:
    stack_new[0] = (stack_old[0] + stack_old[1]) mod 2^256

    → 제약식 (개념):
    stack_new[0] − (stack_old[0] + stack_old[1]) + carry × 2^256 = 0,  carry ∈ {0, 1}
    ```

    증명 시스템의 유한체는 2^256보다 작다. 예를 들어 BN254 스칼라체는 약 254bit다. 그래서 실제 구현은 256bit 값을 작은 조각(Limb)으로 나누고, 조각별 자리올림(Carry)과 범위 확인(Range Check)을 제약으로 추가한다.

    ```text
    SSTORE 규칙:
    storage_new[slot] = value

    → 제약식 (개념):
    storage_new[slot] − value = 0
    ```

    실제로는 이 한 줄로 부족하다. 어느 계정의 Storage인지, 이전 값이 실제 상태에 있던 값인지(Merkle Proof), 접근 순서, Gas와 Revert 규칙, 최종 State Root까지 연결하는 제약이 함께 필요하다.

    일반화하면 모든 인접한 실행 단계는 다음 조건을 만족해야 한다.

    $$
    P_i\big(\mathrm{trace}[t],\ \mathrm{trace}[t+1]\big) = 0 \quad \text{for all } i \text{ and } t \in [0, N-1]
    $$

    Memory와 Storage 접근의 일관성이나 값의 범위처럼 인접한 두 행만으로 표현하기 어려운 조건은 Lookup이나 Permutation(순열) 논증으로 따로 확인한다.

3. **Proof Generation**
    
    Prover는 Execution Trace(Witness)와 제약식으로 증명을 생성한다. 증명 시스템은 구현마다 다르다.

    - **STARK(Scalable Transparent Argument of Knowledge):** 대형 Trace를 처리하는 데 많이 쓰인다. Trusted Setup(신뢰 설정)이 필요 없지만, 증명 크기가 크다.

    - **Recursion(재귀 증명)과 Aggregation(집계):** 증명을 검증하는 계산을 다시 증명하거나, 여러 증명을 하나로 합친다.

    - **Wrapping:** 최종적으로는 L1에서 적은 비용으로 검증할 수 있도록, 작은 크기의 SNARK(Succinct Non-interactive Argument of Knowledge, 예: Groth16)로 감싸는 구성이 흔하다.

    증명이 보장하는 내용은 다음과 같다.

    - 모든 $t \in [0, N-1]$에 대해 $\mathrm{trace}[t] \to \mathrm{trace}[t+1]$ 전이가 EVM 규칙을 따른다.
    - $\mathrm{trace}[0]$의 상태가 Pre-State Root와 일치한다.
    - $\mathrm{trace}[N]$의 상태가 Post-State Root와 일치한다.

    즉 **전체 실행이 올바른 State Transition이었음이 하나의 증명으로 요약된다.**
    
4. **On-chain Verification**
    
    증명과 상태 정보가 L1의 Verifier 컨트랙트에 제출된다. 일반적으로 제출되는 데이터는 다음과 같다.

    - Pre-State Root
    - Post-State Root
    - 배치 Commitment (L1에 게시한 트랜잭션 데이터와 연결되는 값)
    - ZK Proof $\pi$

    Verifier 컨트랙트는 다음을 확인한다.

    1. Pre-State Root가 현재 승인된 State Root와 같은지 확인한다.
    2. 배치 Commitment가 L1에 게시된 데이터와 일치하는지 확인한다.
    3. 트랜잭션을 다시 실행하지 않고 $\pi$의 수학적 타당성만 검증한다.

    검증이 성공하면 해당 배치의 State Transition은 EVM 규칙을 만족한 것으로 간주되고, 새 State Root가 On-chain에 기록된다. 이 상태가 되돌릴 수 없게 확정되는 시점은 검증 트랜잭션이 포함된 L1 블록이 Finalized되는 시점이다.

    ```text
    Sequencer / Prover:
    1. 트랜잭션 실행 → Execution Trace 생성
    2. 증명 π 생성
    3. (Pre-State Root, Post-State Root, 배치 Commitment, π)를 L1에 제출

    L1 Verifier 컨트랙트:
    1. Pre-State Root와 배치 Commitment 확인
    2. π 검증
    3. Post-State Root 기록 → L1 Finality 이후 확정
    ```

#### Cost Structure and Constraints

Re-execution 방식에서는 검증하는 모든 노드가 같은 계산을 반복한다(Execution × N 노드). zkEVM 방식에서는 소수가 실행하고 증명하며, 나머지는 증명만 검증한다(Execution × 소수의 Prover + Verification × N 노드). L2 Rollup에서는 L1 노드가 L2 트랜잭션을 실행하지 않고 증명만 검증한다. 검증 비용은 다시 실행하는 비용보다 훨씬 작도록 설계되므로, **네트워크 전체의 계산 중복이 크게 줄어든다.**

**같은 연산이라도 증명 생성은 Re-execution보다 훨씬 느리고 비싸다.** EVM 로직을 제약식으로 표현하면 Execution Trace가 커지고, Trace의 행 수는 실행 스텝 수에 비례해 늘어난다. 증명 생성이 실행보다 몇 배 비싼지는 증명 시스템, 하드웨어, 작업량에 따라 크게 다르다.

Ethereum Foundation(EF)은 2025년 7월 L1용 Real-Time Proving(실시간 증명) 기준을 [제시했다](https://blog.ethereum.org/2025/07/10/realtime-proving). 기준은 Mainnet 블록의 99%(P99)를 10초 안에 증명하고, 장비 비용(On-prem CAPEX) 10만 달러 이하와 전력 10kW 이하를 지키는 것이다. 보안 수준(128bit), 증명 크기(300KiB 이하), 코드 공개도 기준에 포함된다. EF는 2025년 12월 18일 이 기준의 성능 목표가 달성되었다고 발표했다. 다만 이는 Ethproofs의 Off-chain 벤치마크 결과이며, Mainnet 검증에 쓰이는 것은 아니다.

Gas 비용과 증명 비용은 일치하지 않는다. Keccak-256 해시나 ECDSA 서명 검증(`ecrecover`)처럼 제약식으로 표현하기 무거운 연산이 많은 트랜잭션은 **Gas는 싸지만 증명은 매우 비싸다.** 이런 트랜잭션을 Prover Killer라고 부른다.

증명 방식 사이에는 개발 편의성과 증명 효율 사이의 Trade-off(상충 관계)가 있다.

- **zkVM(Zero-Knowledge Virtual Machine) 방식:** RISC Zero, SP1 같은 zkVM은 RISC-V 같은 범용 명령어 집합을 증명한다. 일반 프로그램, 예를 들어 Execution Client의 EVM 구현을 그대로 증명할 수 있어 범용성이 높다. 명령어 단위로 증명하므로 전용 Circuit보다 오버헤드가 크다. 그래도 L1 Real-Time Proving 결과는 주로 이 방식에서 나왔다.

- **Circuit 방식:** gnark, ZoKrates 같은 도구는 특정 계산에 맞춘 Circuit(회로, 제약식의 집합)으로 증명 효율을 높인다. 대신 Circuit을 직접 설계해야 하고, 로직이 바뀔 때마다 Circuit을 다시 만들어야 한다. Groth16을 쓰면 Circuit마다 Trusted Setup도 다시 해야 한다

Sequencer나 Prover를 소수가 운영하면 Liveness(활성) 위험이 생긴다. 운영 주체에 문제가 생기면 상태 확정이 지연된다.

#### Adoption Status on Ethereum L1

Ethereum L1에서 zkEVM으로 블록 실행을 검증하는 방안이 연구되고 있다.

**Multiproofs 전략**

Ethereum Foundation(EF, 이더리움 재단)은 [zkEVM Security Overview](https://zkevm.ethereum.foundation/blog/zkevm-security-overview)(2026년 1월 14일)에서 Multiproofs(다중 증명) 전략을 제시했다. Consensus Client가 서로 다른 여러 zkEVM 증명을 확인한 뒤에야 블록을 받아들이는 방식이다. 증명의 다양성은 Execution Client 구현과 zkVM 양쪽에서 확보한다. 지금 Execution Client(Geth, Besu, Nethermind 등)가 여러 개 존재하듯, 증명 쪽에서도 여러 조합을 써서 한 구현체의 치명적인 버그로 잘못된 블록이 받아들여지는 것을 막으려는 것이다.

단일 증명 시스템에 의존할 때의 위험은 L2에서 실제로 드러났다.
- **Scroll:** OpenVM 전환 직후 OpenVM v1.0.0의 `auipc` Opcode Circuit에서 Soundness(건전성) 버그가 발견되어, 2025년 4월 25일 긴급 업그레이드를 했다. 당시 Sequencer와 Prover를 모두 Scroll이 운영하고 있어 제3자가 악용할 수는 없었다.
- **Taiko:** SGX Enclave 서명 개인키가 공개 GitHub 저장소에 올라가 있었다. 공격자는 이 키로 DEBUG 모드 Enclave에 서명한 뒤 위조 증명을 만들었고, 2026년 6월 21~22일 Bridge에서 약 170만 달러를 탈취했다. 이 사고는 zkVM이 아니라 TEE 증명 쪽에서 일어났다.

**EIP-8025와 현재 상태**

2026년 11월 기준 Ethereum Mainnet은 Multiproofs를 채택하지 않았고, 블록은 여전히 Re-execution으로 검증된다.

첫 단계로 제안된 [EIP-8025(Optional Execution Proofs)](https://eips.ethereum.org/EIPS/eip-8025)는 Draft 상태다. 이 제안은 Beacon Node가 Execution Client를 실행하지 않고 증명을 검증해 Execution Payload의 유효성을 판단하게 한다. 증명은 Consensus Layer의 P2P(Peer-to-Peer) 네트워크로 전달된다. 참여를 원하는 노드만 증명을 만들거나 검증하는 선택적 방식이므로, Protocol 업그레이드가 이 기능의 개선에 의존할 수는 없다.

EIP-8025는 Glamsterdam 다음 Hard Fork인 Hegotá의 포함 대상으로 제안되었다. 그러나 2026년 10월 15일 ACDC(All Core Devs Consensus) #189에서 Consensus Layer 쪽 합의가 충분히 모이지 않아 DFI(Declined for Inclusion, 포함 거절)로 결정되었다.

**역할별 비용**

- **증명 생성:** Prover가 담당하며 여러 GPU로 구성된 장비가 필요하다. 모든 실행 과정을 제약식으로 증명하므로 네이티브 실행보다 훨씬 많은 연산이 든다.
- **증명 검증:** 블록의 계산량과 거의 관계없이 작다. 여러 증명을 검증해도 검증 노드의 부담은 크지 않다.

주요 위험은 두 가지다. 증명 생성이 소수의 고성능 운영자에게 집중될 수 있고, 증명 검증이 필수가 되면 증명이 늦게 도착할 때 Attestation이나 Finality가 지연될 수 있다. 결국 zkVM 기반 접근은 **검증 비용을 낮추는 대신 증명 생성 단계에 큰 비용을 옮기며, 그 부담은 Validator가 아니라 Prover가 진다.**


<br />
<br />

## Zero-Knowledge Virtual Machine (zkVM)

### What is zkVM?

zkVM은 특정 로직을 위한 전용 Circuit을 직접 설계하는 대신, ISA(Instruction Set Architecture, 명령어 집합) 자체를 증명 가능한 가상 머신으로 구현한 범용 증명 플랫폼이다. 대표적인 ISA는 RISC-V(오픈 소스 RISC(Reduced Instruction Set Computer) 명령어 집합)다.

**zkEVM이 "무엇을 증명하는가(EVM 실행)"를 가리키는 말이라면, zkVM은 "어떤 방식으로 증명하는가(범용 ISA의 실행 증명)"를 가리키는 말이다.**

- **zkEVM:** "EVM이라는 특정 State Machine이 올바르게 실행되었는가"를 증명한다.
- **zkVM:** "일반 프로그램이 정의된 ISA 규칙에 따라 올바르게 실행되었는가"를 증명한다.

두 개념은 배타적이지 않다. zkVM 위에서 EVM 실행 프로그램(예: Execution Client의 블록 실행 로직)을 돌리면 그 자체로 zkEVM이 된다. 다만 **증명은 프로그램이 ISA 규칙대로 실행되었음만 보장한다.** 그 프로그램이 Ethereum 규칙을 정확히 구현했는지는 별도로 검증해야 한다.

### How It Works

개발자는 Rust, C/C++ 같은 고급 언어로 프로그램을 작성하고, 이를 zkVM의 ISA로 컴파일한다. 지원 언어는 zkVM의 툴체인마다 다르다.

zkVM의 증명 구조는 zkEVM과 같다. 차이는 제약식으로 바꿔 두는 대상이다.

- **zkEVM:** EVM Opcode 규칙을 제약식으로 바꿔 둔다.
- **zkVM:** ISA 명령어(Instruction) 규칙을 제약식으로 바꿔 둔다(Arithmetization).

Prover는 프로그램을 실행해 Execution Trace를 만들고, 각 명령어 실행 단계가 ISA 규칙을 만족함을 증명한다. Verifier는 프로그램을 다시 실행하지 않고 증명만 검증해 실행 결과를 신뢰할 수 있다.

### Benefits and Costs

**zkVM의 장점은 유연성이다.**

- 개발자는 Circuit 설계 지식 없이 일반 프로그램으로 증명할 로직을 구현할 수 있다.
- 로직이 바뀌면 프로그램을 수정하고 다시 컴파일하면 된다.

**대신 성능 비용이 크다.** zkVM은 명령어 단위로 실행을 증명한다. 고급 언어의 코드 한 줄도 여러 명령어로 컴파일되므로, 같은 계산이라도 전용 Circuit보다 Trace가 훨씬 길어진다. 그만큼 증명 생성 시간과 비용이 늘어난다.

이를 줄이기 위해 SHA-256, Keccak-256, 타원곡선 연산 같은 무거운 연산은 zkVM 가속 Circuit으로 처리한다. 프로젝트에 따라 Precompile이나 Accelerator라고 부르며, EVM의 Precompile과는 다른 개념이다. 해당 연산을 명령어 단위로 증명하지 않고 전용 Circuit 하나로 증명한다.

### End-to-End Proving Process

아래는 RISC Zero, SP1 같은 RISC-V 기반 zkVM의 일반적인 구성이다. 모든 zkVM의 공통 구조는 아니다.

1. **Program Execution**

    Prover는 증명할 프로그램을 zkVM 안에서 실행한다. 예를 들어 블록 검증 프로그램이라면 다음을 수행한다.

    - 입력 데이터를 검증한다.
    - State Transition 규칙을 적용한다.
    - 새 State Root를 계산한다.

    zkVM은 실행 과정을 명령어 단위로 기록한다. Execution Trace에는 PC, 레지스터 상태 변화, Memory 읽기와 쓰기 기록 등이 포함된다. **Trace 길이는 실행된 명령어 수에 비례해 늘어난다.**

    긴 Trace는 한 번에 증명하기 어렵다. 그래서 일정 길이의 Segment(RISC Zero) 또는 Shard(SP1)로 나누어 각각 증명한다.

2. **Proof Generation**

    Trace가 ISA 규칙을 지켰음을 증명하는 단계다.

    - **STARK 증명:** 각 Segment에 대해 STARK 기반 증명을 생성한다. STARK는 Trusted Setup이 필요 없고 대형 Trace 처리에 유리하다.
    - **Recursion:** Segment별 증명을 Recursion으로 결합해 하나의 증명으로 합친다.
    - **Wrapping:** L1 On-chain 검증 비용을 줄이기 위해, 크기가 작고 고정된 SNARK 증명(예: Groth16)으로 감싼다.

3. **Verification**

    Verifier(Off-chain 프로그램 또는 스마트 컨트랙트)는 프로그램을 다시 실행하지 않고 증명의 타당성만 수학적으로 확인한다.
    RISC Zero를 예로 들면 증명 결과물은 Receipt라고 부르며, 세 요소로 이루어진다. 이 Receipt는 Ethereum의 트랜잭션 Receipt와 이름만 같고 다른 개념이다.

    - **Image ID:** 실행한 프로그램을 식별하는 값
    - **Journal:** 프로그램이 공개한 출력
    - **Seal:** 증명

    Verifier는 Seal이 "Image ID에 해당하는 프로그램을 실행해 이 Journal을 출력했다"는 사실을 증명하는지 확인한다. **입력은 기본적으로 비공개다.** 공개하려는 입력은 프로그램이 Journal에 직접 기록해야 한다.

    검증이 성공하면, 해당 프로그램이 수정되지 않은 채 주어진 입력으로 실행되어 그 출력을 냈다는 사실이 암호학적으로 보장된다.

### Use Cases

L2 Rollup과 Ethereum L1 실행 증명 모두 zkVM 기반으로 옮겨가고 있다.

- **Scroll:** 2025년 4월 Euclid 업그레이드에서 기존 halo2 Circuit을 폐기하고 OpenVM 기반 Prover로 전환했다.
- **Taiko:** 다중 증명(Multi-proof)을 사용한다. 2026년 11월 기준 SGX(Geth), SGX(Reth), SP1, RISC Zero 네 가지 증명 중 정확히 두 개가 필요하며, 그중 최소 하나는 SP1이나 RISC Zero여야 한다. SGX는 Intel SGX(Software Guard Extensions)를 쓰는 TEE(Trusted Execution Environment, 신뢰 실행 환경) 증명으로, 암호학적 증명이 아니라 하드웨어 신뢰에 기반한다.
- **Ethereum L1:** EIP-8025(Optional Execution Proofs) 설계에서는 Execution Client의 블록 실행 프로그램을 zkVM으로 증명한다(zkEVM 문서 참고).

### Dedicated Circuit vs zkVM

zkEVM은 두 방식 모두로 구현할 수 있다. 따라서 비교 대상은 zkEVM과 zkVM이 아니라 **증명 방식**이다.

| 구분 | 전용 Circuit 방식 | zkVM 방식 |
|---|---|---|
| 증명 대상 | 특정 계산 (예: EVM Opcode) | ISA로 컴파일된 임의의 프로그램 |
| 증명 단위 | 해당 계산에 맞춘 제약식 | ISA 명령어 |
| 증명 효율 | 높음 | 명령어 단위 오버헤드가 크며, 무거운 연산은 가속 Circuit으로 보완 |
| 유지보수 | 로직이 바뀌면 Circuit을 다시 설계 | 프로그램을 수정하고 다시 컴파일 |
| 사례 | Linea | 범용 zkVM: RISC Zero, SP1, OpenVM, Jolt / zkVM 기반 zkEVM: Scroll(OpenVM), Taiko(SP1, RISC Zero와 TEE의 다중 증명) |

Verifier가 확인하는 내용도 다르다.

- **zkEVM:** Verifier는 "블록의 트랜잭션들이 EVM 규칙대로 상태를 바꿨다"는 것을 확인한다.
- **zkVM:** Verifier는 "어떤 프로그램(Image ID)이 주어진 입력으로 실행되어 이 출력을 냈다"는 것을 확인한다. 그 프로그램이 Ethereum 블록 실행 로직이면 zkEVM으로 동작하는 것이고, 게임 엔진이면 게임 결과의 무결성을 증명하는 것이 된다.
