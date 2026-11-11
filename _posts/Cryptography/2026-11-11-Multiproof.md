---
title: "Multiproof"
date: 2026-11-11
categories: [Cryptography]
tags: [Cryptography, PCS, KZG]
math: true
---

## What is Multiproof?

증명을 여러 개 만들어야 하는 상황을 생각해 보자. 어떤 값이 맞는지 보이는 증명이 하나 있고, 같은 종류의 증명이 수천 개 필요하다. 하나씩 만들면 증명 데이터가 수천 배로 늘어난다. **Multiproof(다중 증명)**는 이렇게 같은 종류의 주장 여러 개를 **증명 하나로 묶어서 한 번에 검증하는 방법**이다.

이 글에서 묶는 주장은 "어떤 다항식이 어떤 점에서 어떤 값을 가진다"는 주장이다. 이 주장이 무엇이고 왜 증명이 필요한지부터 보고, 그다음에 여러 개를 하나로 묶는 방법을 설명한다. 다루는 방식은 Dankrad Feist가 제안한 Random Evaluation(무작위 평가) 방식이다.

## Polynomial Commitment and Opening

다항식 $f(X)$를 가진 사람(Prover, 증명자)이 있고, 그 다항식을 모르는 사람(Verifier, 검증자)이 있다. Prover는 다항식을 공개하지 않으면서도 "$f(z)=y$다"라는 사실만 Verifier에게 납득시키고 싶다.

이를 위해 Prover는 먼저 다항식을 **Commit(커밋)**한다. Commit은 다항식 전체를 고정 크기의 값 하나로 압축하는 것이고, 그 결과를 **Commitment(커밋먼트)** $C$라고 한다. Commitment는 두 성질을 가진다.

- $C$만 보고는 $f(X)$를 알 수 없다.
- $C$를 보낸 뒤에는 $f(X)$를 다른 다항식으로 바꿀 수 없다. 이 성질을 Binding(구속성)이라고 한다.

Commitment를 보낸 다음, Prover는 특정 점에서의 값을 공개한다. 이것을 **Opening(열기)**이라고 한다. 점 $z$에서 연다는 것은 "$f(z)=y$다"라는 값 $y$와, 그 값이 $C$에 묶인 다항식에서 나온 것임을 보이는 증명을 함께 보내는 것이다. 이 증명을 **Opening Proof(열기 증명)**라고 한다. Verifier는 $C$, $z$, $y$, Opening Proof만으로 주장을 확인한다. $f(X)$ 자체는 끝까지 보지 않는다.

이 구조 전체, 즉 Commit과 Opening과 그 검증 방법을 묶어 **Polynomial Commitment Scheme(다항식 커밋먼트 스킴, PCS)**이라고 한다.

이제 Multiproof의 문제를 정확히 적을 수 있다. 다항식 $m$개 $f_0, \ldots, f_{m-1}$와 각각의 Commitment $C_0, \ldots, C_{m-1}$이 있다. Prover는 다음 $m$개의 Opening을 한꺼번에 증명하고 싶다.

$$
f_0(z_0)=y_0,\quad f_1(z_1)=y_1,\quad \ldots,\quad f_{m-1}(z_{m-1})=y_{m-1}
$$

Verifier는 $C_i$, $z_i$, $y_i$를 모두 알고 있다. 목표는 Opening Proof를 $m$개 보내는 대신 **하나만 보내는 것**이다.

## Why Multiproof?

이 문제가 실제로 생기는 곳이 Verkle Tree(버클 트리)다. Verkle Tree는 부모 노드가 자식들을 Hash(해시) 대신 Commitment 하나로 요약하는 트리다. 자식 값 $v_0, \ldots, v_{255}$가 있으면, $f(i)=v_i$를 만족하는 다항식 $f$를 만들고 그 Commitment를 부모 노드의 값으로 삼는다. 이렇게 하면 "$i$번째 자식이 $v_i$다"라는 말이 "$f(i)=v_i$다"라는 Opening이 된다.

