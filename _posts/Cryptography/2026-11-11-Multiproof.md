---
title: "Multiproof"
date: 2026-11-11
categories: [Cryptography]
tags: [Cryptography, PCS, KZG]
math: true
---

## 1. What is Multiproof?

**Multiproof(다중 증명)**는 여러 검증 명제를 하나의 증명으로 결합해 검증하는 기법이다. 어떤 명제를 결합하는지는 사용하는 암호학적 방식에 따라 달라진다.

이 문서에서는 **Polynomial Commitment Scheme(다항식 커밋먼트 스킴, PCS)**에서 여러 다항식의 평가 주장을 결합하는 **Dankrad Feist의 Random Evaluation 방식**을 설명한다. 여기서 Polynomial Commitment는 다항식 전체를 공개하지 않고도 특정 점에서의 평가값이 맞는지 검증할 수 있게 하는 Commitment다.

개별적으로 검증하려는 주장은 다음과 같다.

$$
f_0(z_0)=y_0,\quad f_1(z_1)=y_1,\quad \ldots,\quad f_{m-1}(z_{m-1})=y_{m-1}
$$

각 $f_i(X)$에 대한 Commitment를 $C_i$라고 하자. Verifier(검증자)는 $C_i$, $z_i$, $y_i$를 알고 있으며, Prover(증명자)는 다항식 $f_i(X)$에 접근할 수 있다고 가정한다.

**목표는 $m$개 평가 주장을 각각 별도로 증명하는 대신 하나의 결합된 Opening Proof로 확인하는 것이다.** Opening(평가값 증명)은 Commitment에 대응하는 다항식이 어떤 점 $z$에서 값 $y$를 가진다는 주장을 증명하는 절차다.

## 2. 왜 필요한가?

Verkle Tree(버클 트리)는 부모 노드의 여러 자식 값을 Polynomial Commitment 또는 이를 기반으로 한 Vector Commitment로 표현하는 트리 구조다.

특정 Key의 값을 검증하려면 Root부터 해당 Leaf까지 각 단계에서 다음 관계가 올바른지 확인해야 한다.

> 부모 노드가 Commit한 다항식의 특정 위치 값이 실제로 다음 자식 노드를 나타내는가?

각 단계는 $f_i(z_i)=y_i$ 형태의 Opening으로 표현할 수 있다. 여러 Key를 함께 증명하면 검증할 Opening의 수도 늘어난다. 각 Opening에 별도 증명을 붙이면 그만큼 증명 데이터가 커진다.

Multiproof는 **여러 Opening을 하나의 검증 관계로 압축**해 이 비용을 줄인다. 다만 Verkle Tree의 전체 Witness(검증에 필요한 데이터)에는 여전히 경로상의 Commitment와 필요한 Key 및 Value가 포함된다. **결합된 증명의 크기가 일정하다는 것과 전체 Witness의 크기가 일정하다는 것은 다르다.**

## 3. 핵심 아이디어: 올바른 Opening은 나누어떨어진다

Polynomial Remainder Theorem(다항식 나머지 정리)에 따르면

$$
f(z)=y\iff f(X)-y=(X-z)q(X)
$$

를 만족하는 다항식 $q(X)$가 존재한다. 따라서

$$
q(X)=\frac{f(X)-y}{X-z}
$$

가 다항식이면 $f(z)=y$라는 주장이 참이다.

예를 들어 $f(X)=X^2+X+3$이면 $f(2)=9$다.

$$
\frac{f(X)-9}{X-2}=X+3
$$

반면 $f(2)=10$이라고 주장하면

$$
\frac{f(X)-10}{X-2}=X+3-\frac{1}{X-2}
$$

가 되어 다항식이 아닌 Rational Function(유리함수)이 남는다.

Multiproof는 이 성질을 여러 Opening에 동시에 적용한다. 모든 $f_i(z_i)=y_i$가 참이면 개별 몫을 더한 결과도 다항식이 된다. 다만 거짓 Opening들의 나머지가 서로 상쇄될 가능성이 있으므로 **무작위 가중치**를 사용한다.

## 4. 프로토콜의 동작 원리

다음 계산은 소수 $p$에 대한 Finite Field(유한체) $\mathbb{F}_p$에서 수행한다. 분모로 나누는 연산은 일반적인 실수 나눗셈이 아니라, 0이 아닌 체 원소의 역원을 곱하는 연산이다. 다항식의 차수는 $d$ 미만이라고 가정한다.

