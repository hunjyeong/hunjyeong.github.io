---
title: "RISC Zero zkVM: Architecture and Proving Process"
date: 2026-03-27 00:01:00 +0900
last_modified_at: 2026-11-16
categories: [Blockchain]
tags: [Blockchain, ZKP, zkVM, RISC Zero]
math: true
media_subpath: /assets/img/posts/zkevm/
---

>
> 1. [EVM, zkEVM, zkVM: Execution and Verification]({% post_url Blockchain/2026-03-27-zkEVM_zkVM %})
> 2. **RISC Zero zkVM: Architecture and Proving Process**
> 3. [SP1 zkVM: Architecture and Proving Process]({% post_url Blockchain/2026-03-27-SP1-zkVM %})
> 4. [Circuit-Based SNARK Tools: gnark and ZoKrates]({% post_url Blockchain/2026-03-27-gnark-ZoKrates %})
>

이 글은 RISC Zero zkVM 2.0 이후 구조를 기준으로 하며, 수치는 2026년 11월 10일(UTC) 공개된 risc0 3.1.0 릴리스와 그 시점의 공식 문서, 소스를 확인한 값이다. 버전에 따라 달라지는 항목은 본문에 표시했다. 3.1.0 릴리스에 포함된 크레이트는 버전이 따로 매겨진다. SDK(Software Development Kit) 크레이트 `risc0-zkvm`은 3.1.0, 증명 시스템 크레이트 `risc0-zkp`와 바이너리 형식 크레이트 `risc0-binfmt`은 3.0.6, RV32IM Circuit 크레이트 `risc0-circuit-rv32im`은 4.1.0이며, 본문의 docs.rs 링크는 이 버전을 가리킨다. Solidity 컨트랙트는 별도 저장소(risc0-ethereum)의 `v3.1.0` 태그를 링크했다. dev.risczero.com 문서는 버전 경로가 없는 최신판 주소를 링크했으므로, 확인 시점(2026년 11월 16일)의 내용을 기준으로 했다. 이 시점의 최신판은 3.1 스냅샷이며, 2026년 11월 11일 문서 수정([PR #3839](https://github.com/risc0/risc0/pull/3839))으로 Bonsai와 Boundless 관련 내용이 제거된 상태다.


## What is RISC Zero?

RISC Zero는 범용 프로그램의 실행을 ZKP(Zero-Knowledge Proof, 영지식 증명)로 검증할 수 있게 만든 zkVM(Zero-Knowledge Virtual Machine, 영지식 가상 머신)이다. 특정 연산마다 Circuit(회로)을 설계하는 대신, **RISC-V(RV32IM) CPU 자체를 증명 가능한 가상 머신으로 구현한다.** 따라서 RISC Zero가 증명하는 것은 다음 사실이다.

> 주어진 프로그램이 특정 입력에 대해 RISC-V 명령어 규칙에 따라 정상적으로 실행되었다.

RISC Zero는 Verifiable Computing(검증 가능한 연산) 모델을 따른다. 무거운 계산은 Prover(증명자)가 맡고, Verifier(검증자)는 프로그램을 다시 실행하지 않고 증명만 확인한다.

```text
기존 실행 모델: Input → Program Execution → Output
RISC Zero:      Input → Program Execution (RISC-V) → Output + 증명(Receipt)
```

실행 결과로 생성되는 증명 객체를 **Receipt(영수증)**라고 한다. [Key Terminology](https://dev.risczero.com/terminology) 기준으로 Receipt는 두 부분으로 이루어진다.

- **Claim(주장):** 증명하려는 실행에 대한 진술이다. 어떤 프로그램(Image ID)을 실행했는지, 어떻게 종료했는지(종료 상태), 무엇을 공개했는지(Journal), 종료 시 Memory 상태가 어떠했는지를 담는다.

- **Seal:** Claim이 참임을 입증하는 암호학적 증명이다.

**Journal**은 프로그램이 공개한 Public Output(공개 출력)이다. Claim에는 Journal이 Digest(다이제스트, 해시값) 형태로 들어가고, Receipt에는 Journal 원문이 함께 담긴다.

증명 시스템은 STARK(Scalable Transparent Argument of Knowledge)를 기반으로 한다. RISC-V Circuit의 STARK는 BabyBear Field(체, $$p = 15 \cdot 2^{27} + 1$$) 위에서 동작하며, 기본 해시 함수는 Poseidon2다(`risc0-zkvm`의 `ProverOpts::default()`가 `hashfn = "poseidon2"`를 사용). 자세한 API는 [RISC Zero zkVM 문서](https://dev.risczero.com/api/zkvm/)에서 확인할 수 있다.

### Guest와 Host

RISC Zero zkVM은 증명 대상과 실행 관리자를 분리한다.

- **Guest:** 증명 대상 프로그램이다. Rust나 C, C++로 작성해 RISC-V ELF(Executable and Linkable Format) 바이너리로 컴파일한다.

- **Host:** Guest 실행을 관리한다. Guest에 입력을 전달하고 실행기를 구동하며, Receipt 생성을 요청하고 검증한다. 증명은 로컬 장비나 원격 증명 서비스에서 생성한다.

2.0부터 zkVM은 User 모드와 Machine 모드를 구분한다. Guest 쪽 코드는 Kernel(커널)과 User 프로그램으로 나뉘며, 둘 다 RV32IM 명령어로 실행된다. User 프로그램이 입출력이나 가속 Circuit 호출을 위해 ECALL(Environment Call)을 실행하면 zkVM이 이를 가로채 Kernel로 제어를 넘기고, Kernel이 해당 시스템 콜 핸들러로 분기한다. 핸들러는 필요하면 Host에 계산을 위임하고, 결과를 User 프로그램에 돌려준다. 명세는 zkVM이 RISC-V의 CSR(Control and Status Register) 명령을 구현하지 않으므로 일반적인 특권 모드와 User 모드가 없다고 쓰고, 다음 절에서 User 모드와 Machine 모드를 구현한다고 쓴다. 두 문장을 함께 읽으면, 이 User와 Kernel 분리는 RISC-V 표준의 CSR 기반 특권 구조가 아니라 zkVM이 따로 구현한 것으로 해석할 수 있다. 메모리는 User 영역(`0x00010000`부터 약 3GB)과 Kernel 영역(`0xC0000000`부터 약 1GB)으로 나뉘며, 그 앞의 `0x00000000~0x0000FFFF` 구간은 사용할 수 없다. 3.1.0 소스(`risc0-circuit-rv32im`의 `platform.rs`)에서 Kernel 영역은 `0xFF000000`에서 끝나며(`KERNEL_END_ADDR`), 그 뒤 구간에 레지스터가 매핑된 주소(`0xFFFF0000`부터)와 ECALL 분기 주소(`0xFFFF1000`) 등이 있다. 자세한 내용은 [zkVM Technical Specification](https://dev.risczero.com/api/zkvm/zkvm-specification)에 있다.

## End-to-End Proving Process

1. **Guest 프로그램 작성**

    Rust나 C, C++로 증명할 로직을 작성한다. 예를 들어 서명 검증, Merkle Proof 검증, State Transition(잔고, Nonce) 처리 같은 로직이다. **개발자는 Circuit을 정의하지 않는다.**

    - **입력:** `env::read()`로 Host에서 데이터를 받는다. 읽은 값은 기본적으로 비공개다.

    - **출력:** `env::commit()`으로 Journal에 기록한다. Journal은 증명과 함께 외부에 공개된다.

    [guest 모듈 문서](https://docs.rs/risc0-zkvm/3.1.0/risc0_zkvm/guest/index.html)에 따르면 `risc0-zkvm` 크레이트는 기본적으로 Guest에 Rust `core`만 제공하고 `std`는 제공하지 않는다. `std` 기능을 켜지 않으면 `#![no_std]`, `#![no_main]`, `risc0_zkvm::guest::entry!(main)` 세 줄이 필요하다. `std` 기능을 켜면 진입점이 자동으로 링크되므로 세 줄을 모두 생략할 수 있다. 아래 예시는 `std` 기능을 켠 경우다.

    ```rust
    use risc0_zkvm::guest::env;

    fn main() {
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

2. **Guest 컴파일과 Image ID 생성**

    Guest 코드는 zkVM이 실행할 수 있는 RISC-V ELF 바이너리로 컴파일된다. 보통 `methods` 크레이트의 빌드 스크립트(`risc0-build`)가 컴파일하며, 재현 가능한 빌드가 필요하면 Docker 안에서 컴파일하는 `cargo risczero build`를 쓴다.

    ```text
    Rust/C/C++
    ↓
    risc0-build (또는 cargo risczero build)
    ↓
    RISC-V RV32IM ELF 바이너리 (User ELF)
    ↓
    User ELF와 Kernel ELF를 합친 프로그램 바이너리 구성
    ↓
    초기 Memory 이미지로 적재 (1kB Page 단위) 후 Page들로 Merkle Tree 구성
    ↓
    Image ID = 초기 Memory 이미지를 커밋한 32byte 값
    ```

    **Image ID**는 Verifier가 어떤 프로그램이 실행되었는지 식별하는 값이다. 계산 방식은 버전에 따라 다르다.

    - **1.x:** 초기 Memory 이미지의 Merkle Root와 시작 PC(Program Counter)를 묶어 해시한 값이다(`risc0-binfmt` 1.2.6의 `SystemState::digest`).

    - **2.0 이후:** User ELF와 Kernel ELF를 합친 프로그램 바이너리에 대해 계산한다(`risc0_zkvm::compute_image_id`는 "combined user ELF + kernel ELF binary"를 입력으로 받는다). 따라서 Guest 코드뿐 아니라 Kernel이 바뀌어도 Image ID가 바뀐다.

    [FAQ](https://dev.risczero.com/faq)는 Image ID 계산에 바이너리 중 실제로 적재되는 Page만 쓰이므로, 디버그 정보나 타임스탬프처럼 실행에 영향을 주지 않는 부분이 달라도 Image ID는 같다고 설명한다. 다만 이 FAQ 항목은 1.x 형식의 함수(`page_table_addr`, `pc`)를 예로 들고 있어, 2.0 이후 동작의 직접적인 근거로 보기는 어렵다. Page 크기는 1,024byte이며, 이 값은 [`risc0-binfmt` 크레이트](https://docs.rs/risc0-binfmt/3.0.6/src/risc0_binfmt/lib.rs.html)의 `PAGE_BYTES` 상수로 정의되어 있다.

3. **zkVM 실행과 Execution Trace 생성**

    Host의 Executor가 RISC-V Interpreter로 바이너리를 실행하고, 각 명령어 처리 과정을 Execution Trace(실행 기록)로 기록한다. 실행 길이는 **Cycle(사이클)** 단위로 센다. [Key Terminology](https://dev.risczero.com/terminology)는 이를 Clock Cycle이라고 부르며, zkVM 연산의 최소 단위로 물리 CPU의 클럭 사이클에 해당하고 Guest 실행의 복잡도를 재는 기준이라고 설명한다. 개념적으로는 매 Cycle마다 명령어 실행 전후의 상태 변화를 기록한다.

    - **실행 전 상태:** 실행 중인 명령어의 주소(PC), 읽은 레지스터 값, 읽은 Memory 값. 정수 레지스터는 x0~x31의 32개이며 x0은 항상 0이다.

    - **실행 명령어:** RV32IM 명령어 중 하나. ISA(Instruction Set Architecture) 명세 기준으로 RV32I가 40개, M 확장(곱셈, 나눗셈)이 8개로 총 48개이며, ADD, LW(Load), SW(Store), JAL(Jump) 등이 있다. zkVM은 여기에 ECALL을 이용해 영지식 계산에 필요한 기능(입출력, 가속 Circuit 호출)을 추가한다.

    - **실행 후 변화:** 결과가 쓰인 레지스터, Memory에 쓴 주소와 값, 다음 PC

    실제로는 매 Cycle 전체 Memory와 레지스터를 복사하지 않는다. 그 Cycle에서 읽고 쓴 값만 기록하고, Memory 일관성은 별도 논증으로 확인한다.

    Execution Trace는 행이 실행 Cycle, 열이 PC, 명령어, 레지스터 값, Memory 접근 정보인 행렬 형태의 테이블이다. 아래는 x10 = 0x00020000으로 가정한 개념 예시다. 주소는 모두 앞에서 본 User 영역(`0x00010000`부터) 안에 있다.

    ```text
    | Cycle | PC         | 명령            | 레지스터 쓰기 | MemAddr    | Mem 쓰기 |
    |-------|------------|-----------------|---------------|------------|----------|
    | 0     | 0x00010000 | addi x5, x0, 5  | x5 = 5        | -          | -        |
    | 1     | 0x00010004 | addi x6, x0, 3  | x6 = 3        | -          | -        |
    | 2     | 0x00010008 | add  x7, x5, x6 | x7 = 8        | -          | -        |
    | 3     | 0x0001000c | sw   x7, 0(x10) | -             | 0x00020000 | 8        |
    ```

    RV32IM 명령어는 4byte이므로 분기가 없으면 PC는 4씩 증가한다. zkVM은 비정렬 주소 접근을 지원하지 않는다. 명령 코드는 4의 배수 주소에 있어야 하고, 워드(LW, SW) 접근은 4byte 정렬, 하프워드(LH, LHU, SH) 접근은 2byte 정렬이 필요하며, 바이트(LB, LBU, SB) 접근에는 정렬 제한이 없다(`risc0-circuit-rv32im`의 실행기 소스 기준). 명세는 정렬 위반 같은 예외가 발생하면 예외 핸들러를 실행하지 않고 종료한다고 쓴다. 3.1.0 소스의 동작은 조금 다르다. 실행기는 트랩 디스패치 테이블(`TRAP_DISPATCH_ADDR = 0xFFFF2000`)에서 원인별 핸들러 주소를 읽어, 그 값이 정렬된 Kernel 영역 주소이면 Machine 모드로 전환해 분기하고, 아니면 "Invalid trap address" 오류로 실행을 끝낸다. 기본 Kernel(`risc0/zkos/v1compat`. `risc0-build`의 `GuestOptions::kernel()`은 따로 지정하지 않으면 `V1COMPAT_ELF`를 쓴다)은 ECALL 분기 주소만 초기화하고 트랩 디스패치 테이블은 채우지 않으므로, 기본 Kernel에서는 결과적으로 정렬 위반 시 실행이 오류로 종료된다. 참고로 Rust 컴파일러가 비정렬 접근을 바이트 단위 명령으로 풀어 내보내는 경우는 예외가 아니며, Optimization Guide는 비정렬 `u32` 읽기가 정렬된 읽기(1 Cycle)의 12배인 12 Cycle이 든다고 적는다.

    **Continuations:** 실행이 길면 Trace 하나로 처리하기 어렵다. RISC Zero는 실행을 여러 **Segment**로 나누어 Trace를 만든다. Segment 크기는 `segment_limit_po2`로 정한다. po2(Power of Two)는 크기를 $$2^{n}$$ Cycle로 나타낼 때의 지수 $$n$$이다. 기본값은 $$2^{20}$$(1,048,576) Cycle이다([`risc0-circuit-rv32im`](https://docs.rs/risc0-circuit-rv32im/4.1.0/risc0_circuit_rv32im/)의 `execute::DEFAULT_SEGMENT_LIMIT_PO2 = 20`). 허용 범위는 [`risc0-zkp`](https://docs.rs/risc0-zkp/3.0.6/src/risc0_zkp/lib.rs.html)의 `MIN_CYCLES_PO2`(13)부터 `MAX_CYCLES_PO2`(24)까지다. 기본 Verifier 설정이 받아들이는 최대 Segment 크기는 이와 별도로 `risc0-zkvm`의 [`DEFAULT_MAX_PO2`](https://docs.rs/risc0-zkvm/3.1.0/risc0_zkvm/constant.DEFAULT_MAX_PO2.html)로 정해져 있으며, 3.1.0 기준 값은 22다. 각 Segment는 독립적으로 증명된 뒤 Recursion으로 합쳐진다.

    실제 Cycle 수는 명령 수와 정확히 같지 않다. [Guest Optimization Guide](https://dev.risczero.com/api/zkvm/optimization) 기준으로 덧셈, 비교, 점프, 왼쪽 시프트, Load, Store는 한 Cycle이고, 비트 연산(AND, OR, XOR), 나눗셈, 나머지, 오른쪽 시프트는 두 Cycle이다. 부동소수점 명령은 구현되어 있지 않아 소프트웨어로 에뮬레이션되며 60~140 Cycle이 든다. 또한 Segment 안에서 Page를 처음 읽을 때의 Page-In, 수정한 Page를 Segment 끝에 내보내는 Page-Out에도 Cycle이 든다. Page-In은 Image ID에 대한 Merkle 포함 증명을 확인하는 해시 연산이므로 비용이 크다. 확인 시점의 Optimization Guide는 Page-In과 Page-Out 한 번에 1,094~5,130 Cycle, 평균 1,130 Cycle이 든다고 적는다. Segment의 첫 Page-In은 Merkle Root(Image ID)까지 경로를 모두 해시하므로 5,130 Cycle이 들고, 한 번 검증된 경로는 다시 해시하지 않으므로 이후에는 대부분 Page 하나만 해시한다. 마지막 Segment는 이어질 실행이 없어 Page-Out을 하지 않는다. 이 문서는 버전 경로가 없으므로 수치는 확인 시점 기준이다.

4. **Arithmetization**

    Trace의 각 열을 다항식의 Evaluation(평가값)으로 보고, iNTT(Inverse Number Theoretic Transform, 역 NTT)로 그 값들을 지나는 다항식을 구한다. 한 행이나 인접한 두 행만 보는 검증 규칙은 두 종류의 제약으로 표현한다.

    - **Transition Constraint(전이 제약):** Trace의 인접한 두 행 사이의 제약

    - **Boundary Constraint(경계 제약):** 첫 행과 마지막 행에 대한 제약

    이 표현 방식을 **AIR(Algebraic Intermediate Representation)**라고 한다. Guest가 정상 실행되었다면 모든 제약 다항식이 Trace의 정의역에서 0이 된다. Memory 일관성처럼 Trace 전체에 걸친 성질은 한 행이나 인접한 행만 보는 이 두 제약으로 표현할 수 없으므로, Verifier가 정한 무작위 값과 그 값으로 계산한 보조 열을 쓰는 Permutation 논증으로 확인한다(비대화형 증명에서 이 무작위 값을 정하는 방법은 5단계의 Fiat-Shamir 변환에서 설명한다).

    개념적으로 표현하면 다음과 같은 제약이 있다.

    - **PC 제약:** 분기나 점프가 없으면 $$PC_{n+1} = PC_n + 4$$, 분기나 점프가 실행되면 $$PC_{n+1} = \text{target}_n$$

    - **레지스터 제약:** $$\text{rd}_{n+1} = \mathrm{ALU}(\text{rs1}_n, \text{rs2}_n, \text{imm}) \bmod 2^{32}$$이며, $$\text{rd} = x_0$$이면 값이 바뀌지 않는다. rd는 결과를 쓰는 Destination Register(대상 레지스터), rs1과 rs2는 피연산자를 읽는 Source Register(원본 레지스터), imm은 명령에 담긴 Immediate(즉값), ALU(Arithmetic Logic Unit)는 명령이 정하는 산술, 논리 연산이다. 다만 레지스터는 메모리 주소에 매핑되어 있다(3.1.0 소스 기준 Machine 레지스터 `0xFFFF0000`, User 레지스터 `0xFFFF0080`). 따라서 실제 Circuit에서 레지스터 읽기와 쓰기는 Memory 접근으로 처리된다. 명세의 메모리 구조표는 레지스터 내용이 Kernel Memory에 있다고 적지만, 3.1.0 소스의 레지스터 주소는 그 범위(`0xC0000000~0xFF000000`) 밖이다.

    - **Memory 제약:** 어떤 주소에서 읽은 값은 그 주소에 마지막으로 쓴 값과 같아야 한다. Memory 접근 기록을 주소와 시각 순으로 정렬한 기록이 원래 기록의 Permutation(순열)임을 위의 Permutation 논증으로 확인한다.

    - **디코딩과 범위 제약:** 명령 워드가 유효한 RV32IM 명령으로 디코딩되어야 한다. BabyBear Field의 원소는 $$2^{31}$$보다 작아 32bit 값 하나를 담을 수 없으므로, RISC Zero는 32bit 워드를 16bit 하프워드 두 개로 나누어 표현하고 각 하프워드가 16bit 범위 안에 있는지 제약한다. [3.1.0](https://github.com/risc0/risc0/releases/tag/v3.1.0)에서 고친 결함 중에는 Host가 `sys_read`로 넘긴 워드에 이 범위 검사가 빠져 있던 문제도 있다.

    실행 시작 위치에 대한 제약이 부족하면 어떤 일이 생기는지 보여주는 사례가 있다. [3.1.0](https://github.com/risc0/risc0/releases/tag/v3.1.0) 이전의 RV32IM Circuit은 Trace 안에 실행 시작 행(`isFirstCycle`)이 몇 개 있어도 되도록 두었다. Memory 논증은 주소, 시각, 값만으로 접근을 식별하므로, 악의적인 Prover가 한 Trace에 두 실행을 넣고 한 실행의 읽기가 다른 실행의 쓰기 값을 돌려받게 만들 수 있었다. 3.1.0은 Trace 길이를 공개 값(`totalCycles`)으로 두고, 모든 시작 행 직전 행이 Trace의 마지막 행이어야 한다는 제약을 추가해 Trace가 정확히 하나의 실행만 담도록 고쳤다. Verifier는 이 공개 값을 증명 크기($$2^{po2}$$)와 대조한다. 이 수정으로 RV32IM Seal 형식 버전(`RV32IM_SEAL_VERSION`)은 3이 되었다(`totalCycles`와 Seal 버전은 릴리스 페이지가 아니라 릴리스 PR #3831에 적혀 있다).

5. **STARK 증명, Recursion, Wrapping**

    - **커밋(LDE와 Merkle Tree):** Prover는 Trace 다항식을 원래 정의역보다 4배 큰 정의역에서 다시 평가하고(LDE, Low-Degree Extension), 그 값을 Poseidon2 해시로 만든 Merkle Tree로 커밋한다. 이 4배가 Reed-Solomon 확장률의 역수(`INV_RATE = 4`)다. 모든 제약을 무작위 값으로 묶은 다항식을 Trace 정의역에서 0이 되는 Vanishing Polynomial(소거 다항식)로 나눈 몫도 같은 방식으로 커밋한다. 제약이 Trace 정의역 전체에서 0일 때만 이 몫이 낮은 차수의 다항식이 된다.

    - **Fiat-Shamir 변환:** STARK의 바탕인 IOP(Interactive Oracle Proof)는 Verifier가 무작위 값을 보내는 대화형 프로토콜이다. RISC Zero는 Fiat-Shamir 변환으로 지금까지의 커밋(Merkle Root)을 해시해 무작위 값을 만든다. 따라서 Prover 혼자 증명을 만들 수 있고, Verifier는 같은 해시를 다시 계산해 Permutation 논증의 무작위 값과 아래 FRI의 조회 위치를 재현한다.

    - **FRI:** Prover는 FRI(Fast Reed-Solomon Interactive Oracle Proof of Proximity)로 커밋한 값이 낮은 차수의 다항식에 가깝다는 것을 증명한다. FRI는 다항식을 반복해서 Folding(접기)하여 차수를 줄인다. 일반적인 설명에서는 한 번에 차수를 절반으로 줄이며, RISC Zero는 한 번에 16배씩 접어(`FRI_FOLD = 16`) 차수가 `FRI_MIN_DEGREE`(256)에 도달할 때까지 반복한다. Verifier는 Fiat-Shamir로 정해진 무작위 위치 50곳(`QUERIES`)에서 커밋된 값을 Merkle 경로와 함께 확인한다. 상수는 [risc0-zkp 소스](https://docs.rs/risc0-zkp/3.0.6/src/risc0_zkp/lib.rs.html)에서 확인할 수 있으며, 소스 주석은 이 설정이 97bit의 Conjectured(추정) 보안 수준을 목표로 한다고 밝힌다. 이 수치는 Segment 크기에 따라 달라진다. [`DEFAULT_MAX_PO2` 문서](https://docs.rs/risc0-zkvm/3.1.0/risc0_zkvm/constant.DEFAULT_MAX_PO2.html)는 po2 21까지 97bit이고, 21을 넘으면 po2당 1bit씩 낮아져 24에서 94bit라고 설명한다. 같은 문서는 기본값으로 21을 골랐다고 쓰지만, 3.1.0 코드의 실제 값은 22다. 문서의 설명대로라면 기본 설정이 받아들이는 최대 크기(po2 22)의 보안 수준은 96bit가 된다. [ETHBEAT의 ZK Catalog](https://ethbeat.org/zk-catalog/risc0)도 RISC Zero On-chain Verifier의 보안 목표를 96bit로 적고 있어 이 계산과 일치한다.

    - **병렬 증명:** 각 Segment의 STARK 증명은 독립적이므로 여러 GPU나 여러 장비에서 병렬로 생성할 수 있다. 원격 증명 서비스는 2026년 11월 기준으로 공식 경로가 없다. [Boundless 문서](https://docs.boundless.network/developers/tutorials/bonsai)에 따르면 RISC Zero가 운영하던 중앙화된 증명 서비스 Bonsai는 2025년 12월에 종료되었고 Boundless가 그 대체로 안내되었다. 그러나 Boundless Foundation은 2026년 11월 3일 [공지](https://boundless.network/blog/zkc-introducing-tokenized-ai-inference)에서 ZK 증명 운영을 무기한 중단한다고 밝혔고, 2026년 11월 5일부터 새 증명 요청 접수와 증명 보상 청구를 끝냈다. RISC Zero 문서 저장소는 2026년 11월 11일 [PR #3839](https://github.com/risc0/risc0/pull/3839)에서 Boundless 증명이 종료되었다고 적고, Remote Proving 페이지를 삭제하고 Proving options와 Local proving 문서를 dev-mode와 로컬 증명만 다루도록 바꿨다. 같은 날 risc0-ethereum [PR #748](https://github.com/risc0/risc0-ethereum/pull/748)은 Boundless 문서가 증명 집계에 쓴다고 설명하는 `RiscZeroSetVerifier`의 0.9.0을 Emergency Stop 처리하고 Set aggregation이 deprecated라고 적었다.

    - **Recursion:** RISC Zero zkVM은 세 가지 Circuit으로 구성된다. RISC-V 실행을 증명하는 RISC-V Circuit, STARK 증명 검증에 특화된 별도의 STARK Circuit인 Recursion Circuit, 그리고 Recursion Circuit의 증명을 검증하는 **STARK-to-SNARK Circuit**이다. SNARK는 Succinct Non-interactive Argument of Knowledge의 줄임말이다. STARK-to-SNARK Circuit은 R1CS(Rank-1 Constraint System) Circuit 작성 언어인 Circom으로 작성되어 있다. 문서에 따라 Groth16 Circuit이라고도 부르며 소스 파일은 `stark_verify.circom`이다. 이 글에서는 STARK-to-SNARK Circuit으로 부른다. 공식 순서는 다음과 같다. 각 Segment 증명을 `lift`로 Recursion Circuit용 증명인 Succinct Receipt로 변환하고, `join`으로 둘씩 합치는 과정을 하나가 남을 때까지 반복한다. 마지막 Succinct Receipt는 `identity_p254`를 거친다. `identity_p254`는 Poseidon254 해시 함수를 쓰는 Recursion Prover로 증명을 다시 검증하는 단계로, Groth16 Prover 실행 직전에 수행된다. Succinct Receipt는 실행 길이와 무관한 상수 크기이며 약 200kB다. 자세한 내용은 [Recursive Proving 문서](https://dev.risczero.com/api/recursion)에 있다.

    - **Groth16 Wrapping:** STARK 증명은 커서 Ethereum On-chain 검증 비용이 높다. 그래서 Succinct Receipt를 검증하는 로직을 STARK-to-SNARK Circuit으로 구현해 Groth16 증명으로 감싼다(`compress`). Groth16은 증명이 작고 검증 비용이 일정한 Pairing 기반 zk-SNARK이며, BN254는 Ethereum이 Precompiled Contract(사전 컴파일 컨트랙트, EIP-197)로 Pairing 연산을 지원하는 타원곡선이다. Pairing은 G1과 G2라는 두 Group(군)의 원소를 하나씩 받아 GT(Target Group)의 원소를 내는 함수다. Groth16 증명은 G1의 두 점(각 64byte)과 G2의 한 점(128byte)으로 이루어져 256byte이며, On-chain Seal에는 앞에 4byte Selector가 붙어 260byte가 된다.

        모든 Recursion 프로그램은 출력의 첫 칸에 허용된 Recursion 프로그램 목록의 Merkle Root인 **Control Root**를 기록한다. 그중 Recursion 증명을 입력으로 받는 프로그램(`join`, `resolve`, `identity_p254` 등)은 입력 증명을 만든 프로그램이 이 목록에 속하는지 확인한다. STARK-to-SNARK Circuit은 마지막 `identity_p254` 증명을 검증하고, 그 출력의 Control Root와 Claim Digest를 공개 입력으로 내놓는다. `identity_p254` 프로그램을 식별하는 **BN254 Control ID**도 공개 입력이다. On-chain Verifier는 배포할 때 고정한 Control Root와 BN254 Control ID를 공개 입력으로 넣어 검증하므로, 다른 목록으로 만든 증명은 통과하지 못한다.

        STARK-to-SNARK Circuit에는 Trusted Setup이 필요하다. 3.1.0 이전 Circuit에 대해서는 PSE(Privacy and Scaling Explorations)의 p0tion 도구로 공개 세레머니를 진행했다. 진행 시기는 문서에 날짜가 없어, 기여 증명 Gist의 날짜로 보아 2024년 4월경으로 추정된다. [ETHBEAT(구 L2BEAT)의 ZK Catalog](https://ethbeat.org/zk-catalog/risc0)는 이 기존 세레머니의 Phase 2 참여자를 238명으로 집계한다. 검증 절차는 [Trusted Setup Security 문서](https://dev.risczero.com/api/trusted-setup-ceremony)에 있지만, 2026년 11월 16일 기준으로 이 문서는 기존 세레머니(Circom v2.2.2)만 설명한다. 3.1.0에서는 Circuit 수정(출력 limb의 범위 검사 추가, Poseidon254 S-box 수정)에 따라 새 Trusted Setup(2026-10-20.1)과 새 Verifying Key(`risc0-groth16` 3.0.6)를 적용했다. 릴리스 페이지에는 새 Setup의 절차가 없지만, 릴리스 PR [#3831](https://github.com/risc0/risc0/pull/3831)은 새 Setup이 PSE의 `ppot_0080_23` Powers of Tau 위에서 수행한 **단일 참여자(single-party) Phase 2**이며, `snarkjs zkey verify`로 검증했고 transcript를 `groth16_proof/groth16/zkey_verify_2026-10-20.1.txt`에 두었다고 적는다. 즉 기존 세레머니는 참여자 238명의 공개 Phase 2였지만, 3.1.0 Verifier가 쓰는 Setup은 Phase 2 참여자가 RISC Zero 한 곳이다. Phase 2 참여자가 하나이면 그 참여자가 toxic waste를 폐기했다는 것을 외부에서 검증할 수 없으므로, 3.1.0 Groth16 Receipt의 건전성은 이 점에서 RISC Zero를 신뢰하는 데 기댄다.

    최종 [Receipt](https://docs.rs/risc0-zkvm/3.1.0/risc0_zkvm/struct.Receipt.html)는 다음 구조다. Seal은 최상위 필드가 아니라 `inner` 안에 들어 있다.

    ```rust
    pub struct Receipt {
        pub inner: InnerReceipt,       // 내부 증명 (Composite, Succinct, Groth16 등). Seal 포함
        pub journal: Journal,          // 공개 출력 (commit 결과)
        pub metadata: ReceiptMetadata, // 증명 시스템과 SDK 버전 등 메타데이터 (암호학적으로 보증되지 않음)
    }
    ```

    InnerReceipt의 종류는 다음과 같다.

    - **Composite Receipt:** Segment 증명을 모두 담으므로 실행 길이에 비례해 커진다.

    - **Succinct Receipt:** Recursion으로 합친 상수 크기의 STARK 증명이다.

    - **Groth16 Receipt:** Succinct Receipt를 Groth16으로 감싼 증명이다. 가장 작은 형식으로, 직렬화하면 수백 byte다.

6. **Verification**

    Verifier는 **기대하는 Image ID, Seal, Journal**을 대조한다. Image ID는 Receipt에 별도 필드로 들어 있지 않다. Seal이 보증하는 Claim의 시작 상태(Image ID)가 Verifier가 넘긴 Image ID와 같은지 확인한다.

    - **Off-chain 검증:** RISC Zero Rust SDK의 `receipt.verify(image_id)`를 호출한다. 서버, 클라이언트 앱, 다른 체인의 노드 등에서 쓴다. SDK는 InnerReceipt 종류에 맞는 검증 알고리즘을 수행하고, 프로그램이 정상 종료(`Halted(0)`)했는지와 Journal이 증명된 출력과 일치하는지도 확인한다. 정상 종료 여부와 Image ID를 확인하지 않고 Seal이 Claim과 Journal을 보증하는지만 확인하는 `verify_integrity_with_context`도 별도로 있다. 이 함수를 쓰면 호출자가 Claim의 필드를 직접 확인해야 한다.

    - **On-chain 검증:** Solidity Verifier 컨트랙트([RiscZeroGroth16Verifier.sol](https://github.com/risc0/risc0-ethereum/blob/v3.1.0/contracts/src/groth16/RiscZeroGroth16Verifier.sol))의 `verify(bytes seal, bytes32 imageId, bytes32 journalDigest)`를 호출한다. `seal`은 4byte Selector 뒤에 ABI(Application Binary Interface) 인코딩된 Groth16 증명이 붙은 형태다. 검증은 `verify`를 부르기 전 애플리케이션 컨트랙트가 하는 일과 Verifier 컨트랙트 내부에서 일어나는 일로 나뉜다.

        **호출 전 (애플리케이션 컨트랙트)**

        1. Journal을 직접 구성하거나, 전달받은 Journal의 내용(이전 State Root, 배치 번호 등)을 기대값과 비교한다.

        2. `journalDigest = sha256(journal)`을 계산해 `verify`에 넘긴다.

        **Verifier 내부**

        1. `seal`의 처음 4byte(Selector)가 컨트랙트가 기대하는 Verifier 버전과 일치하는지 확인한다. Selector는 Groth16 Verifying Key와 Control Root 등 Verifier 파라미터의 해시에서 만든 값이다. 불일치하면 `SelectorMismatch`로 revert하며, 보통 Prover와 Verifier의 버전이 다를 때 발생한다.

        2. `ReceiptClaimLib.ok(imageId, journalDigest)`로 검증할 Claim을 재구성한다. `ok`는 종료 상태를 정상 종료(`Halted(0)`)로 고정한 Claim을 만든다. 따라서 On-chain `verify`도 Off-chain `receipt.verify`처럼 정상 종료한 실행의 증명만 받아들인다.

        3. Groth16 증명을 검증한다. 공개 입력은 Control Root, Claim Digest, BN254 Control ID다.

    On-chain Verifier는 STARK를 직접 검증하지 않고 Groth16으로 감싼 증명을 검증한다. 따라서 **검증 비용은 원래 계산의 크기와 관계없이 일정하다.** 이 글에서 확인한 Verifier Contracts 문서와 3.1.0 릴리스 노트에는 구체적인 Gas 수치가 없으며, 실제 비용은 배포 버전과 체인에 따라 다르다. RISC Zero는 여러 버전의 Verifier를 하나의 주소 뒤에 두고 Selector로 분기하는 `RiscZeroVerifierRouter`를 운영한다. Router를 거쳐 호출하면 Selector가 등록되지 않았거나 제거된 경우 `SelectorMismatch` 대신 `SelectorUnknown` 또는 `SelectorRemoved`로 revert한다. 3.1.0에서는 새 `RiscZeroGroth16Verifier`(Selector `0xe71e6620`)가 Router에 추가되었고, 이전 Verifier(2.2.0, 3.0.0)는 유예 기간 뒤 Emergency Stop 처리될 예정이다. 버전 이름이 3.1.0인 Groth16 Verifier는 두 개가 배포되었다는 점에 주의해야 한다. risc0-ethereum v3.1.0의 [`deployment.toml`](https://github.com/risc0/risc0-ethereum/blob/v3.1.0/contracts/deployment.toml)에는 Selector `0x8b38038c`와 `0xe71e6620`이 모두 `version = "3.1.0"`으로 기록되어 있다. [PR #748](https://github.com/risc0/risc0-ethereum/pull/748)에 따르면 `0x8b38038c`는 Trace에 실행이 하나만 담기도록 고치기 전의 첫 3.1 Circuit용으로 14개 체인에 배포된 것으로, `0xe71e6620`으로 대체되어 Emergency Stop 처리되었고, 릴리스된 어떤 `risc0-zkvm` 버전도 이 Selector용 Receipt를 만들지 않는다. 유효한 3.1.0 Verifier는 `0xe71e6620` 하나다. 컨트랙트 사용법은 [Verifier Contracts 문서](https://dev.risczero.com/api/blockchain-integration/contracts/verifier)에 있다.

### 트랜잭션 검증 로직의 동작 원리

Guest 프로그램에는 서명 확인, 잔고 확인, Nonce 검증 같은 로직이 들어간다. 검증이 하나라도 실패하면 `panic!`이나 `assert!`로 즉시 중단하게 만들 수 있다. 기본 설정(`ProverOpts`의 필드 `prove_guest_errors = false`)에서는 오류로 끝난 실행(panic 등)을 증명하지 않으며, 오류 종료 Receipt를 만들더라도 Off-chain `receipt.verify`와 On-chain `verify`를 통과하지 못한다. 모든 검증을 통과해야 마지막 단계의 `env::commit(new_state_root)`가 실행된다.

다만 정상 종료만으로는 충분하지 않다. `env::read`로 받은 입력은 비공개이고 Prover가 임의로 정할 수 있으므로, 실제와 다른 이전 상태나 임의로 만든 트랜잭션에서 출발한 실행도 정상 종료할 수 있다. 따라서 Guest는 새 State Root와 함께 이전 State Root와 트랜잭션 묶음의 해시를 Journal에 commit하고, Verifier(On-chain이라면 애플리케이션 컨트랙트)는 이 값이 자신이 알고 있는 현재 State Root, 게시된 트랜잭션 데이터와 일치하는지 확인해야 한다. 이 확인까지 거치면 Verifier는 Receipt만 확인하고도 **검증 로직을 모두 통과한 트랜잭션만 새 상태값을 만들었다**는 사실을 신뢰할 수 있다.

또한 Ethereum State Transition을 증명하는 Guest라면 두 경우를 구분해야 한다.

- **무효 트랜잭션:** 서명이나 Nonce가 틀린 트랜잭션은 블록을 무효로 만든다.

- **Revert된 트랜잭션:** 실행 중 Revert된 트랜잭션은 유효한 트랜잭션이며, 수수료와 Nonce 증가가 반영된다.

Revert된 트랜잭션까지 `panic!`으로 처리하면 유효한 블록을 증명할 수 없다.

### 장점과 한계

**장점**

- **기존 언어 사용:** 별도의 DSL(Domain-Specific Language, 도메인 전용 언어) 없이 Rust나 C, C++로 작성하므로 기존 라이브러리를 많이 재사용할 수 있다. 다만 zkVM 대상 환경의 제약이 있다. ECDSA(Elliptic Curve Digital Signature Algorithm) 서명 검증이나 해시 같은 암호 연산은 가속 Circuit([Precompile](https://dev.risczero.com/api/zkvm/precompiles))을 쓰도록 패치된 크레이트(`sha2`, `k256`, `tiny-keccak` 등)를 쓰는 것이 좋다.

- **규칙 변경에 강함:** 트랜잭션 형식이나 수수료 규칙이 바뀌어도 Guest 코드만 수정해 다시 컴파일하면 된다. 이때 Image ID가 바뀌므로 On-chain 컨트랙트 등 검증 쪽의 허용 Image ID도 갱신해야 업그레이드가 끝난다.

- **일반 제어문 사용:** 반복문과 조건문을 일반 코드처럼 쓸 수 있다. 증명 비용은 실제 실행한 Cycle 수와 Segment 수에 비례한다. 다만 각 Segment는 2의 거듭제곱 크기로 맞춰지므로, 비용은 Cycle 수에 정확히 비례하기보다 계단식으로 늘어난다.

- **일정한 On-chain 검증 비용:** On-chain에서는 `verify(seal, imageId, journalDigest)` 하나로, 계산 크기와 관계없이 일정한 비용으로 배치 실행 결과를 확정할 수 있다. 배치 데이터를 L1(Layer 1, 예: Ethereum)에 게시하는 비용은 별도다.

- **Trusted Setup 부담이 작음:** STARK 단계에는 Trusted Setup이 필요 없다. STARK-to-SNARK Circuit에는 필요하지만, Guest를 바꿀 때마다 다시 할 필요는 없다. 이 Circuit은 허용된 Recursion 프로그램 목록을 Control Root라는 공개 입력으로 받으므로, RISC-V Circuit이나 Recursion 프로그램이 바뀌어도 Control Root만 갱신하면 되고 새 Trusted Setup은 필요 없다. 다만 STARK-to-SNARK Circuit 자체가 바뀌면 새 Setup이 필요하다. 3.1.0에서 이 Circuit의 범위 검사와 Poseidon254 S-box를 수정하면서 새 Setup을 수행한 것이 그 예이며, 앞서 본 대로 이 새 Setup은 단일 참여자 Phase 2다.

**한계**

- **명령어 단위 오버헤드:** 명령어 단위로 증명하므로 특정 연산에 맞춘 전용 Circuit보다 증명 오버헤드가 크다.

- **높은 증명 비용:** 네이티브 실행보다 증명 비용이 크게 늘어난다. 그 배수는 작업량, 가속 Circuit 사용 여부, 장비에 따라 다르다. 블록 시간 안에 증명하려면 GPU 같은 고성능 장비를 직접 운영해야 한다. 2026년 11월 기준으로 Bonsai는 종료되고 Boundless는 증명 운영을 무기한 중단해, RISC Zero가 안내하는 원격 증명 경로는 없다.

- **작은 연산에서의 고정 비용:** 단순 송금처럼 연산량이 매우 적은 경우, Segment 최소 크기, Recursion, Wrapping 같은 고정 비용 때문에 해당 연산에 특화된 작은 Circuit보다 비용이 더 들 수 있다.

- **짧은 블록 시간에서의 한계:** 블록 시간이 매우 짧은 체인에서는 블록 시간 안에 증명을 생성하기 어렵다.

- **증명 시스템 자체의 취약점:** [3.1.0](https://github.com/risc0/risc0/releases/tag/v3.1.0)은 Groth16 Receipt의 Claim을 바꿔치기할 수 있는 결함을 포함해 여러 건전성 문제를 고쳤다. Poseidon254의 S-box가 $$x^8$$이어서 $$\gcd(8, r-1) \neq 1$$이라 순열이 아니었던 문제도 그중 하나로, $$x^7$$로 바뀌었다. RV32IM과 Recursion Circuit이 모두 재생성되어 Seal 형식, Circuit 식별자, Control Root가 바뀌었으므로, 이전 버전으로 만든 Receipt는 3.1.0에서 검증되지 않고 3.1.0 Receipt도 이전 버전에서 검증되지 않는다. 영향받는 크레이트의 이전 버전은 유예 후 crates.io에서 yank될 예정이다. zkVM을 쓰는 시스템은 Verifier 버전 관리와 Receipt 재생성 계획을 갖춰야 한다.