트리에서 Leaf 하나의 값이 맞는지 확인하려면, Root부터 그 Leaf까지 내려오는 경로의 각 단계에서 "부모를 이 위치에서 열면 이 자식이 나온다"를 확인해야 한다. 깊이가 3이면 Opening이 3개다. 블록 하나는 상태 값을 수천 개 읽으므로, Opening이 수천 개에서 수만 개가 된다. Opening Proof를 하나씩 보내면 그 수만큼 증명 데이터가 커진다.

Multiproof는 이 Opening들을 Opening Proof 하나로 묶는다. 다만 묶이는 것은 증명뿐이다. 경로 위의 Commitment와 읽은 값들은 그대로 보내야 한다. 이 점은 8장에서 다시 본다.

## Key Idea

한 점에서의 Opening이 왜 증명 가능한지부터 보자. 출발점은 다항식에 대한 다음 사실이다.

$$
f(z)=y \quad\iff\quad f(X)-y \text{가 } (X-z)\text{로 나누어떨어진다}
$$

$f(z)=y$이면 $f(X)-y$는 $X=z$에서 0이 되므로 $(X-z)$를 인수로 가진다. 거꾸로 $(X-z)$를 인수로 가지면 $X=z$를 넣었을 때 0이므로 $f(z)=y$다. 이 사실을 Polynomial Remainder Theorem(다항식 나머지 정리)이라고 한다.

예를 들어 $f(X)=X^2+X+3$이면 $f(2)=9$다. 실제로 나누어떨어진다.

$$
\frac{f(X)-9}{X-2}=\frac{(X+3)(X-2)}{X-2}=X+3
$$

반면 $f(2)=10$이라고 거짓 주장을 하면 나머지가 남는다.

$$
\frac{f(X)-10}{X-2}=X+3-\frac{1}{X-2}
$$

따라서 "$f(z)=y$다"를 증명하는 것은 **"몫 $q(X)=\dfrac{f(X)-y}{X-z}$가 다항식이다"를 증명하는 것**과 같다. 이것이 Opening Proof의 정체다. Prover는 몫 $q(X)$의 Commitment를 보내고, Verifier는 그것이 정말 $f(X)-y$를 $(X-z)$로 나눈 몫인지 확인한다. 확인하는 방법은 PCS마다 다르며, 5.3에서 KZG의 방법을 본다.

이제 Opening이 $m$개다. 몫도 $m$개 생긴다.

$$
q_i(X)=\frac{f_i(X)-y_i}{X-z_i},\qquad i=0,\ldots,m-1
$$

모든 주장이 참이면 몫 $m$개가 전부 다항식이고, 그러면 **전부 더한 것도 다항식**이다. 반대로 어느 하나가 거짓이면 그 몫에는 $\dfrac{\delta}{X-z}$ 꼴의 나머지가 남는다. 그래서 Multiproof의 핵심 아이디어는 다음 한 줄이다.

> 몫 $m$개를 더해서 하나로 만들고, 그 합이 다항식임을 한 번만 증명한다.

그런데 그냥 더하면 문제가 있다. 거짓 주장이 둘 이상이면 나머지들이 서로 상쇄되어 합이 우연히 다항식이 될 수 있다. 예를 들어 $\dfrac{1}{X-2}$와 $\dfrac{-1}{X-2}$를 더하면 0이다. 이를 막기 위해 각 몫에 **Prover가 미리 알 수 없는 무작위 가중치**를 곱해서 더한다. 가중치를 모르는 상태에서 상쇄가 일어나도록 거짓 값을 맞추는 것은 사실상 불가능하다.

## Protocol

아래 계산은 소수 $p$에 대한 Finite Field(유한체) $\mathbb{F}_p$에서 한다. 나눗셈은 역원을 곱하는 연산이다. 모든 $f_i(X)$의 차수는 $d$ 미만이라고 가정한다.

### 5.1. Combining Quotients with $r$

