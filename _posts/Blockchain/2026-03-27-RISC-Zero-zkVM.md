---
title: "RISC Zero zkVM: Architecture and Proving Process"
date: 2026-03-27 00:01:00 +0900
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

이 글은 RISC Zero zkVM 2.0 이후 구조를 기준으로 하며, 수치는 2026년 11월 10일 공개된 risc0 3.1.0 릴리스와 그 시점의 공식 문서, 소스를 확인한 값이다. 버전에 따라 달라지는 항목은 본문에 표시했다. SDK 크레이트(`risc0-zkvm`) 버전과 회로 크레이트(`risc0-circuit-rv32im`) 버전은 따로 매겨지므로, 소스를 인용할 때는 어느 크레이트의 몇 번 버전인지 함께 적었다.

## RISC Zero zkVM

### What is RISC Zero?

RISC Zero는 범용 프로그램의 실행을 ZKP(Zero-Knowledge Proof, 영지식 증명)로 검증할 수 있게 만든 zkVM(Zero-Knowledge Virtual Machine, 영지식 가상 머신)이다. 특정 연산마다 Circuit(회로)을 설계하는 대신, **RISC-V(RV32IM) CPU 자체를 증명 가능한 가상 머신으로 구현한다.** 따라서 RISC Zero가 증명하는 것은 다음 사실이다.

> 주어진 프로그램이 특정 입력에 대해 RISC-V 명령어 규칙에 따라 정상적으로 실행되었다.

RISC Zero는 Verifiable Computing(검증 가능한 연산) 모델을 따른다. 무거운 계산은 Prover(증명자)가 맡고, Verifier(검증자)는 프로그램을 다시 실행하지 않고 증명만 확인한다.

```text
기존 실행 모델: Input → Program Execution → Output
RISC Zero:      Input → Program Execution (RISC-V) → Output + 증명(Receipt)
```

실행 결과로 생성되는 증명 객체를 **Receipt(영수증)**라고 하며, 개념적으로 두 부분으로 이루어진다.

- **Journal:** 프로그램이 공개한 출력(Public Output)
- **Seal:** Journal이 특정 프로그램(Image ID)의 올바른 실행으로 나왔음을 입증하는 암호학적 증명

