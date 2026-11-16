---
title: "Circuit-Based SNARK Tools: gnark and ZoKrates"
date: 2026-03-27 00:02:00 +0900
categories: [Blockchain]
tags: [Blockchain, ZKP, zk-SNARK, gnark, ZoKrates]
math: true
media_subpath: /assets/img/posts/zkevm/
---

>
> 1. [EVM, zkEVM, zkVM: Execution and Verification]({% post_url Blockchain/2026-03-27-zkEVM_zkVM %})
> 2. [RISC Zero zkVM: Architecture and Proving Process]({% post_url Blockchain/2026-03-27-RISC-Zero-zkVM %})
> 3. [SP1 zkVM: Architecture and Proving Process]({% post_url Blockchain/2026-03-27-SP1-zkVM %})
> 4. **Circuit-Based SNARK Tools: gnark and ZoKrates**
>

## Background

2편과 3편에서 본 zkVM(Zero-Knowledge Virtual Machine)은 RISC-V 같은 범용 ISA(Instruction Set Architecture, 명령어 집합)의 실행을 증명한다. Rust로 작성한 프로그램을 그대로 증명할 수 있다는 장점이 있다. 대신 프로그램이 실행한 모든 명령어를 Execution Trace(실행 기록)로 기록해 증명하므로 Trace가 길다. 또 On-chain에서 검증할 작은 증명을 얻으려면 STARK(Scalable Transparent Argument of Knowledge) 증명을 Recursion(재귀 증명, 증명을 검증하는 계산을 다시 증명하는 기법)으로 합친 뒤 SNARK(Succinct Non-Interactive Argument of Knowledge)로 한 번 더 감싸야 한다.

Circuit 기반 SNARK 도구는 반대로 접근한다. **개발자가 증명할 로직을 직접 Circuit(회로)으로 작성한다.** CPU(Central Processing Unit)를 흉내 내는 계층이 없으므로 증명에 필요한 관계만 제약으로 표현하고, Groth16이나 PLONK로 바로 작은 증명을 만든다. 대신 로직이 바뀌면 Circuit을 다시 설계해야 한다.

SNARK는 증명이 짧고 검증이 빠른 비대화형 증명이고, 여기에 Zero-Knowledge를 더한 것이 zk-SNARK다. 증명을 만드는 쪽을 Prover(증명자), 확인하는 쪽을 Verifier(검증자)라고 한다. 자세한 내용은 [ZKP 글]({% post_url Cryptography/2026-04-06-zkp %})을 참고한다.

Circuit 기반 도구는 다음과 같은 곳에 쓰인다.