무작위 가중치가 필요하다. 그런데 Verifier가 가중치를 골라서 보내면 메시지를 주고받아야 한다. 대신 양쪽이 같은 값을 Hash해서 같은 난수를 얻는다. 이 방법을 Fiat-Shamir Transform(피아트-샤미르 변환)이라고 하고, 이렇게 얻은 난수를 Challenge(챌린지)라고 한다. Hash 입력에는 모든 주장이 들어가므로, Prover는 주장을 고정한 뒤에야 난수를 알 수 있다.

$$
r\leftarrow H(C_0,\ldots,C_{m-1},\;y_0,\ldots,y_{m-1},\;z_0,\ldots,z_{m-1})
$$

Prover는 $i$번째 몫에 $r^i$를 곱해서 더한다.

$$
\boxed{g(X)=\sum_{i=0}^{m-1}r^i\,\frac{f_i(X)-y_i}{X-z_i}}
$$

모든 주장이 참이면 $g(X)$는 다항식이다. Prover는 $g(X)$의 Commitment $D$를 계산해 보낸다.

$$
D=\operatorname{Commit}(g)
$$

여기서 끝나지 않는다. Verifier가 받은 것은 $D$라는 값 하나뿐이다. **이것이 정말 위 $g(X)$의 Commitment인지, 아니면 Prover가 아무 다항식이나 Commit한 것인지 Verifier는 알 수 없다.** 그래서 $D$를 검사하는 단계가 하나 더 필요하다.

### 5.2. Checking $D$ with $t$

$D$가 정말 $g$의 Commitment인지 확인하는 방법은 **무작위 점 $t$에서 $g(t)$를 직접 계산해 보고, $D$를 $t$에서 열었을 때 같은 값이 나오는지 보는 것**이다. 두 다항식이 다르면 무작위 점에서 값이 같을 확률은 매우 작다.

$t$는 $D$가 고정된 뒤에 뽑아야 한다. $D$를 보내기 전에 $t$를 알면 그 점에서만 값을 맞춘 가짜 다항식을 만들 수 있기 때문이다.

$$
t\leftarrow H(r,D)
$$

$g(t)$를 식으로 쓰면 다음과 같다.

$$
g(t)=\sum_{i=0}^{m-1}r^i\,\frac{f_i(t)-y_i}{t-z_i}
$$

문제는 Verifier가 $f_i$를 모르므로 $f_i(t)$를 계산할 수 없다는 것이다. 그래서 식을 $f_i$가 들어간 부분과 들어가지 않은 부분으로 나눈다.

$$
g(t)
=\underbrace{\sum_{i=0}^{m-1}\frac{r^i}{t-z_i}\,f_i(t)}_{h(t)}
-\underbrace{\sum_{i=0}^{m-1}\frac{r^i\,y_i}{t-z_i}}_{y}
$$

뒤쪽 $y$는 $r, t, z_i, y_i$만으로 계산할 수 있으니 Verifier가 직접 구한다.

$$
\boxed{y=\sum_{i=0}^{m-1}\frac{r^i\,y_i}{t-z_i}}
$$

앞쪽은 다항식 $h(X)=\sum_i \frac{r^i}{t-z_i} f_i(X)$를 $t$에서 평가한 값이다. $h$는 $f_i$들의 선형 결합이다. 여기서 PCS의 성질 하나를 쓴다. **Additive Homomorphism(덧셈 동형성)**은 다항식의 선형 결합에 대한 Commitment를 Commitment들의 선형 결합으로 계산할 수 있다는 성질이다.

$$
\operatorname{Commit}(a f+b g)=a\operatorname{Commit}(f)+b\operatorname{Commit}(g)
$$

따라서 Verifier는 $f_i$를 몰라도 $h$의 Commitment $E$를 $C_i$들로 직접 계산할 수 있다.

$$
\boxed{E=\sum_{i=0}^{m-1}\frac{r^i}{t-z_i}\,C_i=\operatorname{Commit}(h)}
$$

이제 확인할 것을 다시 쓰면 $g(t)=h(t)-y$, 즉

$$
\boxed{h(t)-g(t)=y}
$$

이다. $h-g$의 Commitment는 $E-D$다. 그러므로 **$m$개의 Opening을 확인하는 문제가 "$E-D$를 $t$에서 열면 $y$가 나온다"는 Opening 하나로 바뀌었다.** 이 Opening은 4장에서 본 것과 같은 방법으로 증명한다. Prover는 몫