### 4.1. 첫 번째 Challenge $r$: 여러 몫을 결합

Prover와 Verifier는 모든 평가 주장과 Commitment가 고정된 뒤, 같은 입력을 Hash해 Challenge(예측하기 어려운 검증용 값) $r$을 얻는다.

$$
r\leftarrow H(C_0,\ldots,C_{m-1},\;z_0,\ldots,z_{m-1},\;y_0,\ldots,y_{m-1})
$$

Prover는 개별 몫을 $r$의 거듭제곱으로 가중해 결합한다.

$$
\boxed{g(X)=\sum_{i=0}^{m-1}r^i\frac{f_i(X)-y_i}{X-z_i}}
$$

모든 Opening이 참이면 $g(X)$는 다항식이다. Prover는 이 다항식의 Commitment $D$를 계산해 보낸다.

$$
D=\operatorname{Commit}(g)
$$

그러나 **$D$를 보냈다는 사실만으로 $D$가 정말 위 $g(X)$의 Commitment인지는 알 수 없다.** 부정직한 Prover가 다른 다항식에 대한 Commitment를 보낼 수도 있기 때문이다.

### 4.2. 두 번째 Challenge $t$: $D$가 올바른지 확인

$D$를 먼저 고정한 다음 두 번째 Challenge를 얻는다.

$$
t\leftarrow H(r,D)
$$

$t$는 모든 Opening의 평가점 $z_i$와 달라야 한다. $t=z_i$가 나오면 아래 식의 분모가 0이 되므로, 구현에서는 명세에 따른 예외 처리 또는 재추출이 필요하다.

확인하려는 값은 $g(t)$다.

$$
g(t)=\sum_{i=0}^{m-1}r^i\frac{f_i(t)-y_i}{t-z_i}
$$

하지만 Verifier는 $f_i(X)$ 전체를 모르므로 $f_i(t)$를 직접 계산할 수 없다. 이를 해결하기 위해 식을 다음 두 항으로 나눈다.

$$
\begin{aligned}
g(t)
&=\underbrace{\sum_{i=0}^{m-1}\frac{r^i}{t-z_i}f_i(t)}_{h(t)}
-\underbrace{\sum_{i=0}^{m-1}\frac{r^i y_i}{t-z_i}}_{y}
\end{aligned}
$$

여기서 $y$는 공개된 $r,t,z_i,y_i$만으로 계산할 수 있다.

$$
\boxed{y=\sum_{i=0}^{m-1}\frac{r^i y_i}{t-z_i}}
$$

첫 번째 항에 대응하는 다항식은 다음과 같다.

$$
h(X)=\sum_{i=0}^{m-1}\frac{r^i}{t-z_i}f_i(X)
$$

PCS의 **Additive Homomorphism(덧셈 동형성)**은 다항식의 선형 결합에 대한 Commitment를 기존 Commitment의 선형 결합으로 계산할 수 있다는 성질이다.

$$
\operatorname{Commit}(af+bg)=a\operatorname{Commit}(f)+b\operatorname{Commit}(g)
$$

따라서 Verifier는 $f_i(X)$를 몰라도 $h(X)$의 Commitment $E$를 직접 계산한다.

$$
\boxed{E=\sum_{i=0}^{m-1}\frac{r^i}{t-z_i}C_i=\operatorname{Commit}(h)}
$$

이제 여러 평가 주장을 검증하는 문제는 **Commitment $E-D$에 대응하는 다항식의 $t$에서의 값이 $y$인지 검증하는 Opening 하나**로 바뀐다.

$$
\boxed{h(t)-g(t)=y}
$$

정직한 Prover는 다음 몫 다항식을 계산한다.

$$
w(X)=\frac{h(X)-g(X)-y}{X-t}
$$

그리고 $w(X)$의 Opening Proof $\pi$를 만든다. 최종적으로 전송하는 결합 증명은 **$(D,\pi)$**다. $E$와 $y$는 Verifier가 계산하므로 증명에 추가하지 않는다.

### 4.3. KZG로 최종 Opening 검증