- **특정 기능 Circuit:** Rollup, Bridge, Light Client처럼 검증할 로직이 정해진 경우
- **zkVM 증명의 Wrapping:** SP1은 마지막 Groth16 또는 PLONK Wrapping 단계를 gnark로 구현한다(3편).
- **zkEVM의 Outer Proof:** ZK(Zero-Knowledge) Rollup인 Linea의 [Prover](https://docs.linea.build/protocol/architecture/prover)는 Corset으로 만든 zkEVM(Zero-Knowledge Ethereum Virtual Machine) 제약을 자체 증명 체계로 증명한 뒤(Inner Proof), gnark로 L1(Layer 1) 검증용 Outer Proof를 만든다.

이 글은 gnark v0.16.5(2026-11-09 릴리스)와 ZoKrates 최신 릴리스 0.8.8(2023-11-10)을 기준으로 한다. ZoKrates develop 브랜치의 마지막 커밋은 2024-04-15이며, develop 브랜치의 `zokrates_cli` Cargo 버전 문자열은 0.8.7로 남아 있다.

## Core Concepts

### Circuit and Constraint System

Circuit은 계산을 유한체 위의 덧셈과 곱셈으로 표현한 제약식의 집합이다. Circuit은 입력에 따라 실행 경로를 바꿀 수 없다. 그래서 **컴파일 시점에 모든 제약 수가 정적으로 확정되어야 하고**, 조건문과 반복문도 고정된 제약으로 바꿔야 한다. 제약을 표현하는 형식은 두 가지가 많이 쓰인다.

- **R1CS(Rank-1 Constraint System):** Groth16이 쓰는 형식이다. 각 제약은 $$A \times B = C$$ 꼴이며, $$A, B, C$$는 Circuit의 모든 변수 $$v_i$$(입력, 중간값, 상수 $$v_0 = 1$$ 포함)의 Linear Combination(선형 결합)이다.

    $$
    \begin{aligned}
    A &= \alpha_0 v_0 + \alpha_1 v_1 + \cdots + \alpha_n v_n \\
    B &= \beta_0 v_0 + \beta_1 v_1 + \cdots + \beta_n v_n \\
    C &= \gamma_0 v_0 + \gamma_1 v_1 + \cdots + \gamma_n v_n
    \end{aligned}
    $$

    제약 하나는 곱셈 하나($$A \times B$$)만 담는다. $$A, B, C$$ 안의 덧셈은 별도 제약을 만들지 않는다. 그래서 복잡한 수식은 곱셈 하나씩을 포함한 식들로 나누어야 한다. **덧셈을 최대한 곱셈 제약의 괄호 안으로 합쳐 곱셈 수를 줄이는 것**이 R1CS 최적화의 핵심이다.

- **Plonkish 제약:** PLONK(Permutations over Lagrange-bases for Oecumenical Noninteractive arguments of Knowledge)가 쓰는 형식이다. 각 Gate(게이트)는 Wire 값 $$a, b, c$$에 대해 $$q_L a + q_R b + q_O c + q_M ab + q_C = 0$$을 만족해야 하고, Gate 사이의 Wire 연결은 별도의 Copy Constraint(복사 제약)로 확인한다. Gate 하나가 받는 입력 수가 제한되어 있으므로 덧셈에도 Gate가 필요하다.

### Witness

Witness(증인)는 Circuit의 모든 변수에 대입하는 값의 집합이다. 입력값과 중간 계산값을 모두 포함한다. 입력은 두 종류로 나뉜다.

- **Public Input(공개 입력):** Verifier도 아는 값
- **Secret Input(비공개 입력):** Prover만 알고 증명에 공개되지 않는 값

Prover는 모든 제약을 만족하는 Witness를 알고 있다는 사실을, Secret Input을 공개하지 않고 증명한다. Verifier는 로직을 다시 실행하지 않고 증명과 Public Input만으로 이를 확인한다.

### Setup

Groth16과 PLONK는 Prover와 Verifier가 함께 쓰는 공개 파라미터인 SRS(Structured Reference String)가 필요하다. SRS는 비밀 난수로 만든다. 이 난수를 Toxic Waste라고 부르며, Toxic Waste를 아는 사람은 증명을 위조할 수 있으므로 반드시 파기해야 한다. 이렇게 비밀 값의 파기를 믿어야 하는 Setup을 Trusted Setup(신뢰 설정)이라고 한다. 대규모 프로젝트에서는 MPC(Multi-Party Computation) Ceremony(세레머니)로 수행하며, 참여자 중 한 명만 정직하게 파기해도 안전하다.

- **Circuit별 Setup:** Groth16은 Circuit마다 Trusted Setup이 필요하다. Circuit이 조금만 바뀌어도 Setup을 다시 해야 한다.
- **Universal SRS:** PLONK는 KZG(Kate-Zaverucha-Goldberg) Polynomial Commitment 기반의 Universal SRS를 쓴다. Ceremony를 한 번 하면 그 SRS가 지원하는 최대 Circuit 크기 이하의 Circuit에 재사용할 수 있다. 다만 Circuit마다 Proving Key와 Verifying Key를 만드는 전처리는 필요하다. 이 전처리에는 새로운 비밀 값이 없으므로 Ceremony가 필요 없다.

### Pairing

Pairing(페어링) $$e: \mathbb{G}_1 \times \mathbb{G}_2 \to \mathbb{G}_T$$는 $$e(aP, bQ) = e(P, Q)^{ab}$$를 만족하는 함수다. Verifier는 이 성질로 값을 직접 보지 않고도 타원곡선 점 안에 숨은 값들의 곱셈 관계를 확인한다. Groth16의 검증과 KZG Commitment의 확인이 모두 Pairing 등식이다.

BN254 기준으로 $$\mathbb{G}_1$$은 기저체 $$\mathbb{F}_p$$ 위 곡선의 점으로 이루어진 군이고, 점 하나는 비압축 64바이트다. $$\mathbb{G}_2$$는 확장체 $$\mathbb{F}_{p^2}$$ 위 곡선의 점으로 이루어진 군이고, 점 하나는 비압축 128바이트다. Circuit의 산술은 이와 다른 스칼라체 $$\mathbb{F}_r$$ 위에서 이루어진다.

### Proving Cost

제약 수는 연산 종류에 따라 크게 다르다.

- **덧셈과 상수 곱셈:** R1CS에서는 Linear Combination 안에 들어가므로 추가 제약이 거의 없다.
- **변수끼리의 곱셈:** 제약 하나가 된다.
- **해시와 비트 연산:** 값을 비트로 분해하고 비트마다 0 또는 1이라는 제약을 걸어야 하므로, 코드 한 줄이 많은 제약으로 늘어난다. Keccak처럼 비트 연산 위주인 해시가 특히 비싸다.
- **다른 체 위의 연산:** ECDSA(Elliptic Curve Digital Signature Algorithm) 서명 검증에 쓰는 secp256k1 곡선은 BN254 스칼라체 $$\mathbb{F}_r$$와 다른 체 위에서 정의된다. 그래서 gnark `std/signature/ecdsa`는 체 에뮬레이션(`std/math/emulated`)으로 큰 수 연산을 여러 원소로 나누어 구현하며, 제약이 많이 든다.

증명 생성에서 무거운 연산은 두 가지다.

- **MSM(Multi-Scalar Multiplication):** 타원곡선 점들의 Linear Combination $$P = \sum_{i} r_i G_i$$를 계산한다. 증명 생성에서 가장 무거운 연산 중 하나다.
- **FFT(Fast Fourier Transform):** 다항식의 계수 표현과 점 표현을 변환한다. 유한체 위에서는 NTT(Number Theoretic Transform)로 계산한다.

증명 생성의 CPU와 Memory 부담은 Prover가 지고, Verifier는 작은 증명만 확인한다. 따라서 **Circuit 기반 도구의 한계는 검증 비용이 아니라 증명 생성 비용과 Circuit을 직접 설계하는 부담에 있다.** 이는 gnark와 ZoKrates 모두에 해당한다.

### On-chain Verification

gnark와 ZoKrates 모두 Verifying Key로 Solidity Verifier 컨트랙트를 자동 생성한다. BN254 Verifier는 BN254 Precompile인 `ecAdd`(0x06), `ecMul`(0x07), `ecPairing`(0x08)을 호출한다. gnark의 BLS12-381 Verifier는 대신 [EIP-2537](https://eips.ethereum.org/EIPS/eip-2537) Precompile(0x0c `BLS12_G1MSM`, 0x0f `BLS12_PAIRING_CHECK`)을 쓴다. 아래 비용은 BN254 기준이다. EIP(Ethereum Improvement Proposal) 중 [EIP-1108](https://eips.ethereum.org/EIPS/eip-1108) 기준으로 `ecPairing` 호출 비용은 $$34{,}000k + 45{,}000$$ Gas다($$k$$는 Pairing 쌍의 수).

Groth16의 검증식은 다음과 같다. $$A, C \in \mathbb{G}_1$$, $$B \in \mathbb{G}_2$$는 증명이고, $$L$$은 Public Input으로 계산한 $$\mathbb{G}_1$$ 점이다.

$$
e(A, B) = e(\alpha, \beta)\, e(L, \gamma)\, e(C, \delta)
$$

Off-chain에서는 $$e(\alpha, \beta)$$를 Verifying Key에 미리 계산해 두면 Pairing 3번으로 끝난다. 하지만 EVM(Ethereum Virtual Machine)의 `ecPairing` Precompile은 $$\prod_i e(P_i, Q_i) = 1$$인지 여부만 반환하고, $$\mathbb{G}_T$$ 원소를 입력으로 받지 않는다. 그래서 미리 계산한 $$e(\alpha, \beta)$$를 쓸 수 없고, 모든 항을 한쪽으로 옮긴 4쌍을 한 번에 넘긴다.

$$
e(A, B)\, e(-\alpha, \beta)\, e(-L, \gamma)\, e(-C, \delta) = 1
$$

Groth16 Verifier의 비용은 다음과 같다.

- **Pairing 4쌍 확인:** 입력 768바이트(4 × (64 + 128)), $$34{,}000 \times 4 + 45{,}000 = 181{,}000$$ Gas
- **Public Input 하나당:** $$L$$을 계산하는 `ecMul`과 `ecAdd`로 약 6~7천 Gas 추가
- **총비용:** 위 두 항목에 Calldata와 컨트랙트 실행 비용이 더해진다. 즉 181,000 Gas + Public Input 수 × 약 6~7천 Gas + 기타 비용이다.

## gnark

[gnark](https://github.com/Consensys/gnark)는 Consensys가 개발한 Go 기반 zk-SNARK 라이브러리다. Go의 struct와 전용 API(`frontend.API`)로 Circuit을 설계하므로, Go 안에 내장된 DSL(Domain-Specific Language, 도메인 전용 언어)로 제약을 작성하는 방식이다.

gnark는 Frontend(Circuit 정의)와 Backend(증명 생성)를 분리한다. 같은 Circuit 코드를 Builder(빌더)에 따라 두 가지 제약 시스템 중 하나로 컴파일한다.
- **R1CS:** Groth16용 (`r1cs.NewBuilder`)
- **SCS(Sparse Constraint System):** PLONK용 Plonkish 제약 (`scs.NewBuilder`)

지원 곡선은 BN254, BLS12-381, BLS12-377, BW6-761이다. Solidity Verifier는 주로 BN254를 대상으로 한다. gnark v0.15.0(2026년 5월)부터는 BLS12-381 Solidity Verifier도 생성한다. 이 Verifier는 2025년 5월 7일 메인넷에서 활성화된 [Pectra](https://ethereum.org/roadmap/pectra/) 업그레이드의 [EIP-2537](https://eips.ethereum.org/EIPS/eip-2537) BLS12-381 Precompile을 쓴다.

### Workflow

1. **Circuit 설계**

    Circuit은 Go `struct`로 정의한다. 필드는 기본적으로 Secret Input이며, Public Input으로 쓸 필드에 `gnark:",public"`을 붙인다(쉼표 필요). Rollup의 State Transition을 증명한다면 다음과 같이 나뉜다.

    - **Public Input:** 이전 State Root, 새 State Root, 블록 헤더 정보
    - **Secret Input:** 트랜잭션, 각 트랜잭션의 Merkle Proof, 서명

    ```go
    type CubicCircuit struct {
        // X는 Secret Input (태그의 "x"는 이름이며, 공개 지정이 없으므로 Secret)
        X frontend.Variable `gnark:"x"`
        // Y는 Public Input (Verifier도 아는 값)
        Y frontend.Variable `gnark:",public"`
    }
    ```

2. **제약 작성(`Define`)**

    `Define(api frontend.API) error` 메서드에서 제약을 작성한다. `api.Add`, `api.Mul` 같은 산술 연산으로 $$x^3 + x + 5 = y$$ 같은 방정식을 구성하거나, `gnark/std`의 Gadget(가젯, 미리 만들어 둔 Circuit 부품)으로 Merkle Proof 검증과 EdDSA(Edwards-curve Digital Signature Algorithm), ECDSA 서명 검증을 Circuit에 포함한다.

    ```go
    func (circuit *CubicCircuit) Define(api frontend.API) error {
        // x³ + x + 5 == y 를 증명하는 Circuit
        x2 := api.Mul(circuit.X, circuit.X)  // x²
        x3 := api.Mul(x2, circuit.X)         // x³
        sum := api.Add(x3, circuit.X)        // x³ + x
        result := api.Add(sum, 5)            // x³ + x + 5

        // 제약: result == Y
        api.AssertIsEqual(result, circuit.Y)
        return nil
    }
    ```

    Circuit은 실행 경로를 바꿀 수 없으므로, 조건문과 반복문은 다음과 같이 작성한다.

    - **조건문:** `api.Select(condition, a, b)`로 대체한다. `condition`이 1이면 `a`, 0이면 `b`를 고르며, 내부적으로 `b + condition × (a − b)`로 계산해 곱셈 1개로 삽입된다. R1CS Builder는 처음부터 Select 안에서 `condition`이 0 또는 1이라는 Boolean 제약을 함께 추가했다. SCS Builder는 이 검사가 없어 변수 `condition`에 임의 값을 넣을 수 있었고, gnark v0.11.0(2024년 9월, [#1247](https://github.com/Consensys/gnark/pull/1247))에서 추가되었다. 두 Builder 모두 `condition`이 이미 Boolean으로 확인된 값이면 이 제약은 다시 추가되지 않는다.
    - **반복문:** `Define()` 안의 Go for 문은 Circuit을 컴파일할 때 실행되어 반복마다 제약을 생성한다. 이를 Unrolling(펼치기)이라고 한다. 반복 횟수가 컴파일 시점에 정해져 있으면 for 문을 그대로 쓰면 된다. 입력에 따라 길이가 바뀌는 반복은 고정 상한과 Selector(선택자)로 바꾼다([Circuit-Based Proof](#circuit-based-proof-gnark-zokrates) 참고).
    - **입력 개수:** 입력 개수가 정해진 경우에는 고정 크기 배열을 struct 필드로 정의한다.

3. **컴파일**

    `frontend.Compile()`은 지정한 곡선의 스칼라체(예: `ecc.BN254.ScalarField()`) 위에서 Go 코드를 R1CS나 Plonkish 형식으로 변환한다.

    gnark에는 수식을 곱셈 단위로 분해하는 별도 단계가 없다. `Define`이 실행되는 동안 각 API 호출이 바로 제약을 기록한다.
    - **R1CS Builder:** `api.Mul`이 바로 R1CS 제약을 추가하고, `api.Add`는 제약을 만들지 않고 Linear Combination으로만 남는다. 위 예제에서 두 번의 덧셈은 `AssertIsEqual`이 추가하는 등식 제약의 Linear Combination 안에 들어간다.
    - **SCS Builder:** `api.Add`도 덧셈을 2항씩 나누어 내부 변수와 Gate를 바로 만든다.

4. **Setup**

    - **Groth16:** `groth16.Setup(ccs)`로 Circuit별 Proving Key(pk)와 Verifying Key(vk)를 만든다. Circuit별 Trusted Setup이다. 이 함수는 한 프로세스가 Toxic Waste가 되는 난수를 직접 생성하므로, 그 난수가 유출되면 증명을 위조할 수 있다. gnark 문서는 운영 환경에서 MPC Setup(`backend/groth16/bn254/mpcsetup` 패키지) 또는 Circuit별 Setup이 결정적인 PLONK 사용을 권한다.
    - **PLONK:** `plonk.Setup(ccs, srs, srsLagrange)`처럼 Universal SRS를 인자로 넘기고, 이 Circuit의 Proving Key와 Verifying Key를 만든다. Circuit 크기가 SRS가 지원하는 최대 크기 이하이면 새 Ceremony 없이 같은 SRS를 재사용한다. Circuit을 바꾸면 키는 다시 생성해야 한다.

    두 방식의 차이는 다음과 같다(BN254 기준).

    | 구분 | Groth16 | PLONK |
    |---|---|---|
    | Setup | Circuit별 Trusted Setup | Universal SRS 재사용 |
    | 증명 크기 | $$\mathbb{G}_1$$ 점 2개 + $$\mathbb{G}_2$$ 점 1개 (비압축 256바이트, 압축 128바이트) | Solidity 직렬화 기준 768바이트 ($$\mathbb{G}_1$$ 점 9개 + 스칼라 6개). Commitment용 Custom Gate를 쓰면 Gate당 96바이트 추가 |
    | 검증 | Off-chain은 $$e(\alpha, \beta)$$를 미리 계산해 Pairing 3번, EVM에서는 4쌍 | KZG 확인 (Pairing 2번 + 여러 번의 $$\mathbb{G}_1$$ 스칼라 곱) |

5. **증명 생성**

    Prover는 실제 데이터(Assignment)로 `frontend.NewWitness(&assignment, field)`를 호출해 `witness` 객체를 만들고, Proving Key와 함께 `groth16.Prove(cs, pk, witness)`나 `plonk.Prove(cs, pk, witness)`에 넘긴다. gnark의 `witness` 객체는 Public Input과 Secret Input만 담는다. Witness의 나머지인 중간값은 `Prove`가 내부에서 Circuit을 풀어(`Solve`) 계산한다. 핵심 연산은 MSM과 FFT이며, gnark-crypto는 MSM을 Pippenger의 Bucket Method와 멀티코어 병렬화로 계산한다.

6. **검증**

    Verifier는 Verifying Key와 Public Input으로 `groth16.Verify(proof, vk, publicWitness)`나 `plonk.Verify(proof, vk, publicWitness)`를 호출한다. 로직을 다시 실행하지 않고 Pairing 등식만 확인한다.

    On-chain 검증에는 `vk.ExportSolidity(f)`로 Verifying Key를 Solidity 컨트랙트로 변환한다. gnark v0.15.0부터 Groth16 `Verifier.sol`은 증명을 `bytes calldata`로 받으며, Go 쪽에서는 `proof.MarshalSolidity()`로 이 형식에 맞게 직렬화한다. 생성된 Groth16 `Verifier.sol`은 4쌍(768바이트 입력)을 `ecPairing`에 한 번에 넘기며, 비용은 [On-chain Verification](#on-chain-verification)에서 본 대로 181,000 Gas다. Circuit이 Commitment를 쓰면 Pedersen Commitment를 확인하는 2쌍 Pairing 검사가 별도로 한 번 더 호출되어, $$34{,}000 \times 2 + 45{,}000 = 113{,}000$$ Gas가 더 든다.

    여기서 Commitment(`frontend.Committer`)는 Circuit 안의 일부 변수에 Commitment를 만들고, 그 값에서 얻은 난수를 Circuit 안에서 쓰게 하는 기능이다. 직접 호출하지 않아도 생길 수 있다. `std/rangecheck` 같은 Gadget은 Builder가 Commitment를 지원하면 이를 자동으로 쓰기 때문이다. 따라서 이런 Gadget을 쓰기만 해도 113,000 Gas가 추가될 수 있다.

### Full Example

1~6단계를 하나로 묶으면 다음과 같다.

```go
func main() {
    // 예시를 위해 오류 처리를 생략했다

    // 컴파일: Circuit을 R1CS로 변환
    var circuit CubicCircuit
    ccs, _ := frontend.Compile(ecc.BN254.ScalarField(), r1cs.NewBuilder, &circuit)

    // Setup: Groth16 Proving Key, Verifying Key 생성
    pk, vk, _ := groth16.Setup(ccs)

    // Witness 생성: x=3 일 때 y = 3³ + 3 + 5 = 35
    assignment := CubicCircuit{X: 3, Y: 35}
    witness, _ := frontend.NewWitness(&assignment, ecc.BN254.ScalarField())
    publicWitness, _ := witness.Public()

    // 증명
    proof, _ := groth16.Prove(ccs, pk, witness)

    // 검증
    if err := groth16.Verify(proof, vk, publicWitness); err != nil {
        panic("증명 검증 실패!")
    }
    // 검증 성공: Y=35를 만족하는 X를 Prover가 알고 있음을, X를 공개하지 않고 증명
}
```

### Strengths and Limitations

**Strengths**

- **빠른 증명 생성:** 하부 라이브러리 gnark-crypto가 MSM과 FFT를 저수준에서 최적화하고, CPU 병렬화와 Memory 최적화로 증명을 빠르게 만든다. Groth16에는 GPU(Graphics Processing Unit) 가속(ICICLE)도 실험적으로 지원된다.
- **익숙한 개발 환경:** 별도 언어 없이 Go 문법 안에서 struct와 API로 Circuit을 설계한다. Go의 타입 시스템, 테스트 프레임워크, CI/CD(Continuous Integration/Continuous Deployment)를 그대로 쓸 수 있다.
- **Ethereum 검증 지원:** Ethereum Precompile이 있는 곡선(BN254, BLS12-381)에 대해 Solidity Verifier를 자동 생성한다.
- **증명 시스템 선택:** Circuit 코드는 그대로 두고 Groth16과 PLONK를 모두 쓸 수 있다. Builder와 Setup, Prove, Verify 패키지를 바꾸고, PLONK에는 SRS를 따로 준비한다. 증명이 작은 Groth16과 Universal SRS를 쓰는 PLONK 중 선택할 수 있다.

**Limitations**

- **제어문 변환 필요:** 조건문은 `api.Select()`로, 입력에 따라 길이가 바뀌는 반복은 상한과 Selector 방식으로 바꿔야 한다. 제약 수를 의식한 설계가 필수다.
- **큰 개발 부담:** Circuit을 직접 설계해야 하므로 개발과 감사 부담이 크다.
- **Groth16의 운영 부담:** Groth16은 Circuit마다 Trusted Setup이 필요하다. 로직이 조금만 바뀌어도 키를 다시 생성하고 관리해야 한다. PLONK를 쓰면 Universal SRS를 재사용하므로 이 부담이 줄어든다.

## ZoKrates

[ZoKrates](https://zokrates.github.io/)는 Ethereum을 겨냥한 zk-SNARK 툴체인이다. 자체 DSL로 계산 로직을 작성하면, CLI(Command Line Interface) 하나로 컴파일부터 Solidity Verifier 생성까지 처리한다.

### Compilation

컴파일러(`zokrates_core/src/compile.rs`)는 다음 순서로 코드를 R1CS로 바꾼다.
1. **AST 생성:** 코드를 AST(Abstract Syntax Tree, 추상 구문 트리)로 변환하고 import를 처리한다.
2. **Semantic Check(의미 검사):** 타입과 선언을 검사한다.
3. **Static Analysis(정적 분석):** 반복문 Unrolling과 상수 전파를 수행한다. 그래서 반복문은 `for u32 i in 0..10 { ... }`처럼 반복 범위가 컴파일 시점의 상수여야 한다.
4. **Flattening(평탄화):** 조건문과 중첩 수식을 곱셈 하나씩만 포함한 기본 연산 단위로 분해한다. gnark와 달리 별도 단계로 존재한다.
5. **IR 변환:** 분해된 연산을 $$A \times B = C$$ 형식의 R1CS 제약으로 바꾼다. 이때 변수들은 행렬의 인덱스로 매핑된다.
6. **IR 최적화:** 만들어진 제약을 최적화한다.

### Proving Schemes and Setup

[ZoKrates 문서](https://zokrates.github.io/toolbox/proving_schemes.html)에 기재된 증명 체계는 다음과 같다. PLONK는 지원하지 않는다. ALTBN_128은 BN254와 같은 곡선이다. 이 표 외에 0.8.7에서 bellperson Backend 기반의 Nova가 실험적으로 추가되었으나 문서에는 기재되어 있지 않다.

| Scheme | 지원 곡선 | Universal | Backend |
|---|---|---|---|
| G16(Groth16) | ALTBN_128, BLS12_381 | No | bellman, ark |
| GM17(Groth-Maller 2017) | ALTBN_128, BLS12_381, BLS12_377, BW6_761 | No | ark |
| Marlin | ALTBN_128, BLS12_381, BLS12_377, BW6_761 | Yes | ark |

[ZoKrates 문서](https://zokrates.github.io/toolbox/proving_schemes.html) 기준으로 모든 Scheme에는 Circuit별 `setup` 단계가 있다.

- **G16, GM17:** Universal이 아니므로 `setup` 자체가 Circuit별 Trusted Setup이다. Toxic Waste가 생성되며, 생성 직후 반드시 파기해야 한다.
- **Marlin:** Circuit과 무관한 `universal-setup` 단계가 따로 있고, 신뢰가 필요한 것은 이 단계뿐이다. 한 번 실행해 그 SRS가 지원하는 최대 크기 이하의 여러 프로그램에 재사용한다. 이후 Circuit별 `setup`은 새로운 비밀 값 없이 키를 만든다.

### CLI Workflow

[ZoKrates 문서](https://zokrates.github.io/gettingstarted.html)의 예시는 Secret Input `a`의 제곱이 Public Input `b`임을 증명한다.

```zokrates
def main(private field a, field b) {
    assert(a * a == b);
    return;
}
```

```bash
zokrates compile -i root.zok             # 컴파일 (R1CS 생성)
zokrates setup                           # Proving Key, Verifying Key 생성
zokrates compute-witness -a 337 113569   # Witness 생성 (a = 337, b = 113569)
zokrates generate-proof                  # 증명 생성
zokrates export-verifier                 # Solidity Verifier 생성
zokrates verify                          # Off-chain 검증
```

`compute-witness`는 입력값으로 Circuit의 모든 중간 계산값을 구하고, `generate-proof`는 Proving Key와 Witness로 MSM과 FFT를 거쳐 증명을 만든다. G16 증명 크기는 BN254 기준 256바이트로 gnark의 Groth16과 같다. G16 기준으로 `export-verifier`가 만든 컨트랙트도 4쌍을 한 번에 확인하므로(`pairingProd4`), 비용 구조는 [On-chain Verification](#on-chain-verification)과 같다.

### Strengths and Limitations

- **Strengths:** 전용 문법이 간결하고, 암호학 세부 구현 없이 CLI만으로 On-chain 검증까지 구성할 수 있다.
- **Limitations:** PLONK를 지원하지 않으며, G16과 GM17은 Circuit별 Trusted Setup이 필요하다. 또 2024년 4월 이후 develop 브랜치에 새 커밋이 없다.

<br />
<br />

## Comparison

### zkVM vs Circuit: RISC Zero and gnark

두 도구 모두 "어떤 상태에 트랜잭션 집합을 적용했더니 새 상태가 나왔다"는 **State Transition의 정당성**을 증명하는 데 쓸 수 있다. 단, 그렇게 증명하도록 Guest 프로그램이나 Circuit을 설계해야 한다.

- **입력:** 이전 상태(Commitment, 비공개 가능), 트랜잭션, 서명, Merkle Proof
- **출력:** 새 상태(Commitment), 트랜잭션 실행 결과
- **증명 내용:** 정의된 검증 로직을 실행했더니 모든 검증에 성공했고 해당 출력이 나왔다는 사실

차이는 **검증 로직을 어디서 어떻게 표현하느냐**다.

- **RISC Zero:** 검증 로직을 Rust나 C/C++ 코드로 작성하고, 그 프로그램을 실행한 CPU의 Execution Trace 자체를 증명한다.
- **gnark:** 검증 로직을 산술 제약으로 작성하고, 그 방정식 시스템을 만족하는 Witness를 안다는 수학적 관계를 증명한다.

| 항목 | RISC Zero | gnark |
|---|---|---|
| 추상화 수준 | zkVM (RISC-V RV32IM 실행 증명) | Circuit DSL (Go struct + API) |
| 증명 시스템 | STARK + Recursion + Groth16 Wrapping (On-chain용) | Groth16, PLONK (KZG) |
| 프로그래밍 언어 | RISC-V로 컴파일 가능한 언어 (Rust, C/C++) | Go |
| 용도 | 임의 프로그램의 실행 증명, zkVM 기반 Rollup | 특정 기능 Circuit (Rollup, Bridge, Light Client), zkEVM의 Outer Proof (Linea), zkVM 증명 Wrapping (SP1) |
| 검증 로직 표현 | RISC-V Guest 코드 (반복문, 조건문 자유) | Circuit 제약 (`frontend.API`로 기록한 산술식) |
| 증명 대상 | 프로그램 전체의 Execution Trace | 제약 시스템을 만족하는 Witness |
| 암호 연산 | 가속 Circuit용 패치 크레이트 권장 | `gnark/std` Gadget (ECDSA, EdDSA, Merkle, Keccak) |
| On-chain 검증 | `verify(seal, imageId, journalDigest)` Solidity Verifier | `vk.ExportSolidity()`로 만든 Solidity Verifier |
| 로직 변경 시 | Image ID 갱신 (On-chain 허용 목록 포함) | 키 재생성 (Groth16은 Circuit별 Trusted Setup도 다시) |
| 성능 튜닝 | 명령어 수, Memory 접근, 가속 Circuit | 제약 수 최소화, 곡선과 Backend 선택 |

### Loop Handling by Execution Model

같은 Loop를 EVM, Trace 기반 증명, Circuit 기반 증명이 각각 어떻게 처리하는지 비교한다.

#### EVM

EVM에서 Loop는 비교 Opcode(`LT`, `GT` 등)와 조건부 점프(`JUMPI`)로 PC(Program Counter)를 옮겨 만든다. 아래는 `i`를 0부터 10까지 증가시키는 예시다. Stack은 오른쪽이 최상단이다.

```text
0   PUSH1 0x00   [i]            i = 0
2   JUMPDEST     [i]            loop:
3   PUSH1 0x0a   [i, 10]
5   DUP2         [i, 10, i]
6   LT           [i, c]         c = (i < 10)
7   ISZERO       [i, !c]
8   PUSH1 0x11   [i, !c, 17]
10  JUMPI        [i]            !c ≠ 0 (i ≥ 10)이면 17로 점프 (Loop 탈출)
11  PUSH1 0x01   [i, 1]         (Loop 본문 자리)
13  ADD          [i+1]          i = i + 1
14  PUSH1 0x02   [i+1, 2]
16  JUMP         [i+1]          loop로 점프 (PC 뒤로)
17  JUMPDEST     [i]            end:
```

- **실행 흐름:** PC가 2 → 3 → … → 16 → 2로 반복된다. Loop 한 번에 11단계를 실행한다.
- **검증:** 모든 노드가 Re-execution한다.
- **비용:** 실행량에 비례한다. 반복이 $$N$$번이면 전체 $$11N + 9$$단계이며, $$N = 10$$이면 119단계다.
- **제약:** Gas Limit 외에는 반복 횟수 제약이 없다.

**Loop가 1번이든 100만 번이든, 노드는 그만큼 Opcode를 실행해야 한다.**

#### Trace-Based Proof (zkEVM)

zkEVM은 같은 Opcode를 실행하며 Execution Trace를 만든다. zkVM도 같은 원리로 증명하지만, 같은 Loop를 EVM Opcode가 아니라 RISC-V 명령어 단위로 기록한다. 아래 Trace와 규칙은 zkEVM 기준이다.

```text
trace[t]:   PC=6,  opcode=LT,     stack [i, 10, i] → [i, c]
trace[t+1]: PC=7,  opcode=ISZERO, stack [i, c] → [i, !c]
trace[t+2]: PC=8,  opcode=PUSH1 0x11
trace[t+3]: PC=10, opcode=JUMPI,  cond=!c → cond=0이면 PC_next=11, cond≠0이면 PC_next=17
```

모든 스텝에 같은 규칙이 적용된다.

```text
IF opcode=JUMPI ∧ cond≠0 THEN PC_next − dest = 0  (그리고 code[dest] = JUMPDEST)
IF opcode=JUMPI ∧ cond=0 THEN PC_next − (PC + 1) = 0
```

$$N = 10$$일 때 Trace는 다음과 같다.

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

증명이 확인하는 것은 모든 $$t$$에 대해 두 가지다.
1. $$\text{opcode}_t$$에 따른 Stack과 Memory 변화 규칙을 지켰다.
2. 다음 PC 규칙 $$PC_{t+1} = f(PC_t, \text{opcode}_t, \text{cond}_t)$$을 지켰다.

증명은 Loop라는 개념을 다루지 않는다. PC의 점프 패턴이 Opcode 규칙을 지켰는지만 확인한다. 그래서 Loop 횟수 $$N \in \{1,\ 10,\ 10^6\}$$에 관계없이 **같은 제약 공식을 쓰고, Trace 길이만 $$N$$에 비례해 늘어난다.** 이 규칙과 나머지 제약(Stack, Gas, Memory 등)이 모든 행에서 성립하고, 시작과 종료 상태가 Public Input과 맞으면 Loop가 정당하게 실행되었다고 결론 내린다.

- **장점:** 가변 Loop를 자연스럽게 처리한다.
- **비용:** Trace 길이는 Loop 횟수 × Loop당 Opcode 수에 비례한다. Loop를 100만 번 돌면 Trace는 약 1,100만 행이 되어, **Prover의 증명 생성 시간이 늘어난다.**
- **상한:** 증명 하나가 담을 수 있는 Trace 길이에는 상한이 있다. 그래서 긴 실행은 여러 조각으로 나누어 각각 증명하고 Recursion으로 합친다. zkVM에서는 이 조각을 RISC Zero는 Segment, SP1은 Shard라고 부른다.

#### Circuit-Based Proof (gnark, ZoKrates)

Circuit은 **고정 크기의 산술식**이다. 프로그램 전체를 컴파일 시점에 고정 크기의 R1CS나 Plonkish 제약 집합으로 만들기 때문에, Loop 횟수나 리스트 길이가 바뀌면 Circuit 모양 자체가 바뀐다. 따라서 Secret Input에 따라 Loop 길이가 달라지면 Circuit 크기를 미리 정할 수 없다.

이는 **고정 상한과 Padding(채우기)**으로 우회한다. 최대 횟수(MAX)를 정해 두고, 남는 단계는 Selector로 비활성화한다.

```text
1. 고정 Loop (ZoKrates):
for u32 i in 0..10 { ... }           // 반복 횟수는 컴파일 시 상수

2. 가변 Loop (Padding, 의사 코드):
for (i = 0; i < MAX; i++) {          // MAX = 1000 (컴파일 시 상수)
  s_i = (i < secret_N)               // Selector: 비교 Circuit으로 계산 (결과는 Boolean)
  state = s_i × 실제작업(state) + (1 − s_i) × state
}
```

- **필요한 제약:** Selector를 만드는 방식에 따라 다르다.
    - 위처럼 $$s_i$$를 비교 Circuit으로 계산하면, $$s_i$$는 $$i < N$$인 앞쪽 단계에서만 1이다. 그래서 "활성 단계가 앞쪽에 연속한다"와 "활성 단계 수가 $$N$$이다"라는 조건이 자동으로 성립한다.
    - Selector를 Witness로 직접 받으면 이 조건을 따로 제약해야 한다. 빈 자리에 0을 넣는 것만으로는 부족하다. $$s_i$$가 0 또는 1인지($$s_i(1 - s_i) = 0$$), 활성 단계가 앞쪽에 연속하는지($$s_{i+1}(1 - s_i) = 0$$), 활성 단계 수가 $$N$$인지($$\sum_i s_i = N$$)를 함께 제약한다.
- **비용:** 실제 $$N = 10$$이고 $$\text{MAX} = 1000$$이면 Circuit은 항상 1000단계 분량이다. 단계 수 기준으로 100배이며, 990단계가 무의미하게 계산된다. 증명 시간이 정확히 100배가 되는 것은 아니다.
- **대안:** Recursion이나 Folding(접기, 여러 단계의 제약 인스턴스를 하나로 합치는 기법)으로 단계별 증명을 이어 붙이면 단일 고정 Circuit의 제약을 피할 수 있다.