$$
w(X)=\frac{h(X)-g(X)-y}{X-t}
$$

를 계산하고 그 Commitment $\pi=\operatorname{Commit}(w)$를 보낸다. 이 $\pi$가 마지막 Opening Proof다.

Prover가 보내는 것은 **$D$와 $\pi$, 두 개뿐이다.** $E$와 $y$는 Verifier가 계산한다.

$t$가 어떤 $z_i$와 같으면 분모가 0이 된다. $t$는 $\mathbb{F}_p$ 전체에서 뽑히고 $z_i$는 $m$개뿐이므로 그럴 확률은 $m/p$ 이하다. Verkle Tree에서는 $z_i$가 0부터 255까지이고 $p$가 253비트 소수이므로 무시할 수 있다.

### 5.3. Final Verification with KZG

마지막 Opening을 실제로 확인하려면 구체적인 PCS가 필요하다. 여기서는 KZG(Kate-Zaverucha-Goldberg) Commitment를 쓴다.

KZG는 Trusted Setup(신뢰 설정)에서 비밀값 $s$를 하나 뽑고, $[s^0]_1, [s^1]_1, [s^2]_1, \ldots$ 형태의 공개 매개변수만 남긴 뒤 $s$를 폐기한다. $[a]_1$은 Scalar(스칼라) $a$를 타원곡선 군의 생성원에 곱한 원소이고, $[a]_2$는 다른 군에서 같은 것이다. 다항식의 Commitment는 $[f(s)]_1$이다. 다항식의 계수를 공개 매개변수에 곱해서 더하면 $s$를 모르고도 계산할 수 있다.

KZG를 쓰면

$$
D=[g(s)]_1,\qquad E=[h(s)]_1,\qquad \pi=[w(s)]_1
$$

이다. $w$의 정의에서 $h(X)-g(X)-y=(X-t)\,w(X)$이므로, $X=s$를 넣으면 $h(s)-g(s)-y=(s-t)\,w(s)$가 성립해야 한다. 좌변은 $E-D-[y]_1$에 들어 있고, 우변은 $\pi$에 $(s-t)$를 곱한 것이다. 그런데 $s$를 아무도 모르므로 곱셈을 직접 할 수 없다. 이때 쓰는 것이 Bilinear Pairing(쌍선형 페어링) $e$다. Pairing은 두 군의 원소를 받아 지수끼리 곱한 관계를 확인하게 해 준다. Verifier는 다음 등식을 확인한다.

$$
\boxed{e\big(E-D-[y]_1,\;[1]_2\big)=e\big(\pi,\;[s-t]_2\big)}
$$

$[s-t]_2$는 공개 매개변수 $[s]_2$에서 $t[1]_2$를 빼서 얻는다. 등식은 하나이지만 양변에 Pairing이 하나씩 있으므로 Pairing은 두 번 계산한다.

KZG에서 $D$와 $\pi$는 각각 군 원소 하나다. 따라서 **Opening이 몇 개든 증명 $(D,\pi)$의 크기는 같다.**

## Why It Works

**모든 주장이 참인 경우.** $g(X)$는 다항식이고, 식을 그대로 정리하면

$$
h(t)-g(t)
=\sum_i\frac{r^i f_i(t)}{t-z_i}
-\sum_i\frac{r^i\,(f_i(t)-y_i)}{t-z_i}
=\sum_i\frac{r^i y_i}{t-z_i}=y
$$

이므로 $h(X)-g(X)-y$는 $(X-t)$로 나누어떨어진다. 정직한 Prover의 $\pi$는 검증을 통과한다. 이 성질을 Completeness(완전성)라고 한다.

**거짓 주장이 섞인 경우.** $i$번째 주장의 오류를 $\delta_i=f_i(z_i)-y_i$라고 하자. 4장에서 본 대로 몫은 다항식 부분과 나머지로 나뉜다.

