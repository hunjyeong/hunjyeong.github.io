---
title: "Multiproof"
date: 2026-11-11
categories: [Mathematics]
tags: [Mathematics, ZKP]
math: true
---

# Multiproof

이 문서는 Polynomial Commitment Scheme(다항식 커밋먼트 스킴, PCS)에서 여러 개의 평가 주장을 하나의 짧은 증명으로 묶는 **Multiproof(멀티프루프, 다중 증명)**를 설명한다. 기준 자료는 Dankrad Feist의 [PCS multiproofs using random evaluation](https://dankradfeist.de/ethereum/2021/06/18/pcs-multiproofs.html)(2021-06-18)이며, 기호도 이 글을 따른다.

설명 순서는 다음과 같다.

1. 왜 Multiproof가 필요한지 (Commitment, Merkle Tree, Verkle Tree)
2. 이해에 필요한 핵심 개념 (Opening, KZG, IPA, Fiat-Shamir)
3. 프로토콜의 동작 원리
4. 정확성과 안전성의 근거
5. 손으로 계산하는 예시
6. Verkle Tree 적용, 비용, 구현 최적화

Group(군), Field(체), 타원곡선, Pairing 같은 수학 개념은 [부록](#부록)에 정리했다. 본문에서 처음 등장할 때는 한 줄로 뜻을 적고, 자세한 설명은 부록으로 연결한다.

**문서 전체의 공통 가정**

- 모든 스칼라 연산은 큰 소수 $p$에 대한 Finite Field(유한체) $\mathbb{F}_p$에서 이루어진다. $\mathbb{F}_p$는 $0$부터 $p-1$까지의 정수에 대해 덧셈과 곱셈을 $p$로 나눈 나머지로 정의한 집합이다([부록 B](#b-field체와-finite-field유한체)).
- 다항식의 차수는 $d$ 미만이다. Verkle Tree에서 $d$는 한 노드가 가지는 자식 수, 즉 Width(폭)와 같다.
- 군 연산은 덧셈으로 표기한다. 스칼라 $a$와 군 원소 $P$의 곱 $aP$는 $P$를 $a$번 더한 값이다([부록 A](#a-group군), [부록 D](#d-elliptic-curve-group타원곡선-군과-discrete-logarithm-problem이산-로그-문제)).

---

## 1. 배경과 필요성

### 1.1. Commitment Scheme

**Commitment Scheme(커밋먼트 스킴)**은 어떤 값을 지금 고정해 두되 내용은 나중에 공개하는 암호학적 방법이다. 두 단계로 이루어진다.

- $\mathrm{Commit}(m, \rho) \rightarrow C$: 메시지 $m$과 무작위값 $\rho$로 **Commitment(커밋먼트)** $C$를 만든다.
- $\mathrm{Verify}(C, m, \rho) \rightarrow$ accept 또는 reject: 나중에 공개된 $m$, $\rho$가 $C$와 맞는지 확인한다. 이 공개 과정을 **Opening(오프닝)**이라 한다.

Commitment Scheme은 두 가지 성질로 평가한다.

- **Binding(바인딩)**: 하나의 $C$를 서로 다른 두 메시지로 여는 것이 어렵다. 처음에 $A$를 넣었다면 나중에 $B$였다고 말을 바꿀 수 없다.
- **Hiding(하이딩)**: $C$만 보고는 $m$에 대한 정보를 얻을 수 없다.

가장 단순한 예는 Hash 함수(임의 길이 입력을 고정 길이 출력으로 바꾸는 함수)를 이용하는 방식이다. $C = H(m \,\|\, \rho)$로 두면 Hash의 충돌 저항성이 Binding을 제공한다. 여기서 $\|$는 이어 붙이기다. 무작위값 $\rho$ 없이 $C = H(m)$만 쓰면 Hiding이 약해진다. $m$의 후보가 적다면 후보를 모두 Hash해 보고 $C$와 비교할 수 있기 때문이다.

### 1.2. Vector Commitment와 Merkle Tree

**Vector Commitment(벡터 커밋먼트)**는 벡터 $(v_0, \ldots, v_{n-1})$ 전체를 하나의 Commitment $C$로 요약하고, 나중에 "$i$번째 원소는 $v_i$이다"라는 주장만 따로 증명할 수 있게 하는 Commitment Scheme이다. 전체 벡터를 보내지 않고 일부만 증명할 수 있다는 점이 핵심이다.

**Merkle Tree(머클 트리)**는 Vector Commitment의 대표적인 예다. 원소들을 Leaf(리프, 트리의 맨 아래 노드)에 두고, 부모 노드는 자식들의 Hash로 만든다. 맨 위의 Root(루트) Hash가 전체 벡터의 Commitment 역할을 한다. 특정 Leaf를 증명하려면 Leaf에서 Root까지 올라가는 각 단계에서 **Sibling Node(형제 노드, 같은 부모를 공유하는 다른 자식)**를 모두 제공해야 한다. 검증자가 부모의 Hash를 다시 계산하려면 모든 자식이 필요하기 때문이다.

- 이진 Merkle Tree에서 Leaf가 $n$개이면 증명에는 약 $\log_2 n$개의 Hash가 필요하다.
- 폭이 $w$인 Merkle Tree는 단계마다 $w-1$개의 형제 노드를 제공해야 한다. 트리 높이는 낮아지지만 단계당 비용이 커진다.

[Vitalik Buterin의 Verkle Tree 글](https://vitalik.eth.limo/general/2021/06/18/verkle.html)은 이 때문에 Merkle Patricia Tree가 폭 2에서 가장 효율적이고, Ethereum이 사용해 온 16진(폭 16) 구조는 증명 크기 면에서 최적이 아니라고 설명한다.

### 1.3. Polynomial Commitment Scheme

**Polynomial Commitment Scheme(다항식 커밋먼트 스킴, PCS)**은 다항식 $f(X)$에 대한 Commitment $C$를 만들고, 임의의 점 $z$에 대해 "$f(z) = y$"라는 주장을 짧은 증명으로 확인할 수 있게 하는 방식이다. 증명을 만드는 쪽을 **Prover(증명자)**, 확인하는 쪽을 **Verifier(검증자)**라 한다. "$f(z) = y$"라는 주장과 그 증명을 묶어서 Opening이라 부르고, 증명 자체는 **Opening Proof(오프닝 증명)**라 부른다.

PCS는 Vector Commitment로도 쓸 수 있다. 미리 정한 좌표 $x_0, \ldots, x_{n-1}$에 대해 $f(x_i) = v_i$를 만족하는 차수 $n$ 미만의 다항식은 하나뿐이다. 이 다항식을 Lagrange Interpolation(라그랑주 보간)으로 구해서 Commit하면, "$i$번째 원소가 $v_i$"라는 주장은 "$f(x_i) = v_i$"라는 Opening이 된다([부록 F](#f-roots-of-unity단위근과-lagrange-interpolation라그랑주-보간)).

강의 자료에 따라서는 PCS를 Functional Commitment(함수 커밋먼트)의 한 종류로 소개한다. Functional Commitment는 함수 $f$를 Commit한 뒤 "$f(x) = y$"를 증명하는 방식이고, PCS는 함수의 범위를 "차수 $d$ 미만의 다항식"으로 제한한 경우다.

대표적인 PCS 두 가지와 Opening Proof 크기는 다음과 같다. 정확한 정의는 2장에서 다룬다.

- **KZG Commitment**: Opening Proof가 군 원소 1개로 일정하다. Trusted Setup이 필요하다.
- **IPA 기반 Commitment**: Opening Proof가 약 $2\log_2 d$개의 군 원소를 포함한다. Trusted Setup이 필요 없지만 검증 시간이 $d$에 비례한다.

### 1.4. Verkle Tree에서 생기는 문제

**Verkle Tree(버클 트리)**는 Merkle Tree와 같은 트리 구조를 쓰되, 부모 노드를 만들 때 Hash 대신 Vector Commitment(실제로는 PCS)를 쓰는 트리다. 이 아이디어는 John Kuszmaul의 2018년 논문에서 처음 제시되었다고 [Vitalik Buterin의 글](https://vitalik.eth.limo/general/2021/06/18/verkle.html)이 소개한다.

Verkle Tree에서는 형제 노드를 보낼 필요가 없다. 대신 경로 위의 각 단계마다 "자식 노드는 부모 Commitment의 $i$번째 위치 값이다"라는 Opening을 증명한다. 폭을 크게 해도 형제 노드 비용이 늘지 않으므로 트리를 넓고 낮게 만들 수 있다.

이 구조가 필요한 이유는 **Stateless Client(무상태 클라이언트)** 때문이다. Stateless Client는 전체 상태를 저장하지 않고, 블록과 함께 전달된 **Witness(위트니스, 블록 검증에 필요한 상태 값과 그 증명 데이터)**만으로 블록을 검증하는 노드다. [EIP-6800](https://eips.ethereum.org/EIPS/eip-6800)(Ethereum Improvement Proposal, 이더리움 개선 제안)은 기존 16진 Patricia Tree에서 계정 하나를 접근하는 Witness가 평균 약 3 kB이고, Verkle Tree에서는 계정당 평균 약 200 B로 줄어든다고 설명한다.

그런데 Opening은 경로의 단계마다 하나씩 생긴다. 여러 Key를 동시에 증명하면 Opening 수는 더 늘어난다. Dankrad Feist는 이 개수가 수백에서 수천 개에 이를 수 있다고 설명한다. Opening마다 별도의 Opening Proof를 붙이면 증명 크기가 Opening 수에 비례해 커진다.

**Multiproof는 이 문제를 해결한다.** $m$개의 Opening을 묶어서, 핵심 증명의 크기가 $m$과 무관하도록 만든다. 단, 경로 위의 중간 Commitment들은 여전히 Witness에 포함되어야 한다. 이 차이는 [6장](#6-verkle-tree-적용)에서 다시 정리한다.

---

## 2. 핵심 개념

### 2.1. Opening과 Polynomial Remainder Theorem

**Polynomial Remainder Theorem(다항식 나머지 정리)**에 따르면 임의의 다항식 $f(X)$를 $X - z$로 나누면 다음이 성립한다.

$$
f(X) - y = (X - z)\,q(X) + \bigl(f(z) - y\bigr)
$$

여기서 $q(X)$는 **Quotient(몫)**, $f(z) - y$는 **Remainder(나머지)**다. 따라서

$$
f(z) = y \iff (X - z) \text{ 가 } f(X) - y \text{ 를 나누어떨어지게 한다.}
$$

즉, 주장이 참일 때에만 다음 몫이 다항식이 된다.

$$
q(X) = \frac{f(X) - y}{X - z}
$$

주장이 거짓이면 이 식은 다항식이 아니라 **Rational Function(유리함수, 다항식을 다항식으로 나눈 꼴의 함수)**이 된다.

**예시.** $f(X) = X^2 + X + 3$, $z = 2$라 하자. $f(2) = 9$이다.

$$
\frac{f(X) - 9}{X - 2} = \frac{X^2 + X - 6}{X - 2} = \frac{(X - 2)(X + 3)}{X - 2} = X + 3
$$

몫이 다항식이다. 반면 $y = 10$이라고 거짓 주장을 하면 다음과 같다.

$$
\frac{f(X) - 10}{X - 2} = X + 3 - \frac{1}{X - 2}
$$

$-\frac{1}{X - 2}$가 남으므로 다항식이 아니다.

> 참고: Vitalik Buterin의 글에도 같은 예시가 있는데, 몫을 $X - 3$으로 적고 있다. $(X - 2)(X + 3) = X^2 + X - 6$이므로 올바른 몫은 $X + 3$이다.

Opening Proof의 기본 아이디어는 "몫이 다항식임을 보이면 주장이 참임을 보인 것"이다. Multiproof도 이 아이디어에서 출발한다.

### 2.2. KZG Commitment

**Kate-Zaverucha-Goldberg Commitment(KZG Commitment)**는 Kate, Zaverucha, Goldberg가 2010년 논문 *Constant-Size Commitments to Polynomials and Their Applications*(ASIACRYPT 2010)에서 제안한 PCS다. Dankrad Feist의 [KZG polynomial commitments](https://dankradfeist.de/ethereum/2020/06/16/kate-polynomial-commitments.html)에 쉬운 설명이 있다.

**준비물**

- $\mathbb{G}_1$, $\mathbb{G}_2$: 원소 개수(Order, 위수)가 소수 $p$인 타원곡선 군이다. 생성원은 각각 $g_1$, $g_2$다.
- $[a]_1 = a\,g_1$, $[a]_2 = a\,g_2$: 스칼라 $a$를 생성원에 곱한 군 원소다.
- $e: \mathbb{G}_1 \times \mathbb{G}_2 \rightarrow \mathbb{G}_T$: **Bilinear Pairing(쌍선형 페어링)**이다. $e([a]_1, [b]_2) = e(g_1, g_2)^{ab}$를 만족한다. $\mathbb{G}_T$는 곱셈으로 표기한다([부록 E](#e-bilinear-pairing쌍선형-페어링)).

**Trusted Setup(신뢰 설정)**

비밀값 $s \in \mathbb{F}_p$를 정하고 다음 공개 매개변수를 만든 뒤 $s$를 폐기한다.

$$
\mathrm{pp} = \bigl([s^0]_1, [s^1]_1, \ldots, [s^{d-1}]_1,\; [1]_2,\; [s]_2\bigr)
$$

학습 노트에 따라 비밀값을 $\alpha$로 쓰기도 한다. 이 문서는 Dankrad Feist의 표기를 따라 $s$를 쓴다. 누군가 $s$를 알면 거짓 Opening을 만들 수 있으므로 $s$는 누구도 알아서는 안 된다. 실제로는 여러 참여자가 차례로 자신의 비밀값을 섞는 Ceremony(세리머니)로 만들며, 이때 참여자 중 한 명이라도 자신의 비밀값을 폐기했다면 $s$는 알려지지 않는다.

**Commit**

차수 $d$ 미만의 다항식 $f(X) = \sum_{j=0}^{d-1} a_j X^j$에 대해 다음을 계산한다.

$$
C = \mathrm{Commit}(f) = [f(s)]_1 = \sum_{j=0}^{d-1} a_j\,[s^j]_1
$$

$s$를 몰라도 공개 매개변수의 선형 결합으로 계산할 수 있다. 결과는 $\mathbb{G}_1$의 원소 하나다.

**Additive Homomorphism(덧셈 동형성)**

$$
\mathrm{Commit}(a f + b g) = a\,\mathrm{Commit}(f) + b\,\mathrm{Commit}(g)
$$

이 성질 덕분에 Verifier는 다항식을 몰라도 Commitment들의 선형 결합으로 "다항식들의 선형 결합에 대한 Commitment"를 직접 계산할 수 있다. **Multiproof가 동작하는 데 반드시 필요한 성질이다.**

**Opening Proof와 검증**

$f(z) = y$를 증명하려면 Prover는 몫 $q(X) = \frac{f(X) - y}{X - z}$를 계산하고 다음을 보낸다.

$$
\pi = [q(s)]_1
$$

Verifier는 다음 등식을 확인한다.

$$
e\bigl(C - [y]_1,\; [1]_2\bigr) = e\bigl(\pi,\; [s - z]_2\bigr)
$$

$[s - z]_2 = [s]_2 - z\,[1]_2$이므로 Verifier는 $s$를 몰라도 계산할 수 있다. 주장이 참이면 $f(s) - y = q(s)(s - z)$이고, Pairing의 쌍선형성이 이 곱셈 관계를 지수에서 확인해 준다.

**성질과 크기**

- Commitment와 Opening Proof가 각각 $\mathbb{G}_1$ 원소 1개다. Vitalik Buterin의 글은 KZG Commitment와 Proof를 48 B로 가정하는데, 이는 BLS12-381(Pairing에 적합한 타원곡선의 한 종류) $\mathbb{G}_1$ 원소의 압축 표현 크기다.
- 안전성은 $s$가 알려지지 않았다는 전제와 Pairing 군에서의 계산 가정(원 논문은 Strong Diffie-Hellman 계열 가정을 사용)에 의존한다.
- 위의 기본 형태는 무작위값을 넣지 않으므로 표준적인 의미의 Hiding을 제공하지 않는다. 원 논문은 무작위성을 추가한 변형도 제시한다.

### 2.3. Pedersen Vector Commitment와 IPA

Ethereum의 Verkle Tree 제안(EIP-6800)은 KZG가 아니라 Pedersen Vector Commitment와 IPA를 사용한다. Multiproof는 두 방식 모두에 적용되므로 함께 정리한다.

**Pedersen Vector Commitment(페더슨 벡터 커밋먼트)**

서로의 Discrete Logarithm(이산 로그) 관계가 알려지지 않은 군 원소 $G_0, \ldots, G_{d-1}$을 고정한다. "관계가 알려지지 않았다"는 것은 $G_i = c\,G_j$를 만족하는 $c$를 아무도 모른다는 뜻이다([부록 D](#d-elliptic-curve-group타원곡선-군과-discrete-logarithm-problem이산-로그-문제)). 벡터 $v = (v_0, \ldots, v_{d-1})$의 Commitment는 다음과 같다.

$$
C = \sum_{i=0}^{d-1} v_i\,G_i
$$

- 비밀값이 없으므로 Trusted Setup이 필요 없다. 생성원은 Hash 등으로 공개적으로 만들 수 있다.
- Binding은 Discrete Logarithm Problem(이산 로그 문제)의 어려움에 의존한다.
- 위 식만으로는 Hiding이 없다. 무작위값 $\rho$와 별도 생성원 $H$를 더한 $C = \sum v_i G_i + \rho H$ 형태를 쓰면 Hiding을 얻는다.
- KZG와 마찬가지로 Additive Homomorphism을 만족한다.

$v_i$를 다항식의 평가값 $f(x_i)$로 보면, 이 Commitment는 Evaluation Form(평가값 표현)으로 주어진 다항식에 대한 PCS가 된다.

**Inner Product Argument(내적 논증, IPA)**

IPA는 Commitment $C$ 안의 벡터 $v$와 공개 벡터 $b$의 내적 $\langle v, b \rangle = y$를 증명하는 방식이다. 다항식 평가는 내적으로 쓸 수 있다. Evaluation Form에서 $f(t) = \sum_i v_i\,\ell_i(t)$이므로 $b_i = \ell_i(t)$로 두면 $f(t) = \langle v, b \rangle$다. 여기서 $\ell_i$는 Lagrange Basis Polynomial(라그랑주 기저 다항식)이다([부록 F](#f-roots-of-unity단위근과-lagrange-interpolation라그랑주-보간)).

- IPA Opening Proof는 약 $2\log_2 d$개의 군 원소와 소수의 스칼라로 이루어진다.
- Verifier의 계산량은 $d$에 비례한다. Proof는 짧지만 검증이 KZG처럼 상수 시간은 아니다.
- 자세한 원리는 Dankrad Feist의 [Inner Product Arguments](https://dankradfeist.de/ethereum/2021/07/27/inner-product-arguments.html)에 있다.

### 2.4. Fiat-Shamir Transform

Multiproof는 원래 Prover와 Verifier가 메시지를 주고받는 대화형 프로토콜이다. Verifier는 중간에 **Challenge(챌린지, Prover가 미리 예측할 수 없는 무작위값)**를 보낸다.

**[Fiat-Shamir Transform](https://en.wikipedia.org/wiki/Fiat%E2%80%93Shamir_heuristic)(피아트-샤미르 변환)**은 이 Challenge를 "지금까지 주고받은 모든 값", 즉 **Transcript(트랜스크립트)**의 Hash로 대체한다. 그러면 Prover 혼자 증명을 만들 수 있고, Verifier는 같은 Hash를 다시 계산해 확인한다.

- Challenge를 만들 때 그 시점까지 고정된 공개값을 빠짐없이 Hash 입력에 넣어야 한다. 빠진 값이 있으면 Prover가 Challenge를 본 뒤 그 값을 조작할 수 있다.
- 안전성 분석은 보통 Hash를 이상적인 무작위 함수로 보는 **Random Oracle Model(랜덤 오라클 모델)**에서 이루어진다.
- 실제 구현은 Domain Separation(도메인 분리, 용도마다 다른 표지를 Hash 입력에 넣는 것)과 Hash 출력을 $\mathbb{F}_p$ 원소로 바꾸는 규칙을 명세로 정한다. 예를 들어 [go-ipa](https://pkg.go.dev/github.com/crate-crypto/go-ipa)의 Multiproof 함수는 `common.Transcript` 객체를 인자로 받는다.

---

## 3. Multiproof 동작 원리

이 장의 식은 KZG를 기준으로 쓴다. IPA를 쓰는 경우의 차이는 [6.4절](#64-kzg와-ipa-비교)에서 정리한다.

### 3.1. 증명할 관계

$m$개의 Commitment $C_i = [f_i(s)]_1$이 있고, 다음 $m$개의 평가 주장을 한꺼번에 증명하려 한다.

$$
f_0(z_0) = y_0,\quad f_1(z_1) = y_1,\quad \ldots,\quad f_{m-1}(z_{m-1}) = y_{m-1}
$$

- 서로 다른 $i$에서 $f_i$가 같을 수도 있다. 같은 노드를 여러 위치에서 여는 경우다.
- 서로 다른 $i$에서 $z_i$가 같을 수도 있다.
- 서로 다른 평가 지점의 집합을 $Z$, 그 개수를 $k = |Z|$라 하자. $k \le m$이다.
- Verifier는 $C_i$, $z_i$, $y_i$를 알고 있다고 가정한다. Verkle Tree에서는 $z_i$, $y_i$를 다른 데이터로부터 계산한다([6.2절](#62-witness-구성)).

목표는 **증명 크기가 $m$과 무관한 증명**을 만드는 것이다.

### 3.2. 전체 흐름 요약

1. 각 Opening의 몫을 무작위 가중치 $r^i$로 더해 하나의 함수 $g(X)$를 만든다. 모든 주장이 참이면 $g(X)$는 다항식이다.
2. Prover가 $g(X)$의 Commitment $D$를 보낸다.
3. $D$를 고정한 뒤 무작위 점 $t$를 정하고, "$D$가 나타내는 다항식의 $t$에서의 값이 $g(t)$와 같다"는 것을 Opening 하나로 확인한다.

Dankrad Feist의 표현으로는 "남은 모든 작업은 $g(X)$가 유리함수가 아니라 다항식임을 최소 비용으로 보이는 것"이다.

### 3.3. 1단계: Challenge $r$과 집계 함수 $g(X)$

모든 주장을 고정한 뒤 첫 번째 Challenge를 만든다.

$$
r \leftarrow H(C_0, \ldots, C_{m-1},\; y_0, \ldots, y_{m-1},\; z_0, \ldots, z_{m-1})
$$

Prover는 다음 함수를 계산한다.

$$
g(X) = \sum_{i=0}^{m-1} r^i\,\frac{f_i(X) - y_i}{X - z_i}
$$

모든 주장이 참이면 각 항이 다항식이므로 $g(X)$도 다항식이다. 각 $f_i$의 차수가 $d - 1$ 이하이므로 각 몫과 $g(X)$의 차수는 $d - 2$ 이하다.

**왜 그냥 더하지 않고 $r^i$를 곱하는가.** 단순히 몫들을 더하면 서로 다른 거짓 주장의 나머지가 상쇄되어 합이 다항식처럼 보일 수 있다. 예를 들어 같은 점 $z$에서 한 주장의 나머지가 $+1$, 다른 주장의 나머지가 $-1$이면 $\frac{1}{X - z} - \frac{1}{X - z} = 0$이 된다. $r$은 모든 입력이 고정된 뒤 정해지므로 Prover가 이런 상쇄를 미리 맞출 수 없다. 이 부분은 [4.2절](#42-soundness건전성)에서 확률로 정리한다.

### 3.4. 2단계: 집계 Commitment $D$

Prover는 $g(X)$의 Commitment를 보낸다.

$$
D = [g(s)]_1
$$

KZG로 Commit할 수 있는 것은 다항식뿐이다. 그러나 **$D$를 보냈다는 사실만으로 $D$가 위의 $g(X)$에 대한 Commitment라는 보장은 없다.** 부정직한 Prover는 다른 다항식의 Commitment를 보낼 수 있다. $D$가 실제로 Commit하는 다항식을 $f_D(X)$라 하자. 정직하면 $f_D = g$다. 남은 단계는 이것을 확인하는 과정이다.

Dankrad Feist는 Binding 성질 때문에 Prover가 $D$를 열 수 있는 다항식은 많아야 하나라고 설명한다.

### 3.5. 3단계: Challenge $t$와 문제의 분해

$D$를 받은 뒤 두 번째 Challenge를 만든다.

$$
t \leftarrow H(r, D)
$$

$t$는 $D$에 의존하므로 Prover는 $D$를 정할 때 $t$를 알 수 없다. 이 순서가 중요한 이유는 [5.2절](#52-거짓-주장이-들키는-과정) 예시에서 확인할 수 있다.

이제 확인하고 싶은 값은 다음과 같다.

$$
g(t) = \sum_{i=0}^{m-1} r^i\,\frac{f_i(t) - y_i}{t - z_i}
$$

Verifier는 $f_i(t)$를 모르므로 이 값을 직접 계산할 수 없다. 그래서 식을 두 부분으로 나눈다.

$$
g(t) = \underbrace{\sum_{i=0}^{m-1} \frac{r^i}{t - z_i}\,f_i(t)}_{h(t)} \;-\; \underbrace{\sum_{i=0}^{m-1} \frac{r^i\,y_i}{t - z_i}}_{y}
$$

두 번째 부분 $y$는 $r$, $t$, $y_i$, $z_i$만으로 계산되는 스칼라다. Verifier가 직접 계산한다.

$$
y = \sum_{i=0}^{m-1} \frac{r^i\,y_i}{t - z_i}
$$

첫 번째 부분은 다음 다항식 $h(X)$의 $t$에서의 값이다.

$$
h(X) = \sum_{i=0}^{m-1} \frac{r^i}{t - z_i}\,f_i(X)
$$

$h(X)$는 $f_i(X)$들의 선형 결합이고 계수 $\frac{r^i}{t - z_i}$는 Verifier가 알고 있다. 따라서 Additive Homomorphism에 의해 Verifier는 $h(X)$의 Commitment를 직접 계산할 수 있다.

$$
E = \sum_{i=0}^{m-1} \frac{r^i}{t - z_i}\,C_i = [h(s)]_1
$$

결국 확인할 관계는 다음 하나로 줄어든다.

$$
h(t) - f_D(t) = y
$$

좌변은 Commitment $E - D$가 나타내는 다항식 $h(X) - f_D(X)$의 $t$에서의 값이다. 즉, **$m$개의 Opening을 "Commitment $E - D$를 점 $t$에서 열면 $y$이다"라는 Opening 하나로 바꾼 것이다.**

**$t = z_i$인 경우.** 이때는 $t - z_i = 0$이 되어 식이 정의되지 않는다. Dankrad Feist의 글은 이 경우를 따로 다루지 않는다. $t$가 큰 $\mathbb{F}_p$에서 Hash로 정해지므로 $t \in Z$일 확률은 많아야 $k/p$로 매우 작다. 구현이 이 경우를 어떻게 처리하는지는 해당 구현의 명세를 확인해야 한다.

### 3.6. 4단계: 최종 Opening Proof $\pi$

정직한 Prover라면 $f_D = g$이고 $h(t) - g(t) = y$이므로, $h(X) - g(X) - y$는 $X = t$에서 $0$이다. 따라서 다음 몫은 다항식이다.

$$
w(X) = \frac{h(X) - g(X) - y}{X - t}
$$

Prover는 이 몫의 Commitment를 Opening Proof로 보낸다.

$$
\pi = [w(s)]_1
$$

**최종 증명은 $(D, \pi)$, 즉 $\mathbb{G}_1$ 원소 2개다.** $E$와 $y$는 Verifier가 직접 계산하므로 보낼 필요가 없다.

### 3.7. 검증

Verifier는 다음 KZG Opening 등식을 확인한다.

$$
e\bigl(E - D - [y]_1,\; [1]_2\bigr) = e\bigl(\pi,\; [s - t]_2\bigr)
$$

이 등식이 성립하면 $D$를 무작위 점 $t$에서 열었을 때 Verifier가 계산한 $g(t)$와 정확히 같다는 뜻이다. Prover는 $D$를 정할 때 $t$를 몰랐다. Dankrad Feist는 [Schwartz-Zippel Lemma](https://en.wikipedia.org/wiki/Schwartz%E2%80%93Zippel_lemma)(슈바르츠-지펠 보조정리)에 따라 $D$가 실제로 $g(X)$의 Commitment가 아닌데 이 등식이 성립할 가능성은 실질적으로 없으며, 따라서 $g(X)$는 다항식이고 모든 주장이 참이라고 결론짓는다.

검증 등식은 하나지만 Pairing 계산이 한 번이라는 뜻은 아니다. 등식 양변에 Pairing이 하나씩 있다.

### 3.8. 절차 정리

**Prover**

입력: 모든 $f_i(X)$, $C_i$, $z_i$, $y_i$.

1. $r = H(C_0, \ldots, C_{m-1}, y_0, \ldots, y_{m-1}, z_0, \ldots, z_{m-1})$를 계산한다.
2. $g(X) = \sum_i r^i \frac{f_i(X) - y_i}{X - z_i}$를 계산하고 $D = [g(s)]_1$을 만든다.
3. $t = H(r, D)$를 계산한다.
4. $h(X) = \sum_i \frac{r^i}{t - z_i} f_i(X)$와 $y = \sum_i \frac{r^i y_i}{t - z_i}$를 계산한다.
5. $w(X) = \frac{h(X) - g(X) - y}{X - t}$를 계산하고 $\pi = [w(s)]_1$을 만든다.
6. $(D, \pi)$를 보낸다.

**Verifier**

입력: 신뢰할 수 있는 $C_i$, $z_i$, $y_i$, 증명 $(D, \pi)$, 공개 매개변수.

1. 같은 방식으로 $r$을 계산한다.
2. $t = H(r, D)$를 계산한다.
3. $E = \sum_i \frac{r^i}{t - z_i} C_i$와 $y = \sum_i \frac{r^i y_i}{t - z_i}$를 계산한다.
4. $e(E - D - [y]_1, [1]_2) = e(\pi, [s - t]_2)$를 확인한다.

---

## 4. 왜 올바르게 동작하는가

### 4.1. Completeness(완전성)

**Completeness(완전성)**는 모든 주장이 참이고 Prover가 정직하면 검증을 반드시 통과한다는 성질이다.

모든 $f_i(z_i) = y_i$가 참이면 $g(X)$는 다항식이다. $h(t) - g(t)$를 전개하면 다음과 같다.

$$
\begin{aligned}
h(t) - g(t)
&= \sum_i \frac{r^i f_i(t)}{t - z_i} - \sum_i \frac{r^i \bigl(f_i(t) - y_i\bigr)}{t - z_i} \\
&= \sum_i \frac{r^i y_i}{t - z_i} \\
&= y
\end{aligned}
$$

따라서 $h(X) - g(X) - y$는 $X - t$로 나누어떨어지고, 다음이 성립한다.

$$
h(X) - g(X) - y = (X - t)\,w(X)
$$

$X$에 $s$를 넣고 Commitment 표기로 바꾸면 $E - D - [y]_1 = [(s - t)\,w(s)]_1$이다. Pairing의 쌍선형성에 의해

$$
e\bigl(E - D - [y]_1, [1]_2\bigr) = e\bigl([w(s)]_1, [s - t]_2\bigr) = e(\pi, [s - t]_2)
$$

이므로 검증을 통과한다.

### 4.2. Soundness(건전성)

**Soundness(건전성)**는 거짓 주장이 하나라도 있으면 검증을 통과할 확률이 무시할 만큼 작다는 성질이다. 두 Challenge는 서로 다른 역할을 한다.

- $r$: 여러 주장의 오류가 서로 상쇄되는 것을 막는다.
- $t$: Prover가 보낸 $D$가 요구된 관계를 실제로 만족하는지 무작위 점에서 검사한다.

아래 확률은 Challenge가 $\mathbb{F}_p$에서 균등하게 뽑힌다고 이상화한 계산이다. Dankrad Feist의 글은 정성적 논증만 제시하므로, 아래 상계는 그 논증을 수식으로 풀어 쓴 것이다.

#### $r$의 역할: 오류 상쇄 방지

각 주장의 오차를 $\delta_i = f_i(z_i) - y_i$로 두자. 나머지 정리에 의해

$$
\frac{f_i(X) - y_i}{X - z_i} = q_i(X) + \frac{\delta_i}{X - z_i}
$$

이고 $q_i(X)$는 다항식이다. 이를 $g(X)$에 넣고 같은 평가 지점끼리 모으면

$$
g(X) = \sum_i r^i\,q_i(X) + \sum_{z \in Z} \frac{P_z(r)}{X - z},
\qquad
P_z(r) = \sum_{i:\,z_i = z} \delta_i\,r^i
$$

이다. 서로 다른 $z$에 대한 항 $\frac{c_z}{X - z}$들은 다항식 부분과 섞이지 않는다($(X - z)$를 곱한 뒤 $X = z$를 넣으면 $c_z$만 남는다). 따라서

$$
g(X) \text{ 가 다항식} \iff \text{모든 } z \in Z \text{ 에 대해 } P_z(r) = 0
$$

이다. 거짓 주장이 하나라도 있으면 그 점 $z$에서 $P_z$는 $r$에 대한 0이 아닌 다항식이고 차수는 $m - 1$ 이하다. 0이 아닌 다항식의 근은 차수 이하이므로([부록 C](#c-polynomial다항식과-rational-function유리함수)), 주장을 고정한 뒤 $r$을 균등하게 뽑으면

$$
\Pr\bigl[P_z(r) = 0\bigr] \le \frac{m - 1}{p}
$$

이다.

#### $t$의 역할: $D$의 정확성 검사

Prover가 보낸 $D$가 다항식 $f_D(X)$의 Commitment라 하자. 검증이 통과하면 (KZG Opening의 안전성 아래) $h(t) - f_D(t) = y$, 즉 $f_D(t) = g(t)$가 성립한다. 분모를 없애기 위해

$$
Q(X) = \prod_{z \in Z} (X - z)
$$

를 정의하고, 다음 다항식을 생각한다.

$$
R(X) = Q(X)\bigl(f_D(X) - g(X)\bigr)
$$

$Q(X)\,g(X) = \sum_i r^i (f_i(X) - y_i)\frac{Q(X)}{X - z_i}$이고 각 $\frac{Q(X)}{X - z_i}$는 다항식이므로 $R(X)$는 다항식이다. 차수는 $d + k - 1$ 이하다. $t \notin Z$이면 $Q(t) \ne 0$이므로

$$
f_D(t) = g(t) \iff R(t) = 0
$$

이다. $r$ 단계에서 상쇄가 일어나지 않았다면 $g$는 다항식이 아니므로, 어떤 다항식 $f_D$를 골라도 $f_D - g \ne 0$이고 $R(X)$는 0이 아닌 다항식이다. $D$를 고정한 뒤 $t$를 $Z$ 밖에서 균등하게 뽑으면

$$
\Pr\bigl[R(t) = 0\bigr] \le \frac{d + k - 1}{p - k}
$$

이다. 이것이 Schwartz-Zippel Lemma의 일변수 형태다([부록 G](#g-schwartz-zippel-lemma슈바르츠-지펠-보조정리)).

#### 전체 상계와 전제

거짓 주장이 있는데 검증을 통과하려면 다음 중 하나가 일어나야 한다.

1. $r$에서 오류가 상쇄된다. 확률 $\le \frac{m - 1}{p}$.
2. $t$에서 우연히 $R(t) = 0$이 된다. 확률 $\le \frac{d + k - 1}{p - k}$.
3. KZG 자체의 안전성(Binding, Opening의 안전성)이 깨진다.

$\mathbb{F}_p$의 크기는 BLS12-381의 스칼라 체와 EIP-6800이 쓰는 Bandersnatch 곡선의 군 위수 모두 약 $2^{252}$ 이상이다. $m$, $d$가 수천 수준이면 1과 2의 확률은 무시할 수 있다.

이 계산은 다음 전제 위에서만 성립한다.

- Fiat-Shamir에서 Prover는 입력을 바꿔 가며 Hash를 여러 번 시도할 수 있다. Random Oracle Model에서는 시도 횟수만큼 위 확률이 커질 수 있다. 그래도 $p$가 충분히 크면 무시할 수준이다.
- $r$의 Hash 입력에 모든 $C_i$, $y_i$, $z_i$가, $t$의 Hash 입력에 $r$과 $D$가 들어가야 한다.
- $D$를 열 수 있는 다항식이 하나로 정해진다는 것(Binding)과 KZG Opening의 안전성을 가정한다.
- 엄밀한 암호학적 보안 증명은 이 문서의 범위를 벗어난다. 같은 계열의 KZG 확장 방식과 보안 분석은 Boneh, Drake, Fisch, Gabizon의 [Efficient polynomial commitment schemes for multiple points and polynomials](https://eprint.iacr.org/2020/081)(ePrint 2020/081)를 참고할 수 있다. 이 논문의 초록은 증명이 군 원소 두 개로 이루어지는 방식을 소개한다. 이 문서에서 두 방식의 세부 차이를 비교하지는 않았다.

### 4.3. Multiproof가 보장하지 않는 것

- **$C_i$ 자체의 진위.** Multiproof는 주어진 $C_i$에 대한 평가 주장만 확인한다. $C_i$가 올바른 트리 노드인지는 Verkle Tree에서 Root부터 이어지는 Opening 체인으로 보장된다.
- **Zero-Knowledge(영지식).** Multiproof는 $y_i$를 숨기지 않는다. 공개된 평가 주장의 정확성을 증명할 뿐이다. 이름에 "proof"가 들어간다고 Zero-Knowledge Proof(영지식 증명, ZKP)인 것은 아니다.
- **증명만으로의 집계.** Dankrad Feist는 이 방식이 모든 다항식에 접근할 수 있을 때 효율적이며, 데이터 없이 기존 증명들만으로 집계해야 하는 상황에는 맞지 않는다고 밝힌다.

---

## 5. 손 계산 예시

수식의 흐름을 확인하기 위한 예시다. $r$, $t$는 Hash에서 나온 값이 아니라 계산을 보여 주기 위해 고른 값이다. 모든 계산은 충분히 큰 $p$에 대한 $\mathbb{F}_p$에서 한다고 본다. 숫자가 작아서 일반 정수 계산과 결과가 같다.

### 5.1. 정직한 경우

두 주장을 묶는다.

$$
\begin{aligned}
f_0(X) &= X + 2, & z_0 &= 3, & y_0 &= 5 \\
f_1(X) &= X^2 + 1, & z_1 &= 2, & y_1 &= 5
\end{aligned}
$$

$f_0(3) = 5$, $f_1(2) = 5$이므로 두 주장 모두 참이다.

**① 몫**

$$
q_0(X) = \frac{X + 2 - 5}{X - 3} = 1,
\qquad
q_1(X) = \frac{X^2 + 1 - 5}{X - 2} = \frac{(X - 2)(X + 2)}{X - 2} = X + 2
$$

**② $r = 2$로 집계**

$$
g(X) = r^0 q_0(X) + r^1 q_1(X) = 1 + 2(X + 2) = 2X + 5,
\qquad
D = [2s + 5]_1
$$

**③ $t = 4$에서 분해**

$$
h(X) = \frac{r^0}{4 - 3} f_0(X) + \frac{r^1}{4 - 2} f_1(X) = (X + 2) + (X^2 + 1) = X^2 + X + 3
$$

Verifier는 Commitment만으로 $E$를 계산한다.

$$
E = \frac{1}{1} C_0 + \frac{2}{2} C_1 = C_0 + C_1
$$

$$
y = \frac{r^0 y_0}{4 - 3} + \frac{r^1 y_1}{4 - 2} = \frac{5}{1} + \frac{10}{2} = 10
$$

확인: $h(4) - g(4) = 23 - 13 = 10 = y$.

**④ 최종 Opening Proof**

$$
w(X) = \frac{h(X) - g(X) - y}{X - t} = \frac{X^2 - X - 12}{X - 4} = \frac{(X - 4)(X + 3)}{X - 4} = X + 3,
\qquad
\pi = [s + 3]_1
$$

**⑤ 검증**

$$
\begin{aligned}
e(E - D - [10]_1, [1]_2) &= e\bigl([s^2 - s - 12]_1, [1]_2\bigr) \\
e(\pi, [s - 4]_2) &= e\bigl([s + 3]_1, [s - 4]_2\bigr)
\end{aligned}
$$

$(s + 3)(s - 4) = s^2 - s - 12$이므로 양변이 같다. 두 Opening을 $D$, $\pi$ 두 군 원소로 검증했다.

### 5.2. 거짓 주장이 들키는 과정

이번에는 $y_1 = 6$이라고 거짓 주장을 한다고 하자. 실제 값은 $f_1(2) = 5$다.

$$
\frac{f_1(X) - 6}{X - 2} = \frac{X^2 - 5}{X - 2} = X + 2 - \frac{1}{X - 2}
$$

따라서 $r = 2$일 때 Verifier가 요구하는 $g$는 다항식이 아니다.

$$
g(X) = 2X + 5 - \frac{2}{X - 2}
$$

Prover는 이 함수를 KZG로 Commit할 수 없다. 대신 그럴듯한 다항식 $f_D(X) = 2X + 5$로 $D$를 보낸다고 하자. $t = 4$에서 Verifier가 계산하는 값은 다음과 같다.

$$
y = \frac{5}{1} + \frac{2 \times 6}{2} = 11
$$

그런데 $h(4) - f_D(4) = 23 - 13 = 10 \ne 11$이다. $h(X) - f_D(X) - 11$은 $X = 4$에서 $-1$이므로 $X - 4$로 나누어떨어지지 않는다. 올바른 $\pi$를 만들 수 없으므로 검증에 실패한다.

**$t$를 $D$ 이후에 정해야 하는 이유.** 만약 Prover가 $t = 4$를 미리 안다면 $f_D(X) = 2X + 4$를 고를 수 있다. 그러면 $h(4) - f_D(4) = 23 - 12 = 11 = y$가 되어 검증을 통과한다. 실제로는 $t = H(r, D)$이므로 $D$를 바꾸면 $t$도 바뀐다.

$R(X)$ 관점에서 보면, $Z = \{3, 2\}$, $Q(X) = (X - 3)(X - 2)$이고 $f_D(X) = 2X + 5$일 때

$$
f_D(X) - g(X) = \frac{2}{X - 2},
\qquad
R(X) = Q(X)\bigl(f_D(X) - g(X)\bigr) = 2(X - 3)
$$

이다. $R$은 $X = 3$에서만 0인데, $3 \in Z$이므로 $t$로 허용되지 않는다. 따라서 이 $D$로는 어떤 $t$에서도 통과할 수 없다. $f_D(X) = 2X + 4$라면 $R(X) = (X - 3)(4 - X)$이므로 $Z$ 밖의 근은 $X = 4$ 하나뿐이고, $t$가 우연히 4가 될 때만 통과한다.

### 5.3. $r$ 없이 단순히 더할 때의 상쇄

같은 점 $z$에서 두 주장의 오차가 $\delta_0 = 1$, $\delta_1 = -1$이라 하자. 가중치 없이 더하면 나머지 항이 $\frac{1}{X - z} - \frac{1}{X - z} = 0$이 되어 두 거짓 주장이 함께 통과할 수 있다. $r^i$ 가중치를 쓰면 나머지 항은 $\frac{1 - r}{X - z}$이 되고, $r = 1$일 때만 상쇄된다. $P_z(r) = 1 - r$의 근이 하나뿐이므로 확률은 $\frac{1}{p}$이다.

---

## 6. Verkle Tree 적용

### 6.1. 경로를 Opening으로 바꾸기

Dankrad Feist의 예시에서 Key `0101 0111 1010 1111`, 값 `1213`인 Leaf를 증명하려면 Root, Inner Node A, Inner Node B로 이어지는 경로에서 다음 세 주장을 증명한다. 각 노드의 Commitment를 $C_2$(Root), $C_1$(A), $C_0$(B)라 한다.

$$
\begin{aligned}
f_0(\omega^{1010_2}) &= H(\text{key}, \text{value}) \\
f_1(\omega^{0111_2}) &= H(C_0) \\
f_2(\omega^{0101_2}) &= H(C_1)
\end{aligned}
$$

- 위치 인덱스는 그대로 쓰지 않고 $z = \omega^{\text{index}}$로 바꾼다. $\omega$는 $d$차 Root of Unity(단위근)로, 연산을 효율적으로 만든다([부록 F](#f-roots-of-unity단위근과-lagrange-interpolation라그랑주-보간), [7.3절](#73-evaluation-form-최적화)).
- 오른쪽 값은 다음 단계의 노드(또는 Leaf)를 Hash한 값이다. 원문은 충돌 저항 Hash의 예로 SHA-256(Secure Hash Algorithm 256-bit)을 든다.
- 경로가 깊어지거나 여러 Key를 동시에 증명하면 이런 주장이 수백에서 수천 개로 늘어난다. 이들을 모두 하나의 Multiproof로 묶는다.

오른쪽 값을 어떻게 만드는지는 트리 명세에 따라 다르다. 예를 들어 EIP-6800은 자식 Commitment(군 원소)를 체 원소로 바꾸는 `group_to_scalar_field` 함수를 쓰며, 명세 주석은 이 함수가 충돌 저항 Hash나 Random Oracle은 아니지만 Pedersen Commitment에 대해 Binding이라고 설명한다. Leaf 쪽도 Stem(키 앞 31바이트)과 값의 상하위 16바이트를 나누어 넣는 별도 인코딩을 사용한다.

### 6.2. Witness 구성

Verkle 증명에서 $z_i$와 $y_i$는 따로 보낼 필요가 없다. Vitalik Buterin의 설명은 다음과 같다.

- $y_i$는 경로 위의 다음 값(Commitment 또는 Leaf)의 Hash이므로 Verifier가 계산할 수 있다.
- $z_i$는 Key와 경로에서 정해진다.
- 따라서 필요한 것은 증명할 Leaf(Key와 값)와 각 Leaf에서 Root까지 경로 위의 Commitment들뿐이다.

이것은 Verkle Tree의 구조를 이용한 최적화다. 일반적인 Multiproof에서는 Verifier가 $z_i$, $y_i$를 어떤 방식으로든 알고 있어야 한다.

| 구성 요소 | 내용 | 크기 |
| --- | --- | --- |
| 핵심 Multiproof | $D$, $\pi$ (IPA이면 $D$와 IPA Proof) | Opening 수 $m$과 무관 |
| 경로 Commitment | Root와 Leaf 사이의 중간 노드 Commitment | 서로 다른 경로 노드 수에 비례 |
| Leaf 데이터 | 증명할 Key와 값 | 증명 대상 수에 비례 |
| Root Commitment | Verifier가 블록 헤더 등에서 이미 신뢰하는 값 | 보통 Witness에 다시 넣지 않음 |

여러 Key의 경로가 위쪽에서 겹치면 공유되는 Commitment는 한 번만 넣는다. Vitalik Buterin은 폭 256인 트리에서 증명할 값이 아무리 많아도 최상위 단계에서 256개보다 많은 값을 제공할 필요는 없다고 설명한다.

### 6.3. 크기

**핵심 Multiproof.** KZG에서 $D$와 $\pi$는 $\mathbb{G}_1$ 원소 2개다. BLS12-381의 압축 표현(원소당 48 B)을 쓰면 96 B다. 이 수치는 핵심 증명만의 크기이며, 경로 Commitment와 Leaf 데이터는 포함하지 않는다.

**전체 Witness.** Vitalik Buterin의 글은 폭 256, KZG Commitment와 Proof 48 B, 최대한 균형 잡힌 트리를 가정하고 다음과 같은 증명 크기 추정을 제시한다. 아래는 트리 크기가 $2^{32}$(4,294,967,296)인 행이다.

| 증명할 Key/Value 쌍 수 | 1 | 10 | 100 | 1,000 | 10,000 |
| --- | --- | --- | --- | --- | --- |
| 증명 크기 (B) | 320 | 1,472 | 13,616 | 107,744 | 937,472 |

핵심 Multiproof는 일정하지만 전체 Witness는 증명할 Key 수에 따라 커진다는 점을 보여 준다. 같은 글은 실제 무작위 트리에서는 깊이가 약 0.6단계 늘어나 원소당 약 30 B가 추가되고, Bulletproof 계열 Commitment(32 B)를 쓰면 크기가 약 1/3 줄어든다고 덧붙인다. 모두 2021년 시점의 추정치다.

**구성 설명의 차이.** Vitalik Buterin의 글은 Multiproof를 "다항식 Commitment 1개, 숫자 1개, 증명 2개, 스킴에 따라 128~1,000 B"로 설명한다. 이는 Dankrad Feist 글의 최종 형태인 $(D, \pi)$와 다르다. 이 차이가 어떤 변형에서 나왔는지는 두 글에서 확인하지 못했다. 한 가지 가능성(추정)은 Prover가 $g(t)$ 값을 보내고 $D$와 $E$를 각각 $t$에서 여는 변형이다. 이 경우 Commitment 1개($D$), 숫자 1개($g(t)$), Opening Proof 2개가 된다. 이 문서는 Dankrad Feist 글의 형태를 기준으로 한다.

### 6.4. KZG와 IPA 비교

Dankrad Feist는 이 방식이 KZG를 염두에 두고 작성되었지만, 두 Commitment를 더하면 합 다항식의 Commitment가 되는 모든 덧셈 동형 PCS에 적용되며 IPA에도 적용된다고 밝힌다. 차이는 마지막 Opening, 즉 "$E - D$를 $t$에서 열면 $y$"를 무엇으로 증명하느냐에 있다.

| 항목 | KZG | Pedersen + IPA |
| --- | --- | --- |
| 핵심 Multiproof | $D$와 $\pi$ (군 원소 2개) | $D$와 IPA Opening Proof |
| 마지막 Opening Proof 크기 | 군 원소 1개 | 약 $2\log_2 d$개 군 원소와 소수의 스칼라 |
| 마지막 Opening 검증 | Pairing 등식 1개 | $d$에 비례하는 계산 |
| Trusted Setup | 필요 | 불필요 |
| Pairing | 필요 | 불필요 |

어느 쪽이든 노드 폭 $d$가 고정이면 Opening 수 $m$이 늘어나도 핵심 증명 크기는 늘지 않는다.

[go-ipa](https://pkg.go.dev/github.com/crate-crypto/go-ipa)는 Verkle Tree용 암호 기본 요소 라이브러리로, Bandersnatch 곡선과 Banderwagon 소수 위수 군, Pedersen Vector Commitment, Evaluation Form 다항식에 대한 IPA, Multiproof를 구현한다. 2024-07-24 게시 버전 기준으로 `MultiProof` 구조체는 `IPA`와 `D` 두 필드로 이루어진다. 평가 지점은 `uint8` 인덱스로 받으므로 노드 폭 256과 맞는다. 이 인덱스가 체의 어떤 원소에 대응하는지는 라이브러리가 따르는 [verkle-trie-ref](https://github.com/crate-crypto/verkle-trie-ref) 명세에서 확인해야 한다. pkg.go.dev는 이 패키지가 모듈의 최신 버전에 포함되어 있지 않다고 표시하므로, 최신 구조는 저장소에서 다시 확인해야 한다.

### 6.5. Ethereum에서의 현황 (2026-11-11 확인)

이 절은 시간이 지나면 바뀔 수 있는 내용이다.

- [EIP-6800](https://eips.ethereum.org/EIPS/eip-6800)(통합 Verkle Tree로 Ethereum 상태 저장)은 eips.ethereum.org에서 Stagnant 상태로 표시된다. 명세는 Bandersnatch 곡선 위의 Pedersen 기저 256개로 Commitment를 계산한다.
- [EIP-7864](https://ethereum-magicians.org/t/eip-7864-ethereum-state-using-a-unified-binary-tree/22611)는 Hash 기반 이진 트리로 상태 트리를 바꾸는 제안이며, 2025-01-20에 만들어진 Draft다. 논의 글은 Hash 기반 상태 트리에 다시 관심이 생긴 이유로 양자 컴퓨터 관련 우려와 SNARK, STARK 같은 증명 시스템의 발전을 든다. 사용할 Hash 함수는 아직 확정되지 않았다.
- Vitalik Buterin의 2021년 글도 결론에서, 양자 컴퓨터가 등장하면 Verkle Tree가 의존하는 선형 동형성이 안전하지 않게 되어 Hash 기반 증명으로 옮겨야 할 것이라고 적었다.

따라서 Multiproof는 Verkle Tree를 이해하는 데 핵심인 기법이지만, Ethereum 상태 트리에 실제로 채택될지는 현재 로드맵을 따로 확인해야 한다.

---

## 7. 계산 비용과 구현

### 7.1. Verifier

Dankrad Feist는 Verifier의 주된 작업이 $E = \sum_i \frac{r^i}{t - z_i} C_i$를 계산하는 Multiexponentiation 한 번이라고 설명한다. 군을 덧셈으로 표기하면 같은 연산을 **Multiscalar Multiplication(다중 스칼라 곱셈, MSM)**이라 부른다. 여러 군 원소에 각각 스칼라를 곱해 더하는 연산이다. 같은 Commitment가 여러 번 나오면 계수를 먼저 더해 MSM의 항 수를 줄일 수 있다.

Vitalik Buterin은 $N$개 값을 증명하는 Verkle 증명의 검증 비용을 다음과 같이 정리한다.

- 크기 $N$의 타원곡선 선형 결합 1회
- 약 $4N$회의 체 연산(256비트 모듈러 연산)
- 증명 크기와 무관한 소량의 고정 작업

같은 글은 수천 개 값도 100 ms 안에 검증할 수 있다고 쓰지만, 이는 하드웨어와 구현을 명시하지 않은 2021년 추정이다. 일반적인 성능 보장으로 쓰면 안 된다. 또한 KZG가 아니라 IPA를 쓰면 마지막 Opening 검증에 $d$에 비례하는 계산이 추가된다.

### 7.2. Prover

Vitalik Buterin은 증명 생성에서 가장 큰 비용이 각 항 $r^i \frac{f_i(X) - y_i}{X - z_i}$를 계산하는 것이며, 항마다 대략 노드 폭의 4배 정도의 체 연산이 든다고 설명한다. 타원곡선 곱셈 한 번이 보통 수백 번의 체 연산이므로 이는 작은 비용이다.

Prover는 각 $f_i(X)$를 알아야 한다. Verkle Tree에서는 증명에 포함되는 각 노드의 자식 값 전체(또는 같은 다항식을 다시 만들 수 있는 데이터)가 필요하다. 기존의 개별 Opening Proof만으로는 이 방식으로 집계할 수 없다. 다만 블록체인의 전체 상태가 반드시 있어야 한다는 뜻은 아니다. 증명 대상 노드의 데이터에 접근할 수 있으면 된다.

### 7.3. Evaluation Form 최적화

Dankrad Feist는 실제 구현에서 다항식을 계수로 다루지 않고 **Evaluation Form(평가값 표현)**으로만 다룬다고 설명한다. 이 절은 정확성 이해에는 필요 없지만 효율적인 구현에는 필요하다.

**Evaluation Form.** 정의역 $x_j = \omega^j$ $(j = 0, \ldots, d-1)$에서의 값 $(v_0, \ldots, v_{d-1})$로 차수 $d$ 미만의 다항식을 표현한다. 이 표현과 Coefficient Form(계수 표현)은 일대일로 대응한다. 두 표현 사이의 변환은 Discrete Fourier Transform(이산 푸리에 변환, DFT)이며, Fast Fourier Transform(고속 푸리에 변환, FFT)으로 약 $\frac{d}{2}\log d$회 곱셈에 계산할 수 있다.

**Commit.** Trusted Setup에서 $[\ell_j(s)]_1$을 미리 계산해 두면 계수 변환 없이 다음과 같이 Commit한다.

$$
[f(s)]_1 = \sum_{j=0}^{d-1} v_j\,[\ell_j(s)]_1
$$

**나눗셈.** 나누어떨어지는 나눗셈은 각 점에서 값끼리 나누면 된다. 계수 표현에서 $O(d^2)$인 작업이 $d$회 나눗셈으로 줄어든다. 단, 평가 지점 $z = x_m$이 정의역 안에 있으면 $q(X) = \frac{f(X) - y}{X - x_m}$의 $x_m$에서의 값이 $\frac{0}{0}$이 된다. 이때는 다음 식을 쓴다.

$$
q_j = \frac{v_j - y}{x_j - x_m} \quad (j \ne m),
\qquad
q_m = \sum_{i \ne m} (v_i - y)\,\frac{A'(x_m)}{A'(x_i)}\,\frac{1}{x_m - x_i}
$$

여기서 $A(X) = \prod_{i}(X - x_i)$이고 $A'$는 형식 미분이다. 정의역이 단위근이면 $A(X) = X^d - 1$, $A'(\omega^i) = d\,\omega^{-i}$이므로 $\frac{A'(x_m)}{A'(x_i)} = \omega^{i - m}$으로 간단해진다.

**정의역 밖의 점에서 평가.** $t$는 정의역 밖의 점이다. 계수 표현으로 바꾸지 않고 Barycentric Formula(무게중심 공식)로 $O(d)$에 계산한다. 단위근 정의역에서는 다음과 같다.

$$
f(t) = \frac{t^d - 1}{d} \sum_{i=0}^{d-1} v_i\,\frac{\omega^i}{t - \omega^i}
$$

$t$가 정의역 밖에 있으므로 최종 몫 $w(X) = \frac{h(X) - g(X) - y}{X - t}$는 $\frac{0}{0}$ 문제 없이 각 점에서 바로 나누어 계산할 수 있다.

---

## 8. 핵심 정리

1. 단일 Opening $f(z) = y$의 참 여부는 $f(X) - y$가 $X - z$로 나누어떨어지는지와 같다.
2. Multiproof는 $m$개의 몫을 무작위 가중치 $r^i$로 더해 $g(X)$를 만든다. $r$은 거짓 주장들의 나머지가 상쇄되는 것을 막는다.
3. Prover는 $g(X)$의 Commitment $D$를 보낸다. $D$가 올바른지는 $D$를 고정한 뒤 정한 무작위 점 $t$에서 확인한다.
4. Verifier는 Additive Homomorphism으로 $E = \sum_i \frac{r^i}{t - z_i} C_i$를 직접 계산하고, "$E - D$를 $t$에서 열면 $y$"라는 Opening 하나로 모든 주장을 확인한다.
5. KZG 기준 핵심 증명은 $(D, \pi)$ 두 군 원소이고 크기는 $m$과 무관하다. Verkle Tree의 전체 Witness에는 경로 Commitment와 Leaf 데이터가 따로 들어가므로 Key 수에 따라 커진다.
6. 안전성은 큰 유한체, Commitment의 Binding, 올바른 Fiat-Shamir Transcript 구성, Challenge의 무작위성 가정에 의존한다.
7. Multiproof는 Zero-Knowledge를 제공하지 않으며, Prover는 증명 대상 다항식의 데이터에 접근할 수 있어야 한다.

---

## 9. 참고 자료

- Dankrad Feist, [PCS multiproofs using random evaluation](https://dankradfeist.de/ethereum/2021/06/18/pcs-multiproofs.html), 2021-06-18. 이 문서의 기준 자료.
- Dankrad Feist, [KZG polynomial commitments](https://dankradfeist.de/ethereum/2020/06/16/kate-polynomial-commitments.html), 2020-06-16.
- Dankrad Feist, [Inner Product Arguments](https://dankradfeist.de/ethereum/2021/07/27/inner-product-arguments.html), 2021-07-27.
- Dankrad Feist, [Verkle trie for Eth1 state](https://dankradfeist.de/ethereum/2021/06/18/verkle-trie-for-eth1.html), 2021-06-18.
- Vitalik Buterin, [Verkle trees](https://vitalik.eth.limo/general/2021/06/18/verkle.html), 2021-06-18.
- Vitalik Buterin, [An approximate introduction to how zk-SNARKs are possible](https://vitalik.eth.limo/general/2021/01/26/snarks.html), 2021-01-26. PCS가 SNARK에서 쓰이는 방식.
- Dan Boneh, Justin Drake, Ben Fisch, Ariel Gabizon, [Efficient polynomial commitment schemes for multiple points and polynomials](https://eprint.iacr.org/2020/081), ePrint 2020/081.
- [EIP-6800: Ethereum state using a unified verkle tree](https://eips.ethereum.org/EIPS/eip-6800).
- [EIP-7864 논의 글: Ethereum state using a unified binary tree](https://ethereum-magicians.org/t/eip-7864-ethereum-state-using-a-unified-binary-tree/22611).
- [go-ipa multiproof package](https://pkg.go.dev/github.com/crate-crypto/go-ipa).

---

## 부록

### A. Group(군)

**Binary Operation(이항 연산)**

집합 $S$ 위의 이항 연산은 함수 $*: S \times S \rightarrow S$다. 두 원소 $(a, b)$를 하나의 원소 $a * b$로 보낸다. 결과가 항상 $S$ 안에 있어야 한다는 조건이 정의에 포함되어 있다([Binary operation](https://en.wikipedia.org/wiki/Binary_operation)).

**Group(군)의 정의**

공집합이 아닌 집합 $G$와 이항 연산 $*$가 다음을 만족하면 $(G, *)$를 Group이라 한다.

- **Closure(닫힘성)**: 모든 $a, b \in G$에 대해 $a * b \in G$다. 이항 연산의 정의에 이미 포함되지만 보통 따로 적는다. 예: 정수 두 개를 더하면 정수다.
- **Associativity(결합법칙)**: 모든 $a, b, c \in G$에 대해 $(a * b) * c = a * (b * c)$다.
- **Identity Element(항등원)**: 모든 $a \in G$에 대해 $e * a = a * e = a$인 $e \in G$가 있다. 예: 정수 덧셈에서는 0, 0이 아닌 유리수의 곱셈에서는 1.
- **Inverse Element(역원)**: 모든 $a \in G$에 대해 $a * b = b * a = e$인 $b \in G$가 있다. 예: 덧셈에서는 $-a$, 곱셈에서는 $a^{-1}$.

연산 순서를 바꿔도 결과가 같은 군($a * b = b * a$)을 **Abelian Group(아벨 군)**이라 한다. 이 문서의 군은 모두 Abelian Group이다.

**표기법.** 연산을 $+$로 쓰면 항등원은 $0$(타원곡선에서는 $\mathcal{O}$), 역원은 $-a$, $a$를 $k$번 더한 것은 $ka$로 쓴다. 연산을 곱셈으로 쓰면 항등원은 $1$, $k$번 곱한 것은 $a^k$로 쓴다. 같은 구조를 다르게 적은 것일 뿐이다.

**Order(위수)와 Cyclic Group(순환군)**

- 군의 Order는 원소의 개수다.
- 어떤 원소 $G$의 배수만으로 군 전체를 만들 수 있으면 그 군을 Cyclic Group, $G$를 **Generator(생성원)**라 한다.
- Order가 $p$인 Cyclic Group은 다음과 같다.

$$
\mathbb{G} = \{\mathcal{O}, G, 2G, 3G, \ldots, (p-1)G\}, \qquad pG = \mathcal{O}
$$

$(p-1)G$에 $G$를 한 번 더 더하면 항등원 $\mathcal{O}$로 돌아온다. 따라서 $aG$에서 스칼라 $a$는 $p$로 나눈 나머지로만 의미가 있고, 스칼라를 $\mathbb{F}_p$의 원소로 볼 수 있다. KZG의 $[a]_1 = a\,g_1$이 바로 이 구조다. Order가 소수인 군에서는 항등원이 아닌 모든 원소가 Generator다.

### B. Field(체)와 Finite Field(유한체)

**Field(체)**는 덧셈과 곱셈 두 연산이 있고, 다음을 만족하는 집합이다.

- 덧셈에 대해 Abelian Group이다(항등원 0).
- 0을 제외한 원소들이 곱셈에 대해 Abelian Group이다(항등원 1). 즉, 0이 아닌 모든 원소에 곱셈 역원이 있어 나눗셈이 가능하다.
- 분배법칙 $a(b + c) = ab + ac$가 성립한다.

유리수, 실수가 Field의 예다. 원소가 유한한 Field를 **Finite Field(유한체)**라 한다. 원소 개수는 항상 소수의 거듭제곱 $p^n$이다. 이 문서는 $n = 1$인 경우, 즉 $p$가 소수일 때 $0$부터 $p-1$까지의 정수에 대해 $p$로 나눈 나머지로 연산하는 $\mathbb{F}_p$만 사용한다. $\mathrm{GF}(p)$(Galois Field, 갈루아 체)로도 쓴다.

**예: $\mathbb{F}_{13} = \{0, 1, \ldots, 12\}$**

- $10 + 5 = 15 \equiv 2$
- $3 - 10 = -7 \equiv 6$
- $5 \times 4 = 20 \equiv 7$
- $1 / 2 = 7$ ($2 \times 7 = 14 \equiv 1$이므로 2의 곱셈 역원은 7)

**암호학에서 Finite Field를 쓰는 이유**

- **정확한 계산**: 실수는 컴퓨터에서 근사값으로만 다룰 수 있어 오차가 생긴다. $\mathbb{F}_p$의 연산은 오차 없이 정확하다.
- **고정 크기 인코딩**: 모든 원소가 $0$ 이상 $p$ 미만이므로 고정된 바이트 수로 저장할 수 있다.
- **균등 표본 추출**: 유한 집합이므로 "균등하게 무작위로 고른다"가 잘 정의된다. Challenge와 Schwartz-Zippel Lemma의 확률 계산이 이 성질에 의존한다.
- **어려운 문제의 존재**: Finite Field 위에서 정의한 군에서는 Discrete Logarithm Problem 같은 계산 문제가 어렵다고 알려져 있다([부록 D](#d-elliptic-curve-group타원곡선-군과-discrete-logarithm-problem이산-로그-문제)).

### C. Polynomial(다항식)과 Rational Function(유리함수)

$\mathbb{F}_p$ 위의 **Polynomial(다항식)**은 $f(X) = a_0 + a_1 X + \ldots + a_n X^n$ ($a_j \in \mathbb{F}_p$) 꼴의 식이다. $a_n \ne 0$일 때 Degree(차수)는 $n$이다.

- **Root(근)**: $f(z) = 0$인 $z$.
- **Factor Theorem(인수 정리)**: $f(z) = 0 \iff (X - z)$가 $f(X)$를 나누어떨어지게 한다. 2.1절의 나머지 정리에서 $y = 0$인 경우다.
- **근의 개수**: 0이 아닌 차수 $n$ 다항식은 Field 위에서 근이 최대 $n$개다. 근 하나마다 $(X - z)$ 인수가 하나씩 생기는데, 차수가 $n$이므로 이런 인수를 $n$개보다 많이 가질 수 없기 때문이다. Soundness 계산의 기초가 되는 성질이다.

**Rational Function(유리함수)**은 $\frac{a(X)}{b(X)}$ ($b \ne 0$) 꼴의 함수다. 서로 다른 점 $z$들에 대해 "다항식 $+ \sum_z \frac{c_z}{X - z}$" 꼴로 쓴 표현은 유일하다. 따라서 이 함수가 다항식이려면 모든 $c_z$가 0이어야 한다. 4.2절에서 이 성질을 사용했다.

### D. Elliptic Curve Group(타원곡선 군)과 Discrete Logarithm Problem(이산 로그 문제)

**Elliptic Curve(타원곡선)**는 Finite Field $\mathbb{F}_q$ 위에서 $y^2 = x^3 + ax + b$ 같은 방정식을 만족하는 점 $(x, y)$들의 집합이다. 여기에 Point at Infinity(무한원점) $\mathcal{O}$를 더하고, 기하학적으로 정의된 점 덧셈을 연산으로 주면 Abelian Group이 된다. 이를 **Elliptic Curve Group(타원곡선 군)**이라 한다.

- 점의 좌표는 Base Field(기저체) $\mathbb{F}_q$의 원소다.
- 암호학에서는 보통 Order가 큰 소수 $p$인 부분군을 쓴다. 스칼라는 이 $p$에 대한 Scalar Field(스칼라 체) $\mathbb{F}_p$의 원소다. 다항식 계수와 Challenge가 사는 체가 이 스칼라 체다.
- EIP-6800 명세의 `BANDERSNATCH_MODULUS`는 주석에 Bandersnatch 곡선의 군 위수(Curve Order)로 적혀 있다.

**Scalar Multiplication(스칼라 곱셈)**: 스칼라 $x$와 점 $G$에 대해 $xG$는 Double-and-Add(두 배와 덧셈 반복) 방식으로 빠르게 계산할 수 있다.

**Discrete Logarithm Problem(이산 로그 문제)**: $G$와 $xG$가 주어졌을 때 $x$를 구하는 문제다. 적절히 고른 곡선에서는 현재 알려진 고전 컴퓨터 알고리즘으로 효율적으로 풀 수 없다고 여겨진다. 곱셈 표기 군에서는 "$g$와 $g^x \bmod p$에서 $x$를 구하는 문제"와 같은 구조다. 양자 컴퓨터의 Shor 알고리즘은 이 문제를 효율적으로 풀 수 있으므로, 이에 의존하는 Pedersen Commitment와 KZG는 충분히 큰 양자 컴퓨터가 생기면 안전하지 않다. 6.5절의 Ethereum 논의가 이 점과 관련된다.

### E. Bilinear Pairing(쌍선형 페어링)

Bilinear Pairing은 세 개의 Order $p$ 군 $\mathbb{G}_1$, $\mathbb{G}_2$, $\mathbb{G}_T$ 사이의 함수 $e: \mathbb{G}_1 \times \mathbb{G}_2 \rightarrow \mathbb{G}_T$로 다음을 만족한다.

- **Bilinearity(쌍선형성)**: 모든 $P \in \mathbb{G}_1$, $Q \in \mathbb{G}_2$, $a, b \in \mathbb{F}_p$에 대해 $e(aP, bQ) = e(P, Q)^{ab}$.
- **Non-degeneracy(비퇴화성)**: $e(g_1, g_2) \ne 1$.
- 효율적으로 계산할 수 있다.

Pairing을 쓰면 $[a]_1$, $[b]_2$, $[c]_1$만 보고 $ab = c$인지 확인할 수 있다. $e([a]_1, [b]_2) = e([c]_1, [1]_2)$인지 보면 되기 때문이다. KZG 검증은 이 성질로 "$f(s) - y = q(s)(s - z)$"를 $s$를 모른 채 확인한다. Pairing을 효율적으로 계산할 수 있는 곡선을 Pairing-friendly Curve(페어링 친화 곡선)라 하며, BLS12-381이 대표적인 예다.

### F. Roots of Unity(단위근)와 Lagrange Interpolation(라그랑주 보간)

**Root of Unity(단위근)**: $\omega^d = 1$이고 $d$보다 작은 양의 정수 $k$에 대해 $\omega^k \ne 1$인 $\omega$를 원시 $d$차 단위근이라 한다. $\mathbb{F}_p$에서 원시 $d$차 단위근은 $d$가 $p - 1$을 나눌 때 존재한다. 이때 $\omega^0, \omega^1, \ldots, \omega^{d-1}$은 서로 다르고 다음이 성립한다.

$$
\prod_{i=0}^{d-1} (X - \omega^i) = X^d - 1
$$

FFT를 쓸 수 있고, 7.3절의 공식이 단순해지는 이유가 이 성질이다.

**Lagrange Interpolation(라그랑주 보간)**: 서로 다른 $d$개의 점 $x_0, \ldots, x_{d-1}$과 값 $v_0, \ldots, v_{d-1}$이 주어지면, $f(x_j) = v_j$를 만족하는 차수 $d$ 미만의 다항식은 정확히 하나다. **Lagrange Basis Polynomial(라그랑주 기저 다항식)**

$$
\ell_j(X) = \prod_{i \ne j} \frac{X - x_i}{x_j - x_i}
$$

은 $x_j$에서 1, 다른 $x_i$에서 0이다. 따라서 그 다항식은 다음과 같다.

$$
f(X) = \sum_{j=0}^{d-1} v_j\,\ell_j(X)
$$

PCS를 Vector Commitment로 쓸 때(1.3절)와 Evaluation Form으로 Commit할 때(7.3절) 이 식을 사용한다. [Lagrange polynomial](https://en.wikipedia.org/wiki/Lagrange_polynomial)에 자세한 설명이 있다.

### G. Schwartz-Zippel Lemma(슈바르츠-지펠 보조정리)

[Schwartz-Zippel Lemma](https://en.wikipedia.org/wiki/Schwartz%E2%80%93Zippel_lemma)는 0이 아닌 다항식이 무작위 점에서 우연히 0이 될 확률의 상한을 준다. 이 문서에서 쓰는 일변수 형태는 다음과 같다.

> 0이 아닌 차수 $D$ 이하의 다항식 $R(X)$와 유한 집합 $S \subseteq \mathbb{F}_p$에 대해, $t$를 $S$에서 균등하게 뽑으면 $\Pr[R(t) = 0] \le \dfrac{D}{|S|}$이다.

근이 최대 $D$개이므로([부록 C](#c-polynomial다항식과-rational-function유리함수)) 바로 나오는 결과다. 다변수 형태는 변수가 여러 개인 다항식에 대해 같은 상한을 준다.

이 보조정리는 "두 다항식이 같은지 확인하려면 무작위 점 하나에서 값을 비교하면 된다"는 원리의 근거다. 두 다항식이 다르면 차이 다항식은 0이 아니므로, 무작위 점에서 값이 같을 확률은 (차수) / (선택 가능한 점의 수) 이하다. $p$가 $2^{252}$ 이상이고 차수가 수천이면 이 확률은 무시할 수 있다. Multiproof에서는 $r$ 단계(4.2절, $S = \mathbb{F}_p$)와 $t$ 단계(4.2절, $S = \mathbb{F}_p \setminus Z$)에서 사용했다.