KZG Commitment에서는 비밀 설정값 $s$를 직접 알지 않고도 공개 매개변수로 $[f(s)]_1$을 계산한다. $[a]_1$과 $[a]_2$는 스칼라 $a$를 각각 두 타원곡선 군의 생성원에 곱한 원소를 뜻한다. $e$는 두 군 원소 사이의 곱셈 관계를 확인할 수 있는 Bilinear Pairing(쌍선형 페어링)이다.

KZG를 사용하면

$$
D=[g(s)]_1,\qquad E=[h(s)]_1,\qquad \pi=[w(s)]_1
$$

이고, 정의에 따라

$$
h(X)-g(X)-y=(X-t)w(X)
$$

이므로 Verifier는 다음 등식을 확인한다.

$$
\boxed{e(E-D-[y]_1,[1]_2)=e(\pi,[s-t]_2)}
$$

이 등식이 최종적으로 확인하는 것은 **$E-D$에 대응하는 다항식이 점 $t$에서 값 $y$를 갖는가**이다. Pairing 등식 하나로 검증하지만, 등식의 양변에 Pairing 연산이 각각 들어간다는 점은 구분해야 한다.

KZG에서 $D$와 $\pi$는 각각 군 원소 하나다. 따라서 **핵심 Multiproof의 크기는 Opening 개수 $m$에 의존하지 않는다.** 다른 PCS를 쓰면 마지막 Opening Proof의 형식과 크기가 달라질 수 있다.

## 5. 왜 검증이 성립하는가?

### 5.1. 모든 Opening이 올바른 경우

모든 $f_i(z_i)=y_i$가 참이면 $g(X)$는 다항식이다. 식을 그대로 정리하면

$$
\begin{aligned}
h(t)-g(t)
&=\sum_i\frac{r^if_i(t)}{t-z_i}
-\sum_i\frac{r^i(f_i(t)-y_i)}{t-z_i}\\
&=\sum_i\frac{r^iy_i}{t-z_i}=y
\end{aligned}
$$

이므로 $h(X)-g(X)-y$는 $(X-t)$로 나누어떨어진다. 정직한 Prover가 만든 $\pi$는 KZG 검증을 통과한다. 이를 **Completeness(완전성)**라고 한다. 단, $t=z_i$인 예외는 앞서 설명한 방식으로 처리해야 한다.

### 5.2. 거짓 Opening을 섞은 경우

$i$번째 주장의 오류를 $\delta_i=f_i(z_i)-y_i$라고 하자. 그러면

$$
\frac{f_i(X)-y_i}{X-z_i}
=q_i(X)+\frac{\delta_i}{X-z_i}
$$

이고, $q_i(X)$는 다항식이다. 거짓 주장이 있으면 $\delta_i\ne0$인 항이 존재한다.

서로 같은 평가점 $z$에서 발생하는 나머지들의 합은 다음과 같다.

$$
\frac{P_z(r)}{X-z},\qquad
P_z(r)=\sum_{i:z_i=z}\delta_i r^i
$$

$r$ 없이 더하면 서로 다른 오류가 상쇄될 수 있다. 반면 모든 주장을 고정한 뒤 무작위 $r$을 선택하면, 거짓 주장이 있는 평가점에서 $P_z(r)=0$이 우연히 성립할 확률은 작다. 이상적으로 $r$을 $\mathbb{F}_p$에서 균등하게 뽑는다면 그 확률은 최대 $(m-1)/p$다.

그런데 Prover가 다항식이 아닌 $g$ 대신 임의의 다항식에 대한 Commitment $D$를 보낼 수도 있다. 그래서 **$D$를 정한 뒤 $t$를 선택**한다. 잘못된 다항식 관계가 무작위 점 $t$에서 우연히 일치할 가능성은 다항식의 근 개수로 제한된다. 이 원리가 Schwartz-Zippel Lemma(슈바르츠-지펠 보조정리)다.

즉, $r$은 **오류의 상쇄를 억제**하고, $t$는 **$D$가 주장된 관계를 만족하는지 검사**한다. 실제 Soundness(건전성)는 이 확률적 논리뿐 아니라 PCS의 Binding(커밋한 내용을 다른 내용으로 열기 어렵다는 성질), Opening Proof의 안전성, Fiat-Shamir Transform의 안전한 구현을 전제로 한다. Hash에서 Challenge를 만드는 방식은 두 참여자가 직접 난수를 주고받지 않도록 한다.

## 6. 계산 예시

두 개의 Opening을 증명한다고 하자.