$$
\frac{f_i(X)-y_i}{X-z_i}=q_i(X)+\frac{\delta_i}{X-z_i}
$$

$g(X)$에서 같은 점 $z$에 해당하는 나머지들을 모으면 $\dfrac{P_z(r)}{X-z}$, $P_z(r)=\sum_{i:\,z_i=z}\delta_i r^i$가 된다. 거짓 주장이 있으면 $P_z$는 $r$에 대한 0이 아닌 다항식이고 차수는 $m-1$ 이하다. $r$을 뽑은 뒤 $P_z(r)=0$이 될 확률은 최대 $(m-1)/p$다. 서로 다른 점의 나머지는 분모가 다르므로 상쇄되지 않는다. 즉 $r$은 **상쇄를 막는다.**

그래도 Prover는 다항식이 아닌 $g$ 대신 아무 다항식 $\tilde g$를 Commit해 $D$로 보낼 수 있다. 이때 5.2의 검증은 $g(t)=\tilde g(t)$를 확인하는 것과 같다. $g-\tilde g$는 0이 아닌 유리함수이고, 분모를 곱하면 차수 $d+m$ 이하의 0이 아닌 다항식이 된다. 무작위 $t$에서 이것이 0이 될 확률은 최대 $(d+m)/p$다. 0이 아닌 다항식이 무작위 점에서 0이 될 확률은 차수를 $p$로 나눈 값을 넘지 않는다는 사실을 Schwartz-Zippel Lemma(슈바르츠-지펠 보조정리)라고 한다. 즉 $t$는 **$D$가 진짜 $g$의 Commitment인지 검사한다.**

이 확률 논리에 더해, 실제 Soundness(건전성)는 PCS의 Binding, 마지막 Opening Proof의 안전성, Fiat-Shamir Transform의 안전한 구현을 전제로 한다.

## Worked Example

Opening 두 개를 증명한다.

$$
\begin{aligned}
f_0(X)&=X+2,&z_0&=3,&y_0&=5\\
f_1(X)&=X^2+1,&z_1&=2,&y_1&=5
\end{aligned}
$$

실제로 $f_0(3)=5$, $f_1(2)=5$다. 계산을 보이기 위해 $r=2$, $t=4$로 정한다. 실제로는 두 값을 Hash로 얻는다. 이 예시는 모두 정수로 떨어지므로 $\mathbb{F}_p$에서도 같은 결과가 나온다.

**① 몫을 결합한다.**

$$
\begin{aligned}
q_0(X)&=\frac{X+2-5}{X-3}=1\\
q_1(X)&=\frac{X^2+1-5}{X-2}=X+2\\
g(X)&=q_0(X)+2\,q_1(X)=2X+5
\end{aligned}
$$

Prover는 $D=\operatorname{Commit}(2X+5)$를 보낸다.

**② $t=4$에서 Verifier가 계산한다.**

$$
\begin{aligned}
y&=\frac{5}{4-3}+\frac{2\cdot5}{4-2}=10\\
E&=\frac{1}{4-3}C_0+\frac{2}{4-2}C_1=C_0+C_1
\end{aligned}
$$

$E$에 대응하는 다항식은 $h(X)=f_0(X)+f_1(X)=X^2+X+3$이다. 확인할 관계는 실제로 성립한다.

$$
h(4)-g(4)=23-13=10=y
$$

**③ 마지막 Opening Proof를 만든다.**

$$
w(X)=\frac{h(X)-g(X)-10}{X-4}=\frac{X^2-X-12}{X-4}=X+3
$$

KZG라면 $D=[2s+5]_1$, $\pi=[s+3]_1$이다. Verifier는 $E=[s^2+s+3]_1$을 계산한 뒤 다음을 확인한다.

$$
\begin{aligned}
e\big(E-D-[10]_1,\,[1]_2\big)
&=e\big([s^2-s-12]_1,\,[1]_2\big)\\
&=e\big([s+3]_1,\,[s-4]_2\big)\\
&=e\big(\pi,\,[s-4]_2\big)
\end{aligned}
$$

Opening 두 개가 $(D,\pi)$ 하나로 확인됐다.

