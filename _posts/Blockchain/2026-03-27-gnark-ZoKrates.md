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
> 3. **Circuit-Based SNARK Tools: gnark and ZoKrates**
>

## Circuit 기반 SNARK 도구

zkVM이 범용 ISA의 실행을 증명하는 것과 달리, 아래 도구들은 **증명할 로직을 개발자가 직접 산술 제약(Circuit)으로 작성한다.**

### gnark

[gnark](https://github.com/Consensys/gnark)는 Consensys가 개발한 Go 기반 zk-SNARK 라이브러리다. Go의 struct와 전용 API(`frontend.API`)로 산술 Circuit을 설계하므로, Go 안에 내장된 DSL로 제약을 작성하는 방식이다.

gnark는 Frontend(Circuit 정의)와 Backend(증명 생성)를 분리한다. 작성한 Go 코드는 두 가지 제약 시스템 중 하나로 컴파일된다.
- **R1CS(Rank-1 Constraint System):** Groth16용
- **SCS(Sparse Constraint System):** PLONK용 Plonkish 제약

Go의 타입 시스템, 테스트 프레임워크, CI/CD(Continuous Integration/Continuous Deployment)를 그대로 쓸 수 있다. 지원 곡선은 BN254, BLS12-381, BLS12-377, BW6-761 등이며, Solidity Verifier 내보내기는 주로 BN254를 대상으로 한다.

gnark는 특정 연산의 증명부터 EVM 전체 State Transition까지 표현할 수 있다. 실제로 Linea ZK Rollup의 Prover가 gnark를 사용한다. 개발자가 제약을 직접 설계하므로 특정 연산에 맞춰 제약 수를 줄일 수 있다. 예를 들어 Keccak 해시나 ECDSA 서명 검증을 `gnark/std`의 가젯으로 Circuit 안에 직접 구성할 수 있다.

1. **Circuit 설계**

    Circuit은 Go `struct`로 정의한다. 필드는 기본적으로 Secret Witness이며, 공개할 필드에 `gnark:",public"`을 붙인다(쉼표 필요).

    - **Public Witness:** Verifier가 아는 값. 예: 이전 State Root, 새 State Root, 블록 헤더 정보
    - **Secret Witness:** 증명에 공개되지 않는 값. 예: 트랜잭션, 각 트랜잭션의 Merkle Proof, 서명

    ```go
    type CubicCircuit struct {
        // X는 Secret Witness (태그의 "x"는 이름이며, 공개 지정이 없으므로 Secret)
        X frontend.Variable `gnark:"x"`
        // Y는 Public Witness (Verifier도 아는 Public Input)
        Y frontend.Variable `gnark:",public"`
    }
    ```

2. **제약 작성(`Define`)**

    `Define(api frontend.API) error` 메서드에서 제약을 작성한다. `api.Add`, `api.Mul` 같은 산술 연산으로 $$x^3 + x + 5 = y$$ 같은 방정식을 구성하거나, `gnark/std` 가젯으로 Merkle Proof 검증과 EdDSA(Edwards-curve Digital Signature Algorithm), ECDSA 서명 검증을 Circuit에 포함한다.

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

3. **컴파일**

    `frontend.Compile()`이 Go 코드를 R1CS나 Plonkish 형식으로 변환한다. Ethereum Precompile과 호환하기 위해 주로 BN254 곡선을 쓰며, Circuit의 산술은 BN254의 스칼라체 $$\mathbb{F}_r$$(약 254bit 소수 $$r$$) 위에서 이루어진다. 곡선 방정식 $$y^2 = x^3 + 3$$은 기저체 $$\mathbb{F}_p$$ 위의 식으로, Circuit 산술이 이루어지는 스칼라체와는 다른 체다.

    컴파일은 **Flattening(평탄화)**을 거친다. 복잡한 수식을 곱셈 하나씩만 포함한 식들로 분해하는 작업이다. R1CS에서는 각 식이 $$A \times B = C$$ 형식의 제약이 된다. $$A, B, C$$는 Circuit의 모든 변수 $$v_i$$(입력, 중간값, 상수 $$v_0 = 1$$ 포함)의 Linear Combination(선형 결합)이다.

    $$
    \begin{aligned}
    A &= \alpha_0 v_0 + \alpha_1 v_1 + \cdots + \alpha_n v_n \\
    B &= \beta_0 v_0 + \beta_1 v_1 + \cdots + \beta_n v_n \\
    C &= \gamma_0 v_0 + \gamma_1 v_1 + \cdots + \gamma_n v_n
    \end{aligned}
    $$

    R1CS에서 $$A, B, C$$ 안의 덧셈은 별도 제약을 만들지 않는다. 그래서 **덧셈을 최대한 곱셈 제약의 괄호 안으로 합쳐 곱셈 수를 줄이는 것**이 R1CS 최적화의 핵심이다. PLONK의 Plonkish 제약은 게이트 입력 수가 제한되어 있어 덧셈에도 게이트가 필요하다.

    산술 Circuit은 입력에 따라 실행 경로를 바꿀 수 없으므로, 조건문과 반복문을 제약으로 바꿔야 한다.

    - **조건문:** `api.Select(condition, a, b)`로 대체한다. `condition`이 1이면 `a`, 0이면 `b`를 고르며, 내부적으로 `b + condition × (a − b)`로 계산해 곱셈 1개로 삽입된다. 현재 gnark는 R1CS와 SCS 빌더 모두 Select 안에서 `condition`이 0 또는 1이라는 Boolean 제약을 함께 추가한다.
    - **반복문:** 컴파일 시점에 Unrolling(펼치기)된다. `Define()` 안의 Go for 문은 Circuit을 컴파일할 때 실행되어 반복마다 제약을 생성한다. 반복 횟수가 컴파일 시점에 정해져 있으면 for 문을 그대로 쓰면 된다.
    - **입력 개수:** 입력 개수가 정해진 경우에는 고정 크기 배열을 struct 필드로 정의한다.

    이는 **컴파일 시점에 모든 제약 수가 정적으로 확정되어야 한다**는 Circuit 기반 SNARK의 특성 때문이다. 결과적으로 Prover는 이 방정식 시스템을 만족하는 Witness를 알고 있음을 증명한다. Verifier는 로직을 다시 실행하지 않고, 증명과 Verifying Key(검증 키)로 페어링 등식을 확인한다.

4. **Setup**

    - **Groth16:** `groth16.Setup(cs)`로 Circuit별 Proving Key(pk)와 Verifying Key(vk)를 만든다.
        - 이 과정에서 Toxic Waste(비밀 난수 $$\alpha, \beta, \gamma, \delta, \tau$$ 등)로 SRS(Structured Reference String)를 만든다.
        - Toxic Waste를 아는 사람은 증명을 위조할 수 있으므로 반드시 파기해야 한다.
        - 대규모 프로젝트에서는 MPC(Multi-Party Computation) 세레머니로 수행하며, 참여자 중 한 명만 정직하게 파기해도 안전하다.

    - **PLONK:** KZG(Kate-Zaverucha-Goldberg) Polynomial Commitment 기반의 Universal SRS를 쓴다.
        - gnark에서는 `plonk.Setup(ccs, srs, srsLagrange)`처럼 SRS를 인자로 넘긴다.
        - Circuit마다 새 세레머니를 하지 않고 공통 SRS를 재사용한다.
        - 다만 Circuit마다 Proving Key와 Verifying Key를 만드는 전처리는 필요하다. Circuit을 바꾸면 키를 다시 생성해야 하지만, 새 세레머니는 필요 없다.

    두 방식의 차이는 다음과 같다.

    | 구분 | Groth16 | PLONK (gnark, BN254) |
    |---|---|---|
    | Setup | Circuit별 | 공통 SRS 재사용 |
    | 증명 크기 | $$\mathbb{G}_1$$ 점 2개 + $$\mathbb{G}_2$$ 점 1개 (비압축 256byte, 압축 128byte) | Solidity 직렬화 기준 768byte ($$\mathbb{G}_1$$ 점 9개 + 스칼라 6개). Commitment 커스텀 게이트를 쓰면 게이트당 96byte 추가 |
    | 검증 | $$e(\alpha, \beta)$$를 미리 계산하면 페어링 3번 | KZG 확인 (페어링 2번 + 여러 번의 $$\mathbb{G}_1$$ 스칼라 곱) |

5. **증명 생성**

    Prover는 실제 데이터(Assignment)로 Witness를 만들고, Proving Key와 함께 Backend에 넘긴다. gnark에서는 `frontend.NewWitness(&assignment, field)`로 Witness를 만든 뒤 `groth16.Prove(cs, pk, witness)`나 `plonk.Prove(cs, pk, witness)`를 호출한다. 이 단계의 핵심 연산은 두 가지다.

    - **MSM(Multi-Scalar Multiplication):** 타원곡선 점들의 Linear Combination $$P = \sum_{i} r_i G_i$$를 계산한다. 증명 생성에서 가장 무거운 연산 중 하나이며, gnark-crypto는 Pippenger의 Bucket Method와 멀티코어 병렬화로 계산한다.
    - **FFT(Fast Fourier Transform):** 다항식의 계수 표현과 점 표현을 변환한다. 유한체 위에서는 NTT로 계산한다.

    gnark의 강점은 CPU 병렬화와 Memory 최적화를 통한 빠른 증명 생성이다. Groth16에는 GPU 가속(ICICLE)도 실험적으로 지원된다.

6. **검증**

    증명은 Verifying Key와 Public Witness로 `groth16.Verify(proof, vk, publicWitness)`나 `plonk.Verify(proof, vk, publicWitness)`를 호출해 확인한다.

    `vk.ExportSolidity(f)`는 Verifying Key를 Solidity 컨트랙트로 자동 변환한다. 생성된 `Verifier.sol`은 Ethereum의 `ecPairing` Precompile(주소 0x08)을 호출하며, 비용은 다음과 같다.

    - 페어링 4쌍 확인: 181,000 Gas (EIP-1108 기준)
    - Public Input 하나당: 약 6~7천 Gas 추가
    - 일반적인 총비용: 20만~30만 Gas

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

**장점**

- **빠른 증명 생성:** 하부 라이브러리 gnark-crypto가 MSM과 FFT를 저수준에서 최적화하고 병렬 처리한다.
- **익숙한 개발 환경:** Circom 같은 전용 문법 없이 Go struct와 API로 Circuit을 설계하며, Go 테스트 프레임워크와 CI/CD를 그대로 쓴다.
- **Ethereum 검증 지원:** Ethereum Precompile에 맞는 BN254 곡선을 지원하고, Solidity Verifier 자동 생성을 기본 제공한다.
- **증명 시스템 선택:** 하나의 Circuit 정의로 Groth16과 PLONK를 모두 쓸 수 있다(빌더만 바꾼다). 증명이 작은 Groth16과 Universal Setup이 가능한 PLONK 중 선택할 수 있다.

**한계**

- **제어문 변환 필요:** 조건문은 `api.Select()`로, 입력에 따라 길이가 바뀌는 반복은 상한과 Selector(선택자) 방식으로 바꿔야 한다. 제약 수를 의식한 설계가 필수다.
- **큰 개발 부담:** EVM 전체 State Transition도 표현할 수 있지만(Linea 사례), Circuit을 직접 설계해야 하므로 개발과 감사 부담이 크다.
- **Groth16의 운영 부담:** Groth16은 Circuit마다 Trusted Setup이 필요하다. 로직이 조금만 바뀌어도 키를 다시 생성하고 관리해야 한다. PLONK를 쓰면 공통 SRS를 재사용하므로 이 부담이 줄어든다.


### RISC Zero와 gnark 비교

두 도구 모두 "어떤 상태에 트랜잭션 집합을 적용했더니 새 상태가 나왔다"는 **State Transition의 정당성**을 증명하는 데 쓸 수 있다. 단, 그렇게 증명하도록 Guest 프로그램이나 Circuit을 설계해야 한다.

- **입력:** 이전 상태(Commitment, 비공개 가능), 트랜잭션, 서명, Merkle Proof
- **출력:** 새 상태(Commitment), 트랜잭션 실행 결과
- **증명 내용:** 정의된 검증 로직을 실행했더니 모든 검증에 성공했고 해당 출력이 나왔다는 사실

차이는 **검증 로직을 어디서 어떻게 표현하느냐**다.

- **RISC Zero:** 검증 로직을 Rust나 C++ 코드로 작성하고, 그 프로그램을 실행한 CPU의 Execution Trace 자체를 증명한다.
- **gnark:** 검증 로직을 산술 제약으로 작성하고, 그 방정식 시스템을 만족하는 Witness를 안다는 수학적 관계를 증명한다.

| 항목 | RISC Zero | gnark |
|---|---|---|
| 추상화 수준 | zkVM (RISC-V RV32IM 실행 증명) | Circuit DSL (Go struct + API) |
| 증명 시스템 | STARK + Recursion + Groth16 Wrapping (On-chain용) | Groth16, PLONK (KZG) |
| 프로그래밍 언어 | RISC-V로 컴파일 가능한 언어 (Rust, C/C++) | Go |
| 용도 | 임의 프로그램의 실행 증명, zkVM 기반 Rollup | 특정 기능 Circuit (Rollup, Bridge, Light Client), zkEVM Prover (Linea) |
| 검증 로직 표현 | RISC-V Guest 코드 (반복문, 조건문 자유) | Circuit 제약 (`frontend.API`, 평탄화된 산술식) |
| 증명 대상 | 프로그램 전체의 Execution Trace | 제약 시스템을 만족하는 Witness |
| 암호 연산 | 가속 Circuit용 패치 크레이트 권장 | `gnark/std` 가젯 (ECDSA, EdDSA, Merkle, Keccak) |
| On-chain 검증 | `verify(seal, imageId, journalDigest)` Solidity Verifier | `vk.ExportSolidity()`로 만든 Solidity Verifier |
| 로직 변경 시 | Image ID 갱신 (On-chain 허용 목록 포함) | 키 재생성 (Groth16은 Circuit별 Trusted Setup도 다시) |
| 성능 튜닝 | 명령어 수, Memory 접근, 가속 Circuit | 제약 수 최소화, 곡선과 Backend 선택 |


### ZoKrates
 
ZoKrates는 Ethereum을 겨냥한 zk-SNARK 툴체인으로, 자체 DSL로 산술 Circuit 설계를 추상화한다. 개발자가 ZoKrates 언어(예: `def main(private field a) -> field { return a * a; }`)로 계산 로직을 작성하면, 툴체인은 다음 순서로 처리한다.
1. 코드를 AST(Abstract Syntax Tree, 추상 구문 트리)로 변환한다.
2. 조건문, 반복문, 중첩 수식을 기본 연산 단위로 분해하는 Flattening을 수행한다.
3. 분해된 연산을 $$A \times B = C$$ 형식의 R1CS 제약으로 변환한다. 이때 변수들은 행렬의 인덱스로 매핑된다.

**Setup** 단계에서는 R1CS Circuit으로 Proving Key와 Verifying Key를 생성한다. 지원하는 증명 체계는 다음과 같으며, Backend는 bellman과 ark다. PLONK는 지원하지 않는다.
- **G16(Groth16):** Toxic Waste가 생성되며, 생성 직후 반드시 파기해야 한다.
- **GM17(Groth-Maller 2017)**
- **Marlin:** 공통 설정(`universal-setup`)을 재사용할 수 있는 범용 설정 방식이다. 다만 Circuit별 Setup 단계는 여전히 거친다.

**증명 생성** 단계에서는 먼저 입력값으로 Circuit의 모든 중간 계산값을 구하는 Witness 생성이 이루어진다. 이후 Proving Key와 Witness로 MSM과 FFT를 거쳐 증명을 만든다. 증명 크기는 G16 기준 수백 byte다. ZoKrates는 Verifying Key로 Solidity 검증 컨트랙트를 자동 생성하므로, 암호학 세부 구현 없이도 On-chain 검증을 구성할 수 있다.

CLI(Command Line Interface) 기반 자동화로 개발은 편하지만, 제약 수는 연산 종류에 따라 크게 다르다. 해시나 비트 연산은 코드 한 줄이 많은 제약으로 늘어나고, 덧셈은 추가 제약이 거의 없다. 증명 생성의 CPU와 Memory 부담은 Prover가 지고, Verifier는 작은 증명만 확인한다. 따라서 **ZoKrates의 한계는 검증 성능이 아니라 증명 생성 비용과 Circuit을 직접 설계하는 방식에 있다.**

<br />
<br />

## 실행 방식별 Loop 처리 비교

같은 Loop를 EVM, Trace 기반 증명(zkEVM, zkVM), Circuit 기반 증명이 각각 어떻게 처리하는지 비교한다.

### EVM

EVM에서 Loop는 비교 Opcode(`LT`, `GT` 등)와 조건부 점프(`JUMPI`)의 조합이다. 아래는 `i`를 0부터 10까지 증가시키는 예시다. Stack은 오른쪽이 최상단이다.

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

### Trace 기반 증명 (zkEVM, zkVM)

같은 Opcode를 실행하며 Execution Trace를 만든다.

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
- **상한:** 증명 하나가 담을 수 있는 Trace 길이에는 상한이 있다. 그래서 zkVM은 긴 실행을 Segment로 나누고 Recursion으로 합친다.

### Circuit 기반 증명 (gnark, ZoKrates, Circom)

Circuit은 **고정 크기의 산술식**이다. 프로그램 전체를 컴파일 시점에 고정 크기의 R1CS나 PLONK 제약 집합으로 만들기 때문에, Loop 횟수나 리스트 길이가 바뀌면 Circuit 모양 자체가 바뀐다. 따라서 Secret Input에 따라 Loop 길이가 달라지면 Circuit 크기를 미리 정할 수 없다.

이는 **고정 상한과 Padding**으로 우회한다. 최대 횟수(MAX)를 정해 두고, 남는 단계는 Selector로 비활성화한다.

```text
1. 고정 Loop (Circom):
for (var i = 0; i < 10; i++) { ... }  // 반복 횟수는 컴파일 시 상수

2. 가변 Loop (Padding):
for (i = 0; i < MAX; i++) {          // MAX = 1000 (컴파일 시 상수)
  s_i = (i < secret_N)               // Selector: 비교 Circuit으로 계산하고 Boolean 제약
  state = s_i × 실제작업(state) + (1 − s_i) × state
}
```

- **필요한 제약:** 빈 자리에 0을 넣는 것만으로는 부족하다. Selector가 0 또는 1인지, 활성 단계가 앞쪽에 연속하는지, 활성 단계 수가 $$N$$인지를 함께 제약해야 한다.
- **비용:** 실제 $$N = 10$$이고 $$\text{MAX} = 1000$$이면 Circuit은 항상 1000단계 분량이다. 단계 수 기준으로 100배이며, 990단계가 무의미하게 계산된다. 증명 시간이 정확히 100배가 되는 것은 아니다.
- **대안:** Recursion이나 Folding 기법으로 단계별 증명을 이어 붙이면 단일 고정 Circuit의 제약을 피할 수 있다.