$$
\begin{aligned}
f_0(X)&=X+2,&z_0&=3,&y_0&=5\\
f_1(X)&=X^2+1,&z_1&=2,&y_1&=5
\end{aligned}
$$

실제로 $f_0(3)=5$, $f_1(2)=5$다. 계산을 보여 주기 위해 $r=2$, $t=4$로 정한다. 실제 프로토콜에서는 이 값들을 Hash로 도출한다.

**① 개별 몫을 결합한다.**

$$
\begin{aligned}
q_0(X)&=\frac{X+2-5}{X-3}=1\\
q_1(X)&=\frac{X^2+1-5}{X-2}=X+2\\
g(X)&=q_0(X)+2q_1(X)=2X+5
\end{aligned}
$$

**② $t=4$에서 필요한 값을 계산한다.**

$$
\begin{aligned}
h(X)&=\frac{1}{4-3}f_0(X)+\frac{2}{4-2}f_1(X)\\
&=X^2+X+3\\
y&=\frac{5}{4-3}+\frac{2\cdot5}{4-2}=10
\end{aligned}
$$

검증해야 할 관계는 실제로 성립한다.

$$
h(4)-g(4)=23-13=10=y
$$

**③ 최종 Opening Proof를 만든다.**

$$
\begin{aligned}
w(X)&=\frac{h(X)-g(X)-10}{X-4}\\
&=\frac{X^2-X-12}{X-4}=X+3
\end{aligned}
$$

KZG라면 $D=[2s+5]_1$, $\pi=[s+3]_1$을 보낸다. Verifier는 기존 $C_0,C_1$로 $E=C_0+C_1$을 계산한 뒤 다음 관계를 검증한다.

$$
\begin{aligned}
e(E-D-[10]_1,[1]_2)
&=e([s^2-s-12]_1,[1]_2)\\
&=e([s+3]_1,[s-4]_2)\\
&=e(\pi,[s-4]_2)
\end{aligned}
$$

즉, **두 개의 Opening을 $(D,\pi)$ 하나의 결합 증명으로 확인했다.**

## 7. Verkle Tree에서 Proof에 포함되는 것

Verkle Tree에 Multiproof를 적용할 때 전체 Witness의 주요 구성은 다음과 같다.

- **Key와 Value 및 필요한 Leaf 정보:** 검증하려는 상태 데이터
- **경로상의 Commitment:** Root부터 해당 Leaf까지의 연결을 검증하는 데 필요한 Commitment. 경로가 겹치면 공유 가능
- **핵심 Multiproof:** KZG 기준 $(D,\pi)$

Root Commitment는 Verifier가 이미 신뢰하는 값이라면 다시 보내지 않아도 된다. 평가점 $z_i$는 Key와 경로로부터, 평가값 $y_i$는 자식 Commitment 또는 Leaf 데이터의 **트리 명세별 인코딩**으로부터 재구성할 수 있다. 따라서 이 값들을 항상 별도로 전송할 필요는 없다.

Prover는 증명에 필요한 각 다항식의 전체 계수 또는 평가값을 재구성할 데이터에 접근해야 한다. 그렇다고 블록체인의 전체 상태를 반드시 보유해야 하는 것은 아니다.

## 핵심 정리

Multiproof의 원리는 다음과 같다.

1. 각 Opening $f_i(z_i)=y_i$를 나누어떨어짐의 문제로 바꾼다.
2. Challenge $r$로 여러 몫을 결합해 $g(X)$를 만들고 Commitment $D$를 보낸다.
3. $D$가 고정된 뒤 Challenge $t$를 선택한다.
4. Verifier는 기존 Commitment들을 선형 결합하여 $E$를 계산한다.
5. **$E-D$를 $t$에서 열었을 때 $y$가 나오는지** 단일 Opening Proof로 확인한다.

따라서 **여러 평가 관계를 하나의 검증 관계로 환원하는 것**이 이 Multiproof의 핵심이다. 이는 Zero-Knowledge(영지식)를 자동으로 제공하는 기술은 아니다.

## 참고 자료

- Dankrad Feist, [PCS multiproofs using random evaluation](https://dankradfeist.de/ethereum/2021/06/18/pcs-multiproofs.html), 2021-06-18.
- Vitalik Buterin, [Verkle trees](https://vitalik.eth.limo/general/2021/06/18/verkle.html), 2021-06-18.