## Multiproof in Verkle Tree

Verkle Tree에서 블록 하나를 검증할 때 보내는 데이터, 즉 Witness(위트니스)는 다음으로 이루어진다.

- **Key와 Value:** 읽은 상태 값
- **경로 위의 Commitment:** Root부터 각 Leaf까지의 연결을 확인하는 데 필요한 Commitment. 경로가 겹치면 한 번만
- **Multiproof:** KZG 기준 $(D,\pi)$

Root Commitment는 Verifier가 이미 아는 값이므로 보내지 않는다. 평가점 $z_i$는 Key에서, 평가값 $y_i$는 자식 Commitment나 Leaf 값에서 Verifier가 직접 계산하므로 이것도 보내지 않는다.

Multiproof로 고정되는 것은 $(D,\pi)$뿐이다. 경로 위의 Commitment 수는 읽은 Key 수에 따라 늘어난다. 그래서 **증명 크기가 일정하다는 말과 Witness 크기가 일정하다는 말은 다르다.**

Prover 쪽 조건도 하나 있다. $g(X)$를 만들려면 각 $f_i(X)$ 전체가 필요하다. 트리로 말하면 경로 위 각 노드의 자식 값 전부(너비 256이면 256개)가 필요하고, 열려는 자식 하나만으로는 만들 수 없다. 이더리움 설계에서는 전체 상태를 가진 블록 생성자가 Witness를 만든다.

## Beyond KZG

5장에서 KZG에 의존한 부분은 5.3의 마지막 검증뿐이다. $D$, $E$, $y$를 만드는 과정은 Additive Homomorphism만 있으면 성립한다. 그래서 같은 방식을 Pedersen Commitment(페더슨 커밋먼트)와 IPA(Inner Product Argument, 내적 증명)에도 쓸 수 있다.

이더리움의 Verkle Tree 제안은 KZG가 아니라 Bandersnatch 곡선 위의 Pedersen Commitment와 IPA를 쓴다. Trusted Setup과 Pairing이 필요 없는 대신, 마지막 Opening Proof가 군 원소 하나가 아니라 $2\log_2 d$개이고 검증 비용이 $d$에 비례한다. Opening을 하나로 묶어 한 번만 검증하므로 이 비용을 모든 Opening이 나누어 부담한다. 이더리움 구현 go-ipa의 Multiproof는 $D$와 IPA Proof 하나로 이루어진다.

## Summary

1. "$f(z)=y$다"는 "몫 $\frac{f(X)-y}{X-z}$가 다항식이다"와 같다. Opening Proof는 이 몫의 Commitment다.
2. Opening이 $m$개면 몫도 $m$개다. 무작위 가중치 $r^i$를 곱해 더한 $g(X)$의 Commitment $D$를 보낸다.
3. $D$가 고정된 뒤 무작위 점 $t$를 뽑는다. Verifier는 $C_i$들을 결합해 $E$를, 공개 값으로 $y$를 계산한다.
4. $E-D$를 $t$에서 열면 $y$가 나오는지 Opening Proof $\pi$ 하나로 확인한다.

증명 $(D,\pi)$의 크기는 Opening 개수와 무관하다. 이 기법은 Zero-Knowledge(영지식)를 제공하지 않는다. $z_i$와 $y_i$는 공개된다.

## References

- Dankrad Feist, [PCS multiproofs using random evaluation](https://dankradfeist.de/ethereum/2021/06/18/pcs-multiproofs.html), 2021-06-18.
- Dankrad Feist, [Inner Product Arguments](https://dankradfeist.de/ethereum/2021/07/27/inner-product-arguments.html), 2021-07-27.
- Vitalik Buterin, [Verkle trees](https://vitalik.eth.limo/general/2021/06/18/verkle.html), 2021-06-18.
- Guillaume Ballet, Dankrad Feist, [Verkle tree structure](https://blog.ethereum.org/2021/12/02/verkle-tree-structure/), 2021-12-02.
- [go-ipa](https://pkg.go.dev/github.com/crate-crypto/go-ipa), Multiproof 구현.