증명 시스템은 STARK(Scalable Transparent Argument of Knowledge)를 기반으로 한다. RISC-V 회로의 STARK는 BabyBear 체($$p = 15 \cdot 2^{27} + 1$$) 위에서 동작하며, 기본 해시 함수는 Poseidon2다(`risc0-zkvm`의 `ProverOpts::default()`가 `hashfn = "poseidon2"`를 사용). 자세한 API는 [RISC Zero zkVM 문서](https://dev.risczero.com/api/zkvm/)에서 확인할 수 있다.

### Guest와 Host

RISC Zero zkVM은 증명 대상과 실행 관리자를 분리한다.

- **Guest:** 증명 대상 프로그램이다. Rust나 C, C++로 작성해 RISC-V ELF(Executable and Linkable Format) 바이너리로 컴파일한다.
- **Host:** Guest 실행을 관리한다. Guest에 입력을 전달하고 실행기를 구동하며, 증명(Receipt) 생성을 요청하고 검증한다. 증명은 로컬 장비나 원격 증명 서비스에서 생성한다.

2.0부터 zkVM은 User 모드와 Machine 모드를 구분한다. Guest 쪽 코드는 Kernel(커널)과 User 프로그램으로 나뉘며, 둘 다 RV32IM 명령어로 실행된다. User 프로그램이 입출력이나 가속 회로 호출을 위해 ECALL을 실행하면 zkVM이 이를 가로채 Kernel로 제어를 넘기고, Kernel이 해당 시스템 콜 핸들러로 분기한다. 핸들러는 필요하면 Host에 계산을 위임하고, 결과를 User 프로그램에 돌려준다. zkVM은 RISC-V의 CSR(Control and Status Register) 명령을 구현하지 않으며, 이 User와 Kernel 분리는 zkVM이 자체적으로 구현한 것이다. 메모리는 User 영역(`0x00010000`부터 약 3GB)과 Kernel 영역(`0xC0000000`부터 약 1GB)으로 나뉜다. 자세한 내용은 [zkVM Technical Specification](https://dev.risczero.com/api/zkvm/zkvm-specification)에 있다.

### End-to-End Proving Process

1. **Guest 프로그램 작성**

    Rust나 C++로 증명할 로직을 작성한다. 예를 들어 서명 검증, Merkle Proof 검증, State Transition(잔고, Nonce) 처리 같은 로직이다. **개발자는 Circuit을 정의하지 않는다.**

    - **입력:** `env::read()`로 Host에서 데이터를 받는다. 읽은 값은 기본적으로 비공개다.
    - **출력:** `env::commit()`으로 Journal에 기록한다. Journal은 증명과 함께 외부에 공개된다.

    `risc0-zkvm` 크레이트는 기본적으로 Guest에 Rust `core`만 제공하고 `std`는 제공하지 않는다. `std` 기능을 켜지 않으면 `#![no_std]`, `#![no_main]`, `risc0_zkvm::guest::entry!(main)` 세 줄이 필요하다. `std` 기능을 켜면 진입점이 자동으로 링크되므로 세 줄을 모두 생략할 수 있다([guest 모듈 문서](https://docs.rs/risc0-zkvm/latest/risc0_zkvm/guest/index.html)). 아래 예시는 `std` 기능을 켠 경우다.

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
    Rust/C++
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

    Image ID 계산에는 바이너리 중 실제로 적재되는 Page만 쓰이므로, 디버그 정보나 타임스탬프처럼 실행에 영향을 주지 않는 부분이 달라도 Image ID는 같다([FAQ](https://dev.risczero.com/faq)). Page 크기는 1,024byte이며, 이 값은 `risc0-binfmt` 크레이트의 `PAGE_BYTES` 상수로 정의되어 있다.

3. **zkVM 실행과 Execution Trace 생성**

    Host의 Executor가 RISC-V Interpreter로 바이너리를 실행하고, 각 명령어 처리 과정을 Execution Trace(실행 기록)로 기록한다. 개념적으로는 매 사이클마다 명령어 실행 전후의 상태 변화를 기록한다.

    - **실행 전 상태:** 실행 중인 명령어의 주소(PC), 읽은 레지스터 값, 읽은 Memory 값. 정수 레지스터는 x0~x31의 32개이며 x0은 항상 0이다.
    - **실행 명령어:** RV32IM 명령어 중 하나. ISA(Instruction Set Architecture) 명세 기준으로 RV32I가 40개, M 확장(곱셈, 나눗셈)이 8개로 총 48개이며, ADD, LW(Load), SW(Store), JAL(Jump) 등이 있다. zkVM은 여기에 ECALL을 이용해 영지식 계산에 필요한 기능(입출력, 가속 회로 호출)을 추가한다.
    - **실행 후 변화:** 결과가 쓰인 레지스터, Memory에 쓴 주소와 값, 다음 PC

    실제로는 매 사이클 전체 Memory와 레지스터를 복사하지 않는다. 그 사이클에서 읽고 쓴 값만 기록하고, Memory 일관성은 별도 논증으로 확인한다.

    Execution Trace는 행이 실행 사이클, 열이 PC, 명령어, 레지스터 값, Memory 접근 정보인 행렬 형태의 테이블이다. 아래는 x10 = 0x2000으로 가정한 개념 예시다.

    ```text
    | Cycle | PC    | 명령            | 레지스터 쓰기 | MemAddr | Mem 쓰기 |
    |-------|-------|-----------------|---------------|---------|----------|
    | 0     | 0x100 | addi x5, x0, 5  | x5 = 5        | -       | -        |
    | 1     | 0x104 | addi x6, x0, 3  | x6 = 3        | -       | -        |
    | 2     | 0x108 | add  x7, x5, x6 | x7 = 8        | -       | -        |
    | 3     | 0x10c | sw   x7, 0(x10) | -             | 0x2000  | 8        |
    ```

    RV32IM 명령어는 4byte이므로 분기가 없으면 PC는 4씩 증가한다. zkVM은 비정렬 주소 접근을 지원하지 않는다. 명령 코드는 4의 배수 주소에 있어야 하고, 워드(LW, SW) 접근은 4byte 정렬, 하프워드(LH, LHU, SH) 접근은 2byte 정렬이 필요하며, 바이트(LB, LBU, SB) 접근에는 정렬 제한이 없다(`risc0-circuit-rv32im`의 실행기 소스 기준). 정렬을 어기면 예외 핸들러 없이 실행이 종료된다.

    실제 사이클 수는 명령 수와 정확히 같지 않다. [Guest Optimization Guide](https://dev.risczero.com/api/zkvm/optimization) 기준으로 덧셈, 비교, 점프, 왼쪽 시프트, Load, Store는 한 사이클이고, 비트 연산(AND, OR, XOR), 나눗셈, 나머지, 오른쪽 시프트는 두 사이클이다. 부동소수점 명령은 구현되어 있지 않아 소프트웨어로 에뮬레이션되며 수십에서 백여 사이클이 든다. 또한 Segment 안에서 Page를 처음 읽을 때의 Page-In, 수정한 Page를 Segment 끝에 내보내는 Page-Out에도 사이클이 든다. Page-In은 Image ID에 대한 Merkle 포함 증명을 확인하는 해시 연산이므로 비용이 크며, 구체적인 사이클 수는 버전에 따라 다르다.

    **Continuations:** 실행이 길면 Trace 하나로 처리하기 어렵다. RISC Zero는 실행을 여러 Segment로 나누어 Trace를 만든다. Segment 크기는 `segment_limit_po2`로 2의 거듭제곱 단위로 정하며, 기본값은 $$2^{20}$$(1,048,576) 사이클이다(`risc0-circuit-rv32im`의 `execute::DEFAULT_SEGMENT_LIMIT_PO2 = 20`). 허용 범위는 `risc0-zkp`의 `MIN_CYCLES_PO2`(13)부터 `MAX_CYCLES_PO2`(24)까지다. 기본 Verifier 설정이 받아들이는 최대 Segment 크기는 이와 별도로 `risc0-zkvm`의 `DEFAULT_MAX_PO2`로 정해져 있으며, 3.x 기준 값은 22다. 각 Segment는 독립적으로 증명된 뒤 Recursion으로 합쳐진다.

4. **Arithmetization**

    Trace의 각 열을 다항식의 평가값(Evaluation)으로 보고, NTT(Number Theoretic Transform)로 그 값들을 지나는 다항식을 구한다. 검증 규칙은 두 종류의 제약으로 표현한다.

    - **전이 제약:** Trace의 인접한 두 행 사이의 제약
    - **경계 제약:** 첫 행과 마지막 행에 대한 제약

    이 표현 방식을 **AIR(Algebraic Intermediate Representation)**라고 한다. Guest가 정상 실행되었다면 모든 제약 다항식이 Trace의 정의역에서 0이 된다.

    - **PC 제약:** 분기나 점프가 없으면 $$PC_{n+1} = PC_n + 4$$, 분기나 점프가 실행되면 $$PC_{n+1} = \text{target}_n$$
    - **레지스터 제약:** $$\text{rd}_{n+1} = \mathrm{ALU}(\text{rs1}_n, \text{rs2}_n, \text{imm}) \bmod 2^{32}$$이며, $$\text{rd} = x_0$$이면 값이 바뀌지 않는다.
    - **Memory 제약:** 어떤 주소에서 읽은 값은 그 주소에 마지막으로 쓴 값과 같아야 한다. Memory 접근 기록을 주소와 시간 순으로 정렬한 Permutation 논증 등으로 확인한다.
    - **디코딩과 범위 제약:** 명령 워드가 유효한 RV32IM 명령으로 디코딩되고, 레지스터와 Memory 값이 32bit 범위 안에 있어야 한다.

    경계 제약이 빠지면 어떤 일이 생기는지 보여주는 사례가 있다. 3.1.0 이전의 RV32IM 회로는 Trace 안에 실행 시작 행(`isFirstCycle`)이 몇 개 있어도 되도록 두었다. Memory 논증은 주소, 시각, 값만으로 접근을 식별하므로, 악의적인 Prover가 한 Trace에 두 실행을 넣고 한 실행의 읽기가 다른 실행의 쓰기 값을 돌려받게 만들 수 있었다. 3.1.0은 Trace 길이를 공개 값(`totalCycles`)으로 두고, 모든 시작 행 직전 행이 Trace의 마지막 행이어야 한다는 제약을 추가해 Trace가 정확히 하나의 실행만 담도록 고쳤다([v3.1.0 릴리스 노트](https://github.com/risc0/risc0/releases/tag/v3.1.0)).

5. **STARK 증명, Recursion, Wrapping**

    - **FRI:** Prover는 Trace 다항식에 커밋한 뒤, FRI(Fast Reed-Solomon Interactive Oracle Proof of Proximity)로 커밋한 값이 낮은 차수의 다항식에 가깝다는 것을 증명한다. FRI는 다항식을 반복해서 접어(Folding) 차수를 줄인다. 일반적인 설명에서는 한 번에 차수를 절반으로 줄이며, RISC Zero는 한 번에 16배씩 접어(`FRI_FOLD = 16`) 차수가 `FRI_MIN_DEGREE`(256)에 도달할 때까지 반복한다. Reed-Solomon 확장률의 역수(`INV_RATE`)는 4다. Verifier는 무작위 위치를 50회(`QUERIES`) 조회하며, 소스 주석은 이 설정이 97bit의 추정(Conjectured) 보안 수준을 목표로 한다고 밝힌다. 상수는 [risc0-zkp 소스](https://docs.rs/risc0-zkp/3.0.1/src/risc0_zkp/lib.rs.html)에서 확인할 수 있다. `DEFAULT_MAX_PO2` 문서에 따르면 Segment 크기가 커질수록 보안 수준은 po2당 약 1bit씩 낮아진다.
    - **병렬 증명:** 각 Segment의 STARK 증명은 독립적이므로 GPU 가속이나 Boundless 같은 분산 증명 시장에서 병렬로 생성할 수 있다. RISC Zero가 운영하던 중앙화된 증명 서비스 Bonsai는 2025년 12월 기준으로 종료되었고, 원격 증명은 Boundless로 대체되었다([Boundless 문서](https://docs.boundless.network/developers/tutorials/bonsai)).
    - **Recursion:** RISC Zero zkVM은 세 가지 회로로 구성된다. RISC-V 실행을 증명하는 RISC-V Circuit, STARK 증명 검증에 특화된 별도의 STARK 회로인 Recursion Circuit, 그리고 Recursion Circuit의 증명을 검증하는 R1CS 회로인 STARK-to-SNARK Circuit이다. 공식 순서는 다음과 같다. 각 Segment 증명을 `lift`로 Recursion Circuit용 증명(Succinct Receipt)으로 변환하고, `join`으로 둘씩 합치는 과정을 하나가 남을 때까지 반복한다. 마지막 Succinct Receipt는 `identity_p254`를 거친다. `identity_p254`는 Poseidon254 해시 함수를 쓰는 Recursion Prover로 증명을 다시 검증하는 단계로, Groth16 Prover 실행 직전에 수행된다. Succinct Receipt는 실행 길이와 무관한 상수 크기이며 약 200kB다. 자세한 내용은 [Recursive Proving 문서](https://dev.risczero.com/api/recursion)에 있다.
    - **Groth16 Wrapping:** STARK 증명은 커서 Ethereum On-chain 검증 비용이 높다. 그래서 Succinct Receipt를 검증하는 로직을 Circom으로 작성한 Groth16 Circuit(`stark_verify` 회로)으로 구현해 Groth16 증명으로 감싼다(`compress`). 증명은 BN254 곡선의 G1 두 점(각 64byte)과 G2 한 점(128byte)으로 256byte이며, On-chain Seal에는 앞에 4byte Selector가 붙어 260byte가 된다. 이 Wrapping Circuit에는 Trusted Setup이 필요하다. RISC Zero는 2024년에 PSE(Privacy and Scaling Explorations)의 p0tion 도구로 공개 세레머니를 진행했으며, L2BEAT의 ZK Catalog는 Phase 2 참여자를 238명으로 집계한다. 3.1.0에서는 회로 수정에 따라 새 Trusted Setup(2026-10-20.1)과 새 Verifying Key를 적용했다. 검증 절차는 [Trusted Setup Security 문서](https://dev.risczero.com/api/trusted-setup-ceremony)에 있다.

    최종 Receipt는 다음 구조다. Seal은 최상위 필드가 아니라 `inner` 안에 들어 있다.

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

    Verifier는 **기대하는 Image ID, Seal, Journal**을 대조한다. Image ID는 Receipt에 별도 필드로 들어 있지 않다. 증명이 담은 Claim(주장)의 시작 상태 값이 Verifier가 넘긴 Image ID와 같은지 확인한다.

    - **Off-chain 검증:** RISC Zero Rust SDK의 `receipt.verify(image_id)`를 호출한다. 서버, 클라이언트 앱, 다른 체인의 노드 등에서 쓴다. SDK는 InnerReceipt 종류에 맞는 검증 알고리즘을 수행하고, 프로그램이 정상 종료(`Halted(0)`)했는지와 Journal이 증명된 출력과 일치하는지도 확인한다. 정상 종료 여부와 Image ID를 확인하지 않고 Seal이 Claim과 Journal을 보증하는지만 확인하는 `verify_integrity_with_context`도 별도로 있다. 이 함수를 쓰면 호출자가 Claim의 필드를 직접 확인해야 한다.

    - **On-chain 검증:** Solidity Verifier 컨트랙트의 `verify(bytes seal, bytes32 imageId, bytes32 journalDigest)`를 호출한다. `journalDigest`는 `sha256(journal)`이다. `seal`은 4byte Selector 뒤에 ABI 인코딩된 Groth16 증명이 붙은 형태다.
        1. `seal`의 처음 4byte(Selector)가 컨트랙트가 기대하는 Verifier 버전과 일치하는지 확인한다. Selector는 Groth16 Verifying Key와 Control Root 등 Verifier 파라미터의 해시에서 만든 값이다. 불일치하면 `SelectorMismatch`로 revert하며, 보통 Prover와 Verifier의 버전이 다를 때 발생한다.
        2. `imageId`와 `journalDigest`로 검증할 Claim을 재구성한다.
        3. Groth16 증명을 검증한다. 공개 입력은 Control Root, Claim Digest, BN254 Control ID다.
        4. 애플리케이션이 Journal 내용(이전 State Root, 배치 번호 등)을 기대값과 비교한다.

    On-chain Verifier는 STARK를 직접 검증하지 않고 Groth16으로 감싼 증명을 검증한다. 따라서 **검증 비용은 원래 계산의 크기와 관계없이 일정하다.** 구체적인 Gas 수치는 공식 문서에 명시되어 있지 않으며 배포 버전과 체인에 따라 다르다. RISC Zero는 여러 버전의 Verifier를 하나의 주소 뒤에 두고 Selector로 분기하는 `RiscZeroVerifierRouter`를 운영한다. 3.1.0에서는 새 `RiscZeroGroth16Verifier`(Selector `0xe71e6620`)가 Router에 추가되었고, 이전 Verifier는 유예 기간 뒤 Emergency Stop 처리될 예정이다. 컨트랙트 사용법은 [Verifier Contracts 문서](https://dev.risczero.com/api/blockchain-integration/contracts/verifier)에 있다.

### 트랜잭션 검증 로직의 동작 원리

Guest 프로그램에는 서명 확인, 잔고 확인, Nonce 검증 같은 로직이 들어간다. 검증이 하나라도 실패하면 `panic!`이나 `assert!`로 즉시 중단하게 만들 수 있다. 중단된 실행으로는 정상 종료를 증명하는 Receipt를 얻을 수 없다. `receipt.verify`가 정상 종료만 받아들이기 때문이다. 모든 검증을 통과해야 마지막 단계의 `env::commit(new_state_root)`가 실행된다. 따라서 Verifier는 Receipt만 확인하고도 **검증 로직을 모두 통과한 트랜잭션만 새 상태값을 만들었다**는 사실을 신뢰할 수 있다.

다만 Ethereum State Transition을 증명하는 Guest라면 두 경우를 구분해야 한다.

- **무효 트랜잭션:** 서명이나 Nonce가 틀린 트랜잭션은 블록을 무효로 만든다.
- **Revert된 트랜잭션:** 실행 중 Revert된 트랜잭션은 유효한 트랜잭션이며, 수수료와 Nonce 증가가 반영된다.

Revert된 트랜잭션까지 `panic!`으로 처리하면 유효한 블록을 증명할 수 없다.

### 장점과 한계

**장점**

- **기존 언어 사용:** 별도의 DSL(Domain-Specific Language, 도메인 전용 언어) 없이 Rust나 C++로 작성하므로 기존 라이브러리를 많이 재사용할 수 있다. 다만 zkVM 대상 환경의 제약이 있다. ECDSA나 해시 같은 암호 연산은 가속 Circuit(Precompile)을 쓰도록 패치된 크레이트(`sha2`, `k256`, `tiny-keccak` 등)를 쓰는 것이 좋다([Precompiles 문서](https://dev.risczero.com/api/zkvm/precompiles)).
- **규칙 변경에 강함:** 트랜잭션 형식이나 수수료 규칙이 바뀌어도 Guest 코드만 수정해 다시 컴파일하면 된다. 이때 Image ID가 바뀌므로 On-chain 컨트랙트 등 검증 쪽의 허용 Image ID도 갱신해야 업그레이드가 끝난다.
- **일반 제어문 사용:** 반복문과 조건문을 일반 코드처럼 쓸 수 있다. 증명 비용은 실제 실행한 사이클 수에 비례한다.
- **일정한 On-chain 검증 비용:** On-chain에서는 `verify(seal, imageId, journalDigest)` 하나로, 계산 크기와 관계없이 일정한 비용으로 배치 실행 결과를 확정할 수 있다. 배치 데이터를 L1에 게시하는 비용은 별도다.
- **Trusted Setup 부담이 작음:** STARK 단계에는 Trusted Setup이 필요 없다. Groth16 Wrapping 단계에는 필요하지만, Guest를 바꿀 때마다 다시 할 필요는 없다. Groth16 Circuit은 허용된 Recursion 프로그램 목록을 Control Root라는 하나의 값으로 입력받으므로, RISC-V 회로나 Recursion 프로그램이 바뀌어도 Control Root만 갱신하면 되고 새 Trusted Setup은 필요 없다. 다만 Groth16 Circuit 자체가 바뀌면 새 Setup이 필요하다. 3.1.0에서 `stark_verify` 회로의 범위 검사와 Poseidon254 S-box를 수정하면서 새 Setup을 수행한 것이 그 예다.

**한계**

- **명령어 단위 오버헤드:** 명령어 단위로 증명하므로 특정 연산에 맞춘 전용 Circuit보다 증명 오버헤드가 크다.
- **높은 증명 비용:** 네이티브 실행보다 증명 비용이 크게 늘어난다. 그 배수는 작업량, 가속 Circuit 사용 여부, 장비에 따라 다르다. 블록 시간 안에 증명하려면 GPU 같은 고성능 장비나 Boundless 같은 증명 시장이 필요하다.
- **작은 연산에서의 고정 비용:** 단순 송금처럼 연산량이 매우 적은 경우, Segment 최소 크기, Recursion, Wrapping 같은 고정 비용 때문에 해당 연산에 특화된 작은 Circuit보다 비용이 더 들 수 있다.
- **짧은 블록 시간에서의 한계:** 블록 시간이 매우 짧은 체인(예: 1초 미만)에서는 블록 시간 안에 증명을 생성하기 어렵다.
- **증명 시스템 자체의 취약점:** 3.1.0은 Groth16 Receipt의 Claim을 바꿔치기할 수 있는 결함을 포함해 여러 건전성 문제를 고쳤고, 이전 버전으로 만든 Receipt는 3.1.0에서 검증되지 않는다. zkVM을 쓰는 시스템은 Verifier 버전 관리와 Receipt 재생성 계획을 갖춰야 한다.

### 참고 자료

- [RISC Zero zkVM 문서](https://dev.risczero.com/api/zkvm/)
- [zkVM Technical Specification](https://dev.risczero.com/api/zkvm/zkvm-specification)
- [Guest Optimization Guide](https://dev.risczero.com/api/zkvm/optimization)
- [Recursive Proving](https://dev.risczero.com/api/recursion)
- [Key Terminology](https://dev.risczero.com/terminology)
- [Trusted Setup Security](https://dev.risczero.com/api/trusted-setup-ceremony)
- [Verifier Contracts](https://dev.risczero.com/api/blockchain-integration/contracts/verifier)
- [RiscZeroGroth16Verifier.sol](https://github.com/risc0/risc0-ethereum/blob/release-3.0/contracts/src/groth16/RiscZeroGroth16Verifier.sol)
- [risc0-zkp 상수 정의](https://docs.rs/risc0-zkp/3.0.1/src/risc0_zkp/lib.rs.html)
- [risc0-zkvm Receipt](https://docs.rs/risc0-zkvm/latest/risc0_zkvm/struct.Receipt.html)
- [risc0-zkvm DEFAULT_MAX_PO2](https://docs.rs/risc0-zkvm/latest/risc0_zkvm/constant.DEFAULT_MAX_PO2.html)
- [risc0-circuit-rv32im](https://docs.rs/risc0-circuit-rv32im/latest/risc0_circuit_rv32im/)
- [risc0-binfmt 소스](https://docs.rs/risc0-binfmt/latest/src/risc0_binfmt/lib.rs.html)
- [risc0 v3.1.0 릴리스 노트](https://github.com/risc0/risc0/releases/tag/v3.1.0)
- [Boundless: What happened to Bonsai?](https://docs.boundless.network/developers/tutorials/bonsai)
- [L2BEAT ZK Catalog: RISC Zero](https://ethbeat.org/zk-catalog/risc0)